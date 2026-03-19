# Seguridad, Autenticación y Despliegue

Este documento explica, con lenguaje natural, cómo se organiza la seguridad en el proyecto `Portal-denuncias`, desde que el usuario llega al frontend, cómo se autentica, cómo se validan inputs, cómo se autoriza por roles y cómo se despliega usando Docker y un proxy inverso.

> Enfoque: “de punta a punta”. Si algo no está implementado (por ejemplo CSRF explícito), también se indica para que el panorama sea completo.

---

## 1. Panorama general (capas)

El proyecto separa responsabilidades en capas:

1. **Frontend (`frontend/`)**: UI en React/Vite. Renderiza pantallas por rol y protege rutas con guards del lado del cliente.
2. **Backend (`backend/`)**: API HTTP en Express + lógica de dominio. Autentica, autoriza (roles) y valida inputs con `express-validator`.
3. **Persistencia**: Prisma contra SQL Server.
4. **Evidencias/archivos**: MinIO/S3 compatible con endpoints “presigned”.
5. **Mensajería en tiempo real**: Socket.io para notificaciones.
6. **Despliegue**: Docker y un **proxy**:
   - En desarrollo, Vite hace proxy de `/api` al backend.
   - En producción, Nginx (en el contenedor frontend) proxy a backend `/api` y a MinIO `/storage`.

---

## 2. Autenticación: qué es “la sesión” en este sistema

La autenticación funciona con **JWT en cookie** (no con token en `localStorage`).

### 2.1 Cookie: qué cookie se usa y por qué

En el backend la cookie principal se llama:

- `COOKIE_NAME = 'token'`

El backend arma ese token en el login clásico y también en el flujo de ClaveÚnica.

Ese JWT contiene principalmente:

- `id` (ID numérico de usuario/persona en BD)
- `rut`
- `nombre`
- `roles` (roles/`Tipo_PC` obtenidos desde tabla `participante_Caso`)

El backend fija además propiedades de seguridad en la cookie:

- `httpOnly: true`
- `secure`: depende de si el servidor está en HTTPS o si `COOKIE_SECURE` está en `true`
- `sameSite`: en producción puede quedar en `none` (ver detalle más abajo)
- `path: '/'`
- `maxAge` de 24h

En el backend, la verificación del JWT ocurre en el middleware `verifyToken`.

### 2.2 Verificación del token (backend)

El middleware `verifyToken` (archivo `backend/src/middlewares/auth.middleware.js`) hace esto:

1. Lee el JWT desde la cookie `req.cookies['token']`.
2. Si no existe, responde `401`.
3. Si existe, valida la firma y expiración con `JWT_SECRET` usando `jwt.verify`.
4. Si todo está bien, guarda el payload decodificado en `req.user`.
5. Si falla, responde `401`.

Resultado: cualquier endpoint que pida `verifyToken` no ve “cualquier usuario”; ve exactamente los claims verificados del JWT.

---

## 3. Autenticación: flujos de login disponibles

Hay dos maneras de entrar:

1. **Login clásico (RUT + contraseña)**: `POST /api/auth/login`
2. **ClaveÚnica**: rutas `GET /api/auth/claveunica/login`, `GET /api/auth/claveunica/callback`, `GET /api/auth/claveunica/logout`

### 3.1 Login clásico (RUT + contraseña)

El endpoint `POST /api/auth/login` en `backend/src/controllers/auth.controller.js` hace:

1. Normaliza RUT “limpiando puntos”.
2. Busca persona en Prisma por el campo `Rut`. Si el RUT vino sin guión, intenta `startsWith` para encontrar el formato con guión y DV.
3. Verifica password con `bcrypt.compare`.
4. Recupera roles desde `participante_Caso`:
   - busca `Tipo_PC` por `ID_Persona`
   - convierte eso a un array `roles`
5. Genera el JWT con `jwt.sign({ id, rut, nombre, roles }, JWT_SECRET, { expiresIn: '24h' })`.
6. Seta la cookie `token`.
7. Responde con el objeto `user` (incluye roles, email, etc.).

### 3.2 Login con ClaveÚnica (OIDC)

El flujo ClaveÚnica en backend usa un patrón “estado anti-falsificación” (CSRF) basado en `state`:

1. `claveunicaLogin`:
   - genera un `state` aleatorio (`crypto.randomBytes`)
   - lo guarda en una cookie `claveunica_state` (httpOnly)
   - redirige al usuario hacia `CLAVEUNICA_AUTHORIZE_URL` con el `state` en la query
2. `claveunicaCallback`:
   - recibe `code` y `state` desde ClaveÚnica
   - valida que coincida el `state` guardado en cookie (`req.cookies.claveunica_state`)
   - si no coincide, responde `403` o rechaza (mensaje: “Posible ataque CSRF”)
   - intercambia el `code` por access token (helper `exchangeCodeForToken`)
   - obtiene `userInfo` (RUN y nombre)
   - busca/crea `persona` en Prisma con ese RUN
   - carga roles desde `participante_Caso`
   - si no tiene roles staff, agrega rol `USER` para que pueda usar el portal
   - genera el JWT y setea cookie `token`
   - finalmente redirige al frontend según rol usando:
     - `getRedirectPathForRoles` en `backend/src/config/auth.config.js`

#### Redirección post-login por rol

En `backend/src/config/auth.config.js` existe un mapeo:

- Prioridad de roles: `REDIRECT_ROLE_PRIORITY`
- Ruta por rol: `REDIRECT_PATH_BY_ROLE`

Eso significa: una persona con múltiples roles elige la primera prioridad que aparece.

---

## 4. Seguridad de cookies y CORS (cómo se evita/permite el acceso)

### 4.1 CORS en backend

En `backend/index.js` el backend usa:

- `cors({ origin: FRONTEND_URL || true, credentials: true })`

Puntos importantes:

1. `credentials: true` permite que el navegador envíe cookies en requests cross-origin (si aplica).
2. `origin: FRONTEND_URL || true`:
   - si `FRONTEND_URL` está bien configurado, se restringe el origen.
   - si no lo está (queda `true`), se puede terminar en una configuración demasiado permisiva (depende de la librería y de cómo esté envuelto).

Recomendación práctica: en producción, asegúrate de que `FRONTEND_URL` sea un dominio exacto permitido.

### 4.2 sameSite: la decisión que impacta directamente la seguridad

El backend setea `sameSite` como:

- `'none'` cuando `secure` es true (producción típica con HTTPS)
- `'lax'` cuando `secure` es false

Cuando `sameSite = none`, el navegador puede enviar la cookie incluso en contextos cross-site (esto es necesario si el frontend está en un dominio distinto, pero también exige cuidado con CSRF).

En este proyecto no se ve un middleware CSRF específico para endpoints protegidos por cookie.

Eso no significa que sea “vulnerable automáticamente”, pero sí significa que la protección CSRF no está reforzada explícitamente. El riesgo real depende del despliegue (mismo sitio vs distinto sitio) y del diseño de endpoints.

---

## 5. Autorización por roles: quién puede hacer qué

La autorización existe en dos lugares complementarios:

1. **Backend** (fuente de verdad para seguridad real).
2. **Frontend** (conveniencia de UI; si falla, el backend igual debe denegar).

### 5.1 Backend: verificación por rol

En `backend/src/middlewares/auth.middleware.js`:

- `hasRole(rolesPermitidos)`:
  - asume `req.user.id` desde el JWT
  - consulta `participante_Caso` y obtiene `Tipo_PC` reales
  - autoriza si alguno de esos roles coincide con `rolesPermitidos`
  - si no coincide, responde `403`

Además hay un helper `isAdmin` (aunque no siempre se usa en las rutas).

### 5.2 Rutas protegidas (denuncias)

En `backend/src/routes/denuncias.routes.js` se ve claramente la separación:

1. **Rutas públicas**:
   - `POST /publica` requiere `verifyTemporaryToken` (token temporal de denuncia pública)
   - `GET /seguimiento/:token` para seguimiento público
   - `POST /seguimiento/:token/evidencia` para subir evidencia al caso público
2. **Rutas protegidas**:
   - se aplica `router.use(verifyToken);` y desde ahí se protege el resto
   - dentro, algunos endpoints exigen además `hasRole([...])`

Ejemplos relevantes:

- `PATCH /:id/estado`: autorizado para roles como Autoridad/Fiscal/Dirgegen/VRA/VRAE/VRIP/CampoClinico/REVISOR
- `POST /:id/enviar-informe-vra`: solo `Dirgegen`
- `DELETE /:id`: solo `Admin`

El punto clave: aunque el frontend oculte cosas, el backend decide.

### 5.3 Frontend: guards de UI

En el frontend:

- `RequireAuth` (archivo `frontend/src/components/RequireAuth.tsx`) redirige si no hay usuario autenticado.
- `ProtectedRoute` (archivo `frontend/src/components/ProtectedRoute.tsx`) redirige si el usuario no tiene los roles permitidos.

Importante: esto es UI; no reemplaza la autorización del backend.

---

## 6. Validaciones de inputs: cómo se reduce el riesgo de payloads maliciosos

Este proyecto usa `express-validator` principalmente en:

- `backend/src/validations/*`
- validación ocurre en las rutas y en algunos casos se revalida en controllers con `validationResult(req)`

### 6.1 Validaciones para denuncias

En `backend/src/validations/denuncia.validation.js`:

1. `createDenunciaRules` valida:
   - `Rut` debe ser string, trim, no vacío
   - `ID_TipoDe` int >= 1
   - `Fecha_Inicio` debe ser ISO8601
   - `Fecha_Fin` es opcional y permite `null`, y valida ISO8601 si viene
   - `Relato_Hechos` string no vacío
   - `Ubicacion` opcional string con `maxLength: 200`
2. `updateDenunciaRules` valida el `param("id")` vía `idParamRule` (int >= 1) y el resto de body como opcional:
   - campos numéricos como int >= 1
   - fechas opcionales como ISO8601
   - `observacion` como string
3. `listDenunciasRules` valida query params:
   - `page` y `pageSize` int válidos
   - `tipoId`, `estadoId` ints
   - `desde`, `hasta` ISO8601

### 6.2 Validaciones para verificación email (código)

En `backend/src/validations/verificacionEmail.validations.js`:

- `rut`:
  - no vacío
  - string
- `codigo`:
  - no vacío
  - string
  - longitud exacta 6
  - regex `^\d{6}$` (solo números)

### 6.3 Validaciones para storage (presigned)

Para `backend/src/routes/storage.routes.js`:

- `POST /presigned-upload` valida:
  - `fileName` string no vacío
  - `mimeType` string no vacío
  - `size` int min 1
- `GET /presigned-download/:objectKey` valida `objectKey` string no vacío
- `DELETE /:objectKey` valida `objectKey` string no vacío

Además, el storage controla tipo y tamaño a nivel “de lógica” con:

- `validateFileType`
- `validateFileSize`

### 6.4 Manejo de multipart + JSON (FormData)

Hay un punto importante de “cómo llega la data”:

- `backend/src/middlewares/upload.middleware.js` usa `multer.memoryStorage()` y recibe archivos en el campo `archivos`.
- `backend/src/middlewares/parseFormData.middleware.js` tiene un helper `parseFormDataJson`:
  - si llega `req.body.data` como string JSON, lo parsea
  - reemplaza `req.body` con los datos parseados

Esto permite enviar una sola petición multipart con:

- archivos binarios (`archivos`)
- un JSON con el resto del formulario dentro de `data`

Eso reduce complejidad en frontend y mantiene la validación del backend centralizada.

---

## 7. Subida y acceso a evidencias (MinIO + Presigned URLs)

Aquí hay dos flujos, porque el proyecto soporta:

1. Subida de archivos cuando se crea/actualiza denuncia (multipart upload al backend).
2. Subida “asistida por presigned” cuando se sube evidencia adicional o se usa storage presigned.

### 7.1 Seguridad de upload cuando llega al backend (multer)

En `backend/src/middlewares/upload.middleware.js`:

- Se usa memoria (`multer.memoryStorage()`) para manejar buffers en vez de disco.
- Se limita tamaño con `limits.fileSize = MAX_FILE_SIZE` (200MB).
- `fileFilter` valida tipo con `validateFileType(file.mimetype, file.originalname)`.

Esto aplica a endpoints que incluyen `uploadMultipleFiles`.

### 7.2 Presigned upload/download (storage.service)

El backend expone endpoints presigned en `backend/src/controllers/storage.controller.js` vía rutas `storage.routes.js`:

- `POST /api/storage/presigned-upload`
- `GET /api/storage/presigned-download/:objectKey`
- `DELETE /api/storage/:objectKey`

El corazón está en `backend/src/services/storage.service.js`:

1. Controla:
   - endpoint MinIO
   - bucket `MINIO_BUCKET_NAME` (default: `evidencia-denuncias`)
2. Valida:
   - `ALLOWED_MIME_TYPES` mapeado por MIME y extensiones permitidas
   - máximo tamaño (200MB)
3. Genera un nombre único:
   - usa UUID + sanitización del nombre original
4. Genera URLs firmadas con `getSignedUrl` para:
   - `PutObjectCommand` (subir)
   - `GetObjectCommand` (descargar)
5. Maneja una particularidad del proxy:
   - `replacePresignedUrlEndpoint(url, '/storage')`
   - inyecta el prefijo `/storage` en la URL firmada para que Nginx la enrute
   - sin romper la firma: la firma se calcula contra la ruta “original” que MinIO ve (sin el prefijo).

### 7.3 Nginx: el “puente” entre la URL y MinIO

En `frontend/nginx.conf` existe:

- `location ^~ /storage/`:
  - reescribe `/storage/(.*)` a `/$1`
  - hace `proxy_pass` a `http://minio:9000`

Por eso el storage service mete el prefijo `/storage` en las URLs presigned.

### 7.4 Bucket privado

En `docker-compose.yml` y variantes existe un inicializador (`minio-init` / `minio-init-dev`) que crea el bucket y lo deja privado:

- `mc anonymous set none $$MINIO_ALIAS/$$BUCKET_NAME`

Eso significa: no hay acceso anónimo al contenido del bucket.

Las URLs de acceso pasan por el mecanismo presigned firmado.

---

## 8. Verificación email y tokens temporales (denuncia pública)

Este proyecto permite crear una denuncia desde un flujo “público” (sin sesión staff).

La seguridad de ese flujo se basa en dos pasos:

1. El usuario solicita un código por email.
2. Si el código es correcto, el backend devuelve un **token temporal**.

Luego el frontend usa ese token temporal para permitir `POST /api/denuncias/publica`.

### 8.1 Endpoint de verificación de email

Rutas en `backend/src/routes/verificacionEmail.routes.js`:

- `POST /api/verificacion-email/solicitar`
- `POST /api/verificacion-email/verificar`

### 8.2 Cómo se generan y verifican códigos

En `backend/src/services/verificacionEmail.service.js`:

- Se guarda el estado en un `Map` en memoria:
  - `codigosVerificacion.set(rut, { codigo, email, expiracion, intentos })`
- Expiración:
  - `TIEMPO_EXPIRACION = 5 * 60 * 1000` (5 minutos)
- Límite de intentos:
  - al superar 3 intentos incorrectos, borra el código

Se envía el correo usando `enviarCorreo` de `backend/src/config/email.config.js`.

### 8.3 Token temporal (JWT)

En `backend/src/controllers/verificacionEmail.controller.js`:

- si el código es correcto, crea un JWT temporal con `jwt.sign`.
- ese token incluye:
  - `rut`
  - `tipo: 'denuncia_publica'`
  - `exp`: 30 minutos

Ese token se devuelve como `tokenTemporal`.

### 8.4 Middleware `verifyTemporaryToken`

En `backend/src/middlewares/auth.middleware.js`:

- el endpoint público `/publica` usa `verifyTemporaryToken`
- el token se lee desde `Authorization: Bearer <token>`
- se valida con `jwt.verify`
- además se verifica que `decoded.tipo === 'denuncia_publica'`
- si es válido, agrega:
  - `req.denuncianteVerificado = { rut: decoded.rut }`

Resultado: el flujo público requiere prueba de email con expiración corta.

---

## 9. Seguimiento público por token UUID

Para no requerir login, existe un endpoint:

- `GET /api/denuncias/seguimiento/:token`

En `backend/src/controllers/denuncia.controller.js`:

1. Busca la denuncia en Prisma por `token_seguimiento`.
2. Incluye relaciones (estado, tipo, archivos).
3. Genera URLs presigned para archivos si existen `MinIO_Key`.
4. Devuelve un payload “seguro”:
   - devuelve campos relevantes
   - evita datos sensibles del sistema staff

Este es el mecanismo que usa el link enviado por email cuando se crea una denuncia pública.

---

## 10. Notificaciones y WebSocket (Socket.io)

El proyecto usa Socket.io para notificar eventos en tiempo real (por ejemplo, cuando se deriva o se recibe una denuncia que debe llegar a un rol).

### 10.1 Inicialización y autenticación del WS

La inicialización ocurre en `backend/src/socket/socket.js` dentro de `initializeSocket(server)`.

El servidor configura:

- `path: '/api/socket.io'` para que el WebSocket atraviese el proxy de Nginx bajo `/api`
- `cors` de Socket.io con `credentials: true`

Para autenticar, Socket.io usa un middleware `ioInstance.use((socket, next) => { ... })` que intenta obtener el JWT del token desde varias fuentes, en este orden:

1. `socket.handshake.auth.token` (si el frontend lo pasa explícitamente en `auth`)
2. `socket.handshake.headers.authorization` si viene `Authorization: Bearer <token>`
3. `socket.handshake.headers.cookie` (busca `token=<...>` en el header de cookies)

Luego hace `jwt.verify(token, JWT_SECRET)` y si es válido:

- asigna `socket.userId = decoded.id`
- asigna `socket.userRoles = decoded.roles || []`

Si el token falta o es inválido, la conexión falla.

### 10.2 Salas por usuario y emisión de eventos

Cuando ocurre `connection`, el servidor:

- se une a una sala llamada `user_${socket.userId}`

Desde `backend/src/services/notificacion.service.js`, la función `crearNotificacion(datos, io)`:

- guarda la notificación en BD (`prisma.notificacion.create`)
- si `io` existe, emite por WebSocket:
  - `io.to('user_${ID_Usuario}').emit('nueva_notificacion', payload)`

Por eso el frontend recibe “nuevas notificaciones” solo para usuarios concretos.

### 10.3 Marcado como leída: WS vs REST

En `backend/src/socket/socket.js` existe un handler `socket.on('marcar_leida', ...)`, pero el código actual solo confirma y emite un evento de vuelta; el guardado real de “leída” se hace por REST en:

- `backend/src/controllers/notificacion.controller.js` → `marcarComoLeida` / `marcarTodasLeidas`

En el frontend, `Notificaciones.tsx` al hacer click usa las APIs REST (no usa el evento WS para persistir).

### 10.4 Relación con el frontend (`useSocket`)

El frontend con `useSocket.ts` intenta conectar solo si `useAuth` indica autenticación, y para el `auth.token` intenta leer la cookie `token` desde `document.cookie`.

El backend igualmente es capaz de validar desde el header de cookies en el handshake, pero el gate del frontend (si no tiene token en JS) puede impedir que se inicie la conexión.

En resumen: la seguridad del WS real está del lado del backend (verifica JWT en handshake), pero conviene revisar que `useSocket.ts` conecte efectivamente en todos los entornos.

---

## 11. Dockerización: cómo se despliega y qué seguridad aporta

Hay 3 piezas clave:

1. **Dockerfiles** del backend y frontend
2. **docker-compose** para levantar stack completo
3. **Nginx** como proxy inverso para enrutar `/api` y `/storage`

### 11.1 Backend Dockerfile: enfoque security-by-default

En `backend/Dockerfile`:

- stage `builder`:
  - usa `node:24-alpine3.23`
  - instala dependencias y corre `prisma generate`
- stage de producción:
  - instala solo `dumb-init` y crea imagen más limpia
  - copia `node_modules` ya pruned (production)
  - corre como usuario `node` (no root)
  - define `HEALTHCHECK` y `ENTRYPOINT` con `dumb-init`

En docker-compose además se setea:

- `read_only: true`
- `tmpfs: /tmp`
- `cap_drop: [ALL]`
- `user: 'node'`

Eso reduce la superficie de ataque:

- no se escribe en filesystem (salvo tmpfs)
- se limitan capacidades del kernel

### 11.2 Frontend Dockerfile + Nginx

En `frontend/Dockerfile`:

- stage `builder`:
  - corre `npm run build` (genera `dist`)
- stage `production`:
  - usa `nginx:1.27-alpine`
  - copia `dist` a `/usr/share/nginx/html`
  - usa `nginx.conf` provisto por el repo
  - corre como usuario `nginx`

Y en `docker-compose` frontend también queda `read_only: true` + `tmpfs` con permisos restringidos:

- `noexec`
- `nosuid`
- tamaño limitado

### 11.3 docker-compose: servicios y dependencia

En el `docker-compose.yml` root se ve un stack típico:

- `minio`
- `minio-init`
- `migrator`
- `backend`
- `frontend`

En `docker-compose.dev.yml`:

- el backend conecta a una DB externa (usa `DATABASE_URL_DEV`)
- el frontend se expone en puerto 9090 para evitar colisión con prod

En `docker-compose.prod.vpn.yml`:

- se usa un contenedor `vpn` (gluetun) y `backend` corre con `network_mode: service:vpn`
- es decir, el backend sale “encapsulado” por VPN, útil si requiere acceder a una red interna

---

## 12. Proxy inverso del frontend: cómo viaja el tráfico a backend y MinIO

### 12.1 Desarrollo: Vite proxy

En `frontend/vite.config.ts`:

- cualquier request con path `/api` se proxy a `http://localhost:3000`
- incluye `ws: true` para WebSocket

Esto significa que durante dev:

- el frontend puede seguir usando rutas relativas `/api/...`
- sin preocuparse por CORS (o por endpoints completos)

### 12.2 Producción: Nginx dentro del contenedor frontend

En `frontend/nginx.conf`:

1. `location /api/`:
   - hace proxy_pass a `backend:3000`
   - setea headers `X-Forwarded-*`
   - habilita `Upgrade`/`Connection` para WebSocket
2. `location ^~ /storage/`:
   - proxy_pass a `minio:9000`
   - rewrite para que MinIO reciba la ruta esperada

Además Nginx agrega un set de headers de seguridad (ejemplos):

- `X-Frame-Options SAMEORIGIN`
- `X-Content-Type-Options nosniff`
- `Content-Security-Policy` con:
  - `connect-src 'self' ws: wss:`
  - `object-src 'none'`
- `Permissions-Policy` limitada (desactiva cámara/mic/etc)

Esto reduce riesgo de clickjacking y de algunos tipos de inyección/ejecución.

---

## 13. Riesgos y “cosas a verificar” para un panorama 100% honesto

Sin ocultar nada, hay puntos que conviene revisar porque impactan seguridad o consistencia:

1. **CSRF no se ve explícito**:
   - se usa cookie con `sameSite='none'` en HTTPS
   - no aparece un token CSRF en endpoints protegidos
   - depende de configuración del despliegue (mismo sitio vs distinto sitio) y de la forma en que el navegador envía cookies
2. **CORS “origin: true” si `FRONTEND_URL` no está seteado**:
   - si se configura mal el entorno, puede abrir permisos
3. **WebSocket: gate del frontend vs token en handshake**:
   - `useSocket.ts` obtiene `token` desde `document.cookie` para pasarlo en `auth`
   - el backend valida JWT también desde `headers.cookie`, pero si el frontend no conecta por el gate (“no hay token”), puede no disparar el handshake
4. **Verificación email usa Map en memoria**:
   - si hay múltiples réplicas del backend o reinicios, se pierden códigos
   - no es necesariamente “inseguro” pero sí limita consistencia
5. **Subida de evidencia adicional**:
   - en rutas se usa `uploadMultipleFiles` por compatibilidad del middleware
   - el controller `subirEvidenciaDenuncia` espera principalmente `req.body` con `objectKey`, `nombreOriginal`, etc.
   - conviene confirmar que frontend esté enviando lo que el controller espera

---

## 14. Dónde leer para entender “en vivo” cada flujo

Si te interesa verificar cualquier afirmación de este documento contra el código, estos archivos son los puntos de arranque más directos:

- Autenticación / sesión (backend):
  - `backend/src/middlewares/auth.middleware.js` (verifyToken, hasRole, verifyTemporaryToken)
  - `backend/src/controllers/auth.controller.js` (login, ClaveÚnica, me, logout)
  - `backend/src/config/auth.config.js` (cookie/JWT/roles/redirecciones)
- Denuncias (backend):
  - `backend/src/routes/denuncias.routes.js`
  - `backend/src/controllers/denuncia.controller.js`
  - `backend/src/services/denuncia.service.js`
- Validaciones:
  - `backend/src/validations/denuncia.validation.js`
  - `backend/src/validations/verificacionEmail.validations.js`
- Evidencias (storage):
  - `backend/src/routes/storage.routes.js`
  - `backend/src/controllers/storage.controller.js`
  - `backend/src/services/storage.service.js`
  - `frontend/nginx.conf` (proxy `/storage`)
- ClaveÚnica:
  - `backend/src/config/claveunica.config.js`
  - `backend/src/controllers/auth.controller.js` (claveunicaCallback)
- Frontend:
  - `frontend/src/app/router.tsx` (rutas y roles)
  - `frontend/src/context/AuthContext.tsx` (checkAuth + login/logout)
  - `frontend/src/components/ProtectedRoute.tsx`
  - `frontend/src/hooks/useSocket.ts`
- Despliegue:
  - `frontend/nginx.conf`
  - `frontend/vite.config.ts` (proxy dev)
  - `backend/Dockerfile` y `frontend/Dockerfile`
  - `docker-compose*.yml`

