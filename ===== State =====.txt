// ===== State =====
let pc = null;            // RTCPeerConnection
let dataChannel = null;   // Direct P2P channel
let myName = '';
let username = 'User';
let messages = [];

// ===== Startup =====
window.onload = () => {
  const saved = localStorage.getItem('p2p_name');
  if (saved) {
    document.getElementById('username').value = saved;
  }
};

function startApp() {
  const name = document.getElementById('username').value.trim();
  if (!name) { alert('Naam daalein!'); return; }

  myName = name;
  username = name;
  localStorage.setItem('p2p_name', name);

  document.getElementById('my-name').textContent = name;
  document.getElementById('my-avatar').textContent = name[0].toUpperCase();

  document.getElementById('setup-screen').classList.remove('active');
  document.getElementById('chat-screen').classList.add('active');

  loadHistory();
  initPeerConnection();
}

// ===== WebRTC Setup =====
function initPeerConnection() {
  const config = {
    iceServers: [
      // Bina internet ke liye ye empty rakhein
      // Same WiFi par direct connect hoga
      { urls: 'stun:stun.l.google.com:19302' } // Internet wale case ke liye
    ]
  };

  pc = new RTCPeerConnection(config);

  pc.onicecandidate = (e) => {
    if (e.candidate) {
      // Local network candidates automatically handle honge
      // Manual exchange mode mein offer/answer mein include karna hoga
    }
  };

  pc.onconnectionstatechange = () => {
    updateStatus(pc.connectionState);
  };

  // Agar doosra device offer banaye to ye trigger hoga
  pc.ondatachannel = (e) => {
    dataChannel = e.channel;
    setupDataChannel();
  };
}

// ===== Data Channel =====
function setupDataChannel() {
  dataChannel.onopen = () => {
    updateStatus('connected');
    addSystemMsg('✅ Connected! Ab messages bhej sakte hain.');
    // Handshake - apna naam bhejein
    dataChannel.send(JSON.stringify({ type: 'name', name: myName }));
  };

  dataChannel.onmessage = (e) => {
    const data = JSON.parse(e.data);
    if (data.type === 'name') {
      document.getElementById('my-name').textContent = `${myName} ↔ ${data.name}`;
    } else if (data.type === 'msg') {
      addMessage(data.text, 'received');
    }
  };

  dataChannel.onclose = () => updateStatus('disconnected');
}

function updateStatus(state) {
  const el = document.getElementById('conn-status');
  if (state === 'connected') {
    el.textContent = '● Connected';
    el.classList.add('online');
  } else if (state === 'connecting') {
    el.textContent = '● Connecting...';
    el.classList.remove('online');
  } else {
    el.textContent = '● Offline';
    el.classList.remove('online');
  }
}

// ===== Offer / Answer Exchange (Manual - Bina Server) =====
async function createOffer() {
  if (!pc) initPeerConnection();

  dataChannel = pc.createDataChannel('chat');
  setupDataChannel();

  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);

  // ICE candidates collect karne ka wait
  await waitForIce(pc);

  const code = btoa(JSON.stringify(pc.localDescription));
  document.getElementById('offer-code').value = code;
  document.getElementById('offer-box').classList.remove('hidden');
}

function waitForIce(pc) {
  return new Promise((resolve) => {
    if (pc.iceGatheringState === 'complete') return resolve();
    const check = () => {
      if (pc.iceGatheringState === 'complete') {
        pc.removeEventListener('icegatheringstatechange', check);
        resolve();
      }
    };
    pc.addEventListener('icegatheringstatechange', check);
    setTimeout(resolve, 3000); // 3 sec max wait
  });
}

async function acceptAnswer() {
  const code = document.getElementById('answer-input').value.trim();
  if (!code) { alert('Answer code paste karein'); return; }
  try {
    const answer = JSON.parse(atob(code));
    await pc.setRemoteDescription(answer);
    addSystemMsg('⏳ Connecting...');
  } catch (e) {
    alert('Invalid code: ' + e.message);
  }
}

function showJoinInput() {
  document.getElementById('join-box').classList.remove('hidden');
  document.getElementById('offer-box').classList.add('hidden');
}

async function joinOffer() {
  const code = document.getElementById('join-offer').value.trim();
  if (!code) { alert('Offer code paste karein'); return; }
  if (!pc) initPeerConnection();

  try {
    const offer = JSON.parse(atob(code));
    await pc.setRemoteDescription(offer);
    const answer = await pc.createAnswer();
    await pc.setLocalDescription(answer);
    await waitForIce(pc);

    const answerCode = btoa(JSON.stringify(pc.localDescription));
    document.getElementById('my-answer').value = answerCode;
    document.getElementById('answer-out').classList.remove('hidden');
  } catch (e) {
    alert('Invalid offer: ' + e.message);
  }
}

// ===== UI Helpers =====
function copyCode(id) {
  const el = document.getElementById(id);
  el.select();
  navigator.clipboard.writeText(el.value);
  alert('✅ Copy ho gaya! Doosre device ko bhejein.');
}

function showPairing() {
  document.getElementById('pairing-panel').classList.remove('hidden');
}
function hidePairing() {
  document.getElementById('pairing-panel').classList.add('hidden');
}

// ===== Messages =====
function sendMessage() {
  const input = document.getElementById('msg-input');
  const text = input.value.trim();
  if (!text) return;

  if (!dataChannel || dataChannel.readyState !== 'open') {
    alert('⚠️ Pehle kisi device se connect karein!');
    return;
  }

  dataChannel.send(JSON.stringify({ type: 'msg', text }));
  addMessage(text, 'sent');
  input.value = '';
  saveHistory();
}

function addMessage(text, type) {
  const div = document.createElement('div');
  div.className = `msg ${type}`;
  const time = new Date().toLocaleTimeString('en-IN', { hour: '2-digit', minute: '2-digit' });
  div.innerHTML = `${text}<span class="time">${time}</span>`;
  document.getElementById('messages').appendChild(div);
  scrollBottom();
  messages.push({ text, type, time });
}

function addSystemMsg(text) {
  const div = document.createElement('div');
  div.className = 'system-msg';
  div.textContent = text;
  document.getElementById('messages').appendChild(div);
  scrollBottom();
}

function scrollBottom() {
  const m = document.getElementById('messages');
  m.scrollTop = m.scrollHeight;
}

function clearChat() {
  if (confirm('Chat clear karein?')) {
    document.getElementById('messages').innerHTML = '';
    messages = [];
    localStorage.removeItem('p2p_history');
  }
}

// ===== History (Local Only - No Cloud) =====
function saveHistory() {
  localStorage.setItem('p2p_history', JSON.stringify(messages.slice(-100)));
}

function loadHistory() {
  const saved = localStorage.getItem('p2p_history');
  if (saved) {
    messages = JSON.parse(saved);
    messages.forEach(m => {
      const div = document.createElement('div');
      div.className = `msg ${m.type}`;
      div.innerHTML = `${m.text}<span class="time">${m.time}</span>`;
      document.getElementById('messages').appendChild(div);
    });
    scrollBottom();
  }
}

// ===== PWA Install =====
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('sw.js');
}