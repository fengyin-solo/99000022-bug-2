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
          <template v-if="!loadError">
            <ArticleCard
              v-for="article in articles"
              :key="article.id"
              :article="article"
              :highlight-query="searchQuery"
              @tag-click="handleTagSelect"
            />
          </template>

          <el-empty v-if="!loading && loadError" description="文章加载失败，请稍后重试">
            <el-button type="primary" @click="retryFetch">重试</el-button>
          </el-empty>
          <el-empty v-else-if="!loading && articles.length === 0" :description="emptyDescription" />
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
import { ref, computed, onMounted, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import api from '../api'
import ArticleCard from '../components/ArticleCard.vue'
import TagFilter from '../components/TagFilter.vue'
import Pagination from '../components/Pagination.vue'

const route = useRoute()
const router = useRouter()

const articles = ref([])
const tags = ref([])
const loading = ref(false)
const loadError = ref(false)
const pagination = ref({
  total: 0,
  page: 1,
  limit: 10,
  totalPages: 0
})

// 查询条件以路由 query 为唯一事实来源，搜索/标签/分页三个入口都只修改 URL
const selectedTag = computed(() => normalizeQueryValue(route.query.tag))
const searchQuery = computed(() => normalizeQueryValue(route.query.search) || '')
const currentPage = computed(() => {
  const page = parseInt(normalizeQueryValue(route.query.page), 10)
  return Number.isInteger(page) && page > 0 ? page : 1
})

function normalizeQueryValue(value) {
  if (Array.isArray(value)) return value[0] ?? null
  return value ?? null
}

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

// 请求序号：只有最新一次请求允许写入列表与分页，
// 避免快速切换条件时旧条件的响应覆盖新结果
let fetchSeq = 0

async function fetchArticles() {
  const seq = ++fetchSeq
  loading.value = true
  loadError.value = false
  try {
    const params = {
      page: currentPage.value,
      limit: pagination.value.limit
    }
    if (selectedTag.value) {
      params.tag = selectedTag.value
    }
    if (searchQuery.value) {
      params.search = searchQuery.value
    }

    const response = await api.get('/articles', { params })
    if (seq !== fetchSeq) return
    articles.value = response.data.articles
    pagination.value = response.data.pagination
  } catch (error) {
    if (seq !== fetchSeq) return
    console.error('Failed to fetch articles:', error)
    // 失败时清空数据并进入可识别的错误态，而不是继续展示上一次的结果
    articles.value = []
    pagination.value = { total: 0, page: 1, limit: 10, totalPages: 0 }
    loadError.value = true
  } finally {
    if (seq === fetchSeq) {
      loading.value = false
    }
  }
}

// 路由 query 变化（搜索、标签、分页、前进后退）是触发请求的唯一入口
watch(
  () => [route.query.tag, route.query.search, route.query.page],
  fetchArticles,
  { immediate: true }
)

onMounted(() => {
  fetchTags()
})

async function fetchTags() {
  try {
    const response = await api.get('/tags')
    tags.value = response.data.tags
  } catch (error) {
    console.error('Failed to fetch tags:', error)
  }
}

// 所有条件变更统一收敛为一次路由导航，由上面的 watch 发起请求
function navigateWithQuery({ tag = selectedTag.value, search = searchQuery.value, page = 1 } = {}) {
  const query = {}
  if (tag) query.tag = tag
  if (search) query.search = search
  if (page > 1) query.page = String(page)
  // 条件未变化时无需导航（避免重复导航告警）
  if (
    query.tag === (route.query.tag || undefined) &&
    query.search === (route.query.search || undefined) &&
    query.page === (route.query.page || undefined)
  ) {
    return
  }
  router.push({ query })
}

function handlePageChange(page) {
  navigateWithQuery({ page })
}

function handleTagSelect(tag) {
  navigateWithQuery({ tag, page: 1 })
}

function clearSearch() {
  navigateWithQuery({ search: '', page: 1 })
}

function retryFetch() {
  fetchArticles()
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
</style>
