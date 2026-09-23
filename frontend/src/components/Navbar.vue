<template>
  <el-menu mode="horizontal" :ellipsis="false" class="navbar">
    <el-menu-item index="home" @click="goHome">
      <span class="logo">Blog Platform</span>
    </el-menu-item>
    <div class="flex-grow"></div>
    <div class="search-container">
      <el-input
        v-model="searchQuery"
        placeholder="搜索文章标题或摘要..."
        clearable
        class="search-input"
        @keyup.enter="handleSearch"
        @clear="handleClear"
      >
        <template #append>
          <el-button @click="handleSearch">
            <el-icon><Search /></el-icon>
          </el-button>
        </template>
      </el-input>
    </div>
    <el-menu-item index="articles" @click="goHome">
      文章
    </el-menu-item>
    <template v-if="authStore.isLoggedIn">
      <el-menu-item index="dashboard" @click="goDashboard">
        管理面板
      </el-menu-item>
      <el-menu-item index="logout" @click="handleLogout">
        退出 ({{ authStore.username }})
      </el-menu-item>
    </template>
    <template v-else>
      <el-menu-item index="login" @click="goLogin">
        管理员登录
      </el-menu-item>
    </template>
  </el-menu>
</template>

<script setup>
import { ref, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useAuthStore } from '../stores/auth'
import { ElMessage } from 'element-plus'
import { Search } from '@element-plus/icons-vue'

const router = useRouter()
const route = useRoute()
const authStore = useAuthStore()

const searchQuery = ref(route.query.search || '')

// URL 是搜索条件的唯一数据源：标签切换、浏览器前进 / 后退等导致 query 变化时，
// 输入框保持同步，避免输入框与列表实际条件不一致
watch(() => route.query.search, (search) => {
  searchQuery.value = search || ''
})

function goHome() {
  router.push('/')
}

function goLogin() {
  router.push('/login')
}

function goDashboard() {
  router.push('/admin')
}

function handleLogout() {
  authStore.logout()
  ElMessage.success('已退出登录')
  router.push('/')
}

function handleSearch() {
  const query = searchQuery.value.trim()
  if (query) {
    router.push({ path: '/', query: { search: query } })
  }
}

function handleClear() {
  if (route.path === '/' && route.query.search) {
    router.push({ path: '/', query: {} })
  }
}
</script>

<style scoped>
.navbar {
  margin-bottom: 20px;
}

.logo {
  font-size: 20px;
  font-weight: bold;
  color: #409eff;
}

.flex-grow {
  flex-grow: 1;
}

.search-container {
  display: flex;
  align-items: center;
  padding: 0 10px;
}

.search-input {
  width: 280px;
}
</style>
