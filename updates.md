<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# describe all fixes to be made precisely and concisely, so I can change the code myself

Based on my full read of the codebase, here are all the precise changes needed:

***

## 1. `config.py.template` — Add `camera_settings` defaults

In the `self.defaults` dict (around line where `'camera_enabled': True` is), add a new key at the same level:

```python
'camera_settings': {
    'brightness':     128,   # 0–255
    'contrast':       128,   # 0–255
    'saturation':     128,   # 0–255
    'hue':            0,     # -180–180
    'sharpness':      128,   # 0–255
    'auto_exposure':  3,     # 1 = manual, 3 = auto (V4L2)
    'exposure':       166,   # 1–10000 (only used when auto_exposure = 1)
},
```


***

## 2. `camera_stream.py` — Add `apply_settings()` method

After the `start()` method, add this new method to the `CameraStream` class:

```python
def apply_settings(self, settings):
    """Apply visual camera settings via OpenCV V4L2 properties."""
    if not self.camera or not self.camera.isOpened():
        return False
    prop_map = {
        'brightness':    cv2.CAP_PROP_BRIGHTNESS,
        'contrast':      cv2.CAP_PROP_CONTRAST,
        'saturation':    cv2.CAP_PROP_SATURATION,
        'hue':           cv2.CAP_PROP_HUE,
        'sharpness':     cv2.CAP_PROP_SHARPNESS,
        'auto_exposure': cv2.CAP_PROP_AUTO_EXPOSURE,
        'exposure':      cv2.CAP_PROP_EXPOSURE,
    }
    for key, prop in prop_map.items():
        if key in settings:
            self.camera.set(prop, float(settings[key]))
            debug_print(f"Camera {key} = {settings[key]}")
    return True
```

Also in the `start()` method, after `self.camera.set(cv2.CAP_PROP_FPS, self.fps)` (the last existing `.set()` call), add:

```python
            # Apply saved visual settings if present
            from config import config as _cfg
            saved_cam = _cfg.get('camera_settings')
            if saved_cam:
                self.apply_settings(saved_cam)
```


***

## 3. `web_server.py` — Add `/api/camera_settings` endpoint

Paste this anywhere near the other config endpoints (e.g., after `gpio_config()`):

```python
@app.route('/api/camera_settings', methods=['GET', 'POST'])
def camera_settings():
    if request.method == 'POST':
        data = request.json or {}
        config.set('camera_settings', data)
        if camera:
            camera.apply_settings(data)
        return jsonify({'success': True})
    return jsonify(config.get('camera_settings', {}))
```


***

## 4. `templates/index.html` — Add dropdown panel in Camera Preview card

**A) CSS** — Add these styles inside the `<style>` block (near the `/* CAMERA STYLES */` section):

```css
.cam-settings-toggle {
    background: none; border: 1px solid #3a3a43; border-radius: 4px;
    color: #adadb8; font-size: 0.72rem; padding: 3px 8px; cursor: pointer;
}
.cam-settings-toggle:hover { background: #2d2d35; color: #efeff1; }
.cam-settings-panel {
    display: none; margin-top: 10px; background: #111;
    border: 1px solid #2d2d35; border-radius: 6px; padding: 12px;
}
.cam-settings-panel.open { display: block; }
.cam-settings-grid {
    display: grid; grid-template-columns: 1fr 1fr; gap: 0 12px;
}
.cam-settings-grid label { margin-top: 8px; }
.cam-settings-grid label:nth-child(1),
.cam-settings-grid label:nth-child(2) { margin-top: 0; }
```

**B) HTML** — In the Camera Preview card, the existing card ends with:

```html
<p style="font-size:0.72rem; color:#6b6b78; ...">Disabling the feed stops...</p>
```

Directly after that `<p>`, add:

```html
<div style="margin-top:10px; display:flex; justify-content:flex-end;">
    <button class="cam-settings-toggle" onclick="toggleCamSettings()">&#9881; Camera Settings</button>
</div>
<div class="cam-settings-panel" id="cam-settings-panel">
    <div class="cam-settings-grid">
        <div><label>Brightness (0–255)</label><input type="number" id="cam-brightness" min="0" max="255" step="1"></div>
        <div><label>Contrast (0–255)</label><input type="number" id="cam-contrast" min="0" max="255" step="1"></div>
        <div><label>Saturation (0–255)</label><input type="number" id="cam-saturation" min="0" max="255" step="1"></div>
        <div><label>Hue (-180–180)</label><input type="number" id="cam-hue" min="-180" max="180" step="1"></div>
        <div><label>Sharpness (0–255)</label><input type="number" id="cam-sharpness" min="0" max="255" step="1"></div>
        <div><label>Exposure (manual only)</label><input type="number" id="cam-exposure" min="1" max="10000" step="1"></div>
        <div style="grid-column:1/-1">
            <label>Auto Exposure</label>
            <select id="cam-auto-exposure">
                <option value="3">Auto</option>
                <option value="1">Manual</option>
            </select>
        </div>
    </div>
    <div class="btn-row" style="margin-top:10px;">
        <button onclick="saveCamSettings()" style="font-size:0.8rem; padding:5px 12px;">&#128190; Apply</button>
        <span class="save-ok" id="cam-ok">&#10003; Applied</span>
    </div>
</div>
```

**C) JavaScript** — Add these three functions anywhere in the `<script>` block:

```javascript
function toggleCamSettings() {
    document.getElementById('cam-settings-panel').classList.toggle('open');
}

async function loadCamSettings() {
    try {
        var d = await api('/api/camera_settings');
        setVal('cam-brightness',    d.brightness    !== undefined ? d.brightness    : 128);
        setVal('cam-contrast',      d.contrast      !== undefined ? d.contrast      : 128);
        setVal('cam-saturation',    d.saturation    !== undefined ? d.saturation    : 128);
        setVal('cam-hue',           d.hue           !== undefined ? d.hue           : 0);
        setVal('cam-sharpness',     d.sharpness     !== undefined ? d.sharpness     : 128);
        setVal('cam-auto-exposure', d.auto_exposure !== undefined ? d.auto_exposure : 3);
        setVal('cam-exposure',      d.exposure      !== undefined ? d.exposure      : 166);
    } catch(e) {}
}

async function saveCamSettings() {
    var body = {
        brightness:    +document.getElementById('cam-brightness').value,
        contrast:      +document.getElementById('cam-contrast').value,
        saturation:    +document.getElementById('cam-saturation').value,
        hue:           +document.getElementById('cam-hue').value,
        sharpness:     +document.getElementById('cam-sharpness').value,
        auto_exposure: +document.getElementById('cam-auto-exposure').value,
        exposure:      +document.getElementById('cam-exposure').value,
    };
    var r = await api('/api/camera_settings', 'POST', body);
    if (r.success) { flashOk('cam-ok'); log('Camera settings applied', '#00c853'); }
    else log('Camera settings failed', '#e91916');
}
```

**D)** In the `window.addEventListener('DOMContentLoaded', ...)` block at the bottom, add `await loadCamSettings();` alongside the other `load*` calls.

***

That's everything. Four files, minimal surgical changes — no restructuring needed.

