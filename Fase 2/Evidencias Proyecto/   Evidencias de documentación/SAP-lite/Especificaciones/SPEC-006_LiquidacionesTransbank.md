# SPEC-006 — Importación de Liquidaciones Transbank (resumen académico)

> **Evidencia académica — Proyecto CAPSTONE.** Resumen de la especificación
> SPEC-006 de SAP-lite y de su implementación real. No reemplaza a la SPEC
> original. Los ejemplos son ficticios.

## 1. Objetivo

Permitir al área administrativa importar **liquidaciones/abonos de Transbank**
desde archivos `.xls`/`.xlsx` —específicamente el reporte
"Resumen histórico de abonos Transbank"— con previsualización, validación de
filas y confirmación de liquidaciones normalizadas. Reduce el ingreso manual,
preserva la trazabilidad y prepara datos confiables para una futura conciliación
(SPEC-003).

> La SPEC-006 **importa**; **no concilia**, no aplica pagos y no contabiliza.

## 2. Contexto del problema

SAP-lite ofrece la página **"Importaciones financieras"** como contenedor común
con dos pestañas: `Cartola bancaria` (SPEC-002) y `Liquidación Transbank` (esta
SPEC). El contenedor (tarjeta, página y disposición de pestañas) lo define
SPEC-002; SPEC-006 **agrega la pestaña** `Liquidación Transbank` reutilizando ese
contenedor **sin mezclar** la lógica bancaria: adaptador, endpoints, *store* y
tablas son independientes. La única muestra confirmada es el
"Resumen histórico de abonos Transbank" (obtenido con el RUT de la empresa, que
contiene las liquidaciones de todos los locales/comercios de esa empresa).

## 3. Alcance de la implementación

- Pestaña `Liquidación Transbank` dentro de "Importaciones financieras".
- Importación de `.xls`/`.xlsx` del "Resumen histórico de abonos".
- Adaptador Transbank que entrega liquidaciones normalizadas; reconocimiento de
  cabeceras **por nombre**, tolerando filas informativas previas.
- **Derivación del comercio/sucursal** a partir de las filas del archivo (no se
  pide como selección aparte ni como fuente independiente).
- Previsualización, confirmación solo de filas válidas, historial de
  importaciones, vista de consulta de liquidaciones y descarga de filas
  rechazadas.
- **Fuera de alcance:** cartolas bancarias (SPEC-002), un segundo formato/fuente
  por comercio (sin muestra que lo confirme), conciliación (SPEC-003), aplicación
  de pagos, contabilización, OCR/IA.

## 4. Criterios de aceptación principales

- **CA-1 / CA-2:** usuario autorizado abre la pestaña; una muestra válida genera
  previsualización en la entidad de liquidación, derivando comercio/sucursal por
  fila cuando corresponde.
- **CA-3 / CA-4:** rechazar archivos dañados, con macros, de formato prohibido o
  cuya estructura no sea la del reporte Transbank (sin reinterpretarlo con el
  adaptador bancario de SPEC-002).
- **CA-5 / CA-6:** la previsualización muestra totales, número de fila original,
  datos normalizados y motivo de rechazo; la confirmación guarda solo filas
  válidas (confirmación adicional si hay rechazos); cancelar no guarda nada.
- **CA-8 / CA-9 / CA-10:** control de duplicados por **SHA-256** del archivo (por
  *tenant*) y por **huella** de la liquidación; las coincidencias por
  identificador inestable se marcan como posible duplicado; un `preview_id`
  reutilizado o expirado no genera registros adicionales.
- **CA-11 / CA-13:** aislamiento multi-tenant; montos enmascarados y en `Decimal`.
- **CA-14:** el estado inicial de la liquidación al guardarse es `PENDIENTE`.
- **CA-15:** resumen tras la importación (procesadas/aceptadas/rechazadas/
  duplicadas) y descarga de filas rechazadas.

## 5. Reglas de negocio

- Reconocer cabeceras **por nombre**, no por posición.
- `preview_id` temporal, de un solo uso, con expiración; confirmación dentro de
  **transacción**.
- **Huella** canónica de la liquidación (fecha, monto, comercio/sucursal,
  referencia); las coincidencias por identificador inestable no se importan
  automáticamente.
- Montos en **`Decimal`** (nunca `float`), CLP, zona horaria `America/Santiago`;
  se almacena `raw_data` saneado y la fila de origen; tamaño máximo de archivo de
  5 MB.

## 6. Importación de archivos Excel

Se aceptan `.xls` y `.xlsx`. La validación identifica el formato real del
archivo y verifica que corresponda al reporte Transbank; de lo contrario, la
previsualización no se habilita.

## 7. Reconocimiento y validación de columnas

El adaptador reconoce las cabeceras por su nombre, tolerando filas informativas
antes del encabezado. El modelo `LiquidacionTransbank` captura, entre otros:
`fechaAbono`, `comercioSucursal`, `totalVentas`, `comision`, `ventasAnuladas`,
`cobrosPorServicio`, `totalMontosDescontados`, `devolucionComision`,
`totalAbono` y `numeroVentas`.

## 8. Procesamiento de fechas y montos

- Fechas normalizadas a tipo fecha (`fechaAbono`), interpretadas en zona horaria
  `America/Santiago`.
- Montos normalizados a `Decimal(14,2)` en CLP; se respetan los valores del
  archivo (sin fórmulas inventadas).

## 9. Identificación de registros duplicados

Dos niveles: a nivel de **archivo** por SHA-256 (`fileHash`, único por *tenant*)
y a nivel de **registro** por `huella`. Una liquidación con huella duplicada o
con coincidencia por identificador inestable se marca como **posible duplicado** y
**no** se importa automáticamente.

## 10. Previsualización de resultados

Antes de guardar, se muestra una previsualización con los totales y la
clasificación de cada fila (válida, rechazada o posible duplicada), con el número
de fila original y el motivo de rechazo.

## 11. Manejo de filas rechazadas

Las filas rechazadas se informan con su motivo y pueden **descargarse** para
corrección. No se importan.

## 12. Confirmación de importaciones

La confirmación guarda **solo las filas válidas**, dentro de una **transacción**,
a través de un `preview_id` de un solo uso. Si existen rechazos, se exige una
confirmación adicional. Cancelar no guarda nada.

## 13. Registro y consulta del historial de liquidaciones

Cada importación se registra en `ImportacionTransbank` (nombre de archivo, hash,
totales de filas aceptadas/rechazadas/duplicadas) y las liquidaciones resultantes
se consultan en una vista dedicada, separada de los movimientos bancarios.

## 14. Relación prevista con la conciliación bancaria

Las liquidaciones quedan **preparadas** (monto, fecha, comercio/sucursal,
referencia, huella) para la conciliación de SPEC-003 (liquidaciones Transbank ↔
medios de pago del POS). Esa conciliación está en estado **Borrador** y **no**
forma parte de SPEC-006.

## 15. Explicación técnica

- **Backend (NestJS):** módulo `liquidaciones-transbank`
  (`transbank-import.controller.ts`, `transbank-import.service.ts`,
  `transbank-adapter.ts`, utilidad de enmascaramiento). Modelos Prisma
  `ImportacionTransbank`, `ImportacionTransbankPreview` y `LiquidacionTransbank`.
- **Frontend (Vue 3):** *store* `liquidaciones-transbank.ts` y vistas en
  `views/liquidaciones-transbank/`.

## 16. Resultados obtenidos y pruebas que los respaldan

- Backend: `transbank-adapter.spec.ts` (parsing del reporte, normalización de
  fechas e importes, derivación de comercio, conteo de ventas) y
  `transbank-import.service.spec.ts` (flujo de importación, duplicados, filas
  rechazadas).
- Frontend: `stores/__tests__/liquidaciones-transbank.test.ts`.

> **Distinción de estado:** la importación de liquidaciones Transbank está
> **implementada y verificada por pruebas unitarias**. Su posterior
> **conciliación** (SPEC-003) permanece **pendiente**.
