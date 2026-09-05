# Seguridad y Buenas Prácticas – NiCarrera.Test

Este documento resume las medidas de seguridad implementadas en el proyecto y cómo verificarlas.

---

## 1. Validación de entradas

**En el cliente (Phaser / formularios):**
```js
function validarRegistro({ correo, contrasena }) {
  const correoValido = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(correo);
  const contrasenaValida = contrasena.length >= 8;

  if (!correoValido) throw new Error('Correo inválido');
  if (!contrasenaValida) throw new Error('La contraseña debe tener al menos 8 caracteres');

  return true;
}
```

**En la base de datos (Supabase):** además de validar en el cliente, se agregan restricciones a nivel de tabla para que un dato inválido nunca pueda guardarse aunque falle la validación del cliente:

```sql
alter table "Usuarios"
  add constraint correo_formato_valido
  check ("Correo" ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
```

---

## 2. Manejo de errores

Toda llamada a Supabase se maneja con `try/catch` y nunca se asume que la respuesta fue exitosa:

```js
async function guardarIntento(intento) {
  try {
    const { data, error } = await supabase
      .from('Intentos_Evaluaciones')
      .insert([intento]);

    if (error) throw error;
    return data;
  } catch (err) {
    console.error('Error al guardar el intento:', err.message);
    mostrarMensajeAlUsuario('No se pudo guardar tu progreso. Intenta de nuevo.');
  }
}
```

---

## 3. Protección de rutas y datos (roles y permisos)

Se usa **Row Level Security (RLS)** de Supabase apoyada en la tabla `Roles`. Un estudiante solo puede leer/escribir su propia información; un administrador tiene acceso ampliado a los catálogos (preguntas, carreras, universidades).

Las políticas completas están en [`supabase_rls_policies.sql`](./supabase_rls_policies.sql) — deben ejecutarse en el **SQL Editor** del proyecto de Supabase.

---

## 4. Desarrollo seguro

- La `anon key` de Supabase es la única clave usada en el cliente. La `service_role key` **nunca** se expone en el frontend.
- Las credenciales viven en variables de entorno (`.env`), nunca hardcodeadas en el código.
- Todas las consultas se hacen con el cliente oficial de Supabase (`@supabase/supabase-js`), que parametriza las consultas automáticamente y evita inyección SQL.
- Ningún control de acceso depende solo del cliente: aunque alguien manipule el juego, RLS bloquea el acceso a nivel de base de datos.

---

## 5. Autenticación de dos factores (2FA)

Supabase Auth soporta MFA por TOTP (Google Authenticator, Authy, etc.). Recomendación: activarlo como obligatorio para cuentas de **Administrador**, y opcional para estudiantes (para no agregar fricción a usuarios jóvenes).

```js
// Inscribir un factor de doble autenticación
const { data, error } = await supabase.auth.mfa.enroll({ factorType: 'totp' });

// El usuario escanea data.totp.qr_code con su app de autenticación,
// y luego se verifica con el código generado:
const { data: verifyData, error: verifyError } = await supabase.auth.mfa.challengeAndVerify({
  factorId: data.id,
  code: '123456' // código ingresado por el usuario
});
```

Esto se activa desde el panel de Supabase en **Authentication → Providers → Enable MFA**, y luego se agrega la pantalla de inscripción en el flujo de login del administrador.

---

## 6. Manejo de estados (expiración de sesión)

Supabase renueva el token automáticamente mientras la app esté activa. Se debe escuchar el evento de cierre de sesión por expiración para redirigir al login:

```js
supabase.auth.onAuthStateChange((event, session) => {
  if (event === 'SIGNED_OUT' || !session) {
    // La sesión expiró o el usuario cerró sesión
    mostrarPantallaDeLogin();
  }
});
```

En la configuración del cliente, mantener activo el auto-refresh:

```js
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  auth: {
    autoRefreshToken: true,
    persistSession: true,
  }
});
```

---

## 7. Cómo verificar que funciona (evidencia real)

1. Ejecutar `supabase_rls_policies.sql` en el SQL Editor de Supabase.
2. Crear dos usuarios de prueba con rol "Estudiante".
3. Iniciar sesión con el Usuario A e intentar leer un `Intento_Evaluacion` del Usuario B directamente por API — debe devolver vacío o error de permisos.
4. Activar MFA en una cuenta de administrador y confirmar que pide el código TOTP al iniciar sesión.
