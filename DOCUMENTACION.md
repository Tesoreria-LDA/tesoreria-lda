# Documentación técnica — App de Tesorería (Logia Lucero del Alba)

> **Generado por Claude el 2026-05-20**, a partir de la lectura completa de `index.html` (2.386 líneas).
> Nivel de confianza: alto sobre el contenido del código. Las valoraciones están marcadas como **[Interpretación]**; los hechos verificables, como **[Hecho]**.

---

## 1. Resumen

Aplicación web de tesorería para logias masónicas. Permite gestionar hermanos, cuotas, pagos, gastos, ingresos, deudas, asistencia a tenidas, estados de cuenta y reportes. Está pensada para uso **multi-logia y multi-usuario**: un mismo despliegue sirve a varias logias, cada una con sus propios datos y usuarios.

Es una aplicación de página única (SPA) contenida en un solo archivo `index.html`, escrita en HTML, CSS y JavaScript puro (sin framework), apoyada en los servicios de Firebase para autenticación, base de datos y hosting.

Producción: **https://tesoreria-lda.web.app/**

---

## 2. Arquitectura y tecnologías

| Capa | Tecnología |
|------|-----------|
| Front-end | HTML + CSS + JavaScript puro, en un único archivo. Sin framework ni build |
| Autenticación | Firebase Authentication (email/contraseña), SDK 9.23.0 (compat) |
| Base de datos | Cloud Firestore, SDK 9.23.0 (compat) |
| Hosting | Firebase Hosting |
| Librería externa | SheetJS (`xlsx` 0.20.3) desde CDN, para exportar la nómina a Excel |

El archivo se organiza en tres bloques: estilos CSS (líneas ~10–188), estructura HTML de las vistas (~190–729) y lógica JavaScript (~731–2383).

Configuración de Firebase (constante `FIREBASE_CONFIG`): proyecto `tesoreria-lda`. La `apiKey` de Firebase está en el código; esto es lo normal y esperado en apps web de Firebase — esa clave no es un secreto, la seguridad real la dan las reglas de Firestore y la autenticación.

---

## 3. Hosting y despliegue

- **Proyecto Firebase:** `tesoreria-lda` (definido en `.firebaserc`).
- **`firebase.json`:** sirve toda la carpeta del proyecto como hosting; todas las rutas reescriben a `/index.html`; usa `firestore.rules` para las reglas de la base de datos.
- **Despliegue:** se publica con `firebase deploy` desde la carpeta del proyecto. Requiere el inicio de sesión de Firebase del titular del proyecto.
- **[Hecho]** El logo por defecto se carga desde una URL externa de GitHub Pages (`tesoreria-lda.github.io`). Es una dependencia externa: si esa página dejara de existir, el logo por defecto no cargaría (los logos subidos por logia sí quedan guardados en Firestore).

---

## 4. Autenticación y control de acceso

El acceso es con **correo electrónico y contraseña** mediante Firebase Authentication. Toda la lógica de la aplicación se activa con el evento `onAuthStateChanged`.

Hay tres niveles de acceso:

**Administrador.** Se identifica por un correo concreto: `cguzman@propertyplus.cl` (comparación directa en el código). El administrador ve todas las logias, puede cambiar entre ellas, y accede al menú **Admin** para gestionar logias y usuarios. También ve el módulo **Cobros**.

**Editor.** Usuario con rol `editor` asignado a una o más logias. Puede ver y **guardar** datos en sus logias. Ve el módulo Cobros.

**Viewer.** Usuario con rol `viewer`. Acceso de solo lectura a sus logias asignadas.

El rol y las logias asignadas de cada usuario se leen primero de los *custom claims* del token de Firebase y, si no existen, del documento `users/{uid}` en Firestore. Un usuario con varias logias ve una pantalla para elegir cuál gestionar.

Las reglas de seguridad están en `firestore.rules`: definen `isAdmin()`, `hasLogiaAccess()` y `canWriteLogia()`, de modo que cada usuario solo lee/escribe los datos de las logias que le corresponden.

---

## 5. Almacenamiento de datos

Los datos viven en **Cloud Firestore**, con una caché local en el navegador.

**Estructura en Firestore:**

| Ruta | Contenido |
|------|-----------|
| `apps/{logiaId}` | Un documento por logia. Contiene todos los datos de esa logia (hermanos, pagos, gastos, etc.), más `name`, `logo`, `whatsappGrupo`, `updatedAt`, `lastModifiedBy` |
| `apps/{logiaId}/attachments/{tipo_id}` | Subcolección con los archivos de respaldo (PDF/imágenes en base64), separados del documento principal |
| `users/{uid}` | Permisos del usuario: `email`, `role`, `logias` |

**Caché local:** `localStorage`, con clave `lda_v4_{logiaId}` por logia.

**Sincronización:** la app abre un *listener* en tiempo real (`onSnapshot`) sobre el documento de la logia. Al guardar, hay un guardado diferido (`schedSave`, 1,5 segundos tras el último cambio). La función `shouldUseRemote` compara la marca de tiempo `updatedAt`: gana la versión más reciente. Los archivos adjuntos se guardan aparte para no inflar el documento principal.

**Respaldo manual:** botones "Backup" y "Restaurar" exportan/importan todos los datos como archivo `.json`.

---

## 6. Modelo de datos

Versión del modelo: **v4**. Cada logia guarda estas colecciones dentro de su documento:

| Colección | Qué guarda | Campos principales |
|-----------|-----------|--------------------|
| `members` | Hermanos | id, name, grado, cargo, ingreso (mes), email, celular, fechaNacimiento, activo |
| `fees` | Valores de cuota por período | id, from (mes inicio), monto, desc |
| `exemptions` | Exenciones de pago | id, memberId, from, to, motivo |
| `payments` | Pagos de hermanos | id, memberId, tipo, monto, imputado[], parcial, fecha, metodo, obs, mes, file |
| `gastos` | Gastos de la logia | id, fecha, monto, dest, cat, metodo, aut1/2/3, obs, file |
| `misc` | Ingresos varios | id, fecha, monto, origen, cat, metodo, obs, file |
| `credits` | Crédito a favor por hermano | mapa { memberId: monto } |
| `cobrosEspeciales` | Cargos/devoluciones especiales | id, nombre, fecha, monto, memberIds[] |
| `tenidas` | Tenidas (reuniones) | id, fecha, titulo |
| `asistencias` | Asistencia a tenidas | tenidaId, memberId |

Tipos de pago de hermano: `capita`, `derecho`, `extraordinaria`, `multa`, `donacion`.

---

## 7. Módulos de la aplicación

La barra de navegación da acceso a doce vistas:

**Panel.** Tablero con métricas (hermanos activos, al día, morosos, parciales, saldos, deuda morosa y cuota pendiente del mes), tarjeta de cumpleaños del día, alertas, lista de deudores y últimos movimientos. Tiene un botón para copiar un resumen para la directiva.

**Hermanos.** Alta, edición y eliminación de hermanos; registro de exenciones; filtro por activos/inactivos. Cada hermano tiene un interruptor de activo/inactivo.

**Cuotas.** Define el valor de la cápita por período.

**Registrar pago.** Registra pagos con imputación automática y previsualización.

**Gastos.** Registro de gastos con categoría, hasta tres autorizadores y archivo de respaldo.

**Ingresos varios.** Ingresos que no provienen de cuotas de hermanos.

**Historial.** Vista unificada de todos los movimientos, con filtros, edición/eliminación y exportación a CSV.

**Estado de cuenta.** Estado de cuenta detallado por hermano, imprimible, con notas al pie para los cobros especiales.

**Reportes.** Con seis sub-pestañas: Tesorería (flujo mensual), Por hermano, Gastos (por categoría), Exenciones, **Libro de cuotas** (grilla de todos los hermanos × meses) y **Nómina** (con exportación a Excel e impresión).

**Cobros.** Cargos o devoluciones especiales asignados a varios hermanos a la vez (ej. ágape, derecho de iniciación, ajustes). Solo visible para editor y admin.

**Asistencia.** Tres sub-pestañas: Tenidas (registro de reuniones), Registro (grilla de asistencia) y Resumen (porcentaje de asistencia por hermano).

**Admin.** Solo administrador. Dos sub-pestañas: Gestionar Logias (crear, limpiar, eliminar logias; subir logo; guardar enlace de grupo de WhatsApp) y Gestionar Usuarios (crear usuarios con rol y logias, listar y eliminar).

---

## 8. Lógica de negocio

**Período "año masónico".** Por defecto de marzo a febrero. También hay modo de rango libre.

**Estados de un mes** (`mStatus`): `paid`, `partial` (abono parcial), `late` (moroso), `pending` (aún sin vencer), `exempt`, `na`. Una cuota se considera morosa a partir del día 6 del mes si no está pagada.

**Total adeudado por mes** (`getTotalDue`): cuota del mes (si no está exento) **más** los cobros especiales asignados a ese hermano en ese mes.

**Imputación de pagos** (`computeImp`): un pago de cápita se aplica al mes más antiguo adeudado y avanza hacia adelante, incluyendo el crédito acumulado del hermano. Si el monto cubre solo parte de un mes, se registra como abono parcial. Si sobra dinero sin completar un mes, queda como crédito a favor. Permite pagos adelantados (mira hasta 24 meses).

**Saldos.** El "saldo anterior al período" es la suma histórica de ingresos menos gastos previos al período. El "saldo final acumulado" suma el neto del período.

---

## 9. Cumpleaños e integración con WhatsApp

La app compara el mes-día de `fechaNacimiento` de cada hermano activo con la fecha de hoy. Cuando hay un cumpleaños, muestra una tarjeta dorada en el Panel.

Desde la tarjeta se puede abrir un mensaje de felicitación con dos modalidades — personal (al chat directo del hermano, requiere celular registrado) o al grupo (usa el enlace de grupo de WhatsApp configurado en Admin). Hay tres variantes de redacción, todas con lenguaje masónico, firmadas por "La Secretaría del Taller", y el texto es editable antes de enviar. El envío abre WhatsApp mediante un enlace `wa.me`.

---

## 10. Estado del repositorio git (al 2026-05-20)

- El proyecto es un repositorio git con remoto en GitHub.
- **[Hecho]** El `index.html` actual tiene cambios **sin commitear** (unas 834 líneas de diferencia respecto del último commit `cff3e23`, "Estado estable antes de mejoras", del 2026-04-23). Las mejoras recientes se desplegaron pero no se guardaron en git.
- **[Hecho]** La rama `main` local está 1 commit por delante de `origin/main` (GitHub).
- **[Hecho]** Hay tres ramas worktree (`claude/...`) con versiones más antiguas del archivo.
- **[Interpretación]** Conviene ordenar esto: commitear el trabajo actual y sincronizar con GitHub, para que el historial refleje lo que está realmente desplegado y no se pierda trabajo.

---

## 11. Observaciones y oportunidades de mejora

- **[Hecho]** El logo por defecto depende de una URL externa de GitHub Pages. Se podría incrustar el logo en el propio proyecto para eliminar esa dependencia.
- **[Hecho]** El correo del administrador está fijo en el código (`cguzman@propertyplus.cl`).
- **[Interpretación]** En la función `openCE` (Cobros), el filtro de hermanos usa `!m.baja`, pero el campo que marca a un hermano inactivo es `activo`. El formulario de cobros mostraría también hermanos inactivos. Conviene revisarlo.
- **[Interpretación]** `printLibro()` ejecuta `window.print()` sobre toda la página en lugar de abrir solo el libro en una ventana de impresión, como sí hacen los otros botones de impresión.
- **[Interpretación]** Al editar el monto de una cápita ya registrada, la imputación no se recalcula (el código lo advierte). Es un comportamiento conocido, no un error.
- **[Interpretación]** Posibles mejoras a evaluar con Cristián: reportes en PDF más formales, mejoras de la grilla del Libro de cuotas, validaciones adicionales en formularios.

---

## 12. Cómo se trabaja sobre la app

- **Proyecto:** `/Users/cristianguzman/Documents/tesoreria-lda/`.
- **Archivo principal:** `index.html`.
- **Método acordado con Cristián:** los cambios se hacen primero sobre una copia/rama de trabajo, se muestran y se aprueban antes de desplegar.
- **Despliegue:** Claude (en Cowork) edita el código; el `firebase deploy` lo ejecuta Cristián, o se hace vía Claude Code, porque requiere el inicio de sesión de Firebase.
