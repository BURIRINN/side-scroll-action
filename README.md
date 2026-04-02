# side-scroll-action
<!doctype html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Mini Platformer</title>
  <style>
    *{box-sizing:border-box}body{margin:0;overflow:hidden;font-family:system-ui;background:linear-gradient(#8fd3ff,#e0f2fe)}canvas{display:block;width:100vw;height:100vh}.ui{position:fixed;left:16px;top:16px;z-index:2;background:rgba(255,255,255,.92);padding:12px 14px;border-radius:14px;box-shadow:0 8px 24px rgba(0,0,0,.12);min-width:320px}.ui p{margin:6px 0;font-size:14px}.small{font-size:12px;color:#475569}.overlay{position:fixed;inset:0;display:none;align-items:center;justify-content:center;background:rgba(0,0,0,.35);z-index:3}.overlay.show{display:flex}.card{background:#fff;padding:24px;border-radius:18px;text-align:center;width:min(92vw,420px)}button{border:0;background:#2563eb;color:#fff;border-radius:999px;padding:10px 14px;cursor:pointer}
  </style>
</head>
<body>
  <div class="ui">
    <p><b>移動</b> ← → / A D</p>
    <p><b>ジャンプ</b> Space / W / ↑</p>
    <p><b>能力使用</b> F</p>
    <p id="stats"></p>
    <p class="small">？ボックスを下から叩くと強化アイテムが出ます。巨大化中は接触で敵を倒せます。</p>
    <button id="restart">最初から</button>
  </div>

  <div class="overlay" id="ov">
    <div class="card">
      <h2 id="ovt">クリア</h2>
      <p id="ovm"></p>
      <button id="ovb">もう一度</button>
    </div>
  </div>

  <canvas id="c"></canvas>

  <script>
    const c = document.getElementById('c');
    const x = c.getContext('2d');
    const stats = document.getElementById('stats');
    const ov = document.getElementById('ov');
    const ovt = document.getElementById('ovt');
    const ovm = document.getElementById('ovm');
    const ovb = document.getElementById('ovb');
    const restartBtn = document.getElementById('restart');

    const R = () => { c.width = innerWidth; c.height = innerHeight; };
    addEventListener('resize', R);
    R();

    const K = {};
    const W = { w: 2600, h: 1100, g: 980, grav: .72, maxFall: 18 };
    const stage = {
      ground: [[0,980,900,140],[1040,980,700,140],[1880,980,720,140]],
      plats: [[240,900,120,28,'b'],[430,820,120,28,'b'],[640,740,120,28,'b'],[960,880,150,22,'p'],[1180,790,150,22,'p'],[1410,700,150,22,'p'],[1980,900,150,28,'b'],[2200,810,150,28,'b']],
      boxes: [[820,860,'fire'],[1610,660,'ice'],[2360,780,'giant']],
      coins: [[275,850],[465,770],[675,690],[995,835],[1215,745],[1445,655],[2015,845],[2235,755],[2460,730]],
      enemies: [['walk',700,946,540,860],['hop',1260,944,1080,1640],['fly',2100,690,1940,2460]],
      flag: [2480,720,260]
    };

    let cam = 0, gameOver = 0, clear = 0, msgAction = () => {};
    const mk = (x, y, w, h) => ({ x, y, y0:y, w, h, vx:0, vy:0, px:x, py:y, dead:0 });
    let p, coins, solids, boxes, items, enemies, shots, flag, lives;

    function label(v){ return v === 'fire' ? 'ファイア' : v === 'ice' ? 'フリーズ' : v === 'giant' ? '巨大化' : 'ノーマル'; }
    function show(t,m,a){ ov.classList.add('show'); ovt.textContent = t; ovm.textContent = m; msgAction = typeof a === 'function' ? a : (()=>{}); }
    function hide(){ ov.classList.remove('show'); }
    function safeList(v){ return Array.isArray(v) ? v : []; }

    function upd(){
      const got = safeList(coins).filter(v => v.got).length;
      const total = safeList(coins).length;
      stats.textContent = `コイン: ${got} / ${total}　残機: ${lives ?? 0}　能力: ${p ? label(p.power) : 'ノーマル'}`;
    }

    function reset(){
      p = mk(80,924,42,56);
      p.face = 1; p.on = 0; p.buf = 0; p.coy = 0; p.inv = 0; p.power = 'normal'; p.timer = 0; p.cool = 0;
      coins = stage.coins.map(a => ({ x:a[0], y:a[1], r:11, t:Math.random()*9, got:0 }));
      solids = [
        ...stage.ground.map(a => ({ x:a[0], y:a[1], w:a[2], h:a[3], t:'g' })),
        ...stage.plats.map(a => ({ x:a[0], y:a[1], w:a[2], h:a[3], t:a[4] }))
      ];
      boxes = stage.boxes.map(a => ({ x:a[0], y:a[1], w:34, h:34, type:a[2], used:0, bump:0, t:'q' }));
      solids.push(...boxes);
      items = [];
      shots = [];
      enemies = stage.enemies.map(e => e[0] === 'walk'
        ? { ...mk(e[1],e[2],40,34), type:'walk', min:e[3], max:e[4], vx:-1.2, fz:0 }
        : { ...mk(e[1],e[2],e[0] === 'hop' ? 38 : 44, e[0] === 'hop' ? 36 : 28), type:e[0], min:e[3], max:e[4], vx:e[0] === 'hop' ? -1.35 : -1.4, j:60, ph:Math.random()*6, fz:0 }
      );
      flag = { x:stage.flag[0], y:stage.flag[1], h:stage.flag[2] };
      cam = 0;
      lives = 3;
      gameOver = 0;
      clear = 0;
      hide();
      upd();
    }

    ovb.onclick = () => { hide(); msgAction(); };
    restartBtn.onclick = reset;

    addEventListener('keydown', e => {
      K[e.key] = 1;
      if (['ArrowUp','ArrowDown','ArrowLeft','ArrowRight',' '].includes(e.key)) e.preventDefault();
      if (e.repeat) return;
      if ([' ','ArrowUp','w','W'].includes(e.key) && p) p.buf = 8;
      if (['f','F'].includes(e.key)) usePower();
    }, { passive:false });

    addEventListener('keyup', e => K[e.key] = 0);

    function A(a,b){
      if (!a || !b) return false;
      return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
    }

    function hcol(o){
      if (!o) return;
      for (const s of safeList(solids)) {
        if (!s || s.t === 'p') continue;
        if (o.y + o.h - 2 <= s.y || o.y >= s.y + s.h - 2) continue;
        const pr = o.px + o.w, pl = o.px, cr = o.x + o.w, cl = o.x;
        if (o.vx > 0 && pr <= s.x && cr >= s.x) {
          o.x = s.x - o.w;
          o.vx = o.type ? -Math.abs(o.vx) : 0;
        } else if (o.vx < 0 && pl >= s.x + s.w && cl <= s.x + s.w) {
          o.x = s.x + s.w;
          o.vx = o.type ? Math.abs(o.vx) : 0;
        }
      }
    }

    function vcol(o){
      if (!o) return 0;
      let land = 0;
      for (const s of safeList(solids)) {
        if (!s) continue;
        if (o.x + o.w - 4 <= s.x || o.x + 4 >= s.x + s.w) continue;
        const pb = o.py + o.h, cb = o.y + o.h, pt = o.py, ct = o.y;
        if (s.t === 'p') {
          if (o.vy >= 0 && pb <= s.y + 4 && cb >= s.y) {
            o.y = s.y - o.h;
            o.vy = 0;
            land = 1;
          }
          continue;
        }
        if (o.vy >= 0 && pb <= s.y && cb >= s.y) {
          o.y = s.y - o.h;
          o.vy = 0;
          land = 1;
        } else if (o.vy < 0 && pt >= s.y + s.h && ct <= s.y + s.h) {
          o.y = s.y + s.h;
          o.vy = 0;
          if (s.t === 'q') hitBox(s);
        }
      }
      return land;
    }

    function hitBox(b){
      if (!b) return;
      b.bump = 6;
      if (b.used) return;
      b.used = 1;
      safeList(items).push({ x:b.x+5, y:b.y-28, w:24, h:24, type:b.type, pop:18, life:900, ph:Math.random()*6, dead:0 });
    }

    function usePower(){
      if (!p || gameOver || clear || p.cool > 0) return;
      if (!['fire','ice'].includes(p.power)) return;
      safeList(shots).push({
        x: p.x + (p.face > 0 ? p.w + 4 : -10),
        y: p.y + p.h * .42,
        w:16,h:16,
        vx: p.face * (p.power === 'fire' ? 11 : 9),
        vy: p.power === 'fire' ? -2.6 : -1.2,
        g: p.power === 'fire' ? .18 : .08,
        type:p.power,
        life:120,
        dead:0
      });
      p.cool = p.power === 'fire' ? 18 : 22;
    }

    function hurt(){
      if (!p || p.inv || gameOver || clear) return;
      lives--;
      upd();
      if (lives <= 0) {
        gameOver = 1;
        show('ゲームオーバー','もう一度遊びますか？', reset);
        return;
      }
      p.x = 80; p.y = 924; p.vx = 0; p.vy = 0; p.inv = 90; p.power = 'normal'; p.timer = 0; p.w = 42; p.h = 56;
      shots = [];
      items = [];
    }

    function setPower(t){
      if (!p) return;
      const bottom = p.y + p.h;
      p.power = t;
      if (t === 'giant') { p.w = 58; p.h = 78; p.timer = 900; }
      else { p.w = 42; p.h = 56; p.timer = 0; }
      p.y = bottom - p.h;
      upd();
    }

    function hitEnemy(e){
      if (!p || !e || p.inv || e.dead) return;
      if (!A(p, { x:e.x+3, y:e.y+2, w:e.w-6, h:e.h-2 })) return;
      const stomp = p.vy > 0 && p.py + p.h <= e.y + 14;
      if (stomp || p.power === 'giant' || e.fz > 0) {
        e.dead = 1;
        if (stomp) { p.vy = -10.8; p.on = 0; }
      } else {
        hurt();
      }
    }

    function enemyStep(e){
      if (!e || e.dead) return;
      if (e.fz > 0) { e.fz--; hitEnemy(e); return; }
      e.px = e.x; e.py = e.y;
      if (e.type === 'fly') {
        e.x += e.vx;
        if (e.x <= e.min) e.vx = Math.abs(e.vx);
        if (e.x + e.w >= e.max) e.vx = -Math.abs(e.vx);
        e.ph += .08;
        e.y = e.y0 + Math.sin(e.ph) * 48;
        hitEnemy(e);
        return;
      }
      if (e.type === 'hop') {
        if (e.j > 0) e.j--;
        else { e.vy = -11.8; e.j = 70; }
      }
      e.vy = Math.min(W.maxFall, e.vy + W.grav);
      e.x += e.vx;
      hcol(e);
      e.y += e.vy;
      vcol(e);
      if (e.x <= e.min) e.vx = Math.abs(e.vx);
      if (e.x + e.w >= e.max) e.vx = -Math.abs(e.vx);
      hitEnemy(e);
    }

    function step(){
      if (!p || gameOver || clear) return;

      p.px = p.x; p.py = p.y;
      const L = K.ArrowLeft || K.a || K.A;
      const Rr = K.ArrowRight || K.d || K.D;
      const J = K[' '] || K.ArrowUp || K.w || K.W;
      const acc = p.on ? .82 : .48;
      const fr = p.on ? .76 : .97;

      if (L && !Rr) { p.vx -= acc; p.face = -1; }
      else if (Rr && !L) { p.vx += acc; p.face = 1; }
      else { p.vx *= fr; if (Math.abs(p.vx) < .04) p.vx = 0; }

      const speed = p.power === 'giant' ? 8.4 : 7.2;
      const jump = p.power === 'giant' ? 16.6 : 15.2;
      p.vx = Math.max(-speed, Math.min(speed, p.vx));
      p.coy = p.on ? 7 : Math.max(0, p.coy - 1);
      if (p.buf > 0) p.buf--;
      if (p.cool > 0) p.cool--;
      if (p.buf && p.coy) { p.vy = -jump; p.buf = 0; p.coy = 0; p.on = 0; }
      if (!J && p.vy < -5) p.vy += .65;
      if (p.power === 'giant' && p.timer > 0 && !--p.timer) setPower('normal');

      p.vy = Math.min(W.maxFall, p.vy + W.grav);
      p.x += p.vx;
      hcol(p);
      p.on = 0;
      p.y += p.vy;
      p.on = vcol(p);
      p.x = Math.max(0, Math.min(W.w - p.w, p.x));
      if (p.y > W.h + 260) hurt();
      if (p.inv > 0) p.inv--;

      for (const c0 of safeList(coins)) {
        if (!c0 || c0.got) continue;
        c0.t += .08;
        if (Math.hypot((p.x + p.w/2) - c0.x, (p.y + p.h/2) - c0.y) < 24) { c0.got = 1; upd(); }
      }

      for (const b of safeList(boxes)) if (b && b.bump > 0) b.bump--;

      for (const it of safeList(items)) {
        if (!it || it.dead) continue;
        if (--it.life <= 0) { it.dead = 1; continue; }
        if (it.pop > 0) { it.y -= 1.8; it.pop--; }
        else it.y += Math.sin((it.ph += .08)) * .3;
        if (A(p, it)) { setPower(it.type); it.dead = 1; }
      }

      for (const s of safeList(shots)) {
        if (!s || s.dead) continue;
        if (--s.life <= 0) { s.dead = 1; continue; }
        s.vy += s.g; s.x += s.vx; s.y += s.vy;
        for (const o of safeList(solids)) {
          if (o && o.t !== 'p' && A(s, o)) s.dead = 1;
        }
        for (const e of safeList(enemies)) {
          if (!e || e.dead) continue;
          if (A(s, { x:e.x+3, y:e.y+2, w:e.w-6, h:e.h-2 })) {
            if (s.type === 'fire') e.dead = 1;
            else e.fz = 220;
            s.dead = 1;
          }
        }
      }

      for (const e of safeList(enemies)) enemyStep(e);

      if (flag && p.x + p.w > flag.x && p.y + p.h > flag.y) {
        clear = 1;
        show('クリア！', `コイン ${safeList(coins).filter(v => v.got).length} / ${safeList(coins).length} を集めました。`, reset);
      }

      cam += ((p.x - c.width * .33 + p.vx * 18) - cam) * .12;
      cam = Math.max(0, Math.min(W.w - c.width, cam));
    }

    function rr(x0,y0,w,h,r){
      x.beginPath();
      x.moveTo(x0+r,y0); x.lineTo(x0+w-r,y0); x.quadraticCurveTo(x0+w,y0,x0+w,y0+r);
      x.lineTo(x0+w,y0+h-r); x.quadraticCurveTo(x0+w,y0+h,x0+w-r,y0+h);
      x.lineTo(x0+r,y0+h); x.quadraticCurveTo(x0,y0+h,x0,y0+h-r);
      x.lineTo(x0,y0+r); x.quadraticCurveTo(x0,y0,x0+r,y0); x.closePath();
    }

    function draw(){
      const g = x.createLinearGradient(0,0,0,c.height);
      g.addColorStop(0,'#8fd3ff'); g.addColorStop(1,'#e0f2fe');
      x.fillStyle = g; x.fillRect(0,0,c.width,c.height);

      x.fillStyle = '#94a3b8';
      for (let i=-1;i<10;i++) {
        const bx = i*340 - (cam*.24 % 340);
        x.beginPath(); x.moveTo(bx,W.g+24); x.lineTo(bx+90,W.g-180); x.lineTo(bx+180,W.g+24); x.closePath(); x.fill();
      }

      x.fillStyle = 'rgba(255,255,255,.9)';
      for (let i=0;i<6;i++) {
        const cx = i*480 - (cam*.12 % 480) + 120, cy = 120 + (i%3)*35;
        x.beginPath(); x.arc(cx,cy,22,0,7); x.arc(cx+28,cy-8,28,0,7); x.arc(cx+58,cy,22,0,7); x.fill();
      }

      for (const s of safeList(solids)) {
        if (!s) continue;
        const X = s.x - cam, Y = s.y - (s.bump ? Math.sin((6-s.bump)/6*Math.PI)*8 : 0);
        if (s.t === 'g') { x.fillStyle='#4caf50'; x.fillRect(X,Y,s.w,12); x.fillStyle='#7c5b3c'; x.fillRect(X,Y+12,s.w,s.h-12); }
        else if (s.t === 'b') { x.fillStyle='#d97745'; x.fillRect(X,Y,s.w,s.h); }
        else if (s.t === 'p') { x.fillStyle='#d9a35f'; x.fillRect(X,Y,s.w,s.h); }
        else {
          x.fillStyle = s.used ? '#94a3b8' : '#fbbf24'; x.fillRect(X,Y,s.w,s.h);
          x.strokeStyle = s.used ? '#64748b' : '#b45309'; x.lineWidth = 3; x.strokeRect(X+1.5,Y+1.5,s.w-3,s.h-3);
          x.fillStyle = s.used ? '#475569' : '#b45309';
          if (s.used) { x.fillRect(X+10,Y+12,14,10); x.fillRect(X+12,Y+24,10,4); }
          else { x.font='bold 22px system-ui'; x.fillText('?',X+10,Y+24); }
        }
      }

      for (const c0 of safeList(coins)) {
        if (!c0 || c0.got) continue;
        x.fillStyle='#facc15'; x.beginPath(); x.ellipse(c0.x-cam,c0.y+Math.sin(c0.t)*3,8+Math.sin(c0.t)*2,11,0,0,7); x.fill(); x.strokeStyle='#ca8a04'; x.stroke();
      }

      for (const it of safeList(items)) {
        if (!it || it.dead) continue;
        const X = it.x - cam, Y = it.y + (it.pop ? 0 : Math.sin(it.ph)*2);
        if (it.type === 'fire') {
          x.fillStyle='#fb7185'; x.beginPath(); x.moveTo(X+12,Y); x.quadraticCurveTo(X+24,Y+7,X+18,Y+22); x.quadraticCurveTo(X+12,Y+18,X+9,Y+24); x.quadraticCurveTo(X+2,Y+16,X+4,Y+8); x.quadraticCurveTo(X+7,Y+2,X+12,Y); x.fill();
        } else if (it.type === 'ice') {
          x.fillStyle='#60a5fa'; x.beginPath(); x.moveTo(X+12,Y); x.lineTo(X+24,Y+12); x.lineTo(X+12,Y+24); x.lineTo(X,Y+12); x.closePath(); x.fill();
        } else {
          x.fillStyle='#34d399'; x.beginPath(); x.arc(X+12,Y+12,11,0,7); x.fill(); x.fillStyle='#065f46'; x.fillRect(X+10,Y+4,4,16); x.fillRect(X+4,Y+10,16,4);
        }
      }

      for (const e of safeList(enemies)) {
        if (!e || e.dead) continue;
        const X = e.x - cam, Y = e.y;
        x.fillStyle = e.fz ? '#93c5fd' : e.type === 'walk' ? '#7c3f00' : e.type === 'hop' ? '#6d28d9' : '#0f766e';
        rr(X,Y+8,e.w,e.h-8,12); x.fill();
        x.fillStyle = e.fz ? '#dbeafe' : '#f5c9a5'; x.fillRect(X+6,Y+4,e.w-12,12);
        x.fillStyle = '#111827'; x.fillRect(X+11,Y+10,4,4); x.fillRect(X+e.w-15,Y+10,4,4); x.fillRect(X+10,Y+e.h-8,8,6); x.fillRect(X+e.w-18,Y+e.h-8,8,6);
      }

      for (const s of safeList(shots)) {
        if (!s || s.dead) continue;
        const X = s.x - cam;
        x.fillStyle = s.type === 'fire' ? '#fb7185' : '#60a5fa';
        if (s.type === 'fire') {
          x.beginPath(); x.arc(X+8,s.y+8,8,0,7); x.fill(); x.fillStyle='#fde68a'; x.beginPath(); x.arc(X+8,s.y+8,4,0,7); x.fill();
        } else {
          rr(X+1,s.y+1,14,14,4); x.fill(); x.fillStyle='#eff6ff'; x.fillRect(X+5,s.y+4,6,8);
        }
      }

      if (p) {
        const PX = p.x - cam, PY = p.y;
        if (!(p.inv && Math.floor(p.inv/4)%2)) {
          const sx = p.w/42, sy = p.h/56;
          if (p.power !== 'normal') {
            x.fillStyle = p.power === 'fire' ? 'rgba(251,113,133,.25)' : p.power === 'ice' ? 'rgba(96,165,250,.22)' : 'rgba(52,211,153,.18)';
            x.beginPath(); x.ellipse(PX+p.w/2,PY+p.h/2,p.w*.7,p.h*.6,0,0,7); x.fill();
          }
          x.save();
          x.translate(PX+p.w/2,PY+p.h/2); x.scale(p.face,1); x.translate(-(PX+p.w/2),-(PY+p.h/2));
          x.fillStyle='#ef4444'; x.fillRect(PX+10*sx,PY+18*sy,22*sx,30*sy);
          x.fillStyle='#f5c9a5'; x.fillRect(PX+11*sx,PY+8*sy,20*sx,16*sy);
          x.fillStyle = p.power === 'fire' ? '#fb7185' : p.power === 'ice' ? '#60a5fa' : p.power === 'giant' ? '#34d399' : '#b91c1c';
          x.fillRect(PX+7*sx,PY+4*sy,28*sx,8*sy); x.fillRect(PX+6*sx,PY+10*sy,16*sx,4*sy);
          x.fillStyle='#111827'; x.fillRect(PX+12*sx,PY+48*sy,8*sx,8*sy); x.fillRect(PX+22*sx,PY+48*sy,8*sx,8*sy);
          x.fillStyle = p.power === 'giant' ? '#059669' : '#2563eb';
          x.fillRect(PX+6*sx,PY+24*sy,8*sx,22*sy); x.fillRect(PX+28*sx,PY+24*sy,8*sx,22*sy);
          x.restore();
        }
      }

      if (flag) {
        const FX = flag.x - cam;
        x.fillStyle='#f8fafc'; x.fillRect(FX,flag.y,8,flag.h);
        x.fillStyle='#22c55e'; x.beginPath(); x.moveTo(FX+8,flag.y+12); x.lineTo(FX+56,flag.y+24); x.lineTo(FX+8,flag.y+40); x.closePath(); x.fill();
      }
    }

    function runTests(){
      const tPlayer = mk(0,0,10,10);
      const tWall = { x:20, y:0, w:10, h:20, t:'g' };
      console.assert(A(tPlayer, tWall) === false, 'A: separated objects should not collide');
      console.assert(A(tPlayer, { x:5, y:5, w:10, h:10 }) === true, 'A: overlapping objects should collide');
      console.assert(A(undefined, tWall) === false, 'A: undefined object should be handled safely');
      const oldSolids = solids;
      solids = [tWall];
      const mover = { x:15, y:0, w:10, h:10, px:5, py:0, vx:10, vy:0 };
      hcol(mover);
      console.assert(mover.x === 10, 'hcol: mover should stop at wall edge');
      solids = oldSolids;
      reset();
      console.assert(!!p && typeof p.x === 'number', 'reset: player should initialize before loop');
      console.assert(Array.isArray(boxes) && boxes.length === 3, 'reset: boxes should initialize');
    }

    reset();
    runTests();

    (function loop(){
      step();
      draw();
      requestAnimationFrame(loop);
    })();
  </script>
</body>
</html>
