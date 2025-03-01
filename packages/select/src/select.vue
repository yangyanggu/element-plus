<template>
  <div
    ref="selectWrapper"
    class="el-select"
    :class="[selectSize ? 'el-select--' + selectSize : '']"
    @click.stop="toggleMenu"
  >
    <el-popper
      ref="popper"
      v-model:visible="dropMenuVisible"
      placement="bottom-start"
      :show-arrow="true"
      :append-to-body="popperAppendToBody"
      pure
      manual-mode
      effect="light"
      trigger="click"
      :offset="6"
    >
      <template #trigger>
        <div class="select-trigger">
          <div
            v-if="multiple"
            ref="tags"
            class="el-select__tags"
            :style="{ 'max-width': inputWidth - 32 + 'px', width: '100%' }"
          >
            <span v-if="collapseTags && selected.length">
              <el-tag
                :closable="!selectDisabled"
                :size="collapseTagSize"
                :hit="selected[0].hitState"
                type="info"
                disable-transitions
                @close="deleteTag($event, selected[0])"
              >
                <span class="el-select__tags-text">{{ selected[0].currentLabel }}</span>
              </el-tag>
              <el-tag
                v-if="selected.length > 1"
                :closable="false"
                :size="collapseTagSize"
                type="info"
                disable-transitions
              >
                <span class="el-select__tags-text">+ {{ selected.length - 1 }}</span>
              </el-tag>
            </span>
            <!-- <div> -->
            <transition v-if="!collapseTags" @after-leave="resetInputHeight">
              <span>
                <el-tag
                  v-for="item in selected"
                  :key="getValueKey(item)"
                  :closable="!selectDisabled"
                  :size="collapseTagSize"
                  :hit="item.hitState"
                  type="info"
                  disable-transitions
                  @close="deleteTag($event, item)"
                >
                  <span class="el-select__tags-text">{{ item.currentLabel }}</span>
                </el-tag>
              </span>
            </transition>
            <!-- </div> -->
            <input
              v-if="filterable"
              ref="input"
              v-model="query"
              type="text"
              class="el-select__input"
              :class="[selectSize ? `is-${ selectSize }` : '']"
              :disabled="selectDisabled"
              :autocomplete="autocomplete"
              :style="{ 'flex-grow': '1', width: inputLength / (inputWidth - 32) + '%', 'max-width': inputWidth - 42 + 'px' }"
              @focus="handleFocus"
              @blur="softFocus = false"
              @keyup="managePlaceholder"
              @keydown="resetInputState"
              @keydown.down.prevent="navigateOptions('next')"
              @keydown.up.prevent="navigateOptions('prev')"
              @keydown.esc.stop.prevent="visible = false"
              @keydown.enter.stop.prevent="selectOption"
              @keydown.delete="deletePrevTag"
              @keydown.tab="visible = false"
              @compositionstart="handleComposition"
              @compositionupdate="handleComposition"
              @compositionend="handleComposition"
              @input="debouncedQueryChange"
            >
          </div>
          <el-input
            :id="id"
            ref="reference"
            v-model="selectedLabel"
            type="text"
            :placeholder="currentPlaceholder"
            :name="name"
            :autocomplete="autocomplete"
            :size="selectSize"
            :disabled="selectDisabled"
            :readonly="readonly"
            :validate-event="false"
            :class="{ 'is-focus': visible }"
            :tabindex="(multiple && filterable) ? '-1' : null"
            @focus="handleFocus"
            @blur="handleBlur"
            @input="debouncedOnInputChange"
            @paste="debouncedOnInputChange"
            @keydown.down.stop.prevent="navigateOptions('next')"
            @keydown.up.stop.prevent="navigateOptions('prev')"
            @keydown.enter.stop.prevent="selectOption"
            @keydown.esc.stop.prevent="visible = false"
            @keydown.tab="visible = false"
            @mouseenter="inputHovering = true"
            @mouseleave="inputHovering = false"
          >
            <template v-if="$slots.prefix" #prefix>
              <slot name="prefix"></slot>
            </template>
            <template #suffix>
              <i v-show="!showClose" :class="['el-select__caret', 'el-input__icon', 'el-icon-' + iconClass]"></i>
              <i
                v-if="showClose"
                :class="`el-select__caret el-input__icon ${clearIcon}`"
                @click="handleClearClick"
              ></i>
            </template>
          </el-input>
        </div>
      </template>
      <template #default>
        <transition
          name="el-zoom-in-top"
          @before-enter="handleMenuEnter"
          @after-leave="doDestroy"
        >
          <el-select-menu
            v-show="visible && emptyText !== false"
            ref="popper"
            v-clickOutside="handleClose"
          >
            <el-scrollbar
              v-show="options.length > 0 && !loading"
              ref="scrollbar"
              tag="ul"
              wrap-class="el-select-dropdown__wrap"
              view-class="el-select-dropdown__list"
              :class="{ 'is-empty': !allowCreate && query && filteredOptionsCount === 0 }"
            >
              <el-option
                v-if="showNewOption"
                :value="query"
                :created="true"
              />
              <slot></slot>
            </el-scrollbar>
            <template v-if="emptyText && (!allowCreate || loading || (allowCreate && options.length === 0 ))">
              <slot v-if="$slots.empty" name="empty"></slot>
              <p v-else class="el-select-dropdown__empty">
                {{ emptyText }}
              </p>
            </template>
          </el-select-menu>
        </transition>
      </template>
    </el-popper>
  </div>
</template>

<script lang="ts">
import ElInput from '@element-plus/input/src/index.vue'
import ElOption from './option.vue'
import ElSelectMenu from './select-dropdown.vue'
import ElTag from '@element-plus/tag/src/index.vue'
import { Popper as ElPopper } from '@element-plus/popper'
import ElScrollbar from '@element-plus/scrollbar/src/index'
import ClickOutside from '@element-plus/directives/click-outside'
import { addResizeListener, removeResizeListener } from '@element-plus/utils/resize-event'
import { t } from '@element-plus/locale'
import { UPDATE_MODEL_EVENT } from '@element-plus/utils/constants'
import { useSelect, useSelectStates } from './useSelect'
import { selectKey } from './token'
import {
  toRefs,
  defineComponent,
  onMounted,
  onBeforeUnmount,
  nextTick,
  reactive,
  provide,
} from 'vue'


export default defineComponent({
  name: 'ElSelect',
  componentName: 'ElSelect',
  components: {
    ElInput,
    ElSelectMenu,
    ElOption,
    ElTag,
    ElScrollbar,
    ElPopper,
  },
  directives: { ClickOutside },
  props: {
    name: String,
    id: String,
    modelValue: {
      type: [Array, String, Number],
    },
    autocomplete: {
      type: String,
      default: 'off',
    },
    automaticDropdown: Boolean,
    size: String,
    disabled: Boolean,
    clearable: Boolean,
    filterable: Boolean,
    allowCreate: Boolean,
    loading: Boolean,
    popperClass: String,
    remote: Boolean,
    loadingText: String,
    noMatchText: String,
    noDataText: String,
    remoteMethod: Function,
    filterMethod: Function,
    multiple: Boolean,
    multipleLimit: {
      type: Number,
      default: 0,
    },
    placeholder: {
      type: String,
      default: t('el.select.placeholder'),
    },
    defaultFirstOption: Boolean,
    reserveKeyword: Boolean,
    valueKey: {
      type: String,
      default: 'value',
    },
    collapseTags: Boolean,
    popperAppendToBody: {
      type: Boolean,
      default: true,
    },
    clearIcon: {
      type: String,
      default: 'el-icon-circle-close',
    },
  },
  emits: ['remove-tag', 'clear', 'change', 'visible-change', 'focus', 'blur', UPDATE_MODEL_EVENT],

  setup(props, ctx) {
    const states = useSelectStates(props)
    const {
      selectSize,
      readonly,
      handleResize,
      collapseTagSize,
      debouncedOnInputChange,
      debouncedQueryChange,
      deletePrevTag,
      deleteTag,
      deleteSelected,
      handleOptionSelect,
      scrollToOption,
      setSelected,
      resetInputHeight,
      managePlaceholder,
      showClose,
      selectDisabled,
      iconClass,
      showNewOption,
      emptyText,
      toggleLastOptionHitState,
      resetInputState,
      handleComposition,
      onOptionDestroy,
      handleMenuEnter,
      handleFocus,
      blur,
      handleBlur,
      handleClearClick,
      doDestroy,
      handleClose,
      toggleMenu,
      selectOption,
      getValueKey,
      navigateOptions,
      dropMenuVisible,

      reference,
      input,
      popper,
      tags,
      selectWrapper,
      scrollbar,
    } = useSelect(props, states, ctx)

    const {
      inputWidth,
      selected,
      inputLength,
      filteredOptionsCount,
      visible,
      softFocus,
      selectedLabel,
      hoverIndex,
      query,
      inputHovering,
      currentPlaceholder,
      menuVisibleOnFocus,
      isOnComposition,
      isSilentBlur,
      options,
      cachedOptions,
      optionsCount,
    } = toRefs(states)

    provide(selectKey, reactive({
      options,
      cachedOptions,
      optionsCount,
      filteredOptionsCount,
      hoverIndex,
      handleOptionSelect,
      selectEmitter: states.selectEmitter,
      onOptionDestroy,
      props,
      inputWidth,
      selectWrapper,
      popper,
    }))

    onMounted(() => {
      states.cachedPlaceHolder = currentPlaceholder.value = props.placeholder
      if (props.multiple && Array.isArray(props.modelValue) && props.modelValue.length > 0) {
        currentPlaceholder.value = ''
      }
      addResizeListener(selectWrapper.value, handleResize)
      if (reference.value && reference.value.$el) {
        const sizeMap = {
          medium: 36,
          small: 32,
          mini: 28,
        }
        const input = reference.value.$el
        states.initialInputHeight = input.getBoundingClientRect().height || sizeMap[selectSize.value]
      }
      if (props.remote && props.multiple) {
        resetInputHeight()
      }
      nextTick(() => {
        if (reference.value.$el) {
          inputWidth.value = reference.value.$el.getBoundingClientRect().width
        }
      })
      setSelected()
    })

    onBeforeUnmount(() => {
      if (selectWrapper.value && handleResize) removeResizeListener(selectWrapper.value, handleResize)
    })

    if (props.multiple && !Array.isArray(props.modelValue)) {
      ctx.emit(UPDATE_MODEL_EVENT, [])
    }
    if (!props.multiple && Array.isArray(props.modelValue)) {
      ctx.emit(UPDATE_MODEL_EVENT, '')
    }
    return {
      selectSize,
      readonly,
      handleResize,
      collapseTagSize,
      debouncedOnInputChange,
      debouncedQueryChange,
      deletePrevTag,
      deleteTag,
      deleteSelected,
      handleOptionSelect,
      scrollToOption,
      inputWidth,
      selected,
      inputLength,
      filteredOptionsCount,
      visible,
      softFocus,
      selectedLabel,
      hoverIndex,
      query,
      inputHovering,
      currentPlaceholder,
      menuVisibleOnFocus,
      isOnComposition,
      isSilentBlur,
      options,
      resetInputHeight,
      managePlaceholder,
      showClose,
      selectDisabled,
      iconClass,
      showNewOption,
      emptyText,
      toggleLastOptionHitState,
      resetInputState,
      handleComposition,
      handleMenuEnter,
      handleFocus,
      blur,
      handleBlur,
      handleClearClick,
      doDestroy,
      handleClose,
      toggleMenu,
      selectOption,
      getValueKey,
      navigateOptions,
      dropMenuVisible,

      reference,
      input,
      popper,
      tags,
      selectWrapper,
      scrollbar,
    }
  },
})
</script>
<style lang="scss" scoped>
.el-popper {
  padding: 0;
}
</style>
