<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// ── State ─────────────────────────────────────────────────────────────────────
const videoRef  = ref(null)
const canvasRef = ref(null)
const status    = ref('Avvio...')
const useFront  = ref(false)
const fps       = ref(0)

// Fasi: 0=TL, 1=TR, 2=BL, 3=BR, 4=piece_registration, 5=playing
const step      = ref(0)
const savedCorners = ref([])   // [{id, label, x, y}]
const pieceId      = ref(null) // id marcatore pezzo
const piecePos     = ref(null) // {col, row}

const detectedNow  = ref([])   // marker visibili in questo frame
const lastDetected = ref(null) // primo marker stabile nel frame

const STEPS = [
  { label: 'Angolo 1',  desc: 'IN ALTO A SINISTRA ↖',  short: 'TL' },
  { label: 'Angolo 2',  desc: 'IN ALTO A DESTRA ↗',    short: 'TR' },
  { label: 'Angolo 3',  desc: 'IN BASSO A SINISTRA ↙', short: 'BL' },
  { label: 'Angolo 4',  desc: 'IN BASSO A DESTRA ↘',   short: 'BR' },
  { label: 'Pezzo',     desc: 'MOSTRA IL MARCATORE PEZZO', short: 'PC' },
]

const GRID_COLS = 19
const GRID_ROWS = 26

let detector   = null
let stream     = null
let rafId      = null
let frameCount = 0
let lastFps    = performance.now()

// ── Helpers geometria ─────────────────────────────────────────────────────────
function markerCenter(marker) {
  const c = marker.corners
  return {
    x: c.reduce((s, p) => s + p.x, 0) / 4,
    y: c.reduce((s, p) => s + p.y, 0) / 4
  }
}

function pointToGrid(px, py) {
  const TL = savedCorners.value.find(c => c.short === 'TL')
  const TR = savedCorners.value.find(c => c.short === 'TR')
  const BR = savedCorners.value.find(c => c.short === 'BR')
  const BL = savedCorners.value.find(c => c.short === 'BL')
  if (!TL || !TR || !BR || !BL) return null

  let u = 0.5, v = 0.5
  for (let i = 0; i < 20; i++) {
    const x    = (1-u)*(1-v)*TL.x + u*(1-v)*TR.x + u*v*BR.x + (1-u)*v*BL.x
    const y    = (1-u)*(1-v)*TL.y + u*(1-v)*TR.y + u*v*BR.y + (1-u)*v*BL.y
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
function drawMarkerBox(ctx, marker, color, label) {
  const c = marker.corners
  ctx.beginPath()
  ctx.moveTo(c[0].x, c[0].y)
  c.slice(1).forEach(p => ctx.lineTo(p.x, p.y))
  ctx.closePath()
  ctx.strokeStyle = color; ctx.lineWidth = 3; ctx.stroke()

  const cx = c.reduce((s,p) => s+p.x, 0) / 4
  const cy = c.reduce((s,p) => s+p.y, 0) / 4
  ctx.font = 'bold 16px monospace'; ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'; ctx.fillStyle = '#fff'
  ctx.shadowColor = '#000'; ctx.shadowBlur = 5
  ctx.fillText(label ?? `ID ${marker.id}`, cx, cy)
  ctx.shadowBlur = 0
}

function drawGrid(ctx) {
  const TL = savedCorners.value.find(c => c.short === 'TL')
  const TR = savedCorners.value.find(c => c.short === 'TR')
  const BR = savedCorners.value.find(c => c.short === 'BR')
  const BL = savedCorners.value.find(c => c.short === 'BL')
  if (!TL || !TR || !BR || !BL) return

  ctx.save()
  ctx.strokeStyle = 'rgba(100,180,255,0.35)'; ctx.lineWidth = 1

  for (let c = 0; c <= GRID_COLS; c++) {
    const t  = c / GRID_COLS
    ctx.beginPath()
    ctx.moveTo(TL.x + t*(TR.x-TL.x), TL.y + t*(TR.y-TL.y))
    ctx.lineTo(BL.x + t*(BR.x-BL.x), BL.y + t*(BR.y-BL.y))
    ctx.stroke()
  }
  for (let r = 0; r <= GRID_ROWS; r++) {
    const t  = r / GRID_ROWS
    ctx.beginPath()
    ctx.moveTo(TL.x + t*(BL.x-TL.x), TL.y + t*(BL.y-TL.y))
    ctx.lineTo(TR.x + t*(BR.x-TR.x), TR.y + t*(BR.y-TR.y))
    ctx.stroke()
  }
  ctx.restore()
}

function drawCellHighlight(ctx, col, row) {
  const TL = savedCorners.value.find(c => c.short === 'TL')
  const TR = savedCorners.value.find(c => c.short === 'TR')
  const BR = savedCorners.value.find(c => c.short === 'BR')
  const BL = savedCorners.value.find(c => c.short === 'BL')
  if (!TL || !TR || !BR || !BL) return

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

  ctx.beginPath()
  ctx.moveTo(p00.x,p00.y); ctx.lineTo(p10.x,p10.y)
  ctx.lineTo(p11.x,p11.y); ctx.lineTo(p01.x,p01.y)
  ctx.closePath()
  ctx.fillStyle = 'rgba(255,170,0,0.4)'; ctx.fill()
  ctx.strokeStyle = '#ffaa00'; ctx.lineWidth = 2; ctx.stroke()
}

// ── Loop ──────────────────────────────────────────────────────────────────────
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
  detectedNow.value = markers.map(m => m.id)

  const cornerIds = new Set(savedCorners.value.map(c => c.id))

  if (step.value < 5) {
    // Fase calibrazione: mostra tutti i marker rilevati
    markers.forEach(m => {
      const isCorner = cornerIds.has(m.id)
      drawMarkerBox(ctx, m,
        isCorner ? '#00ff88' : '#ffaa00',
        isCorner
          ? savedCorners.value.find(c => c.id === m.id)?.short
          : `ID ${m.id}`
      )
    })

    // Salva il primo marker visibile come candidato
    const nonCorner = markers.find(m => !cornerIds.has(m.id))
    if (nonCorner) {
      const center = markerCenter(nonCorner)
      lastDetected.value = { id: nonCorner.id, x: center.x, y: center.y }
    } else if (markers.length > 0 && step.value < 4) {
      // Per i corner mostra tutti
      const first = markers[0]
      const center = markerCenter(first)
      lastDetected.value = { id: first.id, x: center.x, y: center.y }
    } else {
      lastDetected.value = null
    }

  } else {
    // Fase playing
    drawGrid(ctx)

    // Disegna corner salvati (semitrasparenti)
    savedCorners.value.forEach(sc => {
      if (byId[sc.id]) drawMarkerBox(ctx, byId[sc.id], '#ffffff33', sc.short)
    })

    // Rileva pezzo
    const pieceMarker = byId[pieceId.value]
    if (pieceMarker) {
      const center = markerCenter(pieceMarker)
      const pos    = pointToGrid(center.x, center.y)
      if (pos) {
        piecePos.value = pos
        drawCellHighlight(ctx, pos.col, pos.row)
        drawMarkerBox(ctx, pieceMarker, '#00aaff', `${pos.col},${pos.row}`)
      }
    } else {
      piecePos.value = null
    }
  }

  frameCount++
  const now = performance.now()
  if (now - lastFps >= 1000) {
    fps.value = frameCount; frameCount = 0; lastFps = now
  }

  rafId = requestAnimationFrame(loop)
}

// ── Azioni utente ─────────────────────────────────────────────────────────────
function confirmStep() {
  if (!lastDetected.value) return
  const current = STEPS[step.value]
  savedCorners.value.push({
    id:    lastDetected.value.id,
    x:     lastDetected.value.x,
    y:     lastDetected.value.y,
    short: current.short,
    label: current.label,
    desc:  current.desc
  })
  lastDetected.value = null
  step.value++
}

function confirmPiece() {
  if (!lastDetected.value) return
  pieceId.value = lastDetected.value.id
  step.value = 5
}

function reset() {
  step.value = 0
  savedCorners.value = []
  pieceId.value = null
  piecePos.value = null
  lastDetected.value = null
}

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
  status.value = `✓ ${stream.getVideoTracks()[0]?.label ?? 'Camera'}`
}

async function toggleCamera() {
  useFront.value = !useFront.value
  await startCamera(useFront.value)
}

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
</script>

<template>
  <div class="app">

    <header>
      <h1>⚔️ HeroQuest AR</h1>
      <div class="badges">
        <span class="badge fps">{{ fps }} fps</span>
        <button class="btn-sm" @click="toggleCamera">
          {{ useFront ? '📷 Post.' : '🤳 Front.' }}
        </button>
        <button class="btn-sm danger" @click="reset">↺</button>
      </div>
    </header>

    <!-- VIEWPORT -->
    <div class="viewport">
      <video ref="videoRef" playsinline muted class="hidden" />
      <canvas ref="canvasRef" />
    </div>

    <!-- ── STEP 0-3: calibrazione angoli ── -->
    <div v-if="step < 4" class="panel">
      <div class="step-header">
        <span class="step-pill">{{ step + 1 }} / 4</span>
        <h2>{{ STEPS[step].label }}: <span class="step-desc">{{ STEPS[step].desc }}</span></h2>
      </div>

      <p class="hint">Mostra alla camera il marker da usare come <b>{{ STEPS[step].desc }}</b></p>

      <!-- Angoli già salvati -->
      <div v-if="savedCorners.length" class="saved-list">
        <div v-for="c in savedCorners" :key="c.short" class="saved-chip">
          <span class="saved-short">{{ c.short }}</span>
          <span class="saved-desc">{{ c.desc }}</span>
          <span class="saved-id">ID {{ c.id }}</span>
        </div>
      </div>

      <!-- Marker rilevato ora -->
      <div class="detect-box" :class="lastDetected ? 'found' : 'waiting'">
        <template v-if="lastDetected">
          <span class="detect-icon">✓</span>
          <span class="detect-label">Rilevato <b>ID {{ lastDetected.id }}</b></span>
          <button class="btn-confirm" @click="confirmStep">
            Salva come {{ STEPS[step].desc }} →
          </button>
        </template>
        <template v-else>
          <span class="detect-icon spin">⧖</span>
          <span class="detect-label">In attesa del marker...</span>
        </template>
      </div>
    </div>

    <!-- ── STEP 4: registra pezzo ── -->
    <div v-else-if="step === 4" class="panel">
      <div class="step-header">
        <span class="step-pill">5 / 5</span>
        <h2>Registra il <span class="step-desc">MARCATORE PEZZO</span></h2>
      </div>

      <div class="saved-list">
        <div v-for="c in savedCorners" :key="c.short" class="saved-chip">
          <span class="saved-short">{{ c.short }}</span>
          <span class="saved-desc">{{ c.desc }}</span>
          <span class="saved-id">ID {{ c.id }}</span>
        </div>
      </div>

      <p class="hint" style="margin-top:.6rem">
        Mostra il marker che rappresenta il <b>pezzo da giocare</b>
        (deve avere ID diverso dagli angoli)
      </p>

      <div class="detect-box" :class="lastDetected ? 'found' : 'waiting'">
        <template v-if="lastDetected">
          <span class="detect-icon">✓</span>
          <span class="detect-label">Rilevato <b>ID {{ lastDetected.id }}</b></span>
          <button class="btn-confirm" @click="confirmPiece">
            ▶ INIZIA A GIOCARE
          </button>
        </template>
        <template v-else>
          <span class="detect-icon spin">⧖</span>
          <span class="detect-label">In attesa del marker pezzo...</span>
        </template>
      </div>
    </div>

    <!-- ── STEP 5: playing ── -->
    <div v-else class="panel playing">
      <div class="play-header">
        <h2>🗺 Mappa {{ GRID_COLS }}×{{ GRID_ROWS }}</h2>
        <button class="btn-sm danger" @click="reset">↺ Reset</button>
      </div>

      <div class="piece-info" :class="piecePos ? 'active' : 'idle'">
        <template v-if="piecePos">
          <span class="piece-icon">📍</span>
          <span>Pezzo <b style="color:#00aaff">ID {{ pieceId }}</b></span>
          <span class="piece-coords">
            Colonna <b>{{ piecePos.col }}</b> · Riga <b>{{ piecePos.row }}</b>
          </span>
        </template>
        <template v-else>
          <span class="piece-icon">⧖</span>
          <span style="color:#666">Posiziona il pezzo (ID {{ pieceId }}) sulla mappa...</span>
        </template>
      </div>

      <div class="corner-summary">
        <div v-for="c in savedCorners" :key="c.short" class="saved-chip mini">
          <span class="saved-short">{{ c.short }}</span>
          <span class="saved-id">ID {{ c.id }}</span>
        </div>
      </div>
    </div>

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
.badges { display: flex; gap: .4rem; align-items: center }
.badge.fps { background: #1a1a2e; color: #7cb3ff; padding: 3px 10px;
             border-radius: 20px; font-size: .75rem; font-weight: 700 }

.btn-sm {
  background: #1a1a2e; border: 1px solid #333; border-radius: 6px;
  color: #7cb3ff; padding: .35rem .7rem; font-size: .82rem; cursor: pointer;
}
.btn-sm.danger { color: #ff6666; border-color: #ff666633 }

.viewport {
  position: relative; width: 100%; max-width: 900px;
  aspect-ratio: 16/9; background: #000;
  border: 1px solid #222; border-radius: 8px; overflow: hidden;
}
.hidden { display: none }
canvas { width: 100%; height: 100%; display: block }

.panel {
  width: 100%; max-width: 900px; background: #111;
  border: 1px solid #222; border-radius: 10px; padding: 1rem;
  display: flex; flex-direction: column; gap: .7rem;
}

.step-header { display: flex; align-items: center; gap: .7rem }
.step-pill {
  background: #c8a84b22; color: #c8a84b; border: 1px solid #c8a84b44;
  border-radius: 20px; padding: 2px 10px; font-size: .8rem; font-weight: 700;
  white-space: nowrap;
}
h2 { font-size: 1rem; color: #ddd }
.step-desc { color: #ffaa00 }
.hint { font-size: .82rem; color: #777; line-height: 1.5 }

.saved-list { display: flex; flex-direction: column; gap: .3rem }
.saved-chip {
  display: flex; align-items: center; gap: .6rem;
  background: #0d2b1a; border: 1px solid #00ff8822;
  border-radius: 6px; padding: .4rem .7rem;
}
.saved-chip.mini { padding: .25rem .5rem }
.saved-short {
  font-family: monospace; font-weight: 900; color: #00ff88;
  min-width: 28px; font-size: .9rem;
}
.saved-desc { font-size: .8rem; color: #888; flex: 1 }
.saved-id { font-family: monospace; color: #fff; font-size: .85rem }

.detect-box {
  border-radius: 8px; padding: .8rem 1rem;
  display: flex; align-items: center; gap: .8rem; flex-wrap: wrap;
  border: 1px solid;
}
.detect-box.found   { background: #0d2b1a; border-color: #00ff8844 }
.detect-box.waiting { background: #1a1a1a; border-color: #333 }
.detect-icon { font-size: 1.2rem }
.detect-label { flex: 1; font-size: .9rem }
@keyframes spin { to { transform: rotate(360deg) } }
.spin { display: inline-block; animation: spin 1.5s linear infinite }

.btn-confirm {
  background: #c8a84b; border: none; border-radius: 8px;
  color: #000; font-weight: 900; padding: .5rem 1rem;
  font-size: .9rem; cursor: pointer; white-space: nowrap;
}
.btn-confirm:active { background: #a8882b }

.playing { border-color: #00aaff33 }
.play-header { display: flex; justify-content: space-between; align-items: center }

.piece-info {
  border-radius: 8px; padding: .8rem 1rem;
  display: flex; align-items: center; gap: .8rem; flex-wrap: wrap;
  border: 1px solid; font-size: .95rem;
}
.piece-info.active { background: #0d1f2b; border-color: #00aaff44 }
.piece-info.idle   { background: #1a1a1a; border-color: #333 }
.piece-icon { font-size: 1.2rem }
.piece-coords {
  margin-left: auto; font-family: monospace;
  font-size: 1.1rem; font-weight: 700; color: #ffaa00;
}

.corner-summary { display: flex; gap: .4rem; flex-wrap: wrap }
</style>