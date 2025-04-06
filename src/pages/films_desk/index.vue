<template>
  <v-app>
    <!-- 顶部搜索栏 -->
    <v-container fluid class="custom-sticky-top">
      <v-row>
        <v-col cols="3">
          <v-select
            :items="options"
            item-title="type_name"
            item-value="type_id"
            label="选择分类"
            v-model="selectedOption"
            rounded
            outlined
            variant="plain"
            dark
          ></v-select>
        </v-col>
        <v-col cols="9">
          <v-text-field
            v-model="searchQueryInput"
            label="搜索"
            placeholder="输入关键词，按回车搜索"
            rounded
            outlined
            variant="plain"
            dark
            @keydown="handleKeyDown"
          ></v-text-field>
        </v-col>
      </v-row>
    </v-container>

    <!-- 紧凑型瀑布流卡片（每行7-8个） -->
    <v-container class="desktop-container">
      <div ref="masonryRef" class="masonry-desktop">
        <div 
          v-for="item in filteredItems" 
          :key="item.vod_id" 
          class="masonry-item-desktop"
        >
          <v-card 
            class="compact-card"
            @click="handleClick(item)"
          >
            <v-img
              :src="item.vod_pic"
              aspect-ratio="2/3"
              class="rounded-t-lg"
              @load="imageLoaded"
            >
              <template #placeholder>
                <v-skeleton-loader type="image" aspect-ratio="2/3"></v-skeleton-loader>
              </template>
              <template #bottom>
                <v-card-subtitle class="bg-white text-black px-2 py-1 rounded-br-lg">
                  {{ item.tag }}
                </v-card-subtitle>
              </template>
            </v-img>
            <v-card-title class="px-2 py-1 text-caption">{{ item.vod_name }}</v-card-title>
          </v-card>
        </div>
      </div>
    </v-container>

    <v-overlay :model-value="isLoading" class="align-center justify-center">
      <v-progress-circular color="primary" indeterminate size="64"></v-progress-circular>
    </v-overlay>
  </v-app>
</template>

<script setup>
import { ref, onMounted, watch, computed, nextTick } from "vue";
import Masonry from "masonry-layout";
import { get } from "@/plugins/useAxios";
import { useRouter } from 'vue-router';

const router = useRouter();

// 数据状态
const items = ref([]);
const options = ref([]);
const selectedOption = ref("0");
const searchQuery = ref("");
const searchQueryInput = ref("");
const page = ref(1);
const pagecount = ref(Infinity);
const isLoading = ref(false);
const masonryRef = ref(null);
let masonryInstance = null;

// 滚动加载函数
const onScroll = () => {
  if (isLoading.value) return;
  const threshold = 200;
  const { scrollTop, scrollHeight, clientHeight } = document.documentElement;
  if (scrollHeight - (scrollTop + clientHeight) < threshold) {
    fetchData(page.value + 1);
  }
};

const handleKeyDown = (e) => {
  if (e.key === 'Enter') handleSearch();
};

const filteredItems = computed(() => {
  return items.value.filter((item) => {
    if (item.type_id === '19' || item.type_id === '61') return false;
    const isOptionMatch = selectedOption.value === "0" || item.type_id === selectedOption.value;
    const isSearchMatch = !searchQuery.value || 
      item.vod_name.includes(searchQuery.value) || 
      item.vod_content.includes(searchQuery.value);
    return isOptionMatch && isSearchMatch;
  });
});

const handleClick = (item) => {
  router.push({
    path: '/film_detail',
    query: { item: JSON.stringify(item) }
  });
};

const handleSearch = () => {
  searchQuery.value = searchQueryInput.value;
  page.value = 1;
  pagecount.value = Infinity;
  items.value = [];
  fetchData(1);
};

const fetchCategories = async () => {
  try {
    const data = await get("/proxy/api/get_list");
    if (data.class) {
      const filteredClasses = data.class.filter(
        item => item.type_id !== '19' && item.type_id !== '61'
      );
      options.value = [
        { type_id: "0", type_name: "全部" }, 
        ...filteredClasses
      ];
    }
  } catch (error) {
    console.error("获取分类失败:", error);
  }
};

const fetchData = async (pg) => {
  if (isLoading.value || pg < 1 || pg > pagecount.value) return;
  isLoading.value = true;

  try {
    const params = {
      t: selectedOption.value,
      pg,
    };
    if (searchQuery.value) params.wd = searchQuery.value;

    const { code, list, page: currentPage, pagecount: maxPage } = await get(
      "/proxy/api/get_detail", 
      params
    );

    if (code === 1) {
      page.value = currentPage;
      pagecount.value = maxPage;
      updateItems(list, currentPage);
    }
  } catch (error) {
    console.error("数据请求失败:", error);
  } finally {
    isLoading.value = false;
  }
};

const updateItems = (newData, currentPage) => {
  if (currentPage === 1) {
    items.value = [...newData];
  } else {
    items.value.push(...newData);
  }
};

const imageLoaded = () => {
  nextTick(() => {
    masonryInstance?.reloadItems();
    masonryInstance?.layout();
  });
};

onMounted(async () => {
  window.addEventListener("scroll", onScroll);
  await fetchCategories();
  fetchData(1);

  masonryInstance = new Masonry(masonryRef.value, {
    itemSelector: ".masonry-item-desktop",
    columnWidth: 160,
    percentPosition: false,
    gutter: 8,
    transitionDuration: 0
  });

  watch(filteredItems, () => {
    nextTick(() => {
      masonryInstance?.reloadItems();
      masonryInstance?.layout();
    });
  });
});

onUnmounted(() => {
  window.removeEventListener("scroll", onScroll);
});
</script>

<style scoped>
/* 紧凑布局样式 */
.desktop-container {
  max-width: 2000px !important;
  padding: 12px !important;
}

.masonry-desktop {
  display: flex;
  flex-wrap: wrap;
  margin: -4px;
}

.masonry-item-desktop {
  width: 160px; /* 固定宽度 */
  margin: 4px;
  box-sizing: border-box;
}

.compact-card {
  width: 100%;
  height: 100%;
  transition: all 0.2s;
}

/* 紧凑卡片内容 */
.v-img {
  border-radius: 8px 8px 0 0 !important;
}

.v-card-title {
  font-size: 0.75rem !important;
  line-height: 1.2;
  padding: 8px 6px !important;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.v-card-subtitle {
  font-size: 0.6rem !important;
  padding: 2px 4px !important;
}

/* 响应式列数调整（基于最小宽度） */
@media (min-width: 2400px) {
  .masonry-item-desktop {
    width: 140px; /* 超宽屏更紧凑 */
  }
}

@media (max-width: 1600px) {
  .masonry-item-desktop {
    width: 180px; /* 稍大保证可读性 */
  }
}

@media (max-width: 1200px) {
  .masonry-item-desktop {
    width: calc(20% - 8px); /* 5列 */
  }
}

@media (max-width: 800px) {
  .masonry-item-desktop {
    width: calc(25% - 8px); /* 4列 */
  }
}

/* 悬停效果 */
.compact-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15) !important;
}
</style>