<template>
  <div class="admin container">
    <div class="admin-header">
      <div>
        <div class="ornamento"><span>⚙</span></div>
        <h1>Panel de Administración</h1>
        <p class="admin-sub">Control de ventas, reservados, PDFs, QR y acceso.</p>
      </div>
      <button class="btn btn-outline btn-sm" @click="cerrarSesion">Cerrar sesión</button>
    </div>

    <div class="seguridad-admin card">
      <div>
        <strong>Operación protegida</strong>
        <p>Mercado Pago procesa las compras públicas. El admin solo valida boletos, consulta ventas y registra ventas
          manuales auditadas con contraseña.</p>
      </div>
      <span>🔐 Admin protegido</span>
    </div>

    <div class="tabs-admin">
      <button :class="{ active: tab === 'dashboard' }" @click="tab = 'dashboard'">Dashboard</button>
      <button :class="{ active: tab === 'mapa' }" @click="tab = 'mapa'; cargarMapaAdmin()">Mapa VIP</button>
      <button :class="{ active: tab === 'venta' }" @click="tab = 'venta'; cargarMapaAdmin()">Venta manual</button>
      <button :class="{ active: tab === 'compras' }" @click="tab = 'compras'">Compras</button>
      <button :class="{ active: tab === 'validacion' }" @click="tab = 'validacion'">Validación QR</button>
    </div>

    <section v-if="tab === 'dashboard'" class="admin-section">
      <div v-if="cargandoDashboard" class="centered">
        <div class="spinner"></div>
        <p>Cargando dashboard…</p>
      </div>
      <div v-else-if="dashboard">
        <div class="stats-fila">
          <div class="stat-item"><span class="stat-num">{{ dashboard.totales.total_compras || 0 }}</span><span
              class="stat-label">Compras</span></div>
          <div class="stat-item"><span class="stat-num">${{ formatPrecio(dashboard.totales.ingresos_aprobados || 0)
          }}</span><span class="stat-label">Ingresos aprobados</span></div>
          <div class="stat-item"><span class="stat-num">{{ dashboard.boletos.boletos_validos || 0 }}</span><span
              class="stat-label">Boletos válidos</span></div>
          <div class="stat-item"><span class="stat-num">{{ dashboard.boletos.boletos_usados || 0 }}</span><span
              class="stat-label">Boletos usados</span></div>
        </div>

        <div class="dashboard-grid">
          <div class="card">
            <h2 class="bloque-titulo">Inventario activo</h2>
            <div class="inventario-resumen" v-for="tipo in dashboard.inventario" :key="tipo.id_tipo_boleto">
              <div><strong>{{ tipo.nombre }}</strong>
                <p>${{ formatPrecio(tipo.precio) }} · Vendidos {{ tipo.cantidad_vendida }} · Apartados {{
                  tipo.cantidad_reservada }}</p>
              </div>
              <span class="pill-stock">{{ tipo.cantidad_restante }} restantes</span>
            </div>
          </div>
          <div class="card">
            <h2 class="bloque-titulo">Estados</h2>
            <div class="estado-linea" v-for="e in dashboard.por_estado" :key="e.estado">
              <span class="badge" :class="`badge-${e.estado}`">{{ etiquetaEstado(e.estado) }}</span>
              <strong>{{ e.total }}</strong>
            </div>
          </div>
        </div>
      </div>
    </section>

    <section v-if="tab === 'mapa'" class="admin-section">
      <div class="mapa-toolbar card">
        <div>
          <h2 class="bloque-titulo">Mapa de Reservados VIP</h2>
          <p>Disponible, bloqueado, apartado o vendido. Los vendidos muestran comprador y código.</p>
        </div>
        <button class="btn btn-outline btn-sm" @click="cargarMapaAdmin">Actualizar mapa</button>
      </div>
      <div v-if="cargandoMapa" class="centered">
        <div class="spinner"></div>
        <p>Cargando mapa…</p>
      </div>
      <div v-else-if="mapaAdmin.asientos.length" class="card mapa-card">
        <div class="mapa-leyenda">
          <span><i class="dot estado-disponible"></i>Disponible</span>
          <span><i class="dot estado-bloqueado"></i>Bloqueado</span>
          <span><i class="dot estado-reservado"></i>Apartado</span>
          <span><i class="dot estado-vendido"></i>Vendido</span>
        </div>
        <div class="seat-grid-admin">
          <button v-for="a in mapaAdmin.asientos" :key="a.id_asiento" class="seat-admin"
            :class="[`zona-${a.color_zona}`, `estado-${a.estado}`]" @click="seleccionarAsientoAdmin(a)">
            <span v-if="a.estado === 'vendido' || a.estado === 'bloqueado'">×</span>
            <span v-else>{{ a.numero }}</span>
          </button>
        </div>
        <div v-if="asientoAdmin" class="asiento-panel card">
          <h3>Reservado #{{ asientoAdmin.numero }}</h3>
          <p><strong>Estado:</strong> {{ asientoAdmin.estado }} · <strong>Zona:</strong> {{ asientoAdmin.zona }} ·
            <strong>Precio:</strong> ${{ formatPrecio(asientoAdmin.precio) }}
          </p>
          <p v-if="asientoAdmin.nombre_comprador"><strong>Comprador:</strong> {{ asientoAdmin.nombre_comprador }} · {{
            asientoAdmin.telefono_comprador }}</p>
          <p v-if="asientoAdmin.codigo_boleto"><strong>Código:</strong> <span class="mono">{{ asientoAdmin.codigo_boleto
          }}</span></p>
          <div v-if="['disponible', 'bloqueado'].includes(asientoAdmin.estado)" class="accion-admin-grid">
            <div class="form-group"><label>Contraseña admin</label><input v-model="asientoCambio.admin_password"
                type="password"></div>
            <div class="form-group"><label>Motivo</label><input v-model="asientoCambio.motivo"
                placeholder="Motivo del cambio"></div>
            <button class="btn btn-outline btn-sm"
              @click="cambiarEstadoAsiento(asientoAdmin.estado === 'disponible' ? 'bloqueado' : 'disponible')">
              {{ asientoAdmin.estado === 'disponible' ? 'Bloquear' : 'Liberar' }} reservado
            </button>
          </div>
        </div>
      </div>
      <div v-if="mensajeMapa" class="alert alert-success">{{ mensajeMapa }}</div>
      <div v-if="errorMapa" class="alert alert-error">{{ errorMapa }}</div>
    </section>

    <section v-if="tab === 'venta'" class="admin-section venta-grid">
      <div class="card">
        <h2 class="bloque-titulo">Venta manual auditada</h2>
        <p class="texto-muted">Úsala para ventas físicas. Se crean compra aprobada, boletos, QR y PDF. Requiere
          contraseña única del administrador.</p>
        <div class="form-grid">
          <div class="form-group"><label>Nombre completo comprador *</label><input
              v-model.trim="ventaManual.comprador.nombre"></div>
          <div class="form-group"><label>Teléfono comprador *</label><input
              v-model.trim="ventaManual.comprador.telefono" placeholder="2221234567"></div>
          <div class="form-group"><label>Boletos generales</label><input v-model.number="ventaManual.generalCantidad"
              type="number" min="0"></div>
          <div class="form-group"><label>Contraseña admin *</label><input v-model="ventaManual.admin_password"
              type="password"></div>
          <div class="form-group full"><label>Motivo / observación *</label><input v-model.trim="ventaManual.motivo"
              placeholder="Venta física registrada en taquilla"></div>
        </div>
        <button class="btn btn-rojo" :disabled="creandoVentaManual" @click="registrarVentaManual">{{ creandoVentaManual
          ? 'Registrando…' : 'Registrar venta manual' }}</button>
        <div v-if="ventaManualResultado" class="alert alert-success venta-resultado">
          Venta registrada. Compra #{{ ventaManualResultado.id_compra }} ·
          <a :href="ventaManualResultado.pdf_url" target="_blank" rel="noopener">Descargar PDF</a>
        </div>
        <div v-if="errorVentaManual" class="alert alert-error">{{ errorVentaManual }}</div>
      </div>
      <div class="card venta-mapa-card">
        <h2 class="bloque-titulo">Selecciona reservados</h2>
        <div class="mapa-leyenda"><span><i class="dot estado-disponible"></i>Disponible</span><span><i
              class="dot estado-vendido"></i>No disponible</span></div>
        <div class="seat-grid-admin compacto">
          <button v-for="a in mapaAdmin.asientos" :key="`v-${a.id_asiento}`" class="seat-admin"
            :class="[`zona-${a.color_zona}`, a.estado === 'disponible' ? (ventaManual.asientos.includes(a.numero) ? 'estado-seleccionado' : 'estado-disponible') : 'estado-vendido']"
            :disabled="a.estado !== 'disponible'" @click="toggleVentaAsiento(a.numero)">
            <span v-if="a.estado !== 'disponible'">×</span><span v-else>{{ a.numero }}</span>
          </button>
        </div>
        <p class="texto-muted">Seleccionados: {{ ventaManual.asientos.join(', ') || 'ninguno' }}</p>
      </div>
    </section>

    <section v-if="tab === 'compras'" class="admin-section">
      <div class="filtros card">
        <div class="filtros-inner">
          <div class="form-group filtro-estado"><label>Estado</label><select v-model="filtros.estado"
              @change="cargarCompras(1)">
              <option value="">Todos</option>
              <option value="pendiente">Pendiente</option>
              <option value="en_proceso">En proceso</option>
              <option value="aprobado">Aprobado</option>
              <option value="rechazado">Rechazado</option>
              <option value="cancelado">Cancelado</option>
            </select></div>
          <button class="btn btn-outline btn-sm" @click="cargarCompras(paginacion.page)">Actualizar</button>
        </div>
      </div>
      <div v-if="cargando" class="centered">
        <div class="spinner"></div>
        <p>Cargando compras…</p>
      </div>
      <div v-else-if="compras.length === 0" class="vacio card"><span>📋</span>
        <p>No hay compras con el filtro seleccionado.</p>
      </div>
      <div v-else class="tabla-wrapper">
        <table class="tabla">
          <thead>
            <tr>
              <th>#</th>
              <th>Comprador</th>
              <th>Teléfono</th>
              <th>Total</th>
              <th>Estado</th>
              <th>Boletos</th>
              <th>Origen</th>
              <th>Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="c in compras" :key="c.id_compra" @click="verDetalle(c)" class="tabla-fila">
              <td>#{{ c.id_compra }}</td>
              <td>{{ c.nombre_comprador }}</td>
              <td>{{ c.telefono_comprador || '—' }}</td>
              <td>${{ formatPrecio(c.monto_total) }}</td>
              <td><span class="badge" :class="`badge-${c.estado}`">{{ etiquetaEstado(c.estado) }}</span></td>
              <td>{{ c.boletos_usados }}/{{ c.total_boletos }}</td>
              <td>{{ c.origen || 'mercadopago' }}</td>
              <td @click.stop><button class="btn btn-sm btn-outline" @click="verDetalle(c)">Ver</button></td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <section v-if="tab === 'validacion'" class="admin-section validacion-grid">
      <div class="card">
        <h2 class="bloque-titulo">Validar boleto</h2>
        <p class="texto-muted">Escanea el QR o ingresa el código. El QR incluye token seguro; el código visible también
          puede capturarse manualmente.</p>
        <div class="form-group"><label>Código o QR</label><input v-model.trim="codigoBuscar" @keyup.enter="buscarCodigo"
            placeholder="JARIPEO-XXXX o contenido QR"></div>
        <button class="btn btn-dorado" :disabled="buscandoCodigo" @click="buscarCodigo">{{ buscandoCodigo ? 'Buscando…'
          : 'Buscar boleto' }}</button>
        <div v-if="errorCodigo" class="alert alert-error">{{ errorCodigo }}</div>
      </div>
      <div v-if="boletoEncontrado" class="card boleto-resultado">
        <h2 class="bloque-titulo">Resultado</h2>
        <div class="info-row"><span>Código</span><strong class="mono">{{ boletoEncontrado.codigo_boleto }}</strong>
        </div>
        <div class="info-row"><span>Tipo</span><strong>{{ boletoEncontrado.tipo_boleto }}</strong></div>
        <div class="info-row"><span>Titular</span><strong>{{ boletoEncontrado.titular_nombre ||
          boletoEncontrado.nombre_comprador }}</strong></div>
        <div class="info-row"><span>Teléfono</span><strong>{{ boletoEncontrado.titular_telefono ||
          boletoEncontrado.telefono_comprador }}</strong></div>
        <div class="info-row" v-if="boletoEncontrado.numero_asiento"><span>Reservado</span><strong>#{{
          boletoEncontrado.numero_asiento }} · {{ boletoEncontrado.personas_incluidas }} personas</strong></div>
        <div class="info-row"><span>Compra</span><span class="badge"
            :class="`badge-${boletoEncontrado.estado_compra}`">{{ etiquetaEstado(boletoEncontrado.estado_compra)
            }}</span></div>
        <div class="info-row"><span>Estado boleto</span><span class="badge"
            :class="boletoEncontrado.usado ? 'badge-cancelado' : 'badge-aprobado'">{{ boletoEncontrado.usado ? 'Usado' :
              'Válido' }}</span></div>
        <p class="texto-alerta">Solicita identificación oficial del comprador/titular antes de permitir el acceso.</p>
        <button v-if="!boletoEncontrado.usado" class="btn btn-rojo" style="width:100%;margin-top:1rem"
          @click="usarBoleto(boletoEncontrado.id_boleto)" :disabled="procesandoUso">{{ procesandoUso ? 'Procesando…' :
            'Marcar acceso usado' }}</button>
      </div>
    </section>

    <div v-if="compraSeleccionada" class="modal-overlay" @click.self="compraSeleccionada = null">
      <div class="modal modal-grande card">
        <div class="modal-head">
          <h2>Compra #{{ compraSeleccionada.id_compra }}</h2><button class="btn btn-outline btn-sm"
            @click="compraSeleccionada = null">Cerrar</button>
        </div>
        <div v-if="cargandoDetalle" class="spinner"></div>
        <div v-else-if="detalleCompra">
          <div class="detalle-seccion">
            <h3>Comprador</h3>
            <div class="info-row"><span>Nombre</span><strong>{{ detalleCompra.compra.nombre_comprador }}</strong></div>
            <div class="info-row"><span>Teléfono</span><strong>{{ detalleCompra.compra.telefono_comprador || '—'
            }}</strong></div>
            <div class="info-row"><span>Total</span><strong>${{ formatPrecio(detalleCompra.compra.monto_total)
            }}</strong></div>
          </div>
          <div class="detalle-seccion">
            <h3>Boletos</h3>
            <div class="boleto-fila" v-for="b in detalleCompra.boletos" :key="b.id_boleto"><span class="mono">{{
              b.codigo_boleto }}</span><span v-if="b.numero_asiento">Reservado #{{ b.numero_asiento }}</span><span
                class="badge" :class="b.usado ? 'badge-cancelado' : 'badge-aprobado'">{{ b.usado ? 'Usado' : 'Válido'
                }}</span><button v-if="!b.usado" class="btn btn-sm btn-rojo" @click="usarBoletoEnDetalle(b)">Marcar
                usado</button></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import {
  clearAdminToken,
  listarCompras,
  obtenerCompra,
  marcarBoletoUsado,
  buscarPorCodigo,
  obtenerDashboardAdmin,
  obtenerInventarioAdmin,
  obtenerMapaReservadosAdmin,
  actualizarAsientoAdmin,
  crearVentaManualAdmin
} from '@/services/api'

export default {
  name: 'PanelAdmin',
  data() {
    return {
      tab: 'dashboard', dashboard: null, cargandoDashboard: false,
      inventario: null, tiposBoleto: [], compras: [], paginacion: { page: 1, limit: 20, total: 0 }, filtros: { estado: '' }, cargando: false,
      compraSeleccionada: null, detalleCompra: null, cargandoDetalle: false,
      codigoBuscar: '', tokenBuscar: '', boletoEncontrado: null, buscandoCodigo: false, errorCodigo: null, procesandoUso: false,
      mapaAdmin: { tipo_boleto: null, asientos: [] }, cargandoMapa: false, asientoAdmin: null, asientoCambio: { admin_password: '', motivo: '' }, mensajeMapa: null, errorMapa: null,
      ventaManual: { comprador: { nombre: '', telefono: '' }, generalCantidad: 0, asientos: [], motivo: '', admin_password: '' }, creandoVentaManual: false, ventaManualResultado: null, errorVentaManual: null
    }
  },
  async created() {
    if (this.$route.query.codigo) { this.tab = 'validacion'; this.codigoBuscar = this.$route.query.codigo; this.tokenBuscar = this.$route.query.token || ''; await this.buscarCodigo() }
    await Promise.all([this.cargarDashboard(), this.cargarInventario(), this.cargarCompras(1), this.cargarMapaAdmin()])
  },
  methods: {
    esTipoReservado(tipo) { const nombre = String(tipo?.nombre || '').toLowerCase(); return Number(tipo?.requiere_asientos || 0) === 1 && (nombre.includes('reservado') || nombre.includes('vip')) },
    esTipoGeneral(tipo) { const nombre = String(tipo?.nombre || '').toLowerCase(); return Number(tipo?.requiere_asientos || 0) === 0 && nombre.includes('general') },
    tipoPermitido(tipo) { return this.esTipoGeneral(tipo) || this.esTipoReservado(tipo) },
    cerrarSesion() { clearAdminToken(); this.$router.replace('/admin/login') },
    async manejarNoAutorizado(e) { if (e.response?.status === 401 || e.response?.status === 403) { clearAdminToken(); this.$router.replace('/admin/login'); return true } return false },
    async cargarDashboard() { this.cargandoDashboard = true; try { const { data } = await obtenerDashboardAdmin(); if (data.ok) this.dashboard = data } catch (e) { await this.manejarNoAutorizado(e) } finally { this.cargandoDashboard = false } },
    async cargarInventario() { try { const { data } = await obtenerInventarioAdmin(); if (data.ok) { this.inventario = data; this.tiposBoleto = (data.tipos_boleto || []).filter(this.tipoPermitido) } } catch (e) { await this.manejarNoAutorizado(e) } },
    async cargarMapaAdmin() { this.cargandoMapa = true; this.errorMapa = null; try { const { data } = await obtenerMapaReservadosAdmin(); if (data.ok) this.mapaAdmin = data } catch (e) { if (!(await this.manejarNoAutorizado(e))) this.errorMapa = e.response?.data?.mensaje || 'No se pudo cargar el mapa.' } finally { this.cargandoMapa = false } },
    seleccionarAsientoAdmin(a) { this.asientoAdmin = a; this.asientoCambio = { admin_password: '', motivo: '' } },
    async cambiarEstadoAsiento(estado) { if (!this.asientoAdmin) return; this.mensajeMapa = null; this.errorMapa = null; try { const { data } = await actualizarAsientoAdmin(this.asientoAdmin.id_asiento, { estado, ...this.asientoCambio }); if (data.ok) { this.mensajeMapa = data.mensaje; await this.cargarMapaAdmin(); this.asientoAdmin = null } } catch (e) { this.errorMapa = e.response?.data?.mensaje || 'No se pudo actualizar el reservado.' } },
    toggleVentaAsiento(numero) { const n = Number(numero); this.ventaManual.asientos = this.ventaManual.asientos.includes(n) ? this.ventaManual.asientos.filter(x => x !== n) : [...this.ventaManual.asientos, n].sort((a, b) => a - b) },
    async registrarVentaManual() { this.creandoVentaManual = true; this.errorVentaManual = null; this.ventaManualResultado = null; try { const reservado = this.mapaAdmin.tipo_boleto; const general = this.tiposBoleto.find(t => !Number(t.requiere_asientos) && String(t.nombre).toLowerCase().includes('general')); const boletos = []; if (general && Number(this.ventaManual.generalCantidad) > 0) boletos.push({ id_tipo_boleto: general.id_tipo_boleto, cantidad: Number(this.ventaManual.generalCantidad) }); if (reservado && this.ventaManual.asientos.length) boletos.push({ id_tipo_boleto: reservado.id_tipo_boleto, cantidad: this.ventaManual.asientos.length, asientos: this.ventaManual.asientos }); const { data } = await crearVentaManualAdmin({ comprador: this.ventaManual.comprador, boletos, motivo: this.ventaManual.motivo, admin_password: this.ventaManual.admin_password }); if (data.ok) { this.ventaManualResultado = data; this.ventaManual = { comprador: { nombre: '', telefono: '' }, generalCantidad: 0, asientos: [], motivo: '', admin_password: '' }; await Promise.all([this.cargarDashboard(), this.cargarCompras(1), this.cargarMapaAdmin()]) } } catch (e) { this.errorVentaManual = e.response?.data?.mensaje || 'No se pudo registrar la venta manual.' } finally { this.creandoVentaManual = false } },
    async cargarCompras(page = 1) { this.cargando = true; try { const params = { page, limit: 20 }; if (this.filtros.estado) params.estado = this.filtros.estado; const { data } = await listarCompras(params); if (data.ok) { this.compras = data.compras; this.paginacion = data.paginacion } } catch (e) { await this.manejarNoAutorizado(e) } finally { this.cargando = false } },
    async verDetalle(compra) { this.compraSeleccionada = compra; this.detalleCompra = null; this.cargandoDetalle = true; try { const { data } = await obtenerCompra(compra.id_compra); if (data.ok) this.detalleCompra = data } catch (e) { await this.manejarNoAutorizado(e) } finally { this.cargandoDetalle = false } },
    parseCodigoEntrada() { const raw = this.codigoBuscar || ''; try { const url = new URL(raw); return { codigo: url.searchParams.get('codigo') || url.searchParams.get('c') || raw, token: url.searchParams.get('token') || url.searchParams.get('t') || this.tokenBuscar } } catch (e) { /* no era URL */ } if (raw.includes('|')) { const parts = raw.split('|').map(p => p.trim()); return { codigo: parts.find(p => /^JARIPEO-/i.test(p)) || raw, token: parts.find(p => /^[a-f0-9]{40,}$/i.test(p)) || this.tokenBuscar } } return { codigo: raw, token: this.tokenBuscar } },
    async buscarCodigo() { if (!this.codigoBuscar) return; this.buscandoCodigo = true; this.errorCodigo = null; this.boletoEncontrado = null; try { const p = this.parseCodigoEntrada(); const { data } = await buscarPorCodigo(p.codigo, p.token); if (data.ok) this.boletoEncontrado = data.boleto } catch (e) { this.errorCodigo = e.response?.data?.mensaje || 'Boleto no encontrado.' } finally { this.buscandoCodigo = false } },
    async usarBoleto(id) { this.procesandoUso = true; try { const { data } = await marcarBoletoUsado(id); if (data.ok) { this.boletoEncontrado = data.boleto; await Promise.all([this.cargarCompras(this.paginacion.page), this.cargarDashboard(), this.cargarMapaAdmin()]) } else this.errorCodigo = data.mensaje } catch (e) { this.errorCodigo = e.response?.data?.mensaje || 'Error al marcar el boleto.' } finally { this.procesandoUso = false } },
    async usarBoletoEnDetalle(boleto) { await this.usarBoleto(boleto.id_boleto); if (this.compraSeleccionada) await this.verDetalle(this.compraSeleccionada) },
    etiquetaEstado(estado) { const mapa = { pendiente: 'Pendiente', en_proceso: 'En proceso', aprobado: 'Aprobado', rechazado: 'Rechazado', cancelado: 'Cancelado' }; return mapa[estado] || estado },
    formatPrecio(n) { return Number(n || 0).toLocaleString('es-MX', { minimumFractionDigits: 2 }) }
  }
}
</script>

<style scoped>
.admin {
  padding: 4rem 1.25rem 5rem;
}

.admin-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.admin-header h1 {
  font-size: clamp(2rem, 5vw, 4rem);
  margin-bottom: 0.55rem;
}

.admin-sub,
.texto-muted {
  color: var(--gris);
  font-size: 0.92rem;
  line-height: 1.5;
}

.seguridad-admin {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  align-items: center;
  margin-bottom: 1rem;
  background: linear-gradient(135deg, rgba(236, 253, 243, 0.9), rgba(255, 255, 255, 0.72));
}

.seguridad-admin strong {
  color: #17733a;
}

.seguridad-admin p {
  color: var(--gris);
  font-size: 0.9rem;
  margin-top: 0.2rem;
}

.seguridad-admin span {
  flex-shrink: 0;
  color: #17733a;
  font-weight: 760;
}

.tabs-admin {
  display: flex;
  gap: 0.4rem;
  padding: 0.35rem;
  border: 1px solid var(--borde);
  background: rgba(255, 255, 255, 0.62);
  border-radius: 999px;
  margin-bottom: 1.4rem;
  overflow-x: auto;
}

.tabs-admin button {
  border: 0;
  background: transparent;
  padding: 0.62rem 1rem;
  border-radius: 999px;
  color: var(--gris);
  font-weight: 700;
  cursor: pointer;
  white-space: nowrap;
}

.tabs-admin button.active {
  background: #fff;
  color: var(--crema);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.06);
}

.admin-section {
  margin-top: 1rem;
}

.bloque-titulo {
  font-size: 1.15rem;
  margin-bottom: 1rem;
}

.stats-fila {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 0.85rem;
  margin-bottom: 1rem;
}

.stat-item {
  display: flex;
  flex-direction: column;
  gap: 0.1rem;
  padding: 1.1rem;
  border: 1px solid var(--borde);
  border-radius: 22px;
  background: rgba(255, 255, 255, 0.72);
}

.stat-num {
  font-family: var(--fuente-display);
  font-size: 1.75rem;
  color: var(--crema);
  letter-spacing: -0.06em;
}

.stat-label {
  font-size: 0.82rem;
  color: var(--gris);
}

.dashboard-grid,
.venta-grid {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(360px, 0.9fr);
  gap: 1rem;
  align-items: start;
}

.inventario-resumen,
.estado-linea,
.info-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
  padding: 0.75rem 0;
  border-bottom: 1px solid var(--borde);
}

.inventario-resumen p {
  color: var(--gris);
  font-size: 0.84rem;
  margin-top: 0.15rem;
}

.pill-stock {
  padding: 0.32rem 0.72rem;
  border-radius: 999px;
  background: #ecfdf3;
  color: #17733a;
  font-weight: 760;
  font-size: 0.8rem;
  white-space: nowrap;
}

.form-grid,
.accion-admin-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 1rem;
  margin-bottom: 1rem;
}

.full {
  grid-column: 1 / -1;
}

.btn-sm {
  min-height: 38px;
  padding: 0.56rem 1rem;
  font-size: 0.84rem;
}

.mapa-toolbar {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  align-items: center;
  margin-bottom: 1rem;
}

.mapa-card {
  overflow-x: auto;
}

.mapa-leyenda {
  display: flex;
  flex-wrap: wrap;
  gap: 0.65rem;
  margin-bottom: 1rem;
  color: var(--gris);
  font-weight: 700;
  font-size: 0.82rem;
}

.mapa-leyenda span {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
}

.dot {
  width: 12px;
  height: 12px;
  display: inline-block;
  border-radius: 999px;
  border: 1px solid rgba(0, 0, 0, .2);
}

.estado-disponible {
  background: #fff;
}

.estado-bloqueado {
  background: #ffc7cd;
}

.estado-reservado {
  background: #fff0b7;
}

.estado-vendido {
  background: #d1d1d6;
}

.seat-grid-admin {
  min-width: 860px;
  display: grid;
  grid-template-columns: repeat(20, 1fr);
  gap: 0.45rem;
}

.seat-grid-admin.compacto {
  min-width: 620px;
  gap: 0.35rem;
}

.seat-admin {
  height: 38px;
  border-radius: 999px;
  border: 1px solid rgba(0, 0, 0, .12);
  background: #fff;
  font-weight: 800;
  cursor: pointer;
  color: #1d1d1f;
}

.seat-admin.zona-rojo {
  border-color: rgba(255, 25, 47, .35);
}

.seat-admin.zona-amarillo {
  border-color: rgba(178, 128, 0, .35);
}

.seat-admin.zona-verde {
  border-color: rgba(24, 128, 56, .35);
}

.seat-admin.estado-vendido,
.seat-admin.estado-bloqueado {
  background: #eeeeef;
  color: #b40018;
  cursor: pointer;
}

.seat-admin.estado-reservado {
  background: #fff7dc;
  color: #926200;
}

.seat-admin.estado-seleccionado {
  background: var(--rojo);
  color: #fff;
}

.asiento-panel {
  margin-top: 1rem;
  box-shadow: none;
}

.filtros {
  padding: 1.25rem;
  margin-bottom: 1.5rem;
}

.filtros-inner {
  display: flex;
  align-items: flex-end;
  gap: 1rem;
  flex-wrap: wrap;
}

.filtro-estado {
  min-width: 190px;
}

.tabla-wrapper {
  overflow-x: auto;
  border-radius: var(--radio-lg);
  border: 1px solid var(--borde);
  background: rgba(255, 255, 255, 0.76);
  box-shadow: var(--sombra);
}

.tabla {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9rem;
}

.tabla th,
.tabla td {
  padding: 0.85rem;
  text-align: left;
  border-bottom: 1px solid var(--borde);
  white-space: nowrap;
}

.tabla th {
  color: var(--gris);
  font-size: 0.78rem;
}

.tabla-fila {
  cursor: pointer;
}

.tabla-fila:hover {
  background: rgba(255, 25, 47, 0.035);
}

.validacion-grid {
  display: grid;
  grid-template-columns: minmax(0, 0.9fr) minmax(360px, 1fr);
  gap: 1rem;
  align-items: start;
}

.texto-alerta {
  margin-top: 1rem;
  padding: .8rem;
  background: #fff0f1;
  border-radius: 14px;
  color: #b40018;
  font-weight: 700;
}

.mono {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
}

.venta-resultado {
  margin-top: 1rem;
}

.boleto-fila {
  display: flex;
  gap: 0.7rem;
  flex-wrap: wrap;
  align-items: center;
  padding: 0.65rem 0;
  border-bottom: 1px solid var(--borde);
}

.modal-overlay {
  position: fixed;
  inset: 0;
  z-index: 500;
  background: rgba(245, 245, 247, .76);
  backdrop-filter: blur(22px);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
}

.modal {
  width: min(100%, 760px);
  max-height: 88vh;
  overflow: auto;
}

.modal-head {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  align-items: center;
  margin-bottom: 1rem;
}

@media (max-width: 900px) {

  .stats-fila,
  .dashboard-grid,
  .venta-grid,
  .validacion-grid {
    grid-template-columns: 1fr;
  }

  .admin-header,
  .seguridad-admin,
  .mapa-toolbar {
    flex-direction: column;
    align-items: flex-start;
  }
}


/* Responsive final: admin */
@media (max-width: 720px) {
  .admin {
    padding: 2rem 0.75rem 3.5rem;
  }

  .admin-header h1 {
    font-size: clamp(2rem, 12vw, 3rem);
  }

  .tabs-admin {
    border-radius: 22px;
    padding: 0.35rem;
    scrollbar-width: none;
  }

  .tabs-admin::-webkit-scrollbar {
    display: none;
  }

  .tabs-admin button {
    flex: 0 0 auto;
    padding: 0.65rem 0.9rem;
  }

  .stats-fila {
    grid-template-columns: 1fr !important;
  }

  .form-grid,
  .accion-admin-grid {
    grid-template-columns: 1fr !important;
  }

  .inventario-resumen,
  .estado-linea,
  .info-row,
  .modal-head,
  .filtros-inner {
    flex-direction: column;
    align-items: flex-start;
  }

  .filtro-estado,
  .filtros-inner .btn,
  .venta-grid .btn,
  .accion-admin-grid .btn {
    width: 100%;
  }

  .mapa-card {
    padding: 0.85rem;
    overflow-x: auto;
  }

  .seat-grid-admin {
    min-width: 720px;
    grid-template-columns: repeat(20, 34px);
    gap: 0.34rem;
  }

  .seat-grid-admin.compacto {
    min-width: 700px;
  }

  .seat-admin {
    height: 34px;
    min-width: 34px;
    font-size: 0.76rem;
  }

  .tabla {
    min-width: 860px;
  }

  .modal-overlay {
    align-items: flex-end;
    padding: 0;
  }

  .modal {
    width: 100%;
    max-height: 92vh;
    border-radius: 24px 24px 0 0;
  }
}

@media (max-width: 420px) {
  .admin {
    padding-left: 0.65rem;
    padding-right: 0.65rem;
  }

  .seat-grid-admin,
  .seat-grid-admin.compacto {
    min-width: 660px;
    grid-template-columns: repeat(20, 31px);
  }

  .seat-admin {
    height: 31px;
    min-width: 31px;
    font-size: 0.7rem;
  }
}


/* =====================================================
   Fix real de responsive móvil para panel admin
   ===================================================== */
.admin,
.admin-section,
.admin-header,
.seguridad-admin,
.tabs-admin,
.stats-fila,
.dashboard-grid,
.venta-grid,
.validacion-grid,
.form-grid,
.accion-admin-grid,
.mapa-toolbar,
.mapa-card,
.venta-mapa-card,
.tabla-wrapper,
.modal,
.filtros,
.asiento-panel {
  min-width: 0;
  max-width: 100%;
}

.admin .card {
  min-width: 0;
  max-width: 100%;
}

.mapa-card,
.venta-mapa-card,
.tabla-wrapper {
  overflow-x: auto;
  overflow-y: hidden;
  overscroll-behavior-x: contain;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: thin;
}

.seat-grid-admin,
.seat-grid-admin.compacto {
  width: max-content;
  max-width: none;
}

@media (max-width: 720px) {
  .admin {
    width: 100%;
    max-width: 100%;
    padding-left: 0.65rem;
    padding-right: 0.65rem;
  }

  .admin-header {
    gap: 0.8rem;
    margin-bottom: 1rem;
  }

  .admin-header h1 {
    font-size: clamp(1.75rem, 10vw, 2.45rem);
  }

  .admin-sub,
  .texto-muted {
    font-size: 0.84rem;
  }

  .seguridad-admin {
    padding: 0.9rem;
    border-radius: 18px;
  }

  .tabs-admin {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 0.35rem;
    border-radius: 22px;
    overflow: visible;
  }

  .tabs-admin button {
    width: 100%;
    min-height: 42px;
    padding: 0.58rem 0.55rem;
    white-space: normal;
    line-height: 1.1;
    font-size: 0.82rem;
  }

  .tabs-admin button:last-child {
    grid-column: 1 / -1;
  }

  .admin-section {
    margin-top: 0.7rem;
  }

  .bloque-titulo {
    font-size: 1.02rem;
    margin-bottom: 0.75rem;
  }

  .mapa-toolbar {
    padding: 0.9rem;
  }

  .mapa-toolbar .btn,
  .admin-header .btn,
  .seguridad-admin span,
  .venta-grid>.card>.btn {
    width: 100%;
  }

  .form-grid,
  .accion-admin-grid {
    grid-template-columns: 1fr !important;
    gap: 0.75rem;
  }

  .venta-grid {
    display: flex;
    flex-direction: column;
    gap: 0.9rem;
  }

  .seat-grid-admin {
    grid-template-columns: repeat(20, 31px) !important;
    gap: 0.28rem;
  }

  .seat-grid-admin.compacto {
    grid-template-columns: repeat(20, 30px) !important;
    gap: 0.26rem;
  }

  .seat-admin {
    width: 31px;
    min-width: 31px;
    height: 31px;
    font-size: 0.68rem;
  }

  .seat-grid-admin.compacto .seat-admin {
    width: 30px;
    min-width: 30px;
    height: 30px;
    font-size: 0.66rem;
  }

  .mapa-leyenda {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.5rem;
    font-size: 0.76rem;
  }

  .tabla-wrapper {
    border-radius: 20px;
  }

  .tabla {
    width: max-content;
    min-width: 780px;
  }

  .modal {
    width: 100%;
    max-width: 100%;
    padding: 1rem;
  }
}

@media (max-width: 380px) {
  .admin {
    padding-left: 0.5rem;
    padding-right: 0.5rem;
  }

  .seat-grid-admin,
  .seat-grid-admin.compacto {
    grid-template-columns: repeat(20, 28px) !important;
    gap: 0.24rem;
  }

  .seat-admin,
  .seat-grid-admin.compacto .seat-admin {
    width: 28px;
    min-width: 28px;
    height: 28px;
    font-size: 0.62rem;
  }
}

@media (min-width: 901px) {
  .admin.container {
    width: 100% !important;
    max-width: 1180px !important;
    margin: 0 auto !important;
    padding: 4rem 1.25rem 5rem !important;
  }

  .admin-header {
    display: flex !important;
    justify-content: space-between !important;
    align-items: flex-start !important;
    gap: 1.5rem !important;
    margin-bottom: 1.5rem !important;
  }

  .admin-header > div {
    max-width: 760px;
  }

  .admin-header .ornamento {
    justify-content: flex-start;
    margin: 0 0 1rem;
  }

  .admin-header h1 {
    font-size: clamp(2.6rem, 4vw, 4rem) !important;
    line-height: 1.02 !important;
    letter-spacing: -0.055em !important;
    margin-bottom: 0.65rem !important;
  }

  .admin-sub {
    max-width: 720px;
    font-size: 1rem !important;
    line-height: 1.6 !important;
    color: var(--gris) !important;
  }

  .admin-header .btn {
    width: auto !important;
    flex-shrink: 0 !important;
  }

  .seguridad-admin.card {
    display: flex !important;
    flex-direction: row !important;
    justify-content: space-between !important;
    align-items: center !important;
    gap: 1.5rem !important;
    margin-bottom: 1.25rem !important;
    padding: 1.25rem 1.35rem !important;
    border-radius: 24px !important;
    background:
      linear-gradient(135deg, rgba(236, 253, 243, 0.94), rgba(255, 255, 255, 0.78)) !important;
    border: 1px solid rgba(23, 115, 58, 0.16) !important;
    box-shadow: 0 18px 48px rgba(0, 0, 0, 0.06) !important;
  }

  .seguridad-admin strong {
    color: #17733a !important;
    font-size: 1rem !important;
    letter-spacing: -0.02em !important;
  }

  .seguridad-admin p {
    margin-top: 0.2rem !important;
    color: var(--gris) !important;
    font-size: 0.9rem !important;
    line-height: 1.55 !important;
  }

  .seguridad-admin span {
    width: auto !important;
    flex-shrink: 0 !important;
    padding: 0.42rem 0.85rem !important;
    border-radius: 999px !important;
    background: #ffffff !important;
    border: 1px solid rgba(23, 115, 58, 0.15) !important;
    color: #17733a !important;
    font-size: 0.82rem !important;
    font-weight: 760 !important;
    white-space: nowrap !important;
  }

  .tabs-admin {
    display: inline-flex !important;
    width: auto !important;
    max-width: 100% !important;
    gap: 0.4rem !important;
    padding: 0.35rem !important;
    margin: 0 0 1.5rem !important;
    border-radius: 999px !important;
    border: 1px solid var(--borde) !important;
    background: rgba(255, 255, 255, 0.72) !important;
    box-shadow: 0 12px 34px rgba(0, 0, 0, 0.055) !important;
    overflow-x: auto !important;
    overflow-y: hidden !important;
  }

  .tabs-admin button {
    width: auto !important;
    flex: 0 0 auto !important;
    min-height: 42px !important;
    padding: 0.66rem 1.1rem !important;
    border-radius: 999px !important;
    white-space: nowrap !important;
    font-size: 0.88rem !important;
    line-height: 1 !important;
    color: var(--gris) !important;
  }

  .tabs-admin button.active {
    background: #ffffff !important;
    color: var(--crema) !important;
    box-shadow: 0 10px 24px rgba(0, 0, 0, 0.075) !important;
  }

  .admin-section {
    margin-top: 1rem !important;
  }

  .admin-section > .card,
  .dashboard-grid > .card,
  .venta-grid > .card,
  .validacion-grid > .card,
  .mapa-toolbar.card,
  .mapa-card.card,
  .filtros.card,
  .vacio.card,
  .asiento-panel.card {
    border-radius: var(--radio-lg) !important;
    padding: 1.5rem !important;
    background: rgba(255, 255, 255, 0.82) !important;
    border: 1px solid var(--borde) !important;
    box-shadow: var(--sombra) !important;
    backdrop-filter: blur(22px) !important;
  }

  .bloque-titulo {
    font-size: 1.18rem !important;
    line-height: 1.1 !important;
    letter-spacing: -0.035em !important;
    margin-bottom: 1rem !important;
  }

  .stats-fila {
    display: grid !important;
    grid-template-columns: repeat(4, minmax(0, 1fr)) !important;
    gap: 0.9rem !important;
    margin-bottom: 1.1rem !important;
  }

  .stat-item {
    min-width: 0 !important;
    padding: 1.15rem !important;
    border-radius: 24px !important;
    background:
      linear-gradient(145deg, rgba(255, 255, 255, 0.92), rgba(250, 250, 252, 0.76)) !important;
    border: 1px solid var(--borde) !important;
    box-shadow: 0 14px 36px rgba(0, 0, 0, 0.055) !important;
  }

  .stat-num {
    display: block !important;
    font-size: clamp(1.45rem, 2.2vw, 2rem) !important;
    line-height: 1.05 !important;
    color: var(--crema) !important;
    letter-spacing: -0.06em !important;
  }

  .stat-label {
    margin-top: 0.25rem !important;
    color: var(--gris) !important;
    font-size: 0.82rem !important;
    font-weight: 650 !important;
  }

  .dashboard-grid {
    display: grid !important;
    grid-template-columns: minmax(0, 1.2fr) minmax(320px, 0.8fr) !important;
    gap: 1rem !important;
    align-items: start !important;
  }

  .venta-grid {
    display: grid !important;
    grid-template-columns: minmax(0, 0.92fr) minmax(420px, 1.08fr) !important;
    gap: 1rem !important;
    align-items: start !important;
  }

  .validacion-grid {
    display: grid !important;
    grid-template-columns: minmax(0, 0.9fr) minmax(380px, 1fr) !important;
    gap: 1rem !important;
    align-items: start !important;
  }

  .mapa-toolbar.card {
    display: flex !important;
    flex-direction: row !important;
    justify-content: space-between !important;
    align-items: center !important;
    gap: 1rem !important;
    margin-bottom: 1rem !important;
  }

  .mapa-toolbar .btn {
    width: auto !important;
    flex-shrink: 0 !important;
    white-space: nowrap !important;
  }

  .mapa-card,
  .venta-mapa-card {
    overflow-x: auto !important;
    overflow-y: hidden !important;
    overscroll-behavior-x: contain !important;
    -webkit-overflow-scrolling: touch !important;
  }

  .mapa-leyenda {
    display: flex !important;
    flex-wrap: wrap !important;
    align-items: center !important;
    gap: 0.75rem !important;
    margin-bottom: 1rem !important;
    font-size: 0.82rem !important;
  }

  .seat-grid-admin {
    width: max-content !important;
    min-width: 860px !important;
    max-width: none !important;
    display: grid !important;
    grid-template-columns: repeat(20, 38px) !important;
    grid-auto-rows: 38px !important;
    gap: 0.45rem !important;
  }

  .seat-grid-admin.compacto {
    min-width: 760px !important;
    grid-template-columns: repeat(20, 34px) !important;
    grid-auto-rows: 34px !important;
    gap: 0.35rem !important;
  }

  .seat-admin {
    width: 38px !important;
    min-width: 38px !important;
    height: 38px !important;
    border-radius: 999px !important;
    font-size: 0.78rem !important;
    font-weight: 800 !important;
    box-shadow: 0 7px 16px rgba(0, 0, 0, 0.055) !important;
  }

  .seat-grid-admin.compacto .seat-admin {
    width: 34px !important;
    min-width: 34px !important;
    height: 34px !important;
    font-size: 0.72rem !important;
  }

  .asiento-panel.card {
    margin-top: 1rem !important;
    box-shadow: none !important;
    background: rgba(255, 255, 255, 0.7) !important;
  }

  .form-grid,
  .accion-admin-grid {
    display: grid !important;
    grid-template-columns: repeat(2, minmax(0, 1fr)) !important;
    gap: 1rem !important;
    margin-bottom: 1rem !important;
  }

  .form-grid .full,
  .accion-admin-grid .full {
    grid-column: 1 / -1 !important;
  }

  .venta-grid > .card > .btn {
    width: auto !important;
    min-width: 240px !important;
  }

  .filtros.card {
    margin-bottom: 1.2rem !important;
  }

  .filtros-inner {
    display: flex !important;
    flex-direction: row !important;
    align-items: flex-end !important;
    gap: 1rem !important;
    flex-wrap: wrap !important;
  }

  .filtro-estado {
    min-width: 220px !important;
  }

  .filtros-inner .btn {
    width: auto !important;
  }

  .tabla-wrapper {
    width: 100% !important;
    max-width: 100% !important;
    overflow-x: auto !important;
    border-radius: var(--radio-lg) !important;
    border: 1px solid var(--borde) !important;
    background: rgba(255, 255, 255, 0.78) !important;
    box-shadow: var(--sombra) !important;
  }

  .tabla {
    width: 100% !important;
    min-width: 900px !important;
    border-collapse: collapse !important;
    font-size: 0.9rem !important;
  }

  .tabla th,
  .tabla td {
    padding: 0.9rem 0.85rem !important;
    white-space: nowrap !important;
  }

  .tabla th {
    font-size: 0.78rem !important;
    color: var(--gris) !important;
    font-weight: 760 !important;
  }

  .tabla-fila:hover {
    background: rgba(255, 25, 47, 0.04) !important;
  }

  .inventario-resumen,
  .estado-linea,
  .info-row {
    display: flex !important;
    flex-direction: row !important;
    justify-content: space-between !important;
    align-items: center !important;
    gap: 1rem !important;
  }

  .pill-stock {
    flex-shrink: 0 !important;
    white-space: nowrap !important;
  }

  .boleto-resultado .btn,
  .validacion-grid .btn {
    width: auto !important;
  }

  .modal-overlay {
    align-items: center !important;
    justify-content: center !important;
    padding: 1rem !important;
  }

  .modal.card,
  .modal-grande.card {
    width: min(100%, 760px) !important;
    max-height: 88vh !important;
    overflow: auto !important;
    border-radius: var(--radio-lg) !important;
    padding: 1.5rem !important;
  }

  .modal-head {
    display: flex !important;
    flex-direction: row !important;
    justify-content: space-between !important;
    align-items: center !important;
    gap: 1rem !important;
  }

  .modal-head .btn {
    width: auto !important;
  }
}
</style>
