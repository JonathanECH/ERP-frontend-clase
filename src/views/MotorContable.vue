<template>
  <div class="motor-contable">
    <h1>Motor Contable</h1>

    <!-- Manejo de Estados: Error -->
    <div v-if="error" class="error-mensaje">
      {{ error }}
    </div>

    <!-- Panel de KPIs (Ejercicio 4) -->
    <div class="kpi-panel">
      <div class="kpi">
        <h3>Total Ingresos</h3>
        <p>${{ totalIngresos.toFixed(2) }}</p>
      </div>
      <div class="kpi">
        <h3>Total Egresos</h3>
        <p>${{ totalEgresos.toFixed(2) }}</p>
      </div>
      <div class="kpi saldo">
        <h3>Saldo Total</h3>
        <p>${{ saldo.toFixed(2) }}</p>
      </div>
    </div>

    <!-- Formulario de Creación (Ejercicio 3) -->
    <div class="formulario">
      <h3>Registrar Nuevo Movimiento</h3>
      <form @submit.prevent="guardarMovimiento">
        <label>
          Concepto:
          <input type="text" v-model="nuevoMovimiento.concepto" required />
        </label>
        
        <label>
          Tipo:
          <select v-model="nuevoMovimiento.tipo">
            <option value="Ingreso">Ingreso</option>
            <option value="Egreso">Egreso</option>
          </select>
        </label>

        <label>
          Monto:
          <input type="number" step="0.01" v-model="nuevoMovimiento.monto" required />
        </label>

        <button type="submit" :disabled="cargando">
          {{ cargando ? 'Guardando...' : 'Guardar' }}
        </button>
      </form>
    </div>

    <!-- Lista de Movimientos y Estado de Carga (Ejercicio 2) -->
    <div class="lista-movimientos">
      <h3>Historial de Movimientos</h3>
      
      <div v-if="cargando" class="cargando">
        Cargando datos... ⏳
      </div>
      
      <table v-else-if="movimientos.length > 0">
        <thead>
          <tr>
            <th>Concepto</th>
            <th>Tipo</th>
            <th>Monto</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(mov, index) in movimientos" :key="index">
            <td>{{ mov.concepto }}</td>
            <td>{{ mov.tipo }}</td>
            <td>${{ mov.monto }}</td>
          </tr>
        </tbody>
      </table>

      <p v-else>No hay movimientos registrados.</p>
    </div>
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
/* Estilos básicos opcionales para estructurar la vista */
.motor-contable { max-width: 800px; margin: 0 auto; padding: 20px; font-family: sans-serif; }
.error-mensaje { background: #fee2e2; color: #dc2626; padding: 10px; margin-bottom: 20px; border-radius: 4px; }
.kpi-panel { display: flex; gap: 20px; margin-bottom: 30px; }
.kpi { flex: 1; background: #f3f4f6; padding: 15px; border-radius: 8px; text-align: center; }
.kpi.saldo { background: #dbeafe; font-weight: bold; }
.formulario { background: #f9fafb; padding: 20px; border-radius: 8px; margin-bottom: 30px; }
.formulario label { display: block; margin-bottom: 10px; }
.formulario input, .formulario select { margin-left: 10px; padding: 5px; }
.formulario button { margin-top: 15px; padding: 8px 16px; background: #2563eb; color: white; border: none; border-radius: 4px; cursor: pointer; }
.formulario button:disabled { background: #9ca3af; }
table { width: 100%; border-collapse: collapse; margin-top: 15px; }
th, td { text-align: left; padding: 10px; border-bottom: 1px solid #e5e7eb; }
.cargando { font-size: 1.2em; color: #4b5563; }
</style>