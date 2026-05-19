# Sistema de Trazabilidad de Materiales — Alesig

**Cliente:** Consultoría Empresarial Alesig SC
**Contacto operativo:** Ing. Jessica Ramos
**Tipo de sistema:** Aplicación web full-stack para control y conciliación de inventario de materiales
**Audiencia de este documento:** Claude Code (asistente de programación) para arrancar la implementación del proyecto

---

## 0. Cómo usar este documento

Este es el documento maestro de especificaciones. Está diseñado para que un agente de programación (Claude Code) pueda arrancar el proyecto sin necesitar contexto previo. Las secciones están ordenadas de mayor a menor abstracción:

1. **Contexto y problema** (qué hace Alesig, qué duele hoy)
2. **Glosario de dominio** (términos del negocio, no negociables)
3. **Arquitectura propuesta** (stack, capas, decisiones técnicas)
4. **Modelo de datos detallado** (entidades, relaciones, restricciones)
5. **Especificación del importador SAP** (la pieza crítica del MVP)
6. **API y endpoints**
7. **Frontend y UX clave**
8. **Plan de implementación por fases**
9. **Criterios de aceptación del MVP**
10. **Decisiones abiertas pendientes con el cliente** (no asumir, preguntar)

---

## 1. Contexto del negocio

### 1.1 Qué hace Alesig

**Consultoría Empresarial Alesig SC** es una empresa mexicana especializada en **construcción, instalación y mantenimiento de sistemas de gas natural y gas LP**. Opera como contratista B2B para empresas distribuidoras de gas: recibe contratos de obra de sus clientes (las distribuidoras), ejecuta los proyectos con cuadrillas propias y gestiona el ciclo completo de materiales.

Sus servicios incluyen:
- Construcción de instalaciones de gas natural y gas LP (residencial y comercial)
- Asesoría en infraestructura para manejo seguro de gas
- Mantenimiento y soporte técnico de instalaciones existentes

Para prestar estos servicios, Alesig opera **almacenes de materiales especializados** (válvulas, accesorios de polietileno, bridas, tuberías, reguladores, etc.). Actualmente operan dos almacenes:

- **Toluca** — atiende contrato `12T2 Toluca Contrata`
- **CDMX** — atiende contrato `37X2 Metrogas Contrata`

El cliente contratante es **dueño del material** y del sistema SAP. Entrega el material físicamente a Alesig para que lo custodie, lo distribuya a sus cuadrillas y lo instale. Al final del ciclo, Alesig **certifica en el SAP del cliente** que el material fue instalado, y el cliente lo cierra contablemente. Alesig tiene usuario propio en el SAP de Naturgy con permisos de lectura y extracción. Jessica Ramos genera los Excels directamente desde ese usuario cuando lo necesita — no los solicita al cliente.

Los proyectos son tanto de **construcción nueva** de redes como de **mantenimiento** de instalaciones existentes.

> **Implicación para el sistema:** los proyectos deben distinguir entre tipo `CONSTRUCCION` y tipo `MANTENIMIENTO`, ya que pueden tener reglas distintas de certificación o reporteo ante el cliente.

### 1.2 Quiénes son los clientes

Ambos clientes pertenecen al grupo **Naturgy** (multinacional española, antes Gas Natural Fenosa), que opera en México bajo concesión de la **CRE (Comisión Reguladora de Energía)**:

| Centro SAP | Entidad legal | Zona geográfica |
|---|---|---|
| `12T2` | **Naturgy México S.A. de C.V.** | Toluca, Lerma, Metepec. Opera desde 1997. |
| `37X2` | **Comercializadora Metrogas S.A. de C.V.** | 16 alcaldías de CDMX y Zona Metropolitana. Red de ~4.8 millones de metros de tubería. |

Aunque son entidades legales distintas, forman parte del mismo grupo corporativo y comparten el mismo ecosistema SAP. Los códigos de centro (`12T2`, `37X2`) son identificadores dentro del SAP de Naturgy/Metrogas — no son sistemas de Alesig.

**Confirmado por Jessica Ramos (15/05/2026):**
- Los Excels SAP son extracciones directas desde el usuario SAP propio de Alesig dentro del sistema de Naturgy — no los genera Naturgy y los envía, sino que Jessica los exporta ella misma.
- Solo existen estos dos almacenes. No hay planes de un tercero en el corto plazo.
- **La columna clave para la conciliación es `Existencias`** — es el único campo que Jessica utiliza actualmente para comparar contra su control interno.

> **Implicación crítica para el sistema:** el SAP es propiedad del cliente (Naturgy), pero Alesig tiene usuario propio con acceso de lectura/extracción. Los Excels los genera Jessica directamente. La pregunta abierta #10 (integración directa vía API) depende de que Naturgy amplíe los permisos del usuario de Alesig en SAP, lo cual es una decisión corporativa de largo plazo.

### 1.3 El problema central

Hay dos "verdades" sobre el inventario que deben coincidir:

- **Verdad operativa**: lo que Alesig sabe que pasó físicamente (entró, salió a cuadrilla, se instaló, se prestó, se dañó, etc.)
- **Verdad SAP**: lo que el sistema del cliente tiene registrado

Hoy esta conciliación se hace manualmente: Jessica extrae el Excel desde su usuario SAP y lo cruza a mano contra su control interno. Esto genera:

- Falta de visibilidad sobre el estado real del inventario
- Diferencias inexplicables al final del mes que se vuelven pérdidas
- Trabajo manual repetitivo de cruce entre el Excel SAP y el control interno
- Imposibilidad de auditar quién tuvo qué material y en qué momento

### 1.4 Objetivo del sistema

Construir un sistema que:

1. **Trace** cada material desde que entra al almacén hasta que se certifica
2. **Concilie automáticamente** los Excels de SAP contra el estado operativo
3. **Documente todos los movimientos** con vales generados automáticamente
4. **Permita auditoría completa** de cualquier material en cualquier momento

---

## 2. Glosario de dominio

Términos que **deben respetarse en código, UI y base de datos** sin traducir ni inventar sinónimos:

| Término | Definición |
|---|---|
| **Material** | SKU de un artículo en el catálogo (ej. código 196786 = "Acc Obt/Pur DN2\" Wel DN8\" 300# TR"). Identificado en SAP con un código numérico. |
| **Centro** | Contrato del cliente en SAP (ej. `12T2` Toluca, `37X2` Metrogas). Un material puede existir en varios centros con stocks independientes. |
| **Almacén** | Ubicación física donde vive el material (Toluca, CDMX). Distinto del concepto SAP de almacén que puede repetirse. |
| **Cuadrilla** | Equipo de trabajo de campo al que se le entrega material para instalación. |
| **Proyecto** | Obra específica donde el material se instala. |
| **Movimiento** | Evento que cambia el estado o ubicación de un material. Es **inmutable**. |
| **Vale** | Documento generado automáticamente por cada movimiento (entrada, salida, préstamo, devolución, traslado, ajuste). |
| **Préstamo interno** | Transferencia temporal de material entre almacenes Alesig (Toluca ↔ CDMX). |
| **Préstamo externo** | Transferencia temporal a una empresa externa (incluyendo competencia). |
| **Traslado SAP** | Reasignación contable iniciada por Naturgy en su SAP que mueve el inventario de un centro a otro (`12T2` ↔ `37X2`). Puede ser puramente contable (el material no se mueve físicamente) o ir acompañado de movimiento físico entre almacenes. Alesig lo descubre al importar el siguiente Excel SAP — Naturgy no notifica directamente. En SAP puede ocurrir en un paso (movimiento inmediato) o en dos pasos con un estado intermedio de "stock en tránsito" donde el material no aparece en ningún centro. |
| **Certificación SAP** | Acto contable por el cual el material instalado se descarga oficialmente del inventario del cliente. |
| **Reasignación** | Cambio de proyecto destino del material **sin** que regrese al almacén. |
| **Conciliación** | Cruce entre un snapshot de SAP y el estado operativo del sistema, identificando diferencias. |
| **Snapshot SAP** | Foto inmutable del estado de inventario tal como SAP lo reporta, en una fecha específica, vía Excel. |
| **PMV** | Precio Medio Variable, costo unitario del material en SAP. |
| **SDC** | Salida Directa Cuadrilla (campo SAP que indica cantidad entregada a cuadrilla sin certificar aún). |

### 2.1 Estados del material (máquina de estados)

Todo material **siempre** debe estar en exactamente uno de estos estados:

1. `EN_ALMACEN` — disponible físicamente en almacén
2. `PENDIENTE_APROBACION` — salida solicitada por almacenista, esperando autorización de supervisor
3. `EN_CUADRILLA` — salió a cuadrilla, no instalado aún
4. `INSTALADO` — instalado en proyecto, pendiente de certificar en SAP
5. `PENDIENTE_SAP` — instalado, en proceso de certificación
6. `CERTIFICADO_SAP` — ciclo cerrado contablemente
7. `PRESTADO_INTERNO` — en otro almacén Alesig (préstamo)
8. `PRESTADO_EXTERNO` — en empresa externa
9. `TRASLADADO` — reasignado vía SAP a otro centro. Estado transitorio: debe resolverse con una `ENTRADA` en el centro destino para que el material retome su ciclo de vida.
10. `DEFECTUOSO` — registrado como dañado, esperando resolución
11. `PENDIENTE_DEVOLUCION` — cuadrilla debe devolver, no lo ha hecho
12. `PERDIDA` — declarado como pérdida definitiva

Las transiciones válidas se detallan en sección 4.4.

---

## 3. Arquitectura técnica

### 3.1 Stack recomendado

| Capa | Tecnología | Justificación |
|---|---|---|
| **Backend** | Express + Node.js + TypeScript | Framework ligero y flexible, amplio ecosistema, desarrollo rápido |
| **Base de datos** | PostgreSQL 16 — Neon (dev), Railway Postgres (prod) | Constraints fuertes, JSONB para metadatos flexibles, triggers para invariantes. Neon free tier para desarrollo sin gestión local, Railway managed para producción |
| **ORM** | Prisma | Type-safe, migraciones versionadas, excelente DX |
| **Frontend** | Next.js + TypeScript | Full-stack TypeScript, SSR/SSG, API routes integradas |
| **Monorepo** | TurboRepo | Optimización de builds, cachés distribuidas, manejo eficiente de dependencias compartidas. Activar remote cache solo cuando el tiempo de build sea un dolor real; pnpm workspaces solas son suficientes en v1 |
| **UI** | Tailwind CSS + shadcn/ui | Componentes accesibles sin reinventar |
| **Notificaciones** | Brevo | Email, SMS y WhatsApp desde una única plataforma |
| **Storage de Excels** | Digital Ocean Spaces | Snapshots SAP se archivan, escalable y económico |
| **Tareas programadas** | INNGest | Cronjobs y tareas asincrónicas de larga duración |
| **Deployment** | Railway | Despliegue simplificado, CI/CD integrado, entornos efímeros |
| **Auth** | JWT + bcrypt (fase 1) | Migrar a Azure AD u OAuth2 en fases futuras si el cliente lo requiere. **Deuda técnica anotada:** implementar rotación de refresh tokens antes de salir a producción (access token 15 min, refresh token 7 días con rotación en cada uso) |
| **Validación** | Zod + TypeScript | Esquemas DTO tipados, validación cliente-servidor |
| **Logs y telemetría** | Winston + OpenTelemetry | Trazabilidad de operaciones |
| **Error monitoring** | Sentry | Captura de excepciones en producción con contexto de request; integrar desde fase 0 para no depurar a ciegas |
| **Tests** | Jest + Supertest (back), Vitest + Testing Library (front) | |
| **Contenedores** | Docker + docker-compose | Desarrollo local reproducible |

### 3.2 Estructura del repositorio (Monorepo TurboRepo)

```
alesig/
├── packages/
│   ├── backend/
│   │   ├── prisma/
│   │   │   ├── migrations/
│   │   │   └── schema.prisma
│   │   ├── src/
│   │   │   ├── index.ts                 # Entry Express
│   │   │   ├── config.ts                # Settings vía variables de entorno + Zod
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── errorHandler.ts
│   │   │   │   └── logging.ts
│   │   │   ├── domain/                  # Modelos de dominio puros
│   │   │   │   ├── material.ts
│   │   │   │   ├── movimiento.ts
│   │   │   │   ├── conciliacion.ts
│   │   │   │   └── estados.ts           # Máquina de estados
│   │   │   ├── infra/                   # Prisma client, repositories
│   │   │   │   ├── prisma.ts
│   │   │   │   └── repositories/
│   │   │   ├── api/                     # Routers Express
│   │   │   │   ├── v1/
│   │   │   │   │   ├── materiales.ts
│   │   │   │   │   ├── movimientos.ts
│   │   │   │   │   ├── conciliacion.ts
│   │   │   │   │   ├── snapshots.ts
│   │   │   │   │   └── auth.ts
│   │   │   ├── schemas/                 # Zod DTOs
│   │   │   ├── services/                # Casos de uso
│   │   │   │   ├── importador_sap.ts    # Pieza crítica
│   │   │   │   ├── motor_conciliacion.ts
│   │   │   │   └── motor_movimientos.ts
│   │   │   └── workers/                 # Tareas asincrónicas (INNGest)
│   │   ├── tests/
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   └── fixtures/                # Excels SAP de ejemplo
│   │   ├── package.json
│   │   ├── Dockerfile
│   │   └── .env.example
│   └── frontend/
│       ├── src/
│       │   ├── app/
│       │   │   ├── layout.tsx
│       │   │   ├── page.tsx
│       │   │   └── (routes)/
│       │   │       ├── dashboard/page.tsx
│       │   │       ├── materiales/page.tsx
│       │   │       ├── conciliacion/page.tsx
│       │   │       ├── snapshots/page.tsx
│       │   │       └── conteo-fisico/page.tsx
│       │   ├── api/                     # API routes Next.js + tipos
│       │   ├── components/
│       │   │   ├── ui/                  # shadcn/ui
│       │   │   ├── tables/
│       │   │   └── forms/
│       │   ├── hooks/
│       │   ├── lib/
│       │   └── styles/
│       ├── package.json
│       ├── next.config.js
│       ├── tsconfig.json
│       ├── Dockerfile
│       └── .env.example
├── packages/shared/
│   ├── src/
│   │   ├── types/
│   │   │   ├── material.ts
│   │   │   ├── movimiento.ts
│   │   │   └── schemas.ts
│   │   └── utils/
│   ├── package.json
│   └── tsconfig.json
├── docs/
│   ├── adr/                            # Architecture Decision Records
│   │   ├── 0001-stack-tecnologico.md
│   │   ├── 0002-granularidad-material.md
│   │   └── 0003-modelo-snapshot-inmutable.md
│   ├── api/                            # OpenAPI exportado
│   └── flujos/                         # Diagramas de flujo
├── .gitignore
├── docker-compose.yml
├── docker-compose.dev.yml
├── turbo.json
├── package.json
├── pnpm-workspace.yaml
├── README.md
└── SPEC_SISTEMA_ALESIG.md              # Este documento
```

### 3.3 Principios arquitectónicos (no negociables)

1. **Eventos inmutables**: la tabla `movimientos` es append-only. Nunca se actualiza ni se elimina. Errores se corrigen con movimientos compensatorios que referencian el original.

2. **Estado derivado**: el estado actual de un material es una columna *caché*, pero la verdad vive en el historial de movimientos. Debe existir un job/comando que recalcule el estado desde eventos para auditoría.

3. **Máquina de estados infranqueable**: ninguna mutación de estado pasa fuera del `motor_movimientos`. No hay `UPDATE material SET estado = ...` libre en el código.

4. **Snapshots SAP inmutables**: cada Excel importado genera un snapshot con sus líneas. Nunca se modifica un snapshot existente. Si Jessica genera una nueva extracción corregida, se importa como un snapshot nuevo independiente.

5. **Importador defensivo**: el parser de Excel valida esquema, tipos, rangos y duplicados antes de persistir. Cero confianza en el archivo de entrada.

6. **Identificadores SAP respetando la tripleta**: la unicidad de un material en SAP es `(codigo_material, centro, almacen)`, no solo `codigo_material`. Modelar correctamente desde el inicio.

7. **ADRs desde día 1**: cada decisión técnica significativa se documenta en `docs/adr/`. Formato Michael Nygard.

---

### 3.4 Estrategia de bases de datos: Neon (desarrollo) + Railway (producción)

**Desarrollo:**
- **Neon free tier** (`neon.tech`)
  - Cada dev obtiene su propia rama de base de datos (branching)
  - Zero management, autoscaling, backups automáticos
  - Connection string en `.env.local`: `postgresql://user:pass@ep-xxx.neon.tech/neon`
  - Reseteable sin afectar a otros devs ni producción

**Producción:**
- **Railway PostgreSQL managed**
  - Integrado con Railway (mismo dashboard, billing único)
  - Sin cold starts (crítico para operación real)
  - Backups automáticos, failover, encryption
  - Connection string en Railway environment variables

**Tests / CI:**
- GitHub Actions usa contenedor temporal de Postgres (Docker image `postgres:16`)
- Migración con Prisma antes de correr tests

**Ventajas de esta estrategia:**
1. Desarrollo rápido y gratuito (Neon free tier)
2. Producción confiable y sin sorpresas (Railway managed)
3. Cero configuración local de Docker Postgres para devs
4. Cada dev es aislado; no interfiere con otros ni con tests

---

## 4. Modelo de datos

### 4.1 Convenciones

- Todas las tablas tienen `id BIGSERIAL PRIMARY KEY`
- Todas tienen `created_at TIMESTAMPTZ DEFAULT NOW()` y `updated_at TIMESTAMPTZ` (excepto tablas append-only que solo tienen `created_at`)
- Soft delete con `deleted_at TIMESTAMPTZ NULL` cuando aplica (NO en tablas append-only)
- Nombres de tablas en español, plural, snake_case
- Claves foráneas con sufijo `_id`
- Enums de PostgreSQL para estados (no strings libres)

### 4.2 Entidades nucleares

#### 4.2.1 Catálogo

```sql
-- Catálogo maestro de materiales (compartido entre centros)
CREATE TABLE materiales (
    id BIGSERIAL PRIMARY KEY,
    codigo_sap VARCHAR(20) NOT NULL UNIQUE,        -- ej. "196786"
    descripcion TEXT NOT NULL,
    grupo_articulo VARCHAR(20),                     -- ej. "15A040000"
    desc_grupo VARCHAR(100),                        -- ej. "Válv. Acc. Red pol."
    unidad_base VARCHAR(10) NOT NULL DEFAULT 'PZA',
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

-- Centros SAP (contratos del cliente)
CREATE TABLE centros (
    id BIGSERIAL PRIMARY KEY,
    codigo VARCHAR(10) NOT NULL UNIQUE,             -- "12T2", "37X2"
    descripcion VARCHAR(100) NOT NULL,              -- "Toluca Contrata"
    cliente VARCHAR(100),                           -- "Metrogas"
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Almacenes físicos de Alesig
CREATE TABLE almacenes (
    id BIGSERIAL PRIMARY KEY,
    codigo VARCHAR(20) NOT NULL UNIQUE,             -- "ALESIG_TOLUCA", "ALESIG_CDMX"
    codigo_sap VARCHAR(10),                         -- "1DAL"
    descripcion VARCHAR(100) NOT NULL,
    ubicacion_fisica TEXT,
    centro_default_id BIGINT REFERENCES centros(id),
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Material × Centro × Almacén (la unidad de inventario real)
CREATE TABLE material_centro_almacen (
    id BIGSERIAL PRIMARY KEY,
    material_id BIGINT NOT NULL REFERENCES materiales(id),
    centro_id BIGINT NOT NULL REFERENCES centros(id),
    almacen_id BIGINT NOT NULL REFERENCES almacenes(id),
    stock_actual NUMERIC(15,4) NOT NULL DEFAULT 0,  -- existencias físicas en sistema
    pmv NUMERIC(15,4),                              -- precio medio variable SAP
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ,
    UNIQUE (material_id, centro_id, almacen_id)
);
```

#### 4.2.2 Proyectos y cuadrillas

```sql
CREATE TYPE tipo_proyecto AS ENUM ('CONSTRUCCION', 'MANTENIMIENTO');

CREATE TABLE proyectos (
    id BIGSERIAL PRIMARY KEY,
    codigo VARCHAR(50) NOT NULL UNIQUE,
    nombre VARCHAR(200) NOT NULL,
    tipo tipo_proyecto NOT NULL DEFAULT 'CONSTRUCCION',
    centro_id BIGINT NOT NULL REFERENCES centros(id),
    estado VARCHAR(20) NOT NULL DEFAULT 'ACTIVO',   -- ACTIVO, CERRADO, SUSPENDIDO
    fecha_inicio DATE,
    fecha_cierre_estimada DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

CREATE TABLE cuadrillas (
    id BIGSERIAL PRIMARY KEY,
    codigo VARCHAR(50) NOT NULL UNIQUE,
    nombre VARCHAR(200) NOT NULL,
    supervisor_nombre VARCHAR(200),
    supervisor_telefono VARCHAR(20),
    almacen_default_id BIGINT REFERENCES almacenes(id),
    activa BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

CREATE TABLE empresas_externas (
    id BIGSERIAL PRIMARY KEY,
    rfc VARCHAR(13) UNIQUE,
    razon_social VARCHAR(200) NOT NULL,
    contacto VARCHAR(200),
    es_competencia BOOLEAN NOT NULL DEFAULT FALSE,
    activa BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### 4.2.3 Usuarios y roles

```sql
CREATE TYPE rol_usuario AS ENUM (
    'ADMIN', 'DIRECCION', 'SUPERVISOR',
    'ALMACENISTA_CDMX', 'ALMACENISTA_TOLUCA',
    'AUDITOR_CLIENTE', 'CAPTURISTA'
);

CREATE TABLE usuarios (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(200) NOT NULL UNIQUE,
    nombre VARCHAR(200) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    rol rol_usuario NOT NULL,
    almacen_id BIGINT REFERENCES almacenes(id),    -- restringe scope de almacenistas
    activo BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);
```

#### 4.2.4 Movimientos (append-only, corazón del sistema)

```sql
CREATE TYPE tipo_movimiento AS ENUM (
    'ENTRADA',
    'SOLICITUD_SALIDA',          -- almacenista solicita salida, queda PENDIENTE_APROBACION
    'APROBACION_SALIDA',         -- supervisor aprueba → EN_CUADRILLA
    'RECHAZO_SALIDA',            -- supervisor rechaza → regresa EN_ALMACEN
    'SALIDA_CUADRILLA',
    'DEVOLUCION_CUADRILLA',
    'PRESTAMO_INTERNO',
    'PRESTAMO_EXTERNO',
    'DEVOLUCION_PRESTAMO',
    'TRASLADO_SAP',
    'INSTALACION',
    'CERTIFICACION_SAP',
    'REASIGNACION_PROYECTO',
    'REGISTRO_DEFECTUOSO',
    'AJUSTE_INVENTARIO',
    'DECLARACION_PERDIDA',
    'CORRECCION'                                    -- compensa un movimiento previo
);

CREATE TYPE estado_material AS ENUM (
    'EN_ALMACEN', 'PENDIENTE_APROBACION', 'EN_CUADRILLA', 'INSTALADO',
    'PENDIENTE_SAP', 'CERTIFICADO_SAP',
    'PRESTADO_INTERNO', 'PRESTADO_EXTERNO',
    'TRASLADADO', 'DEFECTUOSO',
    'PENDIENTE_DEVOLUCION', 'PERDIDA'
);

CREATE TABLE movimientos (
    id BIGSERIAL PRIMARY KEY,
    folio VARCHAR(30) NOT NULL UNIQUE,              -- generado: MOV-2026-000001
    tipo tipo_movimiento NOT NULL,
    fecha_movimiento TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    material_id BIGINT NOT NULL REFERENCES materiales(id),
    cantidad NUMERIC(15,4) NOT NULL CHECK (cantidad > 0),

    -- Origen
    centro_origen_id BIGINT REFERENCES centros(id),
    almacen_origen_id BIGINT REFERENCES almacenes(id),
    cuadrilla_origen_id BIGINT REFERENCES cuadrillas(id),
    proyecto_origen_id BIGINT REFERENCES proyectos(id),
    empresa_externa_origen_id BIGINT REFERENCES empresas_externas(id),

    -- Destino
    centro_destino_id BIGINT REFERENCES centros(id),
    almacen_destino_id BIGINT REFERENCES almacenes(id),
    cuadrilla_destino_id BIGINT REFERENCES cuadrillas(id),
    proyecto_destino_id BIGINT REFERENCES proyectos(id),
    empresa_externa_destino_id BIGINT REFERENCES empresas_externas(id),

    -- Transición de estado
    estado_anterior estado_material,
    estado_nuevo estado_material NOT NULL,

    -- Metadatos
    usuario_id BIGINT NOT NULL REFERENCES usuarios(id),
    observaciones TEXT,
    referencia_externa VARCHAR(100),                -- folio SAP, ticket, etc.
    movimiento_compensado_id BIGINT REFERENCES movimientos(id),  -- solo para CORRECCION

    -- Anexos
    vale_pdf_path TEXT,
    evidencia_paths JSONB DEFAULT '[]',             -- ["s3://...", ...]

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    -- NO updated_at, NO deleted_at: append-only
);

CREATE INDEX idx_movimientos_material ON movimientos(material_id, fecha_movimiento DESC);
CREATE INDEX idx_movimientos_tipo_fecha ON movimientos(tipo, fecha_movimiento DESC);
CREATE INDEX idx_movimientos_usuario ON movimientos(usuario_id);
```

#### 4.2.5 Snapshots SAP (inmutables)

```sql
CREATE TABLE snapshots_sap (
    id BIGSERIAL PRIMARY KEY,
    fecha_extraccion DATE NOT NULL,                 -- la fecha que dice el Excel
    fecha_importacion TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    archivo_nombre VARCHAR(255) NOT NULL,
    archivo_hash VARCHAR(64) NOT NULL,              -- SHA-256 para dedup
    archivo_storage_path TEXT NOT NULL,             -- ruta al archivo original
    almacen_id BIGINT NOT NULL REFERENCES almacenes(id),
    usuario_id BIGINT NOT NULL REFERENCES usuarios(id),
    total_lineas INTEGER NOT NULL,
    estado VARCHAR(20) NOT NULL DEFAULT 'IMPORTADO', -- IMPORTADO, ANULADO
    observaciones TEXT,
    metadata_extraccion JSONB,                      -- headers, validaciones
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (archivo_hash)                           -- evita reimportación accidental
);

CREATE TABLE lineas_snapshot_sap (
    id BIGSERIAL PRIMARY KEY,
    snapshot_id BIGINT NOT NULL REFERENCES snapshots_sap(id) ON DELETE RESTRICT,
    material_id BIGINT NOT NULL REFERENCES materiales(id),
    centro_id BIGINT NOT NULL REFERENCES centros(id),
    almacen_codigo_sap VARCHAR(10) NOT NULL,

    -- Campos clave del Excel SAP
    existencias NUMERIC(15,4) NOT NULL DEFAULT 0,
    pend_recibir NUMERIC(15,4) NOT NULL DEFAULT 0,
    pend_consignar NUMERIC(15,4) NOT NULL DEFAULT 0,
    pend_devolver NUMERIC(15,4) NOT NULL DEFAULT 0,
    pend_autorizar NUMERIC(15,4) NOT NULL DEFAULT 0,
    comp_curso NUMERIC(15,4) NOT NULL DEFAULT 0,
    comp_total NUMERIC(15,4) NOT NULL DEFAULT 0,
    sdc NUMERIC(15,4) NOT NULL DEFAULT 0,
    sdc2 NUMERIC(15,4) NOT NULL DEFAULT 0,
    valor_sdc NUMERIC(15,4) NOT NULL DEFAULT 0,
    valor_sdc2 NUMERIC(15,4) NOT NULL DEFAULT 0,
    cert_en_curso NUMERIC(15,4) NOT NULL DEFAULT 0,
    docum_borrado NUMERIC(15,4) NOT NULL DEFAULT 0,
    pmv NUMERIC(15,4),
    nivel_servicio NUMERIC(5,2),

    -- Resto del row guardado tal cual por si se necesita
    raw_data JSONB NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (snapshot_id, material_id, centro_id)
);

CREATE INDEX idx_lineas_snapshot_material ON lineas_snapshot_sap(material_id, snapshot_id);
```

#### 4.2.6 Conteo físico y conciliación

```sql
CREATE TABLE conteos_fisicos (
    id BIGSERIAL PRIMARY KEY,
    fecha_conteo DATE NOT NULL,
    almacen_id BIGINT NOT NULL REFERENCES almacenes(id),
    usuario_id BIGINT NOT NULL REFERENCES usuarios(id),
    estado VARCHAR(20) NOT NULL DEFAULT 'EN_CURSO', -- EN_CURSO, COMPLETADO, ANULADO
    observaciones TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

CREATE TABLE lineas_conteo_fisico (
    id BIGSERIAL PRIMARY KEY,
    conteo_id BIGINT NOT NULL REFERENCES conteos_fisicos(id),
    material_id BIGINT NOT NULL REFERENCES materiales(id),
    cantidad_contada NUMERIC(15,4) NOT NULL,
    observacion TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (conteo_id, material_id)
);

CREATE TYPE clasificacion_diferencia AS ENUM (
    'CUADRA',
    'JUSTIFICADA',          -- diferencia explicada por movimientos pendientes
    'PENDIENTE_INVESTIGAR',
    'PERDIDA_CONFIRMADA',
    'AJUSTE_REQUERIDO'
);

CREATE TABLE conciliaciones (
    id BIGSERIAL PRIMARY KEY,
    folio VARCHAR(30) NOT NULL UNIQUE,              -- CONC-2026-000001
    snapshot_id BIGINT NOT NULL REFERENCES snapshots_sap(id),
    conteo_fisico_id BIGINT REFERENCES conteos_fisicos(id),
    fecha_conciliacion TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    usuario_id BIGINT NOT NULL REFERENCES usuarios(id),
    estado VARCHAR(20) NOT NULL DEFAULT 'EN_CURSO',
    total_materiales INTEGER NOT NULL DEFAULT 0,
    total_cuadran INTEGER NOT NULL DEFAULT 0,
    total_pendientes INTEGER NOT NULL DEFAULT 0,
    total_perdidas INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    closed_at TIMESTAMPTZ
);

CREATE TABLE lineas_conciliacion (
    id BIGSERIAL PRIMARY KEY,
    conciliacion_id BIGINT NOT NULL REFERENCES conciliaciones(id),
    material_id BIGINT NOT NULL REFERENCES materiales(id),
    centro_id BIGINT NOT NULL REFERENCES centros(id),

    -- Vista SAP
    sap_existencias NUMERIC(15,4) NOT NULL DEFAULT 0,
    sap_comprometido NUMERIC(15,4) NOT NULL DEFAULT 0,
    sap_certificandose NUMERIC(15,4) NOT NULL DEFAULT 0,

    -- Vista sistema/físico
    sistema_stock NUMERIC(15,4) NOT NULL DEFAULT 0,
    fisico_contado NUMERIC(15,4),

    -- Resultado
    diferencia NUMERIC(15,4) NOT NULL DEFAULT 0,
    clasificacion clasificacion_diferencia NOT NULL DEFAULT 'PENDIENTE_INVESTIGAR',
    justificacion TEXT,
    movimientos_relacionados JSONB DEFAULT '[]',    -- IDs de movimientos que explican

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_lineas_conc_clasif ON lineas_conciliacion(conciliacion_id, clasificacion);
```

### 4.3 Estrategia de stock en el backend

No se usan vistas materializadas ni triggers de Postgres. Toda la lógica de stock vive en la capa de servicio de TypeScript, lo que la hace testeable, debuggeable y consistente con el principio de que ninguna mutación pasa fuera de `motor_movimientos`.

#### 4.3.1 Actualización del caché `stock_actual`

El campo `stock_actual` en `material_centro_almacen` es un caché derivado. `motor_movimientos` lo actualiza **dentro de la misma transacción** en la que inserta el movimiento:

```typescript
// services/motor_movimientos.ts (esquema simplificado)
async function registrarMovimiento(input: MovimientoInput, tx: PrismaTransaction) {
  // 1. Validar transición de estado (máquina de estados)
  // 2. Insertar en movimientos (append-only)
  const mov = await tx.movimientos.create({ data: { ...input } });

  // 3. Actualizar stock — dos piernas simétricas
  if (input.almacenOrigenId) {
    await tx.materialCentroAlmacen.update({
      where: { materialId_centroId_almacenId: {
        materialId: input.materialId,
        centroId:   input.centroOrigenId,
        almacenId:  input.almacenOrigenId,
      }},
      data: { stockActual: { decrement: input.cantidad } },
    });
  }

  if (input.almacenDestinoId) {
    await tx.materialCentroAlmacen.upsert({
      where: { materialId_centroId_almacenId: {
        materialId: input.materialId,
        centroId:   input.centroDestinoId,
        almacenId:  input.almacenDestinoId,
      }},
      update: { stockActual: { increment: input.cantidad } },
      create: { materialId: input.materialId, centroId: input.centroDestinoId,
                almacenId: input.almacenDestinoId, stockActual: input.cantidad },
    });
  }

  return mov;
}
```

Si la transacción falla por cualquier motivo, el INSERT del movimiento y los UPDATE de stock se revierten juntos. El caché nunca queda inconsistente.

#### 4.3.2 Servicio de auditoría de stock

Para verificar que el caché coincide con el historial de movimientos existe `StockAuditService`:

```typescript
// services/stock_audit.ts
async function calcularStockDesdEventos(
  materialId: number, centroId: number, almacenId: number
): Promise<number> {
  const result = await prisma.$queryRaw<[{ stock: number }]>`
    SELECT
      COALESCE(SUM(
        CASE
          WHEN almacen_destino_id = ${almacenId}
           AND centro_destino_id  = ${centroId} THEN cantidad
          WHEN almacen_origen_id  = ${almacenId}
           AND centro_origen_id   = ${centroId} THEN -cantidad
          ELSE 0
        END
      ), 0) AS stock
    FROM movimientos
    WHERE material_id = ${materialId}
      AND (
        (almacen_destino_id = ${almacenId} AND centro_destino_id = ${centroId})
        OR
        (almacen_origen_id  = ${almacenId} AND centro_origen_id  = ${centroId})
      )
  `;
  return Number(result[0].stock);
}

async function verificarConsistenciaTotal(): Promise<DiscrepancyReport> {
  // Compara stock_actual (caché) vs calcularStockDesdEventos para cada fila
  // de material_centro_almacen. Retorna las discrepancias encontradas.
}
```

El comando `pnpm stock:verify` ejecuta `verificarConsistenciaTotal()` y reporta discrepancias. Debe correr en CI antes de cada deploy a producción y puede ejecutarse manualmente ante cualquier duda operativa.

### 4.4 Máquina de estados — transiciones válidas

| Estado origen | Movimiento | Estado destino | Rol que ejecuta |
|---|---|---|---|
| (ninguno) | `ENTRADA` | `EN_ALMACEN` | ALMACENISTA, SUPERVISOR |
| `EN_ALMACEN` | `SOLICITUD_SALIDA` | `PENDIENTE_APROBACION` | ALMACENISTA |
| `PENDIENTE_APROBACION` | `APROBACION_SALIDA` | `EN_CUADRILLA` | SUPERVISOR, ADMIN |
| `PENDIENTE_APROBACION` | `RECHAZO_SALIDA` | `EN_ALMACEN` | SUPERVISOR, ADMIN |
| `EN_ALMACEN` | `PRESTAMO_INTERNO` | `PRESTADO_INTERNO` | SUPERVISOR, ADMIN |
| `EN_ALMACEN` | `PRESTAMO_EXTERNO` | `PRESTADO_EXTERNO` | SUPERVISOR, ADMIN |
| `EN_ALMACEN` | `TRASLADO_SAP` | `TRASLADADO` | SUPERVISOR, ADMIN |
| `TRASLADADO` | `ENTRADA` (nuevo centro) | `EN_ALMACEN` (nuevo centro) | SUPERVISOR, ADMIN |
| `EN_CUADRILLA` | `DEVOLUCION_CUADRILLA` | `EN_ALMACEN` | ALMACENISTA, SUPERVISOR |
| `EN_CUADRILLA` | `INSTALACION` | `INSTALADO` | ALMACENISTA, SUPERVISOR |
| `EN_CUADRILLA` | `REASIGNACION_PROYECTO` | `EN_CUADRILLA` (otro proyecto) | SUPERVISOR, ADMIN |
| `EN_CUADRILLA` | `REGISTRO_DEFECTUOSO` | `DEFECTUOSO` | ALMACENISTA, SUPERVISOR |
| `INSTALADO` | `CERTIFICACION_SAP` | `CERTIFICADO_SAP` | SUPERVISOR, ADMIN |
| `PRESTADO_INTERNO` | `DEVOLUCION_PRESTAMO` | `EN_ALMACEN` | ALMACENISTA, SUPERVISOR |
| `PRESTADO_INTERNO` | `CERTIFICACION_SAP` | `CERTIFICADO_SAP` | SUPERVISOR, ADMIN |
| `PRESTADO_EXTERNO` | `DEVOLUCION_PRESTAMO` | `EN_ALMACEN` | ALMACENISTA, SUPERVISOR |
| `PRESTADO_EXTERNO` | `TRASLADO_SAP` | `TRASLADADO` | SUPERVISOR, ADMIN |
| `DEFECTUOSO` | `AJUSTE_INVENTARIO` | (varios destinos) | SUPERVISOR, ADMIN |
| cualquiera | `CORRECCION` | (revierte el compensado) | SUPERVISOR, ADMIN |
| cualquiera | `DECLARACION_PERDIDA` | `PERDIDA` | ADMIN, DIRECCION |

El motor valida cada transición. Si no está en la tabla, rechaza con HTTP 422.

---

## 5. Importador SAP — la pieza crítica

### 5.1 Estructura de los archivos Excel

Cada Excel tiene **44 columnas** con encabezados en español (con acentos, ° y caracteres especiales). Una sola hoja llamada `Hoja1`. Los campos clave a extraer:

| Columna Excel | Tipo | Notas |
|---|---|---|
| `Material` | int o float→int | Código SAP. Puede venir como `196786.0` |
| `Desc. Material` | string | |
| `Centro` | string | "12T2", "37X2" |
| `Desc. Centro` | string | "Toluca Contrata", "Metrogas Contrata" |
| `Almacén` | string | "1DAL" — OJO: puede repetirse entre centros |
| `Desc. Almacén` | string | "ALESIG Toluca" vs "CONS.EMPR.ALESIG" |
| `Grup.Arti.` | string | "15A040000" |
| `Desc del grupo` | string | "Válv. Acc. Red pol." |
| `Existencias` | numeric | **COLUMNA PRIMARIA DE CONCILIACIÓN** — confirmado por Jessica (15/05/2026). Es el campo contra el que se compara el stock operativo de Alesig. |
| `Pend. Recibir` | numeric | |
| `Pend. Consignar` | numeric | |
| `Pend. Devolver` | numeric | |
| `Pend. Autorizar` | numeric | secundario — cuello de botella, útil para diagnóstico |
| `Comp. Curso` | numeric | secundario |
| `Comp. Total` | numeric | |
| `SDC` | numeric | secundario — Salida Directa Cuadrilla, útil para diagnóstico |
| `SDC2` | numeric | |
| `Valor SDC` | numeric | |
| `Valor SDC2` | numeric | |
| `Certificación en curso` | numeric | secundario — útil para diagnóstico de materiales en proceso de cierre |
| `Docum. Borrado` | numeric | |
| `PMV` | numeric | precio medio variable |
| `Nivel servicio` | numeric | |

Las demás 21 columnas se guardan en `raw_data` JSONB por completitud, pero no se usan activamente.

### 5.2 Pipeline del importador

```
[Upload Excel] → [Validar formato] → [Calcular hash] → [Verificar dedup]
              → [Parsear con exceljs] → [Validar esquema columnas]
              → [Normalizar tipos] → [Resolver foreign keys (material, centro)]
              → [Detectar materiales/centros nuevos] → [Crear/actualizar catálogo]
              → [Crear snapshot] → [Insertar líneas en batch]
              → [Persistir Excel original] → [Reporte de importación]
```

### 5.3 Reglas de validación

1. El archivo debe ser `.xlsx`, no `.xls` ni `.csv`
2. Debe tener exactamente una hoja
3. La primera fila debe contener los 44 encabezados esperados (lista hardcoded, configurable vía settings)
4. Si un encabezado falta o cambia, la importación **falla** y reporta qué columna está mal — no procesa parcial
5. Si `Material` es NULL en una fila, esa fila se rechaza con warning (pero la importación continúa)
6. Hash SHA-256 del archivo: si ya existe el mismo archivo, rechaza con error claro
7. Materiales nuevos (no en catálogo) se crean automáticamente con flag `auto_creado=true` para revisión posterior
8. Centros nuevos NO se crean automáticamente — falla la importación pidiendo configuración manual

### 5.4 Caso de prueba con archivos reales

Los archivos `EXTRACCION_MATERIALES_TOLUCA_20_DE_ABRIL.xlsx` y `EXTRACCION_MATERIALES_CDMX_20_DE_ABRIL.xlsx` deben usarse como **fixtures** en tests. Esperado:

- Toluca: 107 materiales, centro `12T2`
- CDMX: 126 materiales, centro `37X2`
- 91 materiales comunes entre ambos (mismo `codigo_sap` distinto `centro_id`)

---

## 6. API REST

### 6.1 Convenciones

- Prefijo `/api/v1`
- Auth con JWT en header `Authorization: Bearer <token>`
- Respuestas en JSON con envelope:
  ```json
  { "data": ..., "meta": { "page": 1, "total": 250 } }
  ```
- Errores con formato:
  ```json
  { "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [...] } }
  ```
- Paginación con `?page=N&size=M` (default size 50, max 200)
- Filtros con query params explícitos, no DSL
- Fechas en ISO 8601 con timezone

### 6.2 Endpoints del MVP

#### Auth
- `POST /api/v1/auth/login` — email + password → JWT
- `POST /api/v1/auth/refresh` — refresca token
- `GET /api/v1/auth/me` — usuario actual

#### Catálogo
- `GET /api/v1/materiales` — listado paginado, filtros: `q`, `grupo`, `centro_id`
- `GET /api/v1/materiales/{id}` — detalle
- `GET /api/v1/materiales/{id}/historial` — movimientos
- `GET /api/v1/centros` — listado
- `GET /api/v1/almacenes` — listado

#### Snapshots SAP
- `POST /api/v1/snapshots/upload` — sube Excel (multipart/form-data) → crea snapshot async. Límite: **10 MB** máximo (rechaza con 413 antes de procesar). Responde inmediatamente con `{ snapshotId, estado: "PROCESANDO" }`.
- `GET /api/v1/snapshots/{id}/estado` — polling del estado de importación (`PROCESANDO` | `IMPORTADO` | `ERROR`). El frontend hace polling cada 2 segundos hasta estado final. Retorna también `{ totalLineas, lineasConWarning, materialesNuevos }` al completar.
- `GET /api/v1/snapshots` — listado paginado, filtros: `almacen_id`, `desde`, `hasta`
- `GET /api/v1/snapshots/{id}` — detalle con resumen
- `GET /api/v1/snapshots/{id}/lineas` — líneas paginadas
- `GET /api/v1/snapshots/{id}/download` — descarga Excel original

#### Conteos físicos
- `POST /api/v1/conteos` — inicia un conteo
- `POST /api/v1/conteos/{id}/lineas` — agrega línea (material + cantidad)
- `PATCH /api/v1/conteos/{id}/completar`
- `GET /api/v1/conteos/{id}`

#### Conciliación
- `POST /api/v1/conciliaciones` — crea cruzando snapshot + (opcional) conteo físico
- `GET /api/v1/conciliaciones/{id}/lineas` — listado filtrable por `clasificacion`
- `PATCH /api/v1/conciliaciones/{id}/lineas/{linea_id}` — clasifica y justifica una diferencia
- `GET /api/v1/conciliaciones/{id}/reporte.xlsx` — exporta resultado

#### Movimientos (fase 2+)
- `POST /api/v1/movimientos` — body: `{tipo, material_id, cantidad, origen, destino, ...}`
- `GET /api/v1/movimientos` — filtros: `material_id`, `tipo`, `desde`, `hasta`, `usuario_id`
- `POST /api/v1/movimientos/{id}/corregir` — emite movimiento compensatorio

#### Aprobaciones de salida (fase 2+)
- `GET /api/v1/aprobaciones/pendientes` — lista de solicitudes de salida pendientes, filtrable por `almacen_id`. Solo accesible para SUPERVISOR y ADMIN.
- `POST /api/v1/aprobaciones/{movimiento_id}/aprobar` — emite `APROBACION_SALIDA`, material pasa a `EN_CUADRILLA`. Dispara notificación Brevo al almacenista solicitante.
- `POST /api/v1/aprobaciones/{movimiento_id}/rechazar` — body: `{motivo}`. Emite `RECHAZO_SALIDA`, material regresa a `EN_ALMACEN`. Notifica al almacenista.

---

## 7. Frontend — UX clave

### 7.1 Pantallas del MVP

#### 7.1.1 Dashboard
- Tarjetas KPI: total snapshots este mes, conciliaciones pendientes, diferencias sin justificar, materiales con cert en curso
- Gráfica: evolución de diferencias mes a mes
- Tabla de últimos 10 snapshots con estado

#### 7.1.2 Subir snapshot SAP
- Drag-and-drop de archivo Excel
- Selección de almacén destino del snapshot
- Vista previa de los primeros 10 renglones antes de confirmar
- Indicador de progreso durante la importación
- Reporte final: total filas, filas con warning, materiales nuevos detectados

#### 7.1.3 Conciliación
- Selector: snapshot SAP + (opcional) conteo físico
- Tabla con cuatro buckets (cuadran / justificadas / pendientes / pérdidas confirmadas)
- Filtros: por familia, por monto de diferencia, por almacén
- Por cada línea con diferencia: panel lateral para clasificar y justificar
- Botón "Sugerir justificación": busca movimientos no conciliados que podrían explicar la diferencia
- Export a Excel del reporte final

#### 7.1.4 Catálogo de materiales
- Tabla con búsqueda por código o descripción
- Filtros por grupo, centro, almacén
- Drill-down: clic en un material → historial completo de movimientos y snapshots

#### 7.1.5 Conteo físico
- Vista de captura tipo "lector de código": el almacenista escanea/teclea código SAP, ingresa cantidad
- Lista en vivo de lo capturado con totales por familia
- Al cerrar el conteo, genera reporte de diferencias automáticas vs último snapshot

#### 7.1.6 Cola de aprobaciones (supervisores)

- Pantalla exclusiva para roles SUPERVISOR y ADMIN
- Lista de solicitudes de salida pendientes con: material, cantidad, cuadrilla destino, almacenista solicitante, hora de solicitud
- Botones de aprobar / rechazar directamente en la lista (sin drill-down para casos simples)
- Badge en el nav con el conteo de pendientes, actualizado cada 30 segundos (polling)
- Notificación push opcional en el navegador cuando llega una solicitud nueva

### 7.2 Reglas de UX no negociables

- Idioma: **español de México** en toda la UI
- **Responsive mobile-first**: el sistema debe funcionar correctamente en teléfonos (mínimo 375 px de ancho) y tablets, además de escritorio. Los almacenistas operan desde el piso del almacén con el celular. Los supervisores pueden aprobar desde cualquier dispositivo. Diseñar primero para móvil, luego adaptar a escritorio.
- **Táctil**: botones y áreas de toque con mínimo 44×44 px. Sin hover-only interactions.
- Formato de cantidades: separador de miles con coma, decimales con punto, máximo 4 decimales
- Fechas: `DD/MM/AAAA` para display, ISO en API
- Toda tabla con más de 50 filas: paginada, con búsqueda, con filtros. En móvil, preferir cards apiladas sobre tablas horizontales.
- Toda acción destructiva (anular snapshot, declarar pérdida): confirmación de doble toque con captura del motivo
- Toda operación de más de 2 segundos: loading state explícito, no spinner mudo

---

## 8. Plan de implementación por fases

### Fase 0 — Setup (semana 1)
- Monorepo TurboRepo con packages backend y frontend
- Configuración de Neon (free tier) para desarrollo: cada dev obtiene su propia rama de base de datos
- Docker compose con Postgres local opcional (para CI/tests)
- Estructura de carpetas inicial
- CI básico (lint + test)
- Primer ADR: stack tecnológico y estrategia de bases de datos (Neon dev → Railway prod)
- Migraciones iniciales Prisma: tablas de catálogo (materiales, centros, almacenes, material_centro_almacen), proyectos, cuadrillas, usuarios con roles y seed data
- Endpoint de health-check y login (JWT) funcional
- README con instrucciones de arranque local (Neon connection string o docker-compose para tests)

**Entregable:** un dev puede clonar, correr `docker compose up` y hacer login con un usuario seed.

> **Orden recomendado dentro de Fase 0:** antes de correr las primeras migraciones Prisma, escribir el parser de Excel contra los fixtures reales (`EXTRACCION_MATERIALES_TOLUCA_*.xlsx`, `EXTRACCION_MATERIALES_CDMX_*.xlsx`) como script standalone. Los 44 encabezados con acentos, los códigos SAP que vienen como `196786.0` (float), y las columnas con caracteres especiales pueden sorprender. Es mejor descubrir ese comportamiento en 50 líneas de código antes de tener 15 tablas migrando.

### Fase 0.5 — Carga inicial de datos (antes del go-live)

El inventario existente se carga **manualmente** al sistema antes de operar. Este proceso ocurre una sola vez y define el estado inicial desde el que el sistema toma control.

**Proceso:**
1. Jessica (o el almacenista responsable) realiza un **conteo físico completo** de cada almacén el día acordado como fecha de corte.
2. Ese conteo se captura en el sistema usando la pantalla de conteo físico (fase 2).
3. Cada línea del conteo genera un movimiento de tipo `ENTRADA` con `referencia_externa = "CARGA_INICIAL"` y la fecha de corte como `fecha_movimiento`.
4. El Excel SAP más reciente se importa como primer snapshot — sirve como punto de comparación inmediata.
5. Se corre la primera conciliación entre el snapshot SAP y el conteo físico de carga inicial para detectar diferencias desde el arranque.

**Restricción:** no se captura historial previo al go-live. El sistema parte de cero; los movimientos anteriores existen solo en los registros de Jessica fuera del sistema.

### Fase 1 — Importador SAP + conciliación básica (semanas 2-4)
- Modelo completo de snapshots y líneas
- Endpoint de upload con validación de esquema
- Parser robusto con tests usando los Excels reales como fixtures
- Pantalla de upload en frontend
- Pantalla de listado de snapshots
- Endpoint y pantalla de detalle de snapshot (con paginación de líneas)
- Conciliación inicial: comparar dos snapshots del mismo almacén entre dos fechas
- ADR sobre granularidad (material vs lote) y modelo inmutable de snapshots

**Entregable:** Jessica puede subir el Excel de abril y el de mayo, ver diferencias.

### Fase 2 — Conteo físico + conciliación completa (semanas 5-6)
- Modelo y endpoints de conteo físico
- Pantalla de captura optimizada para almacenista
- Motor de conciliación con los cuatro buckets
- Pantalla de conciliación con clasificación y justificación
- Export a Excel del reporte

**Entregable:** ciclo completo de fin de mes (snapshot SAP + conteo físico + conciliación → reporte).

### Fase 3 — Movimientos operativos (semanas 7-10)
- Modelo de movimientos y máquina de estados
- Endpoints de movimientos (entrada, salida cuadrilla, devolución)
- Generación automática de vales PDF
- Sugerencia automática de justificación en conciliación basada en movimientos
- Pantallas de operación diaria para almacenistas

**Entregable:** Alesig deja de llevar control en Excel paralelo; todo entra al sistema.

### Fase 4 — Flujos avanzados (semanas 11-14)
- Préstamos internos y externos
- Reasignación entre proyectos
- Material defectuoso con flujo de evidencia
- Certificación SAP con workflow

### Fase 5 — Auditoría y madurez (semanas 15-16)
- Reportes ejecutivos
- Vistas de auditor cliente (read-only)
- Migración de auth a Azure AD (opcional si el cliente lo requiere en fases futuras)
- Hardening: rate limits, logs estructurados, métricas de Prometheus
- Despliegue a producción en Railway con PostgreSQL managed, dominio personalizado y CI/CD desde Git

---

## 9. Criterios de aceptación del MVP

El MVP (fases 0-2) se considera aceptado cuando:

1. **Importación funcional**: Jessica sube un Excel SAP nuevo y el sistema crea el snapshot sin errores en menos de 30 segundos para archivos de hasta 500 materiales.
2. **Idempotencia**: subir el mismo archivo dos veces NO crea dos snapshots; arroja error claro.
3. **Validación estricta**: si un archivo tiene una columna renombrada o faltante, la importación se rechaza con mensaje específico antes de tocar la BD.
4. **Conteo capturable**: un almacenista puede registrar un conteo de 100 materiales en menos de 30 minutos.
5. **Conciliación produce los cuatro buckets**: dado un snapshot y un conteo, el sistema clasifica automáticamente las líneas que cuadran y deja para revisión humana las que no.
6. **Reporte exportable**: la conciliación se exporta a Excel con formato profesional listo para enviar al cliente.
7. **Permisos respetados**: un almacenista de Toluca no puede ver datos de CDMX.
8. **Tests**: cobertura mínima 70% en backend, 50% en frontend. Los Excels reales están en fixtures y los tests pasan con ellos.
9. **Documentación**: README con setup local funciona en una máquina limpia, OpenAPI accesible en `/docs`.
10. **Trazabilidad**: cada acción de mutación registra usuario, timestamp y motivo (cuando aplica).

---

## 10. Decisiones abiertas — preguntas para Jessica antes de avanzar

Estas preguntas deben responderse antes de cerrar la fase 1. No asumir; agendar reunión:

1. **Granularidad del material**: ¿se trackea por número de serie único, por lote, o solo por SKU + cantidad? Los archivos sugieren SKU + cantidad pero confirmar.
2. **Frecuencia del Excel SAP**: Jessica lo extrae directamente desde su usuario SAP cuando lo necesita. ¿Con qué frecuencia lo hace actualmente — mensual, semanal, on-demand? ¿Hay un momento del mes que sea obligatorio (cierre contable)?
3. ~~**Almacén SAP vs físico**~~ — **RESUELTO (15/05/2026):** Solo existen dos almacenes físicos (Toluca y CDMX). La distinción operativa es por `Centro` (`12T2` vs `37X2`). El código `1DAL` se repite pero corresponde a almacenes distintos identificados por su centro.
4. **"Pend. Autorizar" alto**: en CDMX hay un material con 372 unidades pendientes de autorizar. ¿Es escenario normal o alarma operativa?
5. **"Docum. Borrado"**: ¿qué representan? Hay valores muy altos (820, 5000+). ¿Cancelaciones del cliente o correcciones de Alesig?
6. **Préstamos externos certificables**: ¿una empresa externa puede certificar material en su propio proyecto, o siempre debe devolverlo? El doc original es ambiguo.
7. **Autorización de reasignación entre proyectos**: ¿qué rol autoriza? ¿Hay tope de monto?
8. **Política de pérdidas no justificadas**: cuando una diferencia no se justifica, ¿se castiga al supervisor, se ajusta contra costos, otro?
9. **Material defectuoso por garantía**: ¿quién paga? ¿hay flujo con el proveedor?
10. ~~**Integración con SAP directa**~~ — **PARCIALMENTE RESUELTO (15/05/2026):** Alesig tiene usuario SAP propio y extrae los Excels directamente. La integración vía API sigue siendo una decisión de Naturgy a futuro. Por ahora el flujo es: Jessica exporta → sube al sistema.
11. **Acceso del auditor cliente**: ¿se le da usuario en el sistema o solo se le envían reportes? Si se le da usuario, ¿con qué scope?
12. **Volumen esperado**: ¿cuántos movimientos por día se espera capturar cuando el sistema esté en producción? Afecta sizing de infra.
13. **Proyectos de mantenimiento vs construcción**: ¿tienen reglas distintas de certificación, reporteo o tiempos de cierre ante el cliente? ¿Hay materiales exclusivos de mantenimiento que no aplican a construcción?
14. **Gas LP vs gas natural**: ¿los materiales de gas LP y gas natural están mezclados en el mismo almacén y catálogo SAP, o son almacenes/centros completamente separados?
15. **Traslados SAP — frecuencia e iniciador**: ¿con qué frecuencia Naturgy hace traslados entre centros (12T2 ↔ 37X2)? ¿Avisan a Alesig cuando lo hacen, o Alesig lo descubre al importar el siguiente Excel?
16. **Traslados SAP — acompañan movimiento físico?**: cuando Naturgy hace un traslado contable entre centros, ¿el material siempre se mueve físicamente entre almacenes, nunca, o depende del caso?
17. **Traslados SAP — dos pasos**: ¿Naturgy usa traslados en dos pasos (con "stock en tránsito")? Si es así, en el Excel de extracción ese material no aparecería en ningún centro durante el tránsito — ¿Jessica ha visto materiales "desaparecer" entre una extracción y otra?
18. **Registro del traslado en el sistema**: cuando Jessica detecta un traslado al importar un nuevo Excel, ¿ella registra manualmente el traslado en el sistema de Alesig, o el sistema debería detectarlo automáticamente al comparar snapshots consecutivos?

---

## 11. Notas para Claude Code al arrancar

**Configuración de bases de datos:**
- Crear proyecto en Neon (`neon.tech`) con free tier
- Cada dev: rama propia en Neon para desarrollo aislado
- `.env.local` con connection string de Neon (nunca comitear)
- `.env.example`: placeholder genérico
- Producción: Railway Postgres managed (configurar después de Fase 2)

**Stack Node.js/TypeScript:**
- **React 18 estricto**: Next.js 15 instala React 19 por defecto. Para forzar React 18, fijar explícitamente en el `package.json` del workspace raíz: `"react": "18.3.1"` y `"react-dom": "18.3.1"`. Sin esto, `pnpm install` puede pisar la versión. Hay problemas conocidos de DOM reconciliation en React 19 canary que afectan shadcn/ui.
- **No instalar dependencias sin actualizarlas en `package.json`**: todo debe ser explícito en el monorepo.
- **Exports explícitos**: no usar `export * from` en módulos de dominio, listar cada export.
- **Imports absolutos** en backend (vía `tsconfig.json`), no relativos.
- **Type hints en todo el código TypeScript**: Zod para validación, types estrictos.
- **No commitear `.env`**: solo `.env.example` con valores ficticios.
- **Migraciones siempre revisadas a mano**: `prisma migrate dev --name <descripcion>` genera SQL, **revisar antes de commit**.
- **Nunca `DELETE` ni `TRUNCATE` en producción** de tablas append-only (`movimientos`).
- **Tests de importador con los Excels reales como fixtures** desde el primer día.
- **Logs estructurados** con Winston, nunca `console.log` en producción.
- **Manejo de zonas horarias**: todo en UTC en BD, conversión a `America/Mexico_City` solo en presentación.
- **Empezar por la fase 0 completa antes de tocar fase 1**: setup limpio se paga solo.

---

## Apéndice A — Referencias

- Documento original de flujos: provisto por Ing. Jessica Ramos
- Archivos de muestra SAP: `EXTRACCION_MATERIALES_TOLUCA_20_DE_ABRIL.xlsx`, `EXTRACCION_MATERIALES_CDMX_20_DE_ABRIL.xlsx`
- Patrón de ADRs: <https://github.com/joelparkerhenderson/architecture-decision-record>
- Express.js: <https://expressjs.com>
- Prisma: <https://www.prisma.io>
- Neon PostgreSQL: <https://neon.tech>
- Railway: <https://railway.app>

---

**Fin del documento.**
