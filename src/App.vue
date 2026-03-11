<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

// ── State ─────────────────────────────────────────────────────────────────────
const videoRef   = ref(null)
const canvasRef  = ref(null)
const status     = ref('Avvio...')
const useFront   = ref(false)
const fps        = ref(0)

// Fasi: 'calibration' | 'playing'
const phase      = ref('calibration')

// Calibrazione
const detectedIds     = ref([])
const corners         = ref(null)   // { TL, TR, BR, BL } — centri marker in px
const consecutiveHits = ref(0)
const CONSECUTIVE_NEEDED = 4

// Gioco
const piecePositions = ref([])  // [{ id, col, row }]

const GRID_COLS = 19
const GRID_ROWS = 26
const CORNER_IDS = { TL: 0, TR: 1, BR: 2, BL: 3 }

let detector   = null
let stream     = null
let rafId      = null
let frameCount = 0
let lastFps    = performance.now()

// ── Camera ────────────────────────────────────────────────────────────────────
async function startCamera(front = false) {
  stream?.getTracks().forEach(t => t.stop())
  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: front ? 'user' : { exact: 'environment' },
               width: { ideal: 1280 }, height: { ideal: 720 } }
    })
  } catch {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: front ? 'user' : 'environment',
               width: { ideal: 1280 }, height: { ideal: 720 } }
    })
  }
  videoRef.value.srcObject = stream
  await videoRef.value.play()
  status.value = `✓ ${stream.getVideoTracks()[0]?.label ?? 'Camera attiva'}`
}

async function toggleCamera() {
  useFront.value = !useFront.value
  await startCamera(useFront.value)
}

// ── Geometria ─────────────────────────────────────────────────────────────────
function markerCenter(marker) {
  const c = marker.corners
  return {
    x: c.reduce((s, p) => s + p.x, 0) / 4,
    y: c.reduce((s, p) => s + p.y, 0) / 4
  }
}

// Dato un punto (px,py) nel canvas e i 4 angoli della mappa,
// restituisce la cella (col, row) nella griglia 19x26
// usando interpolazione bilineare prospettica
function pointToGrid(px, py, TL, TR, BR, BL) {
  // Risolve iterativamente col e row con Newton
  let u = 0.5, v = 0.5
  for (let i = 0; i < 20; i++) {
    const x = (1-u)*(1-v)*TL.x + u*(1-v)*TR.x + u*v*BR.x + (1-u)*v*BL.x
    const y = (1-u)*(1-v)*TL.y + u*(1-v)*TR.y + u*v*BR.y + (1-u)*v*BL.y
    const dxdu = -(1-v)*TL.x + (1-v)*TR.x + v*BR.x - v*BL.x
    const dydu = -(1-v)*TL.y + (1-v)*TR.y + v*BR.y - v*BL.y
    const dxdv = -(1-u)*TL.x - u*TR.x + u*BR.x + (1-u)*BL.x
    const dydv = -(1-u)*TL.y - u*TR.y + u*BR.y + (1-u)*BL.y
    const det  = dxdu*dydv - dxdv*dydu
    if (Math.abs(det) < 1e-10) break
    const ex = px - x, ey = py - y
    u += ( dydv*ex - dxdv*ey) / det
    v += (-dydu*ex + dxdu*ey) / det
    u = Math.max(0, Math.min(1, u))
    v = Math.max(0, Math.min(1, v))
  }
  return {
    col: Math.floor(u * GRID_COLS) + 1,
    row: Math.floor(v * GRID_ROWS) + 1
  }
}

// ── Draw ──────────────────────────────────────────────────────────────────────
function drawMarkerOverlay(ctx, marker, color = '#00ff88') {
  const c = marker.corners
  ctx.beginPath()
  ctx.moveTo(c[0].x, c[0].y)
  c.slice(1).forEach(p => ctx.lineTo(p.x, p.y))
  ctx.closePath()
  ctx.strokeStyle = color; ctx.lineWidth = 3; ctx.stroke()

  const cx = c.reduce((s, p) => s + p.x, 0) / 4
  const cy = c.reduce((s, p) => s + p.y, 0) / 4
  ctx.font = 'bold 18px monospace'; ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'; ctx.fillStyle = '#fff'
  ctx.shadowColor = '#000'; ctx.shadowBlur = 5
  ctx.fillText(`ID ${marker.id}`, cx, cy)
  ctx.shadowBlur = 0
}

function drawGrid(ctx, TL, TR, BR, BL) {
  ctx.save()
  ctx.strokeStyle = 'rgba(100,180,255,0.4)'
  ctx.lineWidth = 1

  for (let c = 0; c <= GRID_COLS; c++) {
    const t  = c / GRID_COLS
    const tx = TL.x + t*(TR.x-TL.x), ty = TL.y + t*(TR.y-TL.y)
    const bx = BL.x + t*(BR.x-BL.x), by = BL.y + t*(BR.y-BL.y)
    ctx.beginPath(); ctx.moveTo(tx,ty); ctx.lineTo(bx,by); ctx.stroke()
  }
  for (let r = 0; r <= GRID_ROWS; r++) {
    const t  = r / GRID_ROWS
    const lx = TL.x + t*(BL.x-TL.x), ly = TL.y + t*(BL.y-TL.y)
    const rx = TR.x + t*(BR.x-TR.x), ry = TR.y + t*(BR.y-TR.y)
    ctx.beginPath(); ctx.moveTo(lx,ly); ctx.lineTo(rx,ry); ctx.stroke()
  }
  ctx.restore()
}

function drawPieceOnGrid(ctx, piece, TL, TR, BR, BL) {
  const { col, row } = piece
  const u0 = (col-1)/GRID_COLS, u1 = col/GRID_COLS
  const v0 = (row-1)/GRID_ROWS, v1 = row/GRID_ROWS

  function blerp(u, v) {
    return {
      x: (1-u)*(1-v)*TL.x + u*(1-v)*TR.x + u*v*BR.x + (1-u)*v*BL.x,
      y: (1-u)*(1-v)*TL.y + u*(1-v)*TR.y + u*v*BR.y + (1-u)*v*BL.y
    }
  }

  const p00 = blerp(u0,v0), p10 = blerp(u1,v0)
  const p11 = blerp(u1,v1), p01 = blerp(u0,v1)
  const cx = (p00.x+p10.x+p11.x+p01.x)/4
  const cy = (p00.y+p10.y+p11.y+p01.y)/4

  // Highlight cella
  ctx.beginPath()
  ctx.moveTo(p00.x,p00.y); ctx.lineTo(p10.x,p10.y)
  ctx.lineTo(p11.x,p11.y); ctx.lineTo(p01.x,p01.y)
  ctx.closePath()
  ctx.fillStyle   = 'rgba(255,170,0,0.35)'; ctx.fill()
  ctx.strokeStyle = '#ffaa00'; ctx.lineWidth = 2; ctx.stroke()

  // Label
  ctx.font = 'bold 14px monospace'; ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'; ctx.fillStyle = '#fff'
  ctx.shadowColor = '#000'; ctx.shadowBlur = 4
  ctx.fillText(`${col},${row}`, cx, cy)
  ctx.shadowBlur = 0
}

// ── Loop principale ───────────────────────────────────────────────────────────
function loop() {
  const video  = videoRef.value
  const canvas = canvasRef.value
  if (!video || !canvas || video.readyState < 2) {
    rafId = requestAnimationFrame(loop); return
  }

  const w = video.videoWidth, h = video.videoHeight
  if (!w || !h) { rafId = requestAnimationFrame(loop); return }
  if (canvas.width !== w || canvas.height !== h) {
    canvas.width = w; canvas.height = h
  }

  const ctx = canvas.getContext('2d', { willReadFrequently: true })
  ctx.drawImage(video, 0, 0, w, h)

  const markers = detector.detect(ctx.getImageData(0, 0, w, h))
  const byId    = {}
  markers.forEach(m => { byId[m.id] = m })
  detectedIds.value = markers.map(m => m.id)

  if (phase.value === 'calibration') {
    loopCalibration(ctx, byId)
  } else {
    loopPlaying(ctx, byId)
  }

  frameCount++
  const now = performance.now()
  if (now - lastFps >= 1000) {
    fps.value = frameCount; frameCount = 0; lastFps = now
  }

  rafId = requestAnimationFrame(loop)
}

function loopCalibration(ctx, byId) {
  const TL = byId[CORNER_IDS.TL]
  const TR = byId[CORNER_IDS.TR]
  const BR = byId[CORNER_IDS.BR]
  const BL = byId[CORNER_IDS.BL]

  const allFound = TL && TR && BR && BL

  if (allFound) {
    consecutiveHits.value++
    // Disegna griglia provvisoria
    drawGrid(ctx,
      markerCenter(TL), markerCenter(TR),
      markerCenter(BR), markerCenter(BL)
    )
    // Disegna marker corner in arancione
    ;[TL, TR, BR, BL].forEach(m => drawMarkerOverlay(ctx, m, '#ffaa00'))

    if (consecutiveHits.value >= CONSECUTIVE_NEEDED) {
      // Salva calibrazione
      corners.value = {
        TL: markerCenter(TL), TR: markerCenter(TR),
        BR: markerCenter(BR), BL: markerCenter(BL),
        idTL: TL.id, idTR: TR.id, idBR: BR.id, idBL: BL.id
      }
    }
  } else {
    consecutiveHits.value = 0
    // Disegna solo i marker trovati
    Object.values(byId).forEach(m => drawMarkerOverlay(ctx, m, '#00ff88'))
  }
}

function loopPlaying(ctx, byId) {
  const { TL, TR, BR, BL } = corners.value

  // Ridisegna griglia
  drawGrid(ctx, TL, TR, BR, BL)

  // Ignora i 4 corner, rileva solo i pezzi
  const cornerIds = new Set(Object.values(CORNER_IDS))
  const pieces = []

  Object.values(byId).forEach(m => {
    if (cornerIds.has(m.id)) {
      drawMarkerOverlay(ctx, m, '#ffaa0066')
      return
    }
    const center = markerCenter(m)
    const { col, row } = pointToGrid(center.x, center.y, TL, TR, BR, BL)
    pieces.push({ id: m.id, col, row })
    drawMarkerOverlay(ctx, m, '#00aaff')
    drawPieceOnGrid(ctx, { col, row }, TL, TR, BR, BL)
  })

  piecePositions.value = pieces
}

// ── Lifecycle ─────────────────────────────────────────────────────────────────
onMounted(async () => {
  try {
    const mod = await import('js-aruco2')
    const D = mod?.default?.Detector ?? mod?.AR?.Detector ??
              mod?.Detector ?? mod?.default?.AR?.Detector ?? null
    if (!D) throw new Error('Detector non trovato')
    detector = new D()
    await startCamera(false)
    rafId = requestAnimationFrame(loop)
  } catch(e) {
    status.value = `✗ ${e.message}`
  }
})

onUnmounted(() => {
  cancelAnimationFrame(rafId)
  stream?.getTracks().forEach(t => t.stop())
})

function resetCalibration() {
  corners.value = null
  consecutiveHits.value = 0
  phase.value = 'calibration'
  piecePositions.value = []
}
</script>

<template>
  <div class="app">
    <header>
      <h1>⚔️ HeroQuest AR</h1>
      <div class="badges">
        <span class="badge fps">{{ fps }} fps</span>
        <span class="badge" :class="phase === 'playing' ? 'ok' : 'warn'">
          {{ phase === 'playing' ? '▶ Gioco' : '⚙ Calibrazione' }}
        </span>
      </div>
    </header>

    <!-- VIEWPORT -->
    <div class="viewport">
      <video ref="videoRef" playsinline muted class="hidden" />
      <canvas ref="canvasRef" />

      <!-- Overlay calibrazione: barra progressiva -->
      <div v-if="phase === 'calibration'" class="cal-overlay">
        <div v-if="!corners" class="cal-hint">
          <span v-if="consecutiveHits < CONSECUTIVE_NEEDED">
            Inquadra i 4 marker agli angoli della mappa
            <b>{{ consecutiveHits }}/{{ CONSECUTIVE_NEEDED }}</b>
          </span>
        </div>
      </div>
    </div>

    <!-- STATUS + FLIP -->
    <div class="row">
      <div class="status">{{ status }}</div>
      <button class="btn-sm" @click="toggleCamera">
        {{ useFront ? '📷 Post.' : '🤳 Front.' }}
      </button>
    </div>

    <!-- ── CALIBRAZIONE ── -->
    <template v-if="phase === 'calibration'">
      <div v-if="!corners" class="panel">
        <h2>⚙ Calibrazione</h2>
        <p class="hint">Posiziona i marker agli angoli della mappa:<br>
          <b style="color:#ffaa00">ID 0</b> = ↖ &nbsp;
          <b style="color:#ffaa00">ID 1</b> = ↗ &nbsp;
          <b style="color:#ffaa00">ID 2</b> = ↘ &nbsp;
          <b style="color:#ffaa00">ID 3</b> = ↙
        </p>
        <div class="progress-bar">
          <div class="progress-fill"
            :style="{ width: (consecutiveHits / CONSECUTIVE_NEEDED * 100) + '%' }" />
        </div>
        <p class="hint" style="margin-top:.4rem">
          Marker visibili: <b style="color:#00ff88">{{ detectedIds.join(', ') || '—' }}</b>
        </p>
      </div>

      <!-- Calibrazione OK -->
      <div v-else class="panel cal-done">
        <h2>✅ Calibrazione effettuata!</h2>
        <div class="corner-grid">
          <div class="corner-chip">
            <span class="corner-label">Corner 1 — Top Left</span>
            <span class="corner-id">ID {{ corners.idTL }}</span>
          </div>
          <div class="corner-chip">
            <span class="corner-label">Corner 2 — Top Right</span>
            <span class="corner-id">ID {{ corners.idTR }}</span>
          </div>
          <div class="corner-chip">
            <span class="corner-label">Corner 3 — Bottom Right</span>
            <span class="corner-id">ID {{ corners.idBR }}</span>
          </div>
          <div class="corner-chip">
            <span class="corner-label">Corner 4 — Bottom Left</span>
            <span class="corner-id">ID {{ corners.idBL }}</span>
          </div>
        </div>
        <button class="btn-play" @click="phase = 'playing'">▶ GIOCA</button>
        <button class="btn-reset" @click="resetCalibration">↺ Ricalibra</button>
      </div>
    </template>

    <!-- ── GIOCO ── -->
    <template v-if="phase === 'playing'">
      <div class="panel">
        <div class="play-header">
          <h2>🗺 Mappa {{ GRID_COLS }}×{{ GRID_ROWS }}</h2>
          <button class="btn-reset" @click="resetCalibration">↺ Ricalibra</button>
        </div>

        <div v-if="!piecePositions.length" class="hint" style="margin-top:.5rem">
          Posiziona marker sulla mappa per rilevarne la posizione...
        </div>

        <div v-else class="piece-list">
          <div v-for="p in piecePositions" :key="p.id" class="piece-chip">
            <span class="piece-id">ID {{ p.id }}</span>
            <span class="piece-pos">col <b>{{ p.col }}</b> · riga <b>{{ p.row }}</b></span>
          </div>
        </div>
      </div>
    </template>

  </div>
</template>

<style scoped>
* { box-sizing: border-box; margin: 0; padding: 0 }
.app {
  min-height: 100vh; background: #0d0d0d; color: #ddd;
  font-family: 'Segoe UI', sans-serif;
  display: flex; flex-direction: column; align-items: center;
  padding: 1rem; gap: .8rem;
}

header {
  width: 100%; max-width: 900px;
  display: flex; justify-content: space-between; align-items: center;
}
h1 { color: #c8a84b; font-size: 1.2rem }
.badges { display: flex; gap: .4rem }
.badge {
  padding: 3px 10px; border-radius: 20px;
  font-size: .75rem; font-weight: 700; font-family: monospace;
}
.badge.ok   { background: #1a3d2b; color: #00ff88 }
.badge.warn { background: #2b2200; color: #ffaa00 }
.badge.fps  { background: #1a1a2e; color: #7cb3ff }

.viewport {
  position: relative; width: 100%; max-width: 900px;
  aspect-ratio: 16/9; background: #000;
  border: 1px solid #222; border-radius: 8px; overflow: hidden;
}
.hidden { display: none }
canvas { width: 100%; height: 100%; display: block }

.cal-overlay {
  position: absolute; bottom: 0; left: 0; right: 0;
  padding: .5rem; display: flex; justify-content: center;
}
.cal-hint {
  background: rgba(0,0,0,.7); border-radius: 8px;
  padding: .4rem 1rem; font-size: .85rem; color: #ffaa00; text-align: center;
}

.row {
  width: 100%; max-width: 900px;
  display: flex; gap: .5rem; align-items: center;
}
.status {
  flex: 1; background: #161616; border: 1px solid #222;
  border-radius: 6px; padding: .4rem .8rem; font-size: .8rem; color: #aaa;
}
.btn-sm {
  background: #1a1a2e; border: 1px solid #333; border-radius: 6px;
  color: #7cb3ff; padding: .4rem .8rem; font-size: .85rem; cursor: pointer;
  white-space: nowrap;
}

.panel {
  width: 100%; max-width: 900px; background: #111;
  border: 1px solid #222; border-radius: 10px; padding: 1rem;
}
.panel h2 { font-size: 1rem; color: #c8a84b; margin-bottom: .6rem }
.hint { font-size: .85rem; color: #888; line-height: 1.5 }

.progress-bar {
  width: 100%; height: 8px; background: #222;
  border-radius: 4px; overflow: hidden; margin-top: .7rem;
}
.progress-fill {
  height: 100%; background: #ffaa00;
  border-radius: 4px; transition: width .2s;
}

.cal-done { border-color: #00ff8844 }
.corner-grid {
  display: grid; grid-template-columns: 1fr 1fr;
  gap: .5rem; margin: .7rem 0 1rem;
}
.corner-chip {
  background: #0d2b1a; border: 1px solid #00ff8822;
  border-radius: 8px; padding: .5rem .8rem;
  display: flex; flex-direction: column; gap: .2rem;
}
.corner-label { font-size: .75rem; color: #888 }
.corner-id { font-family: monospace; font-weight: 700; color: #00ff88; font-size: 1rem }

.btn-play {
  width: 100%; padding: .9rem; border-radius: 10px;
  background: #c8a84b; border: none; color: #000;
  font-size: 1.1rem; font-weight: 900; cursor: pointer;
  letter-spacing: .05em; margin-bottom: .4rem;
}
.btn-play:active { background: #a8882b }
.btn-reset {
  background: transparent; border: 1px solid #444;
  border-radius: 6px; color: #666; padding: .3rem .8rem;
  font-size: .8rem; cursor: pointer;
}

.play-header {
  display: flex; justify-content: space-between; align-items: center;
}
.piece-list { display: flex; flex-direction: column; gap: .4rem; margin-top: .6rem }
.piece-chip {
  background: #0d1f2b; border: 1px solid #00aaff33;
  border-radius: 8px; padding: .5rem .8rem;
  display: flex; align-items: center; gap: 1rem;
}
.piece-id {
  font-family: monospace; font-weight: 700;
  color: #00aaff; font-size: 1rem; min-width: 50px;
}
.piece-pos { font-size: .9rem; color: #aaa }
.piece-pos b { color: #fff }
</style>