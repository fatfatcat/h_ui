<template>
  <div :class="itemClasses">
    <div :class="headerClasses" @click="toggle">
      <Icon name="play_fill"></Icon>
      <slot></slot>
    </div>
    <collapse-transition>
      <div :class="contentClasses" v-show="isActive">
        <div :class="boxClasses"><slot name="content"></slot></div>
      </div>
    </collapse-transition>
  </div>
</template>
<script>
  import Icon from '../Icon/Icon.vue';
  import CollapseTransition from '../Notice/collapse-transition';
  const prefixCls = 'h-collapse';

  export default {
    name: 'Panel',
    components: { Icon, CollapseTransition },
    props: {
      name: {
        type: String
      }
    },
    data () {
      return {
        index: 0, // use index for default when name is null
        isActive: false
      };
    },
    computed: {
        itemClasses () {
            return [
                `${prefixCls}-item`,
                {
                    [`${prefixCls}-item-active`]: this.isActive
                }
            ];
        },
        headerClasses () {
            return `${prefixCls}-header`;
        },
        contentClasses () {
            return `${prefixCls}-content`;
        },
        boxClasses () {
            return `${prefixCls}-content-box`;
        }
    },
    methods: {
      toggle () {
        this.$parent.toggle({
          name: this.name || this.index,
          isActive: this.isActive
        });
      }
    }
  };
</script>
