<template>
  <div class="motor-contable-view">
    <!-- Encabezado de la Sección -->
    <div class="d-flex align-center justify-space-between mb-6">
      <div>
        <h1 class="text-h4 font-weight-bold d-flex align-center">
          <v-icon color="primary" class="mr-3">mdi-engine</v-icon>
          Motor Contable
        </h1>
        <p class="text-subtitle-1 text-grey">Gestión asíncrona de movimientos y saldo en tiempo real</p>
      </div>

      <v-btn 
        color="primary" 
        variant="tonal" 
        prepend-icon="mdi-refresh" 
        :loading="cargando"
        @click="cargarMovimientos"
      >
        Actualizar
      </v-btn>
    </div>

    <!-- Alerta de Error -->
    <v-alert
      v-if="error"
      type="error"
      variant="tonal"
      closable
      icon="mdi-alert-circle"
      class="mb-6 elevation-1"
      @click:close="error = null"
    >
      <div class="font-weight-medium">{{ error }}</div>
    </v-alert>

    <!-- Panel de KPIs (Ejercicio 4) -->
    <v-row class="mb-6">
      <v-col cols="12" md="4">
        <v-card elevation="2" rounded="lg" class="kpi-card bg-surface">
          <v-card-item>
            <template v-slot:prepend>
              <v-avatar color="success" variant="tonal" rounded="lg" size="48">
                <v-icon size="28">mdi-trending-up</v-icon>
              </v-avatar>
            </template>
            <v-card-title class="text-caption text-grey text-uppercase font-weight-bold">
              Total Ingresos
            </v-card-title>
            <div class="text-h5 font-weight-bold text-success">
              ${{ totalIngresos.toFixed(2) }}
            </div>
          </v-card-item>
        </v-card>
      </v-col>

      <v-col cols="12" md="4">
        <v-card elevation="2" rounded="lg" class="kpi-card bg-surface">
          <v-card-item>
            <template v-slot:prepend>
              <v-avatar color="error" variant="tonal" rounded="lg" size="48">
                <v-icon size="28">mdi-trending-down</v-icon>
              </v-avatar>
            </template>
            <v-card-title class="text-caption text-grey text-uppercase font-weight-bold">
              Total Egresos
            </v-card-title>
            <div class="text-h5 font-weight-bold text-error">
              ${{ totalEgresos.toFixed(2) }}
            </div>
          </v-card-item>
        </v-card>
      </v-col>

      <v-col cols="12" md="4">
        <v-card elevation="3" rounded="lg" class="kpi-card border-primary bg-primary-subtle">
          <v-card-item>
            <template v-slot:prepend>
              <v-avatar color="primary" variant="flat" rounded="lg" size="48">
                <v-icon size="28" color="white">mdi-scale-balance</v-icon>
              </v-avatar>
            </template>
            <v-card-title class="text-caption text-primary text-uppercase font-weight-bold">
              Saldo Total
            </v-card-title>
            <div class="text-h5 font-weight-bold" :class="saldo >= 0 ? 'text-primary' : 'text-error'">
              ${{ saldo.toFixed(2) }}
            </div>
          </v-card-item>
        </v-card>
      </v-col>
    </v-row>

    <v-row class="mb-6">
      <!-- Formulario de Creación (Ejercicio 3) -->
      <v-col cols="12" lg="4">
        <v-card elevation="2" rounded="lg">
          <v-card-item class="border-b bg-surface-light">
            <v-card-title class="d-flex align-center text-subtitle-1 font-weight-bold">
              <v-icon start color="primary">mdi-plus-circle-outline</v-icon>
              Registrar Movimiento
            </v-card-title>
          </v-card-item>

          <v-card-text class="pa-5">
            <form @submit.prevent="guardarMovimiento">
              <v-text-field
                v-model="nuevoMovimiento.concepto"
                label="Concepto"
                placeholder="Ej: Venta de servicios"
                variant="outlined"
                density="comfortable"
                prepend-inner-icon="mdi-text-short"
                required
                class="mb-3"
              />

              <v-select
                v-model="nuevoMovimiento.tipo"
                :items="['Ingreso', 'Egreso']"
                label="Tipo de Movimiento"
                variant="outlined"
                density="comfortable"
                prepend-inner-icon="mdi-swap-horizontal"
                required
                class="mb-3"
              />

              <v-text-field
                v-model="nuevoMovimiento.monto"
                label="Monto ($)"
                type="number"
                step="0.01"
                min="0.01"
                placeholder="0.00"
                variant="outlined"
                density="comfortable"
                prepend-inner-icon="mdi-currency-usd"
                required
                class="mb-4"
              />

              <v-btn
                type="submit"
                color="primary"
                block
                size="large"
                elevation="2"
                :loading="cargando"
                prepend-icon="mdi-content-save"
              >
                Guardar Movimiento
              </v-btn>
            </form>
          </v-card-text>
        </v-card>
      </v-col>

      <!-- Lista de Movimientos y Estado de Carga (Ejercicio 2) -->
      <v-col cols="12" lg="8">
        <v-card elevation="2" rounded="lg">
          <v-card-item class="border-b bg-surface-light d-flex align-center justify-space-between py-3">
            <v-card-title class="d-flex align-center text-subtitle-1 font-weight-bold">
              <v-icon start color="primary">mdi-history</v-icon>
              Historial de Movimientos
            </v-card-title>
            <v-chip size="small" variant="tonal" color="primary">
              {{ movimientos.length }} registros
            </v-chip>
          </v-card-item>

          <!-- Spinner de Carga -->
          <div v-if="cargando" class="text-center py-12 cargando">
            <v-progress-circular indeterminate color="primary" size="50" width="4" />
            <div class="mt-4 text-body-1 text-grey font-weight-medium">
              Cargando datos del servidor... ⏳
            </div>
          </div>

          <!-- Tabla de Datos -->
          <v-table v-else-if="movimientos.length > 0" class="elevation-0">
            <thead>
              <tr>
                <th class="text-left font-weight-bold">Concepto</th>
                <th class="text-center font-weight-bold">Tipo</th>
                <th class="text-right font-weight-bold">Monto</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(mov, index) in movimientos" :key="mov.id || index">
                <td class="font-weight-medium">{{ mov.concepto }}</td>
                <td class="text-center">
                  <v-chip
                    size="small"
                    :color="mov.tipo === 'Ingreso' ? 'success' : 'error'"
                    variant="tonal"
                    class="font-weight-bold"
                  >
                    <v-icon start size="x-small">
                      {{ mov.tipo === 'Ingreso' ? 'mdi-arrow-up-bold' : 'mdi-arrow-down-bold' }}
                    </v-icon>
                    {{ mov.tipo }}
                  </v-chip>
                </td>
                <td 
                  class="text-right font-weight-bold text-subtitle-1"
                  :class="mov.tipo === 'Ingreso' ? 'text-success' : 'text-error'"
                >
                  {{ mov.tipo === 'Ingreso' ? '+' : '-' }}${{ Number(mov.monto).toFixed(2) }}
                </td>
              </tr>
            </tbody>
          </v-table>

          <!-- Estado Vacío -->
          <div v-else class="text-center py-12 px-4">
            <v-icon size="64" color="grey-lighten-1" class="mb-3">mdi-tray-blank</v-icon>
            <div class="text-h6 font-weight-bold text-grey">No hay movimientos registrados</div>
            <div class="text-caption text-grey">Agrega tu primer ingreso o egreso usando el formulario.</div>
          </div>
        </v-card>
      </v-col>
    </v-row>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import { movimientoService } from '@/services/erpApi';

// ==========================================
// EJERCICIO 2: Variables Reactivas
// ==========================================
const movimientos = ref([]);
const cargando = ref(false);
const error = ref(null);

// ==========================================
// EJERCICIO 3: Variable para el Formulario
// ==========================================
const nuevoMovimiento = ref({ 
  concepto: '', 
  tipo: 'Ingreso', 
  monto: '' 
});

// ==========================================
// EJERCICIO 4: Propiedades Computadas (KPIs)
// ==========================================
// El uso del reduce con un valor inicial de 0 evita que el resultado sea NaN si el array está vacío
const totalIngresos = computed(() => {
  return movimientos.value
    .filter(m => m.tipo === 'Ingreso')
    .reduce((sum, m) => sum + Number(m.monto), 0);
});

const totalEgresos = computed(() => {
  return movimientos.value
    .filter(m => m.tipo === 'Egreso')
    .reduce((sum, m) => sum + Number(m.monto), 0);
});

const saldo = computed(() => totalIngresos.value - totalEgresos.value);

// ==========================================
// EJERCICIO 2: Carga de Datos Asíncrona
// ==========================================
async function cargarMovimientos() {
  cargando.value = true;
  error.value = null;

  try {
    const respuesta = await movimientoService.getAll();
    // Se asume que el backend devuelve la estructura: { data: { datos: [...] } }
    movimientos.value = respuesta.data.datos;
  } catch (err) {
    error.value = 'No se pudo conectar con el servidor. Verifica que esté encendido.';
  } finally {
    // Garantiza que el spinner se oculte aunque falle
    cargando.value = false;
  }
}

// ==========================================
// EJERCICIO 3: Guardar Datos
// ==========================================
async function guardarMovimiento() {
  cargando.value = true;
  error.value = null;
  
  try {
    // Aseguramos que el monto se envíe como número
    const payload = {
      ...nuevoMovimiento.value,
      monto: Number(nuevoMovimiento.value.monto)
    };

    const respuesta = await movimientoService.create(payload);
    
    // Actualización reactiva inmediata en la UI
    movimientos.value.push(respuesta.data.datos); 
    
    // Limpiar el formulario
    nuevoMovimiento.value = { concepto: '', tipo: 'Ingreso', monto: '' }; 
  } catch (err) {
    // Captura el mensaje específico del backend HTTP 400
    error.value = err.response?.data?.mensaje || 'Error al guardar el movimiento'; 
  } finally {
    cargando.value = false;
  }
}

// Hook del ciclo de vida para cargar datos al montar el componente
onMounted(cargarMovimientos);
</script>

<style scoped>
.kpi-card {
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}
.kpi-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12) !important;
}
</style>