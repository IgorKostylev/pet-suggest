<template>
  <div class="suggest-container">
    <label v-if="label">{{ label }}</label>

    <div v-if="showLimitMessage" class="limit-message">
      Достигнуто максимальное количество выбранных элементов ({{ maxSelected }})
    </div>

    <div
        class="input-wrapper"
        :class="{
        'has-tags': selectedItems.length > 0,
        'single-selection': maxSelected === 1 && selectedItems.length === 1
      }"
    >
      <input
          ref="inputRef"
          v-model="inputValue"
          type="text"
          class="suggest-input"
          :placeholder="computedPlaceholder"
          :disabled="isInputDisabled"
          @input="handleInputDebounced"
          @keydown="handleKeyDown"
          @focus="handleFocus"
          @blur="handleBlur"
      />
      <div v-if="selectedItems.length > 0" class="tags-container">
        <div
            v-for="(item, index) in selectedItems"
            :key="item.id || item.alias"
            class="tag"
        >
          @{{ item.alias }}
          <span class="tag-remove" @click.stop="removeItem(index)">×</span>
        </div>
      </div>
    </div>

    <div v-show="showDropdown" class="dropdown">
      <div v-if="loading" class="dropdown-status">Загрузка...</div>
      <div v-else-if="error" class="dropdown-status error">{{ errorMessage }}</div>

      <div v-else-if="showInitialMessage" class="dropdown-message">
        <div class="message-title">Начните вводить имя</div>

        <div v-if="inputValue.length > 0 && remainingChars > 0" class="message-sub">
          Ещё {{ remainingChars }} {{ remainingCharsText }}
        </div>
      </div>

      <div v-else-if="suggestions.length === 0" class="dropdown-status">Ничего не найдено</div>

      <ul v-else class="suggestions-list">
        <li
            v-for="(item, index) in suggestions"
            :key="item.id || item.alias"
            :class="{ active: activeIndex === index }"
            @mousedown="selectItem(item)"
        >
          <MentionItem :item="item" />
        </li>
      </ul>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, computed, nextTick } from 'vue'
import axios from 'axios'
import type { AxiosError, CancelTokenSource } from 'axios'
import MentionItem from './MentionItem.vue'

export interface MentionItem {
  id?: number
  type: 'user' | 'company'
  name: string | null
  alias: string
  avatar: string | null
}

export default defineComponent({
  name: 'SuggestMention',
  components: { MentionItem },
  props: {
    label: String,
    apiUrl: {
      type: String,
      required: true
    },
    maxSelected: {
      type: Number,
      default: 0
    },
    placeholder: {
      type: String,
      default: 'Введите 3 или более символа'
    }
  },
  emits: ['update:selected'],
  setup(props, { emit }) {
    const inputValue = ref('')
    const suggestions = ref<MentionItem[]>([])
    const selectedItems = ref<MentionItem[]>([])
    const loading = ref(false)
    const error = ref<string | null>(null)
    const showDropdown = ref(false)
    const activeIndex = ref(-1)
    const inputRef = ref<HTMLInputElement | null>(null)
    const cancelTokenSource = ref<CancelTokenSource | null>(null)
    const inputFocused = ref(false)
    const showLimitMessage = ref(false)

    const errorMessage = computed(() => error.value || 'Ошибка при загрузке данных')

    const remainingChars = computed(() => {
      return Math.max(0, 3 - inputValue.value.length)
    })

    const remainingCharsText = computed(() => {
      const count = remainingChars.value
      const lastDigit = count % 10

      if (lastDigit === 1) return 'символ'
      if ([2, 3].includes(lastDigit)) return 'символа'
    })

    const showInitialMessage = computed(() => {
      return inputFocused.value &&
          !loading.value &&
          !error.value &&
          inputValue.value.length < 3
    })

    const computedPlaceholder = computed(() => {
      if (props.maxSelected !== 1) {
        return props.placeholder
      }
      return selectedItems.value.length === 0 ? props.placeholder : ''
    })

    const isInputDisabled = computed(() => {
      return props.maxSelected === 1 && selectedItems.value.length === 1
    })

    const fetchSuggestions = async (query: string) => {
      if (cancelTokenSource.value) {
        cancelTokenSource.value.cancel('Новый запрос')
      }
      cancelTokenSource.value = axios.CancelToken.source()

      try {
        loading.value = true
        error.value = null
        showDropdown.value = true

        const response = await axios.get<{ data: MentionItem[] }>(props.apiUrl, {
          params: { q: query },
          cancelToken: cancelTokenSource.value.token
        })

        if (response.data && Array.isArray(response.data.data)) {
          suggestions.value = response.data.data
        } else if (Array.isArray(response.data)) {
          suggestions.value = response.data
        } else {
          suggestions.value = []
          error.value = 'Некорректный формат ответа'
        }

        activeIndex.value = -1
      } catch (err) {
        if (axios.isCancel(err)) return

        const axiosError = err as AxiosError
        if (axiosError.response?.status === 400) {
          error.value = 'Некорректный запрос'
        } else if (axiosError.response?.status === 500) {
          error.value = 'Ошибка сервера'
        } else if (axiosError.message === 'Network Error') {
          error.value = 'Сетевая ошибка'
        } else {
          error.value = 'Неизвестная ошибка'
        }
        suggestions.value = []

      } finally {
        loading.value = false
        cancelTokenSource.value = null
      }
    }

    const selectItem = (item: MentionItem) => {
      if (props.maxSelected > 0 && selectedItems.value.length >= props.maxSelected) {
        showLimitMessage.value = true
        return
      }

      const itemWithId = { ...item, id: item.id || Date.now() };

      if (!selectedItems.value.some(selected => selected.alias === itemWithId.alias)) {
        selectedItems.value = [...selectedItems.value, itemWithId]
        inputValue.value = ''
        emit('update:selected', selectedItems.value)
        showLimitMessage.value = false

        if (props.maxSelected === 1 && selectedItems.value.length === 1) {
          nextTick(() => {
            const wrapper = document.querySelector('.input-wrapper.single-selection')
            if (wrapper) {
              (wrapper as HTMLElement).focus()
            }
          })
        }
      }

      suggestions.value = []
      showDropdown.value = false
    }

    const removeItem = (index: number) => {
      selectedItems.value.splice(index, 1)
      emit('update:selected', selectedItems.value)
      showLimitMessage.value = false

      if (props.maxSelected === 1) {
        nextTick(() => {
          focusInput()
        })
      }
    }

    const handleKeyDown = (e: KeyboardEvent) => {
      if (!showDropdown.value || suggestions.value.length === 0) return

      switch (e.key) {
        case 'ArrowDown':
          e.preventDefault()
          activeIndex.value = (activeIndex.value + 1) % suggestions.value.length
          scrollToActiveItem()
          break;
        case 'ArrowUp':
          e.preventDefault()
          activeIndex.value = activeIndex.value > 0 ? activeIndex.value - 1 : suggestions.value.length - 1
          scrollToActiveItem()
          break
        case 'Enter':
          if (activeIndex.value >= 0 && activeIndex.value < suggestions.value.length) {
            e.preventDefault()
            selectItem(suggestions.value[activeIndex.value])
          }
          break
        case 'Escape':
          showDropdown.value = false
          break
        case 'Backspace':
          if (inputValue.value === '' && selectedItems.value.length > 0) {
            removeItem(selectedItems.value.length - 1)
          }
          break
      }
    }

    const scrollToActiveItem = () => {
      const list = document.querySelector('.suggestions-list')
      if (!list) return

      const activeItem = list.querySelector('.active') as HTMLElement
      if (activeItem) {
        activeItem.scrollIntoView({
          behavior: 'smooth',
          block: 'nearest'
        })
      }
    }

    const focusInput = () => {
      inputRef.value?.focus()
    }

    const handleFocus = () => {
      inputFocused.value = true
      showDropdown.value = true
    }

    const handleBlur = () => {
      inputFocused.value = false
      setTimeout(() => {
        showDropdown.value = false
      }, 200)
    }

    let debounceTimer: number | null = null
    const handleInputDebounced = () => {
      if (debounceTimer) {
        clearTimeout(debounceTimer);
      }

      if (inputValue.value.length >= 3) {
        debounceTimer = window.setTimeout(() => {
          fetchSuggestions(inputValue.value);
        }, 300);
      } else {
        suggestions.value = [];
        error.value = null;
      }
    };

    return {
      inputValue,
      suggestions,
      selectedItems,
      loading,
      error,
      showDropdown,
      activeIndex,
      inputRef,
      errorMessage,
      remainingChars,
      remainingCharsText,
      showInitialMessage,
      computedPlaceholder,
      isInputDisabled,
      showLimitMessage,
      handleInputDebounced,
      selectItem,
      removeItem,
      handleKeyDown,
      focusInput,
      handleFocus,
      handleBlur
    }
  }
})
</script>

<style lang="scss">
.suggest-container {
  position: relative;
  width: 100%;
  max-width: 350px;
  font-family: Arial, sans-serif;
  text-align: left;

  label {
    display: block;
    margin-bottom: 8px;
    font-weight: bold;
    font-size: 14px;
    color: #e0e0e0;
  }
}

.limit-message {
  margin-bottom: 8px;
  border-radius: 4px;
  color: #ff9800;
  font-size: 11px;
  font-weight: 500;
}

.input-wrapper {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  flex-wrap: wrap;
  gap: 6px;
  padding: 8px;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  background: white;
  cursor: text;
  min-height: 55px;
  transition: border-color 0.2s;

  &:focus-within {
    border-color: #22e5ba;
    box-shadow: 0 0 6px 3px rgba(64, 255, 172, 0.2);
  }

  &.has-tags {
    align-items: flex-start;
  }

  &.single-selection {
    overflow: hidden;
    cursor: default;

    .tags-container {
      width: 100%;
    }

    .tag {
      width: auto;
      height: auto;
      display: flex;
      justify-content: space-between;
    }
  }

  .tags-container {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    flex: 1;
    min-height: 28px;
    align-items: center;
  }

  .tag {
    display: flex;
    align-items: center;
    padding: 8px 12px;
    background: #468c96;
    border-radius: 4px;
    font-size: 13px;
    height: 26px;
    box-sizing: border-box;
    transition: background 0.2s;
    flex-shrink: 0;

    &:hover {
      background: #468c96;
    }

    .tag-remove {
      margin-left: 6px;
      cursor: pointer;
      font-size: 14px;
      line-height: 1;
      width: 16px;
      height: 16px;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 50%;

      &:hover {
        background: #357d6e;
        color: #000;
      }
    }
  }
}

.suggest-input {
  width: 100%;
  flex: 1;
  min-width: 60px;
  border: none;
  outline: none;
  padding: 4px;
  font-size: 14px;
  height: 28px;
  box-sizing: border-box;
  background: transparent;
  flex-shrink: 0;
  color: #1a1a1a;

  &:disabled {
    display: none;
  }
}

.dropdown {
  position: absolute;
  top: calc(100% + 5px);
  left: 0;
  right: 0;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
  background: white;
  box-shadow: 0 0 6px 3px rgba(64, 255, 172, 0.2);
  z-index: 1000;
  max-height: 300px;
  overflow: hidden;

  .dropdown-status {
    padding: 12px;
    text-align: center;
    color: #909399;
    font-size: 13px;

    &.error {
      color: #f56c6c;
    }
  }

  .dropdown-message {
    padding: 12px;
    text-align: center;

    .message-title {
      font-weight: 500;
      font-size: 14px;
      margin-bottom: 8px;
      color: #333;
    }

    .message-sub {
      font-size: 13px;
      color: #666;
    }
  }

  .suggestions-list {
    list-style: none;
    padding: 0;
    margin: 0;
    max-height: 240px;
    overflow-y: auto;

    li {
      padding: 10px 12px;
      cursor: pointer;
      transition: background 0.2s;
      border-bottom: 1px solid #f0f2f5;

      &:last-child {
        border-bottom: none;
      }

      &:hover, &.active {
        background: #f5f7fa;
      }
    }
  }
}
</style>