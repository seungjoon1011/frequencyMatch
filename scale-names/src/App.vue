<script setup lang="ts">
import { ref, onUnmounted } from 'vue'

const isMicActive = ref(false)
const errorMessage = ref('')
const volume = ref(0)

let audioStream: MediaStream | null = null
let audioContext: AudioContext | null = null
let analyser: AnalyserNode | null = null
let animationId: number | null = null
async function startMicrophone() {
  errorMessage.value = ''
  try {
    audioStream = await navigator.mediaDevices.getUserMedia({
      audio: true,
    })
    audioContext = new AudioContext()

    const source = audioContext.createMediaStreamSource(audioStream)

    analyser = audioContext.createAnalyser()
    analyser.fftSize = 2048

    source.connect(analyser)

    isMicActive.value = true

    updateVolume()
  } catch (error) {
    console.error('Microphone connection error:', error)

    if (error instanceof Error) {
      errorMessage.value = error.message
    } else {
      errorMessage.value = 'Failed to access microphone.'
    }
  }
}
function updateVolume() {
  if (!analyser) return

  const data = new Uint8Array(analyser.fftSize)
  analyser.getByteTimeDomainData(data)

  let sum = 0

  for (const value of data) {
    const normalized = (value - 128) / 128
    sum += normalized * normalized
  }

  const rms = Math.sqrt(sum / data.length)

  volume.value = Math.min(100, Math.round(rms * 100 * 3))

  animationId = requestAnimationFrame(updateVolume)
}
function stopMicrophone() {
  if (animationId !== null) {
    cancelAnimationFrame(animationId)
    animationId = null
  }

  audioStream?.getTracks().forEach((track) => {
    track.stop()
  })

  audioStream = null

  if (audioContext) {
    void audioContext.close()
    audioContext = null
  }
  analyser = null
  isMicActive.value = false
  volume.value = 0
}

onUnmounted(() => {
  stopMicrophone()
})
</script>

<template>
  <main>
    <h1>Guitar Analyzer</h1>

    <p>
      Microphone status:
      {{ isMicActive ? 'Connected' : 'Disconnected' }}
    </p>

    <button v-if="!isMicActive" @click="startMicrophone">Start Microphone</button>

    <button v-else @click="stopMicrophone">Stop Microphone</button>

    <section>
      <h2>Input Level</h2>

      <progress :value="volume" max="100"></progress>
      <p>{{ volume }} / 100</p>
    </section>
    <p v-if="errorMessage">Error: {{ errorMessage }}</p>
  </main>
</template>
