<script setup>
import { ref, computed, onMounted } from 'vue'
import { catalogoService, asientoService } from '@/services/erpApi'

const asientos = ref([])
const listaCuentas = ref([])
const cargando = ref(false)
const registrando = ref(false)
const error = ref(null)
const exitoMensaje = ref(null)

const fecha = ref(new Date().toISOString().substr(0, 10))
const concepto = ref('')
const partidas = ref([
  { cuenta_id: null, debe: 0, haber: 0 },
  { cuenta_id: null, debe: 0, haber: 0 }
])

// ============ CARGA DE DATOS INICIAL ============
async function cargarCatalogosYAsientos() {
  cargando.value = true
  error.value = null
  try {
    const [resCuentas, resAsientos] = await Promise.all([
      catalogoService.getAll(),
      asientoService.getAll()
    ])
    
    listaCuentas.value = (resCuentas.data?.datos || []).map(c => ({
      id: c.id,
      codigo: c.codigo,
      nombre: c.nombre,
      tipo: c.tipo,
      naturaleza: c.naturaleza,
      display: `${c.codigo} - ${c.nombre} (${c.tipo})`
    }))

    asientos.value = resAsientos.data?.datos || []
  } catch (err) {
    console.error('Error al cargar datos contables:', err)
    error.value = 'No se pudo conectar con el backend MySQL. Verifica que el servidor esté activo.'
  } finally {
    cargando.value = false
  }
}

// ============ VALIDACIÓN CONTABLE DE PARTIDA DOBLE ============
const totalDebe = computed(() => 
  Math.round(partidas.value.reduce((a, p) => a + Number(p.debe || 0), 0) * 100) / 100
)
const totalHaber = computed(() => 
  Math.round(partidas.value.reduce((a, p) => a + Number(p.haber || 0), 0) * 100) / 100
)
const diferencia = computed(() => 
  Math.round((totalDebe.value - totalHaber.value) * 100) / 100
)
const esValido = computed(() =>
  totalDebe.value > 0 && Math.abs(diferencia.value) < 0.01 && concepto.value.trim() !== '' &&
  partidas.value.every(p => p.cuenta_id)
)

const agregarPartida = () => {
  partidas.value.push({ cuenta_id: null, debe: 0, haber: 0 })
}

const eliminarPartida = (idx) => {
  if (partidas.value.length > 2) partidas.value.splice(idx, 1)
}

// ============ REGISTRAR ASIENTO CON TRANSACCIÓN ACID ============
const registrarAsiento = async () => {
  if (!esValido.value) return
  
  registrando.value = true
  error.value = null
  exitoMensaje.value = null

  try {
    const payload = {
      fecha: fecha.value,
      descripcion: concepto.value,
      lineas: partidas.value.map(p => ({
        cuenta_id: p.cuenta_id,
        debe: Number(p.debe || 0),
        haber: Number(p.haber || 0)
      }))
    }

    const res = await asientoService.create(payload)

    exitoMensaje.value = `¡Asiento #${res.data?.id} registrado exitosamente con Transacción ACID! (Commit de Partida Doble)`

    // Resetear formulario
    concepto.value = ''
    partidas.value = [
      { cuenta_id: null, debe: 0, haber: 0 },
      { cuenta_id: null, debe: 0, haber: 0 }
    ]

    // Recargar libro diario desde la BD
    const resAsientos = await asientoService.getAll()
    asientos.value = resAsientos.data?.datos || []

  } catch (err) {
    console.error('Error al registrar asiento contable:', err)
    error.value = err.response?.data?.mensaje || 'Error al procesar la transacción contable (Rollback ejecutado).'
  } finally {
    registrando.value = false
  }
}

// Cargar ejemplo de prueba (Venta de Servicios)
const generarVentaEjemplo = () => {
  concepto.value = 'Registro de ingreso por venta de servicios'
  
  // Buscar IDs de Caja General y Ventas de Servicios en el catálogo
  const caja = listaCuentas.value.find(c => c.codigo === '1.1.01')
  const ventas = listaCuentas.value.find(c => c.codigo === '4.1.01')

  partidas.value = [
    { cuenta_id: caja ? caja.id : 1, debe: 1500.00, haber: 0 },
    { cuenta_id: ventas ? ventas.id : 6, debe: 0, haber: 1500.00 }
  ]
}

onMounted(cargarCatalogosYAsientos)
</script>

<template>
  <div class="contabilidad-view">
    <!-- TITULO Y CONTEXTO -->
    <div class="d-flex align-center justify-space-between mb-4">
      <div>
        <h1 class="text-h4 font-weight-bold">
          <v-icon color="primary" class="mr-2">mdi-notebook-edit-outline</v-icon>
          Motor Contable — Partida Doble & Transacciones ACID
        </h1>
        <p class="text-subtitle-1 text-grey">
          Semana 10: Garantía de Consistencia Financiera con MySQL & Node.js
        </p>
      </div>

      <v-btn
        variant="tonal"
        color="primary"
        prepend-icon="mdi-refresh"
        :loading="cargando"
        @click="cargarCatalogosYAsientos"
      >
        Actualizar
      </v-btn>
    </div>

    <!-- REGLA DE ORO / CONTEXTO EDUCATIVO -->
    <v-alert type="info" variant="tonal" class="mb-4" icon="mdi-shield-check">
      <strong>Regla de Oro de Partida Doble:</strong> SUMA(DEBE) == SUMA(HABER). Si la transacción desbalancea la cuenta, la base de datos ejecuta un <strong>ROLLBACK</strong> automático para evitar inconsistencias financieras.
    </v-alert>

    <!-- ALERTAS DE FEEDBACK -->
    <v-alert
      v-if="error"
      type="error"
      variant="tonal"
      closable
      class="mb-4"
      icon="mdi-alert-circle"
      @click:close="error = null"
    >
      <strong>Error / Rollback de Transacción:</strong> {{ error }}
    </v-alert>

    <v-alert
      v-if="exitoMensaje"
      type="success"
      variant="tonal"
      closable
      class="mb-4"
      icon="mdi-check-circle-outline"
      @click:close="exitoMensaje = null"
    >
      {{ exitoMensaje }}
    </v-alert>

    <!-- CARD FORMULARIO DE ASIENTO CONTABLE -->
    <v-card class="elevation-2 rounded-lg">
      <v-card-title class="d-flex align-center bg-surface-light border-b py-3">
        <span class="text-h6 font-weight-bold">Nuevo Asiento Contable</span>
        <v-spacer />
        <v-btn
          variant="outlined"
          color="secondary"
          size="small"
          prepend-icon="mdi-auto-fix"
          @click="generarVentaEjemplo"
        >
          Cargar Asiento Ejemplo ($1,500.00)
        </v-btn>
      </v-card-title>

      <v-card-text class="pa-5">
        <v-row>
          <v-col cols="12" md="4">
            <v-text-field
              v-model="fecha"
              label="Fecha del Asiento"
              type="date"
              variant="outlined"
              density="comfortable"
              prepend-inner-icon="mdi-calendar"
            />
          </v-col>
          <v-col cols="12" md="8">
            <v-text-field
              v-model="concepto"
              label="Concepto / Descripción Contable"
              placeholder="Ej: Registro de venta de servicios"
              variant="outlined"
              density="comfortable"
              prepend-inner-icon="mdi-text-short"
            />
          </v-col>
        </v-row>

        <v-divider class="my-4" />

        <!-- TABLA DE PARTIDAS -->
        <h3 class="text-subtitle-1 font-weight-bold mb-3 d-flex align-center">
          <v-icon start size="small" color="primary">mdi-table-edit</v-icon>
          Líneas de la Partida Doble
        </h3>

        <v-table class="border rounded mb-3">
          <thead>
            <tr class="bg-surface-light">
              <th class="text-left font-weight-bold" style="width: 45%;">Cuenta Contable</th>
              <th class="text-right font-weight-bold" style="width: 25%;">DEBE ($)</th>
              <th class="text-right font-weight-bold" style="width: 25%;">HABER ($)</th>
              <th class="text-center" style="width: 5%;">Acción</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(p, idx) in partidas" :key="idx">
              <td class="py-2">
                <v-select
                  v-model="p.cuenta_id"
                  :items="listaCuentas"
                  item-title="display"
                  item-value="id"
                  placeholder="Selecciona una cuenta..."
                  variant="outlined"
                  density="compact"
                  hide-details
                />
              </td>
              <td class="py-2">
                <v-text-field
                  v-model.number="p.debe"
                  type="number"
                  step="0.01"
                  min="0"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="text-right"
                  :disabled="p.haber > 0"
                />
              </td>
              <td class="py-2">
                <v-text-field
                  v-model.number="p.haber"
                  type="number"
                  step="0.01"
                  min="0"
                  variant="outlined"
                  density="compact"
                  hide-details
                  class="text-right"
                  :disabled="p.debe > 0"
                />
              </td>
              <td class="text-center py-2">
                <v-btn
                  icon="mdi-delete-outline"
                  size="small"
                  color="error"
                  variant="text"
                  :disabled="partidas.length <= 2"
                  @click="eliminarPartida(idx)"
                />
              </td>
            </tr>
            <!-- REGISTRO DE TOTALES -->
            <tr class="bg-surface-light font-weight-bold">
              <td class="text-right pr-4">TOTALES:</td>
              <td class="text-right text-subtitle-1 text-primary">
                ${{ totalDebe.toFixed(2) }}
              </td>
              <td class="text-right text-subtitle-1 text-primary">
                ${{ totalHaber.toFixed(2) }}
              </td>
              <td></td>
            </tr>
          </tbody>
        </v-table>

        <v-btn
          variant="outlined"
          color="primary"
          size="small"
          prepend-icon="mdi-plus"
          @click="agregarPartida"
        >
          Agregar Cuenta
        </v-btn>

        <!-- INDICADOR DE ESTADO DE VALIDADOR CONTABLE -->
        <v-alert
          :type="esValido ? 'success' : Math.abs(diferencia) < 0.01 ? 'info' : 'error'"
          variant="tonal"
          class="mt-4"
          density="comfortable"
        >
          <template v-if="esValido">
            <v-icon start>mdi-check-all</v-icon>
            <strong>Asiento Cuadrado (Partida Doble Balanceada)</strong> — Listo para COMMIT en MySQL.
          </template>
          <template v-else-if="Math.abs(diferencia) < 0.01">
            <v-icon start>mdi-information</v-icon>
            Asiento balanceado. Asigna las cuentas y un concepto para proceder.
          </template>
          <template v-else>
            <v-icon start>mdi-scale-unbalanced</v-icon>
            <strong>Desbalanceado (Diferencia: ${{ Math.abs(diferencia).toFixed(2) }})</strong> — Si intentas enviarlo, el backend ejecutará ROLLBACK.
          </template>
        </v-alert>
      </v-card-text>

      <v-card-actions class="pa-4 bg-surface-light border-t">
        <v-spacer />
        <v-btn
          color="primary"
          size="large"
          elevation="2"
          :disabled="!esValido || registrando"
          :loading="registrando"
          prepend-icon="mdi-content-save-check"
          @click="registrarAsiento"
        >
          Registrar Asiento (Commit ACID)
        </v-btn>
      </v-card-actions>
    </v-card>

    <!-- LIBRO DIARIO - PERSISTENCIA REAL -->
    <v-card class="mt-6 elevation-2 rounded-lg">
      <v-card-title class="d-flex align-center border-b py-3 bg-surface-light">
        <v-icon color="primary" class="mr-2">mdi-book-open-page-variant</v-icon>
        <span class="text-h6 font-weight-bold">Libro Diario (Persistencia en MySQL)</span>
        <v-spacer />
        <v-chip color="primary" size="small" variant="tonal">
          {{ asientos.length }} Asientos Registrados
        </v-chip>
      </v-card-title>

      <v-card-text class="pa-4">
        <div v-if="cargando" class="text-center py-8">
          <v-progress-circular indeterminate color="primary" />
          <div class="mt-2 text-caption text-grey">Cargando Libro Diario desde MySQL...</div>
        </div>

        <div v-else-if="asientos.length === 0" class="text-center py-8 text-grey">
          <v-icon size="48" class="mb-2">mdi-book-remove-outline</v-icon>
          <div>No hay asientos contables en la base de datos.</div>
        </div>

        <v-expansion-panels v-else variant="accordion" class="my-2">
          <v-expansion-panel v-for="asiento in asientos" :key="asiento.id" class="mb-2 border rounded">
            <v-expansion-panel-title class="py-2">
              <div class="d-flex align-center w-100 pr-4">
                <v-chip color="primary" size="small" class="mr-3 font-weight-bold">
                  Asiento #{{ asiento.id }}
                </v-chip>
                <span class="font-weight-medium text-body-1">{{ asiento.descripcion || 'Sin descripción' }}</span>
                <v-spacer />
                <span class="text-caption text-grey mr-4">
                  {{ asiento.fecha ? String(asiento.fecha).substring(0, 10) : '' }}
                </span>
                <v-chip color="success" size="small" variant="tonal" class="font-weight-bold">
                  Cuadrado (${{ Number(asiento.total_debe || 0).toFixed(2) }})
                </v-chip>
              </div>
            </v-expansion-panel-title>

            <v-expansion-panel-text>
              <v-table density="compact" class="elevation-0">
                <thead>
                  <tr class="bg-surface-light">
                    <th>Código & Cuenta Contable</th>
                    <th class="text-right">Debe ($)</th>
                    <th class="text-right">Haber ($)</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="(linea, i) in asiento.lineas" :key="i">
                    <td class="font-weight-medium">
                      {{ linea.cuenta || `${linea.cuenta_codigo} - ${linea.cuenta_nombre}` }}
                    </td>
                    <td class="text-right">
                      {{ Number(linea.debe) > 0 ? '$' + Number(linea.debe).toFixed(2) : '-' }}
                    </td>
                    <td class="text-right">
                      {{ Number(linea.haber) > 0 ? '$' + Number(linea.haber).toFixed(2) : '-' }}
                    </td>
                  </tr>
                  <tr class="font-weight-bold bg-surface-light">
                    <td class="text-right">Totales:</td>
                    <td class="text-right">${{ Number(asiento.total_debe).toFixed(2) }}</td>
                    <td class="text-right">${{ Number(asiento.total_haber).toFixed(2) }}</td>
                  </tr>
                </tbody>
              </v-table>
            </v-expansion-panel-text>
          </v-expansion-panel>
        </v-expansion-panels>
      </v-card-text>
    </v-card>
  </div>
</template>

<style scoped>
.contabilidad-view {
  max-width: 1200px;
  margin: 0 auto;
}
</style>
