
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>Flow-Field Nebula</title>
  <style>
    :root { color-scheme: dark; }

    /* Clean slate + full-viewport */
    html, body { height:100%; margin:0; background:#0b0d10; overflow:hidden; }

    /* Hide anything the theme/README might render */
    body > :not(canvas):not(.hud) { display:none !important; }

    /* Pin canvas to the viewport */
    canvas#scene { position:fixed; inset:0; width:100vw; height:100vh; display:block; }

    .hud{
      position:fixed; left:12px; right:12px; bottom:12px;
      display:flex; justify-content:space-between;
      pointer-events:none; font:12px/1.4 system-ui,-apple-system,Segoe UI,Roboto,sans-serif;
      color:#aab2c0; opacity:.9; z-index:1;
    }
    .hud .left{pointer-events:auto}
    .chip{
      background:#11161caa; border:1px solid #1e2a35; border-radius:10px;
      padding:8px 10px; margin:0 6px 0 0; backdrop-filter:blur(6px);
    }
  </style>
</head>
<body>
  <canvas id="scene"></canvas>
  <div class="hud">
    <div class="left">
      <span class="chip">R = reseed</span>
      <span class="chip">C = new palette</span>
      <span class="chip">P = pause</span>
      <span class="chip">S = save PNG</span>
    </div>
    <div class="chip" id="fps">…</div>
  </div>

<script>
/* ========= SETTINGS ========= */
const SETTINGS = {
  density: 0.0016,        // particles per pixel
  step: 0.8,              // particle step size (px)
  lineWidth: 0.6,         // stroke width
  noiseScale: 0.0013,     // flow-field scale (smaller = wider curls)
  fieldSpeed: 0.3,        // field drift speed
  fadeAlpha: 0.05,        // trail fade (0..1)
  bg: "#0b0d10",
  palettes: [
    ["#A7C7E7","#6EB5FF","#6B9AC4","#E9F1FF","#C1FFD7"],
    ["#FF6B6B","#FFD93D","#6BCB77","#4D96FF","#B983FF"],
    ["#00C2FF","#00FFC6","#7CFFCB","#F7F7FF","#BDB2FF"],
    ["#F94144","#F3722C","#F8961E","#90BE6D","#577590"],
    ["#D1E8E2","#9AD1D4","#6EC5E9","#C490D1","#845EC2"]
  ]
};
/* ============================ */

const TAU = Math.PI*2;
let seed = (Math.random()*1e9)|0;
let paused = false;

const canvas = document.getElementById('scene');
const ctx = canvas.getContext('2d', { alpha: false });
let DPR = Math.max(1, Math.min(2, window.devicePixelRatio||1));
let W=0, H=0, particles=[], palette = pick(SETTINGS.palettes);
resize();
init();

function init(){
  reseed();
  loop(0);
}

function reseed(){
  seed = (Math.random()*1e9)|0;
  palette = pick(SETTINGS.palettes);
  const count = Math.max(200, Math.floor(W*H*SETTINGS.density));
  particles = new Array(count).fill(0).map(()=>spawnParticle());
  clear(true);
}

function spawnParticle(){
  return { x: Math.random()*W, y: Math.random()*H, hue: pick(palette), life: 0 };
}

function clear(hard=false){
  ctx.save();
  ctx.globalAlpha = 1;
  ctx.fillStyle = SETTINGS.bg;
  ctx.fillRect(0,0,W,H);
  if(!hard){
    // subtle grain
    const n = 4000;
    ctx.globalAlpha = 0.04;
    for(let i=0;i<n;i++){
      const x = Math.random()*W, y=Math.random()*H;
      ctx
