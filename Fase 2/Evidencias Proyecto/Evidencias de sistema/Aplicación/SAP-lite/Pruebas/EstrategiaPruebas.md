# Estrategia de Pruebas — SAP-lite (Control de Gestión)

> **Evidencia académica — Proyecto CAPSTONE.** Este documento describe la
> estrategia de pruebas **real** del módulo SAP-lite — Control de Gestión de
> PittySoft, verificada contra el código del proyecto. Los recuentos de archivos
> son reales; **no** se inventan porcentajes de cobertura ni cantidades de
> pruebas aprobadas.

## 1. Herramientas utilizadas

| Capa | Framework de pruebas | Herramientas de apoyo |
| --- | --- | --- |
| Backend (NestJS) | **Jest** (con `ts-jest`) | `tsc --noEmit` (typecheck), ESLint |
| Frontend (`sap-lite-web`, Vue 3) | **Vitest** (entorno `jsdom`) | `vue-tsc --noEmit` (typecheck), ESLint, Vite (build) |

- **Backend:** archivos `*.spec.ts` ubicados junto al código que prueban;
  configuración en `backend/jest.config.js`. El acceso a datos se **mockea**
  (no hay base de datos real en las pruebas unitarias).
- **Frontend:** archivos `*.test.ts` dentro de carpetas `__tests__/`;
  configuración embebida en `sap-lite-web/vite.config.ts`. Las pruebas de
  *stores* Pinia mockean el cliente HTTP.

## 2. Organización de las pruebas

### 2.1 Backend
La *suite* del backend contiene **78** archivos de prueba (`*.spec.ts`) en total.
De ellos, **al menos 34 corresponden directamente** a módulos de SAP-lite —
Control de Gestión, distribuidos así:

| Módulo | Archivos de prueba |
| --- | --- |
| `proveedores` (+ `sii`) | 8 — servicio, DTO, importación CSV/formato, "reciente", servicio SII y *provider* API Gateway |
| `facturas-compra` | 4 — servicio, DTO y sincronización/terceros BSale |
| `cartola-bancaria` | 2 — servicio de importación y adaptador Itaú |
| `liquidaciones-transbank` | 2 — servicio de importación y adaptador Transbank |
| `mercado-pago` | 2 — servicio de importación y DTO |
| `plan-cuentas` | 6 — servicio, controller, *schema*, cuentas excluidas, *manifest* y *parser* del plan maestro |
| `servicios` | 7 — servicios, tipos de servicio y cobros (servicio + permisos + DTO) |
| `arriendos` | 2 — servicio y DTO |
| `reporting` | 1 — servicio de reportería financiera |

Se apoyan además en utilidades compartidas con pruebas propias:
`shared/rut.util.spec.ts`, `shared/day-range.util.spec.ts` y
`locations/dto/location.dto.spec.ts`.

**Tipo de pruebas:** todas son **unitarias** (lógica de negocio en aislamiento,
con Prisma mockeado). Varios archivos referencian explícitamente los criterios de
aceptación de las SPECs (p. ej. CA-2, CA-4, CA-5 en facturas de compra).

### 2.2 Frontend (`sap-lite-web`)
`sap-lite-web` es la aplicación frontend **del módulo de Control de Gestión**;
sus **20** archivos de prueba corresponden a SAP-lite:

- *Stores* Pinia: `proveedores`, `facturas-compra`, `cartola-bancaria`,
  `liquidaciones-transbank`, `mercado-pago`, `plan-cuentas`, `servicios`,
  `tipos-servicio`, `cobros`, `arriendos`, `auth`.
- Vistas/componentes: `views/servicios/` (`ServiciosListView`,
  `ServicioDetailView`, `CobroDocumentosModal`), `views/plan-cuentas/`
  (`plan-cuentas.constants`) y `components/shared/CuentaContableSelector`.
- Utilidades (`lib/__tests__/`): `excel-preview`, `location-options`, `nav`,
  `utils`.

**Tipo de pruebas:** de **integración a nivel de *store*/contrato de API** (estado
de Pinia + llamadas mockeadas) y de componentes/utilidades.

## 3. Funcionalidades verificadas por pruebas

| Funcionalidad | ¿Cubierta por pruebas? | Evidencia (archivo) |
| --- | --- | --- |
| Registro, edición y validación de proveedores | ✅ | `proveedores.service.spec.ts`, `dto/proveedores.dto.spec.ts` |
| Importación masiva de proveedores | ✅ | `proveedores-import.service.spec.ts`, `import-csv.util.spec.ts`, `import-format.util.spec.ts` |
| Registro de facturas de compra | ✅ | `facturas-compra.service.spec.ts` |
| Total y saldo de facturas de compra | ✅ | `facturas-compra.service.spec.ts` (neto, IVA y exento son datos de entrada; el `total` se calcula como `neto + IVA + exento` y el `saldoPendiente` se recalcula en el backend) |
| Pagos parciales y completos / anulación y recálculo de saldo | ✅ | `facturas-compra.service.spec.ts` (motor `PagoCompra` / recálculo de saldo) |
| Registro de servicios y cobros | ✅ | `servicios.service.spec.ts`, `cobros.service.spec.ts` |
| Vinculación de facturas a servicios | ✅ | `cobros.service.spec.ts` (`CobroDocumento`), front `CobroDocumentosModal.test.ts` |
| Permisos de Servicios/Cobros | ✅ | `servicios.permissions.spec.ts`, `cobros.permissions.spec.ts`, `tipos-servicio.permissions.spec.ts` |
| Gestión de arriendos | ✅ | `arriendos.service.spec.ts`, `dto/arriendos.dto.spec.ts` |
| Importación de cartolas Itaú (duplicados, filas rechazadas) | ✅ | `cartola-import.service.spec.ts`, `itau-adapter.spec.ts` |
| Importación de liquidaciones Transbank (duplicados, filas rechazadas) | ✅ | `transbank-import.service.spec.ts`, `transbank-adapter.spec.ts` |
| Plan de Cuentas (carga, cuentas excluidas) | ✅ | `plan-cuentas.service.spec.ts`, `cuentas-excluidas.spec.ts`, `plan-maestro.parser.spec.ts` |
| Consulta de pagos Mercado Pago | ✅ | `mercado-pago-import.service.spec.ts` |
| Normalización/validación de RUT | ✅ | `shared/rut.util.spec.ts` |
| **Conciliación bancaria (SPEC-003)** | ❌ | **No implementada → sin pruebas** |

> Las pruebas anteriores corresponden a funcionalidades realmente presentes en el
> código. No se afirma la existencia de pruebas para la conciliación bancaria
> porque esa funcionalidad no está implementada.

**Permisos y aislamiento multi-tenant — distinción necesaria.** El control de
acceso y el *scope* por *tenant* se ejercitan a **nivel unitario**: las pruebas
de permisos (`servicios.permissions.spec.ts`, `cobros.permissions.spec.ts`,
`tipos-servicio.permissions.spec.ts`) verifican que los endpoints exijan los
permisos correspondientes, y varias pruebas de servicio comprueban que las
consultas se filtren por `tenant_id` (p. ej. el criterio de aislamiento en
`proveedores.service.spec.ts` y en `facturas-compra.service.spec.ts`). Esto es
distinto de una **verificación operativa end-to-end** con dos *tenants* reales:
la propia SPEC-001 declara ese escenario (CA-5) como **pendiente** por no
disponer aún de un segundo *tenant*/usuario para la prueba manual. Es decir: el
aislamiento está **implementado y cubierto por pruebas unitarias**, pero su
validación operativa completa sigue pendiente.

## 4. Integración continua (GitHub Actions)

El *pipeline* de CI (`.github/workflows/ci.yml`) se ejecuta en **cada Pull
Request hacia `main`** y en cada *push* a `main`, actuando como **gate de
calidad**. Define tres *jobs* independientes (Node.js 22):

1. **`backend (tests + typecheck)`** — en `backend/`:
   `npm ci` → `npx prisma generate` → `npx tsc --noEmit` → `npm test` (Jest).
2. **`frontend POS (tests + build)`** — en `frontend/`:
   `npm install` → `npm test` (Vitest) → `npm run build`.
3. **`sap-lite-web (tests + build)`** — en `sap-lite-web/` (el frontend de
   Control de Gestión): `npm install` → `npm test` (Vitest) → `npm run build`.

Validaciones automáticas antes de integrar un cambio: **tipado** (`tsc`/`vue-tsc`),
**pruebas unitarias** (Jest/Vitest) y **build** de los frontends. Un *workflow*
adicional (`automerge.yml`) solo fusiona un PR cuando tiene el *label*
`automerge` **y** el CI concluyó en estado `success` para su *commit*, lo que
evita que código roto llegue a `main` y mantiene la calidad del producto.

## 5. Ejecución de pruebas (comandos reales)

Comandos verificados en los `package.json` del proyecto:

**Backend** (`backend/package.json`):
```bash
npm test            # jest — ejecuta todas las pruebas unitarias
npm run test:watch  # jest --watch
npx tsc --noEmit    # verificación de tipos
npm run build       # nest build
npm run lint        # eslint "{src,test}/**/*.ts" --fix
```

**Frontend de Control de Gestión** (`sap-lite-web/package.json`):
```bash
npm test             # vitest run — ejecuta todas las pruebas
npm run test:watch   # vitest (modo watch)
npm run type-check   # vue-tsc --noEmit
npm run build        # vue-tsc --noEmit && vite build
npm run lint         # eslint src --ext .vue,.ts,.tsx --fix
```

## 6. Resultados

- **Inventario verificado durante esta revisión:** 78 archivos de prueba en el
  backend (≥34 directamente de SAP-lite) y 20 en `sap-lite-web`. Estos recuentos
  son comprobables listando los archivos `*.spec.ts` / `*.test.ts` del
  repositorio.
- **Resultados históricos documentados:** las pruebas se ejecutan en cada PR como
  *gate* obligatorio del CI; un cambio solo se integra a `main` con el CI en
  verde. La *Definition of Done* del proyecto exige pruebas presentes y en verde
  más *typecheck*/build.
- **Pruebas E2E:** **no existe** una *suite* E2E automatizada (no hay Playwright
  ni Cypress, ni configuración de E2E activa). Existe un *script* manual de
  verificación por HTTP (`backend/test/e2e-ocr-llm-keys.sh`), pero corresponde al
  flujo de OCR/claves LLM —**no** a SAP-lite— y no forma parte del CI.
- **No se reportan** porcentajes de cobertura ni cantidades de pruebas aprobadas,
  porque no se ejecutó la *suite* completa durante esta revisión documental (el
  objetivo fue inventariar y verificar la estrategia, no medir resultados). No se
  inventan cifras.

> **Distinción de resultados:** el inventario de archivos de prueba y los comandos
> fueron **verificados durante esta revisión**; la ejecución efectiva de las
> pruebas y su estado "verde" corresponden al **gate de CI documentado** del
> proyecto, no a una corrida realizada en esta revisión.
