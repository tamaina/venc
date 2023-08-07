<template>
  <div :class="$style.root">
    <input type="file" ref="input" @change="onInputChange">
    <video :class="$style.video" :src="sourceUrl" controls />
    <video :class="$style.video" :src="video" controls />
    <button @click="transcode">Start</button>
    <button @click="logs = []">Purge Logs</button>
    <div :class="$style.logs">
      <p v-for="log in logs" v-text="log" />
    </div>
  </div>
</template>

<script lang="ts" setup>
// https://github.com/ffmpegwasm/ffmpeg.wasm/blob/a6eb8af90471808fcf79afd4e232f68f750ea218/apps/vue-vite-app/src/components/FFmpegDemo.vue
import { FFmpeg } from '@ffmpeg/ffmpeg'
import type { LogEvent } from '../node_modules/@ffmpeg/ffmpeg/dist/esm/types'
import coreURL from '../node_modules/@ffmpeg/core/dist/esm/ffmpeg-core.js?url'
import wasmURL from '../node_modules/@ffmpeg/core/dist/esm/ffmpeg-core.wasm?url'
import { computed, ref } from 'vue'

const VOUTDIR = './out'

const video = ref('')
const source = ref(null as File | null)
const input = ref<HTMLInputElement>();
const logs = ref<string[]>(['Click Start to Transcode']);
const sourceUrl = computed(() => source.value ? URL.createObjectURL(source.value) : '')

function onInputChange() {
  source.value = input.value?.files?.item(0) ?? null;
}

async function transcode() {
  const ffmpeg = new FFmpeg()
  logs.value.push('Loading ffmpeg-core.js')
  ffmpeg.on('log', ({ message: msg }: LogEvent) => {
    logs.value.push(msg)
  })
  await ffmpeg.load({ coreURL, wasmURL })
  if (!source.value) return;
  logs.value.push('Start transcoding')
  await ffmpeg.createDir(VOUTDIR);
  await ffmpeg.writeFile('test.avi', new Uint8Array(await source.value.arrayBuffer()))
  const segCode = await ffmpeg.exec(['-i', 'test.avi', '-map', '0', '-c', 'copy', '-flags', '+global_header', '-f', 'segment', '-segment_time', '2', '-segment_format_options', 'movflags=+faststart', '-reset_timestamps', '1', `${VOUTDIR}/%10d.mp4`])
  logs.value.push(`Complete transcoding @${segCode}`)
  logs.value.push((await ffmpeg.listDir(VOUTDIR)).filter(x => !x.isDir).map(x => x.name).join('\n'));
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

  > * {
    margin: 8px;
  }
}

.logs {
  text-align: left;
  font-family: 'Courier New', Courier, monospace;
  white-space: pre-wrap;
}

.video {
  max-height: 50vh;
}
</style>
