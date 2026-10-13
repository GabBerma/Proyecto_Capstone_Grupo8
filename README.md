# PittySoft – Proyecto Capstone Grupo 8

## Descripción del proyecto

PittySoft es una solución informática modular desarrollada como parte del Proyecto APT de la asignatura Capstone de Ingeniería en Informática de Duoc UC.

El proyecto busca centralizar y optimizar procesos administrativos, comerciales y contables de una empresa del rubro óptico, reemplazando progresivamente procesos que actualmente se encuentran distribuidos entre distintos sistemas, planillas y fuentes de información.

La solución se encuentra compuesta por tres módulos principales:

- **POS – Punto de Venta:** orientado a los procesos de venta, clientes, citas, medios de pago, devoluciones y operación comercial de la óptica.
- **SAP-lite – Control de Gestión:** orientado a proveedores, facturas de compra, importaciones financieras, Plan de Cuentas, arriendos, pagos de servicios y procesos administrativos.
- **Kuentas – Contabilidad:** orientado a procesos contables, plan de cuentas, comprobantes, importación de información y procesos tributarios en desarrollo.

---

## Integrantes

- **Axel Blanlot** – Responsable principal del módulo POS.
- **Sebastian Quezada** – Responsable principal del módulo SAP-lite.
- **Gabriel Bermar** – Responsable principal del módulo Kuentas.

Las actividades de integración, pruebas, validación, documentación y seguimiento del proyecto son desarrolladas colaborativamente por los integrantes del equipo.

---

## Objetivo general

Desarrollar una solución informática modular que permita centralizar y optimizar los procesos administrativos, comerciales y contables de una empresa del rubro óptico, mediante la implementación e integración de los módulos SAP-lite, POS y Kuentas.

---

## Módulos del proyecto

### POS – Punto de Venta

El módulo POS contempla funcionalidades relacionadas con:

- Gestión de clientes.
- Gestión de productos.
- Flujo de venta.
- Carrito de compra.
- Medios de pago.
- Gestión de vendedoras por sucursal.
- Agenda y citas.
- Devoluciones.
- Integración con BSale.
- Integración de pruebas con Mercado Pago.
- Indicadores y Dashboard.

Actualmente existen funcionalidades operativas y otras que continúan en proceso de validación e integración.

### SAP-lite – Control de Gestión

SAP-lite contempla funcionalidades relacionadas con:

- Gestión e importación de proveedores.
- Consulta de información de proveedores mediante SII.
- Asociación de proveedores a cuentas contables.
- Facturas de compra y notas de crédito.
- Sincronización de documentos con BSale.
- Importación de cartolas bancarias.
- Importación de liquidaciones Transbank.
- Consulta y sincronización de pagos de Mercado Pago.
- Arriendos y gastos comunes.
- Plan de Cuentas.
- Pagos de servicios.
- Procesos de conciliación financiera.
- Reportes de gastos.

Actualmente, el módulo cuenta con funcionalidades disponibles para Proveedores, Facturas de Compra, Importaciones Financieras, Mercado Pago, Arriendos y Plan de Cuentas. Pagos de Servicios continúa en etapa de prototipo y validación, mientras que Reportes de Gastos permanece como funcionalidad pendiente de desarrollo.

El desarrollo continúa enfocado en validar las reglas de negocio, completar las integraciones pendientes y relacionar correctamente proveedores, documentos, movimientos financieros y cuentas contables.

### Kuentas – Contabilidad

Kuentas contempla funcionalidades relacionadas con:

- Plan de Cuentas.
- Registro de comprobantes contables.
- Importación masiva de vouchers.
- Soporte multimoneda.
- Migración de información contable.
- Libros e informes contables.
- Indicadores contables.
- Procesos tributarios.
- Validación de flujos con el contador de la empresa.

Actualmente se continúa trabajando en la validación de la plataforma y en funcionalidades que dependen de información o integraciones externas.

---

## Metodología de trabajo

Para el desarrollo de PittySoft se utiliza un enfoque de trabajo híbrido, basado principalmente en **Spec-Driven Development (SDD)** y complementado con prácticas seleccionadas del **framework Scrum**.

Las especificaciones funcionales o **SPEC** permiten documentar requerimientos, alcance, comportamiento esperado y criterios de aceptación antes de implementar determinadas funcionalidades.

El equipo también utiliza un Backlog para organizar y priorizar el trabajo, junto con reuniones de seguimiento, validaciones con usuarios y control de versiones mediante GitHub.

---

## Tecnologías y herramientas

Durante el desarrollo del proyecto se utilizan distintas tecnologías y herramientas de apoyo, entre ellas:

- Git y GitHub para control de versiones.
- Aplicaciones web para los módulos de PittySoft.
- Bases de datos para la persistencia de información.
- Integraciones mediante APIs y servicios externos.
- BSale para procesos documentales y tributarios.
- Mercado Pago para procesos asociados a pagos.
- Servicios del SII para consulta y validación de información tributaria de proveedores.
- Archivos Excel y CSV para procesos de importación.
- Documentación funcional mediante SPEC.

Las tecnologías específicas de frontend, backend y base de datos utilizadas por cada módulo serán documentadas en los documentos técnicos correspondientes.

---

## Estado actual del proyecto

El proyecto se encuentra actualmente en **Fase 2 – Desarrollo Proyecto APT**.

Durante esta fase se está trabajando principalmente en:

- Desarrollo y validación de funcionalidades.
- Pruebas de los módulos POS, SAP-lite y Kuentas.
- Ajustes derivados de la retroalimentación de los usuarios.
- Integraciones con servicios externos.
- Documentación técnica y funcional.
- Seguimiento del Plan de Trabajo.
- Preparación progresiva de la integración de PittySoft.

El proyecto aún se encuentra en desarrollo, por lo que algunas funcionalidades pueden encontrarse en estado de prototipo, validación o pendientes de integración.

---

## Estructura de documentación

La documentación del proyecto será organizada progresivamente dentro del repositorio, incluyendo:

- Definición y visión del proyecto.
- Backlog.
- Especificaciones funcionales.
- Arquitectura.
- Modelos de datos.
- Documentación de POS.
- Documentación de SAP-lite.
- Documentación de Kuentas.
- Pruebas y validaciones.
- Evidencias del Proyecto APT.
- Manuales y documentación técnica.

---

## Proyecto académico

Proyecto desarrollado para la asignatura **Capstone – PTY4614**, Ingeniería en Informática, Duoc UC.

**Grupo 8**
