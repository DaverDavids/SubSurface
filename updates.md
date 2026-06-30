
## The Real Fix — Replace `apply_settings()` with `v4l2-ctl`

Replace the entire `apply_settings()` method with this:

```python
def apply_settings(self, settings):
    """Apply visual camera settings via v4l2-ctl (bypasses broken OpenCV property mapping)."""
    import subprocess

    # Map our setting keys to v4l2-ctl control names
    v4l2_map = {
        'brightness':    'brightness',
        'contrast':      'contrast',
        'saturation':    'saturation',
        'hue':           'hue',
        'sharpness':     'sharpness',
        'auto_exposure': 'auto_exposure',   # 1=manual, 3=aperture priority (auto)
        'exposure':      'exposure_time_absolute',
    }

    device = f"/dev/video{self.camera_index}"
    controls = []

    # auto_exposure must be set first to unlock exposure control
    if 'auto_exposure' in settings:
        controls.insert(0, f"auto_exposure={int(settings['auto_exposure'])}")

    for key, v4l2_name in v4l2_map.items():
        if key == 'auto_exposure':
            continue  # already handled above
        if key in settings:
            controls.append(f"{v4l2_name}={int(settings[key])}")

    if not controls:
        return True

    cmd = ['v4l2-ctl', f'--device={device}', '--set-ctrl=' + ','.join(controls)]
    debug_print(f"v4l2-ctl: {' '.join(cmd)}")

    try:
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=3)
        if result.returncode != 0:
            debug_print(f"v4l2-ctl error: {result.stderr.strip()}")
            return False
        return True
    except FileNotFoundError:
        debug_print("v4l2-ctl not found — install with: sudo apt install v4l-utils")
        return False
    except Exception as e:
        debug_print(f"v4l2-ctl exception: {e}")
        return False
```

