<template>
  <div class="home">
    <el-row :gutter="20">
      <el-col :span="18">
        <h2 class="page-title">
          {{ pageTitle }}
          <el-tag v-if="searchQuery" type="info" class="search-tag" closable @close="clearSearch">
            搜索: {{ searchQuery }}
          </el-tag>
        </h2>

        <div v-loading="loading">
          <ArticleCard
            v-for="article in articles"
            :key="article.id"
            :article="article"
            :highlight-query="searchQuery"
            @tag-click="handleTagSelect"
          />

          <el-alert
            v-if="!loading && error"
            type="error"
            show-icon
            :closable="false"
            title="文章加载失败"
            description="当前条件下的列表加载失败，请稍后重试。"
            class="error-alert"
          >
            <el-button type="primary" size="small" @click="fetchArticles">重试</el-button>
          </el-alert>

          <el-empty v-if="!loading && !error && articles.length === 0" :description="emptyDescription" />
        </div>

        <Pagination
          :model-value="currentPage"
          :total="pagination.total"
          :page-size="pagination.limit"
          @change="handlePageChange"
        />
      </el-col>

      <el-col :span="6">
        <TagFilter
          :tags="tags"
          :selected-tag="selectedTag"
          @select="handleTagSelect"
        />
      </el-col>
    </el-row>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import api from '../api'
import ArticleCard from '../components/ArticleCard.vue'
import TagFilter from '../components/TagFilter.vue'
import Pagination from '../components/Pagination.vue'

const PAGE_LIMIT = 10

const route = useRoute()
const router = useRouter()

// URL 是列表条件（标签 / 搜索 / 页码）的唯一数据源，三个入口都只通过
// 改路由 query 来改变条件，由下面的 watcher 统一发起一次请求。
const selectedTag = computed(() => route.query.tag || null)
const searchQuery = computed(() => route.query.search || '')
const currentPage = computed(() => {
  const page = Number.parseInt(route.query.page, 10)
  return Number.isInteger(page) && page > 0 ? page : 1
})

const articles = ref([])
const tags = ref([])
const loading = ref(false)
const error = ref(false)
const pagination = ref({
  total: 0,
  page: 1,
  limit: PAGE_LIMIT,
  totalPages: 0
})

// 只追踪最新一次请求，避免快速切换条件时旧响应晚到、覆盖新条件的结果
let latestController = null

const pageTitle = computed(() => {
  if (searchQuery.value) {
    return '搜索结果'
  }
  return selectedTag.value ? `标签: ${selectedTag.value}` : '最新文章'
})

const emptyDescription = computed(() => {
  if (searchQuery.value) {
    return '未找到匹配的文章'
  }
  return '暂无文章'
})

function buildQuery({ tag = selectedTag.value, search = searchQuery.value, page = 1 } = {}) {
  const query = {}
  if (tag) query.tag = tag
  if (search) query.search = search
  if (page > 1) query.page = String(page)
  return query
}

async function fetchArticles() {
  // 取消上一次尚未完成的请求：快速连续切换条件时只有最后一次会落地
  if (latestController) {
    latestController.abort()
  }
  const controller = new AbortController()
  latestController = controller

  loading.value = true
  error.value = false

  const params = {
    page: currentPage.value,
    limit: PAGE_LIMIT
  }
  if (selectedTag.value) {
    params.tag = selectedTag.value
  }
  if (searchQuery.value) {
    params.search = searchQuery.value
  }

  try {
    const response = await api.get('/articles', { params, signal: controller.signal })
    if (controller !== latestController) return // 已被更新的条件取代，结果丢弃
    // 列表与分页信息一起落地，避免两者来自不同条件
    articles.value = response.data.articles
    pagination.value = response.data.pagination
  } catch (err) {
    if (controller !== latestController || err.code === 'ERR_CANCELED') return
    // 失败时清空上一次条件的残留数据，显式展示可识别的错误状态，
    // 标题/标签/页码仍对应当前 URL 条件
    console.error('Failed to fetch articles:', err)
    error.value = true
    articles.value = []
    pagination.value = {
      total: 0,
      page: currentPage.value,
      limit: PAGE_LIMIT,
      totalPages: 0
    }
  } finally {
    if (controller === latestController) {
      loading.value = false
    }
  }
}

async function fetchTags() {
  try {
    const response = await api.get('/tags')
    tags.value = response.data.tags
  } catch (error) {
    console.error('Failed to fetch tags:', error)
  }
}

// immediate 同时覆盖首次挂载（含从详情页返回时的重建）和前进 / 后退
watch(() => route.query, () => {
  fetchArticles()
}, { immediate: true })

onMounted(() => {
  fetchTags()
})

onBeforeUnmount(() => {
  if (latestController) {
    latestController.abort()
  }
})

function handlePageChange(page) {
  if (page === currentPage.value) return
  // 页码进入 URL：前进 / 后退、从详情页返回都能恢复到准确的页
  router.push({ query: buildQuery({ page }) })
}

function handleTagSelect(tag) {
  // 切换标签重置到第 1 页；只改路由，由 watcher 统一请求，不再重复 fetch
  router.replace({ query: buildQuery({ tag, page: 1 }) })
}

function clearSearch() {
  router.replace({ query: buildQuery({ search: '', page: 1 }) })
}
</script>

<style scoped>
.home {
  padding-top: 20px;
}

.page-title {
  font-size: 24px;
  color: #303133;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  gap: 12px;
}

.search-tag {
  font-size: 14px;
  font-weight: normal;
}

.error-alert {
  margin-bottom: 16px;
}
</style>
