<!-- src/components/AuthWrapper.vue -->
<template>
  <div>
    <!-- Loading Screen -->
    <div v-if="authStore.loading" class="loading-screen">
      <div class="loading-spinner"></div>
      <p>Memuat...</p>
    </div>
    
    <!-- Authentication Screen -->
    <div v-else-if="!authStore.isAuthenticated || !authStore.isAdmin">
      <Login 
        v-if="currentAuthView === 'login'"
        @login-success="handleLoginSuccess"
        @switch-to-register="currentAuthView = 'register'"
      />
      <Register 
        v-else
        @switch-to-login="currentAuthView = 'login'"
      />
    </div>
    
    <!-- Main App (Dashboard) -->
    <div v-else>
      <slot></slot>
    </div>
  </div>
</template>

<script>
import { authStore } from '../store/auth'
import Login from '../views/Login.vue'
import Register from '../views/Register.vue'

export default {
  name: 'AuthWrapper',
  components: {
    Login,
    Register
  },
  data() {
    return {
      currentAuthView: 'login'
    }
  },
  setup() {
    return {
      authStore
    }
  },
  mounted() {
    // Initialize auth state listener
    authStore.init()
  },
  methods: {
    handleLoginSuccess() {
      // Login success is handled automatically by the auth store
      console.log('Login successful')
    }
  }
}
</script>

<style scoped>
.loading-screen {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 4px solid rgba(255, 255, 255, 0.3);
  border-top: 4px solid white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>