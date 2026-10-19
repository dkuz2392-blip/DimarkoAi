<!DOCTYPE html>
<html lang="ru"><head><meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<meta name="color-scheme" content="dark">
<title>Pulse AI офлайн</title>
<style>
:root{--bg:#090a14;--cd:#12141f;--cd2:#191c2a;--bd:#232738;--ac:#8b5cf6;--ac2:#6366f1;--tx:#f1f5f9;--mu:#8791a7}
*{box-sizing:border-box}[hidden]{display:none!important}
html,body{height:100%;margin:0}
body{background:var(--bg);color:var(--tx);font:15px/1.45 system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;display:flex;flex-direction:column;padding-top:env(safe-area-inset-top)}
button,select,textarea{font:inherit;color:inherit}
button,select{background:var(--cd2);border:1px solid var(--bd);border-radius:12px;padding:9px 14px;cursor:pointer}
button:active{transform:scale(.97)}button:disabled{opacity:.5}
.btn{background:linear-gradient(135deg,var(--ac),var(--ac2));border:0;font-weight:700;color:#fff}
header{padding:12px 14px;border-bottom:1px solid var(--bd);background:var(--cd);display:grid;gap:8px}
.row{display:flex;gap:8px;align-items:center}.row>*{min-width:0}.grow{flex:1}
h1{margin:0;font-size:18px}
.badge{font-size:11px;padding:3px 8px;border-radius:99px;background:var(--cd2);border:1px solid var(--bd);color:var(--mu)}
.badge.off{color:#34d399;border-color:#34d39966}
#st{font-size:12px;color:var(--mu);min-height:16px}
progress{width:100%;height:6px;accent-color:var(--ac)}
#log{flex:1;overflow-y:auto;padding:14px;display:flex;flex-direction:column;gap:8px}
.m{max-width:88%;padding:9px 13px;border-radius:16px;white-space:pre-wrap;word-break:break-word;background:var(--cd);border:1px solid var(--bd)}
.m.u{align-self:flex-end;background:linear-gradient(135deg,var(--ac),var(--ac2));border:0;color:#fff}
.empty{margin:auto;text-align:center;color:var(--mu);padding:24px}
form{display:flex;gap:8px;padding:10px 12px calc(10px + env(safe-area-inset-bottom));border-top:1px solid var(--bd);background:var(--cd)}
textarea{flex:1;resize:none;max-height:120px;background:var(--cd2);border:1px solid var(--bd);border-radius:16px;padding:10px 14px;outline:0;font-size:16px}
textarea:focus{border-color:var(--ac)}
</style></head>
<body>
<header>
 <div class="row"><h1 class="grow">⚡ Pulse AI</h1><span id="net" class="badge">онлайн</span></div>
 <div class="row">
  <select id="model" class="grow">
   <option value="Qwen2.5-0.5B-Instruct-q4f16_1-MLC">Лёгкая · 0.5B (~400 МБ)</option>
   <option value="Qwen2.5-1.5B-Instruct-q4f16_1-MLC" selected>Средняя · 1.5B (~1 ГБ)</option>
   <option value="Qwen2.5-3B-Instruct-q4f16_1-MLC">Умная · 3B (~2 ГБ)</option>
  </select>
  <button id="load" class="btn">Загрузить</button>
  <button id="clear">Очистить</button>
 </div>
 <progress id="pg" max="1" value="0" hidden></progress>
 <div id="st">Выберите модель и нажмите «Загрузить». Первый раз нужен интернет.</div>
</header>
<div id="log"><div class="empty">Здесь появится диалог.<br>Модель работает на вашем устройстве, тексты никуда не отправляются.</div></div>
<form id="f">
 <textarea id="in" rows="1" placeholder="Сообщение" disabled></textarea>
 <button id="go" class="btn" disabled>➤</button>
</form>

<script type="module">
const $=i=>document.getElementById(i);
const LIB='https://cdn.jsdelivr.net/npm/@mlc-ai/web-llm@0.2.79/+esm';
const SYS='Ты — Pulse AI, дружелюбный помощник. Отвечай по-русски, кратко и по делу. Если не уверен, так и скажи.';
const store={get(k){try{return localStorage.getItem(k)}catch(e){return null}},set(k,v){try{localStorage.setItem(k,v)}catch(e){}}};
let lib=null,engine=null,busy=false,hist=[];
try{hist=JSON.parse(store.get('pai_hist')||'[]')}catch(e){}
const st=t=>{$('st').textContent=t};
const net=()=>{const o=navigator.onLine;$('net').textContent=o?'онлайн':'офлайн';$('net').className='badge'+(o?'':' off')};
addEventListener('online',net);addEventListener('offline',net);net();

function add(cls,text){const l=$('log');const e=l.querySelector('.empty');if(e)e.remove();const d=document.createElement('div');d.className='m '+cls;d.textContent=text;l.append(d);l.scrollTop=l.scrollHeight;return d}
hist.forEach(m=>add(m.role==='user'?'u':'a',m.content));
const save=()=>store.set('pai_hist',JSON.stringify(hist.slice(-40)));

const saved=store.get('pai_model');if(saved)$('model').value=saved;

$('load').onclick=async()=>{
 if(busy)return;
 if(!navigator.gpu){st('Нужен браузер с WebGPU (Chrome 121+, Android 12+). В обычной оболочке APK он часто недоступен.');return}
 busy=true;$('load').disabled=true;$('in').disabled=true;$('go').disabled=true;$('pg').hidden=false;
 try{
  st('Подключаю движок...');
  if(!lib)lib=await import(LIB);
  if(engine){await engine.unload();engine=null}
  const id=$('model').value;
  engine=await lib.CreateMLCEngine(id,{initProgressCallback:r=>{$('pg').value=r.progress||0;st(r.text||'Загрузка...')}});
  store.set('pai_model',id);
  st('Готово. Можно писать, интернет больше не нужен.');
  $('in').disabled=false;$('go').disabled=false;$('in').focus();
 }catch(e){st('Ошибка: '+(e&&e.message||e)+(navigator.onLine?'':' (нет интернета: движок или модель ещё не скачаны)'))}
 $('pg').hidden=true;$('load').disabled=false;busy=false;
};

$('f').onsubmit=async ev=>{
 ev.preventDefault();
 if(busy&&engine){engine.interruptGenerate();return}
 const q=$('in').value.trim();if(!q||!engine)return;
 $('in').value='';busy=true;$('go').textContent='■';
 add('u',q);const out=add('a','...');
 let acc='';
 try{
  const it=await engine.chat.completions.create({
   messages:[{role:'system',content:SYS},...hist.slice(-8),{role:'user',content:q}],
   stream:true,temperature:0.7,max_tokens:700});
  for await(const c of it){
   acc+=(c.choices[0]&&c.choices[0].delta&&c.choices[0].delta.content)||'';
   out.textContent=acc||'...';$('log').scrollTop=$('log').scrollHeight;
  }
 }catch(e){acc=acc||('Ошибка: '+(e&&e.message||e));out.textContent=acc}
 if(acc){hist.push({role:'user',content:q},{role:'assistant',content:acc});save()}
 busy=false;$('go').textContent='➤';$('in').focus();
};
$('in').addEventListener('keydown',e=>{if(e.key==='Enter'&&!e.shiftKey&&matchMedia('(min-width:700px)').matches){e.preventDefault();$('f').requestSubmit()}});
$('clear').onclick=()=>{if(busy&&engine)engine.interruptGenerate();hist=[];save();$('log').innerHTML='<div class="empty">Диалог очищен.</div>'};

if('serviceWorker' in navigator&&location.protocol.startsWith('http'))navigator.serviceWorker.register('sw.js').catch(()=>{});
</script>
</body></html>
