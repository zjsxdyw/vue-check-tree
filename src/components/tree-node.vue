<template>
  <div class="check_tree__tree-node">
    <div class="check_tree__tree-node-content" @click.stop="handleClick">
      <span class="check_tree__icon-right" 
        :class="{ 
          'is-leaf': node.isLeaf,
          'expanded': !node.isLeaf && node.expanded
        }"></span>
      <span class="check_tree__checkbox"
        :class="{
          'is-checked': node.checked,
          'is-indeterminate': node.indeterminate,
          'is-disabled': disabled
        }"
        @click.stop="handleCheckChange"></span>
      <slot :node="node"></slot>
    </div>
    <div class="check_tree__tree-node-children check_tree__pl18" v-if="node.expanded">
      <v-tree-node
        v-for="child in node.childNodes"
        :node="child"
        :key="child.id"
        :indent="indent"
        :radio="radio"
        :operation="operation"
        :disabled-list="disabledList"
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

    componentName: 'VTreeNode',

    props: {
      node: {
        default() {
          return {};
        }
      },
      indent: {
        type: Number
      },
      radio: {
        type: Boolean
      },
      operation: {
        type: Object
      },
      'disabled-list': {
        type: Array
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

    computed: {
      disabled() {
        return (this.node.checked && this.radio) || this.disabledList.indexOf(this.node.id) > -1;
      }
    },

    methods: {
      handleCheckChange() {
        if(this.disabled) return;
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