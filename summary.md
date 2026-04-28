# Dance Player — Resumen del Proyecto

## Descripción

Dance Player es una **Progressive Web App (PWA)** de una sola página (`index.html`) diseñada para profesores y bailarines. Permite cargar audio y video local, controlar la velocidad de reproducción, marcar loops A/B, y usar un metrónomo avanzado durante los ensayos. Instalable en Android e iOS como app nativa, funciona completamente sin conexión.

**URL de producción:** `https://jurados.github.io/dance-player/`  
**Stack:** HTML + CSS + JavaScript puro (sin frameworks), Web Audio API, PWA  
**Tamaño:** ~1 675 líneas en un solo archivo `index.html`

---

## Archivos del proyecto

| Archivo | Descripción |
|---|---|
| `index.html` | Toda la app: CSS, HTML y JS en un solo archivo |
| `manifest.json` | Metadatos PWA (nombre, iconos, orientación, tema) |
| `sw.js` | Service Worker — caché offline (`dance-player-v2`) |
| `icon-192.png` | Ícono PWA 192×192 |
| `icon-512.png` | Ícono PWA 512×512 |

---

## Funcionalidades implementadas

### Reproductor de audio/video
- Carga de archivos locales (mp3, m4a, wav, ogg, flac, aac, opus, wma, mp4, mov, webm, mkv)
- `showOpenFilePicker()` en Android para abrir el gestor de archivos directamente (evita el selector de cámara/grabadora); fallback `<input>` en iOS
- Cola de reproducción con reordenado por drag & drop (⠿ handle táctil)
- Renombrado de canciones in-drawer
- Navegación prev/next con botones y con swipe rápido horizontal en la forma de onda
- Control de velocidad 50–150% con slider, input numérico y gesto vertical en la onda (3 px = 1%)
- `preservesPitch` activo para mantener el tono al cambiar velocidad
- Reseteo automático de velocidad a 100% al cargar nueva canción (sin afectar syncBPM)
- Control de volumen de la canción
- Nombre de canción visible en el header con truncado con `…`

### Video local y YouTube
- Video local importado se muestra en el panel de video (igual que YouTube)
- Audio del video se separa al elemento `<audio>` para mantener todos los controles
- Sincronización play/pause/seek/velocidad entre `<audio>` y `<video>` con handlers nombrados y flag `_hasLocalVideo`
- **Modo landscape automático**: al rotar el teléfono con video cargado, el video ocupa toda la pantalla con controles superpuestos (play/seek/tiempo/cerrar)
- Wake Lock automático al entrar en modo landscape
- YouTube embed (iframe `youtube-nocookie.com`)
- Instagram: abre externamente (Meta bloquea reproducción en iframes externos)

### Forma de onda
- Decodificación con `Web Audio API` + `decodeAudioData()`
- Barras de amplitud RMS con color único y opacidad variable
- Marcadores visuales de puntos A y B (cian / dorado)
- Región sombreada del loop activo
- Playhead en tiempo real
- Seek por tap o drag horizontal
- Gesto vertical para cambiar velocidad (3 px = 1%)
- Swipe rápido horizontal (>35% del ancho, <380 ms) → canción anterior/siguiente

### Loop A/B
- Marcar punto A y/o punto B de forma independiente
- **Loops de un solo lado**: solo A → loop A→∞; solo B → loop 0→B
- Etiqueta dinámica del botón: `A→B` / `A→∞` / `0→B` / `LOOP`
- Región en onda visible incluso con un solo punto marcado
- Loop implementado en `timeupdate` (0.15 s antes del final para A-only, sin depender del evento `ended`)
- Video sincronizado manualmente en cada salto del loop
- Modal personalizado al cambiar canción con loop activo: "Mantener / Limpiar"
- Protección ante race condition: `_loadGen` cancela `loadSong` obsoletos; `_modalResolve` cierra el modal si llega nueva carga

### Metrónomo
- Scheduler con lookahead 120 ms / tick 25 ms usando Web Audio API
- Sonidos: **Clic** (oscilador), **Palmas** (ruido blanco filtrado)
- 8 beats por compás con subdivisión visual ("y" dots)
- Acentos personalizables por beat (tap en los puntos del círculo)
- Pre-cuenta configurable: sin pre-cuenta / 1 compás / 2 compases
- Control de volumen independiente del metrónomo
- Ajuste de fase (offset) para sincronizar con la canción
- Colapsable/expandible con animación suave
- Auto-pausa al pausar el audio si **Sync BPM** está activo, auto-reinicio al dar play
- Modo espejo (fullscreen con contador de beat grande, ideal para proyectar)
- Contador hablado: `SpeechSynthesisUtterance`, voz `es-ES`, rate 2.2

### BPM
- **Detección automática**: algoritmo pulse-train correlation, 24 fases por BPM candidato, rango 60–220 BPM con preferencia 1.15× para 110–195 (rango salsa/bachata)
- Análisis del tramo central de la canción (evita intro/outro silenciosos)
- **Tap BPM** manual (hasta 8 taps, promedio de intervalos)
- **Ajuste fino post-detección**: botones ±1 / ±5 que aparecen tras aplicar el BPM detectado
- Sync BPM↔Velocidad: al mover el slider de velocidad, el BPM se recalcula proporcionalmente
- Presets de género: Bachata (126 BPM), Salsa (185 BPM)
- Historial de cambio: los ±1 buttons de BPM con hold para cambio continuo
- Atajos de teclado: ← → = ±1 BPM, ↑ ↓ = ±5% velocidad, Space = play/pause, A/B/L = loop, M = metrónomo, T = tap

### Controles Bluetooth / Media Session API
- `play` / `pause` / `stop` handlers
- `previoustrack`: si >3 s → reinicia canción; si <3 s → canción anterior
- `nexttrack`: siguiente canción
- `seekbackward` / `seekforward`: ±10 s (para auriculares que envían seek en vez de prev/next)
- `seekto`: sincronización con la barra de progreso de la pantalla de bloqueo
- `playbackState` sincronizado en todo momento

### PWA e instalación
- `manifest.json` con orientación portrait, iconos 192/512 px
- Service Worker con caché offline (`dance-player-v2`)
- Funciona sin conexión una vez cargada
- Wake Lock (`navigator.wakeLock`) para mantener pantalla activa

### UI / UX
- **Tema oscuro** (default) y **tema claro** (toggle ícono header)
- Mini-modal personalizado para confirmaciones (reemplaza `confirm()`)
- Toast no bloqueante (reemplaza `alert()`, auto-dismiss 2.6 s)
- Doble tap en el área principal → play/pause
- Header sticky con nombre de canción truncado

---

## Arquitectura del código

Todo el código reside en `index.html`. Las secciones principales, marcadas con comentarios `══`, son:

```
ESTADO            — variables globales (loopA/B, speed, queue, bpm, etc.)
WAKE LOCK         — toggleWakeLock()
MEDIA SESSION     — setupMediaSession() con todos los handlers BT
DRAWER            — openDrawer / closeDrawer / addFiles / openAudioPicker
QUEUE / CARGA     — renderQueue, jumpTo, loadSong (async, con _loadGen anti-race)
VIDEO SYNC        — _hasLocalVideo flag, _vid* named handlers, syncLocalVideo()
FORMA DE ONDA     — decodeWaveform, drawWaveform, seekFromWave, gestos touch
TRANSPORT         — togglePlay, ended/timeupdate/loadedmetadata listeners
VELOCIDAD         — setSpeed, speedSlider, speedInput
LOOP A/B          — askLoopModal, setLoopPoint, toggleLoop, clearLoop, updateWaveOverlays
MIRROR            — openMirror, closeMirror, updateMirror
METRÓNOMO CORE    — getCtx, setMetVol, setSongVol, toggleAccent, setPhaseOffset
BPM DETECTION     — detectBPM (pulse-train correlation)
SONIDOS           — SOUNDS object: click, palmas
SCHEDULER         — runScheduler, highlightBeat, highlightSub, startMetronome, restartScheduler
BPM CONTROL       — changeBPM, bpmInput listeners, hold buttons, setGenre
TAP BPM           — tapBPM, applyTapBPM
SPEECH            — toggleSpeech, BEAT_NAMES
RENAME            — renameItem
THEME             — toggleTheme
DRAG REORDER      — initDragReorder (touch drag en cola)
ATAJOS TECLADO    — keydown handlers
YOUTUBE           — toggleYT, loadVideo
SERVICE WORKER    — registro del SW
LANDSCAPE VIDEO   — openLandscape, closeLandscape, lsSeek, handleOrientation
UTILIDADES        — fmt(), escHtml()
```

---

## Variables de estado globales clave

| Variable | Tipo | Descripción |
|---|---|---|
| `queue` | `Array<{file, name}>` | Cola de canciones cargadas |
| `queueIdx` | `number` | Índice de la canción activa |
| `loopA / loopB` | `number \| null` | Tiempos del loop en segundos |
| `loopActive` | `boolean` | Si el loop está corriendo |
| `speed` | `number` | Velocidad actual 50–150 |
| `bpm` | `number` | BPM del metrónomo |
| `baseBPM` | `number` | BPM base para sync con velocidad |
| `syncBPM` | `boolean` | Si BPM se recalcula al mover velocidad |
| `metroRunning` | `boolean` | Si el metrónomo está activo |
| `waveData` | `Float32Array \| null` | Datos PCM de la forma de onda |
| `_hasLocalVideo` | `boolean` | Si el video local está cargado |
| `_loadGen` | `number` | Contador de generación para cancelar loadSong concurrentes |
| `_modalResolve` | `function \| null` | Referencia al resolver del modal de loop |
| `wakeLock` | `WakeLockSentinel \| null` | Objeto de wake lock activo |

---

## Historial de commits

| Hash | Descripción |
|---|---|
| `b6aca8e` | Fix speedFill sync, loop modal race condition, landscape wake lock; remove conga/clave |
| `0ee96e3` | Fix 4 bugs and add drag-to-reorder queue |
| `ea34996` | Fix single-sided loop, speed/BPM reset, and loop modal |
| `8922010` | Fix 5 bugs: loop prompt, metronome auto-pause, header truncation, video loop sync, speed reset |
| `b15e6be` | Improve Bluetooth headphone controls via Media Session API |
| `aa99945` | Add 4 features: fine-tune BPM, landscape video, swipe to change song, single-point loop |
| `5e47983` | Show local video files in the video panel when loaded |
| `3d8949a` | Fix loop buttons: all equal horizontal flex |
| `e3faf12` | Fix iOS file picker; add video file support |
| `da304de` | Fix showOpenFilePicker for direct file access; Instagram opens externally |
| `862ef9a` | Video panel at top, add Instagram embed support |
| `35eed8a` | Dance Player PWA — initial release |
