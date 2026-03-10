<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const videoRef    = ref(null)
const canvasRef   = ref(null)
const status      = ref('Avvio...')
const detectedIds = ref([])
const fps         = ref(0)
const gridReady   = ref(false)

const CORNER_IDS = { topLeft: 0, topRight: 1, bottomRight: 2, bottomLeft: 3 }
const GRID_COLS  = 19
const GRID_ROWS  = 26

let detector   = null
let stream     = null
let rafId      = null
let frameCount = 0
let lastFps    = performance.now()

function markerCenter(marker) {
  const c = marker.corners
  return [
    c.reduce((s, p) => s + p.x, 0) / 4,
    c.reduce((s, p) => s + p.y, 0) / 4
  ]
}

function drawMarker(ctx, marker, isCorner) {
  const c = marker.corners
  ctx.beginPath()
  ctx.moveTo(c[0].x, c[0].y)
  c.slice(1).forEach(p => ctx.lineTo(p.x, p.y))
  ctx.closePath()
  ctx.strokeStyle = isCorner ? '#ffaa00' : '#00ff88'
  ctx.lineWidth   = isCorner ? 4 : 2
  ctx.stroke()
  c.forEach((p, i) => {
    ctx.beginPath(); ctx.arc(p.x, p.y, 6, 0, Math.PI * 2)
    ctx.fillStyle = i === 0 ? '#ff3333' : (isCorner ? '#ffaa00' : '#00ff88')
    ctx.fill()
  })
  const cx = c.reduce((s, p) => s + p.x, 0) / 4
  const cy = c.reduce((s, p) => s + p.y, 0) / 4
  ctx.font = 'bold 18px monospace'
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle'
  ctx.fillStyle = '#fff'; ctx.shadowColor = '#000'; ctx.shadowBlur = 5
  ctx.fillText(`ID ${marker.id}`, cx, cy)
  ctx.shadowBlur = 0
}

function drawGrid(ctx, corners) {
  const { topLeft: TL, topRight: TR, bottomRight: BR, bottomLeft: BL } = corners
  ctx.save()
  ctx.strokeStyle = 'rgba(100,180,255,0.5)'
  ctx.lineWidth   = 1

  for (let c = 0; c <= GRID_COLS; c++) {
    const t  = c / GRID_COLS
    const tx = TL[0] + t * (TR[0] - TL[0])
    const ty = TL[1] + t * (TR[1] - TL[1])
    const bx = BL[0] + t * (BR[0] - BL[0])
    const by = BL[1] + t * (BR[1] - BL[1])
    ctx.beginPath(); ctx.moveTo(tx, ty); ctx.lineTo(bx, by); ctx.stroke()
  }

  for (let r = 0; r <= GRID_ROWS; r++) {
    const t  = r / GRID_ROWS
    const lx = TL[0] + t * (BL[0] - TL[0])
    const ly = TL[1] + t * (BL[1] - TL[1])
    const rx = TR[0] + t * (BR[0] - TR[0])
    const ry = TR[1] + t * (BR[1] - TR[1])
    ctx.beginPath(); ctx.moveTo(lx, ly); ctx.lineTo(rx, ry); ctx.stroke()
  }

  ctx.restore()
}

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

  const TL = byId[CORNER_IDS.topLeft]
  const TR = byId[CORNER_IDS.topRight]
  const BR = byId[CORNER_IDS.bottomRight]
  const BL = byId[CORNER_IDS.bottomLeft]
  gridReady.value = !!(TL && TR && BR && BL)

  if (gridReady.value) {
    drawGrid(ctx, {
      topLeft:     markerCenter(TL),
      topRight:    markerCenter(TR),
      bottomRight: markerCenter(BR),
      bottomLeft:  markerCenter(BL)
    })
  }

  markers.forEach(m => drawMarker(ctx, m, Object.values(CORNER_IDS).includes(m.id)))

  frameCount++
  const now = performance.now()
  if (now - lastFps >= 1000) {
    fps.value = frameCount; frameCount = 0; lastFps = now
  }

  rafId = requestAnimationFrame(loop)
}

onMounted(async () => {
  try {
    const mod = await import('js-aruco2')
    console.log('js-aruco2 keys:', Object.keys(mod))

    const DetectorClass =
      mod?.default?.Detector ??
      mod?.AR?.Detector ??
      mod?.Detector ??
      mod?.default?.AR?.Detector ??
      null

    if (!DetectorClass) throw new Error('Detector non trovato. Keys: ' + Object.keys(mod).join(', '))

    detector     = new DetectorClass()
    status.value = '✓ Detector pronto — avvio camera...'

    stream = await navigator.mediaDevices.getUserMedia({
      video: { width: { ideal: 1280 }, height: { ideal: 720 } }
    })
    videoRef.value.srcObject = stream
    await videoRef.value.play()

    status.value = '✓ Punta i 4 marker agli angoli della mappa'
    rafId = requestAnimationFrame(loop)
  } catch(e) {
    status.value = `✗ ${e.message}`
    console.error(e)
  }
})

onUnmounted(() => {
  cancelAnimationFrame(rafId)
  stream?.getTracks().forEach(t => t.stop())
})
</script>

<template>
  <div class="app">
    <h1>⚔️ HeroQuest AR <span>— Calibrazione 19×26</span></h1>

    <div class="viewport">
      <video ref="videoRef" playsinline muted class="hidden" />
      <canvas ref="canvasRef" />
      <div class="badge" :class="gridReady ? 'ok' : 'wait'">
        {{ gridReady ? '✓ Griglia attiva' : '⧖ Cerco 4 angoli...' }}
      </div>
    </div>

    <div class="status">
      <span>{{ status }}</span>
      <b style="color:#7cb3ff">{{ fps }} fps</b>
    </div>

    <div class="panel">
      <h2>Marker rilevati</h2>
      <div class="chips">
        <span v-if="!detectedIds.length" class="empty">Nessun marker nel campo visivo</span>
        <span v-for="id in detectedIds" :key="id" class="chip"
          :class="Object.values(CORNER_IDS).includes(id) ? 'corner' : ''">
          ID {{ id }}
          <small>{{ {0:'TL',1:'TR',2:'BR',3:'BL'}[id] ?? '' }}</small>
        </span>
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
  padding: 1rem; gap: 1rem;
}
h1 { color: #c8a84b; font-size: 1.3rem }
h1 span { color: #666; font-weight: 400; font-size: 1rem }
.viewport {
  position: relative; width: 100%; max-width: 900px;
  aspect-ratio: 16/9; background: #000;
  border: 1px solid #222; border-radius: 8px; overflow: hidden;
}
.hidden { display: none }
canvas { width: 100%; height: 100%; display: block }
.badge {
  position: absolute; top: 10px; right: 10px;
  padding: 4px 12px; border-radius: 20px;
  font-size: .8rem; font-weight: 700; font-family: monospace;
}
.badge.ok   { background: #1a3d2b; color: #00ff88 }
.badge.wait { background: #2b2200; color: #ffaa00 }
.status {
  width: 100%; max-width: 900px; background: #161616;
  border: 1px solid #222; border-radius: 6px;
  padding: .5rem 1rem; font-size: .85rem; color: #aaa;
  display: flex; justify-content: space-between;
}
.panel {
  width: 100%; max-width: 900px; background: #111;
  border: 1px solid #222; border-radius: 8px; padding: .8rem 1rem;
}
.panel h2 { font-size: .9rem; color: #c8a84b; margin-bottom: .6rem }
.chips { display: flex; flex-wrap: wrap; gap: .4rem }
.empty { color: #444; font-style: italic; font-size: .85rem }
.chip {
  background: #1a3d2b; color: #00ff88; border: 1px solid #00ff8833;
  border-radius: 6px; padding: .25rem .7rem; font-family: monospace; font-weight: 700;
}
.chip.corner { background: #3d2e00; color: #ffaa00; border-color: #ffaa0033 }
.chip small  { margin-left: .3rem; opacity: .7; font-size: .75rem }
</style>