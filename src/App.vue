<template>
  <div>
    <h1>Саджест</h1>
    <div class="counter-controls">
      <label>Максимальный выбор: {{ counter }}</label>

      <div class="counter-controls__buttons">
        <button @click="counter > 0 && counter--">-</button>
        <button @click="counter = 0">Сбросить</button>
        <button @click="incrementCounter" :disabled="counter >= 10">+</button>
      </div>

    </div>

    <SuggestMention
        label="* Пользователь или компания"
        api-url="https://habr.com/kek/v2/publication/suggest-mention"
        :max-selected="counter"
        placeholder="Введите логин"
        @update:selected="handleSelection"
    />
  </div>

  <div class="notice">
    <span>*0 - бесконечное количество выбора пользователей</span>
    <span>*10 - максимальное количество выбора пользователей</span>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'
import SuggestMention from './components/SuggestMention.vue'

export default defineComponent({
  name: 'App',
  components: { SuggestMention },
  setup() {
    const counter = ref(0)

    const incrementCounter = () => {
      if (counter.value < 10) {
        counter.value++
      }
    };

    const handleSelection = (items: any[]) => {
      console.log('Выбранные элементы:', items)
    }

    return {
      counter,
      incrementCounter,
      handleSelection
    }
  }
})
</script>

<style scoped lang="scss">
.counter-controls {
  margin: 15px 0;
  padding: 10px;
  border: 1px solid #eee;
  border-radius: 4px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;

  &__buttons {
    display: flex;
    gap: 10px;
  }
}

button {
  padding: 5px 10px;
  background: #42b983;
  color: white;
  border: none;
  border-radius: 3px;
  cursor: pointer;
}

button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}

.notice {
  display: flex;
  flex-direction: column;
  gap: 2px;
  font-size: 10px;
  margin-top: 20px;
  color: #cccccc;
}
</style>