# SPEC-044 (resumen académico) — Cuenta individual de vendedora

> **Evidencia académica — Proyecto CAPSTONE, Ingeniería en Informática (Duoc UC).**
> Resumen de una especificación real del módulo POS de PittySoft. No reproduce la
> especificación completa ni constituye una especificación operativa. No contiene nombres,
> correos, PIN ni datos personales reales; los ejemplos son ficticios.

## 1. Objetivo

Crear una **cuenta individual** para una vendedora, con credenciales propias y permisos de
rol **vendedora**. Al iniciar sesión, el POS debe identificarla automáticamente como
responsable de sus ventas, eliminando la necesidad de seleccionarla manualmente durante el
cobro.

## 2. Contexto del problema

El POS disponía de autenticación por empresa, usuario y PIN, y durante la venta mostraba un
modal «Seleccionar vendedora». La nueva funcionalidad aprovecha la **identidad de la sesión**
para asociar las ventas de forma automática, conservando el selector manual para una posible
reactivación futura.

## 3. Alcance de la implementación

- Crear una cuenta individual de vendedora con usuario y PIN propios.
- Asignar **permisos operativos** de vendedora, **sin privilegios administrativos**.
- Habilitar el flujo autorizado de ventas, clientes, citas y órdenes de trabajo.
- Asociar **automáticamente** cada nueva venta a la vendedora cuando opere con su sesión.
- Mostrar la vendedora responsable en el detalle de la venta.
- **Desactivar** el modal de selección manual, **conservando** su implementación para una
  futura reactivación por configuración.

**No incluye:** modificar otros usuarios o permisos existentes, eliminar el componente de
selección manual, reasignar ventas históricas ni cambiar la arquitectura multi-tenant.

## 4. Criterios de aceptación principales

- **CA-1 — Inicio de sesión:** con una cuenta activa, la vendedora inicia sesión con su
  identidad individual.
- **CA-2 — Permisos operativos:** con rol de vendedora, accede solo a las operaciones
  autorizadas (ventas, clientes, citas, órdenes de trabajo).
- **CA-3 — Restricción administrativa:** al intentar acceder a configuraciones
  administrativas, gestión de usuarios o metas de facturación, el sistema **deniega** la
  operación tanto en la interfaz como en la API.
- **CA-4 — Selección automática:** al entrar al flujo de venta, el sistema identifica
  automáticamente a la vendedora y **no muestra** el modal de selección.
- **CA-5 — Registro de ventas:** al confirmarse una venta, el **backend** registra su
  identidad como responsable, **sin aceptar que el frontend la sustituya**.
- **CA-6 — Trazabilidad:** al consultar el detalle de una venta, aparece la vendedora
  responsable, incluso después de cerrar sesión.
- **CA-7 — Conservación del selector:** el componente de selección manual permanece disponible
  (desactivado por configuración) para una futura reactivación.
- **CA-8 — Protección de usuarios existentes:** habilitar la nueva cuenta no altera
  credenciales, roles ni datos de cuentas previas.

## 5. Reglas de negocio

- La asignación de la vendedora responsable la determina **la sesión autenticada**, con
  prioridad sobre cualquier valor enviado por el frontend (seguridad e integridad del dato).
- La relación entre usuario y vendedora es **1 a 1** y **opcional**: no todas las vendedoras
  tienen cuenta de login, y un usuario corresponde a lo sumo a una vendedora.
- Solo se auto-asigna una vendedora **activa** (una vendedora desactivada no se asigna).
- La asociación venta → vendedora es **histórica**: no se reasignan ventas anteriores.

## 6. Autenticación, rol y asociación usuario–vendedora–sucursal

- **Autenticación por cuenta individual:** la vendedora accede con su propio usuario y PIN; el
  PIN se almacena de forma segura (nunca en texto plano).
- **Rol de vendedora:** se reutiliza el modelo de roles y permisos existente; el rol concede
  únicamente permisos operativos, sin acceso administrativo.
- **Asociación usuario ↔ vendedora:** el modelo `Vendedora` incorpora una referencia opcional y
  única al usuario (`userId`). Cuando existe, el token de sesión transporta la vendedora
  vinculada para que el POS la identifique automáticamente.
- **Vendedora ↔ sucursal:** la relación es de muchos a muchos a través de la tabla puente
  `VendedoraSucursal`, respetando el `tenant_id` y la sucursal correspondiente.

## 7. Asignación automática de ventas y restricciones de acceso

- **Asignación automática (CA-4/CA-5):** en el cierre de venta del POS, el backend resuelve la
  vendedora responsable **a partir del usuario de la sesión**. Si el usuario autenticado está
  vinculado a una vendedora activa del tenant, esa vendedora se asigna a la venta y se
  **ignora** cualquier identificador enviado por el frontend. Si no hay vínculo, se conserva el
  comportamiento anterior (selección manual).
- **Restricciones de acceso (CA-3):** los permisos del rol de vendedora limitan las operaciones
  disponibles; los intentos de acceder a funciones administrativas se rechazan tanto en la
  interfaz como en la API, mediante los guards de permisos del backend.

## 8. Breve explicación técnica

- **Modelo de datos (Prisma):** se añadió el campo opcional y único `Vendedora.userId` (con su
  relación inversa en `User`), manteniendo la migración **aditiva**.
- **Autenticación (backend):** al autenticar, el sistema busca la vendedora vinculada al
  usuario y la incluye en el payload del token (o `null` si no existe vínculo).
- **Cierre de venta (backend):** la lógica de resolución de la vendedora
  (`resolveVendedora`) prioriza la vendedora de la sesión por sobre el valor del frontend,
  garantizando el CA-5.
- **Frontend (Vue + Pinia):** cuando la sesión ya identifica a la vendedora, el modal de
  selección se omite; el componente permanece en el código, desactivado por configuración.
- **Multi-tenant:** todas las consultas se mantienen acotadas por `tenant_id` y por la
  sucursal correspondiente.

> Ejemplo **ficticio** de datos de prueba (no corresponden a ninguna persona real):
> usuario `vendedora.demo`, sucursal `SUCURSAL_DEMO_1`. Las credenciales reales **no** se
> incluyen en documentación, código ni logs.

## 9. Resultados obtenidos y pruebas que los respaldan

**Implementado y verificado por pruebas automáticas:**

- **Autenticación:** existen pruebas que verifican que el token de sesión incluye la vendedora
  vinculada cuando corresponde y `null` cuando no hay vínculo (`auth.service.spec.ts`).
- **Asignación automática:** las pruebas del service de ventas cubren la resolución de la
  vendedora desde la sesión y su prioridad sobre el frontend (CA-5).
- **Maestro de vendedoras y permisos:** el módulo de vendedoras cuenta con pruebas unitarias
  (`vendedoras.service.spec.ts`), y el frontend incluye pruebas del store asociado
  (`posVendedoras.test.ts`).
- Todas estas pruebas se ejecutan automáticamente en la integración continua (ver
  `../Pruebas/EstrategiaPruebas.md`).

**Distinción entre implementado y verificado operativamente:**

- La funcionalidad (modelo, autenticación, asignación automática y restricciones) está
  **implementada y cubierta por pruebas automáticas**.
- La **verificación funcional end-to-end en el entorno operativo** (alta real de la cuenta e
  inicio de sesión en la aplicación desplegada) depende de un procedimiento administrativo de
  provisión de la cuenta que se realiza por fuera del desarrollo; por tanto, no se afirma como
  verificada en este documento.
