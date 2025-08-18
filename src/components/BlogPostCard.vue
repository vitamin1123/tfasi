<!-- src/components/BlogPostCard.vue -->
<template>
  <motion.div
    :initial="{ opacity: 0, y: 50, scale: 0.95 }"
    :while-in-view="{ opacity: 1, y: 0, scale: 1 }"
    :viewport="{ once: true, amount: 0.3 }"
    :transition="{ duration: 0.6, ease: 'easeOut' }"
    class="group bg-slate-800/50 backdrop-blur-sm border border-slate-700/50 rounded-xl overflow-hidden shadow-lg hover:shadow-sky-500/30 transition-all duration-300"
  >
    <div class="aspect-video bg-gray-700">
      <!-- 你可以用 <img> 替换这里 -->
      <div class="w-full h-full flex items-center justify-center text-slate-400">Image Placeholder</div>
    </div>
    <div class="p-6">
      <h3 class="text-2xl font-bold text-sky-400 group-hover:text-sky-300 transition-colors duration-300 mb-2">{{ post.title }}</h3>
      <p class="text-sm text-slate-400 mb-4">{{ post.date }} - {{ post.excerpt }}</p>
      <button @click="openPost" class="inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-sky-600 hover:bg-sky-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-slate-800 focus:ring-sky-500">
        Read More
      </button>
    </div>
  </motion.div>
</template>

<script setup lang="ts">
import { defineProps } from 'vue';
import { motion } from 'motion-v';

interface Post {
  id: string;
  title: string;
  date: string;
  excerpt: string;
  content: string; // Markdown content
}

const props = defineProps<{
  post: Post;
}>();

const emit = defineEmits<{
  (e: 'openPost', post: Post): void;
}>();

const openPost = () => {
  emit('openPost', props.post);
};
</script>