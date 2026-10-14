# POS – Documentación del módulo Punto de Venta

> **Nota metodológica (lee esto primero).** Este documento se construyó revisando directamente el
> código del repositorio PittySoft: backend NestJS/Prisma, frontend `frontend/` (SPA del POS),
> esquema y migraciones de base de datos, specs (`docs/specs/pos/`) y configuración de despliegue.
> El estado descrito corresponde a la rama integrada **`origin/main`** (estado de referencia del
> proyecto) al **14 de octubre de 2026**. La rama de trabajo revisada localmente
> (`feat/prototipos-importacion-excel`) está **0 commits adelante y 51 commits por detrás de
> `origin/main`** y contiene trabajo sin commitear que **no está integrado**; para no presentar como
> integrado lo que no lo está, la documentación se ancló en `origin/main` (se verificó revisando un
> *worktree* de solo lectura de esa rama). Las diferencias entre la rama local y `origin/main` se
> detallan en la sección final *Hallazgos para revisar con el equipo*. Donde una funcionalidad no pudo
> verificarse en el código se indica de forma expresa.

---

## 1. Descripción general

El **módulo POS (Punto de Venta)** es la aplicación comercial de PittySoft para la operación diaria de
una óptica: registrar ventas en mostrador, cobrar por distintos medios de pago, emitir boletas
tributarias, gestionar clientes y productos, agendar citas de optometría y dar seguimiento a órdenes de
laboratorio.

- **Qué problema resuelve.** Centraliza y digitaliza la venta minorista óptica —que combina producto
  (marcos, cristales, accesorios) con servicio clínico (receta/historia médica, orden de laboratorio)—
  en un flujo único, multi-sucursal y multi-tenant, con cálculo de totales e IVA en el servidor,
  integración de pago con tarjeta (Mercado Pago Point) y emisión de boleta electrónica (BSale).
- **Relación con PittySoft.** El POS comparte el **mismo backend NestJS/Prisma y la base PostgreSQL**
  que el módulo administrativo-financiero SAP-lite; es decir, ambos son clientes del mismo API. El
  frontend del POS (`frontend/`) es una **SPA Vue 3 independiente**, desplegada por separado del
  frontend de SAP-lite (`sap-lite-web/`).
- **Procesos comerciales que soporta (verificables).** Gestión de clientes y productos; flujo de venta
  con carrito; medios de pago (efectivo, tarjeta vía Mercado Pago Point, transferencia, financiamiento);
  ventas por cobrar y abonos; apertura/cierre de caja; emisión de boleta BSale; citas de optometría con
  disponibilidad y optometrista; historias médicas/recetas (con OCR); órdenes de laboratorio;
  devoluciones con nota de crédito interna; dashboard de indicadores; reportes de conciliación Mercado
  Pago.
- **Usuarios de la óptica.** Según el seed de roles: **vendedor(a)/cajero(a)** (`salesperson`:
  `pos:read/write`, `clinical:read`, `lab:read`), **optometrista** (`optometrist`: `clinical:read/write`,
  `pos:read`, `lab:read`) y **administrador** (`admin`: acceso completo a POS/clínico/lab más módulos
  administrativos).

---

## 2. Alcance del módulo

Funcionalidades efectivamente presentes en el código (el estado detallado está en la §16):

| Funcionalidad | ¿Presente en el código? |
|---|---|
| Gestión de clientes (CRUD, búsqueda, orden, RUT) | **Sí** |
| Gestión de productos (catálogo, categorías, stock) | **Sí** |
| Flujo de venta (carrito, subtotal/IVA/total, descuentos) | **Sí** |
| Medios de pago (efectivo/tarjeta/transferencia/financiamiento) | **Sí** |
| Vendedoras por sucursal + asociación a la venta | **Sí** |
| Ventas por cobrar / abonos | **Sí** |
| Agenda y citas (disponibilidad, optometrista, sucursal) | **Sí** |
| Optometristas | **Sí** |
| Sucursales (locations) | **Sí** |
| Órdenes de laboratorio | **Sí** (flujo moderno + legado) |
| Devoluciones | **Sí** (nota de crédito interna; emisión BSale pendiente) |
| Historial de ventas | **Sí** |
| Dashboard / indicadores | **Sí** |
| BSale (boleta) | **Sí** (real + mock; validación sandbox en curso) |
| Mercado Pago Point | **Sí** (real + mock) |
| Impresión (orden de laboratorio térmica) | **Sí**; boleta vía PDF externo de BSale |
| Carga masiva de productos | **No** (no existe importador de productos) |
| Caja (apertura/cierre) | **Sí** |
| Historias médicas + OCR | **Sí** (OCR mediante visión LLM) |
| Reportes de conciliación Mercado Pago | **Sí** |

---

## 3. Arquitectura del módulo

### 3.1 Backend (compartido con SAP-lite)

- **Framework:** NestJS `10.4.0` sobre **Fastify**.
- **ORM / BD:** **Prisma `5.22.0`** + **PostgreSQL 15**.
- **Validación/serialización:** `class-validator 0.14`, `class-transformer 0.5` (DTOs).
- **Autenticación:** `@nestjs/jwt` + `passport-jwt`; `bcrypt`.
- **Colas/caché:** `ioredis` + Bull (`@nestjs/bull`) — p. ej. estado de intenciones de pago MP en Redis.
- **Gráficos/OCR/IA:** `chart.js` + `chartjs-node-canvas` (render de reportes), `@anthropic-ai/sdk` y
  `@google/generative-ai` (OCR de recetas y reportería NL2SQL), `@modelcontextprotocol/sdk`.
- **Otros:** `xlsx` (import/export), `helmet`, `@nestjs/throttler`, `@nestjs/swagger`.
- **Patrón:** **N-Tier** (Controller → Service → Prisma → PostgreSQL); un feature = un módulo.

**Módulos backend del POS** (en `backend/src/modules/`): `sales` (venta moderna — **el vivo**),
`payments` (Mercado Pago Point), `billing` (BSale boleta), `patients` y `clientes` (clientes),
`products` y `productos` (productos), `inventory`, `stock-documents`, `vendedoras`, `optometristas`,
`locations`, `appointments` y `citas` (citas), `historias`, `clinical`, `lab`, `ordenes-trabajo`,
`cash`, `parametros`, `reporting`, `reports`, `ocr`. Los módulos `pos` (vacío/stub), `ventas`, `citas`,
`clientes`/`productos` legados coexisten por compatibilidad (ver §4).

### 3.2 Frontend (`frontend/`)

- **Framework:** Vue `3.5` + **Vite `5.4`**; patrón **MVVM** (lógica en *stores* Pinia; HTTP en
  `src/lib/api.ts`).
- **Router:** Vue Router `4.4` con guard `requiresAuth` (JWT en `localStorage`).
- **Estado:** **Pinia `2.2`**. **HTTP:** **axios `1.7`**.
- **UI:** TailwindCSS `4.0`, Radix Vue, `lucide-vue-next`, **Chart.js** + `vue-chartjs` (gráficos del
  dashboard).
- **Testing:** **Vitest `4.1`** + Vue Test Utils + jsdom.
- **Sin librerías de PDF** (jsPDF/html2canvas): la boleta se abre como PDF externo de BSale; la orden de
  laboratorio se imprime con CSS `@media print` + `window.print()`.

**Capa HTTP (`frontend/src/lib/api.ts`):** adjunta `Authorization: Bearer <token>` y deriva
`x-tenant-id` del payload del JWT; ante `401` intenta `POST /api/v1/auth/refresh` y reintenta.
**Base URL `/`** (proxy a backend). Todos los stores llaman al API real; no hay capa mock en producción
(los mocks existen solo en los tests Vitest).

### 3.3 Base de datos

- Esquema único `backend/prisma/schema.prisma`; ~41 migraciones versionadas por timestamp.
- **Multi-tenant por Row-Level Security (RLS)**: la migración `20260704120000_rls_multitenant` habilita
  políticas `tenant_isolation` en las tablas con `tenant_id`, más un rol de solo lectura
  (`pittysoft_readonly`) para la reportería. **Excepción:** las tablas **legadas** (`clientes`,
  `maestro_productos`, `ventas_clientes`, …) **no tienen `tenant_id`** (ver §15 y Hallazgos).

### 3.4 Despliegue (verificable)

- **GCP Cloud Run** + **Artifact Registry**; IaC en **Terraform** (`infra/`: `cloudrun.tf`,
  `frontend.tf`, `network.tf`, `wif.tf`), autenticación sin llaves (**Workload Identity Federation**).
- **Backend:** `Dockerfile` multi-stage (node:20) que ejecuta `prisma migrate deploy` al arrancar;
  incluye libs de `canvas`/cairo para render de gráficos.
- **Frontend POS:** `Dockerfile` (build Vite → **nginx** que sirve la SPA y proxya `/api/*` al backend).
- **GitHub Actions:** `ci.yml` (jobs de backend `tsc --noEmit` + `jest`, y de frontends `vitest` +
  `build`), `deploy-gcp.yml` (backend), `deploy-frontend.yml` (POS), `deploy-sap-lite.yml`.
- **Local:** `docker-compose.yml` (`postgres:15`, `redis:7`).

---

## 4. Modelo de datos

El POS convive con **tablas modernas tenant-scoped** (Prisma) y **tablas legadas** heredadas de un
sistema Progress SQL (sin `tenant_id`). El flujo de venta moderno usa las primeras; las legadas
conservan datos históricos y sirven de puente por RUT/licencia.

**Entidades modernas (tenant-scoped, con RLS):**

| Entidad (modelo) | Tabla | Rol |
|---|---|---|
| `Location` | `locations` | Sucursal/bodega/ecommerce |
| `Product` / `ProductCategory` | `products` / `product_categories` | Catálogo y categorías (jerárquicas) |
| `InventoryItem` | `inventory_items` | Stock y precio por sucursal |
| `StockDocument` / `StockDocumentLine` | `stock_documents(_lines)` | Movimientos de inventario |
| `SalesOrder` / `SalesOrderItem` | `sales_orders(_items)` | **Venta y sus líneas** |
| `Invoice` | `invoices` | Documento (boleta interna / nota de crédito) |
| `Payment` | `payments` | Pagos (incluye campos Mercado Pago Point) |
| `CashSession` | `cash_sessions` | Apertura/cierre de caja |
| `Patient` / `Prescription` / `PrescriptionEye` | `patients` / `prescriptions` | Cliente clínico y receta |
| `Appointment` | `appointments` | Cita (moderna) |
| `Optometrista` | `optometristas` | Profesional clínico |
| `Vendedora` / `VendedoraSucursal` | `vendedoras` / `vendedora_sucursales` | Vendedora y asignación N:M a sucursal |
| `LabOrder` / `LabOrderItem` | `lab_orders(_items)` | Orden de laboratorio (moderna) |
| `Parametro` | `parametros` | Catálogos de configuración |

**Entidades legadas (sin `tenant_id`):** `Cliente` (`clientes`), `MaestroProducto`
(`maestro_productos`), `VentaCliente` (`ventas_clientes`), `VentaProducto` (`ventas_productos`),
`OtHistoria` (`ot_historias`), `Cita` (`citas`), `HistoriaMedica` (`historias_medicas`).

**Relaciones relevantes verificadas:**

- `SalesOrder` → `Location` (obligatoria), `Patient` (opcional) o `legacyClientRef/Name` (cliente
  legado), `Vendedora` (opcional, SPEC-021), `Prescription`/`HistoriaMedica` (opcional, SPEC-017).
- `SalesOrder 1—N SalesOrderItem`; `SalesOrder 1—N Invoice` (la boleta y, en devoluciones, notas de
  crédito); `Invoice 1—N Payment`.
- `SalesOrder 1—N LabOrder` (orden de laboratorio generada en checkout cuando hay marco + cristal).
- `Appointment` → `Patient`, `Optometrista`, `Location`.
- `InventoryItem` único por `[productId, locationId]`.

> No se incluye un DER formal porque no hay diagrama verificable en el repositorio; las relaciones se
> derivan del `schema.prisma`.

---

## 5. Gestión de clientes

**Estado: Implementado** (sistema dual). Specs: SPEC-022, SPEC-024, SPEC-025, SPEC-031, SPEC-032.

- **Dos entidades de cliente conviven:**
  - **`Patient`** (`patients`, moderna, **tenant-scoped**): API POS principal. Campos: `mrn`,
    `documentId` (RUT normalizado), `firstName`, `lastName`, `dateOfBirth`, `gender`, `email`, `phone`,
    `address`, `insuranceInfo`. Únicos `[tenantId, mrn]` y `[tenantId, documentId]`.
  - **`Cliente`** (`clientes`, **legada, sin tenant_id**): PK `licencia` (autogenerada `LIC-<uuid>`);
    campos `nombrePaciente`, `documentoIdentidad` (RUT), teléfonos, `fechaNacimiento`,
    `fechaUltimoExamen`, contadores `cantidadCompras`/`cantidadOrdenesTrabajo`.
  - **Regla:** al crear un `Patient` (SPEC-031) se **auto-registra** un `Cliente` por RUT
    (`ClientesService.resolverOCrear`); el carrito/checkout del POS referencia al cliente por
    `legacyClientRef` (licencia) y `clienteRut`.
- **Registro / edición / búsqueda:** endpoints REST (`/patients`, `/clientes`) con crear, actualizar,
  eliminar, listar y buscador liviano para "Nueva Cita" (SPEC-024). Búsqueda por nombre y RUT ignorando
  acentos y con *tokens* (`unaccent`).
- **Ordenamiento (SPEC-025):** `recientes` (default), `antiguos`, `nombre_asc`, `nombre_desc`;
  paginación `page`/`limit` (máx. 100).
- **RUT y dígito verificador (SPEC-032):** utilitario `backend/src/shared/rut.util.ts` con
  `isValidRut()` (validación **módulo 11**), `normalizeRut()` (formato `CUERPO-DV`) y
  `rutQueryVariants()` (variantes con/sin puntos y guion para deduplicar sin alterar la BD). El cálculo
  automático del DV está implementado en esta utilidad.
- **Normalización / deduplicación:** por RUT normalizado; el `Cliente` legado no tiene RUT único pero se
  deduplica por variantes de formato (SPEC-022).
- **Relación con ventas y citas:** `SalesOrder.patientId`/`legacyClientRef`; `Appointment.patientId` y
  `Cita.licencia` (legada). Historial de compras del cliente vía `GET /sales/cliente/{licencia}`.

---

## 6. Gestión de productos

**Estado: Implementado** (sistema dual). Carga masiva: **Pendiente**.

- **Dos entidades de producto conviven:**
  - **`Product`** (`products`, moderna, **tenant-scoped**): catálogo vivo del POS. Campos: `sku`
    (autogenerado si falta, único por tenant), `name`, `type` (enum **`frame | lens | contact_lens |
    accessory | service`**), `brand` (**texto libre**, no hay entidad Marca), `basePrice`, `cost`,
    `supplier` (texto), `currency` (default `USD`), `isActive` (baja lógica), `image`, `attributes`
    (JSON), `categoryId` → `ProductCategory` (jerárquica, opcional).
  - **`MaestroProducto`** (`maestro_productos`, **legada, sin tenant_id**): catálogo histórico
    (referencia/código, marca, `precioVentSinIva`, `cantDisp`, `tipoProd`…).
- **Categorías / marcas:** categorías como entidad `ProductCategory` con jerarquía padre-hijo; marcas
  como **texto libre** en `Product.brand`.
- **Precios:** `Product.basePrice` con posibilidad de **override por sucursal** en
  `InventoryItem.price`.
- **Stock:** gestionado en `InventoryItem` por `[producto, sucursal]` (`quantity`, `reservedQty`,
  `reorderPoint`; disponible = `quantity − reservedQty`); ajustes por `inventory` y movimientos por
  `stock-documents` (ASN/recepción/traslado/devolución/ajuste, con confirmación atómica).
- **Búsqueda:** `GET /products?search=` tokenizada (AND) sobre `name`/`sku`/`brand`, sin sensibilidad a
  acentos; detalle con stock por sucursal.
- **Carga masiva:** **no implementada** para productos (no hay endpoint de importación CSV/Excel). Solo
  el módulo de proveedores de SAP-lite tiene importador; productos se crean uno a uno.
- **Relación con la venta:** cada `SalesOrderItem` referencia `productId` (moderno) o usa líneas opacas
  (`legacyName`/`legacySku`) cuando el ítem proviene del carrito legado.

---

## 7. Flujo de venta

**Estado: Implementado.** Módulo **`sales`** (el módulo `pos` es un stub vacío; `ventas` es legado).
Specs: SPEC-001 (modal de pago), SPEC-017 (receta/OT), SPEC-020/021 (vendedora), SPEC-010 (boleta).

Pantalla principal `frontend/src/views/pos/PosRetailView.vue`, con tres columnas: **selección de
cliente** (buscador + ficha + receta disponible), **catálogo de productos** (búsqueda, filtro por tipo,
tarjetas con precio con IVA y badge de stock) y **carrito** (ítems con cantidad/descuento, vendedora,
subtotal, IVA 19 %, total).

- **Selección de cliente:** buscador contra `GET /clientes/buscar`; muestra ficha, compras previas y
  receta (OD/OI) si existe.
- **Productos y carrito:** el carrito (`posCart` store) es estado local; calcula IVA
  `round(subtotal × 1.19)` para coincidir exactamente con el backend.
- **Vendedora (SPEC-020/021):** se selecciona al iniciar la venta (`SelectVendedoraModal`) y queda
  asociada a la `SalesOrder` (`vendedoraId`).
- **Sucursal:** toda venta se liga a una `Location` (del usuario POS).
- **Totales (autoridad del backend):** el servicio `SalesService.checkoutPos/checkout` recalcula
  subtotal neto = Σ(`unitPrice`×`qty` − descuento), **total = `round(subtotal × 1.19)`**, IVA = total −
  subtotal (**redondeo único** para evitar divergencias de decimales). El frontend no puede dictar
  totales.
- **Descuentos:** por línea, en porcentaje (`discountPct`) convertido a monto neto.
- **Medios de pago y confirmación:** ver §8; el `PaymentModal` valida que lo registrado cubra el total
  (salvo abono) y calcula el vuelto del efectivo.
- **Generación de la venta (transacción atómica):** crea `SalesOrder` (confirmada) + `SalesOrderItem[]`,
  descuenta stock de los tipos controlados (`frame`/`accessory`), crea `Invoice` interna (`V-000001`) y
  `Payment[]`; condicionalmente genera `LabOrder` y órdenes de trabajo (`OtHistoria`) si hay marco +
  cristal + historia (SPEC-017). Usa `idempotencyKey` para evitar doble registro.
- **Documentos generados:** boleta interna siempre; **boleta BSale** solo si la venta queda pagada
  (emisión posterior y **no bloqueante**, ver §9).

---

## 8. Medios de pago y Mercado Pago

**Estado: Implementado** (modo real y simulado configurables). Specs: SPEC-006, SPEC-007, SPEC-014.

- **Métodos soportados** (`Payment.method`): **`cash` (efectivo), `card` (tarjeta), `transfer`
  (transferencia), `financing` (financiamiento)**. Tarjeta manual guarda `cardType`
  (débito/crédito), `cardProvider` (visa/mastercard/amex), `authCode`, `installments` (cuotas).
- **Pago parcial / ventas por cobrar (SPEC-014/003):** una venta puede quedar con saldo (`abono=true`);
  `GET /sales/por-cobrar` lista los saldos pendientes y `POST /sales/:id/abonar` o `/pagar-saldo`
  registran abonos; la boleta se emite al llegar el saldo a cero.
- **Mercado Pago Point (tarjeta por lector):**
  - **Real vs simulado:** controlado por **`MP_MOCK`** (por defecto `true` = simulado). El modo real
    requiere `MP_ACCESS_TOKEN`, `MP_DEVICE_ID` (y opcional `MP_BASE_URL`).
  - **Flujo:** el frontend crea la intención (`POST /payments/point/intent`), hace *polling* cada 3 s
    (`GET .../intent/:id`) mostrando spinner; estados `OPEN → ON_TERMINAL → PROCESSING → FINISHED |
    CANCELED | ERROR`. El estado se guarda en **Redis** (TTL `MP_INTENT_TTL_S`, default 600 s),
    tenant-scoped.
  - **Validación server-side:** en el checkout el backend **verifica la intención** (`state=FINISHED` y
    `payment.status=approved`); si MP rechaza → `400` (no se crea la venta); si no responde → `502`. No
    es posible falsear la aprobación desde el frontend.
  - **Resultado y conciliación:** si se aprueba, enriquece el `Payment` con `mpPaymentId`,
    `mpPaymentType` (crédito/débito), `mpCardLast4`, `mpInstallmentAmount`, `mpFee`, `mpNetAmount`
    (comisión/neto pueden llegar nulos y se muestran como "Pendiente").
  - **Ambiente:** si está en modo real, apunta a la API de Mercado Pago según el token configurado
    (sandbox o producción **no es determinable solo desde el código**). El modo por defecto es simulado.
- **Asociación con la venta:** los pagos se crean bajo la `Invoice` de la `SalesOrder`.

---

## 9. Integración con BSale

**Estado: Implementado (boleta, real + mock); validación en sandbox real en curso; nota de crédito por
BSale Pendiente.** Specs: SPEC-010, SPEC-012, SPEC-034. Módulo `billing` (`BsaleService`).

- **Emisión de boletas:** tras un checkout **pagado**, el backend emite la boleta en BSale de forma
  **asíncrona y no bloqueante** (si falla, la venta no se revierte; el error se guarda en
  `Invoice.fiscalData.bsale.error`, sin exponer el token).
- **Real vs mock:** controlado por **`BSALE_MOCK`** (por defecto `true`). El mock genera un folio
  determinístico; el modo real (`BSALE_TOKEN`, `BSALE_OFFICE_ID`, `BSALE_IVA_TAX_ID`, `BSALE_BASE_URL`)
  llama a la API de BSale.
- **Tipo de documento:** **boleta electrónica (`codeSii = 39`)** — el único tipo soportado en SPEC-010.
- **Folios y relación con la venta:** el folio/URL/estado quedan en `Invoice.fiscalData.bsale`
  (`emitida`, `folio`, `bsaleDocId`, `urlPdf`, `informedSii`, `error`), vinculados a la `SalesOrder`
  original. El cliente de la boleta se nomina por RUT chileno válido (nunca por licencia) o queda como
  consumidor final.
- **Conversión de montos:** PittySoft maneja montos **brutos**; convierte a **neto** (`/1.19`) porque
  BSale reconstruye el IVA.
- **Ambiente de prueba (SPEC-012):** existe la spec "Validación de boleta BSale en sandbox real"; el
  modo real depende de credenciales y el valor por defecto es mock ⇒ **en validación**.
- **Facturas:** **no implementadas** (solo boleta `codeSii 39`).
- **Notas de crédito (devolución, SPEC-034):** la nota de crédito se **crea internamente** (ver §11),
  pero su **emisión en BSale está pendiente** (marcada explícitamente en el código:
  `fiscalData.devolucion.bsale.pendiente = true`).

**Diferenciación clara:** *Implementado* → emisión de boleta (real/mock), folios, vínculo con la venta.
*En validación* → operación real en sandbox/producción (depende de token). *Pendiente* → factura y
emisión de nota de crédito por BSale.

---

## 10. Gestión de citas

**Estado: Implementado** (módulo **`appointments`**, moderno; `citas` es legado). Specs: SPEC-002,
SPEC-005, SPEC-023, SPEC-026, SPEC-027, SPEC-031, SPEC-033.

- **Entidad `Appointment`:** `scheduledAt` (instante exacto), `durationMin` (default 30), `type`
  (**`exam | follow_up | contact_fitting | dispense`**), `status` (**`scheduled | confirmed | arrived |
  in_progress | done | no_show | cancelled`**), `patientId`, `optometristaId` (opcional), `locationId`,
  `providerId`, `notes`.
- **Creación y edición:** `GET/POST /appointments`, `PATCH /appointments/:id` (cambio de estado /
  reagendar). Ordenamiento `recientes`/`antiguas` (SPEC-026); actualización de estado de citas pasadas
  (SPEC-027).
- **Búsqueda de clientes / creación desde la cita:** flujo unificado (SPEC-023/024); se puede **crear un
  cliente POS desde "Nueva Cita"** (SPEC-031), lo que crea `Patient` + `Cliente` por RUT.
- **Sucursal y optometrista:** la disponibilidad se calcula por **sucursal + optometrista**; los
  catálogos se cargan dinámicamente (SPEC-005).
- **Intervalos disponibles (regla real, SPEC-033):** la granularidad de los horarios es de **10
  minutos** (constante `SLOT_MIN = 10` en `appointments.service.ts`), en un horario de **09:00 a
  19:30** (hora de Chile). El método `disponibles()` genera todos los bloques de 10 min del día y marca
  cada uno con `ocupado`. *(Observación: un comentario del frontend describe bloques de "30 min"; la
  regla vigente verificada en el backend y en la spec SPEC-033 es de 10 minutos.)*
- **Prevención de reservas duplicadas:** `assertSlotLibre()` rechaza (`409 Conflict`) si ya existe una
  cita **no cancelada** con el mismo `(tenantId, locationId, optometristaId, scheduledAt)`; distintos
  optometristas o sucursales en el mismo instante sí se permiten; al reagendar se excluye la propia
  cita.

---

## 11. Devoluciones

**Estado: Implementado (nota de crédito interna); emisión BSale y reembolso Pendientes.** Spec: SPEC-034.
Módulo `sales` + `frontend/src/views/pos/DevolucionesView.vue`.

- **Búsqueda y selección de venta:** `GET /sales` (lista) y `GET /sales/:orderId/devolucion`
  (previsualización): carga la venta original y, por ítem, la **cantidad disponible a devolver** (compra
  menos lo ya devuelto en notas de crédito previas).
- **Selección de productos:** el usuario elige ítems y cantidades (`{ orderItemId, cantidad }`) con
  validación `cantidad ≤ (comprada − ya devuelta)`.
- **Registro interno / nota de crédito:** `POST /sales/:orderId/devolucion` crea una `Invoice` con
  `status = 'credit_note'`, folio `NC-000000`, montos **en negativo** (neto + IVA 19 %, con la última
  línea absorbiendo el redondeo), y guarda el detalle en `fiscalData.devolucion` (ítems, motivo y
  referencia al documento original).
- **Asociación con el documento original:** la nota de crédito queda ligada a la misma `SalesOrder`; la
  venta original **no se modifica** (trazabilidad).
- **Integración BSale:** **pendiente** — el código marca `fiscalData.devolucion.bsale.pendiente = true`
  con mensaje "Emisión de Nota de Crédito en BSale pendiente de integración (SPEC-034)".
- **Reembolso de dinero:** fuera de alcance (no implementado).
- **Elegibilidad:** solo ventas confirmadas/entregadas (no `draft` ni `cancelled`).

---

## 12. Dashboard e indicadores

**Estado: Implementado.** Specs: SPEC-008, SPEC-009, SPEC-013, SPEC-035. Datos obtenidos por agregación
sobre las tablas del propio POS (no hay un data warehouse separado).

Indicadores verificados (endpoints del módulo `sales`, salvo indicación):

- **Ventas del día / facturación acumulada / boletas emitidas / devoluciones / descuentos:**
  `GET /sales/resumen-dia` → `totalVentas`, `cantidadBoletas`, `totalDevoluciones` (ventas canceladas
  del día), `totalDescuentos`, `totalHistorico`, `cantidadBoletasHistorico` (filtra estados ≠
  draft/cancelled, zona horaria Chile).
- **Ventas por hora:** `GET /sales/ventas-por-hora` (desglose 0–23 h).
- **Ventas por cobrar:** `GET /sales/por-cobrar` (ventas con `saldoPendiente > 0`, con total y conteo).
- **Ticket promedio por vendedora:** `GET /sales/ticket-promedio-vendedora?periodo=dia|mes|ano`
  (incluye todas las vendedoras activas, excluye ventas sin vendedora).
- **Ranking de ticket promedio (SPEC-035):** `GET /sales/ranking/ticket-promedio`.
- **Conciliación Mercado Pago (SPEC-013):** módulo `reporting` → `GET /reporting/mercado-pago`
  (`ventasBrutas`, `comisiones`, `neto`, `transacciones`) a partir de la tabla `Payment` filtrada por
  pagos Point (`mpIntentId` no nulo).
- **Citas y laboratorio:** no se verificó un indicador agregado dedicado de "citas" o "laboratorio" en
  el dashboard de ventas; esos módulos exponen sus propios listados/colas (ver §10 y módulo `lab`).
- **Reportería libre (NL2SQL):** módulo `reports` → `POST /reports/ask` convierte lenguaje natural a SQL
  (solo `SELECT`, validado) ejecutado contra una BD de solo lectura con RLS, usando un LLM; devuelve
  filas + tipo de gráfico. Requiere `DATABASE_READONLY_URL` y clave de LLM.

---

## 13. Impresión

- **Boleta:** la emite BSale (backend). El frontend muestra un enlace **"Ver / imprimir PDF"** al
  `urlPdf` externo de BSale (`SaleConfirmationModal`). **No hay librería de PDF en la app** ni
  integración de **impresora térmica para la boleta**; la impresión queda a cargo del visor de PDF del
  navegador. *Estado: boleta Implementada (PDF externo); impresión térmica de boleta Pendiente.*
- **Orden de laboratorio:** `frontend/src/views/clinical/ordenes/OrdenPrintView.vue` imprime en
  **formato térmico 80 mm** usando CSS `@media print` + `window.print()` (encabezado, paciente, fórmula
  OD/OI, laboratorio, firma). *Estado: Implementado.*
- **Comprobante de abono:** no se verificó un formato/impresión dedicado de comprobante de abono (el
  `PaymentModal`/confirmación muestra el resultado en pantalla). *Estado: Pendiente / no verificado.*
- **Backend:** `chartjs-node-canvas` genera imágenes de gráficos para la reportería; no hay un servicio
  de generación de PDF de boletas propio (lo provee BSale).
- **Configuración de impresión:** solo CSS `@media print` para la orden de laboratorio; sin
  configuración de impresora/driver en el repositorio.

---

## 14. Integraciones externas

| Integración | Estado | Detalle verificable |
|---|---|---|
| **BSale (boleta)** | **Implementada; validación sandbox real en curso** | `billing`/`BsaleService`; `BSALE_MOCK` default `true`; `codeSii 39`; emisión no bloqueante; folios en `Invoice.fiscalData`. |
| **BSale (factura / nota de crédito)** | **Pendiente** | Factura no soportada; emisión de NC por BSale marcada pendiente (SPEC-034). |
| **Mercado Pago Point** | **Implementada (real + mock)** | `payments`; `MP_MOCK` default `true`; intención en Redis; verificación server-side; ambiente real sujeto a token. |
| **Reportería LLM (OCR y NL2SQL)** | **Implementada (sujeta a credenciales)** | `ocr` (visión de recetas) y `reports` (NL2SQL) usan `@anthropic-ai/sdk`/`@google/generative-ai`; requieren claves. |
| **Laboratorio (externo)** | **Parcial** | `LabOrder` modela `supplierId`/`shippingTracking`/`externalRef`; no se verificó integración API con un laboratorio externo (gestión interna de estados). |
| **Servicios de impresión** | **No integrados** | Solo impresión del navegador (térmica para OT); sin servicio/hardware integrado. |

*No se exponen tokens ni credenciales; solo se nombran las claves de configuración.*

---

## 15. Seguridad y reglas de negocio

- **Autenticación:** JWT (access + refresh) con Passport; `JwtAuthGuard` global que respeta `@Public()`.
  El frontend adjunta el token y refresca ante `401`.
- **Roles y permisos:** modelo `Role.permissions` (JSON) con permisos `modulo:accion`; decorador
  `@RequirePermission()` + `PermissionsGuard`. Roles del seed: `admin`, `optometrist`, `salesperson`
  (POS/clinical/lab con distintos niveles).
- **Multi-tenant:** RLS de PostgreSQL (migración `20260704120000_rls_multitenant`) sobre las tablas con
  `tenant_id`, más scope por `tenantId` a nivel de servicio (defensa en profundidad). **Caveat
  verificado:** las **tablas legadas** (`clientes`, `maestro_productos`, `ventas_clientes`, etc.) **no
  tienen `tenant_id` ni RLS** — ver Hallazgos.
- **Validaciones:** DTOs con `class-validator`; validación y normalización de RUT (módulo 11, SPEC-032);
  cálculo de totales/IVA en el servidor; verificación server-side de intenciones de pago MP; control de
  stock antes de descontar; idempotencia de checkout.
- **Protección de datos / integridad:** operaciones de venta en transacción atómica; emisión de boleta
  no bloqueante con el error acotado en `fiscalData`; la venta original no se altera en devoluciones;
  los errores de integraciones externas se envuelven sin filtrar tokens.
- **Manejo de errores:** excepciones NestJS (`400/409/502/503`) con mensajes de usuario; el frontend
  extrae errores de validación para mostrarlos.

---

## 16. Estado actual del desarrollo

| Funcionalidad | Estado | Evidencia en el proyecto | Observaciones |
|---|---|---|---|
| Gestión de clientes (Patient/Cliente, RUT, DV) | **Implementado** | `patients/*`, `clientes/*`, `shared/rut.util.ts` | Sistema dual; DV módulo 11 (SPEC-032) |
| Gestión de productos (catálogo, stock) | **Implementado** | `products/*`, `inventory/*`, `stock-documents/*` | `Product` moderno + `MaestroProducto` legado |
| Carga masiva de productos | **Pendiente** | — | No existe importador de productos |
| Flujo de venta (carrito, totales, IVA) | **Implementado** | `sales/sales.service.ts`, `views/pos/PosRetailView.vue` | Totales calculados en backend (redondeo único) |
| Medios de pago (efectivo/tarjeta/transf./financ.) | **Implementado** | `payments/*`, `views/pos/PaymentModal.vue` | Vuelto, multipago, abono |
| Mercado Pago Point | **Implementado** | `payments/mercadopago.service.ts` | `MP_MOCK` default; real sujeto a token |
| Ventas por cobrar / abonos | **Implementado** | `GET /sales/por-cobrar`, `/abonar`, `/pagar-saldo` | SPEC-003/014 |
| Vendedoras por sucursal + asociación | **Implementado** | `vendedoras/*`, `SalesOrder.vendedoraId` | SPEC-019/020/021 |
| Sucursales (locations) | **Implementado** | `locations/*` | branch/warehouse/ecommerce |
| Optometristas | **Implementado** | `optometristas/*` | baja lógica preserva historial |
| Citas (disponibilidad, 10 min) | **Implementado** | `appointments/appointments.service.ts` | `SLOT_MIN=10` (SPEC-033); anti-doble-reserva |
| Historias médicas + OCR | **Implementado** | `historias/*`, `ocr/*` | OCR por visión LLM |
| Órdenes de laboratorio | **Implementado** | `lab/*`, `ordenes-trabajo/*` | Flujo moderno `LabOrder` + legado `OtHistoria` |
| Caja (apertura/cierre) | **Implementado** | `cash/*`, `CashSession` | SPEC-011 |
| Boleta BSale | **En validación** | `billing/bsale.service.ts` | Implementada; sandbox real sujeto a token (SPEC-012) |
| Factura electrónica BSale | **Pendiente** | — | Solo boleta `codeSii 39` |
| Nota de crédito (devolución) | **Parcial** | `sales/devolucion.util.ts` | NC interna sí; emisión BSale pendiente (SPEC-034) |
| Devoluciones (flujo) | **Implementado** | `views/pos/DevolucionesView.vue`, `/sales/:id/devolucion` | Reembolso fuera de alcance |
| Dashboard / indicadores | **Implementado** | `sales` (resumen-dia, por-hora, por-cobrar, ticket) | Agregaciones sobre tablas POS |
| Reportes conciliación MP | **Implementado** | `reporting/reporting.service.ts` | SPEC-013 |
| Reportería NL2SQL | **Implementado** | `reports/reports.service.ts` | Requiere BD readonly + clave LLM |
| Impresión orden de laboratorio (térmica) | **Implementado** | `OrdenPrintView.vue` (`@media print`) | 80 mm, `window.print()` |
| Impresión térmica de boleta | **Pendiente** | — | Boleta como PDF externo de BSale |
| Comprobante de abono (impresión) | **Pendiente / no verificado** | — | No se verificó formato dedicado |
| Suite E2E | **Pendiente** | — | Solo tests unitarios |

---

## 17. Pruebas y validación

- **Tests automáticos backend (Jest):** ~45 archivos `.spec.ts`. Cobertura POS verificada en: `sales`
  (checkout, carrito, totales, pagos, abonos, devoluciones), `payments` (MP Point, mock), `billing`
  (BSale mock + conversión neto), `appointments` (slots, conflictos), `patients` y `clientes` (RUT,
  dedup, tenant), `vendedoras`, `optometristas`, `historias`, `reporting`. Utilidades: `rut.util`,
  `devolucion.util`.
- **Tests automáticos frontend (Vitest):** ~27 archivos. Cubre stores (`posPayment`, `posCart`,
  `clientes`, `ventas`, `devoluciones`, `appointments`, `ventasPorCobrar`, `parametros`, `historias`) y
  algunas vistas (`SaleConfirmationModal`, `ClienteDetailView`), con el API mockeado.
- **Integración continua:** `.github/workflows/ci.yml` corre, en cada PR, el backend (`tsc --noEmit` +
  `jest`) y ambos frontends (`vitest` + `build`).
- **Datos de prueba:** clientes de prueba de Mercado Pago (SPEC-007, `mp-test-clientes`), folios/intents
  determinísticos en modo mock (BSale/MP).
- **Cobertura parcial / faltante:** `products`, `inventory` y `stock-documents` no muestran archivos de
  test dedicados en la revisión. **No existe una suite E2E**; el gate de CI es de tests unitarios.
- **QA / evidencias con usuarios:** no se verifican planes de prueba manual formales ni evidencias de QA
  con usuarios en el repositorio. La validación de BSale en sandbox real (SPEC-012) figura como trabajo
  pendiente.

---

## 18. Pendientes identificados

Pendientes verificables en código/specs:

- **Nota de crédito mediante BSale:** la NC se crea internamente pero **no se emite en BSale**
  (`fiscalData.devolucion.bsale.pendiente = true`, SPEC-034).
- **Factura electrónica:** no implementada (solo boleta `codeSii 39`); facturas y otros documentos
  (NC/ND) quedan fuera del alcance actual de la integración BSale.
- **Validación de boleta BSale en sandbox real (SPEC-012):** pendiente de ejecutar con credenciales
  reales; el modo por defecto es mock.
- **Carga masiva de productos:** no existe importador CSV/Excel para `products`.
- **Impresión térmica de boleta y comprobante de abono:** no implementadas (la boleta se abre como PDF
  externo; solo la orden de laboratorio tiene formato térmico).
- **Reembolso de dinero en devoluciones:** fuera de alcance.
- **Integraciones externas:** Mercado Pago Point y BSale operan en mock por defecto (validar modo real
  con token); no se verificó integración API con un laboratorio externo.
- **Pruebas finales con usuarios y suite E2E:** no presentes en el repositorio.
- **Deuda de datos multi-tenant:** las tablas legadas (`clientes`, `maestro_productos`,
  `ventas_clientes`) carecen de `tenant_id`/RLS (ver Hallazgos).

---

## 19. Conclusión

El módulo POS presenta un **flujo de venta completo y operativo** de principio a fin: selección de
cliente y productos, carrito con cálculo de totales e IVA **en el servidor**, múltiples medios de pago
(incluido tarjeta por Mercado Pago Point con verificación server-side), registro transaccional de la
venta, emisión de boleta electrónica BSale, ventas por cobrar/abonos, caja, citas de optometría con
disponibilidad de 10 minutos y prevención de doble reserva, historias médicas con OCR, órdenes de
laboratorio, devoluciones con nota de crédito interna y un dashboard de indicadores con reportería.
Todo sobre una arquitectura N-Tier (NestJS + Prisma + PostgreSQL) multi-tenant con RLS, roles/permisos y
pruebas unitarias en CI.

Los **componentes en validación** son esencialmente las **integraciones tributarias y de pago que
dependen de credenciales de terceros**: BSale (boleta en sandbox/producción real) y Mercado Pago Point
operan en modo simulado por defecto. Las **brechas/pendientes** más relevantes son la **emisión de la
nota de crédito por BSale** (hoy la devolución queda como documento interno), la **factura
electrónica**, la **carga masiva de productos**, la **impresión térmica de boleta/comprobante de
abono** y la **suite E2E / pruebas con usuarios**. Los **riesgos y dependencias** principales son: la
dependencia de tokens de BSale y Mercado Pago (y de claves LLM para OCR/reportería), la **deuda
multi-tenant en las tablas legadas** sin `tenant_id`, y la **divergencia entre la rama local y
`origin/main`** descrita a continuación. En conjunto, el POS muestra un **nivel de avance alto en su
operación central**, con las piezas tributarias (NC/factura) y algunas de impresión/validación como
trabajo pendiente acotado.

---

## 20. Evidencias técnicas sugeridas para el informe APT

Solo de funcionalidades reales detectadas:

1. **Home / navegación del POS** (`HomeView`) con los accesos por permisos de rol.
2. **Dashboard** (`DashboardView`): ventas del día, facturación acumulada, boletas, descuentos,
   devoluciones y ventas por hora (gráfico Chart.js).
3. **Modo Venta** (`PosRetailView`): panel de cliente + catálogo + carrito con subtotal, IVA 19 % y
   total, y selección de vendedora.
4. **Modal de pago** (`PaymentModal`): multipago, vuelto de efectivo y procesamiento de tarjeta Mercado
   Pago Point (spinner "Esperando el pago en el terminal…" y resultado aprobado/rechazado).
5. **Confirmación de venta** (`SaleConfirmationModal`) con el enlace a la **boleta BSale** (folio +
   PDF).
6. **Clientes** (`ClientesListView` / `ClienteDetailView`): búsqueda, orden y ficha con historial de
   compras; alta con RUT y dígito verificador.
7. **Citas** (`CitasListView` + "Nueva Cita"): disponibilidad en bloques de **10 minutos** por sucursal
   y optometrista, con prevención de doble reserva.
8. **Devoluciones** (`DevolucionesView`): selección de la venta original, ítems disponibles a devolver y
   nota de crédito interna generada.

*(Complementariamente, para evidencia de calidad: salida de `npm test`/CI en verde, la orden de
laboratorio en formato térmico `@media print`, y el reporte de conciliación Mercado Pago.)*
