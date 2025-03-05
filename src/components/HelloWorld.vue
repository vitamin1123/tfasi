<template>
  <v-app>
    <!-- 固定在顶部的搜索框 -->
    <v-container class="search-container">
      <v-text-field
        v-model="search"
        label="搜索"
        outlined
        dense
        clearable
        append-icon="mdi-magnify"
        class="search-input"
        :class="{ 'search-input-focused': isFocused }"
        @focus="isFocused = true"
        @blur="isFocused = false"
      ></v-text-field>
    </v-container>

    <!-- 响应式卡片布局 -->
    <v-container>
      <v-row>
        <v-col
          v-for="(item, index) in filteredItems"
          :key="index"
          :cols="cols"
        >
          <v-card class="card-item">
            <v-card-title>{{ item.title }}</v-card-title>
            <v-card-text>{{ item.content }}</v-card-text>
          </v-card>
        </v-col>
      </v-row>
    </v-container>
  </v-app>
</template>

<script>
export default {
  data() {
    return {
      search: '',
      isFocused: false, // 控制搜索框聚焦状态
      items: [
        { title: '卡片 1', content: '这是卡片 1 的内容' },
        { title: '卡片 2', content: '这是卡片 2 的内容' },
        { title: '卡片 3', content: '这是卡片 3 的内容' },
        { title: '卡片 4', content: '这是卡片 4 的内容' },
        { title: '卡片 5', content: '这是卡片 5 的内容' },
        { title: '卡片 6', content: '这是卡片 6 的内容' },
        { title: '卡片 7', content: '这是卡片 7 的内容' },
        { title: '卡片 8', content: '这是卡片 8 的内容' },
        { title: '卡片 9', content: '这是卡片 9 的内容' },
        { title: '卡片 10', content: '这是卡片 10 的内容' },
      ],
    };
  },
  computed: {
    // 根据搜索框内容过滤卡片
    filteredItems() {
      return this.items.filter(item =>
        item.title.toLowerCase().includes(this.search.toLowerCase())
      );
    },
    // 根据屏幕宽度计算卡片列数
    cols() {
      const width = window.innerWidth;
      if (width < 600) return 12; // 手机竖屏，1列
      if (width < 960) return 6; // 手机横屏或平板，2列
      if (width < 1264) return 4; // 平板或小屏幕电脑，3列
      return 3; // 大屏幕电脑，4列
    },
  },
};
</script>

<style scoped>
/* 搜索框容器 */
.search-container {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 1000;
  display: flex;
  justify-content: center; /* 水平居中 */
  padding: 16px;
  background-color: white;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

/* 搜索框样式 */
.search-input {
  width: 70%; /* 宽度设置为 70% */
  max-width: 600px; /* 最大宽度限制 */
  transition: all 0.3s ease; /* 添加过渡动画 */
}

/* 搜索框聚焦时的样式 */
.search-input-focused {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2); /* 聚焦时增加阴影 */
  transform: scale(1.02); /* 聚焦时轻微放大 */
}

/* 卡片布局的顶部间距 */
.v-container {
  margin-top: 80px; /* 为搜索框留出空间 */
}

/* 卡片样式 */
.card-item {
  margin-bottom: 16px; /* 卡片之间的间距 */
  transition: transform 0.3s ease, box-shadow 0.3s ease; /* 卡片悬停动画 */
}

.card-item:hover {
  transform: translateY(-5px); /* 悬停时轻微上移 */
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15); /* 悬停时增加阴影 */
}
</style>