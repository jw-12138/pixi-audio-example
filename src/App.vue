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

let frameWidth = 600
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
}

onMounted(async () => {
  changeSource(true)
  initSoundEvents()

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
      frequencyData = frequencyData.slice(0, Math.floor(16000 / freqPerBin))

      // cut the frequency data which is under 20hz
      frequencyData = frequencyData.slice(Math.floor(0 / freqPerBin))

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
    artist: 'Fauzia'
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

let audioSourceInitiated = ref(false)
let audioSourceNode = null

function changeSource(init: boolean = false) {
  let isHTMLAudio = false
  if (sound.value) {
    isHTMLAudio = sound.value._sounds[0]._node instanceof HTMLAudioElement

    if (!isHTMLAudio) {
      sound.value.unload()
    } else {
      sound.value.stop()
    }
  } else {
    sound.value = new Howl({
      src: [playList.value[currentSong.value].url],
      html5: true
    })

    isHTMLAudio = true
  }

  if (isHTMLAudio) {
    sound.value._sounds[0]._node.src = playList.value[currentSong.value].url
    sound.value.load()
  } else {
    sound.value = new Howl({
      src: [playList.value[currentSong.value].url],
      html5: false
    })
  }


  initAnalyzer()

  let sound_node = sound.value._sounds[0]._node

  if (isHTMLAudio) {
    if (!audioSourceNode) {
      audioSourceNode = audioContext.value.createMediaElementSource(sound_node)
      audioSourceNode.connect(analyzer)

      audioSourceInitiated.value = true
    } else {
      audioSourceNode.disconnect()
      audioSourceNode.connect(analyzer)
    }

  } else {
    sound_node.disconnect()
    sound_node.connect(analyzer)
  }

  if (!init) {
    sound.value.play()
  }

  initSoundEvents()
}

function playPrev() {
  currentSong.value--
  if (currentSong.value < 0) {
    currentSong.value = playList.value.length - 1
  }

  changeSource()
}

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
