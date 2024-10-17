<script setup>
import OscillatorUI from './components/OscillatorUI.vue';
import FilterUI from './components/FilterUI.vue';
import AmpUI from './components/AmpUI.vue';
import WaveDisplay from './components/WaveDisplay.vue';
import SpectrumAnalyzer from './components/SpectrumAnalyzer.vue';
import parameterDescriptor from "./parameterDescriptor.js"
import MyWorkletProcessorUrl from '../public/SynthesizerWorklet.js?worker&url';
import MidiHandler from "./MidiHandler.js";
</script>

<template>
  <div class="whole">
    <div class="title">
      <a class="logo" href="https://www.korg.com/jp/" target="_blank">
        <img class="logo-img" src="./assets/korg.jpg" />
      </a>
      <h2 class="title-text">
        Internship Synthesizer
      </h2>
    </div>
    <div>
      <button id="start-button" v-show="!isStarted" @click="setup">
        start
      </button>
    </div>
    <div v-show="isStarted">
      <div>
        <input id="noteOn-button" type="button" value="Note On" @mousedown="noteOn" @mouseup="noteOff" />
      </div>
      <div>
        <WaveDisplay ref="wave" :analyser="analyser" />
        <SpectrumAnalyzer ref="spectrum" :analyser="analyser" />
      </div>
      <div>
        <OscillatorUI @parameterChanged="onParameterChanged" />
        <FilterUI @parameterChanged="onParameterChanged" />
        <AmpUI @parameterChanged="onParameterChanged" />
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "MainUI",
  data() {
    return {
      isStarted: false,
      context: null,
      sampleRate: 48000,
      synthesizer: null,
      analyser: null,
      waveDataSize: 512,
      params: parameterDescriptor.parameters
    }
  },
  methods: {
    setup() {
      this.setupWorklet();
      this.setupMidi();
    },
    setupWorklet() {
      console.log("start")
      this.isStarted = true
      window.AudioContext = window.AudioContext || window.webkitAudioContext;
      const context = new AudioContext()
      this.sampleRate = context.sampleRate
      this.context = context

      let options = {
        processorOptions: {
          sampleRate: this.context.sampleRate,
        },
        numberOfInputs: 0,
        numberOfOutputs: 1,
        outputChannelCount: [1]
      }
      context.audioWorklet.addModule(MyWorkletProcessorUrl).then(() => {
        this.synthesizer = new AudioWorkletNode(context, 'synthesizer-worklet', options)

        console.log(this.synthesizer)

        this.analyser = this.context.createAnalyser();
        this.analyser.maxDecibels = 0;
        this.analyser.fftSize = this.waveDataSize;
        this.synthesizer.connect(this.analyser).connect(context.destination)

        setInterval(this.draw, 1000 / 10) //  オシロスコープ描画用タイマー
      })
    },
    onParameterChanged(value) {
      const data = { type: "param", value: value }
      this.synthesizer.port.postMessage(data)
    },
    noteOn() {
      console.log("noteOn")
      const data = { type: "noteOn", value: true }
      this.synthesizer.port.postMessage(data)
    },
    noteOff() {
      const data = { type: "noteOn", value: false }
      this.synthesizer.port.postMessage(data)
    },
    setupMidi() {
      MidiHandler.init(); // Web MIDIのセットアップ
      MidiHandler.setHandleMidiCallback(this.processMidiMessage); // processMidiMessageをMIDI受信時のコールバックに設定
    },
    processMidiMessage(message) {
      // console.log(message);
      const data = message.data; // データを取得 Uint8Array（8bit符号なし整数値の配列)
      const [status, data1, data2] = data; // 各要素を変数に格納　先頭がステータスバイト、それ以降がデータバイト
      //各データバイトは1byte（8bit）だが先頭の1bitはデータバイトであることを示すフラグで常に０なので得られる値は7bit(0~127)である
      console.log(`status-byte: ${status}, data-byte-1: ${data1}, data-byte-2: ${data2}`)

      switch (status >> 4) {
        case 0x8:  // 8n hex(nはMIDIチャンネル)はノートオフを指す
          //  ノートオフメッセージの場合も、data1：ノート番号、data2：ベロシティ
          this.midiNoteOff(data1, data2);
          break;

        case 0x9:  // 9n hex(nはMIDIチャンネル)はノートオンを指す
          //  ノートオンメッセージの場合、data1：ノート番号、data2：ベロシティ
          this.midiNoteOn(data1, data2);
          break;

        default:
          console.log("This status byte is not supported in this app.");
          break;
      }
    },
    midiNoteOff(noteNumber, velocity) {
      this.noteOff();
    },
    midiNoteOn(noteNumber, velocity) {
      // todo, support note number and velocity
      this.midiFrequency(noteNumber);

      this.noteOn();
    },
    midiFrequency(noteNumber){
      return 440*math.pow(2,((noteNumber-69)/12);
    },
    draw() {
      this.$refs.spectrum.drawSpectrum()
    }
  },
}
</script>

<style scoped></style>
