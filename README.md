<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Mobile Multi YouTube Sync Player</title>
  <style>
    body{margin:0;font-family:sans-serif;background:#0f0f0f;color:#fff}
    header{padding:12px;text-align:center;font-weight:600}
    .controls{padding:12px;display:flex;gap:8px;flex-wrap:wrap}
    input{flex:1;padding:10px;border-radius:8px;border:none}
    button{padding:10px 14px;border:none;border-radius:8px;background:#ff0000;color:#fff;font-weight:600}
    button:disabled{opacity:.5}
    .grid{display:grid;grid-template-columns:1fr;gap:8px;padding:8px}
    @media(min-width:600px){.grid{grid-template-columns:1fr 1fr}}
    @media(min-width:900px){.grid{grid-template-columns:repeat(4,1fr)}}
    .player{aspect-ratio:16/9;background:#000}
    .status{padding:8px 12px;font-size:12px;opacity:.85}
    .error{color:#ff6b6b}
  </style>
</head>
<body>
<header>📱 Mobile YouTube Sync Player</header>

<div class="controls">
  <input id="videoInput" placeholder="Paste YouTube Video ID or URL" />
  <button id="addBtn">Add Video</button>
  <button id="loadBtn" disabled>Load</button>
  <button id="playBtn" disabled>Play</button>
  <button id="pauseBtn" disabled>Pause</button>
</div>
<div id="status" class="status">Loading YouTube API…</div>

<div id="grid" class="grid"></div>

<!-- YouTube Iframe API (hosted) -->
<script src="https://www.youtube.com/iframe_api" async></script>
<script>
'use strict';

/********************
 * State
 ********************/
let players = [];
let videoIds = [];
let apiReady = false;
let apiReadyResolver;
const apiReadyPromise = new Promise(r => apiReadyResolver = r);

/********************
 * DOM refs
 ********************/
const grid = document.getElementById('grid');
const input = document.getElementById('videoInput');
const addBtn = document.getElementById('addBtn');
const loadBtn = document.getElementById('loadBtn');
const playBtn = document.getElementById('playBtn');
const pauseBtn = document.getElementById('pauseBtn');
const statusEl = document.getElementById('status');

/********************
 * Helpers
 ********************/
function updateButtons(){
  loadBtn.disabled = !apiReady || videoIds.length === 0;
  playBtn.disabled = players.length === 0;
  pauseBtn.disabled = players.length === 0;
}

function setStatus(msg, isError=false){
  statusEl.textContent = msg;
  statusEl.classList.toggle('error', !!isError);
}

function safeDestroy(player){ try{ player.destroy(); }catch(e){} }

// Extract video ID from full URL or return raw ID
function normalizeVideoId(val){
  try{
    if(val.includes('youtube.com') || val.includes('youtu.be')){
      const url = new URL(val);
      return url.searchParams.get('v') || url.pathname.split('/').pop();
    }
  }catch(e){}
  return val;
}

/********************
 * YouTube API callback
 ********************/
window.onYouTubeIframeAPIReady = function(){
  apiReady = true;
  apiReadyResolver();
  setStatus('YouTube API ready');
  updateButtons();
};

/********************
 * Events
 ********************/
addBtn.addEventListener('click',()=>{
  const raw = input.value.trim();
  if(!raw) return;
  const id = normalizeVideoId(raw);
  if(!id){ setStatus('Invalid YouTube ID / URL', true); return; }
  videoIds.push(id);
  input.value='';
  updateButtons();
});

loadBtn.addEventListener('click', async ()=>{
  if(videoIds.length === 0) return;
  await apiReadyPromise; // HARD GUARANTEE

  players.forEach(safeDestroy);
  players = [];
  grid.innerHTML = '';
  setStatus('Loading players…');

  videoIds.forEach((id,i)=>{
    const div = document.createElement('div');
    div.className = 'player';
    div.id = 'player_'+i;
    grid.appendChild(div);

    const player = new YT.Player(div.id,{
      videoId: id,
      host: 'https://www.youtube.com', // IMPORTANT for error 153
      playerVars:{
        origin: window.location.origin,
        playsinline: 1,
        rel: 0,
        modestbranding: 1
      },
      events:{
        onReady:(e)=>{
          if(i>0) e.target.mute();
          updateButtons();
          setStatus('Players ready');
        },
        onError:(e)=>{
          // 150/153 = embedding not allowed / configuration error
          setStatus('Video cannot be embedded (Error '+e.data+'). Try another video.', true);
          console.error('YT Error', e.data, id);
        }
      }
    });
    players.push(player);
  });
});

playBtn.addEventListener('click', async ()=>{
  await apiReadyPromise;
  if(players.length === 0) return;
  const master = players[0];
  const t = typeof master.getCurrentTime === 'function' ? master.getCurrentTime() : 0;
  players.forEach(p=>{ try{ p.seekTo(t,true); p.playVideo(); }catch(e){} });
});

pauseBtn.addEventListener('click',()=>{
  players.forEach(p=>{ try{ p.pauseVideo(); }catch(e){} });
});

/********************
 * Tests (runtime assertions)
 ********************/
console.assert(grid instanceof HTMLElement,'Grid element missing');
console.assert(addBtn && loadBtn && playBtn && pauseBtn,'Control buttons missing');
console.assert(typeof apiReadyPromise.then === 'function','API promise not created');
console.assert(typeof normalizeVideoId('https://youtu.be/dQw4w9WgXcQ') === 'string','ID normalization failed');
</script>
</body>
</html>
