<script setup lang="ts">
import * as monaco from "monaco-editor";
import loader from "@monaco-editor/loader";
import {ref, watch} from "vue";

const props = defineProps({
  modelValue: {
    type: String,
    required: false,
    default: '',
  },
  height: {
    type: Number,
    required: false,
    default: 500
  },
  language: {
    type: String,
    required: false,
    default: 'json'
  },
  readOnly: {
    type: Boolean,
    required: false,
    default: false
  },
  theme: {
    type: String,
    required: false,
    default: 'vs-dark'
  }
})

const container = ref<HTMLElement | null>(null)
let instance: monaco.editor.IStandaloneCodeEditor | null = null;

const emit = defineEmits(['update:modelValue'])

loader.init().then(monaco => {
  if (container.value != null) {
    instance = monaco.editor.create(container.value, {
      value: props.modelValue,
      language: props.language,
      readOnly: props.readOnly,
      theme: props.theme,
    })

    instance.onDidChangeModelContent((event: any) => {
      if (instance) {
        const content = instance.getValue();
        emit('update:modelValue', content)
      }
    });
  }
})

watch(
  () => props.modelValue,
  (newValue) => {
    if (instance && newValue !== instance.getValue()) {
      instance.setValue(newValue);
    }
  }
)
</script>

<template>
  <div ref="container" :style="'width: 100%; height: '+height+'px'"></div>
</template>

<style scoped lang="sass">

</style>
