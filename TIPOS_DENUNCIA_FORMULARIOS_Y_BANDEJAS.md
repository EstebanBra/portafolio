## Tipos de denuncia: formularios adaptados y vistas por rol

En el sistema, el flujo de denuncia se construye a partir de un **tipo general** elegido por el usuario (pantalla inicial de `NuevaDenuncia`) y se transforma en el backend a través de **`ID_TipoDe`** (almacenado en `subtipoId` en el frontend).

La parte “clave” para entender el sistema es:
- En `frontend/src/pages/Denuncias/NuevaDenuncia.tsx` se define el **tipo elegido** y se mapea a un **`subtipoId`** (que termina siendo `ID_TipoDe` en el payload).
- En `frontend/src/pages/Denuncias/components/steps/*` se renderizan **campos condicionales** según ese tipo (por ejemplo `tipoId === 3` implica *Campos Clínicos* y cambia la ubicación y validaciones).
- Las **bandejas** de cada rol filtran/agrupan por **`ID_TipoDe`** (y derivaciones) y luego llevan a pantallas `Detalle*` donde se visualiza el contenido (relato, personas denunciadas, testigos, evidencias, y ubicación con lógica distinta para Campo Clínico).

---

## 1) Mapa base: tipo general -> `ID_TipoDe`

En `NuevaDenuncia`, el usuario selecciona una tarjeta (paso de selección tipo). La selección se guarda en `form.tipoId` y se asigna por defecto un `subtipoId` (que es lo que finalmente se envía como `ID_TipoDe`).

Según `handleSelectTipo(id)` en `frontend/src/pages/Denuncias/NuevaDenuncia.tsx`:
- `tipoId === 1` (Género) => `subtipoId = 100` => `ID_TipoDe = 100`
- `tipoId === 2` (Convivencia) => `subtipoId = 200` => `ID_TipoDe = 200`
- `tipoId === 3` (Campos Clínicos) => `subtipoId = 300` => `ID_TipoDe = 300`

En el payload de creación (`enviarDenuncia`), se envía:
- `ID_TipoDe: Number(form.subtipoId)`
- `Ubicacion` con formato diferente para Campo Clínico vs. denuncias “normales”
- `detalleCampoClinico` solo cuando `esCampoClinico` es verdadero (`form.tipoId === 3`)

Este mapeo explica por qué cada bandeja puede filtrar por `d.tipo_denuncia?.ID_TipoDe` (o equivalentes en el estado cargado por `getDenunciaById`).

---

## 2) Formulario adaptado por tipo (lo que ve el denunciante)

El formulario tiene 3 pasos en:
- `frontend/src/pages/Denuncias/components/FormularioLayout.tsx`
- `Paso1Identificacion.tsx` (Paso 1)
- `Paso2Hechos.tsx` (Paso 2)
- `Paso3Confirmacion.tsx` (Paso 3 / revisión)

### 2.1 Paso 1: Identificación del denunciante (común para todos los tipos)

En `Paso1Identificacion` (`frontend/src/pages/Denuncias/components/steps/Paso1Identificacion.tsx`):
- Se muestra “Categoría seleccionada” (tipo y posibilidad de cambiarlo).
- Se capturan datos del denunciante como:
  - `rut`, `nombre`, `sexo`, `genero` (identidad de género opcional)
  - `carreraCargo`
  - `telefono`, `correo`
  - Ubicación del denunciante: `regionDenunciante`, `comunaDenunciante`, `direccionDenunciante`
- Además, existe lógica para:
  - **Reserva de identidad** (radio `reservaIdentidad`)
  - Autenticación por ClaveÚnica: el campo `rut` aparece `readOnly` porque viene desde el login.

Nota de adaptación relevante:
- Aunque el tipo elegido afecta campos posteriores, en Paso 1 el formulario no cambia su estructura de forma radical; los cambios fuertes ocurren en Paso 2 y en el payload final.

### 2.2 Paso 2: Hechos y “participantes” (cambia según tipo)

En `frontend/src/pages/Denuncias/components/steps/Paso2Hechos.tsx`:

#### A) Campos “siempre presentes”
- Víctima:
  - Se define si la víctima es menor (`victimaMenor`) y si el denunciante es la víctima (`esVictima`).
  - Si el denunciante **NO** es la víctima (`!esVictima`), se muestran campos opcionales de víctima: `victimaRut`, `victimaNombre`, `victimaCorreo`, `victimaTelefono`, `victimaSexo`, `victimaGenero`, `regionVictima`, `comunaVictima`, `direccionVictima`.
  - Si el denunciante **SÍ** es la víctima, los campos de víctima se **autocompletan** desde Paso 1 y se muestran `readOnly`.
- Denunciado/s:
  - Se permite agregar personas denunciadas en `form.involucrados` con campos como:
    - `nuevoInvolucrado.nombre` (obligatorio para “confirmar denunciado”)
    - `nuevoInvolucrado.vinculacion` (obligatorio)
    - `nuevoInvolucrado.descripcionDenunciado` (opcional)
    - `nuevoInvolucrado.rut` y `unidadCarrera` (opcional en sección expandible)
  - **Selección de vinculación cambia por tipo clínico** (ver siguiente apartado).
- Fecha y relato:
  - Se define `tipoFecha` como `unica` o `rango`, con campos `fechaHecho` (y `fechaHechoFin` si aplica).
  - Se captura `relato` (obligatorio, mínimo 20 caracteres).
- Testigos:
  - Se agregan en lista (`testigos`) con `nombreCompleto`, `rut` (opcional) y `contacto` (obligatorio para cada testigo).
- Evidencia:
  - Sección `FileUploader` para adjuntar archivos (opcional).

#### B) El “punto de quiebre”: ubicación y validaciones para Campo Clínico

En Paso 2 existe una rama explícita:
- Si `form.tipoId === 3` (Campo Clínico), se renderizan campos del establecimiento:
  - `nombreEstablecimiento` (obligatorio)
  - `regionEstablecimiento` (obligatorio)
  - `comunaEstablecimiento` (obligatorio)
  - `direccionEstablecimiento` (obligatorio)
  - `unidadServicio` (obligatorio)
  - Además, `relato` cambia su placeholder para orientar la descripción.
- Si `form.tipoId !== 3`, se renderiza ubicación “normal” del hecho:
  - `sedeHecho` (obligatorio)
  - `lugarHecho` (opcional/depende selección de sede)
  - `detalleHecho` (opcional)

Esta diferencia también se refleja en:
- El texto/placeholder del `textarea` de `relato`
- La validación de “puedeAvanzar” y “validateForm” en `NuevaDenuncia.tsx`

#### C) Vinculación del denunciado: catálogo distinto para Campo Clínico

En `Paso2Hechos`, el `<select>` de `Vinculación` usa:
- `VINCULACIONES_CAMPO_CLINICO` cuando `form.tipoId === 3`
- `VINCULACIONES` para los otros tipos

También hay una alerta especial cuando:
- `form.tipoId === 3` y se selecciona `TUTOR_HOSPITAL`

### 2.3 Paso 3: Revisión (resumen adaptado)

En `Paso3Confirmacion` (`frontend/src/pages/Denuncias/components/steps/Paso3Confirmacion.tsx`):
- Se muestra un “Resumen de la Denuncia” donde:
  - Se define `esCampoClinico = formulario.tipoId === 3`
  - Se arma `ubicacionTexto` con una lógica distinta:
    - Campo Clínico: `nombreEstablecimiento`, `regionEstablecimiento`, `comunaEstablecimiento`, `unidadServicio`
    - Normal: `sedeHecho`, `lugarHecho`, `detalleHecho`
- Se muestra `relato` completo (con opción de expandir si es largo).
- Se listan denunciados (`involucrados`) y testigos (`testigos`).
- Si hay archivos, se listan con nombre y tamaño.

---

## 3) `enviarDenuncia`: cómo cambia el payload por tipo

En `NuevaDenuncia.tsx` dentro de `enviarDenuncia`:

### 3.1 Ubicación (`Ubicacion`)
- Para Campo Clínico (`esCampoClinico`):
  - `Ubicacion = [nombreEstablecimiento, regionEstablecimiento, comunaEstablecimiento, direccionEstablecimiento, unidadServicio, detalleHecho]` unidas por ` - `
- Para denuncias normales:
  - `Ubicacion = [sedeNombre, lugarHecho, detalleHecho]` unidas por ` - `

Esta elección impacta directamente cómo el sistema “reconstruye” ubicación al mostrar la denuncia en `DetalleDenuncia`, `DetalleRevisor` y `DetalleCampoClinico` (cuando no vienen campos desagregados).

### 3.2 `detalleCampoClinico`
- Solo se envía cuando `form.tipoId === 3`
- Contiene:
  - `nombreEstablecimiento`
  - `unidadServicio`
  - `tipoVinculacionDenunciado` (derivado de `vinculacion` del primer involucrado o del “nuevo involucrado” en edición)
  - `region`, `comuna`, `direccionEstablecimiento`

### 3.3 Denunciados y testigos
- `denunciados` se arma desde `form.involucrados`
- `testigos` se arma desde `form.testigos`
- Si la víctima es externa (no es el denunciante), se envía como `victima` únicamente si existe `victimaRut` (por comentario del propio código).

---

## 4) Bandejas por rol: qué muestran y cómo se filtra por tipo

La revisión por roles ocurre en dos niveles:
1. **Bandeja (listado)**: muestra un conjunto de denuncias filtradas por rol (y por tipo/área).
2. **Detalle (pantalla por caso)**: muestra el contenido para tomar acciones (o para derivar / emitir informe, etc.).

### 4.1 Revisor (vista transversal)

Archivos:
- `frontend/src/pages/Revisor/BandejaRevisor.tsx`
- `frontend/src/pages/Revisor/DetalleRevisor.tsx`

#### Bandeja
- Carga todas las denuncias con `listarDenuncias({ page: 1, pageSize: 100 })`.
- Luego filtra localmente con un selector:
  - `convivencia`: `ID_TipoDe` en `200`, `301`, `302`
  - `genero`: `ID_TipoDe` en `100`, `303`
  - `camposClinicos`: `ID_TipoDe === 300`
- La “Área” en la tabla se calcula con `obtenerAreaGeneralizada`:
  - `300` => “Campos Clínicos”
  - `100` o `303` => “Género y Equidad”
  - `200` o `301` o `302` => “Convivencia Estudiantil”

#### Detalle
- En `DetalleRevisor` se ve:
  - Clasificación del hecho (`denuncia.tipo_denuncia?.Nombre`, `denuncia.tipo_denuncia?.Area`)
  - Relato
  - Personas denunciadas, testigos, evidencias
  - Datos del denunciante y la víctima
  - Ubicación con lógica distinta para Campo Clínico:
    - Si detecta campo clínico, intenta leer `detalle_campo_clinico` y/o parsea `Ubicacion`.

Además, cuando el caso fue derivado (IDs 301/302/303) muestra un banner con `observacionDirgegen`.

### 4.2 DIRGEGEN (Género y Equidad)

Archivos:
- `frontend/src/pages/Dirgegen/BandejaDirgegen.tsx`
- `frontend/src/pages/Dirgegen/DetalleDirgegen.tsx`

#### Bandeja
- Filtra para competencia “Protocolo de Género y Equidad (DUE 4560)” con:
  - `ID_TipoDe === 100` o `ID_TipoDe === 303`
- Luego excluye casos que ya fueron enviados a VRA cuando están en:
  - `estado_denuncia?.Tipo_Estado === 'En Revisión'` y `ID_TipoDe === 100`

Adicionalmente:
- Consulta `listarMedidasPendientes()` y muestra un banner rojo si hay medidas pendientes que requieren Informe Técnico urgente.

En la tabla, la acción típica es `navigate(/dirgegen/denuncia/:idDenuncia)`.

#### Detalle
- En `DetalleDirgegen` se prioriza la visualización del contenido:
  - Clasificación y relato
  - Personas denunciadas (con botón para “Identificar” si no están individualizadas)
  - Testigos
  - Evidencias
  - Datos denunciante y víctima (con lógica de víctima/denunciante y menor de edad)
- Para derivaciones:
  - Detecta si fue derivada por presencia de `observacionDirgegen` y por `tipoActual !== 100`.
  - En el footer aparecen acciones para:
    - Generar/Ver Informe Técnico (`InformeTecnicoModal`)
    - Derivar caso (con `DerivacionModal`, opciones `301: VRA` o `300: Campo Clínico`)
    - Enviar a VRA (solo si existe `informe_tecnico` y con confirmación).

También se muestra la sección de “Medidas Solicitadas” si existe `solicitaMedidas`.

### 4.3 Autoridad (VRA / VRAE / VRIP)

Archivos:
- `frontend/src/pages/Autoridad/BandejaAutoridad.tsx`
- `frontend/src/pages/Autoridad/DetalleAutoridad.tsx`

#### Bandeja
- Determina la autoridad según rol del usuario:
  - `VRA`, `VRAE`, o `VRIP`
- Llama `listarDenuncias({ page: 1, pageSize: 100 })` y luego aplica un filtro extra en frontend con:
  - `ID_TipoDe === 200` (Convivencia)
  - `ID_TipoDe === 301` (VRA General)
  - `ID_TipoDe === 100` (Género enviado con informe)

En el botón de acción:
- se accede a `navigate(/autoridad/denuncia/:idDenuncia)`.

Además, muestra botón para “Descargar PDF” por caso (`generarPdfDenunciaOriginal`).

#### Detalle
- En `DetalleAutoridad` se ve el contenido de la denuncia de forma muy similar a Revisor:
  - Clasificación y relato
  - Personas denunciadas, testigos, evidencias
  - Denunciante y víctima
  - Ubicación (y sección de ubicación cuando `denuncia.Ubicacion` está presente)
- Acciones del footer:
  - “Declarar admisibilidad” (con `DeclararAdmisibilidadModal`)
  - “Derivar a Otra Autoridad” (con `DerivacionModal`, opciones `303: DIRGEGEN` o `300: Campo Clínico`)
  - El caso puede mostrar banner de derivación por `observacionDirgegen` como en otros detalles.

### 4.4 Campo Clínico (ID 300)

Archivos:
- `frontend/src/pages/CampoClinico/BandejaCampoClinico.tsx`
- `frontend/src/pages/CampoClinico/DetalleCampoClinico.tsx`

#### Bandeja
- Filtra directamente por:
  - `d.tipo_denuncia?.ID_TipoDe === 300`
- Ordena por fecha (sin lógica extra por área, porque su bandeja ya es específica de clínica).
- Acción principal:
  - `navigate(/campo-clinico/denuncia/:idDenuncia)`

#### Detalle
- `DetalleCampoClinico` valida que sea realmente clínica:
  - si `tipoId !== 300` muestra error “no corresponde a Campos Clínicos”.
- Visualización:
  - Clasificación del hecho (nombre y área del tipo)
  - Relato
  - Personas denunciadas, testigos, evidencias
  - Denunciante y víctima
  - **Ubicación Campo Clínico**:
    - prioriza `detalle_campo_clinico` (campos como `Nombre_Establecimiento`, `Direccion_Establecimiento`, `Region`, `Comuna`, `Unidad_Servicio`)
    - si no existen campos desagregados, parsea el string `Ubicacion` separando por ` - `
    - además muestra el `Tipo_Vinculacion_Denunciado` si viene.
- Acciones del footer:
  - “Derivar a Otra Autoridad” con opciones `301: VRA` o `303: DIRGEGEN` (a nivel de UI).

---

## 5) Derivaciones entre tipos: cómo se refleja en la UI de bandejas

Aunque el usuario inicia con `ID_TipoDe` base (100/200/300), el backend permite “derivar” cambiando el tipo (`nuevoTipoId`) y el estado (`nuevoEstadoId`).

Eso se ve en dos lugares:

### 5.1 Filtros de bandejas (qué casos llegan a cada rol)
- Revisor agrupa:
  - Género y Equidad: `100` y derivaciones `303`
  - Convivencia Estudiantil: `200`, `301`, `302`
  - Campo Clínico: `300`
- DIRGEGEN recibe:
  - `100` y `303`, excluyendo ciertos casos en “En Revisión” para evitar retrabajo (en el filtro de `BandejaDirgegen`).
- Autoridad recibe (filtro extra en UI):
  - `200`, `301` y `100` (cuando hay informe en el caso).
- Campo Clínico recibe:
  - `300` únicamente.

### 5.2 Banners de “esta denuncia fue derivada”
- En los `Detalle*`, aparece un banner cuando se detecta `observacionDirgegen` y/o cuando el `ID_TipoDe` está en rangos específicos.
  - En `DetalleRevisor` y `DetalleCampoClinico` se considera derivada si `ID_TipoDe` está en `301`, `302` o `303`.
  - En `DetalleDirgegen`, la lógica depende de `observacionDirgen` y de que el tipo no sea el original (`tipoActual !== 100`).

El mensaje usa principalmente:
- `denuncia.tipo_denuncia?.Nombre`
- `observacionDirgegen`

---

## 6) Checklist mental (cómo leer una denuncia ya enviada)

Si estás revisando un caso en cualquier `Detalle*`:
- “Clasificación”:
  - Se toma desde `denuncia.tipo_denuncia?.Nombre` y `denuncia.tipo_denuncia?.Area`.
- “Ubicación”:
  - Si es Campo Clínico, se muestra desde `detalle_campo_clinico` o parseando `Ubicacion`.
  - Si es normal, se muestra desde `sedeHecho/lugarHecho/detalleHecho` o parseando `Ubicacion`.
- “Participantes”:
  - Denunciados y testigos se renderizan listando y abriendo modales cuando corresponde.
- “Derivación”:
  - Si fue derivada, el banner te dice a qué unidad fue y muestra la observación.

Con esto ya puedes relacionar cada tipo con su formulario de origen y con la bandeja correcta donde será revisado.

