<template>
  <v-app>
    <TheHeader />

    <v-main>
      <router-view />
    </v-main>

    <TheFooter />
  </v-app>
</template>

<script setup>
import TheHeader from '@/components/common/TheHeader.vue';
import TheFooter from '@/components/common/TheFooter.vue';
</script>
