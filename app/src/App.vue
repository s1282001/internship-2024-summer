<template>
  <div class="whole">
    <div class="title">
      <a class="logo" href="https://www.korg.com/jp/" target="_blank">
        <img class="logo-img" src="./assets/korg.jpg" />
      </a>
      <h2 class="title-text">Internship Synthesizer</h2>
    </div>
    <div>
      <button id="start-button" v-show="!isStarted" @click="setup">start</button>
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

<script setup>
import OscillatorUI from './components/OscillatorUI.vue';
import FilterUI from './components/FilterUI.vue';
import AmpUI from './components/AmpUI.vue';
import WaveDisplay from './components/WaveDisplay.vue';
import SpectrumAnalyzer from './components/SpectrumAnalyzer.vue';
import parameterDescriptor from "./parameterDescriptor.js";
import MyWorkletProcessorUrl from '../public/SynthesizerWorklet.js?worker&url';
import MidiHandler from "./MidiHandler.js";

const isStarted = ref(false);
const context = ref(null);
const sampleRate = ref(48000);
const synthesizer = ref(null);
const analyser = ref(null);
const waveDataSize = 512;
const params = parameterDescriptor.parameters;

function setup() {
  setupWorklet();
  setupMidi();
}

function setupWorklet() {
  console.log("start");
  isStarted.value = true;
  window.AudioContext = window.AudioContext || window.webkitAudioContext;
  const audioContext = new AudioContext();
  sampleRate.value = audioContext.sampleRate;
  context.value = audioContext;

  const options = {
    processorOptions: {
      sampleRate: audioContext.sampleRate,
    },
    numberOfInputs: 0,
    numberOfOutputs: 1,
    outputChannelCount: [1],
  };

  audioContext.audioWorklet.addModule(MyWorkletProcessorUrl).then(() => {
    synthesizer.value = new AudioWorkletNode(audioContext, 'synthesizer-worklet', options);
    console.log(synthesizer.value);

    analyser.value = audioContext.createAnalyser();
    analyser.value.maxDecibels = 0;
    analyser.value.fftSize = waveDataSize;
    synthesizer.value.connect(analyser.value).connect(audioContext.destination);

    setInterval(draw, 1000 / 10); // Timer for oscilloscope drawing
  });
}

function onParameterChanged(value) {
  const data = { type: "param", value: value };
  synthesizer.value.port.postMessage(data);
}

function noteOn() {
  console.log("noteOn");
  const data = { type: "noteOn", value: true };
  synthesizer.value.port.postMessage(data);
}

function noteOff() {
  const data = { type: "noteOn", value: false };
  synthesizer.value.port.postMessage(data);
}

function setupMidi() {
  MidiHandler.init(); // Setup Web MIDI
  MidiHandler.setHandleMidiCallback(processMidiMessage); // Set MIDI message callback
}

function processMidiMessage(message) {
  const data = message.data; // Retrieve data
  const [status, data1, data2] = data; // Destructure data bytes
  console.log(`status-byte: ${status}, data-byte-1: ${data1}, data-byte-2: ${data2}`);

  switch (status >> 4) {
    case 0x8: // Note Off
      midiNoteOff(data1, data2);
      break;

    case 0x9: // Note On
      midiNoteOn(data1, data2);
      break;

    default:
      console.log("This status byte is not supported in this app.");
      break;
  }
}

function midiNoteOff(noteNumber, velocity) {
  noteOff();
}

function midiNoteOn(noteNumber, velocity) {
  midiFrequency(noteNumber);
  noteOn();
}

function midiFrequency(noteNumber) {
  const frequency = 440 * Math.pow(2, ((noteNumber - 69) / 12));
  return frequency; // This value could be used for further processing
}

function draw() {
  $refs.spectrum.drawSpectrum();
}

</script>

<style scoped></style>
