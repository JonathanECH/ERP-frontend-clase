# Investigación: Programación Asíncrona en Vue 3 y Axios

## 1. Comportamiento ante Servidor Caído (Network Error)
Cuando el servidor backend de Express no está activo o no se puede establecer conexión:
- **Estado de `err.response`:** La propiedad `err.response` es `undefined` porque la petición no llegó a recibir una respuesta HTTP del servidor (ocurre un error a nivel de red o conexión rehusada `ERR_CONNECTION_REFUSED`).
- **Experiencia de Usuario (UX):** Es crucial capturar este escenario dentro del bloque `catch` para mostrar una alerta o mensaje amigable en pantalla (ej: *"No se pudo conectar con el servidor. Verifica que esté encendido."*). Además, gracias al bloque `finally`, se garantiza cambiar el estado `cargando.value = false`, evitando que la interfaz quede congelada con un indicador de carga infinito.

## 2. Diferencia entre Error HTTP 400 vs HTTP 500 / Error de Red
- **Error HTTP 400 (Bad Request):** Indica una validación fallida en la solicitud enviada por el cliente. En Axios, `err.response` **sí existe** (`status === 400`) y contiene el cuerpo de la respuesta enviado por el backend en `err.response.data`. El mensaje específico enviado por Express se extrae con `err.response.data.mensaje`.
- **Error HTTP 500 / Error de Red:** En un error de red o servidor caído, `err.response` es `undefined`. Para evitar que el código lance una excepción al intentar acceder a propiedades de un objeto inexistente (`TypeError: Cannot read properties of undefined`), utilizamos el operador de encadenamiento opcional (*optional chaining*):
  ```javascript
  error.value = err.response?.data?.mensaje || 'Error al procesar la solicitud';
  ```

## 3. Optimización de Peticiones Asíncronas con `Promise.all`
Al cargar un panel de control o vista inicial que requiere múltiples fuentes de datos independientes (por ejemplo, la lista de movimientos y el resumen contable):

### Enfoque Secuencial (Lento):
```javascript
// Espera a que termine la primera petición para iniciar la segunda
const resMovimientos = await movimientoService.getAll(); // ~1000ms
const resResumen = await movimientoService.getResumen();     // ~1000ms
// Tiempo total: ~2000ms
```

### Enfoque Paralelo con `Promise.all` (Optimizado):
```javascript
// Ejecuta ambas peticiones simultáneamente en paralelo
const [resMovimientos, resResumen] = await Promise.all([
  movimientoService.getAll(),
  movimientoService.getResumen()
]);
// Tiempo total: ~1000ms (el tiempo de la promesa más lenta)
```
`Promise.all` recibe un arreglo de promesas y resuelve cuando todas hayan concluido exitosamente, reduciendo significativamente el tiempo de carga percibido por el usuario.
