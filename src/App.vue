<template>
  <div class="container">
    <h1>Freemax Video</h1>
    
    <!-- Экран входа / Создания комнаты -->
    <div v-if="!socket" class="login-screen">
      <div v-if="!roomId" class="create-mode">
        <input v-model="nickname" placeholder="Введите ваш ник" />
        <button @click="createRoom">🏠 Создать комнату</button>
        <button @click="testDevices" style="margin-top: 10px; background: #eee;">🧪 Проверка устройств</button>
        <div class="divider">или</div>
        <p class="hint">Если у вас есть ссылка, просто откройте её</p>
      </div>

      <div v-else class="join-mode">
        <p>Комната: <strong>{{ roomId }}</strong></p>
        <input v-model="nickname" placeholder="Введите ваш ник" />
        <button @click="joinRoom">🚪 Войти в конференцию</button>
        <button @click="copyLink" class="secondary">🔗 Копировать ссылку</button>
        <button @click="resetRoom" class="danger">✖ Отмена</button>
        <button @click="testDevices" style="margin-top: 10px; background: #eee;">🧪 Проверка устройств</button>
      </div>

      <div v-if="shareLink" class="share-block">
        <p>Отправьте эту ссылку собеседнику:</p>
        <input :value="shareLink" readonly @click="copyLink" />
        <button @click="copyLink">Копировать</button>
      </div>
      
      <div v-if="deviceInfo" class="device-info">
        {{ deviceInfo }}
      </div>
    </div>

    <!-- Интерфейс конференции -->
    <div v-else class="conference-room">
      <div class="status-bar">
        <span>Вы: {{ nickname }} | Room: {{ roomId }} | Socket: {{ socket?.id || '---' }}</span>
        <div class="controls">
          <!-- 🔥 FIX: Кнопка видео теперь работает даже если камеры не было изначально -->
          <button @click="toggleVideo">
            {{ hasCamera && isVideoOn ? '📹 Выкл. камеру' : '📷 Вкл. камеру' }}
          </button>
          <button @click="toggleAudio">
            {{ isAudioOn ? '🎤 Выкл. звук' : '🎙️ Вкл. звук' }}
          </button>
          <!-- 🔥 FIX: Убрали disabled="!hasCamera" для демонстрации экрана -->
          <button @click="shareScreen">
            {{ isSharingScreen ? '🖥 Остановить экран' : '🖥 Показать экран' }}
          </button>
          <button @click="leaveRoom" class="danger">Покинуть</button>
        </div>
      </div>

      <div class="grid">
        <div class="video-card local">
          <video ref="localVideoRef" autoplay controls="true" muted playsinline></video>
          <div class="label">
            Вы {{ !hasCamera ? '(без камеры)' : '' }} {{ !isAudioOn ? '(молчит)' : '' }}
          </div>
        </div>

        <div v-for="(stream, userId) in remoteStreams" :key="userId" class="video-card remote">
          <video :srcObject="stream" autoplay controls="true" playsinline></video>
          <div class="label">{{ peers[userId]?.nickname || userId }}</div>
        </div>
      </div>

      <div class="peers-list">
        Подключено: {{ Object.keys(peers).length + 1 }}
      </div>
      
      <div v-if="!hasCamera" class="camera-warning">
        ⚠️ Камера не найдена. Работаем только со звуком.
      </div>

      <!-- Логирование в интерфейсе -->
      <div class="debug-console">
        <h4>🛠 Системный лог:</h4>
        <ul>
          <li v-for="(log, i) in logs" :key="i" :class="log.type">{{ log.msg }}</li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted } from 'vue';
import { io, Socket } from 'socket.io-client';

// --- State ---
const socket = ref<Socket | null>(null);
const nickname = ref('');
const roomId = ref<string | null>(null);
const shareLink = ref('');
const deviceInfo = ref('');
const turnConfig = ref<{ username: string; password: string } | null>(null);

const logs = reactive<{type: string, msg: string}[]>([]);
const addLog = (msg: string, type: 'info' | 'error' | 'success' = 'info') => {
  const timestamp = new Date().toLocaleTimeString();
  logs.unshift({ type, msg: `[${timestamp}] ${msg}` });
  if (logs.length > 20) logs.pop();
  console.log(`[${type.toUpperCase()}] ${msg}`);
};

const localVideoRef = ref<HTMLVideoElement | null>(null);
const remoteStreams = reactive<Record<string, MediaStream>>({});
const peers = reactive<Record<string, { nickname: string }>>({});

// Media State
const localStream = ref<MediaStream | null>(null);
const cameraStream = ref<MediaStream | null>(null);
const isVideoOn = ref(false);
const isAudioOn = ref(true);
const isSharingScreen = ref(false);
const hasCamera = ref(false);

// WebRTC Config
const getRtcConfig = () => {
  const config: RTCConfiguration = { 
    iceServers: [
      { urls: 'stun:stun.l.google.com:19302' },
      { urls: 'stun:stun1.l.google.com:19302' },
    ],
  };

  // 🔥 Добавляем TURN сервер только если получили креды
  if (turnConfig.value) {
    config.iceServers!.push({
      // Замените на адрес ВАШЕГО сервера
      urls: 'turn:turn.zerocomputer.ru:3478', 
      username: turnConfig.value.username,
      credential: turnConfig.value.password,
    });
    addLog('✅ TURN сервер настроен с уникальными кредами', 'success');
  }
  
  return config;
};

const peerConnections = reactive<Record<string, RTCPeerConnection>>({});

// --- Lifecycle ---
onMounted(() => {
  addLog('Компонент смонтирован', 'info');
  const urlParams = new URLSearchParams(window.location.search);
  const roomFromUrl = urlParams.get('room');
  if (roomFromUrl) {
    roomId.value = roomFromUrl;
    addLog(`ID комнаты из URL: ${roomFromUrl}`, 'info');
  }
});

// --- Helper Functions ---
const generateLink = (room: string) => {
  const url = new URL(window.location.href);
  url.searchParams.set('room', room);
  return url.toString();
};

const checkAvailableDevices = async () => {
  addLog('Проверка доступных устройств...', 'info');
  try {
    const devices = await navigator.mediaDevices.enumerateDevices();
    const videoInputs = devices.filter(d => d.kind === 'videoinput');
    const audioInputs = devices.filter(d => d.kind === 'audioinput');
    
    hasCamera.value = videoInputs.length > 0;
    deviceInfo.value = `📹 Камер: ${videoInputs.length} | 🎤 Микрофонов: ${audioInputs.length}`;
    
    addLog(`Найдено: ${videoInputs.length} камер, ${audioInputs.length} микрофонов`, 'success');
    return { hasVideo: videoInputs.length > 0, hasAudio: audioInputs.length > 0 };
  } catch (e: any) {
    addLog(`Ошибка проверки устройств: ${e.message}`, 'error');
    return { hasVideo: false, hasAudio: false };
  }
};

const initAudioOnly = async () => {
  addLog('Запрос доступа к микрофону...', 'info');
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ 
       audio: { 
        echoCancellation: true, 
        noiseSuppression: true,
        autoGainControl: true
      } 
    });
    
    localStream.value = stream;
    isAudioOn.value = true;
    addLog('✅ Доступ к микрофону получен', 'success');
    
    if (localVideoRef.value) {
      localVideoRef.value.srcObject = stream;
      addLog('Видео-элемент обновлен (аудио поток)', 'info');
    }
    
    return true;
  } catch (err: any) {
    addLog(`❌ Ошибка доступа к микрофону: ${err.name}`, 'error');
    alert(`❌ Не удалось получить доступ к микрофону: ${err.name}`);
    return false;
  }
};

const enableCamera = async () => {
  addLog('Запрос доступа к камере...', 'info');
  try {
    const videoStream = await navigator.mediaDevices.getUserMedia({ 
      video: { 
        width: { ideal: 1280 },  
        height: { ideal: 720 },
        facingMode: 'user'
      } 
    });
    
    const videoTrack = videoStream.getVideoTracks()[0];
    if (!videoTrack) throw new Error('No video track');
    
    cameraStream.value = videoStream;
    localStream.value?.addTrack(videoTrack);
    
    if (localVideoRef.value) {
      localVideoRef.value.srcObject = localStream.value;
    }
    
    // 🔥 FIX: Добавляем видеотрек во ВСЕ существующие peer connections
    Object.values(peerConnections).forEach(pc => {
      pc.addTrack(videoTrack, localStream.value!);
      addLog(`Видеотрек добавлен в соединение`, 'info');
    });
    
    hasCamera.value = true;
    isVideoOn.value = true;
    addLog('✅ Камера включена и добавлена в поток', 'success');
    return true;
  } catch (err: any) {
    addLog(`⚠️ Не удалось включить камеру: ${err.name}`, 'error');
    hasCamera.value = false;
    isVideoOn.value = false;
    return false;
  }
};

// 🔥 FIX: Общая функция для старта медиа (используется и в create, и в join)
const startMediaFlow = async () => {
  const devices = await checkAvailableDevices();
  if (!devices.hasAudio && !devices.hasVideo) {
    alert('❌ Не найдено ни микрофона, ни камеры.');
    return;
  }

  const audioOk = await initAudioOnly();
  if (!audioOk) return;

  if (devices.hasVideo) {
    const useCamera = confirm('📷 Найдена камера. Включить видео?');
    if (useCamera) await enableCamera();
  }

  setupSocketListeners();
  socket.value?.emit('join', { nickname: nickname.value, roomId: roomId.value });
  addLog('Отправлен запрос на вход в комнату (join)', 'info');
};

// --- Room Functions ---

const createRoom = async () => {
  if (!nickname.value) return alert('Введите ник!');
  addLog('Создание комнаты...', 'info');
  
  initSocket();
  
  socket.value?.emit('create-room', {}, (response: { roomId: string }) => {
    addLog(`Комната создана: ${response.roomId}`, 'success');
    roomId.value = response.roomId;
    shareLink.value = generateLink(response.roomId);
    
    const url = new URL(window.location.href);
    url.searchParams.set('room', response.roomId);
    window.history.pushState({}, '', url.toString());

    // 🔥 FIX: ВАЖНО! Запускаем медиа-поток после создания комнаты
    startMediaFlow();
  });
};

const resetRoom = () => {
  roomId.value = null;
  shareLink.value = '';
  const url = new URL(window.location.href);
  url.searchParams.delete('room');
  window.history.pushState({}, '', url.toString());
  addLog('Сброс комнаты', 'info');
};

const copyLink = () => {
  const link = shareLink.value || generateLink(roomId.value!);
  navigator.clipboard.writeText(link).then(() => {
    addLog('Ссылка скопирована', 'success');
  });
};

const initSocket = () => {
  if (socket.value) return;
  
  const url = import.meta.env.VITE_WS_URL || 'http://localhost:3000';
  addLog(`Подключение к сокету: ${url}`, 'info');
  
  socket.value = io(url, {
    transports: ['websocket'],
    reconnection: true,
    reconnectionAttempts: 5, 
  });

  socket.value.on('connect', () => {
    addLog(`✅ Socket connected: ${socket.value?.id}`, 'success');
  });
  
  socket.value.on('disconnect', () => {
    addLog('❌ Socket disconnected', 'error');
  });
};

const joinRoom = async () => {
  if (!nickname.value) return alert('Введите ник!');
  if (!roomId.value) return alert('Нет ID комнаты!');

  addLog('Вход в комнату...', 'info');
  initSocket();
  await startMediaFlow();
};

const leaveRoom = () => {
  addLog('Покидаем комнату...', 'info');
  socket.value?.disconnect();
  socket.value = null;
  roomId.value = null;
  shareLink.value = '';
  
  Object.values(peerConnections).forEach(pc => {
    pc.close();
    addLog(`PeerConnection закрыт`, 'info');
  });
  
  Object.keys(peerConnections).forEach(key => delete peerConnections[key]);
  Object.keys(remoteStreams).forEach(key => delete remoteStreams[key]);
  Object.keys(peers).forEach(key => delete peers[key]);
  
  const url = new URL(window.location.href);
  url.searchParams.delete('room');
  window.history.pushState({}, '', url.toString());
  
  if (localStream.value) {
    localStream.value.getTracks().forEach(track => track.stop());
  }
  if (cameraStream.value) {
    cameraStream.value.getTracks().forEach(track => track.stop());
  }
  
  localStream.value = null;
  addLog('Комната покинута, ресурсы очищены', 'success');
};

const testDevices = async () => {
  const devices = await checkAvailableDevices();
  let info = `${deviceInfo.value}\n\n`;
  
  try {
    const audioStream = await navigator.mediaDevices.getUserMedia({ audio: true });
    audioStream.getTracks().forEach(t => t.stop());
    info += '✅ Микрофон работает\n';
  } catch (e: any) {
    info += `❌ Микрофон: ${e.name}\n`;
  }
  
  if (devices.hasVideo) {
    try {
      const videoStream = await navigator.mediaDevices.getUserMedia({ video: true });
      videoStream.getTracks().forEach(t => t.stop());
      info += '✅ Камера работает\n';
    } catch (e: any) {
      info += `❌ Камера: ${e.name}\n`;
    }
  }
  
  alert(info);
};

// --- Socket Listeners ---

const setupSocketListeners = () => {
  if (!socket.value) return;
  addLog('Настройка слушателей сокетов...', 'info');

  socket.value.on('users-list', (data: any) => {
    // 🔥 FIX: Поддержка нового формата ответа (объект с users и turnConfig)
    const users = Array.isArray(data) ? data : data.users;
    
    if (data.turnConfig) {
      turnConfig.value = data.turnConfig;
      addLog('🔐 Получены TURN креды от сервера', 'success');
    }

    addLog(`Получен список пользователей: ${users.length}`, 'info');
    const myId = socket.value?.id || `fallback_${Date.now()}`;
    
    users.forEach((user: any) => {
      if (peerConnections[user.id]) return;
      peers[user.id] = { nickname: user.nickname };
      const isInitiator = myId < user.id;
      createPeerConnection(user.id, isInitiator);
    });
  });

  socket.value.on('user-joined', (user: { id: string; nickname: string }) => {
    addLog(`Пользователь присоединился: ${user.nickname}`, 'success');
    if (peerConnections[user.id]) return;
    
    const myId = socket.value?.id || `fallback_${Date.now()}`;
    peers[user.id] = { nickname: user.nickname };
    const isInitiator = myId < user.id;
    createPeerConnection(user.id, isInitiator);
  });

  socket.value.on('user-disconnected', ({ userId }: { userId: string }) => {
    addLog(`Пользователь отключился: ${userId}`, 'error');
    delete peers[userId];
    delete remoteStreams[userId];
    if (peerConnections[userId]) {
      peerConnections[userId].close();
      delete peerConnections[userId];
    }
  });

  socket.value.on('signal', async (data: any) => {
    const { senderId, type, sdp, candidate } = data;
    addLog(`📩 Signal received from ${senderId}: ${type}`, 'info');
    
    const pc = peerConnections[senderId];
    if (!pc) {
      addLog(`⚠️ PeerConnection для ${senderId} не найден`, 'error');
      return;
    }

    try {
      if (type === 'offer') {
        await pc.setRemoteDescription(new RTCSessionDescription(sdp));
        addLog('Remote Description (Offer) set', 'info');
        
        const answer = await pc.createAnswer();
        await pc.setLocalDescription(answer);
        addLog('Local Description (Answer) created & set', 'info');
        
        socket.value?.emit('signal', { targetId: senderId, type: 'answer', sdp: answer });
      } 
      else if (type === 'answer') {
        await pc.setRemoteDescription(new RTCSessionDescription(sdp));
        addLog('Remote Description (Answer) set', 'info');
      } 
      else if (type === 'ice-candidate' && candidate) {
        await pc.addIceCandidate(new RTCIceCandidate(candidate));
        addLog('ICE Candidate added', 'info');
      }
    } catch (e: any) {
      addLog(`Error processing signal: ${e.message}`, 'error');
    }
  });
};

// --- Peer Connection ---

const createPeerConnection = (targetId: string, isInitiator: boolean) => {
  addLog(`Creating PeerConnection for ${targetId}...`, 'info');
  
  // 🔥 FIX: Используем функцию для получения актуального конфига с TURN
  const pc = new RTCPeerConnection(getRtcConfig());
  peerConnections[targetId] = pc;

  // 🔥 FIX: Логирование ICE состояний для отладки
  pc.oniceconnectionstatechange = () => {
    addLog(`ICE State (${targetId}): ${pc.iceConnectionState}`, 
      pc.iceConnectionState === 'failed' ? 'error' : 'info');
  };

  if (localStream.value) {
    localStream.value.getTracks().forEach((track) => {
      pc.addTrack(track, localStream.value!);
      addLog(`Track ${track.kind} added to PC`, 'info');
    });
  }

  pc.ontrack = (event) => {
    addLog(`🎥 Track received from ${targetId}: ${event.track.kind}`, 'success');
    if (event.streams && event.streams[0]) {
      remoteStreams[targetId] = event.streams[0];
    }
  };

  pc.onicecandidate = (event) => {
    if (event.candidate) {
      socket.value?.emit('signal', {
        targetId,
        type: 'ice-candidate',
        candidate: event.candidate,
      });
    }
  };

  if (isInitiator) {
    pc.createOffer()
      .then((offer) => pc.setLocalDescription(offer))
      .then(() => {
        socket.value?.emit('signal', {
          targetId,
          type: 'offer',
          sdp: pc.localDescription,
        });
      })
      .catch(err => addLog(`Error creating offer: ${err.message}`, 'error'));
  }
};

// --- Media Controls ---

const toggleVideo = async () => {
  if (!localStream.value) {
    addLog('⚠️ localStream не инициализирован', 'error');
    return;
  }
  
  // 🔥 FIX: Если камеры ещё нет — пытаемся включить
  if (!hasCamera.value) {
    await enableCamera();
    return;
  }
  
  const videoTrack = localStream.value.getVideoTracks()[0];
  if (videoTrack) {
    videoTrack.enabled = !videoTrack.enabled;
    isVideoOn.value = videoTrack.enabled;
    addLog(`Video toggled: ${isVideoOn.value}`, 'info');
  }
};

const toggleAudio = () => {
  if (!localStream.value) return;
  const audioTrack = localStream.value.getAudioTracks()[0];
  if (audioTrack) {
    audioTrack.enabled = !audioTrack.enabled;
    isAudioOn.value = audioTrack.enabled;
    addLog(`Audio toggled: ${isAudioOn.value}`, 'info');
  }
};

// В функции shareScreen
const shareScreen = async () => {
  if (!localStream.value) return;

  try {
    if (!isSharingScreen.value) {
      // 1. Получаем только ВИДЕО с экрана
      const screenStream = await navigator.mediaDevices.getDisplayMedia({ 
        video: true, 
        audio: false // ❗ Не захватываем аудио системы, чтобы не было эха
      });
      
      const screenTrack = screenStream.getVideoTracks()[0];
      if (!screenTrack) return;

      // 2. Показываем превью (объединяем аудио камеры + видео экрана)
      const audioTrack = localStream.value.getAudioTracks()[0];
      const combinedStream = new MediaStream([screenTrack, ...(audioTrack ? [audioTrack] : [])]);
      
      if (localVideoRef.value) {
        localVideoRef.value.srcObject = combinedStream;
      }

      // 3. Заменяем трек во всех соединениях
      Object.values(peerConnections).forEach(pc => {
        const sender = pc.getSenders().find(s => s.track?.kind === 'video');
        if (sender) {
          sender.replaceTrack(screenTrack);
        }
      });

      // 4. Обработка остановки (крестик в браузере)
      screenTrack.onended = () => stopScreenShare();
      
      isSharingScreen.value = true;
      // ❗ НЕ заменяем localStream.value полностью, иначе потеряем контекст аудио
    } else {
      stopScreenShare();
    }
  } catch (err) {
    addLog(`Error sharing screen: ${err}`, 'error');
  }
};

// В функции stopScreenShare
const stopScreenShare = async () => {
  if (!cameraStream.value && !localStream.value) return;
  
  // Возвращаем трек камеры
  const camTrack = cameraStream.value?.getVideoTracks()[0] || null;
  
  Object.values(peerConnections).forEach(pc => {
    const sender = pc.getSenders().find(s => s.track?.kind === 'video');
    if (sender) sender.replaceTrack(camTrack);
  });

  if (localVideoRef.value) {
    localVideoRef.value.srcObject = localStream.value;
  }
  
  isSharingScreen.value = false;
};
onUnmounted(() => {
  leaveRoom();
});
</script>

<style>
/* ... (ваши стили остаются без изменений) ... */
.container { font-family: sans-serif; text-align: center; padding: 20px; }
.login-screen { margin-top: 50px; max-width: 400px; margin-left: auto; margin-right: auto; }
input { padding: 10px; font-size: 16px; margin: 5px; width: 80%; }
button { padding: 10px 20px; font-size: 16px; cursor: pointer; margin: 5px; }
button:disabled { opacity: 0.5; cursor: not-allowed; }
button.secondary { background: #e0e0e0; border: 1px solid #ccc; }
button.danger { background: #ffdddd; border: 1px solid #ffaaaa; color: #cc0000; }
.divider { margin: 20px 0; color: #888; font-style: italic; }
.hint { font-size: 14px; color: #666; }
.share-block { margin-top: 20px; padding: 15px; background: #e8f5e9; border-radius: 8px; }
.share-block input { width: 90%; font-size: 14px; }
.conference-room { display: flex; flex-direction: column; gap: 20px; }
.status-bar { display: flex; justify-content: space-between; align-items: center; background: #f0f0f0; padding: 10px; border-radius: 8px; flex-wrap: wrap; }
.controls { display: flex; flex-wrap: wrap; gap: 5px; }
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; }
.video-card { position: relative; background: #000; border-radius: 10px; overflow: hidden; aspect-ratio: 16/9; }
.video-card video { width: 100%; height: 100%; object-fit: cover; }
.label { position: absolute; bottom: 10px; left: 10px; color: white; background: rgba(0,0,0,0.5); padding: 2px 8px; border-radius: 4px; }
.peers-list { color: #666; }
.device-info { margin-top: 15px; padding: 10px; background: #e3f2fd; border-radius: 8px; font-size: 14px; }
.camera-warning { padding: 10px; background: #fff3cd; border: 1px solid #ffc107; border-radius: 8px; color: #856404; }

.debug-console {
  margin-top: 20px;
  text-align: left;
  background: #222;
  color: #0f0;
  padding: 10px;
  border-radius: 8px;
  font-family: monospace;
  font-size: 12px;
  max-height: 200px;
  overflow-y: auto;
}
.debug-console h4 { margin: 0 0 10px 0; color: #fff; }
.debug-console ul { list-style: none; padding: 0; margin: 0; }
.debug-console li { margin-bottom: 4px; border-bottom: 1px solid #333; padding-bottom: 2px; }
.debug-console .error { color: #ff6b6b; }
.debug-console .success { color: #51cf66; }
.debug-console .info { color: #ccc; }
</style>