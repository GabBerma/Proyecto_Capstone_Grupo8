# Estrategia de pruebas del módulo POS — PittySoft

> **Evidencia académica — Proyecto CAPSTONE, Ingeniería en Informática (Duoc UC).**
> Documento elaborado a partir de la configuración y los archivos de prueba reales del
> proyecto. Las cantidades indicadas corresponden a lo observado en el repositorio al momento
> de elaborar esta evidencia; no se inventan métricas ni porcentajes de cobertura.

## 1. Herramientas utilizadas

| Ámbito | Herramienta | Verificación |
|---|---|---|
| **Backend** | **Jest** (con `ts-jest`) | `backend/package.json` (`"test": "jest"`), `backend/jest.config.js` |
| **Frontend (POS)** | **Vitest** | `frontend/package.json` (`"test": "vitest run"`) |
| **Verificación de tipos** | **TypeScript** (`tsc --noEmit`) backend; **vue-tsc** en el build del frontend | scripts de `package.json` y workflow de CI |
| **Pruebas E2E (andamiaje)** | **Playwright** | carpeta `e2e/` (`playwright.config.ts`) |

## 2. Organización de las pruebas

Las pruebas conviven con el código que verifican, siguiendo la convención de cada stack.

**Backend (Jest).** Archivos `*.spec.ts` ubicados junto a cada service o utilidad dentro de
`backend/src/modules/<módulo>/`. En el repositorio se observan **75 archivos de prueba
unitaria** en el backend (toda la aplicación monolítica). De ellos, aproximadamente **28
corresponden a módulos del dominio POS**, entre otros:

- `sales/` — ventas, devoluciones y notas de crédito (varios archivos: `sales.service`,
  `sales-ventas.service`, `sales-devolucion.service`, `sales-notas-credito.service`,
  `devolucion.util`, DTO).
- `payments/` y `mercado-pago/` — pagos y conciliación con Mercado Pago.
- `vendedoras/`, `appointments/`, `cash/`, `clientes/`, `patients/`, `historias/`,
  `optometristas/`, `ordenes-trabajo/`, `locations/`, `metas/`, `reporting/`, `auth/`,
  `billing/` (Bsale), más utilidades compartidas (`rut.util`, `day-range.util`).

El resto de los archivos de prueba del backend corresponde a otros dominios del monolito
(módulo contable/SAP-lite y agente), que quedan fuera del alcance de esta evidencia.

**Frontend POS (Vitest).** Archivos `*.test.ts` en carpetas `__tests__/` junto a stores,
componentes y vistas dentro de `frontend/src/`. En el repositorio se observan **36 archivos
de prueba**, en su mayoría directamente relacionados con flujos del POS: carrito (`posCart`),
cobro (`posPayment`, `posPaymentSaldo`), devoluciones (`devoluciones`, `devolucionCalc`),
notas de crédito (`notasCredito`), ventas (`ventas`, `ventasPorCobrar`), clientes (`clientes`),
citas (`appointments`, `citasFecha`), vendedoras (`posVendedoras`), autenticación (`auth`),
caja (`cash`), más componentes y vistas del POS (modales de pago y confirmación, detalle de
venta, detalle de cliente, listados).

**Tipos de prueba presentes:**

- **Unitarias** — backend (Jest) y frontend (Vitest). Son la base de la estrategia.
- **Integración** — en el backend existen pruebas de integración (`*.int-spec.ts`) bajo
  `backend/test/integration/` (por ejemplo, concurrencia de citas y un escenario del flujo de
  ventas). Se ejecutan con un comando dedicado y requieren una base de datos de prueba.
- **E2E (extremo a extremo)** — existe **andamiaje** con Playwright en la carpeta `e2e/`
  (configuración, scripts de base de datos efímera y una prueba de *smoke*). **No** constituye
  todavía una suite E2E completa (ver sección 6).

## 3. Funcionalidades verificadas por pruebas

Con base en los archivos de prueba presentes, el módulo POS cuenta con verificación automática
de, entre otras:

- **Registro y validación de ventas** — cierre de venta, cálculo de totales y validaciones del
  carrito (`sales.service`, `posCart`, `posPayment`).
- **Procesamiento y estados de pagos** — aprobación, rechazo, timeout/«sin confirmar» y
  recuperación del cobro con Mercado Pago (`posPayment`), y conciliación de pagos
  (`mercado-pago`).
- **Devoluciones y notas de crédito** — cálculo de la devolución, emisión idempotente y
  anti-duplicados (`sales-devolucion.service`, `sales-notas-credito.service`,
  `devolucion.util`, store `devoluciones`).
- **Gestión de clientes** — búsqueda y normalización, y detalle de cliente (`clientes.service`,
  `cliente-search.util`, store `clientes`).
- **Asociación de vendedoras a la venta** — resolución de la vendedora, maestro de vendedoras y
  token de sesión (`vendedoras.service`, `auth.service`, store `posVendedoras`).
- **Control de permisos y sucursales** — guards de permisos por rol y validaciones de
  sucursal presentes en los services del backend.
- **Otras reglas de negocio** — citas y disponibilidad horaria (incluida concurrencia),
  utilidades de RUT y de rangos de fecha, y métricas/indicadores del dashboard.

## 4. Integración continua (GitHub Actions)

El repositorio define un workflow de **CI** (`.github/workflows/ci.yml`) que se ejecuta en
cada *pull request* hacia la rama principal y en los *push* a dicha rama. Actúa como **gate de
calidad**: sus *jobs* son los *status checks* exigidos antes de integrar cambios.

El workflow define tres *jobs* (todos sobre Node.js 22):

1. **backend (tests + typecheck):** instala dependencias, genera el cliente Prisma, ejecuta la
   verificación de tipos (`tsc --noEmit`) y corre las **pruebas unitarias** (`npm test`).
2. **frontend POS (tests + build):** instala dependencias, ejecuta las **pruebas unitarias**
   (Vitest) y realiza el **build** (incluye verificación de tipos con `vue-tsc`).
3. **sap-lite-web (tests + build):** equivalente para el frontend del módulo contable (fuera
   del alcance POS).

**Validaciones automáticas antes de integrar:** verificación de tipos, pruebas unitarias
verdes y build exitoso. Solo cuando estos *checks* pasan puede integrarse el cambio, y el
despliegue se dispara únicamente si el CI del commit correspondiente está en verde. De este
modo, la CI impide que código que no compila o que rompe pruebas llegue a la rama principal,
sosteniendo la calidad del software de forma continua.

## 5. Ejecución de pruebas (comandos reales)

Verificados en los `package.json` de cada carpeta:

**Backend** (`backend/`):

```bash
npm test                 # pruebas unitarias (Jest)
npm run test:watch       # Jest en modo watch
npm run test:integration # pruebas de integración (requiere base de datos de prueba)
npm run test:e2e         # pruebas e2e del backend (configuración dedicada)
npx tsc --noEmit         # verificación de tipos
```

**Frontend POS** (`frontend/`):

```bash
npm test                 # pruebas unitarias (Vitest, modo run)
npm run test:watch       # Vitest en modo watch
npm run build            # build con verificación de tipos (vue-tsc + Vite)
```

**E2E con Playwright** (`e2e/`):

```bash
npm test                 # orquestación E2E (script run-e2e.sh)
npm run test:pw          # ejecutar Playwright directamente
```

## 6. Resultados (solo comprobable)

- La estrategia de pruebas del POS se apoya principalmente en **pruebas unitarias** de backend
  (Jest) y frontend (Vitest), que **se ejecutan automáticamente en la integración continua** y
  actúan como condición para integrar cambios.
- Existen **pruebas de integración** en el backend; se ejecutan mediante un comando dedicado y
  **no forman parte del workflow de CI** observado (requieren una base de datos de prueba).
- El soporte **E2E (Playwright)** está presente como **andamiaje** (configuración + prueba de
  *smoke*); el workflow de CI de E2E existe únicamente como **propuesta**
  (`e2e/ci/e2e-workflow.proposal.yml`) y **no está activo** en `.github/workflows/`. Por tanto,
  **no** se afirma que exista una suite E2E completa.
- **No se reportan** cantidades de pruebas aprobadas ni porcentajes de cobertura, por no ser
  métricas verificadas en este documento. Las cifras de archivos de prueba (75 en backend, 36
  en frontend) corresponden a un conteo directo sobre el repositorio y pueden variar con el
  tiempo.
