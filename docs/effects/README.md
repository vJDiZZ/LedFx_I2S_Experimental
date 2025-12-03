# LedFx Effects System Documentation

This directory contains comprehensive documentation about how the LedFx effects system works.

## 📚 Documentation Files

### [Lightning Effects Explained](lightning_effects_explained.md) 
**Comprehensive Guide** - 📖 *Main Documentation*

A complete, in-depth guide covering:

- **Effect System Architecture** - How effects are structured and organized
- **Effect Base Classes** - AudioReactiveEffect, TemporalEffect, GradientEffect, HSVEffect
- **Lightning and Flash Effects** - Detailed breakdown of:
  - Random Flash Effect (random lightning bolts)
  - BPM Strobe Effect (music-synchronized strobes)
  - Glitch Effect (chaotic patterns)
  - Fire Effect (particle-based fire simulation)
- **Rendering Pipeline** - Frame-by-frame processing flow
- **Audio Processing** - How audio drives visual effects
- **Creating Custom Effects** - Step-by-step guide with complete examples
- **Advanced Techniques** - Numpy operations, waveforms, color utilities
- **Best Practices** - Performance, code style, testing

**Perfect for**: Understanding the system deeply or creating your own effects

### [Effects Quick Reference](../developer/effects_quick_reference.md)
**Developer Cheat Sheet** - ⚡ *Quick Reference*

A concise reference guide with:

- Quick lookup tables for base classes
- Code snippets for common patterns
- Audio data method reference
- Pixel manipulation examples
- Performance tips
- Debugging checklist
- Minimal effect template

**Perfect for**: Quick lookups while coding or refreshing your memory

## 🎯 Quick Start

### Understanding How Effects Work

1. **Start here**: Read the [Architecture section](lightning_effects_explained.md#effect-system-architecture) to understand the big picture
2. **Learn the basics**: Review [Effect Base Classes](lightning_effects_explained.md#effect-base-classes) 
3. **See examples**: Study the [Lightning and Flash Effects](lightning_effects_explained.md#lightning-and-flash-effects)
4. **Try it yourself**: Follow [Creating Custom Effects](lightning_effects_explained.md#creating-custom-effects)

### Creating Your First Effect

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
        # Pulse to bass
        self.pixels[:] = self.color * self.bass
```

See the [Complete Example](lightning_effects_explained.md#example-simple-lightning-effect) for a full lightning effect implementation.

## 🎨 Effect Types Explained

### Audio-Reactive Effects
Effects that respond to music in real-time by analyzing frequency bands, beats, and volume.

**Examples**: Spectrum analyzers, beat-synchronized strobes, music visualizers

**Key Features**:
- Receives audio data every frame
- Can react to bass, mids, treble
- Beat detection and tempo sync
- Volume-based modulation

### Temporal Effects  
Time-based animations that run independently on their own thread.

**Examples**: Scrolling patterns, rotating effects, pulsing lights

**Key Features**:
- Independent timing loop
- Speed control (0.1x - 10x)
- No audio required
- Smooth animations

### Gradient Effects
Effects that map values to color gradients from a palette.

**Examples**: Rainbow effects, color-mapped visualizations

**Key Features**:
- CSS-style gradient support
- Smooth color interpolation
- Gradient rolling/shifting
- Easy color theming

### HSV Effects
Effects that work in HSV color space for easier hue/saturation manipulation.

**Examples**: Rainbow cycles, saturation-based effects, brightness mapping

**Key Features**:
- Automatic RGB conversion
- Perceptual color adjustments
- Waveform utilities
- Easier color math

## 🌩️ Lightning Effects Breakdown

### Random Flash
Creates unpredictable lightning-like flashes:
- Random position on strip
- Configurable probability
- Fade-out envelope
- Size control

### BPM Strobe
Music-synchronized strobe light:
- Beat detection
- Multiple frequencies (1/1, 1/2, 1/4, 1/8, 1/16, 1/32)
- Pattern control (*.*.etc)
- Decay parameters

### Glitch
Chaotic, glitchy visual patterns:
- Complex waveform modulation
- Audio reactivity
- HSV-based rendering
- Dynamic saturation

### Fire
Realistic fire simulation:
- Particle system
- Heat diffusion
- Rising sparks
- Audio-driven intensity

## 🔧 Development Workflow

1. **Choose base class** - Audio, Temporal, Gradient, or HSV
2. **Define schema** - Configuration parameters with validation
3. **Initialize state** - Set up arrays in `on_activate()`
4. **Implement rendering** - Update `self.pixels` array
5. **Add audio reactivity** - Implement `audio_data_updated()` if needed
6. **Test thoroughly** - Various pixel counts, speeds, audio

## 📖 Additional Resources

- **Source Code**: `/ledfx/effects/` - All effect implementations
- **Audio System**: `/ledfx/effects/audio.py` - Audio processing
- **Color Utils**: `/ledfx/color.py` - Color manipulation functions
- **Architecture**: `/docs/developer/architecture.md` - System overview
- **Common Settings**: `common_settings.md` - Built-in effect settings

## 🎓 Learning Path

### Beginner
1. Read "Effect System Architecture"
2. Study one simple effect (e.g., random_flash.py)
3. Try the minimal effect template
4. Experiment with color and speed

### Intermediate
1. Learn about audio data processing
2. Study audio-reactive effects (strobe.py)
3. Implement filters for smooth responses
4. Create a simple audio-reactive effect

### Advanced
1. Master numpy optimization techniques
2. Study complex effects (fire.py, glitch.py)
3. Combine multiple base classes
4. Create sophisticated particle systems or 2D effects

## 🐛 Debugging Tips

**Enable diagnostic logging**:
```python
CONFIG_SCHEMA = vol.Schema({
    vol.Optional("diag", default=False): bool,
})

if self.logsec.diag:
    _LOGGER.info(f"Debug: {value}")
```

**Common issues**:
- Slow performance → Use vectorized numpy operations
- Flickering → Add audio filtering  
- Wrong colors → Check value ranges (0-255)
- No audio response → Verify AudioReactiveEffect inheritance

## 💡 Pro Tips

1. **Use numpy operations** instead of Python loops for 100x+ speedup
2. **Pre-allocate arrays** once in `on_activate()`, reuse them
3. **Filter audio data** for smooth, non-jittery responses
4. **Test with different music** styles and volumes
5. **Profile with** `performance_analyser.py` to find bottlenecks

## 🤝 Contributing

Found an issue or want to improve the documentation? 
- Check the [Contributing Guide](../../CODE_OF_CONDUCT.md)
- Join the [Discord](https://discord.gg/xyyHEquZKQ)
- Submit issues or PRs on GitHub

---

## Summary

This documentation provides everything you need to understand and create effects in LedFx:

- ✅ Complete architecture overview
- ✅ Detailed effect examples with code
- ✅ Step-by-step creation guide
- ✅ Quick reference for developers
- ✅ Best practices and optimization tips
- ✅ Debugging and testing guidance

**Start with** [Lightning Effects Explained](lightning_effects_explained.md) for the full guide, or jump to the [Quick Reference](../developer/effects_quick_reference.md) if you need a fast lookup.

Happy effect creation! ✨🌈
