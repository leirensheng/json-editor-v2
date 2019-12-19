<template>
  <div :style="{marginLeft: level*30+'px',position:'relative'}">
    <span
      v-for="one in levelArr"
      :key="one"
      class="left-line"
      :style="getStyle(one)"
    ></span>

    <div
      v-if="type"
      class="one-item"
    >
      <XLine
        v-if="!isRoot"
        style="position:relative;top:-1px"
      />
      <span
        v-else
        class="has-margin"
      />
      <!-- 字段名 -->
      <my-input
        :value="isRoot ? jsonName : bindKey"
        placeholder="字段"
        type="string"
        :disabled="isRoot || typeof bindKey === 'number'"
        @input="updateKey"
        :validateFunc="validateKey"
      ></my-input>
      <!-- 类型 -->
      <span style="padding:0 10px">=</span>
      <select-type
        :disabled="isRoot && rootType !== ''"
        class="slect-type"
        :isRoot="isRoot"
        :value="type"
        @input="updateType"
      />
      <!-- 值 -->
      <span style="margin-left:10px;display:flex;align-content:center">
        <my-input
          size="mini"
          type="string"
          style="width:100px;"
          v-if="type === 'string'"
          :value="value"
          placeholder="值"
          @input="updateVal"
        />

        <my-input
          placeholder="值"
          v-if="type === 'number'"
          :value="value"
          @input="updateVal"
        ></my-input>

        <el-select
          size="mini"
          style="width:100px;"
          v-if="type === 'boolean'"
          :value="String(value)"
          @input="updateVal"
        >
          <el-option value="true">true</el-option>
          <el-option value="false">false</el-option>
        </el-select>

        <el-color-picker
          v-if="type === 'color'"
          :value="value"
          @input="updateVal"
          size="mini"
        />

        <el-date-picker
          :value="value"
          @input="updateVal"
          type="date"
          size="mini"
          :clearable="false"
          style="width:165px;"
          v-if="type === 'date'"
          placeholder="选择日期"
          format="yyyy 年 MM 月 dd 日"
          value-format="yyyy-MM-dd"
        >
        </el-date-picker>
      </span>


      <delete-btn
        class="delete-btn"
        v-if="!isRoot"
        @click="() => this.$emit('deleteItem')"
      />
    </div>
    <add-btn
      style="padding-top:25px"
      v-if="!type"
      @click="()=>this.$emit('addItem')"
    ></add-btn>

  </div>
</template>

<script>
import DeleteBtn from './delete-btn.vue';
import XLine from './x-line.vue';
import SelectType from './types.vue';
import MyInput from './my-input.vue';
import AddBtn from './add-btn.vue';

const ITEMHEIGHT = 53;
const PADDINGTOP = 25;
const MARGINLEFT = 30;
export default {
  data() {
    return {
      rootType: '',
    };
  },
  props: {
    bindKey: {
      type: [String, Number],
    },
    value: {},
    jsonName: {
      default: () => 'root',
    },
    level: {},
    type: {},
    item: {},

    json: {},
  },
  computed: {
    isRoot() {
      return this.level === 0;
    },
    levelArr() {
      return Array.from({ length: this.level }, (val, index) => index + 1);
    },
  },
  mounted() {
    console.log('mounted');
  },
  components: {
    XLine,
    SelectType,
    DeleteBtn,
    MyInput,
    AddBtn,
  },
  methods: {
    getStyle(idx) {
      const { length } = this.levelArr;
      const isFirstChild = idx === length && this.item.indexOfParent === 0;
      const height = isFirstChild ? PADDINGTOP + (ITEMHEIGHT - PADDINGTOP) / 2 : ITEMHEIGHT;
      const top = isFirstChild ? 0 : -PADDINGTOP / 2;
      const left = -(length - idx) * MARGINLEFT;
      return { height: `${height}px`, left: `${left}px`, top: `${top}px` };
    },
    updateVal(val) {
      const { parent, key } = this.item;
      const newVal = this.type === 'boolean' ? val !== 'false' : val;
      this.item.value = newVal;
      this.$set(parent, key, newVal);
    },
    validateKey(val) {
      const { parent } = this.item;
      const keys = Object.keys(parent);
      let message = '';
      if (val === '') {
        message = '字段名不能为空';
      } else if (val !== this.bindKey && keys.includes(val)) {
        message = `已经存在字段${val}`;
      }
      if (message) {
        this.$notification({
          type: 'warning',
          message,
        });
        return false;
      }
      return true;
    },

    updateKey(val) {
      const { parent, children, value } = this.item;
      delete parent[this.bindKey];
      this.item.key = val;
      this.$set(parent, val, children || value);
    },

    getTypeValue(type) {
      let value;
      if (type === 'string' && this.type === 'number') {
        value = String(this.value);
        // eslint-disable-next-line no-restricted-globals
      } else if (type === 'number' && !isNaN(this.value - 0)) {
        value = Number(this.value);
      } else {
        const maps = {
          array: [],
          object: {},
          string: '',
          number: 0,
          boolean: false,
          null: null,
          color: '#000',
          date: new Date().toLocaleDateString().replace(/\//g, '-'),
        };
        value = maps[type];
      }
      return value;
    },
    updateType(val) {
      const value = this.getTypeValue(val);

      const { parent, key, level } = this.item;
      if (level === 0) {
        this.$emit('changeRootType', value);
        return;
      }
      if (['object', 'array'].includes(this.type)) {
        this.$emit('deleteChildren');
      }

      if (['object', 'array'].includes(val)) {
        this.$emit('addAddBtn', this.item, value);
      }
      this.$set(parent, key, value);
      if (['object', 'array'].includes(val)) {
        delete this.item.value;
        this.item.children = value;
      } else {
        this.item.value = value;
      }
      this.item.type = val;
    },
  },
};
</script>

<style scoped>
.left-line {
  position: absolute;
  /* height: 180px; */
  width: 1px;
  /* top: 0; */
  /* left:30px; */
  background-color: black;
}
.has-border {
  border-left: 1px solid rgb(219, 203, 203);
}
.delete-btn {
  margin-left: 10px;
}
.select-type {
  position: relative;
  top: -0.5px;
}
.one-item {
  position: relative;
  padding-top: 25px;
  top: -1px;
  display: flex;
  align-items: center;
  /* border-left: 1px solid red; */
}
.has-margin {
  margin-left: 30px;
}
</style>
