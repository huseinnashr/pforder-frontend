<script setup lang="ts">
import { RouterView } from 'vue-router'
import  * as config from './config'
import moment from 'moment-timezone';
</script>

<template>
  <v-toolbar>
    <v-toolbar-title>List Order</v-toolbar-title>
    <p style="margin-right: 1rem;"> Timezone: {{config.TIME_ZONE}} (GMT{{ moment().tz(config.TIME_ZONE).format("Z")}})</p>
  </v-toolbar>
  <RouterView />
</template>