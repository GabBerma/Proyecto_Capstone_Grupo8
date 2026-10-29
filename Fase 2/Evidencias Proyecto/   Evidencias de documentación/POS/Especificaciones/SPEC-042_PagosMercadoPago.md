# SPEC-042 (resumen académico) — Gestión y recuperación de pagos Mercado Pago

> **Evidencia académica — Proyecto CAPSTONE, Ingeniería en Informática (Duoc UC).**
> Resumen de una especificación real del módulo POS de PittySoft. No reproduce la
> especificación completa ni constituye una especificación operativa. No contiene datos
> reales, secretos ni información confidencial.

## 1. Objetivo

Hacer más robusto el cobro con **Mercado Pago Point** (terminal de pago con tarjeta) para
evitar **pagos duplicados** o ventas con estados incorrectos cuando se pierde la comunicación
con el terminal, y permitir que el POS **recupere el estado** de un pago sin iniciar un cobro
nuevo innecesariamente.

## 2. Contexto del problema

El POS consultaba repetidamente el estado del pago (*polling*) **sin un límite de espera
definido**. Si ocurría una falla de conexión, la interfaz podía quedar mostrando
«Procesando pago…» de forma indefinida, aunque el terminal ya hubiera aprobado o rechazado la
operación. Esto creaba ambigüedad sobre el resultado real del cobro y riesgo de volver a
cobrar.

## 3. Alcance de la implementación

**Incluye:**

- Un tiempo máximo **configurable** de espera para consultar el pago.
- Diferenciar los estados: **aprobado**, **rechazado** y **pendiente de confirmación**.
- Permitir **consultar nuevamente** el estado del mismo pago.
- **Bloquear** nuevos intentos de cobro mientras exista una operación anterior sin resolver.
- Controles de **idempotencia** para evitar duplicar pagos y ventas.
- Mensajes claros ante una falla de comunicación.

**No incluye:** modificar el flujo normal de pagos aprobados, cambiar la integración
tributaria, alterar otros métodos de pago ni activar cobros reales (las pruebas se realizan en
modo simulado).

## 4. Criterios de aceptación principales

- **CA-1 — Tiempo de espera:** si el terminal no responde dentro del tiempo configurado, el
  POS finaliza la espera activa y muestra «Pago sin confirmar», **sin** considerarlo rechazado.
- **CA-2 — Pago aprobado:** cuando el pago se confirma aprobado, la venta se finaliza **una
  sola vez**.
- **CA-3 — Pago rechazado:** cuando se confirma el rechazo, el POS informa el resultado sin
  registrar una venta pagada.
- **CA-4 — Recuperación:** ante un pago sin confirmar, la acción «Consultar estado» verifica la
  **operación original** sin generar un nuevo cobro.
- **CA-5 — Prevención de duplicados:** mientras un pago esté pendiente de confirmación, se
  **bloquea** cualquier nuevo intento de cobro de la misma operación.
- **CA-6 — Error de conexión:** si no es posible consultar el estado, se muestra un mensaje
  claro y se **conserva la referencia** del pago para recuperarlo después.
- **CA-7 — Compatibilidad:** los pagos normales siguen funcionando sin afectar la integración
  tributaria ni otros métodos de pago.

## 5. Reglas de negocio

- Un **timeout no equivale a un rechazo**: el pago queda «sin confirmar», nunca marcado como
  rechazado automáticamente.
- La pérdida de conexión puede ocurrir **después** de que el pago haya sido aprobado; por ello
  siempre se conserva la referencia de la operación.
- Mientras una operación no se resuelva, **no** se permite iniciar otro cobro sobre ella.
- Si el estado no puede recuperarse, la operación permanece «sin confirmar» hasta su revisión.

## 6. Integración del POS con Mercado Pago y manejo de estados

El POS inicia una **intención de cobro** contra el terminal y consulta su estado de forma
periódica pero **acotada** por el tiempo máximo configurado. Los estados se interpretan así:

| Estado | Significado | Efecto en el POS |
|---|---|---|
| Aprobado | El pago se confirmó correctamente. | Se finaliza la venta (una sola vez). |
| Rechazado | El terminal/servicio rechazó el cobro. | Se informa; no se registra venta pagada. |
| Pendiente / sin confirmar | Se agotó el tiempo o se perdió la conexión. | Se conserva la referencia para recuperación. |

## 7. Control de operaciones pendientes, recuperación y prevención de duplicados

- **Operaciones pendientes:** cada cobro mantiene en memoria su intención y su referencia de
  operación; mientras exista una sin resolver, el botón de cobro queda bloqueado para esa
  operación.
- **Recuperación:** la acción «Consultar estado» **reutiliza la intención original** y vuelve a
  consultar su resultado, en lugar de iniciar un cobro nuevo. Así, un pago que se aprobó
  durante una caída de red puede confirmarse sin volver a cobrar.
- **Prevención de cobros duplicados:** se combinan tres mecanismos — bloqueo de la acción ante
  doble clic, bloqueo de un nuevo cobro mientras exista uno sin confirmar, y conservación de la
  referencia de la operación para no crear intenciones paralelas.

## 8. Breve explicación técnica

- **Frontend (Vue + Pinia):** los cambios se concentran en el store de cobro del POS
  (`posPayment.ts`) y en los componentes del Modo Venta. El *polling* se limita mediante un
  parámetro de tiempo máximo configurable (`pollTimeoutMs`); al superarlo, el estado pasa a
  «sin confirmar» conservando la intención y su fila para poder recuperarla con «Consultar
  estado». El HTTP pasa por el cliente central `lib/api.ts`.
- **Modo de pruebas:** durante el desarrollo y las pruebas se mantiene el **modo simulado** del
  medio de pago, sin activar cobros reales.
- **Multi-tenant:** se respetan el aislamiento y los permisos existentes.

## 9. Resultados obtenidos y pruebas que los respaldan

**Implementado y verificado por pruebas automáticas:**

- El store de cobro del POS cuenta con un conjunto amplio de pruebas unitarias de frontend
  (`posPayment.test.ts`, más pruebas complementarias de cobro de saldo), que cubren los
  escenarios de la especificación: aprobación, rechazo, timeout/«sin confirmar»,
  recuperación mediante «Consultar estado» y bloqueo de cobro duplicado.
- Las pruebas se ejecutan automáticamente en la integración continua (ver
  `../Pruebas/EstrategiaPruebas.md`).

**Alcance de la verificación:**

- La verificación se realizó mediante pruebas automáticas y en **modo simulado** del medio de
  pago, conforme al alcance de la especificación. La operación con un terminal físico real en
  producción queda fuera de este alcance y no se afirma como verificada.
