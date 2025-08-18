<!-- src/components/MarkdownContent.vue -->
<template>
  <div class="prose prose-invert max-w-none" v-html="htmlContent"></div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';
import MarkdownIt from 'markdown-it';

const props = defineProps<{
  content: string;
}>();

const md = new MarkdownIt({
  html: true,
  linkify: true,
  typographer: true
});

const htmlContent = computed(() => md.render(props.content));
</script>