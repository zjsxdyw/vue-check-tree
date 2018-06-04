<template>
  <div class="tree-node">
    <div class="tree-node-content" @click.stop="handleClick">
      <span class="icon-right" 
        :class="{ 
          'is-leaf': node.isLeaf,
          expanded: !node.isLeaf && node.expanded
        }"></span>
      <span class="checkbox"
        :class="{
          'is-checked': node.checked,
          'is-indeterminate': node.indeterminate,
          'is-disabled': node.disabled
        }"
        @click.stop="handleCheckChange"></span>
      <slot :node="node"></slot>
    </div>
    <div class="tree-node-children pl18" v-if="node.expanded">
      <v-tree-node
        v-for="child in node.childNodes"
        :node="child"
        :key="child.id"
        :indent="indent"
        :operation="operation"
        v-if="child.visible">
        <template slot-scope="{node}">
          <slot :node="node"></slot>
        </template>
      </v-tree-node>
    </div>
  </div>
</template>
<script>

  export default {
    name: 'VTreeNode',

    componentName: 'ElTreeNode',

    props: {
      node: {
        default() {
          return {};
        }
      },
      indent: {
        type: Number
      },
      operation: {
        type: Object
      }
    },

    data() {
      return {
        tree: null,
        childNodeRendered: false,
        showCheckbox: true,
        oldChecked: null,
        oldIndeterminate: null
      };
    },

    methods: {
      handleCheckChange() {
        this.operation.checkNode(this.node.id, !this.node.checked);
      },
      handleClick() {
        if(this.node.isLeaf) {
          this.handleCheckChange();
        } else {
          this.node.expanded = !this.node.expanded;
        }
      }
    },

  };
</script>
