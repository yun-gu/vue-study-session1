<template>
  <v-container class="fill-height">
    <v-row justify="center" align="center">
      <v-col cols="12" sm="8" md="6" lg="4">

        <v-card elevation="10" rounded="xl" class="pa-4">
          <v-card-item>

            <div class="text-center mb-6 mt-4">
              <div class="text-h1 mb-2">💬</div>
              <h2 class="text-h4 font-weight-bold mb-2">채팅방 입장</h2>
              <p class="text-body-2 text-medium-emphasis">
                사용할 닉네임을 입력해주세요.
              </p>
            </div>

            <v-form v-model="formValid" @submit.prevent="submitForm">

              <v-text-field v-model="nickname" label="닉네임 (2~10자)" placeholder="닉네임" variant="outlined"
                :rules="nicknameRules" counter="10" class="mb-2" autofocus clearable></v-text-field>

              <div class="d-grid gap-2 mt-4 mb-2">
                <v-btn type="submit" color="primary" size="x-large" block rounded="pill" elevation="4"
                  :disabled="!formValid">
                  입장하기 🚀
                </v-btn>
              </div>

            </v-form>
          </v-card-item>
        </v-card>

        <div class="text-center mt-6 text-caption text-medium-emphasis">
          &copy; 2026 Spring Boot Study
        </div>

      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import { ref } from 'vue';
import { useRouter } from 'vue-router'; // 라우터 사용을 위해 import

const router = useRouter();

// 상태 변수
const formValid = ref(false); // 폼 전체 유효성 상태
const nickname = ref('');

// 유효성 검사 규칙 (Array of Functions)
const nicknameRules = [
  v => !!v || '닉네임을 입력해주세요.',
  v => (v && v.length >= 2) || '2글자 이상 입력해주세요.',
  v => (v && v.length <= 10) || '10글자 이하로 입력해주세요.',
];

// 입장 버튼 클릭 시 동작
const submitForm = () => {
  if (!formValid.value) return;

  // 1. (선택사항) 닉네임을 저장소(Pinia, LocalStorage)에 저장
  // 예: localStorage.setItem('chat-nickname', nickname.value);
  console.log(`입장 닉네임: ${nickname.value}`);

  // 2. 채팅방 페이지로 이동
  // 기존의 th:action="@{/chat/room}" 와 동일한 목적지
  // 쿼리 파라미터로 넘기거나, 상태관리 라이브러리를 사용합니다.
  router.push({
    path: '/chat/room',
    query: { username: nickname.value } // URL에 ?username=... 로 붙여서 이동
  });
};
</script>