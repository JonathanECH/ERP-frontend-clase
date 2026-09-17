<script setup>
import { ref, computed } from 'vue'
import TarjetaKPI from '../components/TarjetaKPI.vue'
// ============ ESTADO REACTIVO ============
// ref() hace que Vue "observe" estas variables y actualice la vista cuando cambian
const clientes = ref([
    {
        id: 1, nombre: 'Distribuidora del Norte S.A.', rfc: 'DNO900101ABC', email:
            'ventas@norte.mx', telefono: '555-1234', tipo: 'cliente', saldo: 15000
    },
    {
        id: 2, nombre: 'Papelería Central', rfc: 'PCE850515XYZ', email:
            'contacto@central.mx', telefono: '555-5678', tipo: 'proveedor', saldo: -3200
    },
    {
        id: 3, nombre: 'Servicios Técnicos López', rfc: 'STL920320DEF', email:
            'lopez@servicios.mx', telefono: '555-9012', tipo: 'cliente', saldo: 8500
    }
])
const busqueda = ref('')
const dialog = ref(false)
const editando = ref(false)
const formulario = ref({
    id: 0, nombre: '', rfc: '', email: '', telefono: '', tipo: 'cliente', saldo: 0
})
// ============ VALIDACIONES ============
// Cada función recibe el valor y retorna true (válido) o un mensaje de error
const reglas = {
    requerido: (v) => !!v || 'Campo obligatorio',
    rfc: (v) => /^[A-ZÑ&]{3,4}\d{6}[A-Z0-9]{3}$/.test(v) || 'RFC inválido (ej: DNO900101ABC) ',
email: (v) => /.+@.+\..+/.test(v) || 'Email inválido'
}
// ============ COMPUTED ============
// computed() recalcula automáticamente cuando cambian sus dependencias
const clientesFiltrados = computed(() => {
    const termino = busqueda.value.toLowerCase()
    return clientes.value.filter(c =>
        c.nombre.toLowerCase().includes(termino) ||
        c.rfc.toLowerCase().includes(termino)
    )
})
const totalClientes = computed(() => clientes.value.filter(c => c.tipo ===
    'cliente').length)
const totalProveedores = computed(() => clientes.value.filter(c => c.tipo ===
    'proveedor').length)
const saldoNeto = computed(() => clientes.value.reduce((acc, c) => acc + c.saldo,
    0))
// ============ MÉTODOS ============
const abrirNuevo = () => {
    editando.value = false
    formulario.value = {
        id: 0, nombre: '', rfc: '', email: '', telefono: '', tipo:
            'cliente', saldo: 0
    }
    dialog.value = true
}
const abrirEditar = (cliente) => {
    editando.value = true
    formulario.value = { ...cliente }  // Copia para no modificar el original
    dialog.value = true
}
const guardar = () => {
    if (editando.value) {
        const idx = clientes.value.findIndex(c => c.id === formulario.value.id)
        if (idx >= 0) clientes.value[idx] = { ...formulario.value }
    } else {
        const nuevoId = Math.max(0, ...clientes.value.map(c => c.id)) + 1
        clientes.value.push({ ...formulario.value, id: nuevoId })
    }
    dialog.value = false
}
const eliminar = (id) => {
    if (confirm('¿Eliminar este registro?')) {
        clientes.value = clientes.value.filter(c => c.id !== id)
    }
}
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
                    <v-chip :color="item.tipo === 'cliente' ? 'primary' : 'secondary'" size="small">
                        {{ item.tipo === 'cliente' ? 'Cliente' : 'Proveedor' }}
                    </v-chip>
                </template>
                <!-- Slot personalizado para columna saldo -->
                <template v-slot:item.saldo="{ item }">
                    <span :class="item.saldo >= 0 ? 'text-success' : 'text-error'">
                        ${{ item.saldo.toLocaleString() }}
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
