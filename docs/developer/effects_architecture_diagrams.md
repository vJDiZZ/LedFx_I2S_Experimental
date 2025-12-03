# LedFx Effect System Visual Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         LedFx Core System                            │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                   │
                ▼                   ▼                   ▼
┌──────────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
│   Audio Input        │ │   Config         │ │   API/WebSocket      │
│   ─────────         │ │   ──────         │ │   ──────────────     │
│  • Microphone        │ │  • Devices       │ │  • REST Endpoints    │
│  • System Loopback   │ │  • Virtuals      │ │  • Real-time Events  │
│  • Web Audio         │ │  • Effects       │ │  • Web UI            │
└──────────┬───────────┘ └────────┬─────────┘ └──────────────────────┘
           │                      │
           │   Audio Data         │ Configuration
           ▼                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Audio Processing Engine                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │     FFT      │─▶│  Mel Scale   │─▶│   Feature    │             │
│  │   Analysis   │  │  Conversion  │  │  Extraction  │             │
│  └──────────────┘  └──────────────┘  └──────┬───────┘             │
│                                              │                       │
│  Audio Features:                             │                       │
│  • Volume/Power (dB)                         │                       │
│  • Frequency Bands (Low/Mid/High)            │                       │
│  • Beat Detection                            │                       │
│  • Tempo (BPM)                               │                       │
└──────────────────────────────────────────────┼───────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Virtual LED Strips                           │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Virtual 1: Living Room (300 LEDs)                          │    │
│  │  ├─ Effect: Fire                                            │    │
│  │  ├─ Active: Yes                                             │    │
│  │  └─ Segments: [Device1: 0-149, Device2: 150-299]           │    │
│  └────────────────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Virtual 2: Bedroom (150 LEDs)                              │    │
│  │  ├─ Effect: BPM Strobe                                      │    │
│  │  ├─ Active: Yes                                             │    │
│  │  └─ Segments: [Device3: 0-149]                             │    │
│  └────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          Effect Rendering                            │
│                                                                       │
│  For each Virtual:                                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ 1. Get Effect Instance                                      │    │
│  │    └─ Effect has: self.pixels (N×3 RGB array)              │    │
│  │                                                             │    │
│  │ 2. Update Audio Data (if AudioReactiveEffect)              │    │
│  │    └─ effect.audio_data_updated(data)                      │    │
│  │                                                             │    │
│  │ 3. Render Effect                                            │    │
│  │    └─ effect.render() → updates self.pixels                │    │
│  │                                                             │    │
│  │ 4. Apply Post-Processing                                    │    │
│  │    └─ Flip, Mirror, Blur, Brightness, Background           │    │
│  │                                                             │    │
│  │ 5. Map to Segments                                          │    │
│  │    └─ Split pixels across multiple devices/segments        │    │
│  └────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Device Output                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  WLED Device │  │  E1.31 DMX   │  │  Other       │             │
│  │  (DDP/UDP)   │  │  (sACN)      │  │  Protocols   │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                  │                      │
└─────────┼─────────────────┼──────────────────┼──────────────────────┘
          │                 │                  │
          ▼                 ▼                  ▼
    ┌─────────┐       ┌─────────┐       ┌─────────┐
    │ ESP8266 │       │ DMX Box │       │ Arduino │
    │  LEDs   │       │  LEDs   │       │  LEDs   │
    └─────────┘       └─────────┘       └─────────┘
```

## Effect Class Hierarchy

```
Effect (Base Class)
│
├─ AudioReactiveEffect
│  │  • Receives audio_data_updated(data)
│  │  • Access to volume, beats, frequency data
│  │
│  ├─ Fire (AudioReactiveEffect + HSVEffect)
│  ├─ Glitch (AudioReactiveEffect + HSVEffect)
│  ├─ Strobe (AudioReactiveEffect + GradientEffect)
│  └─ Many more...
│
├─ TemporalEffect
│  │  • Runs on separate thread
│  │  • effect_loop() called periodically
│  │  • Speed controlled by config
│  │
│  ├─ RandomFlash (TemporalEffect)
│  └─ Other time-based effects...
│
├─ GradientEffect
│  │  • get_gradient_color(position)
│  │  • Gradient rolling
│  │  • CSS-style gradient parsing
│  │
│  └─ Effects using color gradients...
│
└─ HSVEffect (extends GradientEffect)
   │  • render_hsv() instead of render()
   │  • self.hsv_array for calculations
   │  • Automatic RGB conversion
   │  • Waveform utilities
   │
   └─ Effects working in HSV space...
```

## Effect Rendering Pipeline (Per Frame)

```
┌─────────────────────────────────────────────────────────────┐
│ Start Frame (~60 FPS)                                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Audio Processing                                          │
│    ┌──────────────────────────────────────────────────┐    │
│    │ • Capture audio buffer                            │    │
│    │ • Perform FFT                                      │    │
│    │ • Calculate mel-scale bins                        │    │
│    │ • Detect beats                                     │    │
│    │ • Extract frequency bands                         │    │
│    └──────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │ Audio Data Object
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Effect Updates (if AudioReactive)                        │
│    ┌──────────────────────────────────────────────────┐    │
│    │ effect.audio_data_updated(data)                   │    │
│    │ • Store audio features                            │    │
│    │ • Update filters                                  │    │
│    │ • Calculate modulation values                     │    │
│    └──────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Effect Rendering                                          │
│    ┌──────────────────────────────────────────────────┐    │
│    │ effect.render()                                    │    │
│    │ • Update self.pixels array                        │    │
│    │ • Apply effect algorithm                          │    │
│    │ • Calculate colors and positions                  │    │
│    │                                                    │    │
│    │ For HSV effects:                                   │    │
│    │   • render_hsv() updates self.hsv_array          │    │
│    │   • Automatic conversion to RGB                   │    │
│    └──────────────────────────────────────────────────┘    │
│                                                              │
│    self.pixels shape: (N, 3) where N = pixel_count         │
│    Example: [[255, 0, 0], [0, 255, 0], [0, 0, 255], ...]   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Post-Processing (get_pixels())                           │
│    ┌──────────────────────────────────────────────────┐    │
│    │ a. Flip (if enabled)                              │    │
│    │    pixels = np.flipud(pixels)                     │    │
│    │                                                    │    │
│    │ b. Mirror (if enabled)                            │    │
│    │    pixels = mirror(pixels)                        │    │
│    │                                                    │    │
│    │ c. Background Color                               │    │
│    │    pixels += background_color                     │    │
│    │                                                    │    │
│    │ d. Brightness                                      │    │
│    │    pixels *= brightness                           │    │
│    │                                                    │    │
│    │ e. Blur (Gaussian convolution)                    │    │
│    │    pixels = convolve(pixels, gaussian_kernel)     │    │
│    └──────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. Segment Mapping                                           │
│    ┌──────────────────────────────────────────────────┐    │
│    │ Virtual → Physical Device Mapping                │    │
│    │                                                    │    │
│    │ Span Mode:                                         │    │
│    │   Effect spans across all segments               │    │
│    │   Device 1 gets pixels[0:150]                    │    │
│    │   Device 2 gets pixels[150:300]                  │    │
│    │                                                    │    │
│    │ Copy Mode:                                         │    │
│    │   Effect copied to each segment                   │    │
│    │   Device 1 gets pixels[0:150]                    │    │
│    │   Device 2 gets pixels[0:150] (same)             │    │
│    └──────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. Device Transmission                                       │
│    ┌──────────────────────────────────────────────────┐    │
│    │ For each device:                                  │    │
│    │   • Format as protocol (DDP, E1.31, etc.)        │    │
│    │   • Send over network (UDP/TCP)                  │    │
│    │   • Device receives and displays                 │    │
│    └──────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
                     ┌─────────────┐
                     │  LEDs Lit!  │
                     └─────────────┘
```

## Lightning Effect Data Flow Example

### Random Flash Effect

```
Time: t=0.0s
┌────────────────────────────────────────┐
│ 1. Check Probability                    │
│    random() < probability_per_sec       │
│    Result: True → Trigger Flash!       │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 2. Generate Flash Position              │
│    random_pos = random(0, pixel_count) │
│    Result: pos=150                      │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 3. Set Flash Pixels                     │
│    size = pixel_count * 10% = 30       │
│    pixels[150:180] = WHITE             │
│    last_hit_time = now                 │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ Time: t=0.1s                            │
│ 4. Fade Calculation                     │
│    elapsed = 0.1s                       │
│    fade = 1 - (0.1 / 0.5) = 0.8       │
│    pixels = last_pixels * 0.8          │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ Time: t=0.5s                            │
│ 5. Flash Complete                       │
│    elapsed >= duration                  │
│    pixels = zeros (black)               │
│    Wait for next flash...               │
└────────────────────────────────────────┘
```

### BPM Strobe Effect

```
Audio Input: Music at 120 BPM
┌────────────────────────────────────────┐
│ 1. Beat Detection                       │
│    BPM: 120                             │
│    beat_oscillator: 0.0 → 1.0         │
│    (position within current beat)      │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 2. Strobe Timing Calculation            │
│    frequency: 1/4 (4 strobes per beat) │
│    position in sub-beat: 0.0 → 0.25   │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 3. Brightness Calculation               │
│    brightness = (sub_beat)^decay       │
│    At beat start: brightness = 1.0     │
│    Mid sub-beat: brightness = 0.3      │
│    Beat end: brightness *= (1-o)^2     │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 4. Pattern Application                  │
│    pattern = "*.*.""                    │
│    bar_idx = 0 → pattern[0] = "*"      │
│    Apply flash (not skipped)            │
└────────────────┬───────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────┐
│ 5. Color Selection                      │
│    color = gradient_color(bar/4)       │
│    pixels[:] = color * brightness      │
└────────────────────────────────────────┘
```

## Pixel Array Manipulation Examples

### Direct Assignment
```
Operation: Set all pixels to red

Before:  [[0,0,0], [0,0,0], [0,0,0], ...]
Code:    self.pixels[:] = [255, 0, 0]
After:   [[255,0,0], [255,0,0], [255,0,0], ...]
```

### Slice Operations
```
Operation: Set middle 50% to blue

Before:  [[255,0,0], [255,0,0], [255,0,0], [255,0,0]]
Code:    mid = len(self.pixels) // 2
         quarter = mid // 2
         self.pixels[quarter:quarter+mid] = [0,0,255]
After:   [[255,0,0], [0,0,255], [0,0,255], [255,0,0]]
```

### Mathematical Operations
```
Operation: Fade to 50%

Before:  [[200,100,50], [150,75,25], [100,50,0]]
Code:    self.pixels *= 0.5
After:   [[100,50,25], [75,37.5,12.5], [50,25,0]]
```

### Vectorized Assignment
```
Operation: Gradient from red to blue

Code:    positions = np.linspace(0, 1, self.pixel_count)
         red = (1 - positions) * 255
         blue = positions * 255
         self.pixels[:, 0] = red   # R channel
         self.pixels[:, 1] = 0     # G channel
         self.pixels[:, 2] = blue  # B channel

Result:  [[255,0,0], [200,0,55], [150,0,105], ..., [0,0,255]]
```

## Memory Layout

### Effect Instance Memory
```
Effect Object
├─ self.pixels: ndarray(N, 3)      [Main pixel buffer]
│  └─ Memory: N × 3 × 8 bytes (float64)
│
├─ self.hsv_array: ndarray(N, 3)   [For HSV effects]
│  └─ Memory: N × 3 × 8 bytes (float64)
│
├─ self._config: dict              [Configuration]
├─ self.lock: threading.Lock       [Thread safety]
├─ self._virtual: Virtual          [Parent virtual]
└─ Effect-specific state...

Example for 300 LEDs:
  pixels: 300 × 3 × 8 = 7.2 KB
  hsv_array: 300 × 3 × 8 = 7.2 KB
  Total: ~15 KB + overhead
```

## Performance Characteristics

### Good Practices (Fast)
```python
# Vectorized numpy operations
self.pixels[:] = color * brightness          # O(N) - Fast

# Slicing
self.pixels[10:20] = color                   # O(10) - Fast

# In-place operations
np.multiply(self.pixels, 0.5, out=self.pixels)  # Fast

# Pre-allocated arrays
self.temp = np.zeros((self.pixel_count, 3)) # Once in on_activate()
```

### Bad Practices (Slow)
```python
# Python loops
for i in range(self.pixel_count):            # O(N) - Slow!
    self.pixels[i] = color * brightness

# Creating new arrays each frame
temp = np.zeros((self.pixel_count, 3))      # Slow!
```

### Complexity Comparison
| Operation | Time Complexity | Speed |
|-----------|----------------|-------|
| Vectorized assignment | O(N) | ⚡⚡⚡ Very Fast |
| NumPy math operations | O(N) | ⚡⚡ Fast |
| Convolution (blur) | O(N×K) | ⚡ Medium |
| Python loops | O(N) | 🐌 Slow |
| Nested Python loops | O(N²) | 🐌🐌 Very Slow |

---

## Summary

This architecture enables:
- ✅ Real-time audio visualization (60+ FPS)
- ✅ Multiple independent LED strips/devices
- ✅ Flexible effect system with inheritance
- ✅ Network-based LED control
- ✅ Efficient numpy-based rendering
- ✅ Thread-safe concurrent access
- ✅ Modular and extensible design

The key to performance is:
1. Vectorized numpy operations
2. Pre-allocated buffers
3. In-place modifications
4. Minimal Python loops
