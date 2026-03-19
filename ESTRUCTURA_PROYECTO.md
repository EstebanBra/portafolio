# Estructura del Proyecto (Frontend y Backend)

Este documento describe la **estructura real** del repo en `backend/` y `frontend/` y el **rol típico** de cada carpeta/archivo.

> Nota: varias implementaciones concretas (por ejemplo, la lógica exacta de cada controller/service) pueden evolucionar. La responsabilidad descrita aquí refleja lo que se ve por nombres, por el routing y por puntos de entrada leídos (auth/denuncias/storage).

---

## 1. Vista general (Arquitectura)

### Frontend (`frontend/`)
- UI en **React + TypeScript + Vite**
- Navegación con **react-router-dom**
- Autenticación basada en **cookies** emitidas por el backend (JWT en cookie `token`)
- Renderiza pantallas por rol (Dirgegen, Autoridad, Revisor, Campo Clínico, Admin) usando:
  - `ProtectedRoute` (rol)
  - `RequireAuth` (autenticación)
- Comunicación con el backend mediante:
  - `frontend/src/services/*` (wrappers de endpoints)
  - **WebSocket** (Socket.io-client) vía `useSocket`
- Gestión de archivos (evidencias) usando endpoints “presigned” de storage.

### Backend (`backend/`)
- API en **Express** (ESM: `type: module`)
- Persistencia en **SQL Server** mediante **Prisma**
- Storage de evidencias en **MinIO/S3** (URLs presigned)
- Autenticación:
  - Login clásico: `POST /api/auth/login`
  - Login ClaveÚnica: flujo con `GET /api/auth/claveunica/*`
  - Sesión: JWT en cookie httpOnly
- Notificaciones en tiempo real con **Socket.io**

---

## 2. Estructura del Backend (`backend/`)

### Árbol de directorios (alto nivel)

```text
backend/
  prisma/
    schema.prisma
    seed.js
    migrations/
  src/
    config/
    controllers/
    routes/
    services/
    validations/
    middlewares/
    socket/
    utils/
  index.js
  package.json
  Dockerfile
  eslint.config.js
  .env.example
  .gitignore
  .dockerignore
```

---

## 2.1 Backend: raíz

### `backend/index.js`
- Punto de entrada de Express.
- Configura CORS + middlewares (JSON, cookie-parser, morgan).
- Monta:
  - `/api/auth` → `src/routes/auth.routes.js`
  - `/api` → `src/routes/index.routes.js`
- Inicializa Prisma (ping), ejecuta `prisma/seed.js` (setup inicial) y inicializa bucket MinIO.
- Inicializa Socket.io y publica el servidor en el `PORT` configurado.

### `backend/package.json`
- Dependencias principales (Express, Prisma, Socket.io, AWS SDK S3 presigner para MinIO).
- Scripts:
  - `dev` (watch sobre `index.js`)
  - `start`
  - `lint`
  - `format`
  - `validate`
  - `seed`: `node prisma/seed.js`

### `backend/Dockerfile`
- Imagen Docker para levantar el backend.

### `backend/eslint.config.js`
- Configuración de ESLint.

### `backend/.env.example`, `backend/.gitignore`, `backend/.dockerignore`
- Variables de entorno esperadas y reglas de inclusión/exclusión.

---

## 2.2 Backend: `backend/prisma/`

### `backend/prisma/schema.prisma`
- Modelo de datos Prisma: tablas/relaciones/enums del dominio (denuncias, personas, roles, estados, etc.).

### `backend/prisma/seed.js`
- Seed inicial: crea estados/tipos y usuarios de prueba.
- Se ejecuta desde `backend/index.js` en el arranque.

### `backend/prisma/migrations/*/migration.sql`
- Migraciones versionadas del esquema (cambios incrementales).

### `backend/prisma/migrations/migration_lock.toml`
- Estado interno del motor de migraciones.

### `backend/prisma/prisma.config.ts`
- Config del tooling de Prisma (según uso del proyecto).

### `backend/prisma/Untitled`
- Archivo suelto que parece artefacto/placeholder. Si no se usa, conviene evaluarlo para limpieza.

---

## 2.3 Backend: `backend/src/`

### `backend/src/config/` (config)

- `backend/src/config/prisma.js`
  - Instancia única de `PrismaClient`.
  - Ajusta logs y desconexión al terminar.
- `backend/src/config/auth.config.js`
  - Constantes de autenticación (JWT, cookie name, expiraciones, redirect por roles, etc.).
- `backend/src/config/claveunica.config.js`
  - Constantes del flujo ClaveÚnica (urls, scope, credenciales/redirects).
- `backend/src/config/email.config.js`
  - Configuración de nodemailer para el envío de emails (p.ej. link de seguimiento).

### `backend/src/controllers/` (capa HTTP)

Responsables de recibir `req/res`, ejecutar validaciones (si aplica), llamar servicios y responder JSON.

- `auth.controller.js`
  - `login` (RUT/password) → set cookie JWT + retorna `user`
  - `logout`
  - `claveunicaLogin`, `claveunicaCallback`, `claveunicaLogout`
  - `me`: devuelve usuario + roles (desde token y/o desde BD)
- `denuncia.controller.js`
  - CRUD y acciones de “denuncias”: listar, obtener por ID, crear, actualizar, eliminar, cambiar estado
  - Acciones de derivación/envío de informe según rol
  - Subir evidencia adicional (`subirEvidenciaDenuncia`)
  - Obtener por token UUID (seguimiento público) → retorna datos “del denunciante” sin sensibles
- `dirgegen.controller.js`
  - Endpoints específicos del rol “Dirgegen” (según servicios/derivación del dominio).
- `informeTecnico.controller.js`
  - Endpoints de creación/gestión de “Informe Técnico”.
- `solicitudMedida.controller.js`
  - Endpoints de “Solicitud de Medida” y su seguimiento.
- `verificacionEmail.controller.js`
  - Endpoints para verificación de email.
- `user.controller.js`
  - Endpoints administrativos o de usuario (según servicios).
- `datosExternos.controller.js`
  - Endpoints que consumen datos externos unificados (integraciones de terceros).
- `notificacion.controller.js`
  - Endpoints relacionados con notificaciones (lectura/creación según rol).
- `storage.controller.js`
  - Endpoints presigned para subir/descargar y eliminar archivos en MinIO:
    - `/presigned-upload`
    - `/presigned-download/:objectKey`
    - `DELETE /:objectKey`

### `backend/src/routes/` (routing HTTP)

Define el árbol de endpoints Express.

- `index.routes.js`
  - Monta sub-rutas:
    - `/denuncias`, `/gestion`, `/solicitudes`, `/informes-tecnicos`, `/notificaciones`, `/storage`, `/datos-externos`, `/verificacion-email`, `/users`
- `auth.routes.js`
  - `/login`, `/logout`, flujo `/claveunica/*`, `/me` (con `verifyToken`)
- `denuncias.routes.js`
  - Endpoints HTTP del módulo denuncias (controladores `denuncia.controller.js` y relacionados).
- `dirgegen.routes.js`
  - Endpoints HTTP del módulo Dirgegen.
- `informeTecnico.routes.js`
  - Endpoints HTTP de informes técnicos.
- `solicitudMedida.routes.js`
  - Endpoints HTTP de solicitudes de medida.
- `notificaciones.routes.js`
  - Endpoints HTTP de notificaciones (vista/listado/estado).
- `storage.routes.js`
  - Rutas de presigned URLs para MinIO (subir/descargar/eliminar).
- `datosExternos.routes.js`
  - Rutas para obtener/consultar datos externos.
- `verificacionEmail.routes.js`
  - Rutas de verificación de email.
- `users.routes.js`
  - Rutas para gestión de usuarios (admin).

### `backend/src/services/` (lógica de negocio)

Servicios reutilizables que encapsulan lógica de dominio e integraciones.

- `denuncia.service.js`
  - Lógica completa de listados y CRUD de denuncias (incluye relaciones y “fabricar” el denunciante como participante cuando corresponde).
  - Incluye lógica de transacciones y notificación posterior (p.ej. al crear una denuncia notifica a Dirgegen/VRA según flujo).
- `storage.service.js`
  - Implementación MinIO/S3:
    - inicializar bucket
    - validar MIME y tamaño
    - generar nombre único de objeto
    - generar URLs presigned de PUT/GET
    - upload directo desde buffer (si se usa)
    - delete + fileExists + getFileMetadata
  - Importante: inyecta prefijo `/storage` para compatibilidad con Nginx sin romper firma
- `auth/claveunica`:
  - `claveunica.service.js`: intercambio code→token y extracción RUN/nombre.
- `datosExternos.service.js`
  - Obtención/unificación de datos externos para alimentar login/derivaciones.
- `dirgegen.service.js`
  - Lógica específica del rol Dirgegen.
- `informTecnico.service.js`
  - Lógica de negocio del informe técnico (creación/actualización).
- `solicitudMedida.service.js`
  - Lógica de negocio para solicitudes de medida.
- `verificacionEmail.service.js`
  - Lógica de verificación de email.
- `user.service.js`
  - Lógica para usuarios.
- `notificacion.service.js`
  - Lógica de notificaciones (persistencia y/o emisión por WebSocket).

### `backend/src/validations/` (validación de inputs)

Usa `express-validator` para validar `req.body` y params.

- `user.validation.js`
- `denuncia.validation.js`
- `dirgegen.validation.js`
- `verificacionEmail.validations.js`
- `datosExternos.validations.js`

### `backend/src/middlewares/` (middlewares)

- `auth.middleware.js`
  - `verifyToken`: valida JWT desde cookie/header y agrega `req.user`.
- `parseFormData.middleware.js`
  - Normaliza/parsea `form-data`/payload para endpoints que reciben formularios.
- `upload.middleware.js`
  - Middleware de subida (normalmente usando `multer`) para adjuntos.

### `backend/src/socket/`

- `socket.js`
  - Inicialización Socket.io y configuración de eventos.

### `backend/src/utils/`

- `json.utils.js`
  - Helpers como `serializeBigInt` (para que BigInt sea JSON serializable).

---

## 3. Estructura del Frontend (`frontend/`)

### Árbol de directorios (alto nivel)

```text
frontend/
  public/
  src/
    app/
    components/
    context/
    hooks/
    pages/
    services/
    types/
    utils/
    styles/
  index.html
  vite.config.ts
  Dockerfile
  package.json
  eslint.config.js
```

---

## 3.1 Frontend: raíz

### `frontend/index.html`
- HTML base con el contenedor `#root`.

### `frontend/vite.config.ts`
- Configuración del dev server/build (proxy, aliases, plugins, etc.).

### `frontend/Dockerfile`
- Imagen Docker para levantar el frontend.

### `frontend/package.json`
- Dependencias:
  - React 19, react-router-dom, axios, socket.io-client
  - tailwind (v4), exceljs, jspdf, file-saver
- Scripts de `dev`, `build`, `lint`, `format`, `validate`.

### `frontend/tsconfig*.json`, `frontend/eslint.config.js`, `frontend/nginx.conf`
- Config TS/ESLint y configuración Nginx para despliegue.

---

## 3.2 Frontend: `frontend/src/`

### `frontend/src/main.tsx`
- Punto de entrada: monta `RouterProvider` con el router definido en `src/app/router.tsx`.
- Importa `styles/tailwind.css`.

### `frontend/src/App.tsx`
- Composición de layout:
  - Renderiza `AppShell`
  - Incluye un `<Outlet />` donde caen las páginas

### `frontend/src/app/router.tsx`
- Define rutas:
  - Grupo “AUTH” (sin Header/Footer): `/login`, `/directo/*`
  - Grupo “APP” (protegido): `RequireAuth` + `ProtectedRoute` por roles

--- 

## 3.3 Frontend: `frontend/src/context/`

### `frontend/src/context/AuthContext.tsx`
- Proveedor de estado de autenticación:
  - `user`, `loading`
  - `login` (RUT/password) → llama a `services/auth.api.ts`
  - `logout` (RUT/password)
  - `loginClaveUnica` y `logoutClaveUnica` (redirect hacia endpoints backend)
  - `checkAuth()` llama `getMe()` y normaliza roles como array
  - `hasRole(role)` helper

---

## 3.4 Frontend: `frontend/src/hooks/`

- `frontend/src/hooks/useAuth.ts`
  - Hook para consumir `AuthContext`.
- `frontend/src/hooks/useSocket.ts`
  - Conecta Socket.io solo si hay auth válida.
  - Obtiene token desde `document.cookie` (nombre `token`).
  - Usa `path: '/api/socket.io'` para compatibilidad con Nginx/proxy.

---

## 3.5 Frontend: `frontend/src/services/` (acceso a API)

- `frontend/src/services/api.ts`
  - Función `http()` helper encima de `apiClient` (axios).
- `frontend/src/services/api.client.ts`
  - Config central de axios (baseURL, interceptors, etc., según el proyecto).
- `frontend/src/services/routes.ts`
  - Constantes con rutas/paths de backend.

Wrappers por dominio:
- `frontend/src/services/auth.api.ts`
  - `login`, `logout`, `getMe`.
- `frontend/src/services/users.api.ts`
  - Endpoints relacionados a usuarios/admin.
- `frontend/src/services/denuncias.api.ts`
  - Endpoints del módulo denuncias.
- `frontend/src/services/notificaciones.api.ts`
  - Endpoints de notificaciones.
- `frontend/src/services/datosExternos.api.ts`
  - Endpoints para datos externos.
- `frontend/src/services/verificacionEmail.api.ts`
  - Endpoints de verificación email.
- `frontend/src/services/informeTecnico.api.ts`
  - Endpoints de informe técnico.
- `frontend/src/services/digergen.apis.ts`
  - Endpoints específicos del módulo “digergen” (según naming del proyecto).

Exportación/generación de archivos:
- `frontend/src/services/excelExport.service.ts`
  - Exporta listas a Excel (exceljs).
- `frontend/src/services/denunciaPdf.service.ts`
  - Generación/descarga PDF de denuncia (jspdf).
- `frontend/src/services/dirgegenPdf.service.ts`
  - Generación/descarga PDF en flujo Dirgegen.

---

## 3.6 Frontend: `frontend/src/utils/`

- `frontend/src/utils/date.utils.ts`
  - Utilidades de fechas.
- `frontend/src/utils/validation.utils.ts`
  - Helpers de validación.
- `frontend/src/utils/alertStore.ts`
  - Store/estado para alertas.

---

## 3.7 Frontend: `frontend/src/types/`

- `frontend/src/types/denuncia.types.ts`
  - Tipos TS de `Denuncia`.
- `frontend/src/types/denunciante.types.ts`
  - Tipos TS de `Denunciante`/participantes.
- `frontend/src/types/step-props.ts`
  - Tipos de props para los “steps” de formularios.

---

## 3.8 Frontend: `frontend/src/components/` (UI reusable)

### `frontend/src/components/layout/` (layout)
- `AppShell.tsx`
  - Layout global con `Header` + contenedor + `Footer`.
- `AuthShell.tsx`
  - Layout centrado para rutas “públicas” del flujo auth (sin header/footer).
- `Header.tsx`
  - Barra superior con navegación/estado (según el rol).
- `Footer.tsx`
  - Pie de página.

### `frontend/src/components/ui/` (componentes genéricos)
- `Modal.tsx`
  - Modal base reutilizable.
- `Cards.tsx`
  - Cards/presentación reutilizable.
- `InfoTooltip.tsx`
  - Tooltip informativo.
- `Alerta.tsx`
  - Componente de alerta/feedback.

### `frontend/src/components/modals/` (modales de dominio)
- `ModalDetalleDenunciado.tsx`
  - Modal para ver/editar detalle de denunciado.
- `ModalDetalleTestigo.tsx`
  - Modal para ver/editar detalle de testigo.

### Componentes sueltos (no necesariamente bajo subcarpetas)
- `ProtectedRoute.tsx`
  - Guard por rol (usa `user.roles` y renderiza `<Outlet />`).
- `RequireAuth.tsx`
  - Guard base (si no hay usuario → redirige a `/login`).
- `AuthShell.tsx` / `AppShell.tsx` (ya cubiertos arriba)
- `EvidenciaViewer.tsx`
  - Visor/selector de evidencias (archivos) para el frontend.
- `FileUploader.tsx`
  - Componente para subir evidencia usando los endpoints de `storage`.
- `Notificaciones.tsx`
  - Componente que muestra notificaciones (idealmente consumiendo WebSocket/endpoint).

---

## 3.9 Frontend: `frontend/src/pages/` (pantallas por ruta)

Cada archivo TSX corresponde a una pantalla navegable por router. La organización refleja módulos/roles.

### `frontend/src/pages/Home.tsx`
- Dashboard / home tras login.

### `frontend/src/pages/Login/`
- `Login.tsx`
  - Pantalla de login (UI principal).
- `LoginForm.tsx`
  - Formulario de “RUT/contraseña” que llama a `services/auth.api.ts`.

### `frontend/src/pages/AccesoDenuncia/`
- `AccesoDenuncia.tsx`
  - Entrada al flujo de denuncia sin login (rutas “directas” del router).
- `index.ts`
  - Re-export de la página.

### `frontend/src/pages/VerificacionEmail/`
- `VerificacionEmail.tsx`
  - Pantalla de verificación de email.
- `index.ts`

### `frontend/src/pages/SeleccionRol/`
- `SeleccionRol.tsx`
  - Selección de rol en flujo directo.
- `index.ts`

### `frontend/src/pages/Denuncias/` (Denunciante)
- `NuevaDenuncia.tsx`
  - Formulario/flujo multi-step para crear una denuncia.
- `MisDenuncias.tsx`
  - Lista de denuncias del usuario.
- `DetalleDenuncia.tsx`
  - Vista de detalle de una denuncia.
- `components/`
  - `FormularioLayout.tsx`: layout común del formulario multi-step
  - `Derivacion.tsx`: acciones/derivación (según flujo)
  - `SolicitudMedidaModal.tsx`: modal para solicitar medida
  - `DeclararAdmisibilidadModal.tsx`: modal de admisibilidad
  - `steps/`:
    - `Paso1Identificacion.tsx`
    - `Paso2Hechos.tsx`
    - `Paso3Confirmacion.tsx`

### `frontend/src/pages/ConfirmacionDenuncia/`
- `ConfirmacionDenuncia.tsx`
  - Pantalla final tras enviar la denuncia (posible link de seguimiento).

### `frontend/src/pages/SeguimientoDenuncia/`
- `SeguimientoDenuncia.tsx`
  - Pantalla pública que muestra estado por `:token`.

### `frontend/src/pages/Dirgegen/`
- `BandejaDirgegen.tsx`
  - Bandeja/listado de denuncias para rol Dirgegen.
- `DetalleDirgegen.tsx`
  - Detalle para resolver/gestionar esa denuncia.
- `components/`
  - `IdentificarDenunciadoModal.tsx`
  - `InformeTecnicoModal.tsx`

### `frontend/src/pages/Autoridad/` (VRA/VRAE/VRIP)
- `BandejaAutoridad.tsx`
  - Bandeja de casos para Autoridad.
- `DetalleAutoridad.tsx`
  - Detalle para gestión.
- `components/`
  - `InstruirInvestigacionModal.tsx`
  - `SolicitudFiscaliaModal.tsx`

### `frontend/src/pages/Revisor/`
- `BandejaRevisor.tsx`
  - Bandeja del rol Revisor.
- `DetalleRevisor.tsx`
  - Detalle del caso para revisión.

### `frontend/src/pages/CampoClinico/`
- `BandejaCampoClinico.tsx`
  - Bandeja para rol Campo Clínico.
- `DetalleCampoClinico.tsx`
  - Detalle y gestión específica.

### `frontend/src/pages/Admin/`
- `GestionUsuarios.tsx`
  - Pantalla admin para gestionar usuarios.

---

## 4. Cómo se conectan los módulos (Flujos clave)

### Autenticación (Frontend ↔ Backend)
1. Login RUT/password:
   - UI (`frontend/pages/Login/*`) llama `services/auth.api.ts` → `POST /api/auth/login`
   - Backend setea cookie httpOnly con JWT
2. Login ClaveÚnica:
   - UI redirige a `GET /api/auth/claveunica/login`
   - Callback en backend crea sesión y redirige a la ruta según roles
3. Estado de sesión:
   - `AuthContext` llama `GET /api/auth/me` con cookie

### Denuncias y evidencias (Datos + Storage)
1. Frontend crea/consulta denuncias:
   - `services/denuncias.api.ts` y demás wrappers consumen los endpoints en `backend/src/routes/denuncias.routes.js`.
2. Evidencias:
   - Frontend pide una URL presigned de subida (storage)
   - Sube directo a MinIO mediante esa URL
   - Luego registra metadatos en BD (controller `subirEvidenciaDenuncia` y/o flujos del módulo)

### WebSocket (notificaciones)
- `useSocket` conecta a Socket.io usando token (desde cookie) y `path: '/api/socket.io'`.
- El backend emite eventos desde `backend/src/socket/socket.js` y/o servicios que llaman a `getIO()`.

---

## 5. Qué buscar para entender el comportamiento real

Si quieres entender “cómo funciona de verdad” (no solo la estructura), los mejores puntos de inicio son:
- Backend:
  - `backend/index.js` (arranque y montaje)
  - `backend/src/routes/index.routes.js` (qué endpoints existen)
  - `backend/src/controllers/auth.controller.js` (auth completo)
  - `backend/src/controllers/denuncia.controller.js` + `backend/src/services/denuncia.service.js` (dominio de denuncias)
  - `backend/src/services/storage.service.js` (presigned y reglas de archivos)
- Frontend:
  - `frontend/src/app/router.tsx` (rutas y roles)
  - `frontend/src/context/AuthContext.tsx` (estado de usuario)
  - `frontend/src/components/ProtectedRoute.tsx` (guard por rol)
  - `frontend/src/services/*` (endpoints concretos)
  - `frontend/src/hooks/useSocket.ts` (WebSocket)

