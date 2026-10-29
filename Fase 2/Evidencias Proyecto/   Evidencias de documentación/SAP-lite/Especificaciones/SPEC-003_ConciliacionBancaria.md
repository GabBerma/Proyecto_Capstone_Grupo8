# SPEC-003 — Conciliación Bancaria (resumen académico)

> **Evidencia académica — Proyecto CAPSTONE.** Resumen de las especificaciones
> relacionadas con la conciliación y la importación de cartolas bancarias de
> SAP-lite. No reemplaza a las SPECs originales. Los ejemplos son ficticios.

## 0. Aclaración importante de alcance y nomenclatura

En el proyecto real conviven dos conceptos que **no son equivalentes** y que
conviene distinguir con precisión académica:

1. **Importación de cartolas bancarias Itaú (SPEC-002):** carga y normalización
   de los movimientos de la cartola del banco. **Está implementada.**
2. **Conciliación bancaria propiamente tal (SPEC-003):** cruce de las
   **liquidaciones Transbank** contra los **medios de pago registrados en las
   ventas de PittySoft-POS**. **Está en estado "Borrador" y NO implementada.**

Es relevante señalar que la SPEC-003 oficial del proyecto se titula
**"Conciliación Transbank contra medios de pago de PittySoft-POS"** y
**excluye explícitamente** la conciliación directa contra los abonos/movimientos
bancarios de la cartola Itaú (SPEC-002). Es decir: **no existe en el proyecto una
SPEC que concilie específicamente la cartola Itaú**; lo que existe sobre Itaú es
su importación (SPEC-002). Este documento cubre ambos aspectos para dar un
panorama completo y honesto.

> **La importación de cartolas no equivale a conciliación bancaria.** La
> conciliación (SPEC-003) sigue **pendiente de desarrollo**.

---

## Parte A — Importación de cartolas bancarias Itaú (SPEC-002, *Implementada*)

### A.1 Objetivo
Permitir importar cartolas bancarias desde archivos `.xls`/`.xlsx`, comenzando
por **Banco Itaú**, con previsualización, validación de filas y confirmación de
movimientos bancarios normalizados. Reduce el ingreso manual y prepara datos
confiables para una futura conciliación.

### A.2 Contexto del problema
SAP-lite expone una página **"Importaciones financieras"** con dos pestañas:
`Cartola bancaria` (SPEC-002) y `Liquidación Transbank` (SPEC-006). Los
movimientos bancarios y las liquidaciones Transbank son entidades **separadas**.
La primera versión soporta solo Itaú, a partir de una muestra anonimizada.

### A.3 Alcance
- Página `/importaciones-financieras`, pestaña `Cartola bancaria`.
- Selección de banco (Itaú) y de una cuenta del *tenant*.
- Adaptador Itaú que entrega movimientos normalizados; reconocimiento de
  cabeceras **por nombre**, tolerando filas informativas antes del encabezado.
- Previsualización con clasificación de filas en válidas, rechazadas y posibles
  duplicadas; confirmación solo de las válidas; historial de importaciones;
  vista de consulta de movimientos; descarga de filas rechazadas.
- **Fuera de alcance:** Banco de Chile (sin muestra validada), liquidaciones
  Transbank (SPEC-006), conciliación financiera (SPEC-003), aplicación de pagos,
  contabilización, OCR/IA.

### A.4 Criterios de aceptación principales
- **CA-2:** obliga a elegir banco y cuenta antes de procesar el archivo.
- **CA-3:** una muestra Itaú válida genera previsualización en la entidad de
  movimiento bancario.
- **CA-4 / CA-5:** rechaza archivos dañados, con macros, de formato prohibido o
  cuya estructura no corresponde al banco seleccionado.
- **CA-6:** la previsualización muestra totales, número de fila original, datos
  normalizados y motivo de rechazo.
- **CA-7:** la confirmación guarda solo filas válidas (confirmación adicional si
  hay rechazos); cancelar no guarda nada.
- **CA-9 / CA-10 / CA-11:** control de duplicados por **SHA-256** del archivo
  (por *tenant*/banco/cuenta) y por **huella** del movimiento; un `preview_id`
  reutilizado o expirado no genera registros adicionales.
- **CA-12:** aislamiento multi-tenant (acceso solo a cuentas/archivos del
  *tenant*).
- **CA-15:** el estado inicial del movimiento al guardarse es `PENDIENTE`.

### A.5 Reglas de negocio
- Montos en **`Decimal`** (nunca `float`), CLP, zona horaria `America/Santiago`.
- `preview_id` temporal, de un solo uso y con expiración; confirmación dentro de
  **transacción**.
- Huella canónica del movimiento (cuenta, fecha, monto, tipo, referencia) para la
  deduplicación; se almacena `raw_data` saneado y la fila de origen para
  auditoría. Números de cuenta **enmascarados** en la UI.

### A.6 Implementación técnica
Módulo backend `cartola-bancaria` (`cartola-import.controller.ts`,
`cartola-import.service.ts`, `itau-adapter.ts`, utilidades de enmascaramiento) y
modelos Prisma `CuentaBancaria`, `ImportacionBancaria`,
`ImportacionBancariaPreview` y `MovimientoBancario`. Frontend: *store*
`cartola-bancaria.ts` y vistas en `views/cartola-bancaria/`.

### A.7 Pruebas que lo respaldan
- Backend: `cartola-import.service.spec.ts`, `itau-adapter.spec.ts`.
- Frontend: `stores/__tests__/cartola-bancaria.test.ts`.

---

## Parte B — Conciliación bancaria (SPEC-003, *Borrador / Pendiente*)

### B.1 Objetivo
Conciliar las **liquidaciones Transbank** importadas en SAP-lite con los
**medios de pago registrados en las ventas de PittySoft-POS**, proponiendo
coincidencias y destacando diferencias, ambigüedades o registros sin
correspondencia. El POS sigue siendo la **fuente de verdad** de ventas y pagos;
SAP-lite no debe crear una fuente paralela de ventas.

### B.2 Contexto del problema
Transbank entrega liquidaciones (abonos) que deben cruzarse con los pagos de las
ventas del POS. Las fechas (venta, pago y liquidación) pueden diferir, y las
comisiones o retenciones generan diferencias de monto, por lo que el cruce
requiere reglas y tolerancias aprobadas por negocio.

### B.3 Alcance definido en la SPEC original
- **Incluye:** tarjeta `Conciliación Transbank` y página
  `/conciliacion-transbank`; consulta de medios de pago desde el POS; uso de las
  liquidaciones ya importadas (SPEC-006); búsqueda de pagos candidatos;
  comparación por monto, fecha, comercio/sucursal, medio de pago y referencias;
  coincidencias exactas y con advertencias; conciliación uno a uno; confirmación
  manual; rechazo con motivo; deshacer con auditoría; historial de decisiones;
  aislamiento por *tenant*.
- **NO incluye:** importación de liquidaciones (SPEC-006) ni de cartolas
  (SPEC-002); **conciliación directa contra los abonos bancarios de SPEC-002**;
  creación/modificación de ventas o pagos del POS; contabilización;
  aprobación automática sin regla aprobada; decisiones por IA.

### B.4 Criterios de aceptación principales (declarados, aún no verificados)
- **CA-3 / CA-4 / CA-5:** proponer coincidencia cuando hay pago compatible; dejar
  `SIN_CANDIDATO` cuando no lo hay; marcar `AMBIGUA` cuando hay varios candidatos.
- **CA-6:** las diferencias fuera de tolerancia no se presentan como coincidencia
  exacta y quedan visibles.
- **CA-7 / CA-8 / CA-9:** confirmar (registrando usuario, fecha y reglas),
  rechazar con motivo y deshacer con auditoría.
- **CA-10 / CA-14:** impedir sobreasignación; **no modificar** `Sales` ni
  `Payment` del POS.
- **CA-12:** rechazar cualquier cruce entre *tenants*.

### B.5 Reglas de negocio y estados previstos
Bandejas: Pendientes, Sugeridas, Ambiguas, Sin candidato, Con diferencias,
Confirmadas, Rechazadas. Estados: `PENDIENTE`, `SUGERIDA`, `AMBIGUA`,
`SIN_CANDIDATO`, `CON_DIFERENCIAS`, `CONFIRMADA`, `RECHAZADA`, `DESHECHA`. Las
tolerancias de monto/fecha deben ser **configurables por tenant** (no
hardcodeadas); auditoría inmutable; confirmaciones/reversiones en transacción.

### B.6 Relación prevista entre movimientos, pagos, facturas y liquidaciones
La SPEC-003 concilia **liquidaciones Transbank ↔ pagos del POS**. Explícitamente
**no** concilia contra los abonos bancarios de la cartola Itaú, ni contra
facturas de compra. Las cartolas bancarias (SPEC-002) podrán usarse en otros
procesos de Control de Gestión, pero no son dependencia directa de esta
conciliación.

### B.7 Estado real y funcionalidades pendientes
- **Estado:** `Borrador`. La propia SPEC indica que debe permanecer en borrador y
  **no implementarse** mientras no se validen con el equipo POS los campos y
  contratos de `Sales`/`Payment` y no se aprueben las reglas y tolerancias.
- **En el código:** **no existe** módulo de conciliación ni modelo de "match".
  Los movimientos bancarios y las liquidaciones solo tienen estado `PENDIENTE` y
  carecen de relaciones de conciliación. Los datos quedan *preparados*
  (monto normalizado, fecha, referencia, huella) para cuando se implemente.

### B.8 Pruebas existentes
No existen pruebas de conciliación, por ser una funcionalidad **no
implementada**. Las pruebas presentes corresponden a la **importación** de los
insumos (cartola Itaú y liquidaciones Transbank), no a su cruce.

---

## Conclusión (distinción de estados)

| Capacidad | Estado | Verificada por pruebas |
| --- | --- | --- |
| Importación de cartola Itaú (SPEC-002) | ✅ Implementada | ✅ Sí |
| Importación de liquidaciones Transbank (SPEC-006) | ✅ Implementada | ✅ Sí |
| **Conciliación bancaria (SPEC-003)** | ⏳ **Borrador / no implementada** | ❌ No |
