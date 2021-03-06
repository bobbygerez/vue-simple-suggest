<template>
  <div class="vue-simple-suggest" :class="{ designed: !destyled, focus: isInFocus }"
    @keydown.tab="isTabbed = true"
  >
    <div class="input-wrapper" ref="inputSlot">
      <slot>
        <input class="default-input" v-bind="$attrs" :value="text || ''">
      </slot>
    </div>
    <div class="suggestions" v-if="!!listShown && !removeList"
      @mouseenter="hoverList(true)"
      @mouseleave="hoverList(false)"
    >
      <slot name="misc-item-above"
        :suggestions="suggestions"
        :query="text"
      ></slot>

      <div class="suggest-item" v-for="(suggestion, index) in suggestions"
        @mouseenter="hover(suggestion, $event.target)"
        @mouseleave="hover(null, $event.target)"
        @click="suggestionClick(suggestion, $event)"
        :key="isPlainSuggestion ? 'suggestion-' + index : valueProperty(suggestion)"
        :class="{
          selected: selected && (valueProperty(suggestion) == valueProperty(selected)),
          hover: hovered && (valueProperty(hovered) == valueProperty(suggestion))
        }">
        <slot name="suggestion-item"
          :autocomplete="() => autocompleteText(displayProperty(suggestion))"
          :suggestion="suggestion"
          :query="text">
          <span>{{ displayProperty(suggestion) }}</span>
        </slot>
      </div>

      <slot name="misc-item-below"
        :suggestions="suggestions"
        :query="text"
      ></slot>
    </div>
  </div>
</template>

<script>
import {
  defaultControls,
  modes,
  fromPath,
  hasKeyCode
} from './misc'

let event = 'input'

export default {
  name: 'vue-simple-suggest',
  model: {
    prop: 'value',
    get event() { return event }
  },
  props: {
    controls: {
      type: Object,
      default: () => defaultControls
    },
    minLength: {
      type: Number,
      default: 1
    },
    maxSuggestions: {
      type: Number,
      default: 10
    },
    displayAttribute: {
      type: String,
      default: 'title'
    },
    valueAttribute: {
      type: String,
      default: 'id'
    },
    list: {
      type: [Function, Array],
      default: () => []
    },
    removeList: {
      type: Boolean,
      default: false
    },
    destyled: {
      type: Boolean,
      default: false
    },
    filterByQuery: {
      type: Boolean,
      default: false
    },
    filter: {
      type: Function,
      default(el, value) {
        return value ? ~this.displayProperty(el).toLowerCase().indexOf(value.toLowerCase()) : true
      }
    },
    debounce: {
      type: Number,
      default: 0
    },
    value: {},
    mode: {
      type: String,
      default: event,
      validator: value => !!~Object.keys(modes).indexOf(value.toLowerCase())
    }
  },
  // Handle run-time mode changes (not working):
  watch: {
    mode: {
      handler(current, old) {
        event = current
      },
      immediate: true
    },
    value: {
      handler(current) {
        this.text = current
      },
      immediate: true
    }
  },
  //
  data () {
    return {
      selected: null,
      hovered: null,
      suggestions: [],
      listShown: false,
      inputElement: null,
      canSend: true,
      timeoutInstance: null,
      text: this.value,
      isPlainSuggestion: false,
      isClicking: false,
      isOverList: false,
      isInFocus: false,
      isTabbed: false,
      controlScheme: {}
    }
  },
  computed: {
    listIsRequest () {
      return typeof this.list === 'function'
    },
    inputIsComponent () {
      return (this.$slots.default && this.$slots.default.length > 0) && !!this.$slots.default[0].componentInstance
    },
    input () {
      return this.inputIsComponent ? this.$slots.default[0].componentInstance : this.inputElement
    },
    on () {
      return this.inputIsComponent ? '$on' : 'addEventListener'
    },
    off () {
      return this.inputIsComponent ? '$off' : 'removeEventListener'
    },
    hoveredIndex () {
      return this.suggestions.findIndex(el => this.hovered && (this.valueProperty(this.hovered) == this.valueProperty(el)))
    },
    textLength () {
      return (this.text && this.text.length) || (this.inputElement.value.length) || 0
    }
  },
  created () {
    this.controlScheme = Object.assign({}, defaultControls, this.controls)
  },
  mounted () {
    this.inputElement = this.$refs['inputSlot'].querySelector('input')

    this.prepareEventHandlers(true)
  },
  beforeDestroy () {
    this.prepareEventHandlers(false)
  },
  methods: {
    prepareEventHandlers(enable) {
      const binder = this[enable ? 'on' : 'off']
      const keyEventsList = {
        click: this.showSuggestions,
        keydown: $event => (this.moveSelection($event), this.onAutocomplete($event)),
        keyup: this.onListKeyUp
      }
      const eventsList = Object.assign({
        blur: this.onBlur,
        focus: this.onFocus,
        input: this.onInput,
      }, keyEventsList)

      for (const event in eventsList) {
        this.input[binder](event, eventsList[event])
      }

      const listenerBinder = enable ? 'addEventListener' : 'removeEventListener'

      for (const event in keyEventsList) {
        this.inputElement[listenerBinder](event, keyEventsList[event])
      }
    },
    isScopedSlotEmpty (slot) {
      if (slot) {
        const vNode = slot(this)
        return !(Array.isArray(vNode) || (vNode && (vNode.tag || vNode.context || vNode.text || vNode.children)))
      }

      return true
    },
    miscSlotsAreEmpty () {
      const slots = ['misc-item-above', 'misc-item-below'].map(s => this.$scopedSlots[s])

      if (slots.every(s => !!s)) {
        return slots.every(this.isScopedSlotEmpty.bind(this))
      }

      const slot = slots.find(s => !!s)

      return this.isScopedSlotEmpty.call(this, slot)
    },
    getPropertyByAttribute (obj, attr) {
      return this.isPlainSuggestion ? obj : typeof obj !== undefined ? fromPath(obj, attr) : obj
    },
    displayProperty (obj) {
      return String(this.getPropertyByAttribute(obj, this.displayAttribute))
    },
    valueProperty (obj) {
      return this.getPropertyByAttribute(obj, this.valueAttribute)
    },
    autocompleteText (text) {
      this.$emit('input', text)
      this.inputElement.value = text
      this.text = text
    },
    select (item) {
      this.hovered = null
      this.selected = item

      this.$emit('select', item)

      this.autocompleteText(this.displayProperty(item))
    },
    hover (item, elem) {
      this.hovered = item

      if (this.hovered != null) {
        this.$emit('hover', item, elem)
      }
    },
    hoverList (isOverList) {
      this.isOverList = isOverList
    },
    hideList () {
      if (this.listShown) {
        this.listShown = false
        this.$emit('hide-list')
      }
    },
    showList () {
      if (!this.listShown) {
        if (this.textLength >= this.minLength
          && ((this.suggestions.length > 0) || !this.miscSlotsAreEmpty())
        ) {
          this.listShown = true
          this.$emit('show-list')
        }
      }
    },
    async showSuggestions () {
      if (this.suggestions.length === 0 && this.minLength === this.textLength) {
        await this.research()
      }

      this.showList()
    },
    moveSelection (e) {
      if (hasKeyCode([this.controlScheme.selectionUp, this.controlScheme.selectionDown], e)) {
        e.preventDefault()
        this.showSuggestions()

        const isMovingDown = hasKeyCode(this.controlScheme.selectionDown, e)
        const direction = isMovingDown * 2 - 1
        const listEdge = isMovingDown ? 0 : this.suggestions.length - 1
        const hoversBetweenEdges = isMovingDown ? this.hoveredIndex < this.suggestions.length - 1 : this.hoveredIndex > 0

        let item = null

        if (!this.hovered) {
          item = this.selected || this.suggestions[listEdge]
        } else if (hoversBetweenEdges) {
          item = this.suggestions[this.hoveredIndex + direction]
        } else /* if hovers on edge */ {
          item = this.suggestions[listEdge]
        }

        this.hover(item)
      }
    },
    onListKeyUp (e) {
      const select = this.controlScheme.select,
          hideList = this.controlScheme.hideList

      if (hasKeyCode([select, hideList], e)) {
        e.preventDefault()
        if (this.listShown) {
          if (hasKeyCode(select, e) && this.hovered) {
            this.select(this.hovered)
          }

          this.hideList()
        } else if (hasKeyCode(select, e)) {
          this.research()
        }
      }
    },
    onAutocomplete (e) {
      if (hasKeyCode(this.controlScheme.autocomplete, e)
        && (e.ctrlKey || e.shiftKey)
        && (this.suggestions.length > 0 && this.suggestions[0])
        && (this.listShown)
      ) {
        e.preventDefault()
        this.hover(this.suggestions[0])
        this.autocompleteText(this.displayProperty(this.suggestions[0]))
      }
    },
    suggestionClick (suggestion, e) {
      this.$emit('suggestion-click', suggestion, e)
      this.select(suggestion)
      this.hideList()

      /// Ensure, that all needed flags are off before finishing the click.
      this.isClicking = this.isOverList = false
    },
    onBlur (e) {
      if (this.isInFocus) {

        /// Clicking starts here, because input's blur occurs before the suggestionClick
        /// and exactly when the user clicks the mouse button or taps the screen.
        this.isClicking = this.isOverList && !this.isTabbed

        if (!this.isClicking) {
          this.isInFocus = false
          this.hideList()

          this.$emit('blur', e)
        } else if (e && e.isTrusted && !this.isTabbed) {
          this.inputElement.focus()
        }
      } else {
        this.inputElement.blur()
        console.error(
          `This should never happen!
          If you encouneterd this error, please report at https://github.com/KazanExpress/vue-simple-suggest/issues`
        )
      }

      this.isTabbed = false
    },
    onFocus (e) {
      this.isInFocus = true

      // Only emit, if it was a native input focus
      if (e && e.sourceCapabilities) {
        this.$emit('focus', e)
      }

      // Show list only if the item has not been clicked
      if (!this.isClicking) {
        this.showSuggestions()
      }
    },
    onInput (inputEvent) {
      const value = !inputEvent.target ? inputEvent : inputEvent.target.value

      if (this.text === value) { return }

      this.text = value
      this.$emit('input', this.text)

      if (this.selected) {
        this.selected = null
        this.$emit('select', null)
      }

      if (this.debounce) {
        clearTimeout(this.timeoutInstance)
        this.timeoutInstance = setTimeout(this.research, this.debounce)
      } else {
        this.research()
      }
    },
    async research () {
      try {
        if (this.canSend) {
          this.canSend = false
          this.$set(this, 'suggestions', await this.getSuggestions(this.text))
        }
      }

      catch (e) {
        this.clearSuggestions()
        throw e
      }

      finally {
        this.canSend = true

        if ((this.suggestions.length === 0) && this.miscSlotsAreEmpty()) {
          this.hideList()
        } else {
          this.showList()
        }

        return this.suggestions
      }
    },
    async getSuggestions (value) {
      value = value || '';

      if (value.length < this.minLength) {
        if (this.listShown) {
          this.hideList()
          return []
        }

        return this.suggestions
      }

      this.selected = null

      // Start request if can
      if (this.listIsRequest) {
        this.$emit('request-start', value)

        if ((this.suggestions.length > 0) || (!this.miscSlotsAreEmpty())) {
          this.showList()
        }
      }

      let result = []
      try {
        if (this.listIsRequest) {
          result = (await this.list(value)) || []
        } else {
          result = this.list
        }

        // IFF the result is not an array (just in case!) - make it an array
        if (!Array.isArray(result)) { result = [result] }

        this.isPlainSuggestion = (typeof result[0] !== 'object') || Array.isArray(result[0])

        if (this.filterByQuery) {
          result = result.filter((el) => this.filter(el, value))
        }

        if (this.listIsRequest) {
          this.$emit('request-done', result)
        }
      }

      catch (e) {
        if (this.listIsRequest) {
          this.$emit('request-failed', e)
        } else {
          throw e
        }
      }

      finally {
        if (this.maxSuggestions) {
          result.splice(this.maxSuggestions)
        }

        return result
      }
    },
    clearSuggestions () {
      this.suggestions.splice(0)
    }
  }
}
</script>

<style>
.vue-simple-suggest.designed {
  position: relative;
}

.vue-simple-suggest.designed, .vue-simple-suggest.designed * {
  box-sizing: border-box;
}

.vue-simple-suggest.designed .input-wrapper input {
  display: block;
  width: 100%;
  padding: 10px;
  border: 1px solid #cde;
  border-radius: 3px;
  color: black;
  background: white;
  outline:none;
  transition: all .1s;
  transition-delay: .05s
}

.vue-simple-suggest.designed.focus .input-wrapper input {
  border: 1px solid #aaa;
}

.vue-simple-suggest.designed .suggestions {
  position: absolute;
  left: 0;
  right: 0;
  top: 100%;
  top: calc(100% + 5px);
  border-radius: 3px;
  border: 1px solid #aaa;
  background-color: #fff;
  z-index: 1000;
}

.vue-simple-suggest.designed .suggestions .suggest-item {
  cursor: pointer;
  user-select: none;
}

.vue-simple-suggest.designed .suggestions .suggest-item,
.vue-simple-suggest.designed .suggestions .misc-item {
  padding: 5px 10px;
}

.vue-simple-suggest.designed .suggestions .suggest-item.hover {
  background-color: #2874D5 !important;
  color: #fff !important;
}

.vue-simple-suggest.designed .suggestions .suggest-item.selected {
  background-color: #2832D5;
  color: #fff;
}
</style>

