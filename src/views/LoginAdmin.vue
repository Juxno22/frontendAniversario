<template>
  <div class="login-admin container">
    <div class="login-shell card">
      <div class="login-badge">Admin seguro</div>
      <h1>Panel de boletos</h1>
      <p class="login-sub">
        Acceso restringido para gestión de compras, inventario y validación de boletos.
      </p>

      <form class="login-form" @submit.prevent="entrar">
        <div class="form-group">
          <label for="usuario">Usuario</label>
          <input
            id="usuario"
            v-model.trim="form.usuario"
            type="text"
            autocomplete="username"
            placeholder="admin"
          />
        </div>

        <div class="form-group">
          <label for="password">Contraseña</label>
          <input
            id="password"
            v-model="form.password"
            type="password"
            autocomplete="current-password"
            placeholder="••••••••••••"
          />
        </div>

        <div v-if="error" class="alert alert-error">{{ error }}</div>

        <button class="btn btn-dorado login-btn" :disabled="cargando || !form.password">
          <span v-if="cargando">Verificando acceso…</span>
          <span v-else>Entrar al panel</span>
        </button>
      </form>
    </div>
  </div>
</template>

<script>
import { loginAdmin, setAdminToken } from '@/services/api'

export default {
  name: 'LoginAdmin',
  data () {
    return {
      form: { usuario: 'admin', password: '' },
      cargando: false,
      error: null
    }
  },
  methods: {
    async entrar () {
      this.error = null
      this.cargando = true
      try {
        const { data } = await loginAdmin(this.form)
        if (data.ok && data.token) {
          setAdminToken(data.token)
          this.$router.replace(this.$route.query.redirect || '/admin')
        } else {
          this.error = data.mensaje || 'No se pudo iniciar sesión.'
        }
      } catch (e) {
        this.error = e.response?.data?.mensaje || 'Acceso denegado. Verifica tus credenciales.'
      } finally {
        this.cargando = false
      }
    }
  }
}
</script>

<style scoped>
.login-admin {
  width: 100%;
  min-height: calc(100vh - 170px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: clamp(2.5rem, 6vw, 5rem) 1.25rem;
}

.login-shell.card {
  width: min(100%, 520px) !important;
  max-width: 520px !important;
  margin: 0 auto;
  padding: clamp(1.45rem, 3vw, 2.4rem);
  border-radius: 32px;
  background: rgba(255, 255, 255, 0.86);
  border: 1px solid rgba(0, 0, 0, 0.08);
  box-shadow: 0 28px 80px rgba(0, 0, 0, 0.08);
  backdrop-filter: blur(24px);
  overflow: hidden;
}

.login-badge {
  display: inline-flex;
  align-items: center;
  width: fit-content;
  padding: 0.36rem 0.78rem;
  border-radius: 999px;
  background: #ecfdf3;
  color: #17733a;
  font-size: 0.78rem;
  font-weight: 760;
  margin-bottom: 1rem;
}

.login-shell h1 {
  max-width: 9ch;
  font-size: clamp(2.15rem, 6vw, 3.35rem);
  line-height: 0.96;
  letter-spacing: -0.055em;
  margin-bottom: 0.8rem;
}

.login-sub {
  max-width: 420px;
  color: var(--gris);
  font-size: 0.96rem;
  line-height: 1.6;
  margin-bottom: 1.45rem;
}

.login-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.login-form .form-group input {
  min-height: 50px;
  border-radius: 18px;
}

.login-btn {
  width: 100%;
  min-height: 52px;
  margin-top: 0.25rem;
  font-weight: 800;
}

.login-security {
  display: flex;
  align-items: flex-start;
  gap: 0.75rem;
  margin-top: 1.25rem;
  padding: 0.95rem 1rem;
  border-radius: 20px;
  background: rgba(255, 255, 255, 0.72);
  border: 1px solid var(--borde);
  color: var(--gris);
  font-size: 0.86rem;
  line-height: 1.55;
}

.login-security span {
  flex: 0 0 auto;
}

.login-security p {
  min-width: 0;
}

@media (max-width: 640px) {
  .login-admin {
    min-height: calc(100vh - 135px);
    padding: 1.4rem 0.85rem 2.5rem;
    align-items: flex-start;
  }

  .login-shell.card {
    width: 100% !important;
    max-width: 100% !important;
    border-radius: 26px;
    padding: 1.25rem;
  }

  .login-shell h1 {
    max-width: 100%;
    font-size: clamp(2rem, 11vw, 2.7rem);
  }

  .login-sub {
    font-size: 0.9rem;
    margin-bottom: 1.15rem;
  }

  .login-security {
    border-radius: 18px;
    font-size: 0.82rem;
  }
}
</style>