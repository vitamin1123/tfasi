<!-- src/App.vue -->
<template>
  <div class="min-h-screen bg-transparent text-slate-100">
    <ThreeScene />

    <!-- 遮罩层，让前景内容更突出 -->
    <div class="fixed inset-0 bg-black/50 -z-10"></div>

    <!-- 主内容区 -->
    <main class="relative z-0 container mx-auto px-4 sm:px-6 lg:px-8 py-12">
      <header class="text-center py-16">
        <motion.h1
          :initial="{ opacity: 0, y: -30 }"
          :animate="{ opacity: 1, y: 0 }"
          :transition="{ duration: 1, delay: 0.2 }"
          class="text-5xl md:text-7xl font-extrabold tracking-tight"
        >
          <span class="bg-clip-text text-transparent bg-gradient-to-r from-sky-400 to-indigo-500">
            Midnight Musings
          </span>
        </motion.h1>
        <motion.p
          :initial="{ opacity: 0 }"
          :animate="{ opacity: 1 }"
          :transition="{ duration: 1, delay: 0.5 }"
          class="mt-4 text-xl text-slate-300"
        >
          A sanctuary for thoughts under the warm glow of a lamp.
        </motion.p>
      </header>

      <section class="py-12">
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8">
          <BlogPostCard
            v-for="post in posts"
            :key="post.id"
            :post="post"
            @open-post="handleOpenPost"
          />
        </div>
      </section>
    </main>

    <!-- 文章详情模态框 -->
    <motion.div
      v-if="selectedPost"
      :initial="{ opacity: 0 }"
      :animate="{ opacity: 1 }"
      :exit="{ opacity: 0 }"
      class="fixed inset-0 bg-black/80 backdrop-blur-md flex items-center justify-center p-4 z-50"
      @click.self="closePost"
    >
      <motion.div
        :initial="{ scale: 0.9, opacity: 0 }"
        :animate="{ scale: 1, opacity: 1 }"
        :exit="{ scale: 0.9, opacity: 0 }"
        :transition="{ type: 'spring', damping: 25, stiffness: 200 }"
        class="bg-slate-800 rounded-xl shadow-2xl max-w-3xl w-full max-h-[90vh] overflow-y-auto p-6 sm:p-8"
      >
        <div class="flex justify-between items-center mb-6">
          <h2 class="text-3xl font-bold text-sky-400">{{ selectedPost.title }}</h2>
          <button @click="closePost" class="text-slate-400 hover:text-slate-200 transition-colors">
            <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-8 h-8">
              <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
            </svg>
          </button>
        </div>
        <p class="text-sm text-slate-500 mb-4">{{ selectedPost.date }}</p>
        <MarkdownContent :content="selectedPost.content" />
      </motion.div>
    </motion.div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import ThreeScene from './ThreeScene.vue';
import BlogPostCard from './BlogPostCard.vue';

import MarkdownContent from './MarkdownContent.vue';
import { motion } from 'motion-v';

interface Post {
  id: string;
  title: string;
  date: string;
  excerpt: string;
  content: string;
}

// 示例数据
const posts = ref<Post[]>([
  {
    id: '1',
    title: 'The Art of Solitude',
    date: 'August 18, 2025',
    excerpt: 'Exploring the profound depths of being alone, without being lonely.',
    content: `# The Art of Solitude

In the quiet corners of our existence, solitude beckons not as an absence, but as a presence. It's the space where the mind, unburdened by the immediate demands of society, can stretch its legs and truly *breathe*.

> "Solitude is fine but you need someone to tell that solitude is fine." - Honoré de Balzac

## Finding Your Inner Landscape

This journey isn't about isolation. It's about connection. A deeper, more intrinsic connection with the self. The gentle hum of the lamp, the soft creak of the old sofa – these become the subtle soundtrack to introspection.

- **Mindfulness**: Pay attention to the texture of your thoughts.
- **Creativity**: Let the blank page become a universe.
- **Peace**: Understand that silence is not empty, but full of answers.

The warmth of the light against the cool night air outside forms a perfect metaphor for the internal world we cultivate when we dare to step away from the crowd.`
  },
  {
    id: '2',
    title: 'Digital Dreams in Analog Nights',
    date: 'August 15, 2025',
    excerpt: 'When code meets candlelight, a unique kind of magic happens.',
    content: `## Digital Dreams in Analog Nights

There's a peculiar synergy between the cold logic of machines and the warm, flickering light of a single bulb. It speaks to a duality within us all – the part that seeks precision and the part that yearns for poetry.

\`\`\`javascript
const dream = () => {
    while (true) {
        create();
        reflect();
        iterate();
    }
}
\`\`\`

In these hours, the world feels both vast and intimate. The glow of the screen and the glow of the lamp compete, yet complement.`
  },
  {
    id: '3',
    title: 'Echoes from the Bookshelf',
    date: 'August 12, 2025',
    excerpt: 'Each book is a ghost, whispering stories from lives we never lived.',
    content: `### Echoes from the Bookshelf

The old wooden shelves, heavy with the weight of countless worlds, stand as silent sentinels. Their spines face outwards, a mosaic of colors and titles. Each one a portal, a memory, a friend.

1.  **Pick a book at random**.
2.  **Read the first paragraph**.
3.  **Let your mind wander**.
4.  **Return it gently**.

These objects hold more than paper and ink; they hold time itself.`
  }
]);

const selectedPost = ref<Post | null>(null);

const handleOpenPost = (post: Post) => {
  selectedPost.value = post;
  document.body.style.overflow = 'hidden'; // Prevent background scroll
};

const closePost = () => {
  selectedPost.value = null;
  document.body.style.overflow = ''; // Restore scroll
};
</script>