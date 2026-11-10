





<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1">
<title>مانیتور کامپیوتر سه‌بعدی</title>
<style>
  :root{
    --ink:#0f141b; --panel:#18202b; --line:#2a3544; --text:#e8edf3; --muted:#9aa7b6; --accent:#ffb454;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  html,body{height:100%;background:var(--ink);color:var(--text);
    font-family:Tahoma,"Vazirmatn","Segoe UI",Arial,sans-serif;overflow:hidden}
  .app{display:flex;flex-direction:column;height:100%}
  header{padding:14px 18px 4px;flex:none}
  header h1{font-size:20px;font-weight:700}
  header p{font-size:13px;color:var(--muted);margin-top:3px}
  #stage{flex:1;min-height:0;position:relative;touch-action:none;cursor:grab}
  #stage:active{cursor:grabbing}
  #stage canvas{display:block;width:100%;height:100%}
  #hint{position:absolute;inset-inline:0;bottom:8px;text-align:center;font-size:12px;color:var(--muted);pointer-events:none;transition:opacity .6s}
  .sheet{flex:none;background:var(--panel);border-top:1px solid var(--line);padding:12px 16px 10px;max-height:44vh;overflow:auto}
  .chips{display:flex;gap:8px;overflow-x:auto;padding-bottom:10px;scrollbar-width:none}
  .chips::-webkit-scrollbar{display:none}
  .chip{flex:none;border:1px solid var(--line);background:transparent;color:var(--text);
    font:inherit;font-size:13px;padding:7px 13px;border-radius:999px;cursor:pointer}
  .chip:focus-visible{outline:2px solid var(--accent);outline-offset:2px}
  .chip[aria-pressed="true"]{background:var(--accent);border-color:var(--accent);color:#1a1205;font-weight:700}
  #title{font-size:16px;font-weight:700;margin-bottom:4px}
  #desc{font-size:14px;line-height:1.9;color:#cfd8e3;max-width:70ch}
  footer{margin-top:10px;padding-top:9px;border-top:1px solid var(--line);font-size:13px;color:var(--accent);text-align:center;font-weight:700}
  #fallback{display:none;padding:30px;text-align:center;color:var(--muted)}
</style>
</head>
<body>
<div class="app">
  <header>
    <h1>مانیتور کامپیوتر، سه‌بعدی</h1>
    <p>با انگشت یا ماوس بچرخانید. روی هر بخش بزنید تا توضیحش را ببینید.</p>
  </header>

  <div id="stage">
    <div id="fallback">بارگذاری کتابخانهٔ سه‌بعدی ممکن نشد. اتصال اینترنت را بررسی کنید.</div>
    <div id="hint">بکشید تا بچرخد</div>
  </div>

  <section class="sheet" aria-live="polite">
    <div class="chips" id="chips"></div>
    <div id="title"></div>
    <p id="desc"></p>
    <footer>ساخته شده توسط محمد متین ارم تن</footer>
  </section>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
(function(){
  const stage = document.getElementById('stage');
  if(typeof THREE === 'undefined'){ document.getElementById('fallback').style.display='block'; return; }

  /* ---------- Parts & explanations ---------- */
  const INFO = {
    screen:{ label:'صفحه نمایش', yaw:0,
      text:'صفحهٔ نمایش (پنل) تصویر را می‌سازد. از میلیون‌ها نقطهٔ کوچک به نام پیکسل تشکیل شده و هر پیکسل از سه رنگ قرمز، سبز و آبی ساخته می‌شود. در مانیتورهای LCD یک نور پس‌زمینه (Backlight) از پشت پنل را روشن می‌کند و کریستال‌های مایع تعیین می‌کنند چه مقدار نور از هر پیکسل عبور کند.'},
    bezel:{ label:'قاب جلو', yaw:0,
      text:'قاب (Bezel) پنل و قطعات الکترونیکی را نگه می‌دارد و از آن‌ها محافظت می‌کند. در مانیتورهای جدید قاب را باریک می‌سازند تا سطح نمایش بزرگ‌تر به نظر برسد.'},
    led:{ label:'دکمه و چراغ روشن', yaw:0.25,
      text:'دکمهٔ روشن/خاموش و چراغ وضعیت. معمولاً چراغ سفید یا سبز یعنی مانیتور تصویر دارد و چراغ نارنجی یا چشمک‌زن یعنی در حالت آماده‌باش (Standby) است و سیگنالی دریافت نمی‌کند.'},
    neck:{ label:'پایه (گردن)', yaw:0.5,
      text:'گردن مانیتور وزن صفحه را تحمل می‌کند. در بسیاری از مدل‌ها مکانیزمی دارد که ارتفاع و زاویهٔ صفحه را تنظیم می‌کند تا چشم و گردن کمتر خسته شوند. پشت آن اغلب یک سوراخ برای عبور کابل‌ها هست.'},
    base:{ label:'قاعده', yaw:0.5,
      text:'قاعده (Base) مانیتور را روی میز پایدار نگه می‌دارد و مانع افتادن آن می‌شود. لایهٔ لاستیکی زیرش هم جلوی لیز خوردن را می‌گیرد.'},
    ports:{ label:'پورت‌های پشت', yaw:Math.PI,
      text:'پشت مانیتور درگاه‌های اتصال قرار دارد. HDMI و DisplayPort تصویر (و گاهی صدا) را از کامپیوتر می‌گیرند، و درگاه برق، مانیتور را به برق شهری وصل می‌کند. کیفیت کابل روی رزولوشن و نرخ تازه‌سازی (Hz) اثر دارد.'}
  };

  /* ---------- Renderer / scene ---------- */
  const renderer = new THREE.WebGLRenderer({antialias:true, alpha:true});
  renderer.setPixelRatio(Math.min(window.devicePixelRatio||1, 2));
  stage.insertBefore(renderer.domElement, stage.firstChild);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(38, 1, 0.1, 100);
  camera.position.set(0, 2.5, 9.2);
  camera.lookAt(0, 1.55, 0);

  scene.add(new THREE.HemisphereLight(0xcfe0ff, 0x1a1f28, 0.9));
  const key = new THREE.DirectionalLight(0xffffff, 0.9); key.position.set(4,6,6); scene.add(key);
  const rim = new THREE.DirectionalLight(0xffb454, 0.5); rim.position.set(-5,3,-5); scene.add(rim);

  /* ---------- Materials ---------- */
  const mat = (c, r=0.45, m=0.2) => new THREE.MeshStandardMaterial({color:c, roughness:r, metalness:m, emissive:0x000000});
  const M = {
    bezel: mat(0x14181e,0.35,0.3),
    shell: mat(0x232a33,0.5,0.35),
    neck:  mat(0x9aa4b0,0.35,0.85),
    base:  mat(0x8d97a3,0.35,0.85),
    port:  mat(0x05070a,0.8,0.1),
    led:   mat(0xffb454,0.3,0)
  };
  M.led.emissive.set(0xffb454); M.led.emissiveIntensity = 1.2;

  /* ---------- Screen texture (live clock) ---------- */
  const cv = document.createElement('canvas'); cv.width = 1024; cv.height = 560;
  const g = cv.getContext('2d');
  const tex = new THREE.CanvasTexture(cv);
  tex.anisotropy = 4;
  function drawScreen(){
    const w=cv.width,h=cv.height;
    const bg=g.createLinearGradient(0,0,w,h);
    bg.addColorStop(0,'#10243d'); bg.addColorStop(.55,'#1d4f6e'); bg.addColorStop(1,'#e58a3c');
    g.fillStyle=bg; g.fillRect(0,0,w,h);
    // sun and hills
    g.fillStyle='rgba(255,220,160,.9)'; g.beginPath(); g.arc(760,330,70,0,Math.PI*2); g.fill();
    g.fillStyle='#0c1a2a'; g.beginPath(); g.moveTo(0,h); g.lineTo(0,400); g.quadraticCurveTo(260,300,520,410); g.quadraticCurveTo(780,500,w,380); g.lineTo(w,h); g.fill();
    // window
    g.fillStyle='rgba(15,20,27,.82)'; g.fillRect(90,70,420,230);
    g.fillStyle='#2a3544'; g.fillRect(90,70,420,34);
    ['#ff6b5e','#ffbd44','#3ddc84'].forEach((c,i)=>{g.fillStyle=c;g.beginPath();g.arc(112+i*22,87,6,0,7);g.fill();});
    const d=new Date();
    const t=d.toLocaleTimeString('en-GB',{hour:'2-digit',minute:'2-digit',second:'2-digit'});
    g.fillStyle='#ffb454'; g.font='bold 78px monospace'; g.textAlign='center'; g.fillText(t,300,200);
    g.fillStyle='#9aa7b6'; g.font='22px sans-serif'; g.fillText(d.toLocaleDateString('en-GB'),300,250);
    // taskbar
    g.fillStyle='rgba(10,14,20,.85)'; g.fillRect(0,h-48,w,48);
    for(let i=0;i<5;i++){ g.fillStyle=i==1?'#ffb454':'#3a4656'; g.fillRect(24+i*52,h-36,34,24); }
    tex.needsUpdate=true;
  }
  drawScreen(); setInterval(drawScreen,1000);

  /* ---------- Build monitor ---------- */
  const monitor = new THREE.Group(); scene.add(monitor);
  const pickables = [];
  function part(mesh, key){ mesh.userData.key = key; pickables.push(mesh); monitor.add(mesh); return mesh; }

  const CY = 2.2; // monitor center height
  // front bezel
  part(new THREE.Mesh(new THREE.BoxGeometry(4.9,2.85,0.1), M.bezel), 'bezel').position.set(0,CY,0);
  // back shell (slightly smaller)
  const shell = new THREE.Mesh(new THREE.BoxGeometry(4.7,2.65,0.26), M.shell);
  shell.position.set(0,CY,-0.17); shell.userData.key='bezel'; pickables.push(shell); monitor.add(shell);
  // screen
  const screen = new THREE.Mesh(new THREE.PlaneGeometry(4.5,2.5), new THREE.MeshBasicMaterial({map:tex}));
  screen.position.set(0,CY+0.1,0.052); screen.userData.key='screen'; pickables.push(screen); monitor.add(screen);
  // power LED
  const led = new THREE.Mesh(new THREE.CylinderGeometry(0.045,0.045,0.02,20), M.led);
  led.rotation.x = Math.PI/2; led.position.set(1.95,CY-1.29,0.055); led.userData.key='led'; pickables.push(led); monitor.add(led);
  // neck
  part(new THREE.Mesh(new THREE.BoxGeometry(0.38,1.35,0.14), M.neck),'neck').position.set(0,0.78,-0.32);
  // base
  const base = new THREE.Mesh(new THREE.CylinderGeometry(1.1,1.2,0.1,64), M.base);
  base.scale.set(1.25,1,0.8); base.position.set(0,0.05,-0.32); base.userData.key='base'; pickables.push(base); monitor.add(base);
  // ports on back
  [[-1.3,0.34,0.12],[-0.8,0.34,0.12],[-0.3,0.26,0.1],[0.55,0.5,0.14]].forEach(p=>{
    const m = new THREE.Mesh(new THREE.BoxGeometry(p[1],p[2],0.05), M.port);
    m.position.set(p[0]*-1,CY-0.95,-0.3); m.userData.key='ports'; pickables.push(m); monitor.add(m);
  });
  // subtle floor shadow disc
  const sh = new THREE.Mesh(new THREE.CircleGeometry(2.6,48),
    new THREE.MeshBasicMaterial({color:0x000000,transparent:true,opacity:0.35}));
  sh.rotation.x=-Math.PI/2; sh.position.set(0,0.001,-0.32); sh.scale.set(1.1,0.7,1); scene.add(sh);

  /* ---------- Selection ---------- */
  const chipsEl = document.getElementById('chips');
  const titleEl = document.getElementById('title');
  const descEl = document.getElementById('desc');
  const chipMap = {};
  Object.keys(INFO).forEach(k=>{
    const b=document.createElement('button'); b.className='chip'; b.textContent=INFO[k].label;
    b.setAttribute('aria-pressed','false'); b.onclick=()=>select(k,true); chipsEl.appendChild(b); chipMap[k]=b;
  });
  let selected=null;
  const glow = {bezel:[M.bezel,M.shell], screen:[], led:[M.led], neck:[M.neck], base:[M.base], ports:[M.port]};
  function select(k, rotate){
    selected=k; autoRotate=false;
    Object.values(M).forEach(m=>{ if(m!==M.led) m.emissive.set(0x000000); });
    (glow[k]||[]).forEach(m=>{ if(m!==M.led) m.emissive.set(0x5a3a08); });
    Object.keys(chipMap).forEach(c=>chipMap[c].setAttribute('aria-pressed', c===k));
    titleEl.textContent=INFO[k].label; descEl.textContent=INFO[k].text;
    if(rotate){ // rotate to the nearest equivalent angle
      const full=Math.PI*2, tgt=INFO[k].yaw;
      targetY = tgt + Math.round((targetY - tgt)/full)*full;
    }
    const c=chipMap[k]; if(c.scrollIntoView) c.scrollIntoView({block:'nearest',inline:'center'});
  }

  /* ---------- Interaction ---------- */
  let targetY=-0.5, curY=-0.5, targetX=0.05, curX=0.05, autoRotate=true;
  let down=false, moved=0, lx=0, ly=0;
  const hint=document.getElementById('hint');
  const dom=renderer.domElement;
  dom.addEventListener('pointerdown',e=>{down=true;moved=0;lx=e.clientX;ly=e.clientY;dom.setPointerCapture(e.pointerId);});
  dom.addEventListener('pointermove',e=>{
    if(!down) return;
    const dx=e.clientX-lx, dy=e.clientY-ly; lx=e.clientX; ly=e.clientY; moved+=Math.abs(dx)+Math.abs(dy);
    targetY+=dx*0.008; targetX=Math.max(-0.35,Math.min(0.5,targetX+dy*0.004));
    autoRotate=false; hint.style.opacity=0;
  });
  dom.addEventListener('pointerup',e=>{
    down=false;
    if(moved<6){ // tap = pick
      const r=dom.getBoundingClientRect();
      const ndc=new THREE.Vector2(((e.clientX-r.left)/r.width)*2-1, -((e.clientY-r.top)/r.height)*2+1);
      const ray=new THREE.Raycaster(); ray.setFromCamera(ndc,camera);
      const hit=ray.intersectObjects(pickables,false)[0];
      if(hit) select(hit.object.userData.key,false);
    }
  });

  /* ---------- Resize & loop ---------- */
  function resize(){
    const w=stage.clientWidth, h=stage.clientHeight;
    renderer.setSize(w,h,false); camera.aspect=w/h;
    camera.fov = w/h < 0.8 ? 52 : 38; // wider view on portrait phones
    camera.updateProjectionMatrix();
  }
  window.addEventListener('resize',resize); resize();

  const reduce = window.matchMedia && matchMedia('(prefers-reduced-motion: reduce)').matches;
  if(reduce) autoRotate=false;
  function loop(){
    requestAnimationFrame(loop);
    if(autoRotate) targetY+=0.004;
    curY+=(targetY-curY)*0.09; curX+=(targetX-curX)*0.09;
    monitor.rotation.y=curY; monitor.rotation.x=curX;
    sh.rotation.z=0;
    renderer.render(scene,camera);
  }
  loop();
  select('screen',false);
  autoRotate = !reduce; // keep gentle intro spin
})();
</script>
</body>
</html>
