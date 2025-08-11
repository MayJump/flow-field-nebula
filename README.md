<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover" />
  <title>Flow-Field Nebula</title>
  <style>
    :root { color-scheme: dark; }
    html, body { height: 100%; margin: 0; background: #0b0d10; }
    canvas { display:block; width:100%; height:100%; }
    .hud{
      position:fixed; inset:auto 12px 12px 12px; 
      display:flex; justify-content:space-between; pointer-events:none; font:12px/1.4 system-ui, -apple-system, Segoe UI, Roboto, sans-serif; color:#aab2c0; opacity:.9;
    }
    .hud .left{pointer-events:auto}
    .chip{
      background:#11161caa; border:1px solid #1e2a35; border-radius:10px; padding:8px 10px; margin:0 6px 0 0;
      backdrop-filter: blur(6px);
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
  density: 0.0016,        // particles per pixel (auto-scales with screen)
  step: 0.8,              // particle step size (pixels)
  lineWidth: 0.6,         // stroke width
  noiseScale: 0.0013,     // flow-field scale (smaller = wider curls)
  fieldSpeed: 0.3,        // how fast the field drifts over time
  fadeAlpha: 0.05,        // frame fade (0 = no fade, 1 = full clear)
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
      ctx.fillStyle = Math.random() < 0.5 ? "#0b0d10" : "#0c0f13";
      ctx.fillRect(x,y,1,1);
    }
  }
  ctx.restore();
}

function loop(t){
  if(!paused){
    // gentle fade for trails
    ctx.save();
    ctx.globalAlpha = SETTINGS.fadeAlpha;
    ctx.fillStyle = SETTINGS.bg;
    ctx.fillRect(0,0,W,H);
    ctx.restore();

    ctx.lineWidth = SETTINGS.lineWidth * DPR;
    ctx.lineCap = 'round';
    ctx.lineJoin = 'round';

    const z = t * 0.001 * SETTINGS.fieldSpeed;

    ctx.globalCompositeOperation = 'lighter';
    for(let p of particles){
      const ax = p.x, ay = p.y;
      const angle = valueNoise(p.x*SETTINGS.noiseScale, p.y*SETTINGS.noiseScale, z) * TAU*2;
      p.x += Math.cos(angle) * SETTINGS.step * DPR;
      p.y += Math.sin(angle) * SETTINGS.step * DPR;

      ctx.beginPath();
      ctx.moveTo(ax, ay);
      ctx.lineTo(p.x, p.y);
      ctx.strokeStyle = withAlpha(p.hue, 0.06 + 0.12*Math.random());
      ctx.stroke();

      p.life++;
      if (p.x< -10 || p.x>W+10 || p.y<-10 || p.y>H+10 || p.life>1500){
        Object.assign(p, spawnParticle());
      }
    }
    ctx.globalCompositeOperation = 'source-over';
  }
  fpsMeter(t);
  requestAnimationFrame(loop);
}

/* ---------- Value Noise (smooth, seedable) ---------- */
function valueNoise(x, y, z=0){
  const xi = Math.floor(x), yi = Math.floor(y), zi = Math.floor(z);
  const xf = x - xi, yf = y - yi, zf = z - zi;
  const u = fade(xf), v = fade(yf), w = fade(zf);
  const aaa = hash3(xi,   yi,   zi  );
  const aab = hash3(xi,   yi,   zi+1);
  const aba = hash3(xi,   yi+1, zi  );
  const abb = hash3(xi,   yi+1, zi+1);
  const baa = hash3(xi+1, yi,   zi  );
  const bab = hash3(xi+1, yi,   zi+1);
  const bba = hash3(xi+1, yi+1, zi  );
  const bbb = hash3(xi+1, yi+1, zi+1);
  const x1 = lerp(aaa, baa, u);
  const x2 = lerp(aba, bba, u);
  const y1 = lerp(x1, x2, v);
  const x3 = lerp(aab, bab, u);
  const x4 = lerp(abb, bbb, u);
  const y2 = lerp(x3, x4, v);
  return lerp(y1, y2, w); // 0..1
}
function hash3(x, y, z){
  let h = x*374761393 + y*668265263 + z*1442695040888963407 + seed;
  h = (h ^ (h >> 13)) * 1274126177;
  h ^= h >> 16;
  return (h >>> 0) / 4294967295;
}
const fade = t => t*t*t*(t*(t*6-15)+10);
const lerp = (a,b,t)=>a+(b-a)*t;

/* ---------- Utils ---------- */
function pick(arr){ return arr[(Math.random()*arr.length)|0]; }
function withAlpha(hex, a){
  const {r,g,b} = hexToRGB(hex);
  return `rgba(${r},${g},${b},${a})`;
}
function hexToRGB(hex){
  hex = hex.replace('#','');
  if(hex.length===3){ hex = hex.split('').map(c=>c+c).join(''); }
  const num = parseInt(hex,16);
  return { r:(num>>16)&255, g:(num>>8)&255, b:num&255 };
}

function resize(){
  DPR = Math.max(1, Math.min(2, window.devicePixelRatio||1));
  const w = Math.floor(window.innerWidth);
  const h = Math.floor(window.innerHeight);
  canvas.width  = Math.floor(w * DPR);
  canvas.height = Math.floor(h * DPR);
  canvas.style.width = w + "px";
  canvas.style.height = h + "px";
  ctx.setTransform(DPR,0,0,DPR,0,0);
  W = w; H = h;
}
window.addEventListener('resize', resize);

/* ---------- Keys ---------- */
window.addEventListener('keydown', (e)=>{
  if(e.key==='r' || e.key==='R'){ reseed(); }
  if(e.key==='c' || e.key==='C'){ palette = pick(SETTINGS.palettes); }
  if(e.key==='p' || e.key==='P'){ paused = !paused; }
  if(e.key==='s' || e.key==='S'){ savePNG(); }
});

function savePNG(){
  const a = document.createElement('a');
  a.download = 'flow-field-nebula.png';
  a.href = canvas.toDataURL('image/png');
  a.click();
}

/* ---------- FPS (tiny) ---------- */
let last=0, acc=0, frames=0;
function fpsMeter(t){
  const el = document.getElementById('fps');
  const dt = t-last; last=t; acc+=dt; frames++;
  if(acc>500){
    el.textContent = (1000*frames/acc).toFixed(0) + " fps • " + particles.length + " pts";
    acc=0; frames=0;
  }
}
</script>
</body>
</html>
