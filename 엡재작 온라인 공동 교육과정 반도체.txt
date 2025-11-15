<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>PN Junction Visualizer — 간단한 반도체 시뮬레이터</title>
  <style>
    body{font-family:system-ui,-apple-system,"Segoe UI",Roboto,Arial;margin:18px;background:#f7fafc;color:#0f172a}
    h1{font-size:1.4rem;margin-bottom:6px}
    .row{display:flex;gap:12px;align-items:flex-start}
    .panel{background:white;padding:12px;border-radius:8px;box-shadow:0 6px 18px rgba(2,6,23,0.06);flex:1}
    label{display:block;font-size:0.85rem;margin-top:8px}
    input[type=range]{width:100%}
    canvas{background:#0b1220;border-radius:6px;width:100%;height:240px;display:block}
    .small{font-size:0.8rem;color:#334155}
    .controls{max-width:320px}
    button{margin-top:8px;padding:8px 12px;border-radius:8px;border:0;background:#2563eb;color:white;cursor:pointer}
    footer{margin-top:12px;font-size:0.8rem;color:#475569}
    .legend{display:flex;gap:8px;align-items:center;margin-top:8px}
    .swatch{width:12px;height:12px;border-radius:2px}
  </style>
</head>
<body>
  <h1>PN Junction Visualizer (간단한 반도체 시뮬레이터)</h1>
  <div class="row">
    <div class="panel controls">
      <strong>설정</strong>
      <label>격자 크기: <span id="nCellsVal">200</span></label>
      <input id="nCells" type="range" min="50" max="500" value="200">

      <label>왼쪽 농도(p-type) (cm⁻³): <span id="leftNVal">1e18</span></label>
      <input id="leftN" type="range" min="12" max="21" step="0.1" value="18">

      <label>오른쪽 농도(n-type) (cm⁻³): <span id="rightNVal">1e16</span></label>
      <input id="rightN" type="range" min="12" max="21" step="0.1" value="16">

      <label>확산계수 D (cm²/s): <span id="DVal">1</span></label>
      <input id="D" type="range" min="0.1" max="5" step="0.1" value="1">

      <label>이동도 μ (cm²/V·s): <span id="muVal">1000</span></label>
      <input id="mu" type="range" min="10" max="2000" step="10" value="1000">

      <label>외부 바이어스 (V): <span id="biasVal">0</span></label>
      <input id="bias" type="range" min="-1" max="1" step="0.01" value="0">

      <label>타임스텝 (dt, s): <span id="dtVal">0.02</span></label>
      <input id="dt" type="range" min="0.005" max="0.1" step="0.005" value="0.02">

      <div style="margin-top:8px;display:flex;gap:8px">
        <button id="resetBtn">초기화</button>
        <button id="stepBtn">한 단계 진행</button>
        <button id="runBtn">실행/정지</button>
      </div>

      <div class="small" style="margin-top:8px">
        <div class="legend"><div class="swatch" style="background:#3b82f6"></div>전자 농도 (n)</div>
        <div class="legend"><div class="swatch" style="background:#fb7185"></div>정공 농도 (p)</div>
      </div>

      <footer>이 페이지는 단순화된 1D 확산+드리프트 모델을 사용합니다. 교육/시각화 목적용이며 실제 반도체 공정 모델과는 다릅니다.</footer>
    </div>

    <div class="panel" style="flex:2">
      <canvas id="plot" width="900" height="320"></canvas>
      <div style="display:flex;gap:12px;margin-top:8px;align-items:center">
        <div class="small">시간: <span id="timeLabel">0.00</span> s</div>
        <div class="small">전류 추정: <span id="currentLabel">0.00</span> (arb)</div>
      </div>
    </div>
  </div>

  <script>
    // 간단한 1D 확산+드리프트 시뮬레이션 (교육용)
    const canvas = document.getElementById('plot');
    const ctx = canvas.getContext('2d');

    const nCellsSlider = document.getElementById('nCells');
    const leftNSlider = document.getElementById('leftN');
    const rightNSlider = document.getElementById('rightN');
    const DSlider = document.getElementById('D');
    const muSlider = document.getElementById('mu');
    const biasSlider = document.getElementById('bias');
    const dtSlider = document.getElementById('dt');

    const nCellsVal = document.getElementById('nCellsVal');
    const leftNVal = document.getElementById('leftNVal');
    const rightNVal = document.getElementById('rightNVal');
    const DVal = document.getElementById('DVal');
    const muVal = document.getElementById('muVal');
    const biasVal = document.getElementById('biasVal');
    const dtVal = document.getElementById('dtVal');

    const resetBtn = document.getElementById('resetBtn');
    const stepBtn = document.getElementById('stepBtn');
    const runBtn = document.getElementById('runBtn');

    const timeLabel = document.getElementById('timeLabel');
    const currentLabel = document.getElementById('currentLabel');

    let N = parseInt(nCellsSlider.value);
    let leftN = Math.pow(10, parseFloat(leftNSlider.value));
    let rightN = Math.pow(10, parseFloat(rightNSlider.value));
    let D = parseFloat(DSlider.value);
    let mu = parseFloat(muSlider.value);
    let bias = parseFloat(biasSlider.value);
    let dt = parseFloat(dtSlider.value);

    let n = new Float64Array(N); // 전자
    let p = new Float64Array(N); // 정공
    let time = 0;
    let running = false;
    let rafId = null;

    function updateParams(){
      N = parseInt(nCellsSlider.value);
      leftN = Math.pow(10, parseFloat(leftNSlider.value));
      rightN = Math.pow(10, parseFloat(rightNSlider.value));
      D = parseFloat(DSlider.value);
      mu = parseFloat(muSlider.value);
      bias = parseFloat(biasSlider.value);
      dt = parseFloat(dtSlider.value);

      nCellsVal.textContent = N;
      leftNVal.textContent = formatSci(leftN);
      rightNVal.textContent = formatSci(rightN);
      DVal.textContent = D.toFixed(2);
      muVal.textContent = mu.toFixed(0);
      biasVal.textContent = bias.toFixed(2);
      dtVal.textContent = dt.toFixed(3);
    }

    function formatSci(x){
      const e = Math.floor(Math.log10(x));
      const m = x / Math.pow(10,e);
      return m.toFixed(2)+"e"+e;
    }

    function initArrays(){
      n = new Float64Array(N);
      p = new Float64Array(N);
      const mid = Math.floor(N/2);
      for(let i=0;i<N;i++){
        if(i<mid){ // p-side
          p[i] = leftN; n[i] = 1e10; // 기본 전자농도 낮음
        } else { // n-side
          n[i] = rightN; p[i] = 1e10;
        }
      }
      time = 0;
      draw();
    }

    function step(){
      // 전계는 간단하게 바이어스로 인한 선형 전압 강하로 근사
      const E = bias / N; // V per cell (very simplified)
      // drift velocity v = mu * E
      const v = mu * E; // cm/s (units are mixed but OK for visualization)

      const nNew = new Float64Array(N);
      const pNew = new Float64Array(N);

      // 이웃 차분
      for(let i=1;i<N-1;i++){
        // 확산항 (중앙차분)
        const diffN = D*(n[i+1]-2*n[i]+n[i-1]);
        const diffP = D*(p[i+1]-2*p[i]+p[i-1]);
        // 드리프트 근사 (업윈드 차분)
        const driftN = -v*(n[i]-n[i-1]);
        const driftP = v*(p[i+1]-p[i]); // 정공 방향 반대

        nNew[i] = n[i] + (diffN + driftN) * dt;
        pNew[i] = p[i] + (diffP + driftP) * dt;
        // 물리적으로 음수가 되면 0으로 제한
        if(nNew[i]<0) nNew[i]=0;
        if(pNew[i]<0) pNew[i]=0;
      }

      // 경계는 고정된 도핑 농도로 유지 (간단 모사)
      nNew[0] = Math.max(1e6, n[0]);
      pNew[0] = Math.max(1e6, p[0]);
      nNew[N-1] = Math.max(1e6, n[N-1]);
      pNew[N-1] = Math.max(1e6, p[N-1]);

      // 내부 정합: 중간에서 도핑 유지
      const mid = Math.floor(N/2);
      for(let i=0;i<mid;i++){ pNew[i] = leftN; }
      for(let i=mid;i<N;i++){ nNew[i] = rightN; }

      // swap
      n = nNew;
      p = pNew;

      time += dt;
      draw();
    }

    function draw(){
      ctx.clearRect(0,0,canvas.width,canvas.height);
      // 배경 그리기
      ctx.fillStyle = '#0b1220';
      ctx.fillRect(0,0,canvas.width,canvas.height);

      // find ranges
      let maxVal = 0;
      for(let i=0;i<N;i++){
        if(n[i]>maxVal) maxVal = n[i];
        if(p[i]>maxVal) maxVal = p[i];
      }
      maxVal = Math.max(maxVal,1e10);

      // draw p (pink)
      ctx.beginPath();
      for(let i=0;i<N;i++){
        const x = (i/(N-1))*canvas.width;
        const y = canvas.height - (p[i]/maxVal)*canvas.height;
        if(i===0) ctx.moveTo(x,y); else ctx.lineTo(x,y);
      }
      ctx.lineWidth = 2;
      ctx.strokeStyle = '#fb7185';
      ctx.stroke();

      // draw n (blue)
      ctx.beginPath();
      for(let i=0;i<N;i++){
        const x = (i/(N-1))*canvas.width;
        const y = canvas.height - (n[i]/maxVal)*canvas.height;
        if(i===0) ctx.moveTo(x,y); else ctx.lineTo(x,y);
      }
      ctx.lineWidth = 2;
      ctx.strokeStyle = '#3b82f6';
      ctx.stroke();

      // labels
      ctx.fillStyle = '#94a3b8';
      ctx.font = '12px system-ui';
      ctx.fillText('왼쪽: p-side (높은 정공 농도)', 8, 16);
      ctx.fillText('오른쪽: n-side (높은 전자 농도)', 8, 34);

      timeLabel.textContent = time.toFixed(3);

      // 간단한 '전류' 추정치: 농도 기울기 * v 평균
      const mid = Math.floor(N/2);
      const gradient = (n[mid+2]-n[mid-2]) / 4.0;
      const currentEst = Math.abs(gradient * mu * bias);
      currentLabel.textContent = currentEst.toExponential(2);
    }

    function loop(){
      step();
      if(running) rafId = requestAnimationFrame(loop);
    }

    // 이벤트
    nCellsSlider.addEventListener('input', ()=>{ updateParams(); initArrays(); });
    leftNSlider.addEventListener('input', ()=>{ updateParams(); initArrays(); });
    rightNSlider.addEventListener('input', ()=>{ updateParams(); initArrays(); });
    DSlider.addEventListener('input', ()=>{ updateParams(); });
    muSlider.addEventListener('input', ()=>{ updateParams(); });
    biasSlider.addEventListener('input', ()=>{ updateParams(); });
    dtSlider.addEventListener('input', ()=>{ updateParams(); });

    resetBtn.addEventListener('click', ()=>{ initArrays(); });
    stepBtn.addEventListener('click', ()=>{ step(); });
    runBtn.addEventListener('click', ()=>{
      running = !running;
      runBtn.textContent = running ? '정지' : '실행/정지';
      if(running) loop(); else cancelAnimationFrame(rafId);
    });

    // 초기화
    updateParams();
    initArrays();
  </script>
</body>
</html>
