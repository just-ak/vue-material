<template>
  <md-menu
    class="md-select"
    :class="{ 'md-disabled': disabled }"
    :md-close-on-select="false"
    :md-active.sync="showSelect"
    :md-offset-x="offset.x"
    :md-offset-y="offset.y"
    @md-opened="onOpen"
    @md-closed="onClose">
    <input
      class="md-input md-select-value"
      v-model="MdSelect.label"
      readonly
      v-on="$listeners"
      @focus.prevent="onFocus"
      @click="openSelect"
      @keydown.down="openSelect"
      @keydown.enter="openSelect"
      @keydown.space="openSelect"  />
    <md-drop-down-icon ref="icon" @blur.native="removeHighlight" @click.native="openSelect" />

    <keep-alive>
      <md-menu-content ref="menu" class="md-select-menu" :md-list-class="mdDense && 'md-dense'" :style="menuStyles" :id="uniqueId">
        <slot />
      </md-menu-content>
    </keep-alive>

    <md-input class="md-input-fake" v-model="content" :disabled="disabled" readonly />
    <select readonly v-model="content" v-bind="attributes" :required="required"></select>
  </md-menu>
</template>

<script>
  import raf from 'raf'
  import MdUuid from 'core/utils/MdUuid'
  import MdComponent from 'core/MdComponent'
  import MdDropDownIcon from 'core/icons/MdDropDownIcon'
  import MdMenu from 'components/MdMenu/MdMenu'
  import MdMenuContent from 'components/MdMenu/MdMenuContent'
  import MdInput from '../MdInput/MdInput'
  import MdFieldMixin from '../MdFieldMixin'

  const defaultOffset = {
    x: -15,
    y: -48
  }

  export default {
    name: 'MdSelect',
    components: {
      MdInput,
      MdMenu,
      MdMenuContent,
      MdDropDownIcon
    },
    mixins: [MdFieldMixin],
    props: {
      mdDense: Boolean,
      multiple: Boolean,
      id: String,
      name: String
    },
    inject: ['MdField'],
    data: () => ({
      uniqueId: 'md-select-menu-' + MdUuid(),
      menuStyles: {},
      offset: {
        x: defaultOffset.x,
        y: 0
      },
      showSelect: true,
      didMount: false,
      MdSelect: {
        items: {},
        label: null,
        multiple: false,
        modelValue: null
      }
    }),
    provide () {
      const MdSelect = this.MdSelect

      MdSelect.setValue = this.setValue
      MdSelect.setContent = this.setContent
      MdSelect.setMultipleValue = this.setMultipleValue
      MdSelect.setMultipleContent = this.setMultipleContent
      MdSelect.modelValue = this.content

      return { MdSelect }
    },
    watch: {
      value: {
        immediate: true,
        handler () {
          this.setFieldContent()
        }
      },
      multiple: {
        immediate: true,
        handler (isMultiple) {
          this.MdSelect.multiple = isMultiple
        }
      }
    },
    methods: {
      elHasScroll (el) {
        return el.scrollHeight > el.offsetHeight
      },
      scrollToSelectedOption (el, menu) {
        const top = el.offsetTop
        const elHeight = el.offsetHeight
        const menuHeight = menu.offsetHeight

        menu.scrollTop = top - ((menuHeight - elHeight) / 2)
      },
      async setOffsets (target) {
        await this.$nextTick()

        const menu = document.getElementById(this.uniqueId)

        if (menu) {
          const selected = target || menu.querySelector('.md-selected')

          if (selected) {
            this.scrollToSelectedOption(selected, menu)
            this.offset.y = defaultOffset.y - selected.offsetTop + menu.scrollTop + 8
            this.menuStyles = {
              'transform-origin': `0 ${Math.abs(this.offset.y)}px`
            }
          } else {
            this.offset.y = defaultOffset.y + 1
            this.menuStyles = {}
          }
        }
      },
      onOpen () {
        if (this.didMount) {
          this.setOffsets()
          window.setTimeout(() => {
            this.MdField.focused = true
          }, 10)
        }
      },
      applyHighlight () {
        this.MdField.focused = false
        this.MdField.highlighted = true
        this.$refs.icon.$el.focus()
      },
      onClose () {
        if (this.didMount) {
          this.$refs.icon.$el.setAttribute('tabindex', 1)
          this.applyHighlight()
        }
      },
      onFocus () {
        this.applyHighlight()
      },
      removeHighlight () {
        this.MdField.highlighted = false
        this.$refs.icon.$el.removeAttribute('tabindex')
      },
      openSelect () {
        if (!this.disabled) {
          this.showSelect = true
        }
      },
      toggleArrayValue (array, value) {
        if (array.includes(value)) {
          const index = array.indexOf(value)

          array.splice(index, 1)
        } else {
          array.push(value)
        }
      },
      setValue (newValue) {
        this.content = newValue
        this.setFieldValue()
        this.showSelect = false
      },
      setContent (newLabel) {
        this.MdSelect.label = newLabel
      },
      setContentByValue () {
        const textContent = this.MdSelect.items[this.value]

        if (textContent) {
          this.setContent(textContent)
        } else {
          this.setContent('')
        }
      },
      setMultipleValue (value) {
        const newValue = value

        this.toggleArrayValue(this.content, newValue)
        this.setFieldValue()
      },
      setMultipleContentByValue () {
        let content = []

        this.value.forEach(item => {
          const textContent = this.MdSelect.items[item]

          if (textContent) {
            content.push(textContent)
          }
        })

        this.setContent(content.join(', '))
      },
      setFieldContent () {
        if (this.multiple) {
          this.setMultipleContentByValue()
        } else {
          this.setContentByValue()
        }
      }
    },
    mounted () {
      this.showSelect = false
      this.didMount = true
      this.setFieldContent()
    }
  }
</script>

<style lang="scss">
  @import "~components/MdAnimation/variables";

  .md-menu.md-select {
    display: flex;
    flex: 1;

    &:not(.md-disabled) {
      .md-input,
      .md-icon {
        cursor: pointer;
        outline: none;
      }
    }

    .md-input {
      flex: 1;
    }

    select,
    .md-input-fake {
      width: 1px;
      height: 1px;
      margin: -1px;
      padding: 0;
      overflow: hidden;
      position: absolute;
      clip: rect(0 0 0 0);
      border: 0;
    }
  }

  .md-menu-content.md-select-menu {
    width: 100%;

    &.md-menu-content-enter {
      transform: translate3d(0, -8px, 0) scaleY(.3);
    }

    .md-list {
      transition: opacity .3s $md-transition-drop-timing;
    }

    .md-dense .md-ripple.md-list-item-content {
      font-size: 14px;
    }
  }
</style>