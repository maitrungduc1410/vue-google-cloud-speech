<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png">
    <h1>Vue-Node-Google Cloud Speech Demo</h1>
    <div>
      <button @click="start">Start</button>
      <button @click="stop">Stop</button>
    </div>
    <div>
      <textarea rows="10" v-model="transcript"></textarea>
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client'

export default {
  data () {
    return {
      socket: null,
      transcript: '',
      isStreaming: false,
      context: null,
      processor: null,
      input: null,
      globalStream: null,
      bufferSize: 2048
    }
  },
  created () {
    this.socket = io('http://localhost:3001', {
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
      reconnectionAttempts: Infinity,
    })

    this.socket.on('connect', () => {
      console.log('connected')
    })

    this.socket.on('data', (data) => {
      this.transcript = data
    })
  },
  methods: {
    async setup () {
      this.context = new (window.AudioContext || window.webkitAudioContext)({
        // if Non-interactive, use 'playback' or 'balanced' // https://developer.mozilla.org/en-US/docs/Web/API/AudioContextLatencyCategory
        latencyHint: 'interactive',
      })
      this.processor = this.context.createScriptProcessor(this.bufferSize, 1, 1)
      this.processor.connect(this.context.destination)
      this.context.resume()

      const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
      this.globalStream = stream
      this.input = this.context.createMediaStreamSource(stream)
      this.input.connect(this.processor)

      this.processor.onaudioprocess = (e) => {
        this.microphoneProcess(e)
      }
    },
    microphoneProcess (e) {
      var left = e.inputBuffer.getChannelData(0)
      var left16 = this.downsampleBuffer(left, 44100, 16000)
      this.socket.emit('stream', left16)
    },
    downsampleBuffer (buffer, sampleRate, outSampleRate) {
      if (outSampleRate === sampleRate) {
        return buffer
      }
      if (outSampleRate > sampleRate) {
        throw new Error('downsampling rate show be smaller than original sample rate')
      }
      var sampleRateRatio = sampleRate / outSampleRate
      var newLength = Math.round(buffer.length / sampleRateRatio)
      var result = new Int16Array(newLength)
      var offsetResult = 0
      var offsetBuffer = 0
      while (offsetResult < result.length) {
        var nextOffsetBuffer = Math.round((offsetResult + 1) * sampleRateRatio)
        var accum = 0, count = 0
        for (var i = offsetBuffer; i < nextOffsetBuffer && i < buffer.length; i++) {
          accum += buffer[i]
          count++
        }

        result[offsetResult] = Math.min(1, accum / count) * 0x7FFF
        offsetResult++
        offsetBuffer = nextOffsetBuffer
      }
      return result.buffer
    },
    start () {
      this.setup()
      this.isStreaming = true
    },
    stop () {
      if (this.isStreaming) {
        // stop record track from browser
        let track = this.globalStream.getTracks()[0]
        track.stop()

        // stop recorder from browser
        this.input.disconnect(this.processor)
        this.processor.disconnect(this.context.destination)
        this.context.close().then(() => {
          this.input = null
          this.processor = null
          this.context = null
          this.socket.emit('stop')
        })

        this.isStreaming = false
      }
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

img {
  width: 100px;
}

button {
  margin: 0 10px;
  width: 100px;
  height: 30px;
  font-size: 20px;
}

textarea {
  resize: none;
  width: 500px;
  margin-top:30px;
  font-size: 24px;
}
</style>
