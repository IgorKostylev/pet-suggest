<template>
  <div class="mention-item">
    <div class="avatar">
      <img v-if="item.avatar" :src="item.avatar" alt="avatar">
      <div v-else class="avatar-placeholder">
        <span></span>
      </div>
    </div>
    <div class="info">
      <div class="name">{{ displayName }}</div>
      <div class="meta">
        <template v-if="item.type === 'user'">@{{ item.alias }}</template>
        <template v-else>Компания</template>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, computed } from 'vue'
import type { PropType } from 'vue'
import type { MentionItem } from './SuggestMention.vue'

export default defineComponent({
  name: 'MentionItem',
  props: {
    item: {
      type: Object as PropType<MentionItem>,
      required: true
    }
  },
  setup(props) {
    const displayName = computed(() => {
      return props.item.name || props.item.alias
    })

    return {
      displayName
    }
  }
})
</script>

<style scoped lang="scss">
.mention-item {
  display: flex;
  align-items: center;
}

.avatar {
  width: 32px;
  height: 32px;
  margin-right: 10px;
  flex-shrink: 0;

  img {
    width: 100%;
    height: 100%;
    border-radius: 50%;
    object-fit: cover;
  }
}

.avatar-placeholder {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  background: #e0e0e0;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #909399;
  font-size: 14px;
}

.info {
  color: #1a1a1a;
  overflow: hidden;
  text-align: left;
}

.name {
  font-weight: 500;
  font-size: 14px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.meta {
  font-size: 12px;
  color: #909399;
  margin-top: 2px;
}
</style>