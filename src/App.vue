<script setup>
import { ref, nextTick } from 'vue';
import { useRoute, RouterView } from 'vue-router';
import { useTheme } from 'vuetify';

const theme = useTheme();
const route = useRoute();
const drawer = ref(true);

const menuItems = [
  { title: 'Dashboard', icon: 'mdi-view-dashboard', to: '/' },
  { title: 'Clientes', icon: 'mdi-account-group', to: '/clientes' },
  { title: 'Facturación', icon: 'mdi-receipt', to: '/facturacion' },
  { title: 'Contabilidad (Motor)', icon: 'mdi-calculator', to: '/contabilidad' }
];

async function toggleTheme(event) {
  const isDark = theme.global.current.value.dark;
  const targetTheme = isDark ? 'light' : 'dark';

  // Fallback para navegadores sin soporte a View Transitions API
  if (!document.startViewTransition) {
    theme.global.name.value = targetTheme;
    return;
  }

  // REQUERIMIENTO 3: Cálculo confiable del centro (x, y) del botón independientemente de nodos hijos (v-icon, svg)
  const button = event?.currentTarget || event?.target?.closest?.('.v-btn') || event?.target;
  const rect = button?.getBoundingClientRect?.() ?? {
    left: window.innerWidth - 60,
    top: 30,
    width: 40,
    height: 40
  };

  const x = (event?.clientX && event.clientX > 0)
    ? event.clientX
    : (rect.left + rect.width / 2);

  const y = (event?.clientY && event.clientY > 0)
    ? event.clientY
    : (rect.top + rect.height / 2);

  const endRadius = Math.hypot(
    Math.max(x, window.innerWidth - x),
    Math.max(y, window.innerHeight - y)
  );

  const root = document.documentElement;

  // REQUERIMIENTO 1 y 2: Asignar clases en :root (<html>) y neutralizar transiciones de Vuetify durante el snapshot
  root.classList.add('no-theme-transitions');
  if (!isDark) {
    root.classList.add('wave-to-dark');
    root.classList.remove('wave-to-light');
  } else {
    root.classList.add('wave-to-light');
    root.classList.remove('wave-to-dark');
  }

  // Ejecutar View Transition y esperar la renderización síncrona de Vue con nextTick()
  const transition = document.startViewTransition(async () => {
    theme.global.name.value = targetTheme;
    await nextTick();
  });

  await transition.ready;

  const clipPath = [
    `circle(0px at ${x}px ${y}px)`,
    `circle(${endRadius}px at ${x}px ${y}px)`
  ];

  if (!isDark) {
    // EXPANSIÓN: La vista NUEVA (oscura) con z-index: 9999 se expande desde (x, y) sobre la vista clara
    root.animate(
      { clipPath: clipPath },
      {
        duration: 500,
        easing: 'cubic-bezier(0.4, 0, 0.2, 1)',
        pseudoElement: '::view-transition-new(root)'
      }
    );
  } else {
    // RETRACCIÓN: La vista ANTIGUA (oscura) con z-index: 9999 se encoge de regreso a (x, y) revelando la vista clara
    root.animate(
      { clipPath: clipPath.slice().reverse() },
      {
        duration: 500,
        easing: 'cubic-bezier(0.4, 0, 0.2, 1)',
        pseudoElement: '::view-transition-old(root)'
      }
    );
  }

  // Limpiar clases al concluir la animación
  transition.finished.finally(() => {
    root.classList.remove('wave-to-dark', 'wave-to-light', 'no-theme-transitions');
  });
}
</script>

<template>
  <v-app>
    <!-- BARRA SUPERIOR CON ESTILO ELEGANTE -->
    <v-app-bar elevation="1" density="comfortable" class="px-2">
      <v-app-bar-nav-icon @click="drawer = !drawer" />
      <v-app-bar-title class="font-weight-bold d-flex align-center">
        <v-icon start color="primary" class="mr-2">mdi-calculator-variant-outline</v-icon>
        <span>ERP Contable</span>
        <v-chip size="x-small" color="primary" class="ml-3 font-weight-medium" variant="tonal">
          Semana 8
        </v-chip>
      </v-app-bar-title>

      <v-spacer />

      <!-- BOTÓN MODO OSCURO CON ONDA EXPANSIVA Y RETRÁCTIL -->
      <v-btn 
        :icon="theme.global.current.value.dark ? 'mdi-weather-sunny' : 'mdi-weather-night'" 
        variant="tonal"
        color="primary" 
        class="mr-3 theme-toggle-btn" 
        title="Cambiar tema con efecto de onda" 
        @click="toggleTheme" 
      />

      <v-chip color="secondary" variant="outlined" class="mr-2 hidden-xs-only">
        <v-icon start size="small">mdi-school</v-icon>
        Vue 3 + Express
      </v-chip>
    </v-app-bar>

    <!-- MENÚ LATERAL -->
    <v-navigation-drawer v-model="drawer" width="280" class="elevation-1">
      <div class="pa-4 text-center border-b">
        <v-avatar color="primary" variant="tonal" size="48" class="mb-2">
          <v-icon size="28">mdi-bank-transfer</v-icon>
        </v-avatar>
        <div class="text-subtitle-1 font-weight-bold">Motor Contable</div>
        <div class="text-caption text-grey">Programación Asíncrona</div>
      </div>

      <v-list nav class="px-3 py-3">
        <v-list-item 
          v-for="item in menuItems" 
          :key="item.to" 
          :to="item.to" 
          :prepend-icon="item.icon"
          :title="item.title" 
          :active="route.path === item.to" 
          color="primary" 
          rounded="lg" 
          class="mb-1" 
        />
      </v-list>

      <template v-slot:append>
        <div class="pa-4 border-t">
          <div class="d-flex align-center text-caption text-grey">
            <v-icon size="small" class="mr-2" color="success">mdi-check-decagram</v-icon>
            <span>Estado: Conectado a Node.js</span>
          </div>
        </div>
      </template>
    </v-navigation-drawer>

    <!-- CONTENIDO PRINCIPAL -->
    <v-main>
      <v-container fluid class="pa-6 max-width-container">
        <RouterView />
      </v-container>
    </v-main>
  </v-app>
</template>

<style>
/* REQUERIMIENTO 2: Neutralizar transiciones CSS nativas de Vuetify durante la captura del snapshot para eliminar el flashazo */
html.no-theme-transitions *,
html.no-theme-transitions *::before,
html.no-theme-transitions *::after {
  transition: none !important;
}

/* Reglas universales para la View Transitions API en :root (<html>) */
::view-transition-old(root),
::view-transition-new(root) {
  animation: none;
  mix-blend-mode: normal;
  display: block;
}

/* REQUERIMIENTO 1: Apilamiento z-index aplicando clases en :root (<html>) */

/* 1. MODO CLARO -> OSCURO (Onda expansiva):
   La vista NUEVA (oscura) se coloca por ENCIMA (z-index: 9999) y se expande */
html.wave-to-dark ::view-transition-old(root) {
  z-index: 1;
}
html.wave-to-dark ::view-transition-new(root) {
  z-index: 9999;
}

/* 2. MODO OSCURO -> CLARO (Onda retráctil):
   La vista ANTIGUA (oscura) se coloca por ENCIMA (z-index: 9999) y se encoge hacia el botón */
html.wave-to-light ::view-transition-old(root) {
  z-index: 9999;
}
html.wave-to-light ::view-transition-new(root) {
  z-index: 1;
}

.max-width-container {
  max-width: 1280px;
  margin: 0 auto;
}

.theme-toggle-btn {
  transition: transform 0.2s ease;
}

.theme-toggle-btn:hover {
  transform: scale(1.1) rotate(15deg);
}
</style>