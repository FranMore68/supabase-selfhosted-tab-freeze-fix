---
name: fixing-supabase-tab-freeze
description: Resuelve el problema de congelamiento o pérdida de acceso al cambiar de pestaña usando Supabase (especialmente auto-hospedado). Úsalo cuando el usuario mencione bloqueos en Supabase al hacer focus en la ventana o problemas de sesión al cambiar de tab.
---

# Fixing Supabase Tab Freeze

## Cuándo usar este skill

- El usuario reporta que la aplicación se congela al cambiar de pestaña.
- El usuario menciona problemas de timeout o pérdida de sesión con Supabase en entornos self-hosted (GoTrue lento).
- El usuario señala errores relacionados con `visibilitychange` o recarga de foco en pestañas con Supabase.

## Flujo de trabajo

- [ ] 1. Verificar que el middleware **no** esté usando `getUser()` ni `getSession()`. Debe ser estrictamente una comprobación de existencia de cookies (ej. `sb-*`).
- [ ] 2. Eliminar cualquier llamada a `getSession()` dentro de los event listeners de `visibilitychange`.
- [ ] 3. Implementar `supabase.auth.startAutoRefresh()` y `supabase.auth.stopAutoRefresh()` vinculados a la visibilidad de la pestaña en el `AuthProvider` del cliente.
- [ ] 4. Asegurarse de que el cliente de Supabase instanciado en el navegador tenga `autoRefreshToken: true` y `persistSession: true`.
- [ ] 5. Implementar lógica de reintentos en consultas a la base de datos si ocurren justo después del evento `INITIAL_SESSION` (para mitigar el arranque en frío de GoTrue).

## Instrucciones

El problema principal ocurre porque GoTrue en entornos hospedados por cuenta propia puede tardar de 5 a 15 segundos en responder tras un arranque en frío. Si la aplicación intenta validar la sesión o el usuario de forma síncrona al enfocar la pestaña (ej. usando `getSession()`), el hilo de red se bloquea, congelando la aplicación o provocando un bucle infinito de redirecciones si falla por timeout.

Para solucionar la congelación de pestañas y problemas de sesión con Supabase:

- **Middleware**: NUNCA instancies o llames métodos del cliente de Supabase que contacten al servidor (como `getUser()` o `getSession()`). El middleware solo debe comprobar si la cookie de sesión existe y dejar que el cliente o el servidor decida la validez real posteriormente.
- **Event listeners del cliente**: No utilices llamadas bloqueantes en los manejadores de eventos como `visibilitychange` o `focus`.

```javascript
// ✅ IMPLEMENTACIÓN CORRECTA (No bloqueante)
document.addEventListener("visibilitychange", () => {
  if (document.visibilityState === "visible") {
    supabase.auth.startAutoRefresh();
  } else {
    supabase.auth.stopAutoRefresh();
  }
});

// ❌ IMPLEMENTACIÓN INCORRECTA (Bloquea y congela la UI)
document.addEventListener("visibilitychange", async () => {
  if (document.visibilityState === "visible") {
    await supabase.auth.getSession(); // <- ESTO PROVOCA EL CONGELAMIENTO
  }
});
```

- **Manejo de signOut**: Nunca uses `await` con `supabase.auth.signOut()`. Lánzalo al fondo (fire-and-forget) y limpia las cookies manualmente en el cliente.
- **Flujo de sesión**: Ignora el evento `SIGNED_IN` y espera al evento `INITIAL_SESSION` para iniciar cualquier carga de datos de usuario. Hasta ese momento, GoTrue podría no estar listo.

## Recursos

- Rutas de referencia base (si aplican al proyecto):
  - Middleware de cookies: `src/middleware.js` y `src/lib/supabase/middleware.js`
  - Cliente de navegador: `src/lib/supabase.js`
  - Proveedor general: `src/components/AuthProvider.jsx`
