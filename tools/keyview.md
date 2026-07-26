# KeyView
<div id="kv-container">
  <div id="kv-main"></div>
  <div id="kv-keys"></div>
  <div id="kv-cfg">
    <div id="kv-cfg-top"></div>
    <div id="kv-cfg-bottom"></div>
  </div>
  <!-- 添加按键弹窗 -->
  <div class="kv-modal-overlay" id="kv-modal">
    <div class="kv-modal-box">
      <h3>添加按键</h3>
      <input type="text" id="kv-newKeyName" placeholder="输入按键名（如 L）">
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
#kv-container { width:100%; height:520px; background:var(--vp-c-bg); border:1px solid var(--vp-c-divider); border-radius:8px; position:relative; overflow:hidden; margin:24px 0; }
#kv-container #kv-main { position:absolute; inset:0; bottom:200px; }
#kv-container #kv-keys { position:absolute; left:0; right:0; bottom:200px; display:flex; flex-direction:column; align-items:center; gap:14px; padding:10px 0; pointer-events:none; }
#kv-container .kv-row { display:flex; gap:12px; pointer-events:auto; }
#kv-container .key { min-width:50px; height:50px; border:2px solid var(--vp-c-divider); border-radius:8px; font-weight:bold; display:flex; flex-direction:column; align-items:center; justify-content:center; font-size:15px; background:var(--vp-c-bg-soft); color:var(--vp-c-text-1); cursor:pointer; transition:all 0.1s; }
#kv-container .key.pressed { background:var(--vp-c-brand-dimm); border-color:var(--vp-c-brand); transform:scale(0.96); }
#kv-container .info-key { background:var(--vp-c-bg-mute) !important; cursor:default; min-width:110px; }
#kv-container .kps-key .kv-val { color:#ff9f43; font-size:22px; }
#kv-container .total-key .kv-val { color:#00b96b; font-size:22px; }
#kv-container #kv-cfg { position:absolute; bottom:0; left:0; right:0; height:200px; background:var(--vp-c-bg-soft); border-top:1px solid var(--vp-c-divider); display:flex; flex-direction:column; }
#kv-container #kv-cfg-top { flex:1; display:flex; gap:12px; padding:10px; overflow:auto; }
#kv-container #kv-cfg-bottom { height:80px; display:flex; gap:16px; padding:0 12px; align-items:center; overflow:auto; }
#kv-container .cfg-item { min-width:72px; padding:6px; border:1px solid var(--vp-c-divider); border-radius:6px; background:var(--vp-c-bg-mute); display:flex; flex-direction:column; align-items:center; gap:3px; position:relative; }
#kv-container .del-btn { position:absolute; top:-5px; right:-5px; width:15px; height:15px; border-radius:50%; background:var(--vp-c-danger); color:white; border:none; cursor:pointer; font-size:10px; }
#kv-container .row-btn { width:16px; height:16px; border-radius:2px; border:1px solid var(--vp-c-divider); background:var(--vp-c-bg-soft); cursor:pointer; font-size:8px; }
#kv-container .row-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .span-btn { width:14px; height:14px; border-radius:2px; border:1px solid var(--vp-c-divider); background:var(--vp-c-bg-soft); cursor:pointer; font-size:7px; }
#kv-container .span-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .add-btn { padding:8px 14px; border:2px solid var(--vp-c-brand); background:var(--vp-c-bg-soft); color:var(--vp-c-brand); border-radius:6px; cursor:pointer; }
#kv-container .kv-modal-overlay { display:none; position:fixed; inset:0; background:rgba(0,0,0,0.6); z-index:99999; align-items:center; justify-content:center; }
#kv-container .kv-modal-overlay.show { display:flex; }
#kv-container .kv-modal-box { background:var(--vp-c-bg-elv); border:1px solid var(--vp-c-divider); border-radius:12px; padding:20px; width:320px; } /* 之前这里的lv是笔误，已修正 */
#kv-container .kv-type-btn { flex:1; padding:8px; border:2px solid var(--vp-c-divider); border-radius:4px; cursor:pointer; font-size:11px; text-align:center; }
#kv-container .kv-type-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .kv-btn { flex:1; padding:10px; border:none; border-radius:6px; cursor:pointer; font-weight:bold; }
#kv-container .kv-btn.cancel { background:var(--vp-c-bg-mute); }
#kv-container .kv-btn.ok { background:var(--vp-c-brand); color:white; }
#kv-container .kv-bar { position:absolute; border-radius:2px; pointer-events:none; min-height:2px; }
</style>

<script>
// 核心修正：删除顶层return，改用条件包裹所有代码
// Node构建阶段window不存在，里面的代码完全不会被解析执行，彻底避开Vue编译器报错
if (typeof window !== 'undefined') {
  window.addEventListener('DOMContentLoaded', () => {
    const SAVE_KEY = 'keyview_page';
    const BAR_SPEED = 300;
    const ROWS = [
      { id: 0, width: 52, color: '#ff3366' },
      { id: 1, width: 40, color: '#ff9f43' },
      { id: 2, width: 28, color: '#ffcc00' },
      { id: 3, width: 16, color: '#00ff88' }
    ];
    let keys = [
      { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal', span: 1 },
      { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal', span: 1 },
      { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal', span: 1 },
      { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal', span: 1 },
      { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps', span: 2 },
      { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total', span: 2 }
    ];
    let nextKeyId = 7, pressedKeys = new Set(), activeBars = {}, barId = 0, kpsRecords = [], firstRowBottom = 0, mainRect = null;
    const $ = sel => document.querySelector(`#kv-container ${sel}`);
    const $$ = sel => document.querySelectorAll(`#kv-container ${sel}`);
    
    const saveConfig = () => { try { localStorage.setItem(SAVE_KEY, JSON.stringify({ keys, nextKeyId, ROWS })); } catch {} };
    const loadConfig = () => { try { const d = JSON.parse(localStorage.getItem(SAVE_KEY)); d && (keys = d.keys || keys, nextKeyId = d.nextKeyId || nextKeyId, ROWS.splice(0, ROWS.length, ...(d.ROWS || ROWS))); } catch {} };
    
    const renderKeys = () => {
      const c = $('#kv-keys'); c.innerHTML = '';
      const rm = {}; keys.forEach(k => (rm[k.rowId] ||= []).push(k));
      Object.values(rm).sort((a,b)=>a[0].rowId-b[0].rowId).forEach(rk => {
        const rd = document.createElement('div'); rd.className = 'kv-row';
        rk.forEach(k => {
          const kd = document.createElement('div'); kd.className = 'key'; kd.dataset.id = k.id; kd.dataset.code = k.code; kd.style.width = `${50 + (k.span-1)*56}px`;
          if (k.type === 'kps') { kd.innerHTML = '<span class="kv-lbl">KPS</span><span class="kv-val">0</span>'; kd.classList.add('info-key','kps-key'); }
          else if (k.type === 'total') { kd.innerHTML = '<span class="kv-lbl">TOTAL</span><span class="kv-val">0</span>'; kd.classList.add('info-key','total-key'); }
          else { kd.innerHTML = `${k.label}<span class="kv-cnt">${k.cnt}</span>`; kd.addEventListener('mousedown', ()=>handleKeyDown(k.code)); kd.addEventListener('mouseup', ()=>handleKeyUp(k.code)); kd.addEventListener('touchstart', e=>{e.preventDefault();handleKeyDown(k.code)}); kd.addEventListener('touchend', e=>{e.preventDefault();handleKeyUp(k.code)}); }
          rd.appendChild(kd);
        }); c.appendChild(rd);
      }); updateInfo();
      const m = $('#kv-main'); mainRect = m.getBoundingClientRect();
      const fk = $('.key:not(.info-key)'); if (fk) { let el=fk,t=0; while(el&&el!==document.body){t+=el.offsetTop||0;el=el.offsetParent;} firstRowBottom = mainRect.height - t; }
    };
    
    const renderCfg = () => {
      const t = $('#kv-cfg-top'); t.innerHTML = '';
      keys.forEach(k => {
        const i = document.createElement('div'); i.className = 'cfg-item'; i.innerHTML = `<button class="del-btn" data-id="${k.id}">×</button><span class="nm">${k.label}</span><div class="row-btns">${ROWS.map(r=>`<div class="row-btn ${k.rowId===r.id?'on':''}" data-id="${k.id}" data-rid="${r.id}">${r.id+1}</div>`).join('')}</div><div class="span-btns">${[1,2,3,4].map(s=>`<div class="span-btn ${k.span===s?'on':''}" data-id="${k.id}" data-span="${s}">${s}</div>`).join('')}</div>`;
        t.appendChild(i);
      });
      t.querySelectorAll('.del-btn').forEach(b=>b.addEventListener('click',e=>{keys=keys.filter(k=>k.id!==+e.target.dataset.id);saveConfig();renderKeys();renderCfg();}));
      t.querySelectorAll('.row-btn').forEach(b=>b.addEventListener('click',e=>{const id=+e.target.dataset.id,rid=+e.target.dataset.rid;keys.forEach(k=>k.id===id&&(k.rowId=rid));saveConfig();renderKeys();renderCfg();}));
      const ab = document.createElement('div'); ab.className='add-btns'; ab.innerHTML='<button class="add-btn">+ 键</button>'; ab.querySelector('.add-btn').addEventListener('click', openModal); t.appendChild(ab);
    };
    
    const handleKeyDown = code => {
      if (pressedKeys.has(code)) return; pressedKeys.add(code);
      $$(`.key[data-code="${code}"]`).forEach(el=>el.classList.add('pressed'));
      keys.filter(k=>k.code===code&&k.type==='normal').forEach(k=>{k.cnt++;$(`.key[data-id="${k.id}"] .kv-cnt`).innerText=k.cnt;spawnBar(k.id);});
      kpsRecords.push(Date.now()); saveConfig();
    };
    
    const handleKeyUp = code => { if(!pressedKeys.has(code))return; pressedKeys.delete(code); $$(`.key[data-code="${code}"]`).forEach(el=>el.classList.remove('pressed')); };
    
    const spawnBar = keyId => {
      const k=keys.find(x=>x.id===keyId), r=ROWS.find(r=>r.id===k.rowId), ke=$(`.key[data-id="${keyId}"]`);
      if(!k||!r||!ke||!mainRect)return; const kr=ke.getBoundingClientRect();
      const b=document.createElement('div'); b.className='kv-bar'; b.style.cssText=`width:${r.width}px;height:2px;background:${r.color};left:${kr.left+kr.width/2-r.width/2-mainRect.left}px;bottom:${firstRowBottom}px;opacity:1;`;
      $('#kv-main').appendChild(b); activeBars[++barId]={el:b,y:firstRowBottom,speed:BAR_SPEED}; animateBars();
    };
    
    const animateBars = () => { let ha=false; for(const id in activeBars){const b=activeBars[id];b.y+=b.speed*0.016;b.el.style.bottom=`${b.y}px`;b.el.style.opacity=Math.max(0,1-(b.y-firstRowBottom)/200);if(b.y<firstRowBottom+200)ha=true;else{b.el.remove();delete activeBars[id];}} ha&&requestAnimationFrame(animateBars); };
    
    const updateInfo = () => { const n=Date.now();kpsRecords=kpsRecords.filter(t=>n-t<1000);const kps=kpsRecords.length,total=keys.reduce((s,k)=>s+(k.cnt||0),0);$$('.kps-key .kv-val').forEach(el=>el.innerText=kps);$$('.total-key .kv-val').forEach(el=>el.innerText=total); };
    
    const openModal = () => { const rb=$('#kv-rowSelectBtns'); rb.innerHTML=ROWS.map(r=>`<div class="kv-row-btns" data-rid="${r.id}">第${r.id+1}行</div>`).join(''); rb.querySelectorAll('div').forEach(b=>b.addEventListener('click',()=>{rb.querySelectorAll('div').forEach(d=>d.classList.remove('on'));b.classList.add('on');})); rb.querySelector('div').classList.add('on'); $$('.kv-type-btn').forEach(b=>b.addEventListener('click',()=>{$$('.kv-type-btn').forEach(d=>d.classList.remove('on'));b.classList.add('on');})); $('#kv-modal').classList.add('show'); $('#kv-newKeyName').focus(); };
    
    $('#kv-modalOk').addEventListener('click',()=>{const l=$('#kv-newKeyName').value.trim()||'N',t=$('.kv-type-btn.on').dataset.type,rid=+$('.kv-row-btns.on').dataset.rid,co=t==='kps'?`KPS_${nextKeyId}`:t==='total'?`TOTAL_${nextKeyId}`:`Key${l.toUpperCase()}`; keys.push({id:nextKeyId++,label:l,code:co,rowId:rid,cnt:0,type:t,span:1}); saveConfig();renderKeys();renderCfg();$('#kv-modal').classList.remove('show');});
    $('#kv-modalCancel').addEventListener('click',()=>$('#kv-modal').classList.remove('show'));
    
    window.addEventListener('keydown',e=>{if($('#kv-modal').classList.contains('show')){if(e.key==='Escape')$('#kv-modal').classList.remove('show');return;} const ii=['INPUT','TEXTAREA'].includes(document.activeElement.tagName);if(!ii)e.preventDefault();handleKeyDown(e.code);});
    window.addEventListener('keyup',e=>{if($('#kv-modal').classList.contains('show'))return;const ii=['INPUT','TEXTAREA'].includes(document.activeElement.tagName);if(!ii)e.preventDefault();handleKeyUp(e.code);});
    
    loadConfig(); renderKeys(); renderCfg(); setInterval(updateInfo,200);
    window.addEventListener('resize',()=>{const m=$('#kv-main');mainRect=m.getBoundingClientRect();const fk=$('.key:not(.info-key)');if(fk){let el=fk,t=0;while(el&&el!==document.body){t+=el.offsetTop||0;el=el.offsetParent;}firstRowBottom=mainRect.height-t;}});
  });
}
</script>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
