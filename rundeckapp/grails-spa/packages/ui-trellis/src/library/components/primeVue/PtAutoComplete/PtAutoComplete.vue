<template>
  <AutoComplete
    v-model="value"
    :suggestions="suggestions"
    :optionLabel="optionLabel"
    :name="name"
    :placeholder="placeholder"
    :invalid="invalid"
    @complete="onComplete"
    @change="onChange"
    @input="updateValue"
  ></AutoComplete>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import AutoComplete, {
  AutoCompleteCompleteEvent,
  AutoCompleteChangeEvent,
} from "primevue/autocomplete";

export default defineComponent({
  name: "PtAutoComplete",
  // eslint-disable-next-line vue/no-reserved-component-names
  components: { AutoComplete },
  props: {
    modelValue: {
      type: String,
      required: true,
    },
    suggestions: {
      type: Array,
      default: () => [],
    },
    defaultValue: {
      type: String,
      default: "",
    },
    name: {
      type: String,
      default: "",
    },
    optionLabel: {
      type: String,
      default: "label",
    },
    invalid: {
      type: Boolean,
      default: false,
    },
    placeholder: {
      type: String,
      default: "",
    },
  },
  emits: ["update:modelValue", "onChange", "onComplete"],
  data() {
    return {
      value: this.modelValue || this.defaultValue,
    };
  },
  watch: {
    modelValue(newVal) {
      this.value = newVal;
    },
  },
  methods: {
    onComplete(event: AutoCompleteCompleteEvent) {
      this.$emit("onComplete", event);
    },
    onChange(event: AutoCompleteChangeEvent) {
      this.$emit("onChange", event);
    },
    updateValue() {
      this.$emit("update:modelValue", this.value);
    },
  },
});
</script>

<style lang="scss">
.p-autocomplete {
  width: 100%;

  .p-inputtext {
    background-color: var(--colors-white);
    border: 1px solid var(--colors-gray-500);
    color: var(--colors-gray-900);
    width: inherit;

    &::placeholder {
      color: var(--colors-gray-600);
    }

    &:enabled:hover {
      border-color: var(--colors-blue-500);
    }
    &:enabled:focus {
      border-color: var(--colors-blue-500);
      box-shadow: 0 0 0 0.2rem var(--colors-blue-100);
    }

    /* Error State */
    &.p-invalid {
      border-color: var(--colors-red-500);
    }
  }
}
</style>
