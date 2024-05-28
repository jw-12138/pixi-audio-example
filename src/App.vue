<script setup lang="ts">
import {Application, Graphics} from 'pixi.js'
import {AsciiFilter} from "pixi-filters"
import {ref, onMounted, watch} from 'vue'
import {Howl, Howler} from 'howler'

const app = new Application();

let audioContext = ref(null)
let analyzer = null
let ftSizeInPow = ref(13)
let fftSize = ref(Math.pow(2, Number(ftSizeInPow.value)))

let frameWidth = Math.min(window.innerWidth, 800)
let frameHeight = 200

function initAnalyzer() {
  audioContext.value = Howler.ctx

  analyzer = audioContext.value.createAnalyser()
  console.log('sample rate: ', analyzer.context.sampleRate)

  analyzer.fftSize = fftSize.value // 2^5 to 2^15
  analyzer.smoothingTimeConstant = 0.5

  analyzer.connect(audioContext.value.destination)

  console.log('fftSize:', fftSize.value)
  console.log('analyzer enabled')
}

watch(fftSize, (newVal) => {
  if (analyzer) {
    analyzer.fftSize = newVal
  }
})

watch(ftSizeInPow, (newVal) => {
  fftSize.value = Math.pow(2, Number(newVal))
})

function uint8ArrayToArray(uint8Array: Uint8Array) {
  let array = []

  for (let i = 0; i < uint8Array.byteLength; i++) {
    array[i] = uint8Array[i]
  }

  return array
}

function drawPath(path: Graphics, data: number[]) {
  path.clear()
  path.moveTo(0, frameHeight + data[0])

  let cellSize = frameWidth / data.length

  for (let i = 1; i < data.length; i++) {
    path.lineTo(cellSize * i, frameHeight + data[i])
  }

  path.stroke({
    color: 0xffffff,
    width: 6
  })
}

function drawFreqPath(path: Graphics, data: number[]) {
  path.clear()
  path.moveTo(0, frameHeight)

  let log = true

  let cellSize = frameWidth / data.length

  if (!log) {
    for (let i = 1; i < data.length; i++) {
      path.lineTo(cellSize * i, (frameHeight) - (data[i] * 0.75))
    }
  } else {
    let max = Math.log(data.length)

    for (let i = 1; i < data.length; i++) {
      let x = Math.log(i) / max * frameWidth
      path.lineTo(x, (frameHeight) - (data[i] * 0.75))
    }
  }


  path.stroke({
    color: 0xffffff,
    width: 6
  })
}

const sound = ref(null)

function initSoundEvents() {
  sound.value.on('play', () => {
    audioPlaying.value = true
  })

  sound.value.on('stop', () => {
    audioPlaying.value = false
  })

  sound.value.on('pause', () => {
    audioPlaying.value = false
  })
}

function finalizeSeeking() {
  setTimeout(() => {
    userIsSeeking.value = false
  }, 50)

  let progress = Number(seekingProgress.value) / 100
  let shouldBeAt = sound.value.duration() * progress

  sound.value.seek(shouldBeAt)
}

function parseSecondsToPlayerTime(seconds: number) {
  let minute = Math.floor(seconds / 60)
  let second = Math.floor(seconds % 60)
  let milSeconds = Math.floor((seconds % 1) * 1000)

  let min_str = minute + ''
  let sec_str = second + ''
  let mil_str = milSeconds + ''

  if (minute < 10) {
    min_str = '0' + minute
  }

  if (second < 10) {
    sec_str = '0' + second
  }

  if (milSeconds < 100) {
    mil_str = '0' + milSeconds
  }

  if (milSeconds < 10) {
    mil_str = '00' + milSeconds
  }

  return min_str + ':' + sec_str + '.' + mil_str
}

onMounted(async () => {

  document.getElementById('seekBar').addEventListener('mousedown', () => {
    userIsSeeking.value = true
  })

  document.getElementById('seekBar').addEventListener('mouseup', () => {
    finalizeSeeking()
  })

  document.getElementById('seekBar').addEventListener('touchstart', () => {
    userIsSeeking.value = true
  })

  document.getElementById('seekBar').addEventListener('touchend', () => {
    finalizeSeeking()
  })

  changeSource(true)

  await app.init({
    resizeTo: document.getElementById('timeDomain'),
    resolution: window.devicePixelRatio || 1
  })

  const path = new Graphics()
  const pathFreq = new Graphics()

  path.filters = [new AsciiFilter({
    size: 6
  })]
  pathFreq.filters = [new AsciiFilter({
    size: 6
  })]

  app.stage.addChild(path)
  app.stage.addChild(pathFreq)

  document.getElementById('timeDomain').appendChild(app.canvas)

  app.ticker.add((time) => {
    if (audioContext.value) {
      let bufferLength = analyzer.frequencyBinCount
      let data = new Uint8Array(bufferLength)
      let dataFreq = new Uint8Array(bufferLength)
      analyzer.getByteTimeDomainData(data)
      analyzer.getByteFrequencyData(dataFreq)

      let timeDomainData = uint8ArrayToArray(data) // array range from 0 to 255

      let frequencyData = uint8ArrayToArray(dataFreq)

      frequencyData = frequencyData.map(el => {
        return el / 255 * frameHeight
      })

      // trim frequency data, we only need 20hz t0 20000hz
      // first we need to know the maximum frequency
      let maxFreq = analyzer.context.sampleRate / 2
      // then we need to know the frequency per bin
      let freqPerBin = maxFreq / bufferLength

      // cut the frequency data which is over 20000hz
      frequencyData = frequencyData.slice(0, Math.floor(20000 / freqPerBin))

      // cut the frequency data which is under 20hz
      frequencyData = frequencyData.slice(Math.floor(20 / freqPerBin))

      drawFreqPath(pathFreq, frequencyData)

      // map array data to 0 - 80
      timeDomainData = timeDomainData.map(el => {
        return el / 255 * (frameHeight)
      })

      drawPath(path, timeDomainData)
    }
  })
})

function play() {
  if (sound.value) {
    sound.value.play()
  }
}

function stopPlaying() {
  if (sound.value) {
    sound.value.stop()
  }
}

let audioPlaying = ref(false)

const playList = ref([
  {
    url: '/formula_1_theme.mp3',
    name: 'Formula 1 Theme',
    artist: 'Brian Tyler'
  },
  {
    url: '/cloud.mp3',
    name: '向云端 ',
    artist: '小霞，海洋Bo'
  },
  {
    url: '/lalala.mp3',
    name: 'La La La',
    artist: 'Faouzia'
  },
  {
    url: '/RAINY_NIGHT_IN_TALLINN_-_Ludwig_Goransson.mp3',
    name: 'Rainy Night In Tallinn',
    artist: 'Ludwig Goransson'
  },
  {
    url: '/Repeat_After_Me_-_Dimitri_Vegas_Like_Mike_&_Armin_van_Buuren.mp3',
    name: 'Repeat After Me',
    artist: 'Dimitri Vegas Like Mike, Armin van Buuren'
  },
  {
    url: 'The_One_(NGHTMRE_Remix)_-_Habstrakt.mp3',
    name: 'The One (NGHTMRE Remix)',
    artist: 'Habstrakt'
  },
  {
    url: '/sweep.mp3',
    name: 'Sine Wave Sweep (30hz - 20000hz)',
    artist: 'just for the demo, try use the maximum fftSize'
  }
])

let currentSong = ref(0)

function playNext() {
  currentSong.value++
  if (currentSong.value >= playList.value.length) {
    currentSong.value = 0
  }

  changeSource()
}

function changeSource(init: boolean = false) {
  if(sound.value){
    sound.value.unload()
  }

  sound.value = new Howl({
    src: [playList.value[currentSong.value].url]
  })

  initSoundEvents()

  initAnalyzer()

  let soundNode = sound.value._sounds[0]._node
  soundNode.disconnect()
  soundNode.connect(analyzer)

  if (!init) {
    sound.value.play()
  }
}

function playPrev() {
  currentSong.value--
  if (currentSong.value < 0) {
    currentSong.value = playList.value.length - 1
  }

  changeSource()
}

function updateSeek() {
  let duration = sound.value.duration()
  let currentTime = sound.value.seek()

  globalCurrentTime.value = currentTime
  globalDuration.value = duration

  if (!userIsSeeking.value) {
    seekingProgress.value = currentTime / duration * 100
  }
}

let globalDuration = ref(0)
let globalCurrentTime = ref(0)

setInterval(updateSeek, 1000 / 24)

let userIsSeeking = ref(false)
let seekingProgress = ref(0)

</script>

<style>
#timeDomain canvas {
  width: 100%;
  height: 100%;
}
</style>

<template>

  <div class="w-dvw h-dvh">
    <div class="w-dvw flex flex-wrap justify-center mb-2 pt-8">
      <div class="w-dvw flex flex-wrap justify-center">
        <div class="w-dvw text-center">
          fftSize: {{ fftSize }}
        </div>
        <div class="w-dvw text-center">
          <input type="range" min="5" max="15" v-model="ftSizeInPow">
        </div>
      </div>
    </div>

    <div class="w-dvw flex justify-center">
      <button class="bg-neutral-700 text-white rounded text-sm px-2 py-1 mx-1" @click="playPrev">Prev</button>
      <button class="bg-neutral-700 text-white rounded text-sm px-2 py-1 mx-1" @click="play" v-show="!audioPlaying">Play</button>
      <button class="bg-neutral-700 text-white rounded text-sm px-2 py-1 mx-1" @click="stopPlaying" v-show="audioPlaying">Stop</button>
      <button class="bg-neutral-700 text-white rounded text-sm px-2 py-1 mx-1" @click="playNext">Next</button>
    </div>

    <div class="flex flex-wrap mt-4">
      <div class="w-full text-center text-sm">
        {{ playList[currentSong].name }}
      </div>
      <div class="w-full text-center text-xs italic opacity-80">
        {{ playList[currentSong].artist }}
      </div>
    </div>
    <div class="mt-4 flex justify-center" id="seekBar">
      <input type="range" max="100" min="0" step="0.1" v-model="seekingProgress" class="w-[500px]">
    </div>
    <div class="mt-2 flex justify-center text-xs opacity-80 font-mono">
      {{ parseSecondsToPlayerTime(globalCurrentTime) }} / {{ parseSecondsToPlayerTime(globalDuration) }}
    </div>

    <div class="w-dvw flex justify-center mt-4">
      <div :style="{
        width: frameWidth + 'px',
        height: (frameHeight * 2) + 'px'
      }" id="timeDomain">

      </div>
    </div>
  </div>

</template>

<style scoped>

</style>
