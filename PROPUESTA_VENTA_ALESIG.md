# PROPUESTA DE DESARROLLO DE SOFTWARE
## Sistema de Trazabilidad de Materiales — ALESIG

---

<!-- DIAPOSITIVA 1: PORTADA -->

# Sistema ALESIG
### Trazabilidad y Conciliación de Inventario de Materiales

**Preparado para:** Ing. Jessica Ramos  
**Consultoría Empresarial Alesig SC**

**Fecha:** Mayo 2026  
**Versión:** 1.0

---

<!-- DIAPOSITIVA 2: EL PROBLEMA -->

# El Problema Actual

> "Cada fin de mes, el equipo de Alesig cruza manualmente el Excel de SAP contra el control interno."

### Consecuencias directas:

| Problema | Impacto |
|---|---|
| Conciliación manual y lenta | Horas de trabajo repetitivo |
| Diferencias inexplicables | Pérdidas sin origen trazable |
| Sin visibilidad en tiempo real | Decisiones basadas en datos viejos |
| Sin auditoría completa | Imposible saber quién tuvo qué material |

---

<!-- DIAPOSITIVA 3: LA SOLUCIÓN -->

# La Solución

### Un sistema web que:

1. **Traza** cada material desde que entra al almacén hasta que se certifica en SAP
2. **Concilia automáticamente** los reportes de SAP contra el estado operativo
3. **Documenta** todos los movimientos con vales generados automáticamente
4. **Audita** cualquier material en cualquier momento

### Operación cubierta:
- Almacén Toluca → Centro `12T2` (Naturgy México)
- Almacén CDMX → Centro `37X2` (Metrogas)

---

<!-- DIAPOSITIVA 4: ALCANCE FUNCIONAL -->

# Alcance Funcional

### Módulos incluidos:

- **Importador SAP** — Carga de reportes Excel directamente desde el usuario SAP de Alesig en Naturgy
- **Snapshots inmutables** — Fotografías del estado SAP archivadas y auditables
- **Conciliación automática** — Cruce entre SAP y estado operativo en 4 categorías
- **Conteo físico** — Captura desde celular para almacenistas en piso
- **Control de movimientos** — Entradas, salidas, préstamos, devoluciones, certificaciones
- **Generación de vales** — PDF automático por cada movimiento
- **Roles y permisos** — Control granular por almacén y función
- **Reportes ejecutivos** — Exportables a Excel para el cliente Naturgy

---

<!-- DIAPOSITIVA 5: FASES DEL PROYECTO -->

# Plan de Implementación

## 6 Fases — 16 Semanas

```
FASE 0      FASE 1       FASE 2     FASE 3         FASE 4          FASE 5
Setup     Importador   Concil.   Movimientos    Avanzados      Producción
Sem. 1-2  Sem. 3-6    Sem. 7-8  Sem. 9-14     Sem. 15-20     Sem. 21-23

[███████] [████████████] [██████] [█████████████] [█████████████] [█████████]
   MVP ←──────────────────────→
```

---

<!-- DIAPOSITIVA 6: FASE 0 -->

# Fase 0 — Setup y Cimientos
### Semanas 1–2 | Entregable: Ambiente funcional para el equipo

**Qué se construye:**
- Infraestructura del proyecto (monorepo, CI/CD)
- Base de datos con tablas de catálogo (materiales, centros, almacenes, usuarios)
- Login seguro con roles (Almacenista, Supervisor, Admin, Auditor)
- Datos de prueba iniciales (almacenes Toluca y CDMX, centros 12T2 y 37X2)

---

<!-- DIAPOSITIVA 7: FASE 0.5 -->

# Fase 0.5 — Carga Inicial de Datos
### Previo al go-live | Una sola vez

**Proceso:**
1. El almacenista responsable realiza un **conteo físico** en cada almacén en la fecha de corte acordada
2. El conteo se captura en el sistema como inventario inicial
3. El Excel SAP más reciente se importa como primer snapshot
4. Se ejecuta la **primera conciliación** para detectar diferencias desde el arranque

> El sistema parte de este momento en adelante.  
> Los registros históricos previos permanecen en los archivos del equipo operativo.

---

<!-- DIAPOSITIVA 8: FASE 1 -->

# Fase 1 — Importador SAP + Conciliación Básica
### Semanas 3–6 | Entregable: Primera conciliación real

**Qué se construye:**
- Importación de archivos Excel SAP (44 columnas, validación estricta)
- Detección automática de materiales nuevos y cambios de catálogo
- Pantalla de carga con drag-and-drop y vista previa
- Historial de snapshots importados con descarga del original
- **Conciliación entre dos snapshots**: diferencias mes a mes

**Resultado visible para Alesig:**
> El equipo de Alesig sube el Excel de abril y el de mayo.  
> El sistema muestra qué materiales variaron y cuánto.

---

<!-- DIAPOSITIVA 9: FASE 2 -->

# Fase 2 — Conteo Físico + Conciliación Completa
### Semanas 7–8 | Entregable: Ciclo completo de fin de mes

**Qué se construye:**
- App mobile-first para captura de conteo físico desde celular
- Motor de conciliación con **4 categorías automáticas**:

| Categoría | Qué significa |
|---|---|
| **Cuadra** | SAP y físico coinciden |
| **Justificada** | Diferencia explicada por movimientos |
| **Pendiente investigar** | Requiere revisión humana |
| **Pérdida confirmada** | Diferencia declarada como pérdida |

- Clasificación y justificación manual por línea
- **Exportación a Excel** del reporte final listo para cliente

**Resultado visible para Alesig:**
> Ciclo completo: snapshot SAP + conteo físico → reporte en Excel para Naturgy.

---

<!-- DIAPOSITIVA 10: FASE 3 -->

# Fase 3 — Movimientos Operativos
### Semanas 9–14 | Entregable: Fin del control en Excel paralelo

**Qué se construye:**
- Registro completo de movimientos:
  - Entradas al almacén
  - Salidas a cuadrilla (con aprobación de supervisor)
  - Devoluciones
  - Instalaciones
  - Certificaciones SAP
- Generación automática de **vales PDF** por cada movimiento
- **Sugerencia automática de justificación**: el sistema busca movimientos que expliquen las diferencias en conciliación
- Pantallas de operación diaria para almacenistas

**Resultado visible para Alesig:**
> Alesig deja de llevar el control en Excel. Todo queda en el sistema.

---

<!-- DIAPOSITIVA 11: FASE 4 -->

# Fase 4 — Flujos Avanzados
### Semanas 15–20 | Entregable: Operación completa cubierta

**Qué se construye:**
- **Préstamos internos**: transferencias temporales Toluca ↔ CDMX
- **Préstamos externos**: material entregado a empresa externa con trazabilidad
- **Reasignación entre proyectos**: cambio de destino sin devolver al almacén
- **Material defectuoso**: flujo con captura de evidencia fotográfica
- **Certificación SAP con workflow**: seguimiento hasta cierre contable

---

<!-- DIAPOSITIVA 12: FASE 5 -->

# Fase 5 — Producción y Madurez
### Semanas 21–23 | Entregable: Sistema en producción estable

**Qué se construye:**
- Reportes ejecutivos para dirección
- Vista de auditor cliente (Naturgy/Metrogas) — solo lectura
- Despliegue a producción con dominio personalizado
- Seguridad: rate limiting, logs estructurados, monitoreo de errores
- Documentación técnica y manual de usuario

---

<!-- DIAPOSITIVA 13: RESUMEN DE TIEMPOS -->

# Resumen de Tiempos

| Fase | Descripción | Duración | Semanas |
|---|---|---|---|
| **Fase 0** | Setup e infraestructura | 2 semanas | 1–2 |
| **Fase 0.5** | Carga inicial de datos | Coordinada con Alesig | — |
| **Fase 1** | Importador SAP + conciliación básica | 4 semanas | 3–6 |
| **Fase 2** | Conteo físico + conciliación completa | 2 semanas | 7–8 |
| **Fase 3** | Movimientos operativos | 6 semanas | 9–14 |
| **Fase 4** | Flujos avanzados | 6 semanas | 15–20 |
| **Fase 5** | Producción y madurez | 3 semanas | 21–23 |
| **TOTAL** | | **23 semanas** | **~6 meses** |

### MVP (Fases 0–2): **8 semanas (~2 meses)**
> Con el MVP, Alesig ya puede realizar el ciclo completo de conciliación mensual sin hojas de cálculo manuales.

---

<!-- DIAPOSITIVA 14: TECNOLOGÍA -->

# Stack Tecnológico

### Sin costos de licencias. Todo open source o con plan gratuito en fases iniciales.

| Capa | Tecnología |
|---|---|
| **Frontend** | Next.js + TypeScript + shadcn/ui |
| **Backend** | Node.js + Express + TypeScript |
| **Base de datos** | PostgreSQL (Neon dev · Railway producción) |
| **Almacenamiento** | Digital Ocean Spaces (archivos Excel SAP) |
| **Notificaciones** | Brevo (email, SMS, WhatsApp) |
| **Despliegue** | Railway (CI/CD automático desde Git) |

---

<!-- DIAPOSITIVA 15: ROLES DE USUARIO -->

# Roles del Sistema

| Rol | Quién | Qué puede hacer |
|---|---|---|
| **Admin** | Dirección / Responsable operativo | Todo |
| **Supervisor** | Jefe de almacén | Aprobar salidas, ver ambos almacenes |
| **Almacenista Toluca** | Operador de almacén | Solo Toluca |
| **Almacenista CDMX** | Operador de almacén | Solo CDMX |
| **Auditor Cliente** | Naturgy / Metrogas | Solo lectura, sin datos internos |
| **Dirección** | Alta dirección Alesig | Reportes ejecutivos |

---


<!-- DIAPOSITIVA 18: CIERRE -->

# ¿Por qué ahora?

> Cada mes sin el sistema, las diferencias de inventario se acumulan sin causa documentada.

### Lo que Alesig gana:

- **Visibilidad en tiempo real** del inventario en ambos almacenes
- **Conciliación en horas**, no días
- **Evidencia auditable** ante Naturgy en cualquier momento
- **Eliminación del riesgo** de pérdidas no trazadas

---

---

<!-- DIAPOSITIVA 19: INVERSIÓN POR FASES -->

# Inversión — Pago por Fases

> Cada fase tiene entregables concretos y verificables antes del siguiente pago.

> **Todos los precios indicados en esta propuesta son más IVA (16%).**

---

### Pago 1 — MVP (Fases 0, 0.5, 1 y 2)
**Alcance:** Setup completo + Importador SAP + Conteo físico + Conciliación + Reporte exportable  
**Duración:** 8 semanas (~2 meses)

| Concepto | Monto |
|---|---|
| Desarrollo e infraestructura del MVP | MXN 35,000 |
| **Subtotal MVP** | **MXN 35,000 + IVA** |

**Forma de pago:**
- 50% al firmar contrato → MXN 17,500 + IVA
- 50% al entregar el MVP funcionando → MXN 17,500 + IVA

---

### Pago 2 — Fase 3: Movimientos Operativos
**Alcance:** Registro completo de movimientos, vales PDF, pantallas de operación diaria  
**Duración:** 6 semanas

| Concepto | Monto |
|---|---|
| Desarrollo (motor de movimientos, vales, flujo de aprobaciones) | MXN 20,000 |
| **Subtotal Fase 3** | **MXN 20,000 + IVA** |

**Forma de pago:** 100% al entregar la fase funcionando

---

### Pago 3 — Fase 4: Flujos Avanzados
**Alcance:** Préstamos internos/externos, reasignaciones, material defectuoso, certificación SAP  
**Duración:** 6 semanas

| Concepto | Monto |
|---|---|
| Desarrollo (préstamos, reasignaciones, defectuosos, certificaciones) | MXN 20,000 |
| **Subtotal Fase 4** | **MXN 20,000 + IVA** |

**Forma de pago:** 100% al entregar la fase funcionando

---

### Pago 4 — Fase 5: Producción y Madurez
**Alcance:** Reportes ejecutivos, acceso auditor cliente, hardening, despliegue final en Railway  
**Duración:** 3 semanas

| Concepto | Monto |
|---|---|
| Desarrollo (reportes, roles auditor, seguridad, documentación) | MXN 20,000 |
| **Subtotal Fase 5** | **MXN 20,000 + IVA** |

**Forma de pago:** 100% al despliegue exitoso en producción

---

<!-- DIAPOSITIVA 20: RESUMEN DE INVERSIÓN -->

# Resumen de Inversión — Desarrollo

| Pago | Entregable principal | Duración | Monto MXN |
|---|---|---|---|
| **MVP** (Fases 0–2) | Importador SAP + conciliación + reporte | 8 semanas | MXN 35,000 |
| **Fase 3** | Movimientos operativos + vales PDF | 6 semanas | MXN 20,000 |
| **Fase 4** | Flujos avanzados (préstamos, cert. SAP) | 6 semanas | MXN 20,000 |
| **Fase 5** | Producción, auditor cliente, hardening | 3 semanas | MXN 20,000 |
| **TOTAL PROYECTO** | **Sistema completo en producción** | **23 semanas** | **MXN 95,000 + IVA** |

> Los precios incluyen honorarios de desarrollo e infraestructura asociada a cada fase.  
> Todos los precios son **más IVA (16%)**.

---

<!-- DIAPOSITIVA 21: PRECIO BASE DE OPERACIÓN -->

# Precio Base de Operación Mensual
### A partir de la entrega en producción (Fase 5)

> Cubre todo lo necesario para que el sistema opere sin interrupciones.

---

**¿Qué incluye la mensualidad?**

| Servicio | Descripción |
|---|---|
| **Infraestructura** | Railway (app + base de datos PostgreSQL managed), Digital Ocean Spaces (almacenamiento de Excels SAP), Brevo (email y notificaciones), Sentry (monitoreo de errores) |
| **Mantenimiento técnico** | Actualizaciones de dependencias, parches de seguridad, migraciones de base de datos menores |
| **Soporte operativo** | Atención a incidencias, ajustes de configuración, respuesta a errores reportados |
| **Backup y monitoreo** | Verificación periódica de integridad de datos y respaldos automáticos |

---

| Plan | Tiempo de respuesta | SLA | Precio mensual |
|---|---|---|---|
| **Prioritario** | 24 horas hábiles | Lunes–Viernes 9:00–18:00 | **MXN 10,000 / mes + IVA** |

> Incluye infraestructura, monitoreo, mantenimiento técnico y soporte operativo.

**La mensualidad no incluye:**  
Nuevas funcionalidades o módulos adicionales (se cotizan por separado como una fase nueva).

---

<!-- DIAPOSITIVA 22: CONDICIONES COMERCIALES -->

# Condiciones Comerciales

### Garantía de entrega
- Cada fase incluye **10 días hábiles de ajustes** sin costo adicional a partir de la entrega
- Los ajustes deben estar dentro del alcance acordado para esa fase

### Licencia de uso y propiedad del código
- El cliente (Alesig) recibe una **licencia perpetua de uso** del sistema, incluyendo todas las actualizaciones derivadas del contrato
- El código fuente permanece como propiedad intelectual del desarrollador
- **Opción de adquisición de código fuente:** si Alesig requiere la titularidad del código en cualquier momento, puede adquirirla mediante un acuerdo de cesión de derechos. El precio se determina con base en las tarifas de mercado vigentes al momento de la solicitud. A modo de referencia, los valores actuales de mercado por fase son:

| Fase | Referencia de mercado (Mayo 2026) |
|---|---|
| MVP (Fases 0–2) | MXN 128,000 |
| Fase 3 | MXN 78,000 |
| Fase 4 | MXN 75,000 |
| Fase 5 | MXN 44,000 |

> Estos montos son referenciales y están sujetos a actualización según las condiciones del mercado al momento de ejercer la opción. Los montos ya pagados por servicio se descontarían del precio de adquisición correspondiente.

### Cambios de alcance
- Cambios menores (ajustes de UX, columnas adicionales): incluidos dentro de los 10 días de ajuste
- Cambios mayores (nuevos módulos, nuevas integraciones): se cotizan como una fase adicional

### Confidencialidad
- Toda la información de Alesig, Naturgy y Metrogas es tratada como **estrictamente confidencial**
- Firma de NDA disponible si el cliente lo requiere

### Vigencia de la propuesta
- Esta cotización tiene vigencia de **30 días** a partir de la fecha de emisión

---

<!-- DIAPOSITIVA 23: CUADRO COMPARATIVO -->

# ¿Cuánto vale el sistema para Alesig?

| Situación actual (sin sistema) | Con sistema ALESIG |
|---|---|
| Conciliación manual: 1–2 días/mes | Conciliación automática: < 2 horas |
| Diferencias sin origen trazable | Cada diferencia tiene historial de movimientos |
| Sin visibilidad de material en cuadrilla | Estado en tiempo real por material |
| Riesgo de pérdidas no documentadas | Trazabilidad completa desde entrada hasta certificación SAP |
| Imposible auditar sin Excel | Auditoría on-demand para Naturgy/Metrogas |

> Si el sistema recupera o evita **1 pérdida de inventario no trazada al mes**, la inversión se paga sola.

---

<!-- DIAPOSITIVA 24: CIERRE FINAL -->

# Inversión Total en Resumen

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  DESARROLLO COMPLETO (23 semanas)    MXN  95,000 + IVA  │
│                                                          │
│  MVP  — Fases 0–2           MXN  35,000  (sem. 1–8)    │
│  Fase 3 — Movimientos        MXN  20,000  (sem. 9–14)   │
│  Fase 4 — Avanzados          MXN  20,000  (sem. 15–20)  │
│  Fase 5 — Producción         MXN  20,000  (sem. 21–23)  │
│                                                          │
│  OPERACIÓN MENSUAL (prioritario)  MXN 10,000/mes + IVA  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Inversión en el primer año:**  
MXN 95,000 desarrollo + MXN 120,000 operación (12 meses) = **MXN 215,000** + IVA

---

**Elaborado por:** Equipo de desarrollo  
**Para:** Ing. Jessica Ramos — Consultoría Empresarial Alesig SC  
**Contacto:** —  

---

*Este documento es confidencial y fue preparado exclusivamente para Consultoría Empresarial Alesig SC.*
