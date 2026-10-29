# SPEC-017 — Registro de Voucher (resumen)

| Campo | Valor |
|---|---|
| **ID** | SPEC-017 |
| **Módulo** | Registro de Voucher |
| **Prioridad** | Alta |

## Objetivo

Proveer el módulo de **Registro de Voucher**, el corazón del sistema contable: el
asiento donde se registra cada movimiento bajo **partida doble** (Debe/Haber),
asociándolo a una cuenta del plan de cuentas, un tipo de movimiento, un centro de
costo, un cliente/proveedor y un tipo de documento del SII.

Esta fase busca **paridad funcional para la operación manual** respecto del sistema
contable anterior de la empresa: una interfaz de grilla + formulario para registrar
y consultar comprobantes por período (mes/año), respetando el aislamiento multi-tenant. Las
automatizaciones asistidas se abordan en fases posteriores.

El Registro de Voucher es el módulo más importante del sistema porque genera el
asiento contable central que alimenta aguas abajo a los libros (Diario, Mayor) y a
los reportes tributarios.

## Alcance

**Incluye:**
- Grilla/listado de líneas de voucher con columnas contables (código, cuenta, Debe,
  Haber, centro de costo, cliente/proveedor, tipo y número de documento, comentario,
  fecha, comprobante, tipo de movimiento, monto, razón social).
- Formulario de ingreso con todos los campos del asiento.
- Selector de período (mes/año) y buscador, con filtro por rango de fechas.
- Comentario y glosa a nivel de comprobante completo.
- Registro, consulta y eliminación de comprobantes con aislamiento por `tenantId`.
  Un comprobante registrado no se edita (solo su comentario), para resguardar la
  trazabilidad contable.

**No incluye:** automatizaciones asistidas, generación de libros y reportes derivados
(specs propias) y la lógica de plantillas de voucher (spec aparte).

## Criterios de aceptación

- **CA-1 — Cuadratura de partida doble (Debe = Haber):** al guardar un comprobante, el
  sistema valida la cuadratura **a nivel de comprobante completo**: la suma total del
  Debe debe igualar la suma total del Haber. Si no cuadra, bloquea el guardado y
  muestra la diferencia exacta. No se permite guardar un comprobante descuadrado.

- **CA-2 — Selección de cuenta del plan de cuentas:** al escribir el código de cuenta,
  el sistema resuelve y muestra el nombre de la cuenta desde el plan de cuentas del
  tenant, y rechaza el guardado si la cuenta no existe o no es imputable.

- **CA-3 — Asociación a centro de costo:** el usuario puede seleccionar un centro de
  costo del catálogo de la empresa o dejar la línea sin asignar; la selección se
  persiste junto a la línea.

- **CA-4 — RUT de cliente/proveedor:** al seleccionar la entidad (cliente o proveedor)
  e ingresar el RUT, el sistema resuelve y muestra la razón social desde el catálogo
  correspondiente del tenant y valida el formato del RUT.

- **CA-5 — Tipo de documento SII:** el usuario puede seleccionar un tipo de documento
  del catálogo del SII y asociar su número de documento (folio).

- **CA-6 — Comentario por comprobante:** el comentario ingresado queda persistido y
  asociado al comprobante completo, visible en la grilla.

- **CA-7 — Filtro por período (mes/año):** la grilla muestra únicamente los vouchers
  cuya fecha cae dentro del mes/año seleccionado; un rango de fechas permite acotar
  aún más el listado.

- **CA-8 — Aislamiento multi-tenant:** un usuario de un tenant solo ve y afecta los
  vouchers asociados a su `tenantId`, sin posibilidad de ver ni modificar los de otro.

- **CA-9 — Campos obligatorios:** al guardar una línea, el sistema exige **Cuenta,
  Monto y Glosa**; el resto de los campos exigidos depende de la cuenta seleccionada.

- **CA-10 — Tipo de movimiento:** el comprobante registra su tipo de movimiento —
  **Ingreso, Egreso o Traspaso** — que se refleja en la grilla.

## Reglas de negocio

- **Cuadratura:** se exige a nivel de comprobante completo al guardar (Debe total =
  Haber total); no se admite guardar descuadrado.
- **Numeración de comprobantes:** automática y **correlativa mensual, por tipo de
  comprobante** (el sistema asigna el número; no es manual).
- **Tipos de movimiento:** Ingreso, Egreso, Traspaso.
- **Campos obligatorios:** Cuenta, Monto y Glosa; el resto es condicional según la
  cuenta seleccionada (por ejemplo, una cuenta de cliente/proveedor puede exigir RUT y
  tipo de documento).
- **Monto vs. Debe/Haber:** son columnas separadas visualmente en la grilla.
- **Integridad contable:** un maestro (cliente, proveedor o tipo de documento) usado
  en un asiento no puede borrarse mientras siga referenciado; primero hay que
  desvincular o anular el comprobante.

## Ejemplo (datos ficticios)

Comprobante de tipo **Egreso**, período 05/2026, por el pago de un servicio:

| Cuenta | Centro de costo | Debe | Haber | Documento |
|---|---|---:|---:|---|
| Gastos de servicios | ADMINISTRACIÓN | 100.000 | 0 | 33 — Factura Electrónica, folio 1001 |
| Banco | — | 0 | 100.000 | — |

Totales: Debe = 100.000, Haber = 100.000 → **cuadra**, se permite guardar.
Contraparte: proveedor ficticio "Servicios Generales SpA", RUT 11.111.111-1.
