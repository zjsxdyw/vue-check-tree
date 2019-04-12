<template>
  <div class="check_tree__tree" role="tree">
    <v-tree-node
      v-for="child in root"
      :node="child"
      :key="child.id"
      :indent="indent"
      :radio="radio"
      :operation="operation"
      :disabled-list="disabledList"
      v-if="child.visible">
        <template slot-scope="{node}">
          <slot v-if="$scopedSlots && $scopedSlots.default" :node="node"></slot>
          <span v-else class="check_tree__tree-node-label" v-text="node.label"></span>
        </template>
    </v-tree-node>
    <div class="check_tree__empty-block" v-show="showEmpty">
      <span class="check_tree__empty-text" v-text="emptyText"></span>
    </div>
  </div>
</template>
<script>
  import '../assets/tree.css'
  import VTreeNode from './tree-node.vue'
  import './pinyin.js'

  // 尾递归函数
  function tailCallOptimize(f) { 
    let value;
    let active = false;
    const accumulated = [];
    return function accumulator() {
      accumulated.push(arguments);
      if (!active) {
        active = true;
        while (accumulated.length) {
          value = f.apply(this, accumulated.shift())
        }
        active = false
        return value
      }
    }
  }

  // 生成树状结构
  const generate = tailCallOptimize(({nodes, data, props, level, pid, map, defaultExpand}) => {
    data.forEach(item => {
      let node = {
        childNodes: [],
        label: item[props.label || 'label'] || '',
        id: item[props.id || 'id'],
        level: level,
        isLeaf: false,
        checked: false,
        indeterminate: false,
        visible: true,
        expanded: defaultExpand,
        pid: pid,
        data: item
      };
      let children = props.children || 'children';
      if(item[children] && item[children].length) {
        generate({
          nodes: node.childNodes, 
          data: item[children],
          props,
          level: level + 1,
          pid: node.id,
          map,
          defaultExpand
        });
      } else {
        node.isLeaf = true;
      }
      nodes.push(node);
      map[node.id] = node;
    });
  });

  // 根据关键字过滤，并返回找到显示节点的个数
  const filter = (nodes, keyword, defaultExpand, map, pVisible) => {
    let visibleCount = 0;
    nodes.forEach(node => {
      if(!keyword) {
        node.visible = true;
        node.expanded = defaultExpand;
      } else if(node.label.indexOf(keyword) > -1) {
        node.visible = true;
        let pNode = map[node.pid];
        while(pNode) {
          pNode.visible = true;
          pNode.expanded = true;
          pNode = map[pNode.pid];
        }
      } else {
        node.visible = !!pVisible;
      }
      if(node.visible) visibleCount++;

      if(!node.isLeaf) {
        visibleCount += filter(node.childNodes, keyword, defaultExpand, map, node.visible);
      }
    });

    return visibleCount;
  };

  // 计算状态的递归函数
  const dfs = (node) => {
    if(!node) return;
    if(node.isLeaf) return node;
    let allChecked = true;
    let hasChecked = false;
    node.childNodes && node.childNodes.forEach(child => {
      child = dfs(child);
      if(!(child && child.visible)) return;
      if(!child.checked) {
        allChecked = false;
      }
      if(child.checked || child.indeterminate) {
        hasChecked = true;
      }
    });
    node.checked = allChecked;
    node.indeterminate = !allChecked && hasChecked;
    return node;
  };

  export default {
    name: 'VTree',

    components: {
      VTreeNode
    },

    data() {
      return {
        root: [],
        map: {},
        operation: {
          checkNode: this.checkNode
        },
        checkedList: [],
        checkedMap: {},
        // 默认显示数
        visibleCount: 1
      }
    },

    props: {
      'empty-text': {
        type: String,
        default() {
          return 'empty';
        }
      },
      data: {
        type: Array
      },
      props: {
        default() {
          return {
            children: 'children',
            label: 'label',
            id: 'id'
          };
        }
      },
      indent: {
        type: Number,
        default: 18
      },
      'default-expand': {
        type: Boolean,
        default: false
      },
      radio: {
        type: Boolean,
        default: false
      },
      'disabled-list': {
        type: Array,
        default: []
      }
    },

    computed: {
      showEmpty() {
        return !this.root || this.root.length === 0 || this.visibleCount === 0;
      }
    },

    methods: {
      // 初始化
      init() {
        this.root = [];
        generate({
          nodes: this.root,
          data: this.data,
          props: this.props,
          level: 1,
          pid: null,
          map: this.map,
          defaultExpand: this.defaultExpand
        });
      },
      // 筛选
      filter(keyword) {
        this.visibleCount = filter(this.root, keyword, this.defaultExpand, this.map);
        this.recalculate();
      },
      // 通过key值勾选多节点
      setCheckedByKeys(keys) {
        if(this.radio && keys.length > 1) {
          console.error('Radio cannot set more than one node');
          return;
        }
        this.checkedMap = {};
        this.checkedList = [];
        for(let key in this.map) {
          this.map[key].checked = false;
          this.map[key].indeterminate = false;
        }
        keys.forEach(key => {
          this.setCheckedByKey(key, true);
        });
      },
      // 勾选节点并触发事件
      checkNode(key, value) {
        if(this.map[key]) {
          if(this.radio) {
            let seleced = this.setRadioByKey(key);
            this.$emit('check-change', this.map[seleced].data, value);
            return
          }
          let {addList, removeList} = this.setCheckedByKey(key, value);
          addList = addList.map(id => this.map[id].data);
          removeList = removeList.map(id => this.map[id].data);
          this.$emit('check-change', this.map[key].data, value, addList, removeList);
        }
      },
      // 单选节点
      setRadioByKey(key) {
        if(this.map[key]) {
          let node = this.map[key];
          if(!node.isLeaf) {
            return this.setRadioByKey(node.childNodes[0].id);
          }
          if(this.checkedList.length) {
            let rNode = this.map[this.checkedList[0]];
            let rpNode = this.map[rNode.pid];
            while(rpNode) {
              rpNode.indeterminate = false;
              rpNode = this.map[rpNode.pid];
            }
            rNode.checked = false;
          }
          this.checkedList = [node.id];
          this.checkedMap = {[node.id]: true};
          let pNode = this.map[node.pid];
          while(pNode) {
            pNode.indeterminate = true;
            pNode = this.map[pNode.pid];
          }
          node.checked = true;
          return key;
        }
      },
      // 通过key值勾选单节点
      setCheckedByKey(key, value, isClick) {
        if(this.map[key]) {
          let node = this.map[key];
          if(isClick && this.disabledList.indexOf(node.id) > -1) return;
          this.addList = [];
          this.removeList = [];
          if(!node.isLeaf) {
            node.indeterminate = this.setAllChildChecked(key, value, isClick);
          }
          this.setNodeChecked(node, value, isClick);
          let pNode = this.map[node.pid];
          while(pNode) {
            let allChecked = true;
            let hasChecked = false;
            pNode.childNodes.forEach(node => {
              if(!node.visible) return;
              if(!node.checked) {
                allChecked = false;
              }
              if(node.checked || node.indeterminate) {
                hasChecked = true;
              }
            });
            this.setNodeChecked(pNode, allChecked, isClick);
            pNode.indeterminate = !allChecked && hasChecked;
            pNode = this.map[pNode.pid];
          }


          let removeMap = {};
          this.removeList.forEach(id => {
            removeMap[id] = true;
          });
          let indexList = [];
          this.checkedList.forEach((id, index) => {
            if(removeMap[id]) {
              indexList.unshift(index);
              delete this.checkedMap[id];
            }
          });

          indexList.forEach(index => {
            this.checkedList.splice(index, 1);
          });

          this.addList.forEach(id => {
            this.checkedList.push(id);
            this.checkedMap[id] = true;
          });
          let result = {
            addList: this.addList,
            removeList: this.removeList
          };
          delete this.addList;
          delete this.removeList;
          return result;
        }
      },
      // 勾选所有子节点
      setAllChildChecked(key, value, isClick) {
        if(this.map[key]) {
          let pNode = this.map[key];
          let indeterminate = false;
          pNode.childNodes && pNode.childNodes.forEach(node => {
            if(!node.visible) return;
            let result = this.setNodeChecked(node, value, isClick);
            if(result !== undefined && result !== value) {
              indeterminate = true;
            }
            node.indeterminate = this.setAllChildChecked(node.id, value, isClick);
            if(node.indeterminate) {
              indeterminate = true;
            }
          });
          return indeterminate;
        }
      },
      // 获取勾选列表
      getCheckedList() {
        let list = [];
        this.checkedList.forEach(id => {
          list.push(this.map[id].data);
        });
        return list;
      },
      // 设置节点勾选状态
      setNodeChecked(node, value, isClick) {
        if(isClick && this.disabledList.indexOf(node.id) > -1) return node.checked;
        if(node.checked !== value) {
          node.checked = value;
          if(node.isLeaf) {
            if(value && !this.checkedMap[node.id]) {
              this.addList.push(node.id);
            } else if(!value && this.checkedMap[node.id]) {
              this.removeList.push(node.id);
            }
          }
        }
      },
      // 重新修改所有非叶子节点勾选状态
      recalculate() {
        for(let node of this.root) {
          dfs(node);
        }
      },
    },
    watch: {
      data() {
        this.init();
      }
    },

    created() {
      this.init();
    }
  };
</script>