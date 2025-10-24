# Kaushal25376.github.io
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Spider Silk Microscope — Bio-inspired Thread Simulator</title>
  <style>
    :root{--bg:#0f1724;--panel:#0b1220;--accent:#7dd3fc;--muted:#94a3b8;--glass: rgba(255,255,255,0.03)}
    html,body{height:100%;margin:0;font-family:Inter,ui-sans-serif,system-ui,Segoe UI,Roboto,'Helvetica Neue',Arial}
    body{background:linear-gradient(180deg,var(--bg),#071028);color:#e6eef8;display:flex;gap:16px;padding:18px;box-sizing:border-box}
    .panel{background:var(--panel);border-radius:12px;padding:12px;box-shadow:0 6px 18px rgba(2,6,23,0.6);min-width:320px}
    h1{font-size:18px;margin:0 0 8px}
    label{display:block;font-size:12px;color:var(--muted);margin-top:8px}
    input[type=range]{width:100%}
    .row{display:flex;gap:8px;align-items:center}
    .controls{width:360px}
    canvas{background:linear-gradient(180deg,#071025 0%, #08122b 100%);border-radius:8px;display:block}
    button{background:transparent;border:1px solid rgba(125,211,252,0.12);padding:8px 12px;border-radius:8px;color:var(--accent);cursor:pointer}
    .btn-primary{background:linear-gradient(90deg,#0369a1,#0ea5e9);border:none;color:#042331}
    .small{font-size:12px;color:var(--muted)}
    .status{margin-top:8px;font-size:13px}
    .hash{font-family:monospace;font-size:11px;color:#9ae6ff;background:rgba(125,211,252,0.06);padding:6px;border-radius:6px;display:inline-block}
    .footer{font-size:12px;color:var(--muted);margin-top:10px}
    .flex{display:flex;gap:8px;align-items:center}
    .spacer{flex:1}
    select,input[type=number]{background:var(--glass);border:1px solid rgba(255,255,255,0.04);padding:6px;border-radius:6px;color:inherit}
  </style>
</head>
<body>
  <div class="panel controls">
    <h1>Spider Silk Microscope — Simulator</h1>
    <div class="small">Bio‑inspired microscopic threads. Adjust simple parameters and generate reproducible micrographs.</div>

    <label>Thread density <span class="small">(<span id="denVal">120</span>)</span></label>
    <input id="density" type="range" min="10" max="600" value="120" />

    <label>Thread thickness <span class="small">(<span id="thVal">1.8</span> px)</span></label>
    <input id="thickness" type="range" min="0.2" max="6" step="0.1" value="1.8" />

    <label>Thread length <span class="small">(<span id="lenVal">0.7</span>)</span></label>
    <input id="length" type="range" min="0.2" max="1.6" step="0.01" value="0.7" />

    <label>Waviness <span class="small">(<span id="wavVal">0.6</span>)</span></label>
    <input id="waviness" type="range" min="0" max="2" step="0.01" value="0.6" />

    <label>Curvature bias <span class="small">(<span id="curVal">0.0</span>)</span></label>
    <input id="curvature" type="range" min="-1" max="1" step="0.01" value="0" />

    <label class="small">Thread color</label>
    <div class="row"><input id="color" type="color" value="#cfeaf8" /><div class="spacer"></div>
      <button id="downloadBtn">Download PNG</button>
    </div>

    <div style="margin-top:12px" class="flex">
      <button id="generateBtn" class="btn-primary">Generate</button>
      <button id="resimBtn">Re-simulate (same seed)</button>
      <button id="clearBtn">Clear</button>
      <div class="spacer"></div>
    </div>

    <div class="status" id="status">Seed: <span class="hash" id="seedHash">—</span></div>
    <div class="status" id="uniqueStatus">Uniqueness: <span id="uniq">—</span></div>
    <div class="footer">Stored unique images in localStorage. To reset, open DevTools → Application → Local Storage and delete <code>spider_silk_hashes</code>.</div>
  </div>

  <div class="panel" style="flex:1">
    <canvas id="view" width="1024" height="768"></canvas>
  </div>

<script>
// --- Utilities: seeded RNG (mulberry32) ---
function mulberry32(a) {
  return function() {
    a |= 0; a = a + 0x6D2B79F5 | 0;
    var t = Math.imul(a ^ a >>> 15, 1 | a);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  }
}

// Simple hash (hex) of ArrayBuffer using SubtleCrypto
async function hashBuffer(buf) {
  const h = await crypto.subtle.digest('SHA-256', buf);
  return Array.from(new Uint8Array(h)).map(b => b.toString(16).padStart(2,'0')).join('');
}

// Save & check uniqueness in localStorage
async function checkAndStoreHash(hex) {
  const key = 'spider_silk_hashes_v1';
  const stored = JSON.parse(localStorage.getItem(key) || '[]');
  const found = stored.includes(hex);
  if (!found) {
    stored.push(hex);
    // keep at most 200 records
    if (stored.length > 200) stored.shift();
    localStorage.setItem(key, JSON.stringify(stored));
  }
  return {found, storedCount: stored.length};
}

// --- Drawing ---
const canvas = document.getElementById('view');
const ctx = canvas.getContext('2d');

function clear() {
  ctx.clearRect(0,0,canvas.width,canvas.height);
  // subtle background texture
  ctx.fillStyle = 'rgba(6,14,28,1)';
  ctx.fillRect(0,0,canvas.width,canvas.height);
}

function drawThreads(params, seed) {
  const rnd = mulberry32(seed);
  clear();

  // add slight vignette and noise
  const g = ctx.createLinearGradient(0,0,0,canvas.height);
  g.addColorStop(0,'rgba(2,10,18,0.05)');
  g.addColorStop(1,'rgba(0,0,0,0.35)');
  ctx.fillStyle = g;
  ctx.fillRect(0,0,canvas.width,canvas.height);

  const density = params.density; // number of threads
  const thickness = params.thickness;
  const lengthFactor = params.lengthFactor;
  const waviness = params.waviness;
  const curvature = params.curvature;
  const color = params.color;

  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';

  for (let i=0;i<density;i++){
    const x = rnd()*canvas.width;
    const y = rnd()*canvas.height;

    // direction mostly vertical or horizontal randomly biased
    const verticalBias = 0.5 + curvature*0.5; // -1..1 -> 0..1 range
    const angle = (rnd() * 2 - 1) * Math.PI * 0.5 * (1 - verticalBias) + (verticalBias * (Math.PI/2));

    const len = (0.05 + lengthFactor * (0.3 + rnd()*0.7)) * Math.min(canvas.width, canvas.height);

    // thread path via points with small random offsets
    const points = [];
    const segments = 8 + Math.floor(rnd()*8);
    for (let s=0;s<=segments;s++){
      const t = s/segments;
      // base position along primary direction
      const bx = x + Math.cos(angle) * len * t;
      const by = y + Math.sin(angle) * len * t;
      // waviness amplitude scales with t*(1-t)
      const amp = waviness * (5 + 15 * (1 - Math.abs(0.5 - t))) * (1 + rnd()*0.6);
      const nx = bx + (rnd()*2-1) * amp;
      const ny = by + (rnd()*2-1) * amp;
      points.push({x:nx,y:ny});
    }

    // draw with variable width to mimic silk (thin ends thicker middle)
    ctx.beginPath();
    ctx.moveTo(points[0].x, points[0].y);
    for (let p=1;p<points.length-1;p++){
      const cp = points[p];
      const np = points[p+1];
      const midx = (cp.x + np.x)/2;
      const midy = (cp.y + np.y)/2;
      ctx.quadraticCurveTo(cp.x, cp.y, midx, midy);
    }
    ctx.strokeStyle = color;
    const baseWidth = thickness * (0.6 + 0.8 * Math.sin(i*0.1 + seed%50));
    ctx.lineWidth = baseWidth;
    ctx.stroke();

    // add micro-reflections: thin white overlays
    if (rnd() > 0.6) {
      ctx.beginPath();
      ctx.moveTo(points[0].x, points[0].y);
      for (let p=1;p<points.length-1;p++){
        const cp = points[p];
        const np = points[p+1];
        const midx = (cp.x + np.x)/2;
        const midy = (cp.y + np.y)/2;
        ctx.quadraticCurveTo(cp.x + (rnd()-0.5)*1.5, cp.y + (rnd()-0.5)*1.5, midx, midy);
      }
      ctx.strokeStyle = 'rgba(255,255,255,0.06)';
      ctx.lineWidth = Math.max(0.2, baseWidth*0.25);
      ctx.stroke();
    }
  }

  // subtle grain
  const img = ctx.getImageData(0,0,canvas.width,canvas.height);
  for (let i=0;i<img.data.length;i+=4){
    const add = (Math.random()-0.5) * 8;
    img.data[i] = Math.max(0, Math.min(255, img.data[i] + add));
    img.data[i+1] = Math.max(0, Math.min(255, img.data[i+1] + add));
    img.data[i+2] = Math.max(0, Math.min(255, img.data[i+2] + add));
  }
  ctx.putImageData(img,0,0);
}

// --- UI wiring ---
const densityEl = document.getElementById('density');
const thicknessEl = document.getElementById('thickness');
const lengthEl = document.getElementById('length');
const wavinessEl = document.getElementById('waviness');
const curvatureEl = document.getElementById('curvature');
const colorEl = document.getElementById('color');
const seedHashEl = document.getElementById('seedHash');
const uniqEl = document.getElementById('uniq');

const denVal = document.getElementById('denVal');
const thVal = document.getElementById('thVal');
const lenVal = document.getElementById('lenVal');
const wavVal = document.getElementById('wavVal');
const curVal = document.getElementById('curVal');

densityEl.oninput = () => denVal.textContent = densityEl.value;
thicknessEl.oninput = () => thVal.textContent = thicknessEl.value;
lengthEl.oninput = () => lenVal.textContent = lengthEl.value;
wavinessEl.oninput = () => wavVal.textContent = wavinessEl.value;
curvatureEl.oninput = () => curVal.textContent = curvatureEl.value;

let lastSeed = null;

async function generate(newSeed=null, store=true){
  // if newSeed is null -> create new random seed
  const seed = newSeed !== null ? newSeed : Math.floor(Math.random() * 2**31);
  lastSeed = seed;
  seedHashEl.textContent = seed.toString();

  const params = {
    density: parseInt(densityEl.value,10),
    thickness: parseFloat(thicknessEl.value),
    lengthFactor: parseFloat(lengthEl.value),
    waviness: parseFloat(wavinessEl.value),
    curvature: parseFloat(curvatureEl.value),
    color: colorEl.value
  };

  drawThreads(params, seed);

  // compute uniqueness via SHA-256 of PNG bytes
  const blob = await new Promise(res => canvas.toBlob(res, 'image/png'));
  const arr = await blob.arrayBuffer();
  const h = await hashBuffer(arr);
  const {found, storedCount} = await checkAndStoreHash(h);
  uniqEl.textContent = found ? 'DUPLICATE (seen before)' : 'UNIQUE — new!';
  uniqEl.style.color = found ? '#fca5a5' : '#9ef6ff';
  if (!found && !store) {
    // if we didn't want to store, remove last pushed
    const key = 'spider_silk_hashes_v1';
    const stored = JSON.parse(localStorage.getItem(key) || '[]');
    stored.pop();
    localStorage.setItem(key, JSON.stringify(stored));
  }
}

// initial render
generate();

// Buttons
document.getElementById('generateBtn').addEventListener('click', ()=> generate(null,true));
// re-simulate: reproduce exactly lastSeed
document.getElementById('resimBtn').addEventListener('click', ()=> {
  if (lastSeed === null) return alert('No previous seed — press Generate first');
  generate(lastSeed,true);
});

// clear canvas
document.getElementById('clearBtn').addEventListener('click', ()=>{ clear(); seedHashEl.textContent = '—'; uniqEl.textContent='—'; lastSeed = null; });

// download
document.getElementById('downloadBtn').addEventListener('click', ()=>{
  const a = document.createElement('a');
  a.href = canvas.toDataURL('image/png');
  a.download = `spider_silk_${Date.now()}.png`;
  a.click();
});

// keyboard: G generate, R re-simulate
window.addEventListener('keydown', (e)=>{
  if (e.key === 'g' || e.key === 'G') document.getElementById('generateBtn').click();
  if (e.key === 'r' || e.key === 'R') document.getElementById('resimBtn').click();
});

</script>
</body>
</html>
