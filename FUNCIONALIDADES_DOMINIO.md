# Funcionalidades del Dominio (Notificaciones, Derivaciones, Informes, Medidas)

Este documento describe, con lenguaje natural y siguiendo el flujo real del código, las funcionalidades más importantes del sistema:

- Notificaciones (REST + WebSocket)
- Derivaciones entre unidades (DIRGEGEN, Autoridad/VRA, Campo Clínico)
- Informes Técnicos (crear/editar/pre-carga y envío)
- Medidas de Resguardo (solicitud de medida y su resolución por Informe Técnico)

---

## 1. Notificaciones (cómo se generan, cómo se reciben y cómo se marcan)

Las notificaciones cumplen dos roles:

1. **Notificar en tiempo real** (Socket.io) para avisar al usuario cuando un evento relevante ocurre.
2. **Persistir** en base de datos para que el usuario pueda ver el historial al entrar (REST).

### 1.1 Dónde se guardan (BD)

Las notificaciones se crean en el backend con Prisma contra la entidad `notificacion` (en `backend/src/services/notificacion.service.js`).

La función principal es:

- `crearNotificacion(datos, io)`
  - crea el registro en BD (`prisma.notificacion.create`)
  - opcionalmente envía email (`enviarEmail`)
  - si existe `io`, emite un evento por WebSocket al usuario

### 1.2 Notificaciones por WebSocket (Socket.io)

El servidor de Socket.io se inicializa en `backend/src/socket/socket.js`:

- el servidor autentica la conexión validando el JWT (busca token en `handshake.auth.token`, luego `Authorization`, y finalmente en cookies)
- si el token es válido, asigna:
  - `socket.userId = decoded.id`
  - `socket.userRoles = decoded.roles`
- al conectarse, el usuario entra a una “sala”:
  - `user_${socket.userId}`

Cuando el backend llama `crearNotificacion(..., io)`:

- emite `nueva_notificacion` a la sala `user_${ID_Usuario}`

Payload (lo que llega al frontend) incluye (según `crearNotificacion`):

- `id` (ID_Notificacion)
- `tipo`, `titulo`, `mensaje`
- `fecha`, `leida`
- `denunciaId`
- `urlDestino` (si aplica)

### 1.3 Notificaciones por REST (consulta + marcado)

Las rutas REST están en `backend/src/routes/notificaciones.routes.js` y todas requieren autenticación (`verifyToken`):

- `GET /api/notificaciones` → lista (controlado por `getNotificaciones`)
- `GET /api/notificaciones/contador` → contador no leídas
- `PATCH /api/notificaciones/:id/leida` → marca como leída
- `PATCH /api/notificaciones/marcar-todas` → marca todas como leídas

En `backend/src/controllers/notificacion.controller.js`:

- `getNotificaciones` usa `req.user.id` para asegurar que la consulta sea del usuario autenticado
- `marcarComoLeida` en el service hace seguridad adicional:
  - solo actualiza si `ID_Notificacion` coincide y `ID_Usuario` es el dueño

### 1.4 Cómo lo ve el usuario (frontend)

En el frontend:

- `frontend/src/components/layout/Header.tsx` renderiza `<Notificaciones />` solo si `isAuthenticated`
- `frontend/src/components/Notificaciones.tsx`:
  1. al montar:
     - carga `getNotificaciones({ limit: 10 })`
     - carga `getContadorNoLeidas()`
  2. si hay socket:
     - escucha `socket.on('nueva_notificacion', ...)`
     - agrega la notificación al estado local y aumenta el contador
  3. al hacer click:
     - marca como leída con REST
     - navega a `urlDestino` (si viene) o calcula un fallback con el rol

---

## 2. Derivaciones entre unidades (cómo se cambia tipo/estado y cómo se notifica)

Derivar una denuncia significa:

1. Cambiar su **tipo** (`ID_TipoDe`) y su **estado** (`ID_EstadoDe`).
2. Registrar historial cuando cambia el estado.
3. Disparar notificaciones (y email) a los usuarios “destino” para que puedan gestionar el caso.

### 2.1 Derivación desde la UI: endpoint “gestionar”

En el frontend, la derivación usada en pantallas reales llama siempre a `gestionarDenuncia` (ver `frontend/src/services/denuncias.api.ts`):

- `PATCH /api/denuncias/:id/gestionar`

Parámetros que envía:

- `observacion` (obligatoria en UI)
- `nuevoEstadoId` (ej. `3` como estado “Derivada”)
- `nuevoTipoId` (ej. `300`, `301`, `302`, `303`)

### 2.2 backend: ruta protegida y validación

La ruta está en `backend/src/routes/denuncias.routes.js`:

- `PATCH /:id/gestionar`
- usa `hasRole(['Dirgegen', 'VRA', 'VRAE', 'VRIP', 'CampoClinico'])`
- ejecuta `updateDenuncia` (controller) con validaciones `idParamRule`

### 2.3 backend: lógica real de derivación (disparadores de notificación)

La lógica vive en `backend/src/services/denuncia.service.js` dentro de:

- `updateDenunciaService(id, data)`

Después de actualizar campos base, guardar historial y actualizar participantes/evidencias si corresponde, la derivación “dispara” notificaciones cuando:

- hay `data.observacion` (nota de derivación)
- y el `nuevoTipoId` cae en estos casos:
  - `303` → deriva a **DIRGEGEN**
  - `301` o `302` → deriva a **Autoridad (VRA/VRAE)**
  - `300` → deriva a **Campo Clínico**

Para cada destino, el service busca destinatarios por `Tipo_PC` en `participante_Caso`, crea notificaciones con:

- `Tipo: "DENUNCIA_DERIVADA"`
- `Titulo: ...`
- `Mensaje` con la observación
- `ID_Denuncia`
- `enviarEmail: true`
- `urlDestino`:
  - `/dirgegen/denuncia/:id` para `303`
  - `/autoridad/denuncia/:id` para `301/302`
  - `/campo-clinico/denuncia/:id` para `300`

### 2.4 Derivación “legacy”/alternativa: DIRGEGEN /gestion/denuncias/:id/derivar

Existe además una ruta específica en DIRGEGEN:

- `PATCH /api/gestion/denuncias/:id/derivar`
- en `backend/src/routes/dirgegen.routes.js`
- que llama `derivarDenuncia` → `derivarDenunciaService`

Esta ruta:

- valida `nuevoTipoId`, `nuevoEstadoId` y `observacion` (ver `backend/src/validations/dirgegen.validation.js`)
- actualiza campos en BD
- y notifica destinatarios con la misma idea (pero mediante otra implementación en `backend/src/services/dirgegen.service.js`)

Nota de contexto: en las pantallas actuales del frontend que vimos, la derivación se hace con `gestionarDenuncia` (endpoint `/denuncias/:id/gestionar`). La ruta “/gestion/denuncias/:id/derivar” queda disponible como alternativa.

---

## 3. Informes Técnicos (crear/editar/pre-carga + envío a VRA)

Los Informes Técnicos son el “documento oficial” que:

- consolida análisis (social, psico, jurídico, etc.)
- propone “Medidas de Resguardo”
- y cambia estados para habilitar derivación hacia VRA.

### 3.1 Endpoints de informes técnicos

En `backend/src/routes/informeTecnico.routes.js`:

- `POST /api/informes-tecnicos` → `crearInformeTecnico`
- `GET /api/informes-tecnicos/pre-carga/:idDenuncia` → `preCargarInforme` (requiere autenticación)
- `GET /api/informes-tecnicos/:idDenuncia` → `obtenerInforme`
- `PUT /api/informes-tecnicos/:idDenuncia` → `actualizarInformeTecnico`

### 3.2 Qué guarda el informe (y qué cambia en el caso)

En `backend/src/services/informTecnico.service.js`:

1. **Crear informe (`createInforme`)**
   - verifica que exista la denuncia
   - verifica que aún no exista un informe (1 a 1 por `ID_Denuncia`)
   - en una transacción:
     - crea `informeTecnico` con campos:
       - `Antecedentes`
       - `Analisis_Social`
       - `Analisis_Psico`
       - `Analisis_Juridico`
       - `Sugerencias` (recomendaciones de medidas de resguardo)
       - conecta con `denuncia` y con `autor`
     - actualiza la denuncia:
       - `ID_EstadoDe = 3`
     - actualiza solicitudes de medida:
       - todas las `solicitudMedida` para esa denuncia con `Estado = 'Pendiente Informe'`
       - pasan a `Estado = 'En Informe Técnico'`

2. **Actualizar informe (`updateInforme`)**
   - actualiza los mismos campos de texto en el informe existente.

### 3.3 Pre-carga y permisos (para llenar/editar el informe)

En `preCargarInforme` (controller `backend/src/controllers/informeTecnico.controller.js`):

- valida `req.user?.id` (si no hay sesión → 401)
- consulta roles del usuario en `participante_Caso`
- si el usuario es `CampoClinico`:
  - valida que la denuncia corresponda al área de Campo Clínico
  - si no, responde `403`
- si pasa permisos, llama a `preCargarDatosInforme` (service) y retorna:
  - identificación (rut/nombre/correo/teléfono/carrera/estamento/fechas)
  - info de si existe informe o no

En el frontend, `frontend/src/pages/Dirgegen/components/InformeTecnicoModal.tsx`:

- vuelve a validar permisos en UI (extra)
- llama a `preCargarDatosInforme(idDenuncia)`
- si existe informe: modo edición
- si no existe: modo emisión (crear)

### 3.4 Enviar el caso desde DIRGEGEN a VRA

En `backend/src/controllers/denuncia.controller.js` existe:

- `enviarInformeAVra(req, res, next)`

La ruta está en `backend/src/routes/denuncias.routes.js`:

- `POST /api/denuncias/:id/enviar-informe-vra`
- solo para `Dirgegen`

Lógica:

1. obtiene la denuncia y revisa que tenga `informe_tecnico`
2. busca el estado “En Revisión” en `estado_Denuncia`
3. actualiza la denuncia:
   - `ID_EstadoDe` a “En Revisión”
   - `ID_TipoDe` a `100` (Género y Equidad)

En `frontend/src/pages/Dirgegen/DetalleDirgegen.tsx`:

- el botón “Enviar a VRA” está deshabilitado si `!tieneInforme`
- al confirmar:
  - llama `enviarInformeAVra(idCaso)`
  - vuelve a bandeja

### 3.5 Observación importante (no “guardada”)

Aunque el frontend controla permisos, los endpoints `POST /api/informes-tecnicos` y `PUT /api/informes-tecnicos/:idDenuncia` **no aparecen protegidos con `verifyToken`** en `informeTecnico.routes.js` (solo `pre-carga` usa `verifyToken`).

En un análisis completo, este punto debe verificarse como riesgo: el backend podría estar permitiendo creación/edición sin autenticación, dependiendo del montaje real del router.

---

## 4. Medidas de Resguardo (solicitud por la víctima y resolución con Informe Técnico)

Aquí el sistema maneja “medidas de resguardo” en forma de **solicitud**.

La idea del flujo es:

1. La víctima solicita una medida con su justificación.
2. Si corresponde, la solicitud queda en espera para que DIRGEGEN elabore un informe técnico urgente.
3. Al emitir el informe técnico, la solicitud cambia de estado (de pendiente → en informe técnico).
4. Luego el caso puede seguir el flujo de derivaciones.

### 4.1 Crear solicitud de medida (víctima / denunciante)

Endpoint en `backend/src/routes/solicitudMedida.routes.js`:

- `POST /api/solicitudes/medidas`
- requiere `verifyToken`
- llama `createSolicitud` (controller)

Payload que usa el backend (controller → service):

- `ID_Denuncia` (req.body)
- `Tipo_Medida` (req.body)
- `Observacion` (req.body, opcional)

En `backend/src/services/solicitudMedida.service.js`:

1. Busca la denuncia y lee su `ID_TipoDe`
2. Determina estado inicial:
   - si `ID_TipoDe < 200` → `Estado = 'Pendiente Informe'`  
     (requiere DIRGEGEN + Informe Técnico)
   - si `ID_TipoDe >= 200` → `Estado = 'Pendiente Resolución'`
3. Crea `solicitudMedida` con esos campos

Además, si el caso es serie `100` (género; `ID_TipoDe < 200`):

- el service notifica a usuarios DIRGEGEN creando registros en BD con `Tipo = "ALERTA_MEDIDA"`
- este paso se hace con `prisma.notificacion.createMany`
- en el código actual no se usa el emisor de Socket.io, ni se envía email desde esta función (solo crea notificaciones persistentes)

### 4.2 Listar medidas pendientes para DIRGEGEN

Endpoint:

- `GET /api/solicitudes/medidas/pendientes/dirgegen`
- requiere `verifyToken` y rol `Dirgegen`

En `backend/src/services/solicitudMedida.service.js`:

- filtra `Estado = 'Pendiente Informe'`
- ordena por `Fecha_Solicitud` asc
- incluye:
  - denuncia (incluye tipo de denuncia y datos de denunciante)
  - persona del solicitante (víctima)

### 4.3 Cómo se ve en la UI (bandejas y alertas)

En `frontend/src/pages/Dirgegen/BandejaDirgegen.tsx`:

- se cargan en paralelo:
  - `listarDenuncias(...)`
  - `listarMedidasPendientes()`
- si hay medidas pendientes:
  - aparece un banner rojo con “Atención Requerida: Medidas de Resguardo”
  - cada fila muestra `Caso #ID_Denuncia` y `Tipo_Medida`
  - el botón “Gestionar →” navega a `/dirgegen/denuncia/:id`

En `frontend/src/pages/Dirgegen/DetalleDirgegen.tsx`:

- hay alertas que muestran si existen `denuncia.solicitudes_medidas` y aún no existe `denuncia.informe_tecnico`
- esa alerta guía la acción: “use esta información para elaborar el Informe Técnico”

### 4.4 Modal de solicitud (víctima) y estado “temporalmente deshabilitado”

Existe `frontend/src/pages/Denuncias/components/SolicitudMedidaModal.tsx`, que:

- permite seleccionar `Tipo_Medida` de una lista
- pide `Observacion` justificando la necesidad urgente
- llama a:
  - `crearSolicitudMedida(...)` → `POST /api/solicitudes/medidas`

Pero en `frontend/src/pages/Denuncias/DetalleDenuncia.tsx`:

- el modal está presente
- pero hay comentarios y variables que lo dejan “temporalmente deshabilitado” (por defecto `showSolicitudModal=false` y la sección visible está ocultada).

---

## 5. Flujo completo (un caso típico, paso a paso)

Un flujo típico que conecta todas las funcionalidades anteriores:

1. **Se crea una denuncia** (con o sin token temporal según el caso).
2. **La víctima solicita una medida de resguardo**:
   - `POST /api/solicitudes/medidas` crea `solicitudMedida`
   - si el caso es de la serie que requiere DIRGEGEN (`ID_TipoDe < 200`), queda `Estado = 'Pendiente Informe'`
3. **DIRGEGEN detecta solicitudes pendientes**:
   - `GET /api/solicitudes/medidas/pendientes/dirgegen` alimenta banner de bandeja
   - además el caso puede mostrar alerta dentro del detalle si aún no existe informe
4. **DIRGEGEN elabora el Informe Técnico**:
   - `pre-carga` llena identificación
   - `POST /api/informes-tecnicos` crea el informe y:
     - actualiza `denuncia.ID_EstadoDe = 3`
     - actualiza `solicitudMedida.Estado` → `En Informe Técnico`
5. **DIRGEGEN envía el caso a VRA**:
   - `POST /api/denuncias/:id/enviar-informe-vra`
   - solo si existe `informe_tecnico`
   - deja el caso en “En Revisión” y con `ID_TipoDe = 100`
6. **Luego ocurren derivaciones posteriores** (si corresponde):
   - mediante `gestionarDenuncia` se cambia tipo/estado y se disparan notificaciones a los roles destino.

---

## 6. Observaciones / “no ocultar nada” (brechas o comportamientos a revisar)

1. **Informes técnicos (riesgo potencial de autorización)**
   - en `backend/src/routes/informeTecnico.routes.js`:
     - `POST /` y `PUT /:idDenuncia` no se ven protegidos con `verifyToken`
   - el frontend controla permisos, pero el backend debería ser la última barrera.

2. **Solicitudes de medida notifican por BD, no necesariamente por WS**
   - `createSolicitudService` crea notificaciones con `prisma.notificacion.createMany`
   - no emite por Socket.io ni envía email desde esa función
   - por eso puede que el usuario no vea la alerta en vivo hasta recargar/consultar.

3. **Derivación: dos implementaciones (una en servicio general y otra legacy DIRGEGEN)**
   - la derivación “activa en UI” usa `PATCH /denuncias/:id/gestionar`
   - existe ruta legacy `/gestion/denuncias/:id/derivar` que usa `dirgegen.service.js`
   - son parecidas, pero no idénticas. Conviene mantener consistencia.

