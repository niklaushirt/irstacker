# NICKS IR STACKER

A browser-based Impulse Response (IR) stacking tool for audio engineers and producers. Load multiple IRs, align them in phase, and export a single blended result.

## Features

- **Multi-Slot Loading**: Load up to 8 IR files via drag-and-drop or file picker
- **Individual Controls**: Per-slot volume, pan (L/R), delay (sample-accurate), and polarity flip
- **Phase Coherence Matrix**: Visual heatmap showing cross-correlation between all loaded IRs
- **Auto Phase Alignment**: Automatically finds optimal delay offset using cross-correlation
- **Mixed Waveform Display**: Real-time visualization of the stacked result (first 10%)
- **WAV Export**: Render and download the mixed IR as a 16-bit WAV file with timestamp

## How to Use

1. **Load IRs**: Drag and drop WAV/MP3/OGG/AIFF/FLAC files into the slot drop zones, or click to browse
2. **Adjust Controls**: Fine-tune volume, pan, and delay for each slot
3. **Auto Align**: Click "AUTO ALIGN" on a slot to automatically find its optimal phase alignment against the first loaded IR
4. **Check Phase Coherence**: Review the heatmap to see how well your IRs are aligned
5. **Export**: Click "EXPORT WAV" to download the mixed result

## Technical Details

- **Audio Engine**: Web Audio API with 48kHz sample rate
- **Phase Detection**: Normalized cross-correlation with ±256 sample search window
- **Export Format**: 16-bit stereo WAV, timestamped filename (`ir-stacked-YYYYMMDD_HHMMSS.wav`)
- **Polyphony**: Manual left/right gain routing for precise panning without StereoPannerNode

## Browser Compatibility

Works in all modern browsers (Chrome, Firefox, Safari, Edge). Safari requires explicit user gesture to start audio playback.

## License

MIT
