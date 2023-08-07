<template>
  <div :class="$style.root">
    <input type="file" ref="input" @change="onInputChange">
    <video :class="$style.video" :src="sourceUrl" controls />
    <video :class="$style.video" :src="video" controls />
    <br />
    <button @click="transcode">Start</button>
    <div :class="$style.logs">
      <p v-for="log in logs" v-text="log" />
    </div>
  </div>
</template>

<script lang="ts" setup>
// https://github.com/ffmpegwasm/ffmpeg.wasm/blob/a6eb8af90471808fcf79afd4e232f68f750ea218/apps/vue-vite-app/src/components/FFmpegDemo.vue
import { FFmpeg } from '@ffmpeg/ffmpeg'
import type { LogEvent } from '../node_modules/@ffmpeg/ffmpeg/dist/esm/types'
import coreURL from '../node_modules/@ffmpeg/core-mt/dist/esm/ffmpeg-core.js?url'
import workerURL from '../node_modules/@ffmpeg/core-mt/dist/esm/ffmpeg-core.worker.js?url'
import wasmURL from '../node_modules/@ffmpeg/core-mt/dist/esm/ffmpeg-core.wasm?url'
import { computed, ref } from 'vue'

const ffmpeg = new FFmpeg()
const video = ref('')
const source = ref(null as File | null)
const input = ref<HTMLInputElement>();
const logs = ref<string[]>(['Click Start to Transcode']);
const sourceUrl = computed(() => source.value ? URL.createObjectURL(source.value) : '')

function onInputChange() {
  source.value = input.value?.files?.item(0) ?? null;
}

async function transcode() {
  logs.value.push('Loading ffmpeg-core.js')
  ffmpeg.on('log', ({ message: msg }: LogEvent) => {
    logs.value.push(msg)
  })
  await ffmpeg.load({ coreURL, workerURL, wasmURL })
  if (!source.value) return;
  logs.value.push('Start transcoding')
  const fr = await (() => {
    const fr = new FileReader()
    return new Promise<FileReader>((resolve) => {
      fr.onload = () => resolve(fr);
      fr.readAsArrayBuffer(source.value as File)
    });
  })();
  await ffmpeg.writeFile('test.avi', new Uint8Array(fr.result as ArrayBuffer))
  await ffmpeg.exec(['-i', 'test.avi', '-movflags', '+faststart', '-fflags', '+bitexact', 'test.mp4'])
  logs.value.push('Complete transcoding')
  const data = await ffmpeg.readFile('test.mp4')
  video.value = URL.createObjectURL(new Blob([(data as Uint8Array).buffer], { type: 'video/mp4' }))
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  margin-top: 60px;
}
</style>

<style module>
.root {
  display: flex;
  flex-direction: column;
  max-width: 80vw;
}

.logs {
  text-align: left;
  font-family: 'Courier New', Courier, monospace;
}

.video {
  max-height: 50vh;
}
</style>
