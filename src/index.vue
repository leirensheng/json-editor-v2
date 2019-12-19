<template>
  <div
    class="json-editor"
    @scroll="handleScroll"
  >
    <div
      class="fake-content"
      :style="{ height: contentHeight }"
    >----------------------</div>
    <div
      :style="{ transform: `translateY(${offset}px)` }"
    >
      <div
        v-for="(item,index) in visibleData"
        :key="item.uniqueKey"
      >
        <my-item
          :level="item.level"
          :value="item.value"
          :bindKey="item.key"
          :item="item"
          :type="item.type"
          :json="value"
          @deleteChildren="handleDeleteChildren(index,item)"
          @addAddBtn="(clickItem,parent )=>addAddBtn(index,clickItem,parent)"
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
      flattenTree: [],
      visibleData: [],
      offset: 0,
      startIndex: 0, // 页面上可见的开始元素的索引
      startRenderIndex: 0, // dom上开始渲染的元素索引
      curScrollTop: 0,
    };
  },
  props: {
    value: {
      default: () => ({}),
    },
    option: {
      // 配置对象
      type: Object,
      default: () => ({
        renderCount: 60,
        topRetainCount: 17,
      }),
    },
  },
  computed: {
    contentHeight() {
      return `${this.flattenTree.length * ITEMHEIGHT}px`;
    },
    levels() {
      return this.flattenTree.map(one => one.level);
    },
  },
  methods: {
    changeRootType(val) {
      this.$emit('input', val);
      this.refresh(val);
    },
    refresh(val) {
      this.$nextTick(() => {
        this.flattenTree = [];
        this.getFlatten(val);
        this.updateVisibleData();
      });
    },
    getRandomId() {
      return Math.random()
        .toString(16)
        .slice(-6);
    },
    handleDeleteChildren(index, item) {
      let count = 0;
      const startIndex = index + 1 + this.startRenderIndex;
      let curIndex = startIndex;
      const { level } = item;
      while (this.flattenTree[curIndex] && this.flattenTree[curIndex].level > level) {
        const cur = this.flattenTree[curIndex];
        delete cur.children;
        delete cur.parent;
        count += 1;
        curIndex += 1;
      }
      this.flattenTree.splice(startIndex, count);
      delete this.flattenTree[index].children;
      this.updateVisibleData(this.curScrollTop);
    },

    addAddBtn(index, item, parent) {
      const addBtn = {
        level: item.level + 1,
        parent,
        indexOfParent: 0,
        isAddBtn: true,
        uniqueId: this.getRandomId(),
      };
      const startIndex = index + this.startRenderIndex + 1;
      this.flattenTree.splice(startIndex, 0, addBtn);
      this.updateVisibleData(this.curScrollTop);
    },
    deleteItem(index, item) {
      const {
        parent, key, level, indexOfParent,
      } = item;
      const isArray = Array.isArray(parent);
      if (isArray) {
        parent.splice(indexOfParent, 1);
      } else {
        this.$delete(parent, key);
      }
      const indexOfTree = this.startRenderIndex + index;
      let curIndex = indexOfTree + 1;
      let count = 1;
      while (this.flattenTree[curIndex].level > level) {
        count += 1;
        curIndex += 1;
      }

      while (this.flattenTree[curIndex] && this.flattenTree[curIndex].level >= level) {
        const curItem = this.flattenTree[curIndex];

        if (curItem.level === level) {
          curItem.indexOfParent -= 1;
          if (isArray) {
            curItem.key -= 1;
          }
        }
        curIndex += 1;
      }
      this.flattenTree.splice(indexOfTree, count);

      this.updateVisibleData(this.curScrollTop);
    },
    getType,
    handleScroll(e) {
      const { scrollTop } = e.target;
      window.requestAnimationFrame(() => {
        this.updateVisibleData(scrollTop);
        this.curScrollTop = scrollTop;
      });
    },
    // 在dom上的元素
    updateVisibleData(scrollTop = 0) {
      const start = Math.floor(scrollTop / ITEMHEIGHT);
      const startIndex = Number(start);
      this.startIndex = startIndex; // 视图中可见索引

      let startRenderIndex = startIndex; // 开始渲染的索引
      startRenderIndex = startIndex > this.option.topRetainCount ? (startIndex - this.option.topRetainCount) : 0;
      this.startRenderIndex = startRenderIndex;
      const end = startRenderIndex + this.option.renderCount;
      const diff = startRenderIndex - startIndex;

      this.visibleData = this.flattenTree.slice(startRenderIndex, end);
      this.offset = (startIndex + diff) * ITEMHEIGHT;
    },

    addItem(index, item) {
      const {
        level, parent, indexOfParent,
      } = item;
      const indexOfTree = this.startRenderIndex + index;
      const isArray = Array.isArray(parent);
      if (!isArray) {
        const preItem = this.flattenTree[indexOfTree - 1];
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

      this.flattenTree.splice(indexOfTree, 0, newItem);
      this.updateVisibleData(this.curScrollTop);
    },
    // parent 和 children都是保留原数据，非扁平化的
    getFlatten(json) {
      const flat = ({
        val, key, level, res, parent, indexOfParent,
      }) => {
        const type = this.getType(val);

        if (type === 'object') {
          const keys = Object.keys(val);
          res.push({
            key,
            level,
            parent,
            indexOfParent,
            children: val,
            type,
            uniqueId: this.getRandomId(),
          });
          const childLevel = level + 1;

          keys.forEach((objKey, index) => {
            flat({
              val: val[objKey],
              key: objKey,
              level: childLevel,
              res,
              parent: val,
              indexOfParent: index,
            });
          });

          res.push({
            level: childLevel,
            parent: val,
            indexOfParent: keys.length,
            isAddBtn: true,
            uniqueId: this.getRandomId(),
          });
        } else if (type === 'array') {
          res.push({
            key,
            level,
            parent,
            indexOfParent,
            children: val,
            type,
            uniqueId: this.getRandomId(),

          });
          const childLevel = level + 1;
          val.forEach((one, index) => {
            flat({
              val: one,
              key: index,
              level: childLevel,
              res,
              parent: val,
              indexOfParent: index,
            });
          });

          res.push({
            level: childLevel,
            parent: val,
            indexOfParent: val.length,
            isAddBtn: true,
            uniqueId: this.getRandomId(),
          });
        } else {
          res.push({
            key,
            value: val,
            level,
            parent,
            type,
            indexOfParent,
            uniqueId: this.getRandomId(),
          });
        }
      };

      flat({
        val: json,
        key: '',
        res: this.flattenTree,
        level: 0,
        parent: null,
        indexOfParent: null,
      });
    },
  },
  created() {
    console.time(1);
    this.getFlatten(this.value);
  },
  mounted() {
    this.updateVisibleData();
    console.timeEnd(1);
  },
};
</script>

<style scoped>
.json-editor {
  position: relative;
  height: 1080px;
  max-height: 90vh;
  overflow-y: scroll;
  /* padding: 10px 0; */
}
.fake-content {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
}

</style>
