# Kuentas – Documentación técnica del módulo contable

> Evidencia de documentación del Proyecto APT – PittySoft (Capstone Grupo 8).
> Estado del módulo al 13 de octubre de 2026.

## 1. Descripción general

**Kuentas** es el módulo contable de PittySoft. Es una aplicación web de
contabilidad orientada a la gestión contable y tributaria chilena (plan de
cuentas, comprobantes contables, libros e informes, Formulario 22).

**Problema que resuelve.** Centralizar el ciclo contable de la empresa y sus
clientes en un solo sistema: mantener el plan de cuentas, registrar comprobantes
con partida doble, generar libros e informes contables y preparar la información
tributaria del cierre anual, reduciendo la digitación manual.

**Relación con PittySoft.** Kuentas es uno de los módulos de la solución
PittySoft. Técnicamente se desarrolla como una aplicación independiente
(repositorio, base de datos y frontend propios), con datos aislados del resto de
los módulos, y comparte con ellos la infraestructura en la nube.

**Levantamiento de requerimientos.** El desarrollo partió del análisis del
sistema contable que la empresa utilizaba anteriormente: se levantaron sus menús
y formularios, se validaron las necesidades con el usuario contable de la
empresa y cada funcionalidad se definió como una especificación (SPEC) antes de
programarla.

## 2. Equipo y forma de trabajo

- **Desarrollo del módulo:** a cargo de Gabriel Bermar, junto a su compañero
  Pablo, trabajando en conjunto sobre backend y frontend.
- **Metodología:** desarrollo guiado por especificaciones (SPECS). Cada
  funcionalidad se describe primero en un documento con su objetivo, criterios
  de aceptación y reglas de negocio; luego se implementa, se prueba y se valida
  con el usuario.
- **Uso de IA:** se utilizaron asistentes de IA como apoyo para acelerar tareas
  de desarrollo a partir de las SPECS, con revisión y validación del equipo de
  todo el código incorporado.
- **Validación con el usuario:** las dudas de reglas contables y formatos de
  archivo se registran y se validan con el usuario contable de la empresa.

## 3. Alcance del módulo

**Implementado:**
- **Plan de Cuentas** (SPEC-002/013): catálogo jerárquico por empresa, creado a
  partir de un Plan Base de **305 cuentas** en formato SII `N.NN.NN.NN`, con
  cuentas protegidas.
- **Registro de comprobantes (Voucher)** (SPEC-017): partida doble con validación
  de cuadratura y numeración correlativa.
- **Importación masiva de comprobantes** (SPEC-053) y **plantillas de
  comprobantes** recurrentes (SPEC-018).
- **Homologación y balance histórico** (SPEC-003): migración de saldos de un
  sistema anterior con generación del asiento de apertura.
- **Libros e informes:** Libro Diario (019), Libro Mayor (021), Estado de
  Resultado (026), Balance General de 8 columnas (028), Balance Clasificado
  (027), Reporte por Centro de Costo (025) e Informes Analíticos (024).
- **Maestros:** clientes, proveedores, representantes, contadores, centros de
  costo, tipos de documento, parámetros, factores y datos de facturación.
- **Registro de compras, ventas y honorarios** (036/037/029) y sus importaciones
  (050/051/052); clonar e importar plan de cuentas (057/058/059); generador de
  pagos (039); plantillas de importación (043); foliador de hojas (056).

**Implementado parcialmente:**
- **Utilitario de Apertura** (054): carga manual lista; carga automática desde
  archivos pendiente.
- **Libro de Compras y Ventas** (020), **Libros de Honorarios y Boletas**
  (022/023), **Libros 14 Ter** (033/034/035) y **Conciliación Bancaria** (040):
  funcionales, con algunos cálculos tributarios aún simplificados.
- **Motor Tributario / Cierre** (004) y **Formulario 22** (006): en etapa piloto.
- **Sincronización de documentos con el SII** (005): lógica implementada sobre
  un conector de pruebas, a la espera de la integración real.
- **Migración desde el sistema anterior con detalle por RUT** (078):
  implementada, en revisión antes de integrarse a la rama principal.

**No incluido en esta etapa:** dashboard F29, vistas diferenciadas por perfil de
usuario y soporte multimoneda (ver sección 12).

## 4. Arquitectura

**Monorepo** con dos aplicaciones: `kuentas-api` (backend) y `kuentas-web`
(frontend).

**Backend (`kuentas-api`)**
- **NestJS 10** sobre **Fastify**, **Prisma 5** como ORM y **PostgreSQL**.
- **Arquitectura N-Tier:** `HTTP → Controller (rutas, DTO, guards) → Service
  (lógica de negocio + Prisma) → PostgreSQL`.
- Un módulo por funcionalidad (`*.module.ts`, `*.controller.ts`, `*.service.ts`,
  `dto/`), más una capa compartida (`shared/`) con acceso a datos, cifrado,
  guards y validadores (incluida la validación de RUT).
- API REST versionada (`/api/v1/...`), validación global de entradas y
  documentación con **Swagger**.
- Autenticación con **JWT** y contraseñas con **bcrypt**; límite de peticiones
  por minuto (rate limiting).

**Frontend (`kuentas-web`)**
- **Vue 3** (Composition API) + **TypeScript** + **Vite**.
- **Pinia** (estado), **Vue Router**, **Tailwind CSS** y **axios**.
- **Patrón MVVM:** Vista (`.vue`) → ViewModel (store Pinia) → Modelo (tipos y
  cliente HTTP). Cerca de 65 vistas y 50 stores; las rutas exigen sesión
  iniciada.

**Base de datos**
- PostgreSQL gestionado con Prisma: **44 modelos** y **35 migraciones**
  versionadas.
- **Multi-tenant:** cada registro pertenece a una organización (`tenantId`) y
  todas las consultas se filtran por ella.

**Despliegue e integración continua**
- Contenedores Docker desplegados en **Google Cloud Run**.
- **GitHub Actions**: el pipeline ejecuta las pruebas antes de cada despliegue;
  las migraciones de base de datos se aplican automáticamente al iniciar.
- Las credenciales y secretos se administran fuera del repositorio.

## 5. Modelo de datos

Entidades principales (`prisma/schema.prisma`):

| Grupo | Entidades | Propósito |
|---|---|---|
| Núcleo | `Tenant`, `User`, `Company` | Organización, usuarios y empresas clientes |
| Plan de cuentas | `PlanCuentaBase`, `CuentaBase`, `PlanCuentaEmpresa`, `Cuenta`, `EquivalenciaPlan` | Plantilla maestra y copia editable por empresa |
| Comprobantes | `Voucher`, `VoucherLinea`, `PlantillaVoucher` | Asientos contables con sus líneas |
| Migración y apertura | `CargaHistorica`, `MapeoHistorico`, `ComprobanteApertura` | Homologación de saldos y asiento de apertura |
| Tributario | `PeriodoContable`, `AjusteRenta`, `CorreccionMonetariaDetalle`, `DocumentoF22`, `CasillaF22` | Cierre anual y Formulario 22 |
| Maestros | `Cliente`, `Proveedor`, `Representante`, `Contador`, `CentroCosto`, `TipoDocumento` | Datos de referencia para los asientos |

**Relaciones clave**
- `Tenant 1—N Company 1—N Cuenta`: el plan de cuentas es **por empresa**.
- `Company 1—N Voucher 1—N VoucherLinea N—1 Cuenta`: cada línea se imputa a una
  cuenta y puede asociar cliente o proveedor, centro de costo y documento.
- Integridad contable: no se puede eliminar un maestro que ya fue usado en un
  asiento.

## 6. Plan de Cuentas

- **Jerarquía:** árbol de cuentas donde el código define la posición (por
  ejemplo, `1.01.001` depende de `1.01`).
- **Catálogo maestro y catálogo por empresa:** el Plan Base (no editable) se
  copia a cada empresa, que puede agregar cuentas propias o clonar el plan de
  otra empresa. Cada cuenta registra su origen (base, cliente o clonada).
- **Protección de cuentas:**
  1. **Cuentas intocables:** 22 cuentas críticas (incluidas Caja, Bancos e IVA)
     no se pueden modificar ni eliminar; el listado fue validado con el usuario
     contable.
  2. **Cuentas de agrupación no imputables:** el sistema rechaza asientos que
     usen cuentas que agrupan a otras.
- **Vigencia:** una cuenta puede desactivarse para nuevos asientos sin perder su
  historial.
- **Uso en otros procesos:** alimenta los comprobantes, los informes, el mapeo a
  casillas del Formulario 22 y la clasificación tributaria del cierre anual.

## 7. Registro de comprobantes (Voucher)

- **Tipos:** Ingreso, Egreso y Traspaso.
- **Datos:** período, fecha, glosa, comentario y líneas con Debe y Haber.
- **Reglas de negocio validadas por el sistema:**
  - **Cuadratura:** el total del Debe debe ser igual al del Haber; si no, el
    comprobante se rechaza indicando la diferencia.
  - **Numeración correlativa** automática por mes y tipo de comprobante, segura
    ante registros simultáneos.
  - Solo se imputa a cuentas activas e imputables de la empresa.
  - Cada línea se asocia a un cliente **o** a un proveedor, nunca a ambos.
  - Los centros de costo y maestros deben pertenecer a la misma empresa u
    organización.
- **Inmutabilidad:** un comprobante registrado no se modifica (solo su
  comentario), para resguardar la trazabilidad contable.

## 8. Importación masiva

- **Comprobantes** (SPEC-053): carga desde archivo; cada comprobante pasa por las
  mismas validaciones que el registro manual (cuadratura, correlativo, cuentas
  válidas).
- **Libros y maestros** (compras/ventas, honorarios, boletas, plan de cuentas y
  auxiliares): carga desde CSV con reconocimiento flexible de columnas. Las filas
  con errores (código duplicado, RUT repetido, datos faltantes) se informan sin
  detener el resto de la importación.
- **Pendiente:** confirmar con el usuario los formatos definitivos de algunos
  archivos.

## 9. Migración y apertura contable

- **Homologación del balance histórico** (SPEC-003): se carga el balance del
  sistema anterior, cada cuenta de origen se asocia a una cuenta del nuevo plan
  y, si el balance cuadra, se genera el asiento de apertura. Una vez confirmada,
  la homologación no se puede alterar.
- **Asiento de apertura manual** (SPEC-054): carga directa de saldos iniciales.
- **Migración con detalle por RUT** (SPEC-078): apertura que conserva el detalle
  por cliente y proveedor; implementada y en revisión.
- **Cambio de plan en empresas con movimientos:** herramienta de equivalencias
  entre el plan antiguo y el nuevo, que conserva los informes históricos.

## 10. Reportes y libros contables

Todos los informes se calculan con datos reales de la base de datos, filtrados
por empresa y período. Se exportan mediante la impresión del navegador (a PDF o
papel).

| Reporte | Qué entrega | Estado |
|---|---|---|
| Libro Diario (019) | Asientos del período en orden cronológico | Implementado |
| Libro Mayor (021) | Movimientos y saldo acumulado por cuenta, incluyendo apertura | Implementado |
| Estado de Resultado (026) | Ingresos, gastos y resultado en 3 niveles de detalle | Implementado |
| Balance General (028) | Balance de 8 columnas | Implementado |
| Balance Clasificado (027) | Activo, Pasivo, Patrimonio y Resultado | Implementado |
| Centro de Costo (025) | Movimientos agrupados por centro de costo | Implementado |
| Informes Analíticos (024) | Balance de comprobación por rubro | Implementado |
| Libro Compras y Ventas (020) | Neto, IVA y total por documento | Parcial: identificación de IVA simplificada |
| Libros 14 Ter (033/034/035) | Libro de caja e ingresos/egresos | Parcial: columnas tributarias por afinar |
| Formulario 22 (006) | Casillas del F22 con trazabilidad por cuenta, exportable a CSV | Piloto |

## 11. Seguridad y reglas de negocio

- **Autenticación:** inicio de sesión con contraseña cifrada (bcrypt) y tokens
  JWT con renovación; todas las rutas de la API están protegidas por defecto.
- **Aislamiento de datos:** cada organización solo accede a su propia
  información.
- **Validaciones:** validación estricta de todas las entradas, verificación del
  dígito verificador del RUT, cuadratura de comprobantes y unicidad de RUT y
  códigos.
- **Datos sensibles:** las credenciales tributarias se almacenan cifradas.
- **Integridad contable:** cuentas protegidas, comprobantes inmutables y
  restricciones en la base de datos que impiden borrar información usada en
  asientos.

## 12. Estado actual y trabajo futuro

**Integraciones externas.** La integración con el SII está diseñada mediante un
conector intercambiable. Actualmente funciona con un **conector de pruebas** que
permite desarrollar y demostrar los flujos; la conexión real se abordará en la
Fase 7 (SPEC-066).

**Próximas etapas:**
- Integración real con el SII y sincronización periódica automática.
- Control de roles y permisos por perfil de usuario (SPEC-070).
- Ampliación del motor tributario: Global Complementario y pérdida tributaria
  de arrastre (SPEC-067/068), y mapeo de cuentas a Declaraciones Juradas
  (SPEC-069).
- Salidas oficiales: Formulario 22 y libros en PDF con formato oficial,
  Declaraciones Juradas y control de folios.
- Mejoras de experiencia de usuario: dashboard F29, selector de empresa y otros
  ajustes (SPEC-060 a 065).
- Conectar configuraciones ya disponibles (centralizaciones, configuración del
  Estado de Resultado) con los procesos que las utilizan, y generar
  automáticamente el asiento de las notas de crédito.
- Pantallas en desarrollo: certificados y resumen de honorarios,
  reordenamiento de comprobantes.
- Nuevo módulo de Remuneraciones (SPEC-072) y soporte multimoneda.

## 13. Pruebas y validación

- **Pruebas automatizadas:** 51 archivos de prueba en el backend (Jest) y 52 en
  el frontend (Vitest), que cubren servicios, validaciones y vistas.
- **Integración continua:** las pruebas se ejecutan automáticamente y bloquean
  el despliegue si fallan.
- **Reglas de negocio cubiertas por pruebas:** cuadratura, correlativo,
  aislamiento entre organizaciones, rechazo de cuentas no imputables y unicidad
  de RUT.
- **Datos de prueba:** scripts de carga de datos demo para desarrollo y
  presentaciones.
- **Validación funcional:** registro de dudas de negocio validadas con el
  usuario contable de la empresa.
- **Pendiente:** pruebas de extremo a extremo (end-to-end) y pruebas contra el
  SII real.

## 14. Conclusión

Kuentas tiene **implementado el ciclo contable principal**: maestros, plan de
cuentas, registro de comprobantes con partida doble validada, libros e informes
financieros. Esto se apoya en una base técnica consistente (NestJS N-Tier con
Prisma y PostgreSQL multi-tenant, Vue 3 con MVVM y Pinia), cerca de 100 archivos
de pruebas automatizadas y despliegue continuo en Google Cloud Run.

El bloque tributario (motor de cierre, Formulario 22 y sincronización con el
SII) se encuentra en etapa piloto y su avance depende principalmente de la
integración real con el SII y de la validación de reglas tributarias con el
usuario. En resumen, Kuentas es un **MVP contable operativo**, listo para
demostrar el ciclo contable completo, con una hoja de ruta definida para su uso
tributario en producción.

## 15. Evidencias sugeridas

1. Plan de Cuentas: árbol jerárquico y cuentas protegidas.
2. Registro de comprobante: partida doble y mensaje de error cuando no cuadra.
3. Importación masiva: resultado de una carga con filas correctas y con error.
4. Libro Diario y Libro Mayor de un período.
5. Balance General y Estado de Resultado.
6. Diagrama del modelo de datos (`Company`, `Cuenta`, `Voucher`, `VoucherLinea`).
7. Documentación de la API en Swagger.
8. Resultado de la ejecución de las pruebas automatizadas.
