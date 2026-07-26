# KeyView

<div id="kv-container">
  <div id="kv-main"></div>
  <div id="kv-keys"></div>
  <div id="kv-cfg">
    <div id="kv-cfg-top"></div>
    <div id="kv-cfg-bottom"></div>
  </div>
  <div class="kv-modal-overlay" id="kv-modal">
    <div class="kv-modal-box">
      <h3>添加按键</h3>
      <input type="text" id="kv-newKeyName" placeholder="输入按键名（如 L）" autocomplete="off">
      <div class="kv-select-group">
        <label>按键类型</label>
        <div class="kv-type-btns">
          <div class="kv-type-btn normal on" data-type="normal">普通键</div>
          <div class="kv-type-btn kps" data-type="kps">KPS键</div>
          <div class="kv-type-btn total" data-type="total">TOTAL键</div>
        </div>
      </div>
      <div class="kv-select-group">
        <label>绑定行</label>
        <div class="kv-row-btns" id="kv-rowSelectBtns"></div>
      </div>
      <div class="kv-actions">
        <button class="kv-btn cancel" id="kv-modalCancel">取消</button>
        <button class="kv-btn ok" id="kv-modalOk">确定</button>
      </div>
    </div>
  </div>
</div>

<style>
/* 全部使用 VitePress 官方主题变量，自动适配亮/暗色模式 */
#kv-container {
  position: relative;
  width: 100%;
  height: 520px;
  background: var(--vp-c-bg);
  border: 1px solid var(--vp-c-divider);
  border-radius: 8px;
  overflow: hidden;
  touch-action: manipulation;
  user-select: none;
  -webkit-user-select: none;
  margin: 24px 0;
}

#kv-container #kv-main { position: absolute; inset: 0; bottom: 200px; }
#kv-container #kv-keys {
  position: absolute; left: 0; right: 0; bottom: 200px;
  display: flex; flex-direction: column; align-items: center; justify-content: flex-end;
  gap: 14px; padding: 10px 0; pointer-events: none;
}

#kv-container .kv-row { display: flex; gap: 12px; pointer-events: auto; }
#kv-container .key {
  min-width: 50px; height: 50px; border: 2px solid var(--vp-c-divider); border-radius: 8px;
  font-weight: bold; display: flex; flex-direction: column; align-items: center; justify-content: center;
  font-size: 15px; background: var(--vp-c-bg-soft); color: var(--vp-c-text-1);
  transition: background-color 0.05s, transform 0.05s, border-color 0.15s; cursor: pointer; position: relative; z-index: 1;
}
#kv-container .key.pressed {
  background: var(--vp-c-brand-dimm); border-color: var(--vp-c-brand); transform: scale(0.96); z-index: 2;
}

#kv-container .info-key {
  background: var(--vp-c-bg-mute) !important; border-color: var(--vp-c-divider-light) !important;
  color: var(--vp-c-text-2) !important; cursor: default; min-width: 110px; pointer-events: none;
}
#kv-container .info-key .kv-val { font-size: 22px; font-weight: bold; }
#kv-container .kps-key .kv-val { color: #ff9f43; }
#kv-container .total-key .kv-val { color: #00b96b; }
#kv-container .kv-lbl { font-size: 9px; color: var(--vp-c-text-3); }

#kv-container #kv-cfg {
  position: absolute; bottom: 0; left: 0; right: 0; height: 200px;
  background: var(--vp-c-bg-soft); border-top: 1px solid var(--vp-c-divider);
  display: flex; flex-direction: column;
}
#kv-container #kv-cfg-top {
  flex: 1; min-height: 0; display: flex; align-items: flex-start; padding: 10px 12px; gap: 12px;
  overflow-y: auto; overflow-x: auto;
}
#kv-container #kv-cfg-bottom {
  height: 80px; display: flex; align-items: center; padding: 0 12px; gap: 16px;
  overflow-x: auto; overflow-y: hidden; flex-shrink: 0;
}

#kv-container .cfg-item {
  display: flex; flex-direction: column; align-items: center; min-width: 72px; max-width: 90px; gap: 3px;
  position: relative; padding: 6px 4px 4px; border: 1px solid var(--vp-c-divider); border-radius: 6px;
  background: var(--vp-c-bg-mute); flex-shrink: 0;
}
#kv-container .cfg-item .del-btn {
  position: absolute; top: -5px; right: -5px; width: 15px; height: 15px; border-radius: 50%;
  background: var(--vp-c-danger); color: var(--vp-c-white); font-size: 10px;
  display: flex; align-items: center; justify-content: center; cursor: pointer; border: none;
  z-index: 10; opacity: 0.7; line-height: 1;
}
#kv-container .cfg-item .del-btn:hover { opacity: 1; }
#kv-container .cfg-item .nm { font-size: 12px; color: var(--vp-c-text-1); font-weight: bold; }
#kv-container .cfg-item .row-btns { display: flex; gap: 2px; margin-bottom: 2px; overflow-x: auto; max-width: 100%; padding-bottom: 2px; scrollbar-width: none; }
#kv-container .cfg-item .row-btns::-webkit-scrollbar { display: none; }
#kv-container .cfg-item .row-btn {
  width: 16px; height: 16px; border-radius: 2px; border: 1px solid var(--vp-c-divider); cursor: pointer;
  background: var(--vp-c-bg-soft); flex-shrink: 0; display: flex; align-items: center; justify-content: center;
  font-size: 8px; color: var(--vp-c-text-2); transition: all 0.15s;
}
#kv-container .cfg-item .row-btn:hover { background: var(--vp-c-bg-mute); }
#kv-container .cfg-item .row-btn.on { border-color: var(--vp-c-brand); background: var(--vp-c-brand-dimm); color: var(--vp-c-brand-dark); }
#kv-container .cfg-item .span-btns { display: flex; gap: 1px; margin-bottom: 2px; }
#kv-container .cfg-item .span-btn {
  width: 14px; height: 14px; border-radius: 2px; border: 1px solid var(--vp-c-divider); cursor: pointer;
  font-size: 7px; color: var(--vp-c-text-3); display: flex; align-items: center; justify-content: center;
  background: var(--vp-c-bg-soft); transition: all 0.15s;
}
#kv-container .cfg-item .span-btn.on { border-color: var(--vp-c-brand); color: var(--vp-c-brand-dark); background: var(--vp-c-brand-dimm); }
#kv-container .cfg-item .row-info { font-size: 8px; color: var(--vp-c-text-3); text-align: center; white-space: nowrap; }

#kv-container .add-btns { display: flex; flex-direction: column; gap: 6px; margin-left: 12px; padding-left: 12px; border-left: 1px solid var(--vp-c-divider); align-self: flex-start; flex-shrink: 0; }
#kv-container .add-btn { padding: 8px 14px; border-radius: 6px; font-size: 12px; font-weight: bold; cursor: pointer; border: 2px solid var(--vp-c-brand); background: var(--vp-c-bg-soft); color: var(--vp-c-brand); white-space: nowrap; transition: opacity 0.15s; }
#kv-container .add-btn:hover { opacity: 0.8; }

#kv-container .row-config { display: flex; flex-direction: column; gap: 3px; min-width: 90px; padding: 4px 6px; border: 1px solid var(--vp-c-divider); border-radius: 4px; flex-shrink: 0; }
#kv-container .row-config .rc-label { font-size: 10px; color: var(--vp-c-text-2); margin-bottom: 1px; }
#kv-container .row-config .rc-row { display: flex; align-items: center; gap: 6px; }
#kv-container .row-config .rc-label-w { font-size: 9px; color: var(--vp-c-text-3); width: 18px; }
#kv-container .row-config input[type="range"] { width: 48px; height: 12px; }
#kv-container .row-config input[type="color"] { width: 36px; height: 14px; border: none; background: none; }

#kv-container .kv-bar { position: absolute; border-radius: 2px; pointer-events: none; will-change: height, bottom, opacity; min-height: 2px; }

/* 弹窗样式 */
#kv-container .kv-modal-overlay { display: none; position: fixed; inset: 0; background: rgba(0, 0, 0, 0.6); z-index: 99999; align-items: center; justify-content: center; }
#kv-container .kv-modal-overlay.show { display: flex; }
#kv-container .kv-modal-box {
  background: var(--vp-c-bg-elv); border: 1px solid var(--vp-c-divider); border-radius: 12px;
  padding: 20px; width: 320px; max-width: 90vw; box-shadow: var(--vp-shadow-3);
}
#kv-container .kv-modal-box h3 { color: var(--vp-c-brand); margin-bottom: 14px; font-size: 16px; text-align: center; }
#kv-container .kv-modal-box input[type="text"] { width: 100%; padding: 10px; border-radius: 6px; border: 1px solid var(--vp-c-divider); background: var(--vp-c-bg-soft); color: var(--vp-c-text-1); font-size: 14px; margin-bottom: 14px; outline: none; }
#kv-container .kv-modal-box input[type="text"]:focus { border-color: var(--vp-c-brand); }
#kv-container .kv-select-group { margin-bottom: 14px; }
#kv-container .kv-select-group > label { font-size: 10px; color: var(--vp-c-text-2); display: block; margin-bottom: 6px; }
#kv-container .kv-type-btns { display: flex; gap: 6px; }
#kv-container .kv-type-btn { flex: 1; padding: 8px; border-radius: 4px; border: 2px solid var(--vp-c-divider); cursor: pointer; font-size: 11px; color: var(--vp-c-text-2); text-align: center; background: var(--vp-c-bg-soft); transition: all 0.15s; }
#kv-container .kv-type-btn.on { border-color: var(--vp-c-brand); color: var(--vp-c-brand-dark); background: var(--vp-c-brand-dimm); }
#kv-container .kv-type-btn.kps.on { border-color: #ff9f43; background: rgba(255, 159, 67, 0.1); color: #ff9f43; }
#kv-container .kv-type-btn.total.on { border-color: #00b96b; background: rgba(0, 185, 107, 0.1); color: #00b96b; }
#kv-container .kv-row-btns { display: flex; gap: 6px; flex-wrap: wrap; }
#kv-container .kv-row-btns div { padding: 6px 10px; border-radius: 4px; border: 2px solid var(--vp-c-divider); cursor: pointer; font-size: 11px; color: var(--vp-c-text-2); background: var(--vp-c-bg-soft); transition: all 0.15s; }
#kv-container .kv-row-btns div.on { border-color: var(--vp-c-brand); color: var(--vp-c-brand-dark); background: var(--vp-c-brand-dimm); }
#kv-container .kv-actions { display: flex; gap: 10px; margin-top: 4px; }
#kv-container .kv-btn { flex: 1; padding: 10px; border-radius: 6px; font-size: 13px; font-weight: bold; cursor: pointer; border: none; transition: opacity 0.15s; }
#kv-container .kv-btn.cancel { background: var(--vp-c-bg-mute); color: var(--vp-c-text-1); }
#kv-container .kv-btn.ok { background: var(--vp-c-brand); color: var(--vp-c-white); }
#kv-container .kv-btn:hover { opacity: 0.85; }
</style>



# 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。