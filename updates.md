<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# I uploaded the changes, check the new code. when I change settings the camera feed doesn't change. I even saved and restarted the service but they have no effect

The code structure is correct. The root cause is a well-known Linux V4L2/OpenCV limitation. Here's exactly what's wrong and how to fix it:

***

## Root Cause: MJPG Codec Bypasses Kernel ISP

When you set `MJPG` format (`cv2.CAP_PROP_FOURCC`), the camera firmware compresses frames internally **before** any V4L2 controls (brightness, hue, exposure, etc.) can be applied by the kernel. OpenCV's `camera.set()` calls return `True` but the hardware ignores them because the image is already baked into the JPEG by the camera's onboard processor.

***

## Fix in `camera_stream.py`

**In `start()`, you must set `auto_exposure` BEFORE setting MJPG, and apply all other settings AFTER format+resolution are set.** Also, `CAP_PROP_AUTO_EXPOSURE` must be set first since enabling manual mode is a prerequisite for `EXPOSURE` to take effect.

Replace this block in `start()`:

```python
# OLD — wrong order
self.camera.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
self.camera.set(cv2.CAP_PROP_FRAME_WIDTH, self.width)
self.camera.set(cv2.CAP_PROP_FRAME_HEIGHT, self.height)
self.camera.set(cv2.CAP_PROP_FPS, self.fps)

from config import config as _cfg
saved_cam = _cfg.get('camera_settings')
if saved_cam:
    self.apply_settings(saved_cam)
```

With:

```python
# NEW — correct order
from config import config as _cfg
saved_cam = _cfg.get('camera_settings', {})

# 1. Auto-exposure FIRST — must be set before format negotiation
if saved_cam.get('auto_exposure') is not None:
    self.camera.set(cv2.CAP_PROP_AUTO_EXPOSURE, float(saved_cam['auto_exposure']))

# 2. Format + resolution
self.camera.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
self.camera.set(cv2.CAP_PROP_FRAME_WIDTH, self.width)
self.camera.set(cv2.CAP_PROP_FRAME_HEIGHT, self.height)
self.camera.set(cv2.CAP_PROP_FPS, self.fps)

# 3. All other visual controls after format is locked in
if saved_cam:
    self.apply_settings(saved_cam)
```


***

## Fix in `apply_settings()`

The `auto_exposure` property must always be applied **first** so that if it's switching to manual, the `exposure` value that follows actually sticks. Change `apply_settings()` to enforce this order:

```python
def apply_settings(self, settings):
    if not self.camera or not self.camera.isOpened():
        return False
    # auto_exposure must be set first — it gates whether exposure is writable
    if 'auto_exposure' in settings:
        self.camera.set(cv2.CAP_PROP_AUTO_EXPOSURE, float(settings['auto_exposure']))
        debug_print(f"Camera auto_exposure = {settings['auto_exposure']}")

    prop_map = {
        'brightness':  cv2.CAP_PROP_BRIGHTNESS,
        'contrast':    cv2.CAP_PROP_CONTRAST,
        'saturation':  cv2.CAP_PROP_SATURATION,
        'hue':         cv2.CAP_PROP_HUE,
        'sharpness':   cv2.CAP_PROP_SHARPNESS,
        'exposure':    cv2.CAP_PROP_EXPOSURE,
    }
    for key, prop in prop_map.items():
        if key in settings:
            self.camera.set(prop, float(settings[key]))
            debug_print(f"Camera {key} = {settings[key]}")
    return True
```


***

## Also: Live Apply Won't Work with MJPG on Some Cameras

Some cheap USB webcams (common on Pi setups) **freeze all V4L2 controls once MJPG streaming starts** — the only way to change settings is to stop/reopen the camera. If settings still don't apply after the above fix, add a restart to the `web_server.py` endpoint:

```python
@app.route('/api/camera_settings', methods=['GET', 'POST'])
def camera_settings():
    if request.method == 'POST':
        data = request.json or {}
        config.set('camera_settings', data)
        if camera:
            # Stop and restart so format negotiation picks up new settings
            camera.stop()
            camera.apply_settings(data)  # attempt live apply
            if not camera.is_running():
                camera.start()           # restart if it stopped
        return jsonify({'success': True})
    return jsonify(config.get('camera_settings', {}))
```

The `stop()` → `apply_settings()` → `start()` sequence forces the driver to renegotiate, which is the only reliable way to get controls to take on locked-down MJPG cameras.

