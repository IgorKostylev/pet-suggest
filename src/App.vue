<template>
    <h1>Саджест</h1>
    <SuggestMention
        label="* Пользователь или компания"
        api-url="https://habr.com/kek/v2/publication/suggest-mention"
        :max-selected="2"
        placeholder="Введите логин"
        @update:selected="handleSelection"
    />
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import SuggestMention from './components/SuggestMention.vue';

export default defineComponent({
  name: 'App',
  components: { SuggestMention },
  methods: {
    handleSelection(items: any[]) {
      console.log('Выбранные элементы:', items);
    }
  }
});
</script>