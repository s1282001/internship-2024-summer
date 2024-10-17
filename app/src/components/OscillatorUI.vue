<script setup>
import { ref, watch, defineProps, defineEmits } from 'vue';
import parameterDescriptor from "../parameterDescriptor.js";

// Define props
const props = defineProps({
  frequency: {
    type: Number,
    required: true,
  },
});

// Define emits
const emit = defineEmits(['parameterChanged']);

// Define reactive properties
const params = parameterDescriptor.parameters;
const oscTypes = parameterDescriptor.oscTypes;

const oscType = ref(parameterDescriptor.parameters.oscType.defaultValue);

// For log scale slider
const logFreq = ref(Math.log(props.frequency));
const minLogFreq = ref(Math.log(parameterDescriptor.parameters.frequency.minValue));
const maxLogFreq = ref(Math.log(parameterDescriptor.parameters.frequency.maxValue));

// Watch for frequency changes to update logFreq
watch(() => props.frequency, (newVal) => {
  logFreq.value = Math.log(newVal);
});

// Emit frequency change
const frequencyChanged = () => {
  const freq = Math.round(Math.exp(logFreq.value));
  if (freq !== props.frequency) {
    emit("parameterChanged", { id: params.frequency.id, value: freq });
  }
};

const oscTypeChanged = () => {
  emit("parameterChanged", { id: params.oscType.id, value: oscType.value });
};

const oscTypeName = (i) => {
  const key = Object.keys(oscTypes).find(key => oscTypes[key].index === i - 1);
  return oscTypes[key].name;
};
</script>

<template>
  <div> <!-- Added a single root element -->
    <h3 class="section-title">Oscillator</h3>
    <div class="section">
      <div class="param" id="oscType">
        <h5>{{ params.oscType.name }}</h5>
        <select v-model="oscType" @change="oscTypeChanged">
          <option v-for="i in Object.keys(oscTypes).length" :key="i" :value="i - 1">
            {{ oscTypeName(i) }}
          </option>
        </select>
      </div>
      <div class="param" id="oscFrequency">
        <h5>{{ params.frequency.name }}</h5>
        <input type="range" :min="minLogFreq" :max="maxLogFreq" :step="(maxLogFreq - minLogFreq) / 10000"
          v-model="logFreq" @input="frequencyChanged" />
        <div>{{ Math.round(Math.exp(logFreq)) }} Hz</div>
      </div>
    </div>
  </div> <!-- Single root element -->
</template>

<style scoped>
.section {
  margin: 16px;
}

.param {
  margin-bottom: 16px;
}

.section-title {
  font-size: 1.5rem;
  margin-bottom: 12px;
}

h5 {
  margin: 8px 0;
}
</style>

