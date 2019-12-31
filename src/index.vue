<template>
  <div
    class="json-editor"
    @scroll="handleScroll"
  >
    <div
      class="fake-content"
      :style="{ height: contentHeight }"
    ></div>
    <div
      class="real-content"
      :style="{ transform: `translateY(${topOffset}px)` }"
    >
      <div
        v-for="(item,index) in visibleData"
        :key="item.uniqueKey"
      >
        <my-item
          :bindKey="item.key"
          :item="item"
          :json="value"
          v-bind="$attrs"
          @deleteChildren="handleDeleteChildren(index,item)"
          @addAddBtn="(clickItem,parent)=>addAddBtn(index,clickItem,parent)"
          @addItem="addItem(index,item)"
          @deleteItem="deleteItem(index,item)"
          @changeRootType="changeRootType"
        ></my-item>
      </div>
    </div>
  </div>
</template>
<script>
import { getType } from './utils';
import MyItem from './components/item.vue';

const ITEMHEIGHT = 53;
/* eslint-disable no-param-reassign */
/* eslint-disable no-restricted-syntax */

export default {
  components: {
    MyItem,
  },
  data() {
    return {
      flattenData: [],
      // offset: 0,
      startRenderIndex: 0, // dom上开始渲染的元素索引
      curScrollTop: 0,
      oneScreenCnt: 0, // 当前屏幕最多可见数
    };
  },
  props: {
    value: {
      default: () => ({}),
    },
    jsonName: {
      default: () => 'root',
    },
    option: {
      // 配置对象
      type: Object,
      default: () => ({
        topRetainCount: 10, // 上面最小保留
        cacheCount: 20, // 缓冲数
      }),
    },
  },
  computed: {
    topMaxCount() {
      return this.option.cacheCount + this.option.topRetainCount;
    },
    topOffset() {
      return this.startRenderIndex * ITEMHEIGHT;
    },
    renderCount() {
      return this.topMaxCount + Math.ceil(this.oneScreenCnt * 1.5);
    },
    contentHeight() {
      return `${this.flattenData.length * ITEMHEIGHT}px`;
    },
    levels() {
      return this.flattenData.map(one => one.level);
    },
    visibleData() {
      return this.flattenData.slice(this.startRenderIndex, this.startRenderIndex + this.renderCount);
    },
  },
  methods: {
    getOneScreenCnt() {
      this.oneScreenCnt = Math.floor(document.body.offsetHeight / ITEMHEIGHT);
    },
    changeRootType(val) {
      this.$emit('input', val);
      this.refresh(val);
    },
    refresh(val) {
      this.$nextTick(() => {
        this.flattenData = [];
        this.getFlattenData(val);
      });
    },
    getRandomId() {
      return Math.random()
        .toString(16)
        .slice(-6);
    },
    handleDeleteChildren(index, { level }) {
      let count = 0;
      const indexOfTree = this.startRenderIndex + index;
      const startIndex = indexOfTree + 1;
      let curIndex = startIndex;
      while (this.flattenData[curIndex] && this.flattenData[curIndex].level > level) {
        count += 1;
        curIndex += 1;
      }
      this.flattenData.splice(startIndex, count);
      delete this.flattenData[indexOfTree].children;
    },

    addAddBtn(index, item, parent) {
      const addBtn = {
        level: item.level + 1,
        parent,
        indexOfParent: 0,
        isAddBtn: true,
        uniqueId: this.getRandomId(),
      };
      const addIndex = index + this.startRenderIndex + 1;
      this.flattenData.splice(addIndex, 0, addBtn);
    },
    deleteItem(index, {
      parent, key, level, indexOfParent,
    }) {
      const isArray = Array.isArray(parent);
      if (isArray) {
        parent.splice(indexOfParent, 1);
      } else {
        this.$delete(parent, key);
      }
      const indexOfTree = this.startRenderIndex + index;
      let curIndex = indexOfTree + 1;
      let deleteCount = 1;
      while (this.flattenData[curIndex].level > level) {
        deleteCount += 1;
        curIndex += 1;
      }

      while (this.flattenData[curIndex] && this.flattenData[curIndex].level >= level) {
        const curItem = this.flattenData[curIndex];

        if (curItem.level === level) {
          curItem.indexOfParent -= 1;
          if (isArray) {
            curItem.key -= 1;
          }
        }
        curIndex += 1;
      }
      this.flattenData.splice(indexOfTree, deleteCount);
    },
    getType,
    handleScroll(e) {
      const { scrollTop } = e.target;
      const hasHiddenCnt = Number(Math.floor(scrollTop / ITEMHEIGHT));
      if (hasHiddenCnt - this.startRenderIndex > this.topMaxCount) {
        this.startRenderIndex = hasHiddenCnt - this.option.topRetainCount;
      } else if ((this.startRenderIndex > 0) && (hasHiddenCnt - this.startRenderIndex < this.option.topRetainCount)) {
        this.startRenderIndex = (hasHiddenCnt - this.topMaxCount < 0) ? 0 : (hasHiddenCnt - this.topMaxCount);
      }
    },


    addItem(index, item) {
      const {
        level, parent, indexOfParent,
      } = item;
      const indexOfTree = this.startRenderIndex + index;
      const isArray = Array.isArray(parent);
      if (!isArray) {
        const preItem = this.flattenData[indexOfTree - 1];
        if (preItem.level === level && preItem.key === '') {
          this.$notification({
            message: '请先填写空的字段',
            type: 'warning',
          });
          return;
        }
      }

      const newItem = {
        value: '',
        level,
        parent,
        type: 'string',
        indexOfParent,
        key: isArray ? indexOfParent : '',
        uniqueId: this.getRandomId(),
      };

      item.indexOfParent += 1;
      if (isArray) {
        parent.push('');
      }

      this.flattenData.splice(indexOfTree, 0, newItem);
    },
    // parent 和 children都是保留原数据，非扁平化的
    getFlattenData(json) {
      const flat = ({
        val, key, level, res, parent, indexOfParent,
      }) => {
        const type = this.getType(val);
        const isObject = type === 'object';
        const isArray = type === 'array';
        const isArrayOrObject = isObject || isArray;
        const one = {
          key,
          level,
          parent,
          type,
          indexOfParent,
          uniqueId: this.getRandomId(),
        };
        if (isArrayOrObject) {
          one.children = val;
        } else {
          one.value = val;
        }
        res.push(one);

        if (isArrayOrObject) {
          const arr = isObject ? Object.keys(val) : val;
          const childLevel = level + 1;
          arr.forEach((item, index) => {
            flat({
              val: isObject ? val[item] : item,
              key: isObject ? item : index,
              level: childLevel,
              res,
              parent: val,
              indexOfParent: index,
            });
          });

          res.push({
            level: childLevel,
            parent: val,
            indexOfParent: arr.length,
            isAddBtn: true,
            uniqueId: this.getRandomId(),
          });
        }
      };

      flat({
        val: json,
        key: this.jsonName,
        res: this.flattenData,
        level: 0,
      });
    },
  },
  created() {
    this.getFlattenData(this.value);
  },
  mounted() {
    this.getOneScreenCnt();
  },
};
</script>

<style scoped>
.json-editor {
  position: relative;
  height: 100%;
  max-height: 90vh;
  overflow-y: scroll;
}
.fake-content {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
}
.real-content{
  position: absolute;
  left: 0;
  top: 0;
}

</style>
