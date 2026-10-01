<script setup>
import { ref, computed, onMounted } from 'vue';
import TarjetaKPI from '../components/TarjetaKPI.vue';
import { contactoService } from '@/services/erpApi';

// ============ ESTADO REACTIVO ============
const clientes = ref([]);
const cargando = ref(false);
const error = ref(null);
const busqueda = ref('');
const dialog = ref(false);
const editando = ref(false);
const formulario = ref({
  id: 0, nombre: '', rfc: '', email: '', telefono: '', tipo: 'Cliente', saldo: 0
});

// ============ VALIDACIONES ============
const reglas = {
  requerido: (v) => !!v || 'Campo obligatorio',
  rfc: (v) => !!v || 'RFC obligatorio',
  email: (v) => !v || /.+@.+\..+/.test(v) || 'Email inválido'
};

// ============ CARGA ASÍNCRONA DESDE MYSQL ============
async function cargarClientes() {
  cargando.value = true;
  error.value = null;
  try {
    const respuesta = await contactoService.getAll();
    clientes.value = respuesta.data.datos;
  } catch (err) {
    error.value = 'No se pudieron cargar los contactos del servidor MySQL.';
  } finally {
    cargando.value = false;
  }
}

// ============ COMPUTED ============
const clientesFiltrados = computed(() => {
  const termino = busqueda.value.toLowerCase();
  return clientes.value.filter(c =>
    (c.nombre && c.nombre.toLowerCase().includes(termino)) ||
    (c.rfc && c.rfc.toLowerCase().includes(termino))
  );
});

const totalClientes = computed(() => clientes.value.filter(c => c.tipo === 'Cliente').length);
const totalProveedores = computed(() => clientes.value.filter(c => c.tipo === 'Proveedor').length);
const saldoNeto = computed(() => clientes.value.reduce((acc, c) => acc + Number(c.saldo || 0), 0));

// ============ MÉTODOS ============
const abrirNuevo = () => {
  editando.value = false;
  formulario.value = { id: 0, nombre: '', rfc: '', email: '', telefono: '', tipo: 'Cliente', saldo: 0 };
  dialog.value = true;
};

const abrirEditar = (cliente) => {
  editando.value = true;
  formulario.value = { ...cliente };
  dialog.value = true;
};

const guardar = async () => {
  cargando.value = true;
  try {
    if (editando.value) {
      await contactoService.update(formulario.value.id, formulario.value);
    } else {
      await contactoService.create(formulario.value);
    }
    await cargarClientes();
    dialog.value = false;
  } catch (err) {
    alert(err.response?.data?.mensaje || 'Error al guardar el contacto en MySQL');
  } finally {
    cargando.value = false;
  }
};

const eliminar = async (id) => {
  if (confirm('¿Eliminar este registro de la base de datos?')) {
    cargando.value = true;
    try {
      await contactoService.delete(id);
      await cargarClientes();
    } catch (err) {
      alert('Error al eliminar contacto');
    } finally {
      cargando.value = false;
    }
  }
};

onMounted(cargarClientes);
</script>
<template>
    <div>
        <!-- ENCABEZADO CON TARJETAS KPI (reutilizamos el componente) -->
        <v-row class="mb-4">
            <v-col cols="12" md="4">
                <TarjetaKPI titulo="Clientes activos" :valor="totalClientes" icono="mdi-account-group"
                    color="primary" />
            </v-col>
            <v-col cols="12" md="4">
                <TarjetaKPI titulo="Proveedores" :valor="totalProveedores" icono="mdi-truck-delivery"
                    color="secondary" />
            </v-col>
            <v-col cols="12" md="4">
                <TarjetaKPI titulo="Saldo neto" :valor="'$' + saldoNeto.toLocaleString()" icono="mdi-cash-multiple"
                    color="success" />
            </v-col>
        </v-row>
        <!-- TABLA DE CLIENTES -->
        <v-card>
            <v-card-title class="d-flex align-center">
                <span>Catálogo de Clientes y Proveedores</span>
                <v-spacer />
                <v-text-field v-model="busqueda" prepend-inner-icon="mdi-magnify" label="Buscar..." variant="outlined"
                    density="compact" hide-details style="max-width: 300px" class="mr-3" />
                <v-btn color="primary" @click="abrirNuevo">
                    <v-icon start>mdi-plus</v-icon>
                    Nuevo
                </v-btn>
            </v-card-title>
            <v-data-table :headers="[
                { title: 'Nombre', key: 'nombre' },
                { title: 'RFC', key: 'rfc' },
                { title: 'Email', key: 'email' },
                { title: 'Tipo', key: 'tipo' },
                { title: 'Saldo', key: 'saldo', align: 'end' },
                { title: 'Acciones', key: 'acciones', sortable: false, align: 'center' }
            ]" :items="clientesFiltrados" :items-per-page="5">
                <!-- Slot personalizado para columna tipo -->
                <template v-slot:item.tipo="{ item }">
                    <v-chip :color="item.tipo?.toLowerCase() === 'cliente' ? 'primary' : 'secondary'" size="small">
                        {{ item.tipo }}
                    </v-chip>
                </template>
                <!-- Slot personalizado para columna saldo -->
                <template v-slot:item.saldo="{ item }">
                    <span :class="(item.saldo || 0) >= 0 ? 'text-success' : 'text-error'">
                        ${{ Number(item.saldo || 0).toLocaleString('es-MX', { minimumFractionDigits: 2 }) }}
                    </span>
                </template>
                <!-- Slot personalizado para acciones -->
                <template v-slot:item.acciones="{ item }">
                    <v-btn icon="mdi-pencil" size="small" color="primary" variant="text" @click="abrirEditar(item)" />
                    <v-btn icon="mdi-delete" size="small" color="error" variant="text" @click="eliminar(item.id)" />
                </template>
            </v-data-table>
        </v-card>
        <!-- DIÁLOGO PARA CREAR/EDITAR -->
        <v-dialog v-model="dialog" max-width="600" persistent>
            <v-card>
                <v-card-title>
                    {{ editando ? 'Editar' : 'Nuevo' }} Registro
                </v-card-title>
                <v-card-text>
                    <v-form>
                        <v-text-field v-model="formulario.nombre" label="Razón Social" :rules="[reglas.requerido]"
                            variant="outlined" class="mb-2" />
                        <v-text-field v-model="formulario.rfc" label="RFC" :rules="[reglas.requerido, reglas.rfc]"
                            variant="outlined" class="mb-2" hint="Ej: DNO900101ABC" />
                        <v-text-field v-model="formulario.email" label="Email" type="email"
                            :rules="[reglas.requerido, reglas.email]" variant="outlined" class="mb-2" />
                        <v-text-field v-model="formulario.telefono" label="Teléfono" variant="outlined" class="mb-2" />
                        <v-select v-model="formulario.tipo" :items="['cliente', 'proveedor']" label="Tipo"
                            variant="outlined" class="mb-2" />
                        <v-text-field v-model.number="formulario.saldo" label="Saldo inicial" type="number"
                            variant="outlined" prefix="$" />
                    </v-form>
                </v-card-text>
                <v-card-actions>
                    <v-spacer />
                    <v-btn @click="dialog = false">Cancelar</v-btn>
                    <v-btn color="primary" @click="guardar">Guardar</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </div>
</template>
