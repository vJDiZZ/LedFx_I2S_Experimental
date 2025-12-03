# Lightning Effects and Patterns in LedFx

## Overview

LedFx is a real-time LED visualization system that transforms audio input into dynamic light shows. This document explains how the lightning effects and visual patterns work, focusing on the effect system architecture and implementation details.

## Table of Contents

1. [Effect System Architecture](#effect-system-architecture)
2. [Effect Base Classes](#effect-base-classes)
3. [Lightning and Flash Effects](#lightning-and-flash-effects)
4. [Rendering Pipeline](#rendering-pipeline)
5. [Audio Processing and Effects](#audio-processing-and-effects)
6. [Creating Custom Effects](#creating-custom-effects)

---

## Effect System Architecture

### Core Components

The LedFx effect system is built on a modular architecture with several key components:

```
┌─────────────────────────────────────────────────────────────┐
│                        Effect System                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                    │
│  │ Base Effect  │◄─────│ Audio Data   │                    │
│  │   Class      │      │  Processor   │                    │
│  └──────┬───────┘      └──────────────┘                    │
│         │                                                    │
│         ├─► AudioReactiveEffect (responds to music)        │
│         ├─► TemporalEffect (time-based animations)         │
│         ├─► GradientEffect (color gradient support)        │
│         └─► HSVEffect (HSV color space operations)         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Pixel Array (self.pixels)                │  │
│  │   RGB values for each LED: [[R,G,B], [R,G,B], ...]   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### The Effect Base Class

Located in `ledfx/effects/__init__.py`, the `Effect` base class provides:

- **Pixel Management**: The `self.pixels` numpy array stores RGB values for each LED
- **Configuration Schema**: Voluptuous-based validation for effect parameters
- **Rendering Method**: The `render()` method that effects implement to generate patterns
- **Post-processing**: Built-in transformations (blur, flip, mirror, brightness, background color)
- **Thread Safety**: Automatic locking with `self.lock` for concurrent access

```python
class Effect(BaseRegistry):
    def render(self):
        """
        To be implemented by child effect
        Must act on self.pixels, setting the values of it
        """
        pass

    def get_pixels(self):
        """
        Get the current pixels for the effect and apply transformations
        Returns: numpy.ndarray with RGB values
        """
        # Applies flip, mirror, blur, brightness, and background color
        return processed_pixels
```

---

## Effect Base Classes

### 1. AudioReactiveEffect

Audio-reactive effects respond to music in real-time by analyzing:

- **Frequency bands**: Low, mid, high frequency content
- **Beat detection**: Onset detection for rhythmic patterns
- **Volume/Power**: Overall energy in different frequency ranges
- **Tempo**: BPM detection for synchronized effects

```python
class MyAudioEffect(AudioReactiveEffect):
    def audio_data_updated(self, data):
        # Called when new audio data is available
        self.volume = data.volume()
        self.bass_power = data.lows_power()
        
    def render(self):
        # Use audio data to drive the effect
        brightness = self.bass_power
        self.pixels[:] = color * brightness
```

### 2. TemporalEffect

Temporal effects are time-based animations that run independently on a separate thread:

- **Fixed frame rate**: Configurable speed from 0.1x to 10x
- **Thread-based**: Runs asynchronously from audio processing
- **Speed control**: `effect_loop()` method called periodically

```python
class MyTemporalEffect(TemporalEffect):
    def effect_loop(self):
        # Called periodically based on speed setting
        # Update animation state
        self.position += 1
        self.pixels[self.position % self.pixel_count] = color
```

### 3. GradientEffect

Provides color gradient functionality for effects that map values to color palettes:

- **Gradient parsing**: CSS-style gradient strings
- **Color interpolation**: Smooth transitions between colors
- **Gradient rolling**: Animated color shifts

```python
class MyGradientEffect(GradientEffect):
    def render(self):
        # Map positions to gradient colors
        for i in range(self.pixel_count):
            position = i / self.pixel_count
            self.pixels[i] = self.get_gradient_color(position)
```

### 4. HSVEffect

Operates in HSV (Hue, Saturation, Value) color space for easier color manipulation:

- **HSV array**: Internal `self.hsv_array` for calculations
- **Automatic conversion**: Converts HSV to RGB automatically
- **Waveform utilities**: Built-in triangle, sin, square wave functions
- **Perceptual hue**: Optional "fix_hues" for perceptually even rainbows

```python
class MyHSVEffect(HSVEffect):
    def render_hsv(self):
        # Work in HSV space
        self.hsv_array[:, 0] = hue_values      # H: 0-1
        self.hsv_array[:, 1] = saturation      # S: 0-1
        self.hsv_array[:, 2] = brightness      # V: 0-1
        # Automatically converted to RGB
```

---

## Lightning and Flash Effects

### Random Flash Effect

**File**: `ledfx/effects/random_flash.py`

The Random Flash effect creates lightning-like flashes at random positions:

**How it works**:

1. **Probabilistic triggering**: Each frame has a chance to trigger a flash based on `hit_probability_per_sec`
2. **Random positioning**: Flash appears at a random location on the LED strip
3. **Time envelope**: Flash brightness follows a fade curve over `hit_duration`
4. **Size control**: `hit_relative_size` determines the percentage of LEDs affected

```python
class RandomFlashEffect(TemporalEffect):
    def effect_loop(self):
        # Check if flash is still active
        hit_is_still_active = time_passed < self.hit_duration
        
        if hit_is_still_active and self.last_hit_pixels is not None:
            # Fade out the flash
            self.pixels = self.last_hit_pixels * (
                1 - time_passed / self.hit_duration
            )
        else:
            # Check for new hit
            is_hit = np.random.random() < self.probability_per_sec
            if is_hit:
                # Create flash at random position
                random_pos = random.randrange(
                    self.pixel_count - hit_absolute_size + 1
                )
                self.pixels[random_pos : random_pos + hit_absolute_size] = (
                    self.hit_color
                )
```

**Configuration Parameters**:
- `hit_color`: Color of the flash (default: white)
- `hit_duration`: How long the flash lasts in seconds (0.1-5.0)
- `hit_probability_per_sec`: Chance of flash per second (0.01-1.0)
- `hit_relative_size`: Size of flash as % of strip (1-100%)

### BPM Strobe Effect

**File**: `ledfx/effects/strobe.py`

A music-synchronized strobe that flashes to the beat:

**How it works**:

1. **Beat synchronization**: Uses BPM detection to align with music tempo
2. **Frequency control**: `strobe_frequency` sets flashes per beat (1/1, 1/2, 1/4, 1/8, 1/16, 1/32)
3. **Decay curves**: Two decay parameters control flash fade and beat fade
4. **Pattern control**: `strobe_pattern` allows skipping beats (e.g., "*.*.") 

```python
class Strobe(AudioReactiveEffect, GradientEffect):
    def audio_data_updated(self, data):
        # Get beat position (0-1 within current beat)
        o = data.beat_oscillator()
        
        # Calculate strobe brightness based on position in beat
        self.strobe_brightness = (
            ((-o % (1 / self.freq)) * self.freq) ** self.strobe_decay
        ) * (1 - o) ** self.beat_decay
        
        # Apply pattern mask
        bar_idx = int(data.bar_oscillator())
        strobe_mask = int(self.strobe_pattern[bar_idx] == "*")
        self.strobe_brightness *= strobe_mask
```

**Configuration Parameters**:
- `strobe_frequency`: Flashes per beat
- `strobe_decay`: Flash fade speed (1-10, higher = faster)
- `beat_decay`: Brightness decay across beat (0-10)
- `strobe_pattern`: Pattern string ("****", "*.*.") 
- `gradient`: Color gradient for the strobe

### Glitch Effect

**File**: `ledfx/effects/glitch.py`

Creates chaotic, glitchy patterns with audio reactivity:

**How it works**:

1. **Waveform modulation**: Uses multiple time-varying waveforms
2. **Audio reactivity**: Bass power drives the speed/intensity
3. **HSV manipulation**: Creates complex color patterns in HSV space
4. **Saturation control**: `saturation_threshold` ensures vibrant colors

```python
class Glitch(AudioReactiveEffect, HSVEffect):
    def render_hsv(self):
        # Multiple time-based waveforms
        t1 = self.time(self._config["speed"] * 0.5) * np.pi * 2
        t2 = self.time(self._config["speed"] * 0.5)
        
        # Modulate hue with complex function
        h = np.copy(self.i)  # Position array
        m = 0.3 + self.triangle(t2) * 0.2
        c = self.triangle(t3) * 10 + 4 * np.sin(t4)
        
        np.multiply(h, c, out=h)
        np.mod(h, m, out=h)
        
        # Apply to HSV array
        self.hsv_array[:, 0] = h
```

### Fire Effect

**File**: `ledfx/effects/fire.py`

Simulates fire with rising sparks and heat diffusion:

**How it works**:

1. **Particle system**: Sparks rise from bottom with random velocities
2. **Heat diffusion**: Neighboring pixels blur heat values
3. **Audio reactivity**: Bass affects cooling rate and spark acceleration
4. **Color mapping**: Heat values mapped to fire colors (red-orange-yellow)

```python
class Fire(AudioReactiveEffect, HSVEffect):
    def render_hsv(self):
        # Cool down all pixels
        np.multiply(self.spark_pixels, self.cooling, out=self.spark_pixels)
        
        # Diffuse heat to neighbors
        if self.pixel_count > 5:
            pixels[5:] = (
                pixels[4:-1] + pixels[3:-2] + 
                pixels[2:-3] * 2 + pixels[1:-4] * 3
            ) / 7
        
        # Advance sparks upward
        step = sparks**2 * delta_scaled * (pixel_limit / 100)
        sparkX += step
        
        # Map heat to fire colors (HSV)
        self.hsv_array[:, 0] = heat_hue    # Red-orange-yellow
        self.hsv_array[:, 1] = saturation
        self.hsv_array[:, 2] = brightness
```

---

## Rendering Pipeline

### Frame-by-Frame Process

```
┌───────────────────────────────────────────────────────────┐
│                    Frame Rendering (~60 FPS)               │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│  1. Audio Processing                                       │
│     - FFT analysis                                         │
│     - Beat detection                                       │
│     - Frequency band extraction                            │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│  2. Effect Rendering (effect.render())                     │
│     - Audio-reactive effects receive audio_data_updated()  │
│     - Effect updates self.pixels array                     │
│     - Temporal effects run on separate thread              │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│  3. Post-Processing (effect.get_pixels())                  │
│     a. Flip (if enabled)                                   │
│     b. Mirror (if enabled)                                 │
│     c. Background color addition                           │
│     d. Brightness adjustment                               │
│     e. Blur (Gaussian convolution)                         │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│  4. Virtual-to-Device Mapping                              │
│     - Segment mapping (span/copy/grouping)                 │
│     - Pixel remapping per segment                          │
└───────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────┐
│  5. Device Output                                          │
│     - Protocol formatting (DDP, E1.31, etc.)               │
│     - Network transmission to LED controllers              │
└───────────────────────────────────────────────────────────┘
```

### The Pixel Array

The heart of every effect is the `self.pixels` numpy array:

```python
# Shape: (pixel_count, 3) for RGB
# Each row is one LED: [Red, Green, Blue]
# Values range from 0.0 to 255.0

self.pixels = np.array([
    [255.0, 0.0, 0.0],    # LED 0: Red
    [0.0, 255.0, 0.0],    # LED 1: Green
    [0.0, 0.0, 255.0],    # LED 2: Blue
    # ... more LEDs
])
```

### Blur Implementation

The blur effect uses Gaussian convolution for smooth light diffusion:

```python
def fast_blur_pixels(pixels: NDArray, sigma: float) -> NDArray:
    """
    Applies a fast blur effect using a Gaussian kernel.
    
    Args:
        pixels: Input array of pixels (N x 3)
        sigma: Standard deviation of Gaussian kernel
        
    Returns:
        Blurred pixels array
    """
    kernel = _gaussian_kernel1d(sigma, 0, len(pixels))
    pixels[:, 0] = np.convolve(pixels[:, 0], kernel, mode="same")  # R
    pixels[:, 1] = np.convolve(pixels[:, 1], kernel, mode="same")  # G
    pixels[:, 2] = np.convolve(pixels[:, 2], kernel, mode="same")  # B
    return pixels
```

---

## Audio Processing and Effects

### Audio Data Flow

```
┌──────────────────┐
│  Audio Input     │  (Microphone or Loopback)
│  Device          │
└────────┬─────────┘
         │ Raw PCM Audio
         ▼
┌──────────────────┐
│  FFT Analysis    │  (Fast Fourier Transform)
│                  │
└────────┬─────────┘
         │ Frequency Domain
         ▼
┌──────────────────┐
│  Mel Scale       │  (Perceptually uniform frequency bins)
│  Conversion      │
└────────┬─────────┘
         │ Melbank Data
         ▼
┌──────────────────┐
│  Feature         │  (Volume, beat, frequency bands)
│  Extraction      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Effect          │  Receives audio_data_updated(data)
│  Audio Update    │
└──────────────────┘
```

### Audio Data Object

Effects receive rich audio information:

```python
def audio_data_updated(self, data):
    # Volume/Power
    volume = data.volume()              # Overall volume (dB)
    lows_power = data.lows_power()      # Bass energy
    mids_power = data.mids_power()      # Mid-range energy
    highs_power = data.highs_power()    # Treble energy
    
    # Beat Detection
    beat_now = data.beat_now()          # Is beat happening?
    beat_oscillator = data.beat_oscillator()  # Position in beat (0-1)
    bar_oscillator = data.bar_oscillator()    # Bar number
    
    # Frequency Data
    melbank = data.melbank()            # Mel-scale frequency bins
    fft = data.fft()                    # Raw FFT data
```

### Filters for Smooth Audio Response

Effects often use exponential filters to smooth audio data:

```python
# Create filter
self._bass_filter = self.create_filter(
    alpha_decay=0.05,   # How fast it decays
    alpha_rise=0.99     # How fast it rises
)

# Use filter
smoothed_bass = self._bass_filter.update(raw_bass_value)
```

---

## Creating Custom Effects

### Step-by-Step Guide

#### 1. Choose Your Base Class

```python
from ledfx.effects.audio import AudioReactiveEffect
from ledfx.effects.temporal import TemporalEffect
from ledfx.effects.gradient import GradientEffect
from ledfx.effects.hsv_effect import HSVEffect

# Audio-reactive effect
class MyEffect(AudioReactiveEffect, GradientEffect):
    pass

# Or time-based effect
class MyEffect(TemporalEffect):
    pass

# Or HSV-based effect
class MyEffect(AudioReactiveEffect, HSVEffect):
    pass
```

#### 2. Define Configuration Schema

```python
import voluptuous as vol
from ledfx.color import validate_color

class MyEffect(AudioReactiveEffect):
    NAME = "My Effect"
    CATEGORY = "Atmospheric"  # or "BPM", "Classic", etc.
    
    CONFIG_SCHEMA = vol.Schema({
        vol.Optional(
            "speed",
            description="Effect speed",
            default=1.0
        ): vol.All(vol.Coerce(float), vol.Range(min=0.1, max=10.0)),
        
        vol.Optional(
            "color",
            description="Primary color",
            default="#FF0000"
        ): validate_color,
    })
```

#### 3. Implement Initialization (Optional)

```python
def on_activate(self, pixel_count):
    """Called when effect is activated"""
    # Initialize any arrays or state
    self.positions = np.zeros(pixel_count)
    self.velocities = np.random.rand(pixel_count)
```

#### 4. Implement Config Update (Optional)

```python
def config_updated(self, config):
    """Called when configuration changes"""
    self.speed = self._config["speed"]
    self.color = np.array(parse_color(self._config["color"]))
```

#### 5. Implement Audio Update (For Audio-Reactive)

```python
def audio_data_updated(self, data):
    """Called when new audio data is available"""
    self.bass = data.lows_power()
    self.beat = data.beat_now()
```

#### 6. Implement Rendering

```python
def render(self):
    """Main rendering method - update self.pixels"""
    
    # Example: Pulse effect based on bass
    brightness = self.bass
    
    # Set all pixels to color with brightness
    self.pixels[:] = self.color * brightness
    
    # Or: Individual pixel control
    for i in range(self.pixel_count):
        self.pixels[i] = [255, 0, 0]  # Red
```

#### For HSV Effects:

```python
def render_hsv(self):
    """Update self.hsv_array instead"""
    # Create rainbow
    hue = np.linspace(0, 1, self.pixel_count)
    
    self.hsv_array[:, 0] = hue           # Hue
    self.hsv_array[:, 1] = 1.0           # Saturation
    self.hsv_array[:, 2] = self.bass     # Brightness from audio
    
    # Automatic RGB conversion happens
```

#### For Temporal Effects:

```python
def effect_loop(self):
    """Called periodically based on speed setting"""
    
    # Update animation state
    self.position = (self.position + 1) % self.pixel_count
    
    # Clear pixels
    self.pixels.fill(0)
    
    # Draw moving dot
    self.pixels[self.position] = [255, 255, 255]
    
    # Return speed modifier (optional)
    return 1.0
```

### Example: Simple Lightning Effect

Here's a complete example of a custom lightning effect:

```python
import random
import numpy as np
import voluptuous as vol
from ledfx.color import parse_color, validate_color
from ledfx.effects.temporal import TemporalEffect


class SimpleLightningEffect(TemporalEffect):
    """
    Creates random lightning bolts with branches
    """
    
    NAME = "Simple Lightning"
    CATEGORY = "Atmospheric"
    
    CONFIG_SCHEMA = vol.Schema({
        vol.Optional(
            "strike_color",
            description="Lightning color",
            default="#FFFFFF"
        ): validate_color,
        
        vol.Optional(
            "strike_probability",
            description="Chance of strike per second",
            default=0.2
        ): vol.All(vol.Coerce(float), vol.Range(min=0.01, max=1.0)),
        
        vol.Optional(
            "strike_duration",
            description="How long strike lasts (seconds)",
            default=0.15
        ): vol.All(vol.Coerce(float), vol.Range(min=0.05, max=1.0)),
        
        vol.Optional(
            "branches",
            description="Number of branches",
            default=3
        ): vol.All(vol.Coerce(int), vol.Range(min=1, max=10)),
    })
    
    def on_activate(self, pixel_count):
        self.strike_active = False
        self.strike_start_time = 0
        self.strike_pixels = np.zeros((pixel_count, 3))
        
    def config_updated(self, config):
        self.strike_color = np.array(
            parse_color(self._config["strike_color"]), 
            dtype=float
        )
        self.strike_probability = self._config["strike_probability"]
        self.strike_duration = self._config["strike_duration"]
        self.num_branches = self._config["branches"]
        
    def effect_loop(self):
        if self.strike_active:
            # Calculate fade
            elapsed = self.now - self.strike_start_time
            
            if elapsed < self.strike_duration:
                # Exponential fade
                fade = (1 - elapsed / self.strike_duration) ** 2
                self.pixels = self.strike_pixels * fade
            else:
                # Strike ended
                self.strike_active = False
                self.pixels.fill(0)
        else:
            # Check for new strike
            if random.random() < self.strike_probability / 50:  # Adjusted for frame rate
                self._create_strike()
                
        return 1.0  # Speed modifier
        
    def _create_strike(self):
        """Generate a lightning bolt with branches"""
        self.strike_pixels.fill(0)
        
        # Main bolt
        start = random.randint(0, self.pixel_count - 1)
        length = random.randint(
            int(self.pixel_count * 0.3), 
            int(self.pixel_count * 0.7)
        )
        
        for i in range(length):
            pos = (start + i) % self.pixel_count
            # Brighter in the middle
            brightness = 1.0 - abs(i - length/2) / (length/2)
            self.strike_pixels[pos] = self.strike_color * brightness
            
        # Add branches
        for _ in range(self.num_branches):
            branch_start = random.randint(0, length - 1)
            branch_length = random.randint(5, length // 3)
            
            for i in range(branch_length):
                pos = (start + branch_start + i) % self.pixel_count
                brightness = (1 - i / branch_length) * 0.7  # Dimmer than main
                self.strike_pixels[pos] = np.maximum(
                    self.strike_pixels[pos],
                    self.strike_color * brightness
                )
        
        self.strike_active = True
        self.strike_start_time = self.now
```

---

## Advanced Techniques

### Numpy Operations for Performance

LedFx effects use numpy for efficient array operations:

```python
# GOOD: Vectorized operations
self.pixels[:] = color * brightness  # All at once

# BAD: Python loops (slow)
for i in range(self.pixel_count):
    self.pixels[i] = color * brightness
```

### Common Numpy Patterns

```python
# Fill all pixels with a color
self.pixels.fill(0)  # Black
self.pixels[:] = [255, 0, 0]  # All red

# Slice operations
self.pixels[10:20] = color  # Set pixels 10-19
self.pixels[::2] = color    # Every other pixel

# Array math
self.pixels *= 0.5              # Halve all values
self.pixels += background_color # Add colors
self.pixels = np.maximum(self.pixels, min_brightness)

# Smooth transitions
new_pixels = old_pixels * 0.9 + target_pixels * 0.1
```

### Waveform Generators

For cyclic patterns:

```python
# Triangle wave (0 -> 1 -> 0)
def triangle(t):
    return 1 - 2 * abs(t - 0.5)

# Sine wave (smooth oscillation)
def sine(t):
    return 0.5 * np.sin(t * 2 * np.pi) + 0.5

# Square wave (on/off)
def square(t, duty=0.5):
    return 0.5 * np.sign(duty - t) + 0.5
```

### Color Utilities

```python
from ledfx.color import parse_color, hsv_to_rgb

# Parse color from string
color = parse_color("#FF0000")  # Returns (255, 0, 0)

# HSV to RGB conversion
rgb = hsv_to_rgb(hue, saturation, value)
# Arrays work too
rgb_array = hsv_to_rgb(hue_array, sat_array, val_array)

# Mix colors
mixed = color1 * 0.7 + color2 * 0.3
```

---

## Best Practices

### Performance

1. **Use numpy operations** instead of Python loops
2. **Pre-allocate arrays** in `on_activate()`
3. **Cache expensive calculations** in `config_updated()`
4. **Avoid creating new arrays** each frame - reuse existing ones

### Code Style

1. **Follow existing patterns** in the effects directory
2. **Use descriptive parameter names** in CONFIG_SCHEMA
3. **Add clear descriptions** to configuration options
4. **Test with different pixel counts** (small and large)

### Configuration

1. **Provide sensible defaults** for all parameters
2. **Use appropriate ranges** (vol.Range) to prevent invalid values
3. **Group related settings** logically
4. **Mark advanced options** using ADVANCED_KEYS

### Audio Reactivity

1. **Use filters** for smooth audio response
2. **Test with different music** genres and volumes
3. **Provide non-audio fallback** behavior
4. **Don't make effects too sensitive** - allow parameter tuning

---

## Debugging and Testing

### Enable Diagnostic Logging

```python
# In your effect's config
vol.Optional("diag", description="Enable diagnostic logging", default=False): bool

# In render() or other methods
if self.logsec.diag:
    _LOGGER.info(f"Debug info: {some_value}")
```

### Testing Your Effect

1. **Test with different pixel counts**: 10, 100, 300 LEDs
2. **Test with different speeds**: 0.1x to 10x
3. **Test configuration changes**: Change settings while running
4. **Test audio reactivity**: Different music styles and volumes
5. **Monitor CPU usage**: Use `performance_analyser.py`

### Common Issues

| Issue | Solution |
|-------|----------|
| Effect too slow | Use vectorized numpy operations |
| Colors clipping | Ensure values stay in 0-255 range |
| Flickering | Use filters to smooth rapid changes |
| Not audio reactive | Check audio_data_updated() is called |
| Config changes ignored | Implement config_updated() properly |

---

## Summary

The LedFx effect system provides a powerful framework for creating dynamic LED visualizations:

- **Multiple base classes** for different effect types (audio, temporal, gradient, HSV)
- **Rich audio data** for music synchronization
- **Efficient numpy operations** for real-time performance
- **Flexible configuration** with schema validation
- **Post-processing pipeline** for common transformations

Lightning and flash effects demonstrate key techniques:
- **Random Flash**: Probabilistic triggers with time-based fades
- **BPM Strobe**: Beat synchronization with pattern control
- **Glitch**: Complex waveform modulation
- **Fire**: Particle systems with diffusion

By understanding these patterns, you can create your own stunning visual effects!

---

## Additional Resources

- **Source Code**: `ledfx/effects/` directory
- **Effect Examples**: See existing effects for patterns
- **Audio Data**: `ledfx/effects/audio.py` and `ledfx/effects/melbank.py`
- **Color Utilities**: `ledfx/color.py`
- **Documentation**: `docs/effects/` directory

Happy effect creation! ✨
