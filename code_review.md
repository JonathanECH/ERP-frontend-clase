# Code Review: Evaluación de Calidad de Código - Semana 8

## 1. Arquitectura y Capa de Servicios (`src/services/erpApi.js`)
- **Fortaleza:** Excelente aplicación del principio de Separación de Responsabilidades (*Separation of Concerns*). Desacoplar Axios de los componentes Vue facilita el mantenimiento y la reutilización del código.
- **Configuración Centralizada:** Uso de `axios.create` definiendo `baseURL`, `timeout` y `headers`, lo que evita la duplicación de código en la configuración HTTP.
- **Mejora Futura:** Extraer `baseURL` a una variable de entorno de Vite (`import.meta.env.VITE_API_BASE_URL`) para entornos de desarrollo y producción.

## 2. Manejo de Estados Asíncronos (`src/views/MotorContable.vue`)
- **Control Estricto de Carga:** Cumplimiento de la regla de oro: asignar `cargando.value = false` obligatoriamente en el bloque `finally`. Garantiza la liberación del spinner independientemente del resultado de la promesa.
- **Ciclo de Vida:** Invocar `cargarMovimientos()` dentro de `onMounted` asegura la ejecución automática del fetch una vez montado el componente.

## 3. Manejo de Errores y Resiliencia
- **Encadenamiento Opcional (*Optional Chaining*):** Uso adecuado de `err.response?.data?.mensaje` para prevenir excepciones por objetos `undefined` en escenarios donde el servidor está caído.
- **Mensaje de Respaldo (*Fallback*):** Implementación de valor por defecto mediante el operador `||` para dar retroalimentación clara al usuario ante cualquier tipo de falla.

## 4. Lógica de Negocio y Reactividad (`computed`)
- **Rendimiento:** Uso eficiente de `computed` para métricas financieras (`totalIngresos`, `totalEgresos`, `saldo`), aprovechando el almacenamiento en caché de Vue 3.
- **Protección contra `NaN`:** Inicialización explícita con el valor `, 0` en el método `.reduce()`, garantizando estabilidad numérica aunque el array de movimientos esté vacío.

## 5. Formateo y UX en el Formulario
- **Sanitización de Datos:** Conversión explícita del monto a tipo numérico (`Number(...)`) antes de enviar la carga útil (*payload*) al servidor.
- **Actualización Reactiva Inmediata:** Modificación del array reactivo con `.push(respuesta.data.datos)` para reflejar cambios en la UI sin necesidad de realizar peticiones HTTP innecesarias.
