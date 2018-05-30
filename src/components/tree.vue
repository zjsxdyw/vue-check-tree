<template>
  <div
    class="el-tree"
    role="tree"
  >
    <v-tree-node
      v-for="child in root"
      :node="child"
      :key="child.id"
      :indent="indent"
      :operation="operation">
        <template slot-scope="{node}">
          <slot v-if="$scopedSlots && $scopedSlots.default" :node="node"></slot>
          <span v-else class="tree-node-label" v-text="node.label"></span>
        </template>
    </v-tree-node>
    <div class="empty-block" v-if="!root || root.length === 0">
      <span class="empty-text">{{ emptyText }}</span>
    </div>
  </div>
</template>
<script>
  import VTreeNode from './tree-node.vue';

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

  const generate = tailCallOptimize(({nodes, data, props, level, pid, map}) => {
    data.forEach(item => {
      let node = {
        childNodes: [],
        label: item[props.label],
        id: item[props.id],
        level: level,
        isLeaf: false,
        checked: false,
        indeterminate: false,
        pid: pid,
        data: item
      };
      if(item[props.children] && item[props.children].length) {
        generate({
          nodes: node.childNodes, 
          data: item[props.children],
          props,
          level: level + 1,
          pid: node.id,
          map
        });
      } else {
        node.isLeaf = true;
      }
      nodes.push(node);
      map[node.id] = node;
    });
  });

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
        checkedList: []
      }
    },

    props: {
      emptyText: {
        type: String,
        default() {
          return '暂无数据';
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
            disabled: 'disabled',
            id: 'id'
          };
        }
      },
      indent: {
        type: Number,
        default: 18
      },
      'filter-node-method': {
        type: Function
      },
      'node-key': {
        type: String
      }
    },

    computed: {

    },

    methods: {
      filter() {

      },
      setCheckedByKeys(keys) {
        let addList = [];
        keys.forEach(key => {
          if(this.checkedList.indexOf(key) === -1) {
            addList.push(key);
          }
        });
        let removeList = [];
        this.checkedList.forEach(key => {
          if(keys.indexOf(key) === -1) {
            removeList.push(key);
          }
        });
        addList.forEach(key => {
          this.setCheckedByKey(key, true);
        });
        removeList.forEach(key => {
          this.setCheckedByKey(key, false);
        });
      },
      checkNode(key, value) {
        if(this.map[key]) {
          this.setCheckedByKey(key, value);
          this.$emit('check-change', this.map[key].data);
        }
      },
      setCheckedByKey(key, value) {
        if(this.map[key]) {
          console.time('setChecked')
          let node = this.map[key];
          if(!node.isLeaf) {
            node.indeterminate = false;
            this.setAllChildChecked(key, value);
          }
          this.setNodeChecked(node, value);
          let pNode = this.map[node.pid];
          while(pNode) {
            let allChecked = true;
            let hasChecked = false;
            pNode.childNodes.forEach(node => {
              if(!node.checked) {
                allChecked = false;
              }
              if(node.checked || node.indeterminate) {
                hasChecked = true;
              }
            });
            this.setNodeChecked(pNode, allChecked);
            pNode.indeterminate = !allChecked && hasChecked;
            pNode = this.map[pNode.pid];
          }
          console.timeEnd('setChecked')
        }
      },
      setAllChildChecked(key, value) {
        if(this.map[key]) {
          let pNode = this.map[key];
          pNode.childNodes && pNode.childNodes.forEach(node => {
            this.setNodeChecked(node, value);
            node.indeterminate = false;
            this.setAllChildChecked(node.id, value);
          });
        }
      },
      getCheckedList() {
        let list = [];
        this.checkedList.forEach(id => {
          list.push(this.map[id].data);
        });
        return list;
      },
      setNodeChecked(node, value) {
        if(node.checked !== value) {
          node.checked = value;
          if(node.isLeaf) {
            let index = this.checkedList.indexOf(node.id);
            if(value && index === -1) {
              this.checkedList.push(node.id);
            } else if(!value && index > -1) {
              this.checkedList.splice(index, 1);
            }
          }
        }
      }
    },

    created() {
      generate({
        nodes: this.root,
        data: this.data,
        props: this.props,
        level: 1,
        pid: '',
        map: this.map
      });
    }
  };
</script>
