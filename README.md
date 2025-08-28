[音彩（おんさい）.html](https://github.com/user-attachments/files/22017191/default.html)
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>Voice Particles Interactive</title>
<style>
  body { margin:0; overflow:hidden; background:#000; }
  canvas { display:block; }
  #message {
    position:absolute; top:50%; left:50%;
    transform:translate(-50%, -50%);
    color:#f5f5f5;
    font-family:sans-serif; font-size:2em;
    pointer-events:none; text-align:center;
    opacity:1;
  }
</style>
</head>
<body>
<div id="message">音を鳴らして、赤色を表示させよう！</div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/pixi.js/7.2.4/pixi.min.js"></script>
<script>
(async () => {
  const app = new PIXI.Application({ resizeTo: window, backgroundColor: 0x000000, antialias:true });
  document.body.appendChild(app.view);

  const particleContainer = new PIXI.Container();
  app.stage.addChild(particleContainer);

  const waveform = new PIXI.Graphics();
  app.stage.addChild(waveform);

  const message = document.getElementById("message");
  let soundDetected = false;

  const stream = await navigator.mediaDevices.getUserMedia({ audio:true });
  const audioCtx = new (window.AudioContext||window.webkitAudioContext)();
  const source = audioCtx.createMediaStreamSource(stream);
  const analyser = audioCtx.createAnalyser();
  analyser.fftSize = 1024;
  source.connect(analyser);

  const freqData = new Uint8Array(analyser.frequencyBinCount);
  const timeData = new Uint8Array(analyser.fftSize);
  const smoothWave = new Float32Array(analyser.fftSize);

  const particles = [];

  // 形状管理
  const shapes = ['circle','star','hexagon','heart'];
  let currentShapeIndex = 0;
  const shapeChangeThreshold = 80;
  let lastShapeTrigger = false;

  function hsvToHex(h,s,v){
    let f=(n,k=(n+h/60)%6)=>v - v*s*Math.max(Math.min(k,4-k,1),0);
    const r=Math.round(f(5)*255), g=Math.round(f(3)*255), b=Math.round(f(1)*255);
    return (r<<16)+(g<<8)+b;
  }

  function drawStar(g, size){
    g.moveTo(0,-size);
    for(let i=1;i<5;i++){
      const angle = i*Math.PI*2/5 - Math.PI/2;
      g.lineTo(Math.cos(angle)*size, Math.sin(angle)*size);
    }
  }
  function drawHexagon(g, size){
    g.moveTo(size,0);
    for(let i=1;i<6;i++){
      const angle = i*Math.PI/3;
      g.lineTo(Math.cos(angle)*size, Math.sin(angle)*size);
    }
  }
  function drawHeart(g, size){
    g.moveTo(0,-size/2);
    g.bezierCurveTo(size/2,-size/2, size/2,size/2,0,size);
    g.bezierCurveTo(-size/2,size/2, -size/2,-size/2,0,-size/2);
  }

  function createParticle(avg){
    const p = new PIXI.Graphics();
    const size = 5 + Math.random()*10;

    const shapeType = shapes[currentShapeIndex];

    p.beginFill(getParticleColor(avg));
    if(shapeType==='circle') p.drawCircle(0,0,size);
    else if(shapeType==='star') drawStar(p,size);
    else if(shapeType==='hexagon') drawHexagon(p,size);
    else if(shapeType==='heart') drawHeart(p,size);
    p.endFill();

    const centerX = app.renderer.width/2;
    const centerY = app.renderer.height/2;
    const angle = Math.random()*Math.PI*2;
    const distance = Math.random()*50;
    p.x = centerX + Math.cos(angle)*distance;
    p.y = centerY + Math.sin(angle)*distance;

    const speed = 2 + (avg/120)*5; 
    p.vx = Math.cos(angle)*speed;
    p.vy = Math.sin(angle)*speed;

    // パーティクル寿命を顕著に
    p.life = 80 + (avg/120)*120;

    particleContainer.addChild(p);
    particles.push(p);
  }

  function getParticleColor(avg){
    const db = Math.min(120, Math.max(0, avg));
    const hue = (db/120)*360;
    return hsvToHex(hue,1,1);
  }

  app.ticker.add(()=>{
    analyser.getByteFrequencyData(freqData);
    analyser.getByteTimeDomainData(timeData);

    const avg = freqData.reduce((a,b)=>a+b,0)/freqData.length;

    if(avg>20 && !soundDetected){
      soundDetected = true;
      message.style.display = 'none';
    }

    if(avg >= shapeChangeThreshold && !lastShapeTrigger){
      currentShapeIndex = (currentShapeIndex +1) % shapes.length;
      lastShapeTrigger = true;
    } else if(avg < shapeChangeThreshold){
      lastShapeTrigger = false;
    }

    // 通常パーティクル生成（3個に減量）
    if(avg>20){
      for(let i=0;i<3;i++){
        createParticle(avg);
      }
    }

    // キラキラ弾けるピーク演出
    const peakThreshold = 80;
    if(avg > peakThreshold){
      for(let i=0;i<10;i++){
        const p = new PIXI.Graphics();
        const size = 2 + Math.random()*4;
        p.beginFill(0xffffff);
        p.drawCircle(0,0,size);
        p.endFill();

        const centerX = app.renderer.width/2;
        const centerY = app.renderer.height/2;
        const angle = Math.random()*Math.PI*2;
        const distance = 5 + Math.random()*20;
        p.x = centerX + Math.cos(angle)*distance;
        p.y = centerY + Math.sin(angle)*distance;

        const speed = 4 + Math.random()*4;
        p.vx = Math.cos(angle)*speed;
        p.vy = Math.sin(angle)*speed;

        // キラキラ寿命も少し長め
        p.life = 40 + Math.random()*40;

        particleContainer.addChild(p);
        particles.push(p);
      }
    }

    // パーティクル更新
    for(let i=particles.length-1;i>=0;i--){
      const p = particles[i];
      p.x += p.vx;
      p.y += p.vy;
      p.alpha = p.life/150; // 寿命が長くなった分、透過も調整
      p.life--;
      if(p.life<=0){
        particleContainer.removeChild(p);
        particles.splice(i,1);
      }
    }

    for(let i=0;i<timeData.length;i++){
      smoothWave[i] = (smoothWave[i]||0)*0.9 + timeData[i]*0.1;
    }

    waveform.clear();
    if(soundDetected){
      waveform.lineStyle(2, 0xf5f5f5, 0.8);
      const w = app.renderer.width;
      const h = app.renderer.height;
      waveform.moveTo(0,h/2 + (smoothWave[0]/128-1)*h/2);
      for(let i=1;i<timeData.length;i++){
        const x = i/timeData.length * w;
        const y = h/2 + (smoothWave[i]/128-1)*h/2;
        waveform.lineTo(x,y);
      }
    }
  });
})();
</script>
</body>
</html>
