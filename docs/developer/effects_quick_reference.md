# Effects System Quick Reference

A concise guide for developers working with LedFx effects.

## Effect Base Classes

| Base Class | Purpose | Use When |
|------------|---------|----------|
| `Effect` | Foundation for all effects | Base class only |
| `AudioReactiveEffect` | Responds to audio input | Music visualization |
| `TemporalEffect` | Time-based animations | Independent animations |
| `GradientEffect` | Color gradient support | Color palette effects |
| `HSVEffect` | HSV color operations | Hue/saturation manipulation |

## Key Methods

### All Effects

```python
def on_activate(self, pixel_count):
    """Initialize arrays and state when effect starts"""
    pass

def config_updated(self, config):
    """Called when configuration changes"""
    pass

def render(self):
    """Main rendering - update self.pixels array"""
    pass
```

### Audio Reactive

```python
def audio_data_updated(self, data):
    """Receive audio data"""
    self.volume = data.volume()
    self.bass = data.lows_power()
    self.beat = data.beat_now()
```

### Temporal

```python
def effect_loop(self):
    """Called periodically based on speed"""
    # Update animation
    return 1.0  # Speed modifier (optional)
```

### HSV Effects

```python
def render_hsv(self):
    """Update self.hsv_array instead of self.pixels"""
    self.hsv_array[:, 0] = hue        # 0-1
    self.hsv_array[:, 1] = saturation # 0-1
    self.hsv_array[:, 2] = value      # 0-1
```

## Configuration Schema

```python
import voluptuous as vol
from ledfx.color import validate_color

CONFIG_SCHEMA = vol.Schema({
    vol.Optional(
        "speed",
        description="Speed of effect",
        default=1.0
    ): vol.All(vol.Coerce(float), vol.Range(min=0.1, max=10.0)),
    
    vol.Optional(
        "color",
        description="Primary color",
        default="#FF0000"
    ): validate_color,
    
    vol.Optional(
        "intensity",
        description="Effect intensity",
        default=5
    ): vol.All(vol.Coerce(int), vol.Range(min=1, max=10)),
})
```

## Audio Data Methods

```python
# Volume/Energy
data.volume()          # Overall volume (dB)
data.lows_power()      # Bass energy (0-1)
data.mids_power()      # Mid-range energy (0-1)
data.highs_power()     # Treble energy (0-1)

# Beat Detection
data.beat_now()        # Is beat happening? (bool)
data.beat_oscillator() # Position in beat (0-1)
data.bar_oscillator()  # Bar number (0-3)

# Frequency Data
data.melbank()         # Mel-scale frequency bins
data.fft()             # Raw FFT data
```

## Pixel Array Operations

```python
# Basic operations
self.pixels.fill(0)                    # Clear to black
self.pixels[:] = [255, 0, 0]          # All red
self.pixels[10:20] = color            # Set range

# Math operations
self.pixels *= 0.5                    # Dim by 50%
self.pixels += background             # Add color
self.pixels = np.maximum(pixels, 10)  # Set minimum

# Slicing
self.pixels[::2] = color              # Every other pixel
self.pixels[::-1]                     # Reverse order
```

## Common Patterns

### Fade In/Out

```python
fade = elapsed_time / duration
brightness = 1.0 - fade  # Fade out
# or
brightness = fade  # Fade in

self.pixels = base_color * brightness
```

### Exponential Decay

```python
# Smooth fade
fade = (1 - elapsed / duration) ** 2  # Faster at end
# or
fade = np.exp(-elapsed / tau)  # Exponential
```

### Moving Pattern

```python
def effect_loop(self):
    self.position = (self.position + self.speed) % self.pixel_count
    self.pixels.fill(0)
    self.pixels[int(self.position)] = self.color
```

### Wave Functions

```python
# Triangle (0 -> 1 -> 0)
wave = 1 - 2 * abs(t - 0.5)

# Sine (smooth)
wave = 0.5 * np.sin(t * 2 * np.pi) + 0.5

# Square (on/off)
wave = 0.5 * np.sign(0.5 - t) + 0.5
```

### Gradient Mapping

```python
class MyEffect(GradientEffect):
    def render(self):
        for i in range(self.pixel_count):
            position = i / self.pixel_count  # 0-1
            self.pixels[i] = self.get_gradient_color(position)
```

### Audio Filtering

```python
def config_updated(self, config):
    self._bass_filter = self.create_filter(
        alpha_decay=0.05,  # Decay speed
        alpha_rise=0.99    # Rise speed
    )

def audio_data_updated(self, data):
    raw_bass = data.lows_power()
    smoothed_bass = self._bass_filter.update(raw_bass)
```

## Performance Tips

1. **Use numpy operations** - avoid Python loops
2. **Pre-allocate arrays** in `on_activate()`
3. **Reuse arrays** - don't create new ones each frame
4. **Cache calculations** in `config_updated()`
5. **Profile with** `performance_analyser.py`

## Debugging

```python
# Add diagnostic logging
CONFIG_SCHEMA = vol.Schema({
    vol.Optional("diag", default=False): bool,
})

# Use in code
if self.logsec.diag:
    _LOGGER.info(f"Value: {self.some_value}")
```

## File Structure

```
ledfx/effects/
├── __init__.py          # Effect base class
├── audio.py             # AudioReactiveEffect
├── temporal.py          # TemporalEffect
├── gradient.py          # GradientEffect
├── hsv_effect.py        # HSVEffect
├── your_effect.py       # Your new effect
└── utils/               # Utilities
```

## Minimal Effect Template

```python
import numpy as np
import voluptuous as vol
from ledfx.effects.audio import AudioReactiveEffect
from ledfx.color import validate_color, parse_color


class MyEffect(AudioReactiveEffect):
    NAME = "My Effect"
    CATEGORY = "Atmospheric"
    
    CONFIG_SCHEMA = vol.Schema({
        vol.Optional("color", default="#FF0000"): validate_color,
    })
    
    def config_updated(self, config):
        self.color = np.array(parse_color(self._config["color"]))
    
    def audio_data_updated(self, data):
        self.bass = data.lows_power()
    
    def render(self):
        self.pixels[:] = self.color * self.bass
```

## Testing Checklist

- [ ] Test with 10, 100, 300 pixels
- [ ] Test speed 0.1x to 10x
- [ ] Test all config parameter ranges
- [ ] Test with different audio (genres, volumes)
- [ ] Test config changes while running
- [ ] Check CPU usage
- [ ] Enable `diag` logging and check output

## Common Issues

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| Slow performance | Python loops | Use numpy operations |
| Flickering | Rapid changes | Add audio filtering |
| Colors too bright | No range limiting | Clip to 0-255 |
| Effect not responding | audio_data_updated not called | Inherit from AudioReactiveEffect |
| Config ignored | config_updated not implemented | Add config_updated method |

## See Also

- [Lightning Effects Explained](../effects/lightning_effects_explained.md) - Comprehensive guide
- [Effect Examples](../../ledfx/effects/) - Source code directory
- [Color Utilities](../../ledfx/color.py) - Color manipulation
- [Audio Processing](../../ledfx/effects/audio.py) - Audio data

---

*For detailed information, see the [Lightning Effects Explained](../effects/lightning_effects_explained.md) guide.*
