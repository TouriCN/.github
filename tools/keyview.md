# KeyView

<ClientOnly>
  <canvas id="kv-canvas" style="width:100%;height:520px;border:1px solid var(--vp-c-divider);border-radius:8px;margin:24px 0;display:block;"></canvas>
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

  // 核心数据：只留4个基础按键，没有多余配置
  const keys = [
    { code: 'KeyF', label: 'F', cnt: 0, color: '#ff3366' },
    { code: 'KeyG', label: 'G', cnt: 0, color: '#ff9f43' },
    { code: 'KeyH', label: 'H', cnt: 0, color: '#ffcc00' },
    { code: 'KeyJ', label: 'J', cnt: 0, color: '#00ff88' }
  ];
  const pressed = new Set();
  const bars = []; // 只保留上升条这一个可视化效果

  // 计算按键位置（居中排列，不会偏移）
  const updateKeyPositions = () => {
    const keyW = 50;
    const gap = 12;
    const totalW = keys.length * keyW + (keys.length - 1) * gap;
    let startX = (rect.width - totalW) / 2;
    const keyY = 520 - 200 - 25 - keyW; // 固定在按键区域，不受布局影响
    keys.forEach(key => {
      key.x = startX;
      key.y = keyY;
      key.w = keyW;
      key.h = keyW;
      startX += keyW + gap;
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

  // 绘制按键：样式完全贴合VitePress主题，没有多余效果
  function drawKeys() {
    const isDark = document.documentElement.classList.contains('dark');
    keys.forEach(key => {
      // 按键底色
      ctx.fillStyle = isDark ? 'var(--vp-c-bg-soft)' : '#f6f6f7';
      ctx.fillRect(key.x, key.y, key.w, key.h);
      // 边框：按下时高亮，平时用主题分割线颜色
      ctx.lineWidth = 2;
      ctx.strokeStyle = pressed.has(key.code) ? key.color : 'var(--vp-c-divider)';
      ctx.strokeRect(key.x, key.y, key.w, key.h);
      // 轻微缩放反馈，不晃眼
      if (pressed.has(key.code)) {
        ctx.save();
        ctx.translate(key.x + key.w/2, key.y + key.h/2);
        ctx.scale(0.96, 0.96);
        ctx.translate(-(key.x + key.w/2), -(key.y + key.h/2));
      }
      // 按键文字
      ctx.fillStyle = 'var(--vp-c-text-1)';
      ctx.font = 'bold 15px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillText(key.label, key.x + key.w/2, key.y + key.h/2 - 6);
      // 计数文字
      ctx.font = '10px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto';
      ctx.fillStyle = 'var(--vp-c-text-3)';
      ctx.fillText(key.cnt, key.x + key.w/2, key.y + key.h/2 + 10);
      if (pressed.has(key.code)) ctx.restore();
    });
  }

  // 绘制上升条：干净实色，没有拖尾模糊
  function drawBars() {
    for (let i = bars.length - 1; i >= 0; i--) {
      const bar = bars[i];
      bar.y -= 2.5; // 匀速上升
      // 实色填充，没有渐变模糊
      ctx.fillStyle = bar.color;
      ctx.fillRect(bar.x, 520 - 200 - bar.y - bar.height, bar.width, bar.height);
      // 移出可视区域后删除
      if (bar.y < -bar.height) bars.splice(i, 1);
    }
  }

  // 按键按下逻辑：只更新计数和上升条
  function handleDown(code) {
    if (pressed.has(code)) return;
    pressed.add(code);
    const key = keys.find(k => k.code === code);
    if (!key) return;
    key.cnt++;
    // 生成上升条，宽度和按键一致
    bars.push({
      x: key.x,
      y: 0,
      width: key.w,
      height: 2,
      color: key.color
    });
  }

  // 按键抬起逻辑
  function handleUp(code) {
    pressed.delete(code);
  }

  // 事件绑定：覆盖键盘/鼠标/触摸，没有多余逻辑
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

  canvas.addEventListener('mouseup', () => pressed.clear());
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
    if (keys.some(k => k.code === e.code)) handleDown(e.code);
  });

  document.addEventListener('keyup', (e) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)) return;
    if (keys.some(k => k.code === e.code)) handleUp(e.code);
  });

  window.addEventListener('blur', () => pressed.clear());

  // 动画循环：60fps稳定，没有多余计算
  function animate() {
    ctx.clearRect(0, 0, rect.width, 520);
    drawBars();
    drawKeys();
    requestAnimationFrame(animate);
  }
  animate();
};
</script>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
