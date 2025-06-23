<template>
  <div class="sidebar">
    <div class="sidebar-header">
      <h2>Padira.</h2>
    </div>

    <ul class="nav-list">
      <li class="nav-item" v-for="item in navItems" :key="item.section">
        <a href="#"
           class="nav-link"
           :class="{ active: activeSection === item.section }"
           @click.prevent="changeSection(item.section)">
          <component :is="item.icon" class="icon" aria-hidden="true" />
          <span>{{ item.section }}</span>
        </a>
      </li>
    </ul>
  </div>
</template>

<script>
import {
  HomeIcon,
  NotebookIcon,
  TagIcon,
  NewspaperIcon,
  BellIcon,
  MapPinIcon,
  UsersIcon,
  MapIcon
} from 'lucide-vue-next'

export default {
  name: 'SidebarComponent',
  props: {
    activeSection: {
      type: String,
      default: 'Dashboard'
    }
  },
  components: {
    HomeIcon,
    NotebookIcon,
    TagIcon,
    NewspaperIcon,
    BellIcon,
    MapPinIcon,
    UsersIcon,
    MapIcon
  },
  data() {
    return {
      navItems: [
        { section: 'Dashboard', icon: 'HomeIcon' },
        { section: 'Data Pertanian', icon: 'NotebookIcon' },
        { section: 'Data Penggilingan', icon: 'NotebookIcon' },
        { section: 'Input Harga', icon: 'TagIcon' },
        { section: 'Input Berita', icon: 'NewspaperIcon' },
        { section: 'Input Notifikasi', icon: 'BellIcon' },
        { section: 'Lokasi', icon: 'MapPinIcon' },
        { section: 'Sebaran Petani', icon: 'MapIcon' },
        { section: 'Pengguna', icon: 'UsersIcon' }
      ]
    }
  },
  methods: {
    changeSection(section) {
      this.$emit('change-section', section)
    }
  }
}
</script>

<style scoped>
/* Reset and base styling */
*,
*::before,
*::after {
  box-sizing: border-box;
}

.sidebar {
  width: 250px;
  background-color: #ffffff;
  border-right: 1px solid #e5e7eb;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  transition: all 0.3s ease;
  height: 100vh;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.sidebar-header {
  display: flex;
  align-items: center;
  justify-content: center;  /* Added this line to center the text */
  margin-bottom: 2rem;
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e5e7eb;
}

.sidebar-header h2 {
  font-size: 1.75rem;
  font-weight: 700;
  color: #111827;
  margin: 0;
  letter-spacing: -0.025em;
}

.nav-list {
  list-style: none;
  margin: 0;
  padding: 0;
  flex: 1;
}

.nav-item {
  margin-bottom: 0.25rem;
}

.nav-link {
  display: flex;
  align-items: center;
  padding: 0.875rem 1rem;
  color: #6b7280;
  text-decoration: none;
  border-radius: 0.5rem;
  transition: all 0.15s ease;
  font-weight: 500;
  font-size: 0.875rem;
  line-height: 1.25rem;
  position: relative;
}

.nav-link:hover {
  background-color: #f9fafb;
  color: #374151;
  transform: translateX(2px);
}

.nav-link.active {
  background-color: #f3f4f6;
  color: #111827;
  font-weight: 600;
  box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
}

.nav-link.active::before {
  content: '';
  position: absolute;
  left: 0;
  top: 50%;
  transform: translateY(-50%);
  width: 3px;
  height: 60%;
  background-color: #111827;
  border-radius: 0 2px 2px 0;
}

.nav-link .icon {
  width: 18px;
  height: 18px;
  margin-right: 0.75rem;
  stroke-width: 1.5;
  flex-shrink: 0;
}

/* Responsive design */
@media (max-width: 768px) {
  .sidebar {
    width: 80px;
    padding: 1.5rem 1rem;
  }

  .sidebar-header {
    justify-content: center;
    padding-bottom: 1rem;
    margin-bottom: 1.5rem;
  }

  .sidebar-header h2 {
    display: none;
  }

  .sidebar-header::after {
    content: 'P.';
    font-size: 1.5rem;
    font-weight: 700;
    color: #111827;
  }

  .nav-link {
    justify-content: center;
    padding: 0.75rem;
  }

  .nav-link span {
    display: none;
  }

  .nav-link .icon {
    margin-right: 0;
    width: 20px;
    height: 20px;
  }

  .nav-link:hover {
    transform: none;
  }

  .nav-link.active::before {
    display: none;
  }

  .nav-link.active::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 50%;
    transform: translateX(-50%);
    width: 60%;
    height: 2px;
    background-color: #111827;
    border-radius: 1px 1px 0 0;
  }
}

@media (max-width: 480px) {
  .sidebar {
    width: 70px;
    padding: 1rem 0.5rem;
  }

  .nav-link {
    padding: 0.625rem;
  }

  .nav-link .icon {
    width: 18px;
    height: 18px;
  }
}

/* Focus states for accessibility */
.nav-link:focus {
  outline: 2px solid #111827;
  outline-offset: 2px;
}

.nav-link:focus:not(:focus-visible) {
  outline: none;
}

/* Smooth scrolling for sidebar content */
.nav-list {
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: #d1d5db #ffffff;
}

.nav-list::-webkit-scrollbar {
  width: 4px;
}

.nav-list::-webkit-scrollbar-track {
  background: #ffffff;
}

.nav-list::-webkit-scrollbar-thumb {
  background: #d1d5db;
  border-radius: 2px;
}

.nav-list::-webkit-scrollbar-thumb:hover {
  background: #9ca3af;
}
</style>