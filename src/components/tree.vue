<template>
  <div class="tree" role="tree">
    <v-tree-node
      v-for="child in root"
      :node="child"
      :key="child.id"
      :indent="indent"
      :operation="operation"
      v-if="child.visible">
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

  const generate = tailCallOptimize(({nodes, data, props, level, pid, map, defaultExpand}) => {
    data.forEach(item => {
      let node = {
        childNodes: [],
        label: item[props.label || 'label'],
        id: item[props.id || 'id'],
        level: level,
        isLeaf: false,
        checked: false,
        indeterminate: false,
        visible: true,
        expanded: defaultExpand || false,
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

  const filter = tailCallOptimize((nodes, keyword, map, pVisible) => {
    nodes.forEach(node => {
      if(!keyword) {
        node.visible = true;
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
      if(!node.isLeaf) {
        filter(node.childNodes, keyword, map, node.visible);
      }
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
        checkedList: [],
        checkedMap: {}
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
            id: 'id'
          };
        }
      },
      indent: {
        type: Number,
        default: 18
      },
      'default-expand': {
        type: Boolean
      }
    },

    computed: {

    },

    methods: {
      filter(keyword) {
        filter(this.root, keyword, this.map);
      },
      setCheckedByKeys(keys) {
        let addList = [];
        let keysMap = {};
        keys.forEach(key => {
          if(this.checkedMap[key] === undefined) {
            addList.push(key);
          }
          keysMap[key] = true;
        });
        let removeList = [];
        this.checkedList.forEach(key => {
          if(!keysMap[key]) {
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
          let start = new Date();
          this.addList = [];
          this.removeList = [];
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
            if(value && !this.checkedMap[node.id]) {
              this.addList.push(node.id);
            } else if(!value && this.checkedMap[node.id]) {
              this.removeList.push(node.id);
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
        pid: null,
        map: this.map,
        defaultExpand: this.defaultExpand
      });
    }
  };
</script>
