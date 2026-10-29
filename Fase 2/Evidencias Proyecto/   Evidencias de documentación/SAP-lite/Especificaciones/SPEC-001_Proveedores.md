# SPEC-001 — Gestión de Proveedores (resumen académico)

> **Evidencia académica — Proyecto CAPSTONE.** Resumen de las especificaciones
> de proveedores de SAP-lite (SPEC-001, con sus extensiones SPEC-005 y SPEC-013)
> y de su implementación real. No reemplaza a las SPECs originales; es un
> compendio para portafolio. Los ejemplos son ficticios.

## 1. Objetivo

Construir el **maestro de proveedores** de SAP-lite: permitir crear, consultar,
buscar, modificar, desactivar y reactivar proveedores, de modo que el resto del
módulo (facturas de compra, servicios, arriendos, pagos) pueda relacionarse con
ellos manteniendo trazabilidad.

## 2. Contexto del problema

La información de proveedores estaba dispersa en planillas y sistemas
administrativos. Se requería un maestro centralizado, consistente y aislado por
*tenant*, que sirviera de base para la gestión de compras y pagos.

## 3. Alcance de la implementación

La funcionalidad se compone de tres especificaciones complementarias:

- **SPEC-001 — Gestión de proveedores** (*Implementada*): CRUD, búsqueda, control
  de duplicados por RUT, baja/reactivación lógica y auditoría.
- **SPEC-005 — Importación masiva desde Excel** (*Implementada*): carga de
  proveedores desde `.xlsx`, `.xls` y `.csv` con validación y previsualización.
- **SPEC-013 — Ampliación de ficha y trazabilidad** (*Borrador*): campos
  adicionales de clasificación, dirección, datos de pago y contactos.

Quedan **fuera de alcance** de SPEC-001: la eliminación física de registros, la
administración de facturas/pagos y la conciliación bancaria.

## 4. Criterios de aceptación principales

- **CA-1 / CA-2:** crear proveedor con los campos obligatorios (razón social),
  asociado al `tenant_id`; rechazar formularios incompletos con mensajes claros.
- **CA-3 / CA-4:** normalizar y validar el RUT cuando se informa; impedir RUT
  duplicado dentro del mismo *tenant* (activo o inactivo).
- **CA-6 / CA-7:** listado paginado y búsqueda por RUT, razón social o nombre de
  fantasía.
- **CA-8 / CA-9 / CA-11:** modificar registrando usuario y fecha; desactivar con
  baja lógica (`activo = false`) sin borrado físico; reactivar conservando las
  relaciones históricas.
- **CA-12:** registrar auditoría (`created_by/at`, `updated_by/at`,
  `deactivated_by/at`).
- **CA-5 (multi-tenant):** declarado como **pendiente de validación** en la SPEC
  original por no disponer de un segundo *tenant*/usuario para la prueba manual.

## 5. Reglas de negocio

- El **RUT es opcional**; si se informa, se normaliza y valida. La unicidad
  aplica a la combinación `tenant_id + rut` (varios proveedores sin RUT son
  válidos, porque Postgres trata los `NULL` como distintos).
- La **desactivación es lógica**; nunca hay borrado físico.
- Toda consulta/acción se **scopea por `tenant_id`**.

## 6. Registro y edición de proveedores

El modelo real `Proveedor` almacena identificación (RUT, razón social, nombre de
fantasía, giro), contacto (email, teléfono, dirección), observaciones y campos de
auditoría. Las extensiones de SPEC-013 añaden rubro, datos administrativos de
pago (banco, tipo y número de cuenta, titular) y hasta dos contactos (comercial y
de pagos) con su correo. **Nunca** se almacenan claves, PIN ni tokens.

## 7. Búsqueda y filtrado

El listado es paginado y admite búsqueda por RUT, razón social o nombre de
fantasía, con filtro para incluir u ocultar proveedores inactivos.

## 8. Validación de datos y control de duplicados

- Validación de entrada mediante **DTOs** en el backend.
- Normalización y validación de RUT (utilidad compartida `rut.util`).
- Restricción de unicidad `@@unique([tenantId, rut])` a nivel de base de datos,
  reforzada por la lógica del *service*.

## 9. Importación masiva desde Excel/CSV (SPEC-005)

- Botón "Importar proveedores" integrado en la pantalla existente.
- Soporta `.xlsx`, `.xls` y `.csv`, con **identificación real del formato** (no
  solo por extensión) y reconocimiento del separador de CSV (coma o punto y coma)
  y de codificaciones (UTF-8 / Windows-1252).
- Reconocimiento de cabeceras **por nombre** (independiente de la posición); solo
  "Razón Social" es obligatoria.
- **Previsualización** que clasifica las filas en válidas, duplicadas y
  rechazadas (con número de fila y motivo) antes de guardar.
- La confirmación importa **solo las filas válidas**, dentro de una transacción,
  usando el `tenant_id` de la sesión y la misma auditoría que el alta manual.

## 10. Asociación con el Plan de Cuentas

Implementada de forma **opcional e informativa**. El modelo `Proveedor` conserva
los campos de texto legados (`numeroCuentaContable`, `nombreCuenta`) y, además,
una FK opcional `cuentaContableId` al catálogo global **Plan de Cuentas**
(`CuentaContable`, SPEC-016). Esta referencia no genera asientos contables ni se
usa para calcular facturas o pagos; cuando existe, es la fuente del número y
nombre de cuenta.

## 11. Relaciones con facturas, servicios y arriendos

Desde `Proveedor` cuelgan:

- `facturasCompra` → `FacturaCompra` (una factura de compra puede referenciar a
  un proveedor; la relación es opcional del lado de la factura).
- `servicios` → `Servicio` (un servicio es prestado por un proveedor).
- `arriendos` → `Arriendo` (el proveedor actúa como arrendador).

## 12. Explicación técnica de la implementación

- **Backend (NestJS):** módulo `proveedores` con `proveedores.controller.ts`,
  `proveedores.service.ts`, DTOs, y un subflujo de importación
  (`proveedores-import.controller.ts` / `proveedores-import.service.ts`) con
  utilidades de parsing (`import-csv.util`, `import-format.util`,
  `import-headers.util`). La consulta por RUT al SII vive en el subdirectorio
  `sii/` mediante un patrón *provider/factory*.
- **Frontend (Vue 3):** *store* `proveedores.ts` y vistas en `views/proveedores/`,
  con acceso HTTP centralizado en `lib/api.ts`.
- **Persistencia:** modelo `Proveedor` en Prisma sobre PostgreSQL.

## 13. Resultados obtenidos y pruebas que los respaldan

La funcionalidad cuenta con pruebas unitarias reales en el backend, entre otras:

- `proveedores.service.spec.ts` — reglas de negocio (RUT, duplicados,
  baja/reactivación, aislamiento por *tenant*).
- `dto/proveedores.dto.spec.ts` — validación de DTOs.
- `proveedores-import.service.spec.ts`, `import-csv.util.spec.ts`,
  `import-format.util.spec.ts`, `proveedor-reciente.util.spec.ts` — importación
  masiva.
- `sii/sii.service.spec.ts`, `sii/apigateway-sii.provider.spec.ts` — consulta SII.
- `shared/rut.util.spec.ts` — normalización/validación de RUT.

En el frontend, el *store* se verifica con `stores/__tests__/proveedores.test.ts`.
Estas pruebas se ejecutan automáticamente en el *pipeline* de CI (ver
`Pruebas/EstrategiaPruebas.md`).

> **Distinción de estado:** CRUD, control de duplicados e importación masiva están
> **implementados y verificados por pruebas**. La ampliación de ficha (SPEC-013)
> está **parcialmente presente** en el modelo de datos, pero su SPEC permanece en
> estado **Borrador** (varios criterios de aceptación marcados como pendientes).
