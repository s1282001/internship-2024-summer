<script setup>
import OscillatorUI from './components/OscillatorUI.vue';
import FilterUI from './components/FilterUI.vue';
import AmpUI from './components/AmpUI.vue';
import WaveDisplay from './components/WaveDisplay.vue';
import SpectrumAnalyzer from './components/SpectrumAnalyzer.vue';
import parameterDescriptor from "./parameterDescriptor.js";
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
        <OscillatorUI ref="oscillatorUI" :frequency="currentFrequency" @parameterChanged="onParameterChanged" />
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
      synthesizer: null,
      analyser: null,
      waveDataSize: 512,
      currentFrequency: 440,  // Default frequency
    };
  },
  methods: {
    setup() {
      this.setupWorklet();
      this.setupMidi();
    },
    setupWorklet() {
      console.log("start");
      this.isStarted = true;
      window.AudioContext = window.AudioContext || window.webkitAudioContext;
      const context = new AudioContext();
      this.context = context;

      context.audioWorklet.addModule(MyWorkletProcessorUrl).then(() => {
        this.synthesizer = new AudioWorkletNode(context, 'synthesizer-worklet', {
          numberOfInputs: 0,
          numberOfOutputs: 1,
          outputChannelCount: [1]
        });

        this.analyser = this.context.createAnalyser();
        this.analyser.fftSize = this.waveDataSize;
        this.synthesizer.connect(this.analyser).connect(context.destination);
      });
    },
    onParameterChanged(value) {
      const data = { type: "param", value: value };
      this.synthesizer.port.postMessage(data);
    },
    noteOn() {
      const data = { type: "noteOn", value: true };
      this.synthesizer.port.postMessage(data);
    },
    noteOff() {
      const data = { type: "noteOn", value: false };
      this.synthesizer.port.postMessage(data);
    },
    setupMidi() {
      MidiHandler.init(); // Initialize Web MIDI
      MidiHandler.setHandleMidiCallback(this.processMidiMessage); // Set callback for MIDI messages
    },
    processMidiMessage(message) {
      const data = message.data;
      const [status, data1] = data;

      switch (status >> 4) {
        case 0x8:  // Note Off
          this.noteOff();
          break;

        case 0x9:  // Note On
          this.midiNoteOn(data1);
          break;

        default:
          console.log("Unsupported status byte.");
          break;
      }
    },
    midiNoteOn(noteNumber) {
      // Update frequency based on MIDI note
      this.currentFrequency = this.midiFrequency(noteNumber);
      // Emit the new frequency to the OscillatorUI
      this.$refs.oscillatorUI.frequencyChanged(Math.log(this.currentFrequency)); // Call the method directly
      this.noteOn();
    },
    midiFrequency(midiNote) {
      return 440 * Math.pow(2, (midiNote - 69) / 12); 
    },
  },
}
</script>

<style scoped></style>
