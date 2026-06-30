<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# v4l controls did work... I changed brightness and it took effect immediately. but changing controls with the web app still doesn't work.

Jun 30 12:22:24 twitchlaser python3[93380]: [DEBUG] Set camera_settings = {'brightness': 2, 'contrast': 128, 'saturation': 128, 'hue': 50, 'sharpness': 128, 'auto_exposure': 3, 'exposure': 128}

rawdr@twitchlaser:~/TwitchLaser \$ v4l2-ctl --device=/dev/video0 --set-ctrl=brightness=1
rawdr@twitchlaser:~/TwitchLaser \$ v4l2-ctl --device=/dev/video0 --list-ctrls

User Controls

                     brightness 0x00980900 (int)    : min=0 max=255 step=1 default=128 value=1
                       contrast 0x00980901 (int)    : min=0 max=255 step=1 default=45 value=45
                     saturation 0x00980902 (int)    : min=0 max=100 step=1 default=70 value=70
                            hue 0x00980903 (int)    : min=-180 max=180 step=1 default=0 value=30
        white_balance_automatic 0x0098090c (bool)   : default=1 value=0
                          gamma 0x00980910 (int)    : min=90 max=150 step=1 default=120 value=120
                           gain 0x00980913 (int)    : min=0 max=128 step=1 default=75 value=75
           power_line_frequency 0x00980918 (menu)   : min=0 max=2 default=1 value=1 (50 Hz)
      white_balance_temperature 0x0098091a (int)    : min=2800 max=6500 step=10 default=4600 value=4000
                      sharpness 0x0098091b (int)    : min=0 max=7 step=1 default=6 value=6
         backlight_compensation 0x0098091c (int)    : min=0 max=2 step=1 default=0 value=0
    Camera Controls

                  auto_exposure 0x009a0901 (menu)   : min=0 max=3 default=3 value=3 (Aperture Priority Mode)
         exposure_time_absolute 0x009a0902 (int)    : min=2 max=1250 step=1 default=156 value=156 flags=inactive
     exposure_dynamic_framerate 0x009a0903 (bool)   : default=0 value=1
    rawdr@twitchlaser:~/TwitchLaser \$

Found it. Two problems:

**Problem 1 — `camera.stop()` kills the thread but `apply_settings()` is called on a dead/released camera object.**  The `stop()` calls `self.camera.release()`, so the `VideoCapture` object is gone. Then `apply_settings()` runs but `self.camera.isOpened()` returns `False` → it bails immediately. Then `camera.start()` reopens it — but `apply_settings()` already failed silently before the restart.

**Problem 2 — The `v4l2-ctl` command uses `self.camera_index` as the device number, but the camera hasn't restarted yet when `apply_settings()` is called**, so the device path is fine but the fix doesn't matter because of problem 1.

***

## The Fix

The endpoint logic order is wrong. Change `camera_settings()` in `web_server.py` to apply settings **after** restart:

```python
@app.route('/api/camera_settings', methods=['GET', 'POST'])
def camera_settings():
    if request.method == 'POST':
        data = request.json or {}
        config.set('camera_settings', data)
        if camera:
            camera.stop()
            camera.start()          # reopen the device first
            camera.apply_settings(data)  # NOW apply — device is live again
        return jsonify({'success': True})
    return jsonify(config.get('camera_settings', {}))
```

And since `v4l2-ctl` doesn't need the camera to be open at all (it talks directly to the kernel driver), you can also simplify `apply_settings()` to not check `self.camera.isOpened()` as a gate — or just remove that guard entirely since `v4l2-ctl` is independent of the OpenCV handle:

```python
def apply_settings(self, settings):
    """Apply visual camera settings via v4l2-ctl."""
    import subprocess

    v4l2_map = {
        'brightness':  'brightness',
        'contrast':    'contrast',
        'saturation':  'saturation',
        'hue':         'hue',
        'sharpness':   'sharpness',
        'auto_exposure': 'auto_exposure',
        'exposure':    'exposure_time_absolute',
    }

    device = f"/dev/video{self.camera_index}"
    controls = []

    if 'auto_exposure' in settings:
        controls.append(f"auto_exposure={int(settings['auto_exposure'])}")

    for key, v4l2_name in v4l2_map.items():
        if key == 'auto_exposure':
            continue
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
        debug_print("v4l2-ctl not found — sudo apt install v4l-utils")
        return False
    except Exception as e:
        debug_print(f"v4l2-ctl exception: {e}")
        return False
```

Also from the `--list-ctrls` output — **update the UI slider ranges** to match your actual camera caps:


| Setting | Current UI | Actual range |
| :-- | :-- | :-- |
| Saturation | 0–255 | **0–100** |
| Sharpness | 0–255 | **0–7** |
| Exposure | 1–10000 | **2–1250** |
| Contrast | 0–255 | 0–255 ✓ |
| Hue | -180–180 | -180–180 ✓ |

Fix those `min`/`max` values in the HTML inputs or users will set values the camera silently clamps/ignores.

