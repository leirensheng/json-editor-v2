<template>
  <select v-bind="$attrs" style="width:100px" v-model="type"

  class="select-type" @change="handleChange">
    <option v-for="one in types" :key="one.value" :value="one.value">{{ one.value }}</option>
  </select>
</template>

<script>
export default {
  name: 'select-type',
  data() {
    return {
      type: '',
      basicTypes: [
        { label: '字符串', value: 'string' },
        { label: '数字', value: 'number' },
        { label: '布尔值', value: 'boolean' },
        { label: 'null', value: 'null' },
        { label: '对象', value: 'object' },
        { label: '数组', value: 'array' },
        { label: '颜色', value: 'color' },
        { label: '日期', value: 'date' },
      ],
    };
  },
  props: {
    value: {},
    isRoot: {
      default: () => false,
    },
  },
  computed: {
    types() {
      return this.isRoot
        ? this.basicTypes.filter(one => ['object', 'array'].includes(one.value))
        : this.basicTypes;
    },
  },
  methods: {
    handleChange(e) {
      this.$emit('input', e.target.value);
    },

  },
  watch: {
    value: {
      handler(val) {
        this.type = val;
      },
      immediate: true,
    },
  },
};
</script>

<style scoped>
.select-type {
  position: relative;
  top: 2px;
}

select {
  border-color: #c0c4cc;
  border-radius: 4px;
  padding: 4px;
  color: #606266;
}
</style>
