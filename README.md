# cpu_occupy
<!DOCTYPE html>
<html>
<head>
  <title>CPU Load Controller</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      padding: 20px;
      background: #f0f8ff;
    }
    button {
      padding: 10px 20px;
      font-size: 16px;
      margin: 10px;
      cursor: pointer;
      background: #4CAF50;
      color: white;
      border: none;
      border-radius: 4px;
    }
    button:disabled {
      background: #cccccc;
      cursor: not-allowed;
    }
    #status {
      margin-top: 20px;
      font-weight: bold;
      padding: 10px;
      border-radius: 4px;
    }
    .idle { color: #e91e63; background: #ffe6e6; }
    .active { color: #2E7D32; background: #e8f5e9; }
    .stopped { color: #f57c00; background: #fff3e0; }
  </style>
</head>
<body>
  <h2>🔥 CPU 负载控制器</h2>
  <p>每个 CPU 核心将以 50% 负载运行，系统核心数: <span id="coreCount">检测中...</span></p>
  <button id="startBtn" onclick="startAll()">开始负载</button>
  <button id="stopBtn" onclick="stopAll()" disabled>停止负载</button>
  <div id="status" class="idle">状态：空闲</div>

  <script>
    const statusEl = document.getElementById('status');
    const coreCountEl = document.getElementById('coreCount');
    const startBtn = document.getElementById('startBtn');
    const stopBtn = document.getElementById('stopBtn');
    let workers = [];

    // 检测核心数
    const numCores = navigator.hardwareConcurrency || 4;
    coreCountEl.textContent = numCores;

    function startAll() {
      statusEl.textContent = `启动中: 创建 ${numCores} 个Worker...`;
      statusEl.className = 'active';
      startBtn.disabled = true;
      stopBtn.disabled = false;
      
      // 终止现有Worker
      stopAll(false);
      
      // 创建Blob URL形式的Worker
      const workerScript = `
        self.onmessage = function(e) {
          if (e.data.action === 'start') {
            const dutyCycle = e.data.dutyCycle;
            const cycleTime = 200; // 周期200ms
            const busyTime = cycleTime * dutyCycle;
            
            function busyWait(duration) {
              const start = Date.now();
              while (Date.now() - start < duration) {
                // 高精度CPU计算
                for(let i=0; i<1e6; i++) Math.sqrt(i);
              }
            }
            
            function cycle() {
              busyWait(busyTime);      // 工作阶段
              setTimeout(cycle, cycleTime - busyTime); // 休眠阶段
            }
            
            cycle();
          }
        };
      `;
      
      const blob = new Blob([workerScript], { type: 'application/javascript' });
      const workerUrl = URL.createObjectURL(blob);

      // 创建Worker实例
      for (let i = 0; i < numCores/3; i++) {
        const worker = new Worker(workerUrl);
        worker.postMessage({ 
          action: 'start', 
          dutyCycle: 1.0  // 50%负载
        });
        workers.push(worker);
      }
      
      statusEl.textContent = `运行中: ${numCores}个Worker激活 (总负载~${numCores * 50}%)`;
    }

    function stopAll(updateUI = true) {
      workers.forEach(w => w.terminate());
      workers = [];
      
      if (updateUI) {
        statusEl.textContent = '已停止所有Worker';
        statusEl.className = 'stopped';
        startBtn.disabled = false;
        stopBtn.disabled = true;
      }
    }

    // 页面关闭时自动清理
    window.addEventListener('beforeunload', () => stopAll());
  </script>
</body>
</html>
