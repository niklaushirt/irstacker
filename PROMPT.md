# IR Stacker Build Prompt

Build a browser-based IR (Impulse Response) stacker application as a single self-contained HTML file with embedded CSS and JavaScript. No build tools, no frameworks — pure vanilla HTML/CSS/JS using the Web Audio API.

## Core Requirements

### 1. IR Loader & Playback Engine
- Use Web Audio API with AudioContext at 48kHz sample rate
- Support WAV, MP3, OGG, AIFF, FLAC file formats
- Auto-convert stereo to mono (average L+R)
- Drag-and-drop OR click-to-browse file loading
- Decode audio files into Float32Array buffers

### 2. Multi-Slot Stacking (up to 8 slots)
- Each slot: load IR, volume (0-1), pan (-1 left to +1 right), delay (sample-accurate, ±256 samples), polarity flip
- Per-slot controls rendered as a grid card with:
  - File name display with drop zone (dashed border when empty, solid border when loaded)
  - Canvas waveform thumbnail (last 512 samples of the loaded IR, with delay offset applied)
  - Three horizontal sliders: volume, pan, delay
  - Polarity toggle button (▲/▼ visual indicator)
  - "AUTO ALIGN" button
  - Remove slot button (×) when more than 1 slot exists
- "Add Slot" button in header

### 3. Phase Coherence Matrix
- Heatmap grid showing cross-correlation between all loaded IR pairs
- X-axis and Y-axis labels: "S1", "S2", etc.
- Diagonal always shows "1.00" (green background)
- Color scale: blue (-1.0) → white (0.0) → red (+1.0)
- Auto-update when slots change or auto-align runs

### 4. Auto Phase Alignment
- Per-slot button that finds optimal delay offset using normalized cross-correlation
- Compare against first loaded slot (reference)
- Search window: ±256 samples
- Apply found lag as slot.delaySamples
- Update delay slider and value display
- Show status message: "Slot N: aligned (lag: X samples, Y ms, r=Z)"

### 5. Mixed Waveform Display
- Full-width canvas above the phase coherence matrix
- Sum all loaded IRs with volume, delay, polarity applied
- Show only the FIRST 10% of the mixed result
- Green waveform line with dashed center line
- Status text: "N slot(s) mixed · first 10% · X samples · Y ms"
- Auto-update on load/remove/auto-align

### 6. Export Functionality
- "EXPORT WAV" button in master output section
- Use OfflineAudioContext to render mixed result
- Manual left/right gain routing (NOT StereoPannerNode) to avoid Safari RangeError
- 16-bit stereo WAV format
- Filename with timestamp: `ir-stacked-YYYYMMDD_HHMMSS.wav`
- Master volume control (0-100%) in export section

### 7. Playback
- Play button (▶/⏸) in export section
- Stop on timeout based on total mixed length
- Resume AudioContext if suspended

## Technical Implementation Details

### Cross-Correlation Algorithm
```javascript
function computeCrossCorrelation(a, b) {
  const len = Math.min(a.length, b.length);
  let maxCorr = -Infinity;
  let maxLag = 0;
  const halfMax = Math.min(256, len - 1);
  
  let sumA = 0, sumB = 0, sumAA = 0, sumBB = 0;
  for (let i = 0; i < len; i++) {
    sumA += a[i]; sumB += b[i];
    sumAA += a[i] * a[i]; sumBB += b[i] * b[i];
  }
  const meanA = sumA / len, meanB = sumB / len;
  
  for (let lag = -halfMax; lag <= halfMax; lag++) {
    let corr = 0;
    const start = lag >= 0 ? 0 : -lag;
    const end = lag >= 0 ? len - lag : len;
    for (let i = start; i < end; i++) {
      corr += (a[i] - meanA) * (b[i + lag] - meanB);
    }
    const denom = Math.sqrt(sumAA * sumBB);
    if (denom > 0) corr /= denom;
    if (corr > maxCorr) {
      maxCorr = corr;
      maxLag = lag;
    }
  }
  return { correlation: maxCorr, lag: maxLag };
}
```

### WAV Export (Critical Fix for Safari)
```javascript
function bufferToWav(buffer) {
  const numCh = buffer.numberOfChannels;
  const sr = buffer.sampleRate;
  const length = buffer.length;
  
  // Extract and interleave channels manually
  const data = new Float32Array(length * numCh);
  for (let ch = 0; ch < numCh; ch++) {
    const chData = buffer.getChannelData(ch);
    for (let i = 0; i < length; i++) {
      data[ch * length + i] = chData[i];
    }
  }
  
  // Convert to 16-bit PCM
  const bytes = new Int16Array(data.length);
  for (let i = 0; i < data.length; i++) {
    const s = Math.max(-1, Math.min(1, data[i]));
    bytes[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
  }
  
  // Build single ArrayBuffer with header + PCM data
  const bufferBytes = bytes.length * 2;
  const totalBytes = 44 + bufferBytes;
  const ab = new ArrayBuffer(totalBytes);
  const view = new DataView(ab);
  
  // Write WAV header
  const writeStr = (o, s) => {
    for (let i = 0; i < s.length; i++) view.setUint8(o + i, s.charCodeAt(i));
  };
  
  writeStr(0, 'RIFF');
  view.setUint32(4, 36 + bufferBytes, true);
  writeStr(8, 'WAVE');
  writeStr(12, 'fmt ');
  view.setUint32(16, 16, true);
  view.setUint16(20, 1, true);  // PCM
  view.setUint16(22, numCh, true);
  view.setUint32(24, sr, true);
  view.setUint32(28, sr * numCh * 2, true);
  view.setUint16(32, numCh * 2, true);
  view.setUint16(34, 16, true);
  writeStr(36, 'data');
  view.setUint32(40, bufferBytes, true);
  
  // Copy PCM data into the buffer
  const uint8 = new Uint8Array(ab);
  const bytesView = new Uint8Array(bytes.buffer);
  for (let i = 0; i < bytesView.length; i++) uint8[44 + i] = bytesView[i];
  
  return uint8;
}
```

### Panning Implementation (Avoid StereoPannerNode)
```javascript
// DO NOT use StereoPannerNode — causes RangeError in Safari
// Instead, use manual left/right gain nodes
const leftGain = ctx.createGain();
const rightGain = ctx.createGain();
const pan = slot.pan;
leftGain.gain.value = 1 - Math.max(-pan, 0);  // Left bias when pan < 0
rightGain.gain.value = 1 - Math.max(pan, 0);   // Right bias when pan > 0
src.connect(leftGain);
src.connect(rightGain);
leftGain.connect(masterGain);
rightGain.connect(masterGain);
```

## UI Design

- **Color Scheme**: Dark audio-plugin aesthetic
  - Background: #1a1a2e
  - Surface: #16213e
  - Accent: #e94560 (red/pink)
  - Secondary accent: #533483 (purple)
  - Success: #4ecca3 (teal/green)
  - Text: #eaeaea
  - Text dim: #8892a4
  - Borders: #2a3a5c

- **Layout**:
  - Header with "NICKS IR STACKER" title (gradient text) and "Add Slot" button
  - Slots grid (responsive, auto-fill columns)
  - Mixed Waveform section (full-width canvas)
  - Phase Coherence Matrix section (full-width canvas)
  - Master Output section (volume slider, play button, export button)
  - Status bar (teal background, auto-hide after 4 seconds)

- **Typography**: Monospace font family (SF Mono, Cascadia Code, Consolas)
- **Controls**: Custom styled range inputs with red knobs
- **Buttons**: Rounded corners, hover effects, primary/success variants

## Edge Cases to Handle

1. Empty slots (no IR loaded) — show drop zone placeholder
2. Single slot — hide remove button
3. Zero-length buffers — guard against division by zero
4. Safari AudioContext suspension — resume on user gesture
5. OfflineAudioContext with zero samples — validate maxLen > 0
6. Delay samples exceeding buffer length — clamp to buffer start

## Browser Compatibility

- Chrome, Firefox, Safari, Edge
- Safari requires: manual gain routing (no StereoPannerNode), manual buffer copying (no copyFromChannel with subarray)
- All modern browsers with Web Audio API support

## File Structure

Single file: `ir-stacker.html`
- `<style>` block with all CSS
- `<script>` block with all JavaScript
- No external dependencies

## Initial State

- 4 empty slots on page load
- Phase heatmap shows "Load at least 2 IRs" message
- Mixed waveform shows "Load IRs to see the mixed waveform" message
