# KeyView
<ClientOnly>
  <canvas id="kv-canvas" style="width:100%;height:520px;border:1px solid var(--vp-c-divider);border-radius:8px;margin:24px 0;display:block;"></canvas>
  <!-- 音效开关 -->
  <button id="sound-toggle" style="position:absolute;bottom:34px;right:10px;width:30px;height:30px;border-radius:50%;border:1px solid var(--vp-c-divider);background:var(--vp-c-bg-soft);cursor:pointer;z-index:100;font-size:12px;">🔇</button>
</ClientOnly>

<script>
window.onload = function() {
  const canvas = document.getElementById('kv-canvas');
  const container = canvas.parentElement;
  if (!canvas || !container) return;

  // 适配高清屏
  const dpr = window.devicePixelRatio || 1;
  const rect = container.getBoundingClientRect();
  canvas.width = rect.width * dpr;
  canvas.height = 520 * dpr;
  canvas.style.width = `${rect.width}px`;
  canvas.style.height = '520px';
  const ctx = canvas.getContext('2d');
  ctx.scale(dpr, dpr);

  // 基础配置
  const keys = [
    { code: 'KeyF', label: 'F', cnt: 0, color: '#ff3366', x: 0, y: 0, w: 50, h: 50 },
    { code: 'KeyG', label: 'G', cnt: 0, color: '#ff9f43', x: 0, y: 0, w: 50, h: 50 },
    { code: 'KeyH', label: 'H', cnt: 0, color: '#ffcc00', x: 0, y: 0, w: 50, h: 50 },
    { code: 'KeyJ', label: 'J', cnt: 0, color: '#00ff88', x: 0, y: 0, w: 50, h: 50 }
  ];
  const pressed = new Set();
  let soundOn = false;
  const particles = [];
  const bars = [];
  const shockwaves = []; // 冲击波效果
  let gridOffset = 0; // 背景网格流动

  // 音效（和之前一样，base64编码，不用额外文件）
  const audioContext = new (window.AudioContext || window.webkitAudioContext)();
  const playSound = () => {
    if (!soundOn) return;
    const oscillator = audioContext.createOscillator();
    const gainNode = audioContext.createGain();
    oscillator.connect(gainNode);
    gainNode.connect(audioContext.destination);
    oscillator.frequency.value = 800;
    oscillator.type = 'sine';
    gainNode.gain.setValueAtTime(0.1, audioContext.currentTime);
    gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.1);
    oscillator.start(audioContext.currentTime);
    oscillator.stop(audioContext.currentTime + 0.1);
  };

  // 音效开关
  const soundToggle = document.getElementById('sound-toggle');
  soundToggle.addEventListener('click', () => {
    soundOn = !soundOn;
    soundToggle.textContent = soundOn ? '🔊' : '🔇';
    if (soundOn && audioContext.state === 'suspended') audioContext.resume();
  });

  // 计算按键位置（居中排列）
  const updateKeyPositions = () => {
    const totalWidth = keys.length * 50 + (keys.length - 1) * 12;
    let startX = (rect.width - totalWidth) / 2;
    const keyY = 520 - 200 - 25 - 50; // 距离底部200px+按键高度一半
    keys.forEach(key => {
      key.x = startX;
      key.y = keyY;
      startX += 50 + 12;
    });
  };
  updateKeyPositions();
  window.addEventListener('resize', () => {
    const newRect = container.getBoundingClientRect();
    canvas.width = newRect.width * dpr;
    canvas.height = 520 * dpr;
    canvas.style.width = `${newRect.width}px`;
    ctx.scale(dpr, dpr);
    updateKeyPositions();
  });

  // ===== 花活1：背景流动网格 =====
  function drawGrid() {
    const isDark = document.documentElement.classList.contains('dark');
    ctx.strokeStyle = isDark ? 'rgba(255,255,255,0.03)' : 'rgba(0,0,0,0.03)';
    ctx.lineWidth = 1;
    gridOffset += 0.5;
    const gridSize = 20;
    // 横向线
    for (let y = gridOffset % gridSize; y < 520; y += gridSize) {
      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(rect.width, y);
      ctx.stroke();
    }
    // 纵向线
    for (let x = 0; x < rect.width; x += gridSize) {
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, 520);
      ctx.stroke();
    }
  }

  // ===== 花活2：霓虹按键 =====
  function drawKeys() {
    const isDark = document.documentElement.classList.contains('dark');
    keys.forEach(key => {
      // 按键底色
      ctx.fillStyle = isDark ? '#242424' : '#f6f6f7';
      ctx.fillRect(key.x, key.y, key.w, key.h);
      // 边框
      ctx.lineWidth = 2;
      ctx.strokeStyle = pressed.has(key.code) ? key.color : (isDark ? '#333' : '#e2e8f0');
      ctx.strokeRect(key.x, key.y, key.w, key.h);
      // 霓虹发光（按下时）
      if (pressed.has(key.code)) {
        ctx.shadowColor = key.color;
        ctx.shadowBlur = 15;
        ctx.strokeRect(key.x, key.y, key.w, key.h);
        ctx.shadowBlur = 0;
      }
      // 文字
      ctx.fillStyle = isDark ? '#fff' : '#1a202c';
      ctx.font = 'bold 15px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillText(key.label, key.x + key.w/2, key.y + key.h/2 - 6);
      // 计数
      ctx.font = '10px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto';
      ctx.fillStyle = isDark ? '#64748b' : '#64748b';
      ctx.fillText(key.cnt, key.x + key.w/2, key.y + key.h/2 + 10);
    });
  }

  // ===== 花活3：拖尾上升条 =====
  function drawBars() {
    for (let i = bars.length - 1; i >= 0; i--) {
      const bar = bars[i];
      bar.y -= 3;
      bar.alpha -= 0.003;
      bar.blur += 0.05;
      // 渐变拖尾
      const gradient = ctx.createLinearGradient(bar.x, 520 - 200 - bar.y, bar.x, 520 - 200 - bar.y - bar.height);
      gradient.addColorStop(0, bar.color);
      gradient.addColorStop(1, 'transparent');
      ctx.fillStyle = gradient;
      ctx.filter = `blur(${bar.blur}px)`;
      ctx.fillRect(bar.x, 520 - 200 - bar.y - bar.height, bar.width, bar.height);
      ctx.filter = 'none';
      // 移除不可见的条
      if (bar.y < -bar.height || bar.alpha <= 0) {
        bars.splice(i, 1);
      }
    }
  }

  // ===== 花活4：粒子喷射 =====
  function drawParticles() {
    for (let i = particles.length - 1; i >= 0; i--) {
      const p = particles[i];
      p.x += p.vx;
      p.y += p.vy;
      p.alpha -= 0.01;
      p.radius *= 0.98;
      ctx.fillStyle = `rgba(${hexToRgb(p.color)}, ${p.alpha})`;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
      ctx.fill();
      if (p.alpha <= 0 || p.radius <= 0.1) {
        particles.splice(i, 1);
      }
    }
  }

  // ===== 花活5：按键冲击波 =====
  function drawShockwaves() {
    for (let i = shockwaves.length - 1; i >= 0; i--) {
      const sw = shockwaves[i];
      sw.radius += 2;
      sw.alpha -= 0.02;
      ctx.strokeStyle = `rgba(${hexToRgb(sw.color)}, ${sw.alpha})`;
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.arc(sw.x, sw.y, sw.radius, 0, Math.PI * 2);
      ctx.stroke();
      if (sw.alpha <= 0) {
        shockwaves.splice(i, 1);
      }
    }
  }

  // 辅助函数：十六进制转RGB
  function hexToRgb(hex) {
    const r = parseInt(hex.slice(1,3), 16);
    const g = parseInt(hex.slice(3,5), 16);
    const b = parseInt(hex.slice(5,7), 16);
    return `${r},${g},${b}`;
  }

  // 按键按下逻辑
  function handleDown(code) {
    if (pressed.has(code)) return;
    pressed.add(code);
    playSound();
    const key = keys.find(k => k.code === code);
    if (!key) return;

    // 生成粒子
    for (let i = 0; i < 8; i++) {
      const angle = Math.random() * Math.PI * 2;
      const speed = 1 + Math.random() * 2;
      particles.push({
        x: key.x + key.w/2,
        y: key.y + key.h/2,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        radius: 3 + Math.random() * 2,
        color: key.color,
        alpha: 1
      });
    }

    // 生成上升条
    bars.push({
      x: key.x + (key.w - key.w)/2, // 和按键同宽
      y: 0,
      width: key.w,
      height: 2,
      color: key.color,
      alpha: 1,
      blur: 0
    });

    // 生成冲击波
    shockwaves.push({
      x: key.x + key.w/2,
      y: key.y + key.h/2,
      radius: 20,
      color: key.color,
      alpha: 0.8
    });

    // 更新计数
    key.cnt++;
  }

  // 按键抬起逻辑
  function handleUp(code) {
    pressed.delete(code);
  }

  // 事件绑定
  canvas.addEventListener('mousedown', (e) => {
    const rect = canvas.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    keys.forEach(key => {
      if (x >= key.x && x <= key.x + key.w && y >= key.y && y <= key.y + key.h) {
        handleDown(key.code);
      }
    });
  });

  canvas.addEventListener('mouseup', () => {
    pressed.clear(); // 鼠标抬起清空所有按下状态
  });

  canvas.addEventListener('touchstart', (e) => {
    e.preventDefault();
    const rect = canvas.getBoundingClientRect();
    const touch = e.touches[0];
    const x = touch.clientX - rect.left;
    const y = touch.clientY - rect.top;
    keys.forEach(key => {
      if (x >= key.x && x <= key.x + key.w && y >= key.y && y <= key.y + key.h) {
        handleDown(key.code);
      }
    });
  }, { passive: false });

  canvas.addEventListener('touchend', (e) => {
    e.preventDefault();
    pressed.clear();
  }, { passive: false });

  document.addEventListener('keydown', (e) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)) return;
    if (keys.some(k => k.code === e.code)) {
      handleDown(e.code);
    }
  });

  document.addEventListener('keyup', (e) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)) return;
    if (keys.some(k => k.code === e.code)) {
      handleUp(e.code);
    }
  });

  window.addEventListener('blur', () => {
    pressed.clear();
  });

  // 动画循环
  function animate() {
    ctx.clearRect(0, 0, rect.width, 520);
    drawGrid(); // 背景网格
    drawBars(); // 上升条
    drawParticles(); // 粒子
    drawShockwaves(); // 冲击波
    drawKeys(); // 按键
    requestAnimationFrame(animate);
  }
  animate();
};
</script>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
