<template>
  <v-container class="fill-height">
    <v-row justify="center" align="center">
      <v-col cols="12" md="8" lg="6">

        <v-card elevation="5" rounded="xl" class="d-flex flex-column" height="80vh">

          <v-card-title class="d-flex align-center bg-grey-lighten-4 py-4 px-5">
            <span class="text-h6 font-weight-bold">💬 실시간 채팅방</span>

            <v-spacer></v-spacer>

            <v-chip color="primary" variant="flat" size="small" class="mr-2">
              <v-icon start icon="mdi-account"></v-icon>
              {{ username }}
            </v-chip>

            <v-chip :color="connected ? 'success' : 'error'" variant="tonal" size="small">
              {{ connected ? 'Online' : 'Offline' }}
            </v-chip>
          </v-card-title>

          <v-divider></v-divider>

          <v-card-text ref="chatContainer" class="flex-grow-1 overflow-y-auto bg-white pa-4">
            <div v-if="messages.length === 0" class="text-center text-grey mt-10">
              <v-icon size="large" icon="mdi-chat-outline" class="mb-2"></v-icon>
              <p>대화가 시작되지 않았습니다.</p>
            </div>

            <ChatMessage v-for="(msg, index) in messages" :key="index" :message="msg" />
          </v-card-text>

          <v-divider></v-divider>

          <v-card-actions class="pa-4 bg-grey-lighten-5">
            <v-text-field v-model="inputMessage" placeholder="메시지를 입력하세요..." variant="outlined" density="comfortable"
              hide-details rounded="pill" bg-color="white" @keyup.enter="sendMessage">
              <template v-slot:append-inner>
                <v-btn icon="mdi-send" variant="text" color="primary" @click="sendMessage"
                  :disabled="!connected || !inputMessage.trim()"></v-btn>
              </template>
            </v-text-field>
          </v-card-actions>

        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import ChatMessage from '@/components/chat/ChatMessage.vue';
import { ref, onMounted, onUnmounted, nextTick } from 'vue';
import { useRoute } from 'vue-router';
import { Client } from '@stomp/stompjs';

const route = useRoute();

const roomId = 'room1';
const username = ref(route.query.username || `익명-${Math.floor(Math.random() * 1000)}`);
const uuid = crypto.randomUUID();

const inputMessage = ref('');
const messages = ref([]);
const connected = ref(false);
const chatContainer = ref(null);

let stompClient = null;

const connect = () => {
  const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
  const brokerURL = `${protocol}//userinsight.co.kr:1080/ws-stomp`;

  stompClient = new Client({
    brokerURL: brokerURL,
    reconnectDelay: 5000,
    onConnect: (frame) => {
      console.log('Connected: ' + frame);
      connected.value = true;

      stompClient.subscribe(`/sub/chat/${roomId}`, (messageOutput) => {
        showMessage(JSON.parse(messageOutput.body));
      });
    },
    onStompError: (frame) => {
      console.error('Broker reported error: ' + frame.headers['message']);
      console.error('Additional details: ' + frame.body);
      connected.value = false;
    },
    onWebSocketClose: () => {
      connected.value = false;
    }
  });

  stompClient.activate();
};

const disconnect = () => {
  if (stompClient) {
    stompClient.deactivate();
  }
};

const sendMessage = () => {
  if (!inputMessage.value.trim() || !stompClient || !connected.value) return;

  const chatMessage = {
    id: uuid,
    username: username.value,
    message: inputMessage.value
  };

  stompClient.publish({
    destination: `/pub/chat/${roomId}`,
    body: JSON.stringify(chatMessage)
  });

  inputMessage.value = '';
};

// 메시지 수신 처리 및 화면 표시
const showMessage = (messageData) => {
  // 내 메시지인지 판별 (uuid 비교)
  const isMine = messageData.id === uuid;

  messages.value.push({
    ...messageData,
    isMine: isMine
  });

  scrollToBottom();
};

// 스크롤을 항상 최하단으로 이동
const scrollToBottom = async () => {
  await nextTick(); // DOM 업데이트 대기
  if (chatContainer.value) {
    chatContainer.value.$el.scrollTop = chatContainer.value.$el.scrollHeight;
  }
};

onMounted(() => {
  connect();
});

onUnmounted(() => {
  disconnect();
});
</script>