<template>
  <main class="container" role="main" aria-label="Login">
    <div class="login-section">
      <div class="headline-card">
        <h1 class="headline">Admin Login</h1>
      </div>

      <div class="form-section">
        <div class="login-form">
          <form @submit.prevent="login">
            <div class="form-group">
              <label>Email:</label>
              <input 
                type="email" 
                v-model="loginForm.email" 
                required 
                placeholder="admin@gmail.com"
              />
            </div>
            <div class="form-group">
              <label>Password:</label>
              <div class="password-input-container">
                <input 
                  :type="showPassword ? 'text' : 'password'" 
                  v-model="loginForm.password" 
                  required 
                  placeholder="Password"
                />
                <button 
                  type="button" 
                  class="password-toggle"
                  @click="showPassword = !showPassword"
                  :aria-label="showPassword ? 'Hide password' : 'Show password'"
                >
                  <Eye v-if="!showPassword" :size="16" />
                  <EyeOff v-if="showPassword" :size="16" />
                </button>
              </div>
            </div>
            <button type="submit" :disabled="loading" class="login-button">
              {{ loading ? 'Login...' : 'Login' }}
            </button>
          </form>
          
          <div v-if="error" class="error-message">
            {{ error }}
          </div>
        </div>
      </div>
    </div>
  </main>
</template>

<script>
import { signInWithEmailAndPassword } from 'firebase/auth'
import { Eye, EyeOff } from 'lucide-vue-next'
import { auth } from '../firebase.js'

export default {
  name: 'LoginPage',
  components: {
    Eye,
    EyeOff
  },
  data() {
    return {
      loading: false,
      error: '',
      showPassword: false,
      loginForm: {
        email: '',
        password: ''
      }
    }
  },
  methods: {
    async login() {
      this.loading = true
      this.error = ''
      
      try {
        await signInWithEmailAndPassword(
          auth,
          this.loginForm.email,
          this.loginForm.password
        )
        
        // Check if user is admin (this will be handled in App.vue)
        this.$emit('login-success')
        
      } catch (error) {
        console.error('Login error:', error)
        this.error = this.getErrorMessage(error.code)
      } finally {
        this.loading = false
      }
    },
    
    getErrorMessage(errorCode) {
      const errorMessages = {
        'auth/email-already-in-use': 'Email sudah digunakan',
        'auth/invalid-email': 'Format email tidak valid',
        'auth/operation-not-allowed': 'Operasi tidak diizinkan',
        'auth/weak-password': 'Password terlalu lemah',
        'auth/user-disabled': 'Akun telah dinonaktifkan',
        'auth/user-not-found': 'Email tidak ditemukan',
        'auth/wrong-password': 'Password salah',
        'auth/invalid-credential': 'Email atau password salah'
      }
      return errorMessages[errorCode] || 'Terjadi kesalahan, silakan coba lagi'
    }
  }
}
</script>

<style scoped>
/* Reset and base - menggunakan style yang sama dengan Dashboard.vue */
*,
*::before,
*::after {
  box-sizing: border-box;
}

.container {
  width: 100%;
  margin: 0;
  padding: 2rem 0;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  color: #374151;
  background-color: #f5f5f5;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

.login-section {
  width: 100%;
  max-width: 400px;
  margin: 0 auto;
}

.headline-card {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
}

.headline {
  font-size: 1.75rem;
  font-weight: 600;
  margin-bottom: 0;
  color: #111827;
  text-align: center;
  padding: 0;
}

.form-section {
  background: #ffffff;
  margin: 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
}

.login-form {
  width: 100%;
}

.form-group {
  margin-bottom: 1.5rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  color: #374151;
  font-weight: 600;
  font-size: 0.875rem;
}

input {
  width: 100%;
  padding: 0.75rem 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  background: #ffffff;
  color: #111827;
  transition: border-color 0.15s ease;
}

input:focus {
  outline: none;
  border-color: #6b7280;
  box-shadow: 0 0 0 3px rgba(107, 114, 128, 0.1);
}

input::placeholder {
  color: #9ca3af;
}

.password-input-container {
  position: relative;
  display: flex;
  align-items: center;
}

.password-input-container input {
  padding-right: 3rem;
}

.password-toggle {
  position: absolute;
  right: 0.75rem;
  background: none;
  border: none;
  color: #6b7280;
  cursor: pointer;
  padding: 0.25rem;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: color 0.15s ease;
}

.password-toggle:hover {
  color: #374151;
}

.password-toggle:focus {
  outline: none;
  color: #374151;
}

.login-button {
  width: 100%;
  padding: 0.75rem 1rem;
  background: #111827;
  color: #ffffff;
  border: none;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.login-button:hover:not(:disabled) {
  background: #1f2937;
}

.login-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  background: #6b7280;
}

.error-message {
  background: #f3f4f6;
  color: #1f2937;  
  padding: 0.75rem 1rem;
  border-radius: 0.375rem;
  margin-top: 1rem;
  text-align: center;
  font-size: 0.875rem;
  border: 1px solid #d1d5db;
}

@media (max-width: 768px) {
  .container {
    padding: 1rem;
  }

  .login-section {
    max-width: 100%;
  }

  .headline-card,
  .form-section {
    padding: 1.5rem 1rem;
  }

  .headline {
    font-size: 1.5rem;
  }
}
</style>