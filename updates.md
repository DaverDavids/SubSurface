<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# make 'twitch integration' web tab one module wide. add a module/panel to engrave the date in two places: one text box input for typing in the date, then four small input fields for x1,y1 and x2,y2 and then a button to engrave that in that specified box. Then repeat all that for the second date. this module will have a 'save' button and all input boxes will be save to the config so they stay.

Tell me the exact code changes to do this, be concise

Now I have enough context. Here are the exact changes needed:

***

## 1. `templates/index.html`

**Change `span-2` → `span-1` on the Twitch card** (line ~559):

```html
<!-- BEFORE -->
<div class="card span-2">
    <h3>&#128250; Twitch Integration</h3>

<!-- AFTER -->
<div class="card span-1">
    <h3>&#128250; Twitch Integration</h3>
```

**Add the new Date Engraving card directly after the closing `</div>` of the Twitch card** (~line 591):

```html
<!-- DATE ENGRAVING -->
<div class="card span-1">
    <h3>&#128197; Date Engraving</h3>
    <p style="font-size:0.78rem;color:#adadb8;margin-bottom:12px;">Engrave a date string into two fixed regions on the board.</p>

    <label style="font-size:0.78rem;color:#adadb8;">Date 1</label>
    <input type="text" id="date1-text" placeholder="e.g. May 2026" style="margin-bottom:6px;">
    <div style="display:grid;grid-template-columns:1fr 1fr 1fr 1fr;gap:6px;margin-bottom:8px;">
        <input type="number" id="date1-x1" placeholder="X1" step="0.1">
        <input type="number" id="date1-y1" placeholder="Y1" step="0.1">
        <input type="number" id="date1-x2" placeholder="X2" step="0.1">
        <input type="number" id="date1-y2" placeholder="Y2" step="0.1">
    </div>
    <button onclick="engraveDate(1)" style="margin-bottom:16px;">&#9889; Engrave Date 1</button>

    <label style="font-size:0.78rem;color:#adadb8;">Date 2</label>
    <input type="text" id="date2-text" placeholder="e.g. May 2026" style="margin-bottom:6px;">
    <div style="display:grid;grid-template-columns:1fr 1fr 1fr 1fr;gap:6px;margin-bottom:8px;">
        <input type="number" id="date2-x1" placeholder="X1" step="0.1">
        <input type="number" id="date2-y1" placeholder="Y1" step="0.1">
        <input type="number" id="date2-x2" placeholder="X2" step="0.1">
        <input type="number" id="date2-y2" placeholder="Y2" step="0.1">
    </div>
    <button onclick="engraveDate(2)" style="margin-bottom:16px;">&#9889; Engrave Date 2</button>

    <div class="btn-row">
        <button onclick="saveDateSettings()">&#128190; Save</button>
        <span class="save-ok" id="date-ok">&#10003; Saved</span>
    </div>
</div>
```

**Add JS functions** — place near the other `async function` blocks (e.g. after `reconnectTwitch()`):

```js
async function loadDateSettings() {
    var d = await api('/api/date_config');
    if (!d) return;
    document.getElementById('date1-text').value = d.date1_text || '';
    document.getElementById('date1-x1').value   = d.date1_x1  || '';
    document.getElementById('date1-y1').value   = d.date1_y1  || '';
    document.getElementById('date1-x2').value   = d.date1_x2  || '';
    document.getElementById('date1-y2').value   = d.date1_y2  || '';
    document.getElementById('date2-text').value = d.date2_text || '';
    document.getElementById('date2-x1').value   = d.date2_x1  || '';
    document.getElementById('date2-y1').value   = d.date2_y1  || '';
    document.getElementById('date2-x2').value   = d.date2_x2  || '';
    document.getElementById('date2-y2').value   = d.date2_y2  || '';
}
async function saveDateSettings() {
    var body = {
        date1_text: document.getElementById('date1-text').value,
        date1_x1:   parseFloat(document.getElementById('date1-x1').value) || 0,
        date1_y1:   parseFloat(document.getElementById('date1-y1').value) || 0,
        date1_x2:   parseFloat(document.getElementById('date1-x2').value) || 0,
        date1_y2:   parseFloat(document.getElementById('date1-y2').value) || 0,
        date2_text: document.getElementById('date2-text').value,
        date2_x1:   parseFloat(document.getElementById('date2-x1').value) || 0,
        date2_y1:   parseFloat(document.getElementById('date2-y1').value) || 0,
        date2_x2:   parseFloat(document.getElementById('date2-x2').value) || 0,
        date2_y2:   parseFloat(document.getElementById('date2-y2').value) || 0,
    };
    var r = await api('/api/date_config', 'POST', body);
    if (r.success) { flashOk('date-ok'); log('Date settings saved', '#00c853'); }
    else log('Date save failed: ' + r.message, '#e91916');
}
async function engraveDate(n) {
    var text = document.getElementById('date'+n+'-text').value.trim();
    if (!text) { log('Date '+n+' text is empty', '#e91916'); return; }
    var body = {
        name: text,
        source: 'manual',
        override_rect: {
            x1: parseFloat(document.getElementById('date'+n+'-x1').value) || 0,
            y1: parseFloat(document.getElementById('date'+n+'-y1').value) || 0,
            x2: parseFloat(document.getElementById('date'+n+'-x2').value) || 0,
            y2: parseFloat(document.getElementById('date'+n+'-y2').value) || 0,
        }
    };
    var r = await api('/api/engrave', 'POST', body);
    if (r.success) log('Date '+n+' queued: ' + text, '#00c853');
    else log('Date '+n+' engrave failed: ' + r.message, '#e91916');
}
```

**Call `loadDateSettings()` in the init block** (near line ~1297 where `loadTwitchSettings()` is called):

```js
await loadDateSettings();
```


***

## 2. `web_server.py`

Add two new routes near the other `twitch_config` routes:

```python
@app.route('/api/date_config', methods=['GET'])
def get_date_config():
    keys = ['date1_text','date1_x1','date1_y1','date1_x2','date1_y2',
            'date2_text','date2_x1','date2_y1','date2_x2','date2_y2']
    return jsonify({k: config.get('date_engraving.' + k, '') for k in keys})

@app.route('/api/date_config', methods=['POST'])
def set_date_config():
    data = request.get_json()
    for k, v in data.items():
        config.set('date_engraving.' + k, v)
    config.save()
    return jsonify({'success': True})
```

The `engraveDate()` JS reuses the existing `/api/engrave` endpoint with `override_rect` — no new backend route needed for the engrave action itself, since `main.py` already handles `override_rect` with full bounding-box scaling.

***

## 3. `config.py` / `config.yaml`

No structural change required — `config.set('date_engraving.date1_text', ...)` will create the `date_engraving` key block automatically on first save, as long as your `config.set()` supports dot-notation nested writes (which it does based on the existing pattern).

