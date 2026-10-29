# Estrategia de Pruebas — Módulo Kuentas

Las pruebas automatizadas son parte del criterio de "Hecho" de cada funcionalidad: una
spec no se considera terminada sin pruebas que pasen. La batería se ejecuta tanto en
local como en el pipeline, y actúa como *gate* antes de integrar o desplegar.

## Herramientas

| Capa | Framework | Apoyo |
|---|---|---|
| **Backend** (API, NestJS) | **Jest** (con `ts-jest` para TypeScript) | utilidades de testing de NestJS, mocks de la capa de datos |
| **Frontend** (Vue 3) | **Vitest** | `@vue/test-utils` para montar componentes y `happy-dom` como entorno DOM |

El backend usa **Jest** por su integración nativa con el ecosistema NestJS; el frontend
usa **Vitest**, que se alinea con el empaquetador Vite del proyecto y ofrece una
ejecución rápida de pruebas de *stores* y componentes.

## Cantidad aproximada de archivos de prueba

| Capa | Archivos de prueba (aprox.) |
|---|---|
| Backend (`*.spec.ts`) | ~70 |
| Frontend (`*.spec.ts`) | ~60 |
| **Total** | **~130** |

Las pruebas se ubican junto al código que verifican: en el backend, un archivo
`*.spec.ts` por *service*/módulo; en el frontend, pruebas por *store* de Pinia y por
vista.

## Reglas de negocio cubiertas

Las pruebas se concentran en la lógica contable y tributaria, que es donde está el
riesgo del dominio. Entre las reglas cubiertas:

- **Cuadratura de partida doble**: un comprobante solo se guarda si el total del Debe
  iguala al total del Haber; los casos de descuadre deben ser rechazados.
- **Resolución e imputación de cuentas**: validar que la cuenta exista en el plan de
  cuentas de la empresa y sea imputable antes de aceptar una línea.
- **Aislamiento multi-tenant**: las operaciones de lectura y escritura quedan acotadas
  al tenant del usuario; nunca deben alcanzar datos de otro tenant.
- **Importación masiva de vouchers**: parseo del archivo, agrupación de líneas en
  comprobantes, validación por fila y reporte de errores sin invalidar las filas
  correctas.
- **Maestros de clientes y proveedores**: altas, unicidad por RUT dentro del tenant y
  enlaces con las líneas de comprobante.
- **Motor tributario**: cálculos contables/tributarios y manejo de casos de descuadre.
- **Plantillas e importaciones auxiliares**: precarga de asientos recurrentes e
  importación de catálogos.

En el frontend, las pruebas verifican el comportamiento de los *stores* (estado,
acciones, manejo de estados de carga y error) y el renderizado de las vistas clave,
como el Registro de Voucher.

## Integración con CI

El pipeline de integración continua se ejecuta en **cada pull request y en cada push**
(excepto a la rama principal, que pasa por el flujo de despliegue). Para cada aplicación
corre, en orden:

1. **Verificación de tipos** (type-check).
2. **Pruebas automatizadas** (Jest en el backend, Vitest en el frontend).
3. **Build** de la aplicación.

Si cualquiera de estos pasos falla, el cambio **no se integra**. Además, el despliegue a
producción está **gateado por las mismas pruebas**: solo se construye y publica la
imagen del contenedor si la verificación de tipos y las pruebas pasan. Así ninguna
versión llega al entorno desplegado sin haber superado la batería completa.

## Comando para ejecutar las pruebas

Desde cada aplicación:

```bash
# Backend (API)
npm test        # ejecuta Jest

# Frontend (web)
npm test        # ejecuta "vitest run"
```

Durante el desarrollo también está disponible el modo interactivo (`npm run test:watch`),
que re-ejecuta las pruebas afectadas al guardar cambios.
