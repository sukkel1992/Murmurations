# Murmuration<script>
(() => {
  const cv = document.getElementById('cv');
  const ctx = cv.getContext('2d');
  let W = 900, H = 600;
  let paused = false;
  const hudStatus = document.getElementById('hudStatus');
  const hudFps = document.getElementById('hudFps');

  // Fullscreen helpers
  const fsBtn = document.getElementById('fsBtn');
  function isFS(){ return document.fullscreenElement || document.body.classList.contains('fs'); }
  async function enterFS(){ document.body.classList.add('fs'); try{ if (!document.fullscreenElement) await document.documentElement.requestFullscreen(); }catch(e){} fit(); }
  async function exitFS(){ document.body.classList.remove('fs'); try{ if (document.fullscreenElement) await document.exitFullscreen(); }catch(e){} fit(); }
  function toggleFS(){ isFS() ? exitFS() : enterFS(); }
  if (fsBtn) fsBtn.addEventListener('click', toggleFS);
  window.addEventListener('keydown', (e)=>{ if ((e.key||'').toLowerCase()==='f') toggleFS(); });
  cv.addEventListener('dblclick', toggleFS);
  let lastTap=0; cv.addEventListener('touchend', (e)=>{ const now=Date.now(); if (now-lastTap<300){ toggleFS(); e.preventDefault(); } lastTap=now; }, {passive:false});
  document.addEventListener('fullscreenchange', ()=>{ if (!document.fullscreenElement) document.body.classList.remove('fs'); fit(); });

  const state = {
    trail: 0.20, n: 600, maxSpeed: 2.0, viewR: 40,
    sep: 1.0, ali: 0.6, coh: 0.25, wrap: true, soft: 0.0,
    fStr: 1.4, fFall: 140, fingerMode: 'attract',
    fieldA: 'None', fieldB: 'None', blend: 0.0, fStrength: 0.9, autoMorph: false, autoMorphSpeed: 0.15,
    camEnabled: false, camStrength: 1.2, camPolarity: 'bright_attract', camRes: 128,
  };

  const bindSlider = (id, key, fmt=(v)=>v) => { 
    const el = document.getElementById(id); 
    const lab = document.getElementById(id+'Val'); 
    const set = v => { 
      state[key] = parseFloat(v); 
      if(lab) lab.textContent = (typeof fmt==='function' ? fmt(state[key]).toString() : state[key]); 
    }; 
    set(el.value); 
    el.addEventListener('input', e => set(e.target.value)); 
  };
  bindSlider('trail','trail',v=>Number(v).toFixed(2));
  bindSlider('boids','n',v=>v);
  bindSlider('maxSpeed','maxSpeed',v=>Number(v).toFixed(2));
  bindSlider('viewR','viewR');
  bindSlider('sep','sep',v=>Number(v).toFixed(2));
  bindSlider('ali','ali',v=>Number(v).toFixed(2));
  bindSlider('coh','coh',v=>Number(v).toFixed(2));
  bindSlider('soft','soft',v=>Number(v).toFixed(2));
  bindSlider('fStr','fStr',v=>Number(v).toFixed(2));
  bindSlider('fFall','fFall');
  bindSlider('blend','blend',v=>Number(v).toFixed(2));
  bindSlider('fStrength','fStrength',v=>Number(v).toFixed(2));

  const wrapChk = document.getElementById('wrapChk'); wrapChk.addEventListener('change', ()=> state.wrap = wrapChk.checked);
  const fingerMode = document.getElementById('fingerMode'); fingerMode.addEventListener('change', ()=> state.fingerMode = fingerMode.value);
  const fieldA = document.getElementById('fieldA'); fieldA.addEventListener('change', ()=> state.fieldA = fieldA.value);
  const fieldB = document.getElementById('fieldB'); fieldB.addEventListener('change', ()=> state.fieldB = fieldB.value);
  const autoMorphChk = document.getElementById('autoMorphChk'); autoMorphChk.addEventListener('change', ()=> state.autoMorph = autoMorphChk.checked);
  const autoMorphSpeed = document.getElementById('autoMorphSpeed'); autoMorphSpeed.addEventListener('input', ()=> state.autoMorphSpeed = parseFloat(autoMorphSpeed.value));

  document.getElementById('resetBtn').addEventListener('click', reset);
  document.getElementById('pulseBtn').addEventListener('click', ()=>{ const id=99999; const cx=W*0.5, cy=H*0.5; pointers.set(id,{id, x:cx, y:cy}); setTimeout(()=>pointers.delete(id),600); });
  document.getElementById('pauseBtn').addEventListener('click', (e)=>{ paused = !paused; e.target.textContent = paused ? 'Play' : 'Pause'; setStatus(paused ? 'paused' : 'running', true); });

  const camEnable = document.getElementById('camEnable');
  const camStr = document.getElementById('camStr');
  const camPolarity = document.getElementById('camPolarity');
  const camRes = document.getElementById('camRes');
  camEnable.addEventListener('change', async ()=>{ state.camEnabled = camEnable.checked; if (state.camEnabled) await startCamera(); });
  camStr.addEventListener('input', ()=> state.camStrength = parseFloat(camStr.value));
  camPolarity.addEventListener('change', ()=> state.camPolarity = camPolarity.value);
  camRes.addEventListener('change', ()=> { state.camRes = parseInt(camRes.value,10); resizeCamBuffer(); });

  const clamp = (v,a,b)=> Math.max(a, Math.min(b, v));
  const lerp = (a,b,t)=> a + (b-a)*t;
  const smoothstep = (t)=>{ t = clamp(t,0,1); return t*t*(3-2*t); };
  const norm = (x,y)=>{ const m=Math.hypot(x,y)||1; return [x/m, y/m]; };

  const FIELD_MAP = {
    'None': (x,y,t)=>[0,0],
    'Hopf Swirl': (x,y,t)=>{ const r=Math.hypot(x,y)+1e-6; const ang=Math.atan2(y,x); const w=0.6+0.9*Math.sin(0.7*t+3*r); const tx=-Math.sin(ang), ty=Math.cos(ang); const breathe=0.15*Math.sin(1.3*t+6*ang); return [ w*tx + breathe*(x/r), w*ty + breathe*(y/r) ]; },
    'Klein Loop': (x,y,t)=>{ const a=0.25*Math.sin(0.6*t); const c=Math.cos(a), s=Math.sin(a); const ux=c*x - s*y, uy=s*x + c*y; const A=0.6; const r2=ux*ux + uy*uy; const dfx=4*r2*ux - 4*A*A*ux; const dfy=4*r2*uy + 4*A*A*uy; let tx=-dfy, ty=dfx; const sgn=Math.sign(Math.sin(t+3*r2))||1; tx*=sgn; ty*=sgn; const ax=-0.12*dfx, ay=-0.12*dfy; const m=Math.hypot(tx,ty)||1; return [ tx/m + ax, ty/m + ay ]; },
    'Trefoil Knot': (x,y,t)=>{ const r=Math.hypot(x,y)+1e-6; const ang=Math.atan2(y,x); const c=0.55+0.05*Math.sin(0.8*t); const target=c*Math.cos(3*ang + 0.6*Math.sin(0.9*t)); const err=r-Math.abs(target); const ax=-0.35*err*Math.cos(ang), ay=-0.35*err*Math.sin(ang); const w=0.9+0.3*Math.sin(0.5*t); return [ ax - w*Math.sin(ang), ay + w*Math.cos(ang) ]; },
  };

  const pointers = new Map();
  const toCanvas = (clientX, clientY)=>{ const rect=cv.getBoundingClientRect(); return { x:(clientX-rect.left)*(cv.width/rect.width), y:(clientY-rect.top)*(cv.height/rect.height) }; };
  cv.addEventListener('pointerdown', e=>{ cv.setPointerCapture(e.pointerId); const p=toCanvas(e.clientX,e.clientY); pointers.set(e.pointerId,{id:e.pointerId,...p}); });
  cv.addEventListener('pointermove', e=>{ if(!pointers.has(e.pointerId)) return; const p=toCanvas(e.clientX,e.clientY); pointers.set(e.pointerId,{id:e.pointerId,...p}); });
  window.addEventListener('pointerup', e=> pointers.delete(e.pointerId));

  let boids = [];
  function reset(){ boids = new Array(state.n).fill(0).map(()=>({ x:Math.random()*W, y:Math.random()*H, vx:(Math.random()*2-1), vy:(Math.random()*2-1) })); }

  function fit(){
    const fs = document.body.classList.contains('fs');
    if (fs){ W = window.innerWidth; H = window.innerHeight; }
    else { const parent = document.querySelector('.panel'); W = Math.max(360, parent.clientWidth); H = Math.max(360, Math.floor(W*0.6)); }
    cv.width = W; cv.height = H;
  }
  window.addEventListener('resize', fit); fit(); reset();

  let last = performance.now(); let blendT = state.blend; let fpsEma=0, fpsAlpha=0.1, fpsTimer=0;

  function ndc(x,y){ return [ (x/W)*2-1, (y/H)*2-1 ]; }
  function setStatus(text, ok){ if(!hudStatus) return; hudStatus.textContent=text; hudStatus.classList.remove('ok','err'); hudStatus.classList.add(ok?'ok':'err'); }
  function updateFps(dt){ const fps = 1/Math.max(1e-6, dt); fpsEma = fpsAlpha*fps + (1-fpsAlpha)*fpsEma; fpsTimer += dt; if (fpsTimer >= 0.25 && hudFps){ hudFps.textContent = Math.round(fpsEma).toString(); fpsTimer = 0; } }

  // High-performance camera pipeline
  const video = document.getElementById('cam');
  let camStream = null;
  const camCanvas = document.createElement('canvas');
  const camCtx = camCanvas.getContext('2d', { willReadFrequently: true });
  let camReady = false;

  async function startCamera(){
    try{
      camStream = await navigator.mediaDevices.getUserMedia({ video: { facingMode:'environment' }, audio:false });
      video.srcObject = camStream; await video.play(); camReady = true; resizeCamBuffer(); setStatus('camera on', true);
    }catch(err){ console.error(err); setStatus('camera denied', false); document.getElementById('camEnable').checked = false; state.camEnabled = false; camReady=false; }
  }
  function resizeCamBuffer(){ if(!video.videoWidth) return; const s = state.camRes; const aspect = video.videoWidth / video.videoHeight; camCanvas.width = s; camCanvas.height = Math.round(s / aspect); }

  function topologyForce(x,y,t){
    const n = ndc(x,y); const fA = FIELD_MAP[state.fieldA](n[0],n[1],t); const fB = FIELD_MAP[state.fieldB](n[0],n[1],t+0.3);
    const sT = smoothstep(blendT);
    return [ lerp(fA[0],fB[0],sT)*state.fStrength, lerp(fA[1],fB[1],sT)*state.fStrength ];
  }

  function step(now){
    const dt = Math.min(0.05, (now-last)/1000); last = now; updateFps(dt);
    if (boids.length !== state.n) reset();
    if (state.autoMorph){ blendT += state.autoMorphSpeed*dt; if (blendT>=1) blendT=0; } else { blendT = state.blend; }

    const vr2 = state.viewR*state.viewR; const ms = state.maxSpeed; const time = now/1000;

    // Capture camera frame once per animation frame
    let camPixels = null, camW = 0, camH = 0;
    if (state.camEnabled && camReady && camCanvas.width > 0){
      camW = camCanvas.width; camH = camCanvas.height;
      camCtx.drawImage(video, 0, 0, camW, camH);
      camPixels = camCtx.getImageData(0, 0, camW, camH).data;
    }

    if (!paused){
      const sgnCam = (state.camPolarity === 'bright_attract') ? 1 : -1;
      for (let i=0; i<boids.length; i++){
        const b = boids[i];
        let sepX=0, sepY=0, sepN=0, aliX=0, aliY=0, aliN=0, cohX=0, cohY=0, cohN=0;

        for (let j=0; j<boids.length; j++){
          if (i === j) continue;
          const o = boids[j];
          let dx = o.x - b.x, dy = o.y - b.y;

          // Shortest toroidal path
          if (state.wrap){
            if (dx > W * 0.5) dx -= W;
            else if (dx < -W * 0.5) dx += W;
            if (dy > H * 0.5) dy -= H;
            else if (dy < -H * 0.5) dy += H;
          }

          const d2 = dx*dx + dy*dy;
          if (d2 > vr2) continue;
          const inv = 1 / (d2 + 1e-4);
          sepX -= dx * inv; sepY -= dy * inv; sepN++;
          aliX += o.vx; aliY += o.vy; aliN++;
          cohX += (b.x + dx); cohY += (b.y + dy); cohN++;
        }

        let ax=0, ay=0;
        if (sepN>0){ const n=norm(sepX,sepY); ax += state.sep*n[0]; ay += state.sep*n[1]; }
        if (aliN>0){ const n=norm(aliX,aliY); ax += state.ali*n[0]; ay += state.ali*n[1]; }
        if (cohN>0){ const cx=(cohX/cohN)-b.x, cy=(cohY/cohN)-b.y; const n=norm(cx,cy); ax += state.coh*n[0]; ay += state.coh*n[1]; }

        // Touch / pointer interaction
        pointers.forEach(p=>{
          let dx = p.x - b.x, dy = p.y - b.y;
          if (state.wrap){
            if (dx > W * 0.5) dx -= W; else if (dx < -W * 0.5) dx += W;
            if (dy > H * 0.5) dy -= H; else if (dy < -H * 0.5) dy += H;
          }
          const d = Math.hypot(dx,dy) + 1e-6;
          if (d < state.fFall){
            const fall = (1 - d/state.fFall);
            const base = state.fStr * (0.4 + 0.6*fall) / (1 + 0.5*d/state.fFall);
            const sgn = state.fingerMode === 'attract' ? 1 : -1;
            ax += sgn * base * (dx/d); ay += sgn * base * (dy/d);
          }
        });

        // Topology force
        const tf = topologyForce(b.x, b.y, time);
        ax += tf[0]; ay += tf[1];

        // Direct memory read for camera gradient
        if (camPixels){
          const ix = clamp(Math.floor((b.x / W) * camW), 1, camW - 2);
          const iy = clamp(Math.floor((b.y / H) * camH), 1, camH - 2);
          const getLum = (px, py) => {
            const idx = (py * camW + px) * 4;
            return 0.2126 * camPixels[idx] + 0.7152 * camPixels[idx+1] + 0.0722 * camPixels[idx+2];
          };
          const Lx = getLum(ix + 1, iy) - getLum(ix - 1, iy);
          const Ly = getLum(ix, iy + 1) - getLum(ix, iy - 1);
          const m = Math.hypot(Lx, Ly) || 1;
          ax += sgnCam * (Lx / m) * state.camStrength;
          ay += sgnCam * (Ly / m) * state.camStrength;
        }

        // Integration
        b.vx += ax * dt * 3.0; b.vy += ay * dt * 3.0;
        const sp = Math.hypot(b.vx, b.vy) || 1;
        if (sp > ms){ b.vx *= ms/sp; b.vy *= ms/sp; }
        b.x += b.vx * dt * 60; b.y += b.vy * dt * 60;

        if (state.wrap){
          if (b.x < 0) b.x += W; else if (b.x >= W) b.x -= W;
          if (b.y < 0) b.y += H; else if (b.y >= H) b.y -= H;
        } else {
          b.x = clamp(b.x, 0, W); b.y = clamp(b.y, 0, H);
        }
      }
    }

    draw();
    setStatus(paused ? 'paused' : (state.camEnabled && camReady ? 'running (cam)' : 'running'), true);
    requestAnimationFrame(step);
  }

  function draw(){
    const a = clamp(state.trail, 0, 1);
    if (a >= 0.999) { ctx.clearRect(0, 0, W, H); }
    else { ctx.fillStyle = `rgba(2,6,23,${a})`; ctx.fillRect(0, 0, W, H); }

    ctx.fillStyle = 'rgba(226,232,240,0.98)';
    for (let i=0; i<boids.length; i++){
      const b = boids[i];
      const ang = Math.atan2(b.vy, b.vx);
      const cos = Math.cos(ang), sin = Math.sin(ang);
      const size = 3.3;

      // Draw oriented triangles without nested save/restore overhead
      ctx.beginPath();
      ctx.moveTo(b.x + size * cos, b.y + size * sin);
      ctx.lineTo(b.x - size * 0.9 * cos - size * 0.62 * sin, b.y - size * 0.9 * sin + size * 0.62 * cos);
      ctx.lineTo(b.x - size * 0.9 * cos + size * 0.62 * sin, b.y - size * 0.9 * sin - size * 0.62 * cos);
      ctx.closePath();
      ctx.fill();
    }
  }

  requestAnimationFrame(step);
  window.addEventListener('beforeunload', ()=> { try{ if(camStream) camStream.getTracks().forEach(t=>t.stop()); }catch(e){} });
})();
</script>