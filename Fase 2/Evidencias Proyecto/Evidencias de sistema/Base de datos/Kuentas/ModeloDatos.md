# Modelo de Datos — Entidades principales

El modelo de datos de Kuentas es **multi-tenant**: todo cuelga de un `Tenant` (la
consultora contable) y de las `Company` (empresas que esa consultora contabiliza).
El corazón transaccional es el **Voucher** (comprobante contable) y sus líneas, que
registran los movimientos bajo partida doble e imputan cada línea a una `Cuenta` del
plan de cuentas, opcionalmente a un `CentroCosto` y a una contraparte (`Cliente` o
`Proveedor`).

A continuación se muestran **solo las entidades principales** con sus campos más
relevantes. Se omiten atributos secundarios, de auditoría y entidades satélite para
mayor claridad.

## Diagrama entidad-relación

```mermaid
erDiagram
    Tenant ||--o{ Company     : "tiene"
    Tenant ||--o{ Cliente     : "catálogo"
    Tenant ||--o{ Proveedor   : "catálogo"

    Company ||--o{ Cuenta       : "plan de cuentas"
    Company ||--o{ CentroCosto  : "centros de costo"
    Company ||--o{ Voucher      : "comprobantes"

    Voucher ||--|{ VoucherLinea : "líneas (partida doble)"

    Cuenta      ||--o{ VoucherLinea : "imputa"
    CentroCosto ||--o{ VoucherLinea : "asigna"
    Cliente     ||--o{ VoucherLinea : "contraparte"
    Proveedor   ||--o{ VoucherLinea : "contraparte"

    Tenant {
        uuid   id PK
        string name
        string slug UK
    }

    Company {
        uuid   id PK
        uuid   tenantId FK
        string rut
        string razonSocial
        string tipoRegimen
    }

    Cuenta {
        uuid    id PK
        uuid    companyId FK
        string  codigo
        string  nombre
        int     nivel
        string  tipo
        boolean esIntocable
    }

    CentroCosto {
        uuid   id PK
        uuid   companyId FK
        string codigo
        string nombre
    }

    Voucher {
        uuid    id PK
        uuid    companyId FK
        int     anio
        int     mes
        int     numero
        string  tipoMovimiento
        date    fecha
        string  glosa
        decimal totalDebe
        decimal totalHaber
    }

    VoucherLinea {
        uuid    id PK
        uuid    voucherId FK
        uuid    cuentaId FK
        uuid    centroCostoId FK "opcional"
        uuid    clienteId FK "opcional"
        uuid    proveedorId FK "opcional"
        decimal debe
        decimal haber
        decimal monto
        string  numeroDocumento
    }

    Cliente {
        uuid   id PK
        uuid   tenantId FK
        string rut
        string rsocial
    }

    Proveedor {
        uuid   id PK
        uuid   tenantId FK
        string rut
        string rsocial
    }
```

## Explicación

- **Tenant** — Representa a la consultora contable. Es la raíz del aislamiento
  multi-tenant: casi todas las consultas se acotan por su identificador. De él
  cuelgan tanto las empresas como los catálogos compartidos de clientes y proveedores.

- **Company** — Empresa contabilizada por la consultora. Una consultora puede tener
  muchas empresas, y cada una posee su propio plan de cuentas, sus centros de costo y
  sus comprobantes.

- **Cuenta** — Cuenta del plan de cuentas de una empresa. Se organiza jerárquicamente
  (niveles) y distingue cuentas "intocables" (estructurales) de las creadas por el
  usuario. Cada línea de comprobante se imputa a una cuenta.

- **CentroCosto** — Clasificador de gestión por empresa (por ejemplo, áreas o
  unidades). Es opcional en cada línea de comprobante.

- **Voucher** — Comprobante contable (el asiento). Agrupa varias líneas en un período
  (año/mes), lleva un número correlativo y mantiene los totales de Debe y Haber, que
  deben cuadrar entre sí.

- **VoucherLinea** — Línea del comprobante. Registra Debe, Haber y Monto, imputa a una
  `Cuenta` y, opcionalmente, a un `CentroCosto` y a una contraparte. La contraparte es
  **mutuamente excluyente**: una línea puede apuntar a un `Cliente` **o** a un
  `Proveedor`, nunca a ambos.

- **Cliente / Proveedor** — Maestros de contrapartes de la consultora (catálogo por
  tenant). Se identifican por RUT y se referencian desde las líneas de los comprobantes
  cuando el movimiento involucra a un tercero.
