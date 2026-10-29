# SPEC-036 (resumen académico) — Devoluciones y notas de crédito

> **Evidencia académica — Proyecto CAPSTONE, Ingeniería en Informática (Duoc UC).**
> Resumen de una especificación real del módulo POS de PittySoft. No reproduce la
> especificación completa ni constituye una especificación operativa. No contiene datos
> reales, secretos ni información confidencial.

## 1. Objetivo

Al registrar una devolución de productos en el POS, emitir una **Nota de Crédito Electrónica
real** en el servicio de documentos tributarios (Bsale, código SII 61), persistir sus datos y
mostrar un modal de confirmación con la opción de **ver/imprimir** el documento.

## 2. Contexto del problema

Una iteración previa ya permitía registrar devoluciones internamente, generando una nota de
crédito **solo a nivel de base de datos** (sin documento tributario electrónico). Faltaba
completar la integración para que la nota de crédito tuviera validez tributaria, reutilizando
el mismo patrón de emisión y confirmación visual ya existente para las boletas.

## 3. Alcance de la implementación

- Emisión de la nota de crédito electrónica en Bsale, referenciando la boleta original y
  únicamente los productos y cantidades devueltos.
- Obtención dinámica del tipo de documento correspondiente (sin valores fijos codificados).
- Persistencia del folio electrónico, el identificador del documento, la URL del PDF y el
  estado de emisión, reutilizando el modelo de documento existente.
- Manejo de errores que preserva la devolución interna aunque falle la emisión.
- Controles de idempotencia para evitar notas de crédito duplicadas.
- Modal de confirmación con el resultado de la emisión.

## 4. Criterios de aceptación principales

1. Una devolución válida emite una nota de crédito real en el entorno de pruebas (sandbox).
2. La nota de crédito referencia correctamente la boleta y los productos devueltos.
3. El folio, el identificador y el PDF quedan asociados al documento de nota de crédito.
4. El modal muestra el resultado y permite visualizar o imprimir el PDF.
5. Un error de emisión **no elimina** la devolución ya registrada.
6. No se generan notas de crédito duplicadas por doble clic ni por reintentos.
7. Los montos coinciden con la devolución original.
8. La venta original y el aislamiento multi-tenant permanecen intactos.

## 5. Reglas de negocio

- La nota de crédito debe cuadrar con el **total efectivamente devuelto**, calculado sobre los
  **precios de la venta original** (no sobre el catálogo actual), considerando cantidades
  devueltas, descuentos, IVA y redondeos.
- Una devolución genera **como máximo una** nota de crédito electrónica.
- No se emite una nueva nota de crédito si ya existe una confirmada para esa devolución.
- No se intenta emitir una nota de crédito si la venta no tiene un documento tributario de
  referencia.
- Debe distinguirse entre emisión **confirmada**, **fallida** y de **resultado incierto**
  (p. ej. ante un timeout de red); ante resultado incierto, se verifica el documento antes de
  permitir un reintento.

## 6. Gestión de devoluciones totales y parciales

El flujo admite devolver la **totalidad** de una venta o solo **parte** de ella: la operación
incluye únicamente los productos y cantidades seleccionados. El monto de la nota de crédito se
calcula a partir de esas líneas devueltas, respetando los precios y descuentos originales, de
modo que una devolución parcial produce una nota de crédito por el subconjunto correspondiente.

## 7. Relación con notas de crédito y Bsale

En el modelo de datos, la devolución se representa como un documento (`Invoice`) con estado
`credit_note` que referencia la venta original; no existe una tabla separada de devoluciones.
La emisión electrónica se delega al servicio de facturación (Bsale), reutilizando el mismo
mecanismo que emite las boletas. Al confirmarse la emisión, el documento interno de nota de
crédito queda enriquecido con el folio, el identificador del documento tributario y la URL del
PDF.

## 8. Breve explicación técnica

- **Backend (NestJS + Prisma):** la lógica reside en el service del módulo de ventas. La
  emisión es **idempotente** y **fail-soft**: si el servicio externo falla, la devolución
  interna se conserva y se refleja que la nota de crédito no fue emitida.
- **Anti-duplicados:** la emisión se protege con un **bloqueo a nivel de transacción**
  (*advisory lock* por correlativo de tenant) y un bloqueo de la fila involucrada
  (`SELECT ... FOR UPDATE`), de modo que un doble clic o dos solicitudes simultáneas no
  produzcan dos documentos.
- **Frontend (Vue + Pinia):** el cálculo del detalle de la devolución y el modal de
  confirmación se manejan en el store y la vista del módulo de devoluciones; el HTTP pasa por
  el cliente central `lib/api.ts`.
- **Multi-tenant:** todas las operaciones se mantienen acotadas por `tenant_id`.

## 9. Resultados obtenidos y pruebas que los respaldan

**Implementado y verificado por pruebas automáticas:**

- El módulo cuenta con pruebas unitarias de backend sobre el cálculo de la devolución y la
  emisión de la nota de crédito (por ejemplo, los archivos `devolucion.util.spec.ts`,
  `sales-devolucion.service.spec.ts` y `sales-notas-credito.service.spec.ts` del módulo de
  ventas), que cubren el cálculo de montos, el comportamiento idempotente/anti-duplicados y el
  manejo de errores.
- El frontend incluye pruebas del store de devoluciones y del cálculo asociado
  (`devoluciones.test.ts`, `devolucionCalc.test.ts`) y de la vista del módulo.
- Las pruebas se ejecutan de forma automática en la integración continua (ver
  `../Pruebas/EstrategiaPruebas.md`).

**Pendiente de verificación en el entorno operativo:**

- La **emisión electrónica real extremo a extremo** contra el entorno de pruebas del servicio
  tributario **no pudo validarse completamente** por una limitación externa del entorno de
  pruebas (disponibilidad de folios en el sandbox para el tipo de documento de nota de
  crédito). Esta distinción se documenta explícitamente: la funcionalidad está **implementada
  y cubierta por pruebas**, pero su validación end-to-end en el entorno externo quedó
  condicionada a esa dependencia.
