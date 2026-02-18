<template>
  <div class="container">
    <h1>Freemax</h1>

    <!-- Экран входа -->
    <div v-if="!socket" class="login-screen">
      <input v-model="nickname" placeholder="Введите ваш ник" />
      <button @click="joinRoom">Войти в конференцию</button>
      <button @click="testMediaAccess">🧪 Тест доступа к медиа</button>
    </div>

    <!-- Интерфейс конференции -->
    <div v-else class="conference-room">
      <div class="status-bar">
        <span>Вы: {{ nickname }}</span>
        <div class="controls">
          <button @click="toggleVideo">{{ isVideoOn ? '📹 Выкл. камеру' : '📷 Вкл. камеру' }}</button>
          <button @click="toggleAudio">{{ isAudioOn ? '🎤 Выкл. звук' : '🎙️ Вкл. звук' }}</button>
          <button @click="shareScreen">{{ isSharingScreen ? '🖥 Остановить экран' : '🖥 Показать экран' }}</button>
        </div>
      </div>

      <div class="grid">
        <!-- Локальное видео -->
        <div class="video-card local">
          <video ref="localVideoRef" autoplay muted playsinline></video>
          <div class="label">Вы</div>
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
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onUnmounted } from 'vue';
import { io, Socket } from 'socket.io-client';

// --- State ---
const socket = ref<Socket | null>(null);
const nickname = ref('');
const localVideoRef = ref<HTMLVideoElement | null>(null);
const remoteStreams = reactive<Record<string, MediaStream>>({});
const peers = reactive<Record<string, { nickname: string }>>({});

// Media State
const localStream = ref<MediaStream | null>(null);
const cameraStream = ref<MediaStream | null>(null); // Храним оригинальную камеру
const isVideoOn = ref(true);
const isAudioOn = ref(true);
const isSharingScreen = ref(false);

// WebRTC Config
const rtcConfig: RTCConfiguration = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    { urls: 'turn:turn.zerocomputer.ru:3478', username: 'test', credential: 'test' }
  ],
};

// PeerConnections
const peerConnections = reactive<Record<string, RTCPeerConnection>>({});

// --- Methods ---

const joinRoom = async () => {
  if (!nickname.value) return alert('Введите ник!');

  // 1. Проверка поддержки API браузером
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    alert('Ваш браузер не поддерживает доступ к камере и микрофону. Пожалуйста, используйте Chrome, Firefox или Safari.');
    return;
  }

  // 2. Проверка HTTPS (требуется для работы медиа-API)
  if (location.protocol !== 'https:' && location.hostname !== 'localhost' && location.hostname !== '127.0.0.1') {
    alert('Для доступа к камере требуется HTTPS соединение или localhost.');
    return;
  }

  socket.value = io(import.meta.env.VITE_WS_URL, {
    transports: ['websocket'],
    reconnection: true,
    reconnectionAttempts: 5,
  });

  try {
    // 3. Явный запрос прав с подробными настройками
    const stream = await navigator.mediaDevices.getUserMedia({
      video: {
        width: { ideal: 1280 },
        height: { ideal: 720 },
        facingMode: 'user'
      },
      audio: {
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true
      }
    });

    localStream.value = stream;
    cameraStream.value = stream;
    
    if (localVideoRef.value) {
      localVideoRef.value.srcObject = stream;
    }
  } catch (err: any) {
    console.error('Ошибка доступа к медиа:', err);
    
    // 4. Детальная обработка ошибок для macOS
    if (err.name === 'NotAllowedError' || err.name === 'PermissionDeniedError') {
      alert('❌ Доступ к камере/микрофону запрещен.\n\nПроверьте:\n1. Настройки сайта в браузере (замок в адресной строке)\n2. Системные настройки macOS: Системные настройки → Защита и безопасность → Конфиденциальность → Камера/Микрофон');
    } else if (err.name === 'NotFoundError') {
      alert('❌ Камера или микрофон не найдены. Проверьте подключение устройств.');
    } else if (err.name === 'NotReadableError') {
      alert('❌ Устройство занято другим приложением. Закройте другие программы, использующие камеру.');
    } else {
      alert('⚠️ Не удалось получить доступ к камере/микрофону: ' + err.message);
    }
    return;
  }

  setupSocketListeners();
  socket.value.emit('join', { nickname: nickname.value });
};

const testMediaAccess = async () => {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
    alert('✅ Доступ разрешен! Камера и микрофон работают.');
    stream.getTracks().forEach(track => track.stop());
  } catch (err: any) {
    alert('❌ Ошибка: ' + err.name + ' - ' + err.message);
  }
};

const setupSocketListeners = () => {
  if (!socket.value) return;

  socket.value.on('users-list', (users: { id: string; nickname: string }[]) => {
    const myId = socket.value?.id || `fallback_${Date.now()}`;
    
    users.forEach((user) => {
      // 🔥 Пропускаем, если соединение уже есть
      if (peerConnections[user.id]) {
        console.log(`[WebRTC] Соединение с ${user.nickname} уже существует`);
        return;
      }
      
      peers[user.id] = { nickname: user.nickname };
      
      // 🔥 Детерминированный выбор инициатора
      const isInitiator = myId < user.id;
      console.log(`[WebRTC] Создаю соединение с ${user.nickname}. Инициатор: ${isInitiator}`);
      
      createPeerConnection(user.id, isInitiator);
    });
  });

  socket.value.on('user-joined', (user: { id: string; nickname: string }) => {
    // 🔥 Пропускаем дубликаты
    if (peerConnections[user.id]) {
      console.log(`[WebRTC] Соединение с ${user.nickname} уже есть, пропускаем`);
      return;
    }
    
    const myId = socket.value?.id || `fallback_${Date.now()}`;
    peers[user.id] = { nickname: user.nickname };
    
    const isInitiator = myId < user.id;
    console.log(`[WebRTC] Новый пользователь ${user.nickname}. Инициатор: ${isInitiator}`);
    
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

const createPeerConnection = (targetId: string, isInitiator: boolean) => {
  const pc = new RTCPeerConnection(rtcConfig);
  peerConnections[targetId] = pc;

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

const toggleVideo = () => {
  if (!localStream.value) return;
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

      // Проверка на существование трека
      if (!screenTrack) {
        console.error('No video track in screen stream');
        return;
      }

      // Обновляем локальный превью
      if (localVideoRef.value) {
        localVideoRef.value.srcObject = screenStream;
      }

      // Заменяем трек во всех соединениях
      Object.values(peerConnections).forEach(pc => {
        const sender = pc.getSenders().find(s => s.track?.kind === 'video');
        if (sender && sender.track) {
          sender.replaceTrack(screenTrack);
        }
      });

      // Обработчик остановки через интерфейс браузера
      screenTrack.onended = () => {
        stopScreenShare();
      };
      
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
  // Восстанавливаем камеру из сохраненной ссылки
  const streamToRestore = cameraStream.value;
  if (!streamToRestore) return;

  const camTrack = streamToRestore.getVideoTracks()[0];
  // Проверка на undefined для TypeScript
  if (!camTrack) return;
  
  if (localVideoRef.value) {
    localVideoRef.value.srcObject = streamToRestore;
  }

  Object.values(peerConnections).forEach(pc => {
    const sender = pc.getSenders().find(s => s.track?.kind === 'video');
    if (sender) {
      // Явно приводим к null, если трек не найден, чтобы удовлетворить типизацию replaceTrack
      sender.replaceTrack(camTrack);
    }
  });

  localStream.value = streamToRestore;
  isSharingScreen.value = false;
};

onUnmounted(() => {
  socket.value?.disconnect();
  localStream.value?.getTracks().forEach(track => track.stop());
  cameraStream.value?.getTracks().forEach(track => track.stop());
});
</script>

<style scoped>
.container { font-family: sans-serif; text-align: center; padding: 20px; }
.login-screen { margin-top: 100px; }
input { padding: 10px; font-size: 16px; margin-right: 10px; }
button { padding: 10px 20px; font-size: 16px; cursor: pointer; }
.conference-room { display: flex; flex-direction: column; gap: 20px; }
.status-bar { display: flex; justify-content: space-between; align-items: center; background: #f0f0f0; padding: 10px; border-radius: 8px; }
.controls button { margin-left: 10px; }
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; }
.video-card { position: relative; background: #000; border-radius: 10px; overflow: hidden; aspect-ratio: 16/9; }
.video-card video { width: 100%; height: 100%; object-fit: cover; }
.label { position: absolute; bottom: 10px; left: 10px; color: white; background: rgba(0,0,0,0.5); padding: 2px 8px; border-radius: 4px; }
.peers-list { color: #666; }
</style>