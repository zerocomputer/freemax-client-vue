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
      
      <!-- 🔥 Инфо о устройствах -->
      <div v-if="deviceInfo" class="device-info">
        {{ deviceInfo }}
      </div>
    </div>

    <!-- Интерфейс конференции -->
    <div v-else class="conference-room">
      <div class="status-bar">
        <span>Вы: {{ nickname }} | Room: {{ roomId }}</span>
        <div class="controls">
          <!-- 🔥 Кнопка теперь включает/выключает камеру по запросу -->
          <button @click="toggleVideo" :disabled="!hasCamera">
            {{ hasCamera && isVideoOn ? '📹 Выкл. камеру' : '📷 Вкл. камеру' }}
          </button>
          <button @click="toggleAudio">
            {{ isAudioOn ? '🎤 Выкл. звук' : '🎙️ Вкл. звук' }}
          </button>
          <button @click="shareScreen" :disabled="!hasCamera">
            {{ isSharingScreen ? '🖥 Остановить экран' : '🖥 Показать экран' }}
          </button>
          <button @click="leaveRoom" class="danger">Покинуть</button>
        </div>
      </div>

      <div class="grid">
        <!-- Локальное видео -->
        <div class="video-card local">
          <video ref="localVideoRef" autoplay muted playsinline></video>
          <div class="label">
            Вы {{ !hasCamera ? '(без камеры)' : '' }}
          </div>
        </div>

        <!-- Удаленные видео -->
        <div v-for="(stream, userId) in remoteStreams" :key="userId" class="video-card remote">
          <video :srcObject="stream" autoplay playsinline></video>
          <div class="label">{{ peers[userId]?.nickname || userId }}</div>
        </div>
      </div>
    
      <div class="peers-list">
        Подключено: {{ Object.keys(peers).length + 1 }}
      </div>
      
      <!-- 🔥 Статус камеры -->
      <div v-if="!hasCamera" class="camera-warning">
        ⚠️ Камера не найдена. Работаем только со звуком.
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

const localVideoRef = ref<HTMLVideoElement | null>(null);
const remoteStreams = reactive<Record<string, MediaStream>>({});
const peers = reactive<Record<string, { nickname: string }>>({});

// Media State
const localStream = ref<MediaStream | null>(null);
const cameraStream = ref<MediaStream | null>(null); // Только видеотрек
const isVideoOn = ref(false); // 🔥 По умолчанию видео ВЫКЛЮЧЕНО
const isAudioOn = ref(true);
const isSharingScreen = ref(false);
const hasCamera = ref(false); // 🔥 Флаг наличия камеры

// WebRTC Config
const rtcConfig: RTCConfiguration = { 
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
  ],
};

// PeerConnections
const peerConnections = reactive<Record<string, RTCPeerConnection>>({});

// --- Lifecycle ---
onMounted(() => {
  const urlParams = new URLSearchParams(window.location.search);
  const roomFromUrl = urlParams.get('room');
  if (roomFromUrl) {
    roomId.value = roomFromUrl;
  }
});

// --- Helper Functions ---

const generateLink = (room: string) => {
  const url = new URL(window.location.href);
  url.searchParams.set('room', room);
  return url.toString();
};

// 🔥 Проверка доступных устройств
const checkAvailableDevices = async () => {
  try {
    const devices = await navigator.mediaDevices.enumerateDevices();
    const videoInputs = devices.filter(d => d.kind === 'videoinput');
    const audioInputs = devices.filter(d => d.kind === 'audioinput');
    
    hasCamera.value = videoInputs.length > 0;
    deviceInfo.value = `📹 Камер: ${videoInputs.length} | 🎤 Микрофонов: ${audioInputs.length}`;
    
    return { hasVideo: videoInputs.length > 0, hasAudio: audioInputs.length > 0 };
  } catch (e: any) {
    console.error('Device check failed:', e);
    deviceInfo.value = '⚠️ Не удалось проверить устройства';
    return { hasVideo: false, hasAudio: false };
  }
};

// 🔥 Инициализация ТОЛЬКО аудио при входе
const initAudioOnly = async () => {
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
    
    // Пустой видео-превью (черный экран)
    if (localVideoRef.value) {
      localVideoRef.value.srcObject = stream;
    }
    
    return true;
  } catch (err: any) {
    console.error('Audio init failed:', err);
    alert(`❌ Не удалось получить доступ к микрофону: ${err.name}`);
    return false;
  }
};

// 🔥 Запрос камеры (опционально, по желанию пользователя)
const enableCamera = async () => {
  try {
    const videoStream = await navigator.mediaDevices.getUserMedia({ 
      video: { 
        width: { ideal: 1280 }, 
        height: { ideal: 720 },
        facingMode: 'user'
      } 
    });
    
    const videoTrack = videoStream.getVideoTracks()[0];
    if (!videoTrack) {
      videoStream.getTracks().forEach(t => t.stop());
      throw new Error('No video track');
    }
    
    // Сохраняем видеотрек
    cameraStream.value = videoStream;
    
    // Добавляем видеотрек к локальному стриму
    localStream.value?.addTrack(videoTrack);
    
    // Обновляем превью
    if (localVideoRef.value) {
      localVideoRef.value.srcObject = localStream.value;
    }
    
    // 🔥 Добавляем трек во все существующие peer connections
    Object.values(peerConnections).forEach(pc => {
      pc.addTrack(videoTrack, localStream.value!);
    });
    
    hasCamera.value = true;
    isVideoOn.value = true;
    
    console.log('✅ Камера включена');
    return true;
  } catch (err: any) {
    console.error('Camera enable failed:', err);
    
    if (err.name === 'NotFoundError') {
      alert('❌ Камера не найдена на этом устройстве.\n\nБудем работать только со звуком.');
    } else if (err.name === 'NotAllowedError') {
      alert('❌ Доступ к камере запрещен.\n\nПроверьте настройки браузера и macOS.');
    } else {
      alert(`⚠️ Не удалось включить камеру: ${err.name}`);
    }
    
    hasCamera.value = false;
    isVideoOn.value = false;
    return false;
  }
};

// 🔥 Выключение камеры (не запрашиваем заново, просто отключаем трек)
const disableCamera = () => {
  const videoTrack = localStream.value?.getVideoTracks()[0];
  if (videoTrack) {
    videoTrack.enabled = false;
    isVideoOn.value = false;
  }
};

// --- Room Functions ---

const createRoom = async () => {
  if (!nickname.value) return alert('Введите ник!');
  
  initSocket();
  
  socket.value?.emit('create-room', {}, (response: { roomId: string }) => {
    roomId.value = response.roomId;
    shareLink.value = generateLink(response.roomId);
    const url = new URL(window.location.href);
    url.searchParams.set('room', response.roomId);
    window.history.pushState({}, '', url.toString());
  });
};

const resetRoom = () => {
  roomId.value = null;
  shareLink.value = '';
  const url = new URL(window.location.href);
  url.searchParams.delete('room');
  window.history.pushState({}, '', url.toString());
};

const copyLink = () => {
  const link = shareLink.value || generateLink(roomId.value!);
  navigator.clipboard.writeText(link).then(() => {
    alert('Ссылка скопирована в буфер обмена!');
  });
};

const initSocket = () => {
  if (socket.value) return;
  
  socket.value = io(import.meta.env.VITE_WS_URL || 'http://localhost:3000', {
    transports: ['websocket'],
    reconnection: true,
    reconnectionAttempts: 5,
  });
};

const joinRoom = async () => {
  if (!nickname.value) return alert('Введите ник!');
  if (!roomId.value) return alert('Нет ID комнаты!');

  // 🔥 1. Проверяем какие устройства доступны
  const devices = await checkAvailableDevices();
  
  if (!devices.hasAudio && !devices.hasVideo) {
    alert('❌ Не найдено ни микрофона, ни камеры.\n\nПроверьте подключения устройств.');
    return;
  }

  // 🔥 2. Инициализируем ТОЛЬКО аудио
  const audioOk = await initAudioOnly();
  if (!audioOk) return;

  // 🔥 3. Если есть камера — спрашиваем пользователя
  if (devices.hasVideo) {
    const useCamera = confirm('📷 Найдена камера. Хотите включить видео?');
    if (useCamera) {
      await enableCamera();
    }
  }

  initSocket();
  setupSocketListeners();
  socket.value?.emit('join', { nickname: nickname.value, roomId: roomId.value });
};

const leaveRoom = () => {
  socket.value?.disconnect();
  socket.value = null;
  roomId.value = null;
  shareLink.value = '';
  
  Object.values(peerConnections).forEach(pc => pc.close());
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
};

const testDevices = async () => {
  const devices = await checkAvailableDevices();
  
  let info = `${deviceInfo.value}\n\n`;
  
  // Тест аудио
  try {
    const audioStream = await navigator.mediaDevices.getUserMedia({ audio: true });
    audioStream.getTracks().forEach(t => t.stop());
    info += '✅ Микрофон работает\n';
  } catch (e: any) {
    info += `❌ Микрофон: ${e.name}\n`;
  }
  
  // Тест видео (только если есть камера)
  if (devices.hasVideo) {
    try {
      const videoStream = await navigator.mediaDevices.getUserMedia({ video: true });
      videoStream.getTracks().forEach(t => t.stop());
      info += '✅ Камера работает\n';
    } catch (e: any) {
      info += `❌ Камера: ${e.name}\n`;
    }
  } else {
    info += '⚠️ Камера не найдена\n';
  }
  
  alert(info);
};

// --- Socket Listeners ---

const setupSocketListeners = () => {
  if (!socket.value) return;

  socket.value.on('users-list', (users: { id: string; nickname: string }[]) => {
    const myId = socket.value?.id || `fallback_${Date.now()}`;
    
    users.forEach((user) => {
      if (peerConnections[user.id]) return;
      
      peers[user.id] = { nickname: user.nickname };
      const isInitiator = myId < user.id;
      createPeerConnection(user.id, isInitiator);
    });
  });

  socket.value.on('user-joined', (user: { id: string; nickname: string }) => {
    if (peerConnections[user.id]) return;
    
    const myId = socket.value?.id || `fallback_${Date.now()}`;
    peers[user.id] = { nickname: user.nickname };
    const isInitiator = myId < user.id;
    createPeerConnection(user.id, isInitiator);
  });

  socket.value.on('user-disconnected', ({ userId }: { userId: string }) => {
    delete peers[userId];
    delete remoteStreams[userId];
    if (peerConnections[userId]) {
      peerConnections[userId].close();
      delete peerConnections[userId];
    }
  });

  socket.value.on('signal', async (data: any) => {
    const { senderId, type, sdp, candidate } = data;
    const pc = peerConnections[senderId];
    
    if (!pc) return;

    if (type === 'offer') {
      await pc.setRemoteDescription(new RTCSessionDescription(sdp));
      const answer = await pc.createAnswer();
      await pc.setLocalDescription(answer);
      socket.value?.emit('signal', { targetId: senderId, type: 'answer', sdp: answer });
    } 
    else if (type === 'answer') {
      await pc.setRemoteDescription(new RTCSessionDescription(sdp));
    } 
    else if (type === 'ice-candidate' && candidate) {
      try {
        await pc.addIceCandidate(new RTCIceCandidate(candidate));
      } catch (e) {
        console.error('Error adding received ice candidate', e);
      }
    }
  });
};

// --- Peer Connection ---

const createPeerConnection = (targetId: string, isInitiator: boolean) => {
  const pc = new RTCPeerConnection(rtcConfig);
  peerConnections[targetId] = pc;

  // 🔥 Добавляем только существующие треки (аудио + видео если есть)
  localStream.value?.getTracks().forEach((track) => {
    pc.addTrack(track, localStream.value!);
  });

  pc.ontrack = (event) => {
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
      .catch(console.error);
  }
};

// --- Media Controls ---

// 🔥 Теперь toggleVideo включает камеру если её ещё нет
const toggleVideo = async () => {
  if (!localStream.value) return;
  
  // Если камеры ещё нет — пытаемся включить
  if (!hasCamera.value) {
    await enableCamera();
    return;
  }
  
  // Если камера есть — просто включаем/выключаем трек
  const videoTrack = localStream.value.getVideoTracks()[0];
  if (videoTrack) {
    videoTrack.enabled = !videoTrack.enabled;
    isVideoOn.value = videoTrack.enabled;
  }
};

const toggleAudio = () => {
  if (!localStream.value) return;
  const audioTrack = localStream.value.getAudioTracks()[0];
  if (audioTrack) {
    audioTrack.enabled = !audioTrack.enabled;
    isAudioOn.value = audioTrack.enabled;
  }
};

const shareScreen = async () => {
  if (!localStream.value) return;
  try {
    if (!isSharingScreen.value) {
      const screenStream = await navigator.mediaDevices.getDisplayMedia({ video: true });
      const screenTrack = screenStream.getVideoTracks()[0];
      if (!screenTrack) return;

      if (localVideoRef.value) localVideoRef.value.srcObject = screenStream;

      Object.values(peerConnections).forEach(pc => {
        const sender = pc.getSenders().find(s => s.track?.kind === 'video');
        if (sender && sender.track) sender.replaceTrack(screenTrack);
      });

      screenTrack.onended = () => stopScreenShare();
      isSharingScreen.value = true;
      localStream.value = screenStream;
    } else {
      stopScreenShare();
    }
  } catch (err) {
    console.error("Error sharing screen:", err); 
  }
};

const stopScreenShare = async () => {
  const streamToRestore = cameraStream.value || localStream.value;
  if (!streamToRestore) return;
  
  const camTrack = streamToRestore.getVideoTracks()[0];
  
  if (localVideoRef.value) localVideoRef.value.srcObject = streamToRestore;

  Object.values(peerConnections).forEach(pc => {
    const sender = pc.getSenders().find(s => s.track?.kind === 'video');
    if (sender) sender.replaceTrack(camTrack || null);
  });

  localStream.value = streamToRestore;
  isSharingScreen.value = false;
};

onUnmounted(() => {
  leaveRoom();
});
</script>

<style>
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
</style>