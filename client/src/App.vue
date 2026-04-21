<template>
  <div class="app-shell">
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }">
      <div class="sidebar-header">
        <div class="sidebar-logo-full" v-show="!sidebarCollapsed">
          <div class="sidebar-logo-name">{{ t('nav.companyName') }}</div>
          <div class="sidebar-logo-sub">{{ t('nav.subtitle') }}</div>
        </div>
        <div class="sidebar-logo-icon" v-show="sidebarCollapsed">CC</div>
      </div>

      <nav class="sidebar-nav">
        <router-link to="/" :class="{ active: $route.path === '/' }" :title="sidebarCollapsed ? t('nav.overview') : ''">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
            <path d="M2 6.5L8 2l6 4.5V13a1 1 0 01-1 1H3a1 1 0 01-1-1V6.5z"/>
            <path d="M6 14V9h4v5"/>
          </svg>
          <span class="nav-label">{{ t('nav.overview') }}</span>
        </router-link>

        <router-link to="/inventory" :class="{ active: $route.path === '/inventory' }" :title="sidebarCollapsed ? t('nav.inventory') : ''">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
            <path d="M8 1L14 4.5V11.5L8 15L2 11.5V4.5L8 1z"/>
            <path d="M8 8L14 4.5M8 8L2 4.5M8 8V15"/>
          </svg>
          <span class="nav-label">{{ t('nav.inventory') }}</span>
        </router-link>

        <router-link to="/orders" :class="{ active: $route.path === '/orders' }" :title="sidebarCollapsed ? t('nav.orders') : ''">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
            <rect x="3" y="1" width="10" height="14" rx="1"/>
            <path d="M6 5h4M6 8h4M6 11h2"/>
          </svg>
          <span class="nav-label">{{ t('nav.orders') }}</span>
        </router-link>

        <router-link to="/spending" :class="{ active: $route.path === '/spending' }" :title="sidebarCollapsed ? t('nav.finance') : ''">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
            <rect x="1" y="10" width="3" height="5" rx="0.5"/>
            <rect x="6" y="6" width="3" height="9" rx="0.5"/>
            <rect x="11" y="2" width="3" height="13" rx="0.5"/>
          </svg>
          <span class="nav-label">{{ t('nav.finance') }}</span>
        </router-link>

        <router-link to="/demand" :class="{ active: $route.path === '/demand' }" :title="sidebarCollapsed ? t('nav.demandForecast') : ''">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
            <polyline points="1,11 5,7 9,9 15,3"/>
            <polyline points="11,3 15,3 15,7"/>
          </svg>
          <span class="nav-label">{{ t('nav.demandForecast') }}</span>
        </router-link>

        <router-link to="/reports" :class="{ active: $route.path === '/reports' }" :title="sidebarCollapsed ? 'Reports' : ''">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
            <rect x="2" y="1" width="12" height="14" rx="1"/>
            <path d="M5 5h6M5 8h6M5 11h3"/>
            <path d="M10 10l4 4"/>
            <circle cx="11" cy="10" r="2.5"/>
          </svg>
          <span class="nav-label">Reports</span>
        </router-link>

        <router-link to="/restocking" :class="{ active: $route.path === '/restocking' }" :title="sidebarCollapsed ? 'Restocking' : ''">
          <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
            <path d="M14 8A6 6 0 102 8"/>
            <polyline points="14,4 14,8 10,8"/>
          </svg>
          <span class="nav-label">Restocking</span>
        </router-link>
      </nav>

      <div class="sidebar-spacer"></div>

      <div class="sidebar-bottom">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>

      <button class="sidebar-toggle" @click="toggleSidebar" :title="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'">
        <svg width="14" height="14" viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <polyline :points="sidebarCollapsed ? '5,2 9,7 5,12' : '9,2 5,7 9,12'" />
        </svg>
      </button>
    </aside>

    <div class="main-area" :style="{ marginLeft: sidebarCollapsed ? '56px' : '220px', width: sidebarCollapsed ? 'calc(100% - 56px)' : 'calc(100% - 220px)' }">
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Sidebar collapse state — toggled by user or auto-set by viewport width
    const sidebarCollapsed = ref(false)

    const toggleSidebar = () => {
      sidebarCollapsed.value = !sidebarCollapsed.value
    }

    // Auto-collapse below 900px, expand above — fires on both mount and resize
    const handleResize = () => {
      if (window.innerWidth < 900) {
        sidebarCollapsed.value = true
      } else {
        sidebarCollapsed.value = false
      }
    }

    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(() => {
      loadTasks()
      handleResize()
      window.addEventListener('resize', handleResize)
    })

    onUnmounted(() => {
      window.removeEventListener('resize', handleResize)
    })

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
      sidebarCollapsed,
      toggleSidebar
    }
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:ital,opsz,wght@0,9..40,300;0,9..40,400;0,9..40,500;0,9..40,600;0,9..40,700;1,9..40,400&family=DM+Mono:wght@400;500&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'DM Sans', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #f8fafc;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ─── Shell layout ─────────────────────────────────────────── */

.app-shell {
  display: flex;
  min-height: 100vh;
}

/* ─── Sidebar ──────────────────────────────────────────────── */

.sidebar {
  background: #0f172a;
  width: 220px;
  min-height: 100vh;
  position: fixed;
  left: 0;
  top: 0;
  display: flex;
  flex-direction: column;
  z-index: 100;
  border-right: none;
  /* Subtle separator between sidebar and content */
  box-shadow: 1px 0 0 0 rgba(255, 255, 255, 0.06);
  /* Animate width on collapse/expand */
  transition: width 0.22s cubic-bezier(0.4, 0, 0.2, 1);
  /* Allow toggle button to overflow the sidebar boundary */
  overflow: visible;
}

.sidebar-header {
  padding: 1.5rem 1.25rem 1.25rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  /* Ensure header aligns content correctly in both states */
  display: flex;
  align-items: center;
  min-height: 68px;
  /* Clip overflowing text inside header without clipping the toggle button */
  overflow: hidden;
}

.sidebar-logo-name {
  font-size: 0.938rem;
  font-weight: 700;
  color: #ffffff;
  letter-spacing: -0.01em;
}

.sidebar-logo-sub {
  font-size: 0.75rem;
  color: rgba(255, 255, 255, 0.4);
  margin-top: 2px;
  font-weight: 400;
}

/* Monogram shown when collapsed — two-letter colored square */
.sidebar-logo-icon {
  width: 32px;
  height: 32px;
  background: rgba(37, 99, 235, 0.5);
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.75rem;
  font-weight: 700;
  color: #ffffff;
  letter-spacing: 0.02em;
  margin: 0 auto;
  flex-shrink: 0;
}

.sidebar-nav {
  padding: 1rem 0.75rem;
  display: flex;
  flex-direction: column;
  gap: 2px;
  /* Clip overflowing text in nav without clipping toggle button */
  overflow: hidden;
}

.sidebar-nav a {
  display: flex;
  align-items: center;
  gap: 0.625rem;
  padding: 0.5rem 0.75rem;
  border-radius: 7px;
  color: rgba(255, 255, 255, 0.55);
  font-size: 0.875rem;
  font-weight: 500;
  text-decoration: none;
  transition: all 0.15s ease;
  letter-spacing: 0.005em;
  /* Clip label overflow within the link; toggle button is outside nav entirely */
  overflow: hidden;
}

.sidebar-nav a:hover {
  color: rgba(255, 255, 255, 0.9);
  background: rgba(255, 255, 255, 0.07);
}

/* router-link-active is set by Vue Router on the matched link;
   .active is the manual :class binding — both must be styled */
.sidebar-nav a.router-link-active,
.sidebar-nav a.active {
  color: #ffffff;
  background: rgba(37, 99, 235, 0.35);
  font-weight: 600;
}

.sidebar-nav a svg {
  flex-shrink: 0;
}

.sidebar-spacer {
  flex: 1;
}

.sidebar-bottom {
  padding: 0.875rem 0.75rem;
  border-top: 1px solid rgba(255, 255, 255, 0.08);
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  /* Clip overflowing content inside bottom section */
  overflow: hidden;
}

/* ─── Main area ────────────────────────────────────────────── */

.main-area {
  margin-left: 220px;
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background: #f8fafc;
  width: calc(100% - 220px);
  /* Animate in sync with sidebar width transition */
  transition: margin-left 0.22s cubic-bezier(0.4, 0, 0.2, 1),
              width 0.22s cubic-bezier(0.4, 0, 0.2, 1);
}

.main-content {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.75rem 2rem;
}

/* ─── Sidebar collapsed state ──────────────────────────────── */

.sidebar.collapsed {
  width: 56px;
}

/* Nav label: animates out when collapsed, in when expanded */
.nav-label {
  overflow: hidden;
  white-space: nowrap;
  transition: opacity 0.15s ease, max-width 0.22s cubic-bezier(0.4, 0, 0.2, 1);
  max-width: 160px;
  opacity: 1;
}

.sidebar.collapsed .nav-label {
  max-width: 0;
  opacity: 0;
}

/* Center the icon when collapsed and provide tooltip positioning context */
.sidebar.collapsed .sidebar-nav a {
  justify-content: center;
  padding: 0.5rem;
  position: relative;
}

/* Tooltip body shown to the right of collapsed nav icons on hover */
.sidebar.collapsed .sidebar-nav a[title]:hover::after {
  content: attr(title);
  position: absolute;
  left: calc(100% + 10px);
  top: 50%;
  transform: translateY(-50%);
  background: #1e293b;
  color: #f1f5f9;
  padding: 0.3rem 0.625rem;
  border-radius: 6px;
  font-size: 0.8rem;
  font-weight: 500;
  white-space: nowrap;
  pointer-events: none;
  z-index: 200;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
}

/* Tooltip arrow pointing left toward the icon */
.sidebar.collapsed .sidebar-nav a[title]:hover::before {
  content: '';
  position: absolute;
  left: calc(100% + 5px);
  top: 50%;
  transform: translateY(-50%);
  border: 5px solid transparent;
  border-right-color: #1e293b;
  pointer-events: none;
  z-index: 200;
}

/* Collapsed bottom section: center items */
.sidebar.collapsed .sidebar-bottom {
  align-items: center;
  padding: 0.875rem 0.5rem;
}

/* ─── Toggle button ────────────────────────────────────────── */

/* Button sits at the right edge of the sidebar, half-inside/half-outside.
   Sidebar uses overflow: visible so the button is not clipped. */
.sidebar-toggle {
  position: absolute;
  right: -12px;
  top: 50%;
  transform: translateY(-50%);
  width: 24px;
  height: 24px;
  background: #1e293b;
  border: 1.5px solid rgba(255,255,255,0.15);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  color: rgba(255,255,255,0.7);
  transition: all 0.15s ease;
  z-index: 110;
  flex-shrink: 0;
}

.sidebar-toggle:hover {
  background: #2d3f5a;
  color: #ffffff;
  border-color: rgba(255,255,255,0.3);
}

/* ─── Global utility classes (used by all views) ───────────── */

.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 0.375rem;
  letter-spacing: -0.025em;
}

.page-header p {
  color: #64748b;
  font-size: 0.938rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  background: white;
  padding: 1.25rem;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: #64748b;
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.625rem;
}

.stat-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: #ea580c;
}

.stat-card.success .stat-value {
  color: #059669;
}

.stat-card.danger .stat-value {
  color: #dc2626;
}

.stat-card.info .stat-value {
  color: #2563eb;
}

.card {
  background: white;
  border-radius: 10px;
  padding: 1.25rem;
  border: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
  border-bottom: 1px solid #e2e8f0;
}

th {
  text-align: left;
  padding: 0.5rem 0.75rem;
  font-weight: 600;
  color: #475569;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: 0.5rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  color: #334155;
  font-size: 0.875rem;
}

tbody tr {
  transition: background-color 0.15s ease;
}

tbody tr:hover {
  background: #f8fafc;
}

.badge {
  display: inline-block;
  padding: 0.313rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.decreasing {
  background: #fecaca;
  color: #991b1b;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  font-size: 0.938rem;
}
</style>
