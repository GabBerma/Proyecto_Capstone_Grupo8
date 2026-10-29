# SPEC-053 — Importación Masiva de Vouchers (resumen)

| Campo | Valor |
|---|---|
| **ID** | SPEC-053 |
| **Módulo** | Importar Voucher Masivo |
| **Prioridad** | Media |

## Objetivo

Proveer la utilidad **"Importar Voucher Masivo"**: una pantalla que permite subir un
archivo Excel con múltiples comprobantes contables para crearlos de forma masiva en un
período, evitando la carga manual uno por uno. Replica la funcionalidad equivalente del
sistema contable anterior de la empresa y acelera la **carga inicial / migración** de
movimientos.

Complementa la creación manual del Registro de Voucher ([SPEC-017](./SPEC-017_RegistroVoucher.md))
y se apoya en el mismo plan de cuentas.

## Alcance

**Incluye:**
- Pantalla con selector de período (mes/año).
- Formulario con carga de archivo Excel (obligatorio) y un indicador de marca de pago.
- Procesamiento del Excel y creación masiva de los vouchers en el período.
- Retroalimentación del resultado (vouchers creados / errores por fila).

**No incluye:** automatizaciones asistidas y la creación/edición manual de vouchers
individuales (cubierta en SPEC-017).

## Criterios de aceptación

- **CA-1 — Importación happy path:** dado un archivo Excel válido con varios vouchers y
  un período seleccionado, al ejecutar la importación el sistema crea los vouchers en
  el período para ese `tenantId` y muestra el total creado.

- **CA-2 — Archivo obligatorio:** si el usuario intenta importar sin adjuntar archivo,
  el sistema bloquea la acción y señala el campo requerido faltante.

- **CA-3 — Marca de pago:** si el usuario activa la marca de pago, los vouchers
  importados se crean con la condición de pago correspondiente.

- **CA-4 — Errores por fila:** ante un Excel con filas inválidas (por ejemplo, cuenta
  inexistente o comprobante descuadrado), el sistema reporta las filas con error con un
  detalle claro, según la política de carga definida (parcial vs. todo o nada).

- **CA-5 — Aislamiento multi-tenant:** los vouchers importados quedan asociados
  exclusivamente al `tenantId` del usuario que ejecuta la importación.

## Reglas de negocio

- Cada comprobante importado debe cumplir las mismas reglas del registro manual, en
  particular la **cuadratura de partida doble** (Debe total = Haber total) y la
  existencia de las cuentas imputadas en el plan de cuentas de la empresa.
- Las líneas del Excel se agrupan en comprobantes según el período indicado.
- La **marca de pago** permite crear los comprobantes ya señalados como pagados; la
  carga manual (SPEC-017) los deja sin esa marca por defecto.
- Toda la importación se persiste acotada al `tenantId` de la consultora.

## Ejemplo (datos ficticios)

Archivo Excel con dos comprobantes para el período 05/2026:

| Comprobante | Cuenta | Debe | Haber |
|---|---|---:|---:|
| 1 | Caja | 50.000 | 0 |
| 1 | Ventas | 0 | 50.000 |
| 2 | Gastos de oficina | 20.000 | 0 |
| 2 | Banco | 0 | 20.000 |

Resultado esperado: **2 vouchers creados**, ambos cuadrados. Si la fila de una cuenta
no existiera en el plan de cuentas, esa fila se reporta como error sin invalidar los
comprobantes correctos (según la política de carga definida).
