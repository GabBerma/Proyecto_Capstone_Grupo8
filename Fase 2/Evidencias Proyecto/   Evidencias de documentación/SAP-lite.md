# SAP-lite – Documentación del módulo de Control de Gestión

> **Nota metodológica (lee esto primero).** Este documento se construyó revisando directamente el
> código del repositorio PittySoft (backend NestJS/Prisma, frontend `sap-lite-web`, migraciones,
> specs y configuración de despliegue). El estado que aquí se describe corresponde a la rama
> integrada **`origin/main`** (estado de referencia del proyecto) al **14 de octubre de 2026**.
> La rama de trabajo revisada localmente (`feat/prototipos-importacion-excel`) está **51 commits por
> detrás de `main`** y contiene trabajo en curso sin commitear; las diferencias relevantes entre esa
> rama y `main` se señalan explícitamente en la sección final *Hallazgos para revisar con el equipo*.
> Donde una funcionalidad no pudo verificarse en el código se indica de forma expresa.

---

## 1. Descripción general

**SAP-lite** es el módulo de **control de gestión administrativo-financiero** de PittySoft. No es un
ERP contable completo ni un reemplazo de SAP; es una pieza acotada (“lite”) orientada a **centralizar
y digitalizar los procesos de cuentas por pagar y de ingreso/consolidación de información financiera**
que hoy se gestionan de forma manual o en planillas Excel.

- **Qué problema resuelve.** Reemplaza el manejo disperso de proveedores, facturas de compra,
  cartolas bancarias, liquidaciones de tarjetas y pagos electrónicos por un sistema único,
  multi-tenant y auditable. Elimina la re-digitación manual mediante importaciones masivas con
  previsualización y validación.
- **Relación con PittySoft.** SAP-lite comparte la plataforma, la base de datos PostgreSQL y el
  modelo de autenticación/multi-tenant del backend de PittySoft (el mismo backend NestJS que sirve al
  POS clínico-óptico). El frontend, en cambio, es una **aplicación web independiente** (`sap-lite-web`,
  Vue 3) desplegada por separado del POS.
- **Procesos que soporta (verificables).** Gestión de proveedores; importación masiva de proveedores
  desde Excel/CSV; consulta de situación tributaria de terceros (SII); facturas de compra y notas de
  crédito (cuentas por pagar); importación de cartolas bancarias (Banco Itaú); importación de
  liquidaciones Transbank; consulta/sincronización de pagos de Mercado Pago; consulta del Plan de
  Cuentas. En estado de **prototipo** soporta además arriendos/gastos comunes y pagos de servicios.
- **Usuarios/áreas.** El área de **administración y finanzas** de la empresa (perfiles con permisos
  `admin`, y según el seed de roles `district_manager`/`store_manager` para consulta del Plan de
  Cuentas). Los permisos finos (`proveedores`, `facturas_compra`, `cartola_bancaria`,
  `liquidaciones_transbank`, `mercado_pago`, `plan_cuentas`) se otorgan por rol.

---

## 2. Alcance del módulo

Funcionalidades efectivamente presentes en el repositorio (el detalle de estado está en la §15):

| Funcionalidad | ¿Presente en el código? |
|---|---|
| Gestión de proveedores (alta/edición/consulta, baja lógica) | **Sí** (backend + frontend) |
| Importación masiva de proveedores desde Excel/CSV (preview + commit) | **Sí** |
| Consulta tributaria de terceros vía SII | **Sí** (mock operativo; real vía API Gateway) |
| Asociación de proveedores a cuentas contables | **Parcial** – como **texto libre**, sin FK al Plan de Cuentas |
| Facturas de compra | **Sí** |
| Notas de crédito | **Sí** (como tipo de documento + regla de 6 meses) |
| Sincronización con BSale (documentos de terceros) | **Parcial / en validación** |
| Importaciones financieras (cartolas) | **Sí** (Banco Itaú) |
| Liquidaciones Transbank (importación) | **Sí** |
| Mercado Pago (consulta de pagos) | **Sí** (solo lectura/sincronización) |
| Arriendos y gastos comunes | **Prototipo** (solo frontend, en memoria) |
| Pagos de servicios | **Prototipo** (solo frontend, en memoria) |
| Plan de Cuentas | **Sí** (catálogo global de solo lectura) |
| Reportes de gastos | **No verificado** como función dedicada de SAP-lite |
| Conciliación financiera (cartola ↔ facturas ↔ Transbank) | **No** (especificación en borrador) |

---

## 3. Arquitectura del módulo

### 3.1 Backend

- **Framework:** NestJS `10.4.0` sobre adaptador **Fastify** (`@nestjs/platform-fastify`).
- **ORM / BD:** **Prisma `5.22.0`** sobre **PostgreSQL 15**. (Importante: el backend usa Prisma, **no**
  TypeORM.)
- **Validación:** `class-validator 0.14` + `class-transformer 0.5` sobre DTOs.
- **Procesamiento Excel/CSV:** `xlsx` (SheetJS) `0.18.5`, configurado sin ejecución de fórmulas ni
  macros (`cellFormula:false`, `bookVBA:false`).
- **Autenticación:** `@nestjs/jwt` + `passport-jwt`; hash de contraseñas con `bcrypt`.
- **Infra de soporte:** Redis (`ioredis`) + colas Bull (`@nestjs/bull`), `helmet`, `@nestjs/throttler`
  (rate limiting), `@nestjs/swagger` (OpenAPI).
- **SDKs de IA presentes** (`@anthropic-ai/sdk`, `@google/generative-ai`, `@modelcontextprotocol/sdk`):
  usados por el módulo genérico `reports` (NL2SQL), no por el núcleo de cuentas por pagar.
- **Patrón:** **N-Tier** — Controller → Service → Prisma → PostgreSQL. Un feature = un módulo
  (`backend/src/modules/<feature>`), consistente con `ARCHITECTURE.md`.

**Organización de módulos backend relevantes a SAP-lite:**

```
backend/src/modules/
├── proveedores/            # SPEC-001/005/008/011/013  (+ subcarpeta sii/)
├── facturas-compra/        # SPEC-004/007/009  (+ bsale-sync, bsale-terceros)
├── cartola-bancaria/       # SPEC-002  (+ itau-adapter)
├── liquidaciones-transbank/# SPEC-006  (+ transbank-adapter)
├── mercado-pago/           # SPEC-012  (import service)
├── plan-cuentas/           # SPEC-016  (solo en origin/main)
└── payments/               # integración Mercado Pago/Point reutilizada
```

### 3.2 Frontend (`sap-lite-web`)

- **Framework:** Vue `3.5` con **Vite `5.4`**; patrón **MVVM** (lógica en *stores*, HTTP centralizado).
- **Router:** Vue Router `4.4` con guard de autenticación (`meta.requiresAuth` / `meta.public`).
- **Estado:** **Pinia `2.2`**.
- **HTTP:** **axios `1.7`** encapsulado en `src/lib/api.ts` (los componentes no llaman HTTP directo).
- **Excel cliente:** `xlsx 0.18.5` (previsualización local). **UI:** TailwindCSS `4.0`, íconos
  `lucide-vue-next`.
- **Testing:** **Vitest `4.1`** + Vue Test Utils.

**Capa HTTP (`src/lib/api.ts`):** interceptor de request que adjunta `Authorization: Bearer <token>`
(desde `localStorage`) y deriva `x-tenant-id` del payload del JWT; interceptor de response que, ante
`401`, intenta `refresh` del token y reintenta, o redirige a `/login`.

### 3.3 Base de datos

- Esquema único en `backend/prisma/schema.prisma`; 36+ migraciones versionadas por timestamp.
- Aislamiento multi-tenant por **Row-Level Security (RLS)** de PostgreSQL: cada tabla tenant-scoped
  tiene columna `tenant_id` y política `tenant_isolation` (`USING (tenant_id = current_setting(
  'app.current_tenant_id', true))`). El middleware de tenant fija esa variable de sesión por request.
- **Excepción:** `cuentas_contables` (Plan de Cuentas) es **catálogo global sin `tenant_id` ni RLS**.

### 3.4 Despliegue (verificable)

- **GCP Cloud Run** + **Artifact Registry**, infra como código en **Terraform** (`infra/`:
  `cloudrun.tf`, `sap-lite.tf`, `frontend.tf`, `network.tf`, `wif.tf`…), con autenticación sin llaves
  (**Workload Identity Federation**).
- **GitHub Actions:** `ci.yml` (typecheck + tests en cada PR), `deploy-sap-lite.yml`,
  `deploy-frontend.yml`, `deploy-gcp.yml`, `automerge.yml`.
- **Desarrollo local:** `docker-compose.yml` levanta `postgres:15-alpine` y `redis:7-alpine`.

---

## 4. Modelo de datos

Entidades Prisma verificadas (tabla ← modelo). Todas incluyen auditoría (`createdAt/By`,
`updatedAt/By`) salvo indicación.

| Entidad (modelo) | Tabla | Rol | Tenant |
|---|---|---|---|
| `Proveedor` | `proveedores` | Maestro de proveedores | Sí (RLS) |
| `FacturaCompra` | `facturas_compra` | Facturas de compra y NC | Sí (RLS) |
| `CuentaBancaria` | `cuentas_bancarias` | Cuentas para cartolas | Sí (RLS) |
| `ImportacionBancaria` / `...Preview` | `importaciones_bancarias(_preview)` | Importaciones de cartola | Sí (RLS) |
| `MovimientoBancario` | `movimientos_bancarios` | Movimientos de cartola | Sí (RLS) |
| `ImportacionTransbank` / `...Preview` | `importaciones_transbank(_preview)` | Importaciones Transbank | Sí (RLS) |
| `LiquidacionTransbank` | `liquidaciones_transbank` | Abonos/liquidaciones Transbank | Sí (RLS) |
| `MercadoPagoPayment` | `mercado_pago_payment` | Pagos sincronizados de MP | Sí (RLS) |
| `MercadoPagoSyncLog` | `mercado_pago_sync_log` | Bitácora de sincronización MP | Sí (RLS) |
| `CuentaContable` | `cuentas_contables` | **Plan de Cuentas (global)** | **No** |
| `Tenant`, `Role`, `User` | (plataforma) | Multi-tenant, roles/permisos, usuarios | — |

**Relaciones relevantes verificadas:**

- `Proveedor 1—N FacturaCompra` (`FacturaCompra.proveedorId`, **opcional**: los documentos de BSale sin
  proveedor identificado quedan `pendienteVinculacion`).
- `FacturaCompra` **auto-relación** `documentoCorregido ↔ correcciones`: una nota de crédito referencia
  la factura original que corrige (base de la regla de 6 meses, SPEC-009).
- `CuentaBancaria 1—N (ImportacionBancaria, MovimientoBancario, Preview)`.
- **No existe** FK entre `Proveedor`/arriendos/servicios y `CuentaContable`: la “cuenta contable” se
  almacena como **texto libre** (`numeroCuentaContable`, `nombreCuenta`).

> No se incluye un DER formal porque no existe un diagrama verificable en el repositorio; las
> relaciones anteriores se derivan directamente del `schema.prisma`.

---

## 5. Gestión de proveedores

**Estado: Implementado** (backend + frontend). Specs: SPEC-001, SPEC-005, SPEC-008, SPEC-011, SPEC-013.

**Campos del modelo `Proveedor`:** `rut` (opcional, único por tenant cuando existe), `razonSocial`
(obligatorio), `nombreFantasia`, `giro`, `email`, `telefono`, `direccion`, `tipoCodigo` (código de
catálogo `parametros`, **no FK**), `numeroCuentaContable` y `nombreCuenta` (**texto libre**),
`observaciones`; campos ampliados de SPEC-013: `rubro`, contactos comercial y de pagos
(`contacto*Nombre/Email`), datos bancarios de pago (`pagoBanco`, `pagoTipoCuenta`, `pagoNumeroCuenta`,
`pagoTitular`, `pagoRutTitular`); `origen` (enum **`MANUAL | MOCK | SII`**), `activo` (baja lógica), y
auditoría incluyendo `deactivatedAt/By`.

- **RUT:** validación mód-11 y normalización (`shared/rut.util`) cuando se informa; unicidad por tenant
  (`@@unique([tenantId, rut])`, permitiendo múltiples proveedores sin RUT).
- **Estado del proveedor:** `activo` booleano con baja/reactivación lógica
  (`POST /:id/desactivar`, `POST /:id/reactivar`) y traza de quién/cuándo.
- **Búsquedas y filtros:** `GET /api/v1/proveedores` paginado, búsqueda OR por rut/razón social/nombre
  de fantasía, filtro `estado` (`activos|inactivos|todos`) y filtro por fecha de carga (`YYYY-MM-DD`).
- **Importación masiva:** ver §cuenta de importaciones más abajo (SPEC-005).
- **Consulta SII:** sí, real; ver §13.
- **Cuenta contable:** se captura pero **no está asociada al Plan de Cuentas** (texto libre) — ver §11.

**Importación masiva de proveedores (SPEC-005, Implementado):** flujo **preview → commit**.
`POST /import/preview` parsea el archivo (detección de formato por *magic bytes*; CSV UTF-8/Windows-1252
con separador `,` o `;`; XLSX/XLS vía SheetJS), mapea columnas **por nombre** (alias flexibles,
incluidos los campos de SPEC-013), valida fila a fila (RUT, duplicados dentro del archivo y contra la
BD, emails) y devuelve conteo de válidos/duplicados/rechazados con motivo, **sin persistir**.
`POST /import/commit` **re-parsea y re-valida** el archivo (no confía en el estado del cliente) e inserta
solo las filas válidas dentro de una transacción. Límites: 5 MB / 5000 filas; columna `razonSocial`
obligatoria.

---

## 6. Facturas de compra y documentos

**Estado: Implementado.** Specs: SPEC-004 (facturas y cuentas por pagar), SPEC-009 (límite 6 meses NC),
SPEC-007 (BSale, parcial).

**Modelo `FacturaCompra`:** `tipoDocumento` (enum **`FACTURA | NOTA_CREDITO | OTRO`**), `folio`,
`fechaEmision`, `fechaRecepcion`, `fechaVencimiento`, `neto`, `iva`, `exento`, `total`,
`saldoPendiente`, `observaciones`, `centroSucursal`, `origen` (enum **`MANUAL | BSALE`**), `estado`
(enum con único valor **`PENDIENTE`**), y campos de integración BSale (`bsaleId`, `codeSii`,
`emisorRut`, `emisorRazonSocial`, `pendienteVinculacion`, `urlPdf`, `urlXml`, `anulado`, `bsaleRaw`),
más auditoría (`operacion` = `CREACION_MANUAL | INCORPORACION_INTEGRACION`).

- **Cálculo en backend:** `total = neto + iva + exento` (nunca desde el cliente); `saldoPendiente`
  inicial = `total`. La aplicación de pagos que reduce el saldo **no está implementada todavía**.
- **“Vencida”:** es un flag **derivado** en tiempo de consulta (`estado=PENDIENTE && saldo>0 &&
  fechaVencimiento < hoy`), **no persistido**.
- **Notas de crédito:** no son una entidad separada; se modelan como `tipoDocumento=NOTA_CREDITO` con
  auto-relación a la factura corregida. **Regla SPEC-009:** la NC debe referenciar una FACTURA del mismo
  proveedor y emitirse dentro de **6 meses calendario** (con ajuste de fin de mes).
- **Antiduplicados:** `@@unique([tenantId, proveedorId, tipoDocumento, folio])`, `@@unique([tenantId,
  bsaleId])` y clave de negocio `[tenantId, emisorRut, codeSii, folio]`.
- **Origen del documento:** `MANUAL` (alta por usuario) o `BSALE` (incorporado por integración).
- **Sincronización con BSale:** `POST /bsale/sync` + `GET /bsale/sync` (historial); mapea códigos SII
  (33→FACTURA, 61→NOTA_CREDITO, otros→OTRO conservando `codeSii`), idempotente por `bsaleId`, y marca
  `pendienteVinculacion` cuando no logra asociar el emisor a un proveedor. **Estado: parcial / en
  validación** (ver §13).

---

## 7. Importaciones financieras

### 7.1 Cartolas bancarias (SPEC-002 — Implementado)

- **Banco soportado:** **Banco Itaú únicamente** (enum `BancoCartola = ITAU`). Extensible a otros
  bancos mediante nuevos *adapters*, pero solo Itaú está implementado.
- **Cuenta:** entidad `CuentaBancaria` (número almacenado completo, **enmascarado** en la API).
- **Archivo:** Excel (`.xls`/`.xlsx`) exportado de la cartola Itaú, máx. 5 MB; parseo con `xlsx` +
  `itau-adapter` (reconoce preámbulo, línea de período, encabezado de dos filas, filas de movimiento y
  bloque “Resumen de Saldos”, que se ignora).
- **Flujo preview → commit** transaccional: en *preview* se clasifica cada fila como **aceptada /
  rechazada / duplicada** sin persistir (archivo y hash guardados en tabla temporal con TTL de 30 min,
  un solo uso); en *commit* se re-parsea desde el archivo guardado e inserta solo las válidas.
- **Movimientos:** `MovimientoBancario` con `tipo` (`ABONO | CARGO`), `cargo/abono/saldo`, `monto`
  normalizado con signo, `huella` (fingerprint) y `@@unique([tenantId, cuentaBancariaId, huella])` para
  deduplicación. `estado = PENDIENTE`.
- **Rechazos/duplicados/reimportación:** se registra motivo por fila; `@@unique([tenantId,
  cuentaBancariaId, fileHash])` evita reimportar el mismo archivo.
- **Historial:** `GET /importaciones`, detalle por id y `GET /movimientos`.

### 7.2 Liquidaciones Transbank (SPEC-006 — Implementado)

- **Archivo:** reporte Excel “Resumen histórico de abonos Transbank”; parseo con `transbank-adapter`
  (ignora bloque de totales/resumen; lee encabezado de dos filas).
- **Modelo `LiquidacionTransbank`:** `fechaAbono`, `comercioSucursal` (derivado de “Cuenta de
  depósito”), `totalVentas`, `comision`, `ventasAnuladas`, `cobrosPorServicio`,
  `totalMontosDescontados`, `devolucionComision`, `totalAbono`, `numeroVentas`, `huella` con
  `@@unique([tenantId, huella])`. Filas con `totalAbono = 0` se descartan.
- **Flujo preview → commit** idéntico al de cartola; estados aceptada/rechazada/duplicada.
- **Procesamiento y errores:** validación de fecha y montos; motivos de rechazo por fila.
- **Conciliación contra POS (SPEC-003):** **no implementada** — las liquidaciones se importan de forma
  **independiente** de los medios de pago del POS.

---

## 8. Mercado Pago

**Estado: Implementado como consulta/sincronización de solo lectura (SPEC-012).**

- **Qué sincroniza:** pagos desde la cuenta de Mercado Pago hacia `MercadoPagoPayment`:
  `mpPaymentId`, `status`, `statusDetail`, `amount`, `currency`, `dateCreated`, `dateApproved`,
  `lastModified`, `description`, `paymentType`, `paymentMethod`, `installments` (cuotas), y
  `fee`/`netAmount` **opcionales** (no se inventan si MP no los entrega). Bitácora en
  `MercadoPagoSyncLog` (`procesados/creados/actualizados/duplicados/errores`, `resultado`
  `OK|PARCIAL|ERROR`).
- **Endpoints:** `POST /mercado-pago/sync` (rango de fechas opcional), `GET /mercado-pago/sync`
  (historial), `GET /mercado-pago/payments` (listado con filtros de estado y día).
- **Estados de pago:** `approved | pending | in_process | rejected | refunded` (filtros de la UI).
  **Medios/cuotas/montos/moneda:** se persisten según lo que entrega la API.
- **Operativo vs. prueba:** controlado por `MP_MOCK` (**por defecto `true` = modo simulado**). El modo
  real requiere `MP_ACCESS_TOKEN` (y opcionalmente `MP_BASE_URL`); por tanto el ambiente (sandbox o
  producción) depende del token configurado — **no verificable solo desde el código**.
- **Alcance:** SPEC-012 es **solo lectura** (reutiliza `searchPayments`); el flujo de captura de pago en
  terminal (`createIntent`, módulo `payments`/Point) es **independiente** y pertenece al POS, no a
  SAP-lite.

---

## 9. Arriendos y gastos comunes

**Estado: Prototipo (solo frontend, en memoria). Spec en borrador: SPEC-014.**

- **Qué existe:** vistas Vue (`ArriendosListView`, `ArriendoFormView`, `ArriendoDetailView`) y un store
  **mock** (`arriendos-mock.ts`) 100 % en memoria. **No existe módulo backend ni tabla** (no hay
  migración de arriendos).
- **Persistencia:** **ninguna** — los arriendos y movimientos registrados se pierden al refrescar. La
  propia UI muestra un badge “Prototipo / En validación” y un aviso explícito de que los datos se
  guardan solo en la sesión.
- **Datos:** 10 entradas semilla reales del negocio (proveedor/RUT/tipo/local); los campos no
  disponibles (monto, período, vencimiento, contrato, movimientos) quedan en `null` y se muestran como
  pendientes — **no se inventan**.
- **Única llamada real al backend:** `GET /api/v1/locations?type=branch&onlyActive=true` para poblar el
  selector de locales.
- **Qué falta para darlo por terminado:** modelo/tabla y servicio backend, API propia, importador
  “Importar arriendos” real, y definición de reglas de negocio (montos, períodos, vencimientos,
  contratos, reajustes) que hoy están **pendientes de confirmación**.

---

## 10. Pagos de servicios

**Estado: Prototipo (solo frontend, en memoria). Spec en borrador: SPEC-015.**

- **Qué existe:** vistas Vue (`ServiciosListView`, `ServicioFormView`, `ServicioDetailView`) y un store
  **mock** (`servicios-mock.ts`) 100 % en memoria. **No existe módulo backend ni tabla.**
- **Tipos de servicio:** electricidad / telefonía / internet / agua / otro (selector del formulario).
- **Datos:** 3 entradas semilla reales (ENEL, ENTEL, ENTEL PCS) con su número de cuenta contable; el
  resto de campos (número de cliente, local, dirección, vencimientos, cobros) queda en `null` y se
  muestra como “Pendiente de información”.
- **Persistencia:** **ninguna** (se pierde al refrescar); misma advertencia de prototipo en la UI.
- **Única llamada real al backend:** `GET /api/v1/locations` para el selector de locales.
- **Implementado vs pendiente:** implementada solo la maqueta de captura/listado; pendientes modelo,
  servicio y API backend, importador “Importar servicios” real y las reglas de negocio.

---

## 11. Plan de Cuentas

**Estado: Implementado en `origin/main` como catálogo global de solo lectura (SPEC-016).**
*(No presente en la rama de trabajo revisada — ver nota metodológica.)*

- **Estructura y datos:** modelo `CuentaContable` → tabla `cuentas_contables`. Campos: `codigo`
  (formato `d.dd.dd.dd`, original, **no único** a propósito), `nombre`, `tipo`
  (enum **`AGRUPADORA | DETALLE`**, derivado de la posición en el Excel), `nivel` (1–4 derivado del
  código), `codigoPadre` (derivado del propio código, **string, no FK**), `filaOrigen`
  (`@@unique`, clave de idempotencia). Índices por `codigo` y `tipo`.
- **Plan Base / Maestro:** la carga usa la hoja **“Plan de cuenta MAESTRO”** de `Plan de Cuenta
  Base.xlsx`. Se cargan **309 filas** mediante una migración generada automáticamente
  (`20261009120000_spec_016_carga_plan_maestro`) con **carga atómica verificada por checksum**
  (SHA-256 del Excel + md5 de contenido); ante divergencia hace `RAISE EXCEPTION` (rollback) — el deploy
  **falla** en vez de cargar datos incompatibles. No usa `ON CONFLICT DO NOTHING`.
- **Persistencia y global:** catálogo **compartido entre todos los tenants**: la tabla **no tiene
  `tenant_id` ni RLS** (única excepción al modelo multi-tenant).
- **Importación:** no hay carga manual por usuario; el plan se precarga vía migración/seed.
- **API (solo lectura):** `GET /api/v1/plan-cuentas` (listado + búsqueda por código/nombre) y
  `GET /api/v1/plan-cuentas/:id`, ambos con `@RequirePermission('plan_cuentas:read')`. No hay endpoints
  de escritura expuestos. Una migración de datos otorga el permiso `plan_cuentas:read` a los roles
  existentes (`admin`, `district_manager`, `store_manager`).
- **Frontend:** mantenedor de **consulta** (`views/plan-cuentas/PlanCuentasListView.vue` +
  `CuentaDetailModal.vue`, store `plan-cuentas.ts`), ruta `plan-cuentas` con `meta.permission`.
- **Asociación con proveedores/otros:** **no cableada.** Toda “cuenta contable” en proveedores,
  arriendos y servicios sigue siendo **texto libre** sin FK al Plan de Cuentas. La SPEC-016 excluye
  explícitamente la categorización automática y la conversión de códigos legados `410xxx`.
- **Reglas de negocio pendientes:** la distinción contable “imputable/seleccionable” y el tratamiento de
  4 códigos duplicados en la fuente quedan **pendientes** de definición del área de negocio.
- **Archivos/spec:** `docs/specs/sap-lite/SPEC-016 - Plan de Cuentas.md`; módulo
  `backend/src/modules/plan-cuentas/`; migraciones `20261008120000_spec_016_plan_cuentas`,
  `20261009120000_spec_016_carga_plan_maestro`, `20261009130000_grant_plan_cuentas_permission`.

---

## 12. Conciliación financiera

**Estado: No implementada (especificación SPEC-003 en borrador).**

No se verifica en el código ninguna lógica de conciliación: no hay relación automática entre cartolas y
facturas, ni entre liquidaciones Transbank y los medios de pago del POS, ni manejo de pagos
parciales/agrupados, reglas de coincidencia o marcado de movimientos conciliados/no conciliados. Las
tres fuentes de datos (cartola, Transbank, Mercado Pago) hoy se **importan de forma independiente** y
quedan disponibles (`estado = PENDIENTE`) para una conciliación futura. La aplicación de pagos que
reduciría el `saldoPendiente` de las facturas tampoco está implementada.

---

## 13. Integraciones externas

| Integración | Estado | Detalle verificable |
|---|---|---|
| **SII** (situación tributaria de terceros) | **Implementada vía API Gateway, en validación** | En `origin/main`, `ApiGatewaySiiProvider` hace `fetch` real a `https://app.apigateway.cl` usando `APIGW_TOKEN`. Seleccionable con `SII_PROVIDER` (`mock` por defecto / `sii`). El `MockSiiProvider` está **plenamente operativo** para desarrollo. El modo real **depende de un token de un servicio externo de pago** y no es el modo por defecto ⇒ pendiente de validación en ambiente real. |
| **BSale** (documentos de terceros) | **Parcial / en validación** | `bsale-sync.service` + `bsale-terceros.service` en `facturas-compra`; controlado por `BSALE_MOCK`/`BSALE_TOKEN`. Mapea códigos SII y maneja `pendienteVinculacion`. La prueba E2E con token real está pendiente. |
| **Mercado Pago** | **Implementada (solo lectura), operativa según configuración** | Sincronización de pagos; `MP_MOCK=true` por defecto; modo real requiere `MP_ACCESS_TOKEN`. |
| **Bancos (cartola Itaú)** | **Implementada (importación de archivo)** | No es API en línea: se importa el Excel exportado por el banco. Solo Itaú. |
| **Transbank (liquidaciones)** | **Implementada (importación de archivo)** | Importación del reporte de abonos; sin API en línea; sin conciliación con POS. |
| **Plan de Cuentas (fuente Excel)** | **Implementada (carga por migración)** | Carga desde Excel maestro vía migración con checksum. |

*No se exponen tokens ni credenciales en este documento; solo se nombran las claves de configuración.*

---

## 14. Seguridad y reglas de negocio

- **Autenticación:** JWT (access + refresh) vía Passport; guard global de JWT que respeta rutas
  `@Public()`. El frontend adjunta el token y refresca automáticamente ante `401`.
- **Multi-tenant (innegociable):** middleware de contexto de tenant que valida `tenantId` del JWT y fija
  `app.current_tenant_id`; **RLS de PostgreSQL** en todas las tablas tenant-scoped como defensa en
  profundidad. Única excepción: `cuentas_contables` (catálogo global de solo lectura).
- **Roles y permisos:** modelo `Role.permissions` (JSON) con permisos tipo `modulo:accion`; decorador
  `@RequirePermission('proveedores:read' | 'facturas_compra:write' | …)` + `PermissionsGuard`. El seed
  define roles (`admin`, perfiles de tienda); los permisos nuevos se propagan a tenants existentes por
  migraciones de datos.
- **Validación de inputs:** DTOs con `class-validator` en todos los endpoints; validación de RUT
  (mód-11), de emails y de formatos de fecha.
- **Manejo de errores:** los importadores devuelven motivos de rechazo por fila; las integraciones
  envuelven errores externos en mensajes de usuario sin filtrar detalles técnicos; timeouts
  configurables en SII.
- **Protección de datos / integridad:** importaciones con patrón **preview→commit** transaccional y
  **re-validación en servidor** (no se confía en el cliente); deduplicación por *fingerprint*/hash;
  Excel leído **sin ejecutar fórmulas ni macros**; `helmet`, rate limiting y CORS configurados.

---

## 15. Estado actual del desarrollo

| Funcionalidad | Estado | Evidencia en el proyecto | Observaciones |
|---|---|---|---|
| Gestión de proveedores (CRUD + baja lógica) | **Implementado** | `modules/proveedores/*`, `views/proveedores/*`, migración `add_proveedores` | CRUD + desactivar/reactivar; filtros y búsqueda |
| Importación masiva de proveedores (Excel/CSV) | **Implementado** | `proveedores-import.service.ts`, `import/preview` + `import/commit` | preview→commit, re-validación, 5 MB/5000 filas |
| Consulta SII de terceros | **En validación** | `sii/`, `ApiGatewaySiiProvider`, `SII_PROVIDER` | real vía API Gateway (token externo); mock operativo y por defecto |
| Asociación proveedor ↔ cuenta contable | **Parcial** | `Proveedor.numeroCuentaContable/nombreCuenta` | texto libre, **sin FK** al Plan de Cuentas |
| Facturas de compra | **Implementado** | `modules/facturas-compra/*`, `views/facturas-compra/*` | total/saldo en backend; `estado` solo `PENDIENTE` |
| Notas de crédito (regla 6 meses) | **Implementado** | `facturas-compra.service.ts` (SPEC-009) | NC = tipoDocumento + auto-relación |
| Sincronización BSale (terceros) | **Parcial** | `bsale-sync.service.ts`, `bsale-terceros.service.ts` | `BSALE_MOCK`; E2E con token real pendiente |
| Cartolas bancarias (Itaú) | **Implementado** | `cartola-bancaria/*`, `itau-adapter.ts` | solo Itaú; preview→commit; dedupe por huella |
| Liquidaciones Transbank | **Implementado** | `liquidaciones-transbank/*`, `transbank-adapter.ts` | importación; sin conciliación con POS |
| Mercado Pago (consulta de pagos) | **Implementado** | `mercado-pago/*`, `MercadoPagoPayment` | solo lectura; `MP_MOCK=true` por defecto |
| Plan de Cuentas | **Implementado** | `modules/plan-cuentas/*`, migraciones SPEC-016 | catálogo global de solo lectura; 309 cuentas |
| Arriendos y gastos comunes | **Pendiente** (prototipo) | `views/arriendos/*`, `arriendos-mock.ts` | solo frontend en memoria; sin backend |
| Pagos de servicios | **Pendiente** (prototipo) | `views/servicios/*`, `servicios-mock.ts` | solo frontend en memoria; sin backend |
| Conciliación financiera | **Pendiente** | SPEC-003 (borrador) | sin lógica en código |
| Aplicación de pagos a facturas (saldo) | **Pendiente** | — | `saldoPendiente` nunca disminuye aún |
| Reportes de gastos | **Pendiente / no verificado** | módulo genérico `reports` (NL2SQL) | no verificado como función dedicada de SAP-lite |
| Tabla de amortización de crédito | **Bloqueado** | SPEC-010 | a la espera de datos/negocio |

---

## 16. Pruebas y validación

- **Tests automáticos backend (Jest):** amplia cobertura unitaria por módulo —
  `proveedores.service.spec.ts`, `proveedores-import.service.spec.ts`, utilidades de import/CSV/formato,
  `sii.service.spec.ts`, `facturas-compra.service.spec.ts`, `bsale-sync`/`bsale-terceros.service.spec`,
  `itau-adapter.spec.ts`, `transbank-adapter.spec.ts`, `cartola-import`/`transbank-import.service.spec`,
  `mercado-pago-import.service.spec.ts`, y en `origin/main` las pruebas de `plan-cuentas/*` (parser,
  manifest, schema, service y controller). Utilidades compartidas: `rut.util.spec.ts`,
  `day-range.util.spec.ts`.
- **Tests automáticos frontend (Vitest):** stores (`proveedores`, `cartola-bancaria`,
  `facturas-compra`, `liquidaciones-transbank`, `mercado-pago`, `arriendos-mock`, `servicios-mock`,
  `plan-cuentas`) y utilidades (`utils`, `excel-preview`).
- **Integración continua:** `.github/workflows/ci.yml` ejecuta en cada PR `npx tsc --noEmit` (typecheck)
  y `npm test` sobre el backend (con `prisma generate`).
- **Datos de prueba:** `MockSiiProvider` con RUTs de comportamiento determinístico; modo `MP_MOCK`;
  semillas reales del negocio en los prototipos de arriendos/servicios; carpeta `Datos-a-cargar/`.
- **QA / pruebas manuales / evidencias:** no se verifican planes de prueba manual formales ni evidencias
  de QA en el repositorio. Tampoco se verifican pruebas E2E con credenciales reales de SII/BSale
  (marcadas como pendientes). Los prototipos (arriendos/servicios) **no tienen pruebas de persistencia**
  porque no persisten.

---

## 17. Pendientes identificados

Detectados directamente en código, specs y comentarios:

- **Plan de Cuentas (SPEC-016):** asociar efectivamente las cuentas a proveedores/arriendos/servicios
  (hoy texto libre, sin FK); definir regla “imputable/seleccionable”; resolver 4 códigos duplicados de
  la fuente; mecanismo de reemplazo del plan (write) aún no expuesto a usuario.
- **Categorización de proveedores:** `tipoCodigo` es un código de catálogo (texto), no una clasificación
  ligada al Plan de Cuentas; la categorización automática está explícitamente fuera de alcance.
- **Conciliación financiera (SPEC-003):** sin implementar (cartola ↔ facturas ↔ Transbank ↔ MP).
- **Aplicación de pagos a facturas:** el `saldoPendiente` se inicializa pero no se reduce; falta el flujo
  de pagos/abonos.
- **Arriendos (SPEC-014) y Pagos de servicios (SPEC-015):** persistencia real (modelo, migración,
  servicio, API), importadores propios (“Importar arriendos”/“Importar servicios”) y reglas de negocio;
  hoy son prototipos **en memoria**.
- **Integraciones externas:**
  - *SII:* validar el modo real (API Gateway) con token en ambiente real; en la rama de trabajo existe
    un refactor divergente no integrado (ver Hallazgos).
  - *BSale:* prueba E2E con token real pendiente; vinculación de documentos `pendienteVinculacion`
    requiere flujo de UI/resolución.
  - *Mercado Pago:* validar operación en ambiente real (token) vs. mock por defecto.
- **Reportes de gastos:** no existe una función dedicada verificable en SAP-lite.

---

## 18. Conclusión

SAP-lite presenta un **núcleo de cuentas por pagar e ingesta financiera funcional y bien estructurado**.
Las funcionalidades **operativas y persistentes** —gestión e importación de proveedores, facturas de
compra con notas de crédito, importación de cartolas Itaú, liquidaciones Transbank, consulta de pagos de
Mercado Pago y consulta del Plan de Cuentas— están implementadas sobre una arquitectura N-Tier
(NestJS + Prisma + PostgreSQL) con aislamiento multi-tenant por RLS, control de permisos por rol y un
patrón robusto de importación *preview→commit* con re-validación en servidor y pruebas unitarias en CI.

Los **componentes en validación** son principalmente las **integraciones que dependen de credenciales de
terceros**: la consulta SII real (vía API Gateway, operativa en mock y supeditada a un token externo) y
la sincronización de documentos BSale (pendiente de prueba E2E con token real); Mercado Pago opera en
modo simulado por defecto.

Las **brechas principales** son la **conciliación financiera** (SPEC-003, sin implementar), la
**aplicación de pagos** que reduzca el saldo de las facturas, la **asociación efectiva al Plan de
Cuentas** (hoy texto libre) y la **persistencia de arriendos y pagos de servicios**, que permanecen como
**prototipos de frontend en memoria**. Los **riesgos/dependencias** más relevantes son: dependencia de
servicios y tokens externos (SII/API Gateway, BSale, Mercado Pago), definiciones de negocio pendientes
(reglas de arriendos/servicios, imputabilidad de cuentas) y la **divergencia entre la rama de trabajo y
`main`** señalada a continuación. En conjunto, el módulo muestra un **avance sólido en su columna
vertebral**, con las piezas de conciliación, pagos y prototipos como trabajo pendiente claramente
acotado.

---

## 19. Evidencias técnicas sugeridas para el informe APT

Solo de funcionalidades reales detectadas:

1. **Listado y ficha de proveedores** (`ProveedoresListView` / `ProveedorDetailView`) mostrando
   búsqueda, filtro por estado y campos ampliados (SPEC-013).
2. **Previsualización de importación de proveedores** (modal de preview) con el desglose
   válidos/duplicados/rechazados y motivos por fila.
3. **Consulta SII** (`ConsultarSiiModal`) devolviendo datos de un tercero (modo mock) y el indicador
   “ya existe en el tenant”.
4. **Facturas de compra:** ficha con neto/IVA/exento/total/saldo, flag “vencida” derivado y una **nota
   de crédito** vinculada a su factura original (regla de 6 meses).
5. **Importación de cartola Itaú** (`ImportCartolaModal`): pantalla de preview con movimientos
   aceptados/rechazados/duplicados y confirmación transaccional.
6. **Importación de liquidaciones Transbank** (`TransbankTab`): historial y detalle de abonos.
7. **Mercado Pago** (`MercadoPagoView`): listado de pagos con estados/montos y resumen de la última
   sincronización.
8. **Plan de Cuentas** (`PlanCuentasListView` + `CuentaDetailModal`): consulta del catálogo global
   (código, nombre, tipo AGRUPADORA/DETALLE) con las 309 cuentas cargadas.

*(Complementariamente, para evidencia de calidad: salida de `npm test`/CI en verde y el `schema.prisma`
con las políticas RLS por tabla.)*
