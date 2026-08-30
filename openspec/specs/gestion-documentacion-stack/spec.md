# Gestión de Documentación del Stack

## Purpose

Este repositorio (`stack_ticket_doc`) es un agregador puramente documental: no contiene
código ejecutable, scripts de build, generador de sitio estático, ni configuración de
aplicación alguna (no hay `package.json`, `pyproject.toml`, pipelines de CI, ni código
fuente). Su única función es reunir, en un solo lugar versionado con Git, una carpeta
`docs/` espejo por cada servicio del ecosistema de tickets, de modo que la documentación
técnica y de usuario de todo el stack pueda consultarse y versionarse sin necesidad de
clonar cada repositorio de servicio por separado.

Esta capacidad documenta la estructura, convenciones y garantías reales que el repo
cumple hoy (baseline), no un comportamiento en tiempo de ejecución.

## Requirements

### Requirement: Espejo de documentación por servicio
El repositorio SHALL mantener una carpeta de nivel superior por cada servicio del
ecosistema de tickets, y cada una de esas carpetas SHALL contener una subcarpeta
`docs/` con la documentación de ese servicio.

#### Scenario: Servicios cubiertos
- **WHEN** se lista el contenido de la raíz del repositorio
- **THEN** existen las carpetas `infra-monitoring/`, `notification-service/`,
  `ticket-agent/`, `ticket-classification/`, `ticket-ingestion-light/` y
  `ticket-management/`
- **AND** cada una contiene una subcarpeta `docs/`

### Requirement: Convención de runbooks numerados
Cada servicio SHALL organizar su documentación operativa en `docs/runbooks/` usando
archivos Markdown con un prefijo numérico de dos dígitos que indica el orden de lectura
recomendado, y el conjunto mínimo común de runbooks SHALL cubrir: una guía conceptual
("for dummies"), la arquitectura técnica, y la guía operacional.

#### Scenario: Runbooks mínimos presentes en todos los servicios
- **WHEN** se inspecciona `docs/runbooks/` de cualquiera de los seis servicios
- **THEN** existe un archivo `01_..._for_dummies.md` (o equivalente `01_ai_stack_for_dummies.md`
  en `ticket-classification`), un archivo `02_arquitectura_tecnica.md` y un archivo
  `03_guia_operacional.md`

#### Scenario: Runbooks adicionales específicos de un servicio
- **WHEN** un servicio tiene necesidades documentales adicionales (p. ej. `ticket-agent`
  con `04_arquitectura_langgraph.md`, `05_api_rest.md`, `06_deploy_contenedor.md`, o
  `infra-monitoring` con `09_reset_datos_pruebas.md`, `10_redes_y_acceso_lan.md`,
  `11_puertos_y_redes.md`)
- **THEN** esos runbooks adicionales SHALL seguir el mismo prefijo numérico de dos dígitos
  dentro de la misma carpeta `docs/runbooks/`, sin romper la numeración mínima común

### Requirement: Documentación técnica adicional bajo `docs/tecnico`
Un servicio SHALL poder incluir una subcarpeta `docs/tecnico/` con documentos de
planeación o migración técnica que no son runbooks operativos (p. ej. resúmenes
ejecutivos, planes de tareas, planes de UI, prompts de referencia).

#### Scenario: ticket-agent y ticket-management tienen docs/tecnico
- **WHEN** se inspecciona `ticket-agent/docs/tecnico/` o `ticket-management/docs/tecnico/`
- **THEN** existen documentos como `RESUMEN_EJECUTIVO.md`, `TAREAS.md`,
  `ORACLE_MIGRACION.md`, `PLAN_UI_AGENTE.md` o `PROMPT_FINAL.md` según el servicio

### Requirement: Documentación de usuario final segmentada por rol (ticket-management)
`ticket-management` SHALL mantener, además de sus runbooks técnicos, una subcarpeta
`docs/usuarios/` con documentación dirigida a usuarios finales, segmentada por rol
(`admin`, `manager`, `usuario`). Las carpetas `admin/` y `manager/` SHALL incluir como
mínimo los documentos `estados.md`, `flujos.md`, `glosario.md` y `permisos.md`; la
carpeta `usuario/` actualmente SOLO contiene `estados.md`, `flujos.md` y `glosario.md`
(le falta `permisos.md`), lo cual es una brecha conocida frente al mínimo común y no
un diseño intencional (ver Requirement "Inconsistencias conocidas de la estructura
actual").

#### Scenario: Cobertura de los tres roles
- **WHEN** se inspecciona `ticket-management/docs/usuarios/`
- **THEN** existen las subcarpetas `admin/`, `manager/` y `usuario/`
- **AND** `admin/` y `manager/` contienen `estados.md`, `flujos.md`, `glosario.md` y
  `permisos.md`
- **AND** `usuario/` contiene únicamente `estados.md`, `flujos.md` y `glosario.md`
  (sin `permisos.md`)

#### Scenario: Metadatos de audiencia en los documentos de usuario
- **WHEN** se abre cualquier archivo Markdown bajo `docs/usuarios/<rol>/`
- **THEN** el archivo SHALL comenzar con un bloque de frontmatter YAML que declara al
  menos `rol`, `tipo`, `audiencia` y `fuente`

#### Scenario: `configuracion.md` exclusivo de admin
- **WHEN** se inspecciona `ticket-management/docs/usuarios/admin/`
- **THEN** existe un archivo adicional `configuracion.md` que no tiene equivalente en
  `manager/` ni `usuario/`
- **AND** esto es intencional y no una brecha de replicación: su contenido (gestión de
  usuarios, gestión de clasificaciones, configuración de workflow, gestión de permisos
  y archivado de tickets) cubre exclusivamente acciones que requieren permisos
  reservados a Admin (`MANAGE_USERS`, `MANAGE_CLASSIFICATIONS`, `MANAGE_WORKFLOW`,
  `ARCHIVE_TICKET`), inexistentes para Manager o Usuario

### Requirement: Diagramas de arquitectura como artefactos versionados
Un servicio SHALL poder documentar su arquitectura mediante diagramas en formato Mermaid
(`.mmd`) versionados junto al resto de la documentación, en lugar de (o además de)
descripciones puramente textuales.

#### Scenario: Diagrama de ecosistema en ticket-agent
- **WHEN** se inspecciona `ticket-agent/docs/arquitectura/`
- **THEN** existe el archivo `ecosistema.mmd` con el diagrama del ecosistema completo

### Requirement: Materiales de referencia no-Markdown
El repositorio SHALL permitir incluir materiales de referencia en formatos distintos a
Markdown (p. ej. PDF) dentro de la carpeta `docs/runbooks/` de un servicio cuando el
contenido lo amerite (formularios, plantillas de onboarding, etc.).

#### Scenario: PDF de onboarding en ticket-ingestion-light
- **WHEN** se inspecciona `ticket-ingestion-light/docs/runbooks/`
- **THEN** existe el archivo `cortex-soporte-tecnico-onboarding-form.pdf` junto a los
  runbooks Markdown del servicio

### Requirement: Inconsistencias conocidas de la estructura actual
El baseline SHALL documentar explícitamente las desviaciones existentes respecto a la
convención de espejo `**/docs/`, en lugar de ocultarlas, para que una futura limpieza
pueda planificarse con información precisa.

#### Scenario: Carpeta `tecnico/` duplicada fuera de `docs/` en ticket-agent
- **WHEN** se inspecciona la raíz de `ticket-agent/`
- **THEN** existe una carpeta `ticket-agent/tecnico/` (con `RESUMEN_EJECUTIVO.md` y
  `TAREAS.md`) que duplica, fuera del árbol `docs/`, contenido también presente en
  `ticket-agent/docs/tecnico/`
- **AND** esta duplicación es una desviación de la convención de espejo declarada en
  el Requirement "Espejo de documentación por servicio" y no un patrón a replicar en
  otros servicios
- **AND** el contenido de ambas copias diverge y `ticket-agent/docs/tecnico/` es la
  versión más reciente y avanzada: en `TAREAS.md`, `ticket-agent/tecnico/` documenta
  el ítem "Routing KB_RUNBOOKS" como **pendiente** (con fecha de última actualización
  2026-05-29 y un plan de 6 checkboxes sin marcar), mientras que
  `ticket-agent/docs/tecnico/TAREAS.md` documenta ese mismo ítem como **✅ Resuelto
  2026-06-01** (6 checkboxes marcados, evidencia de verificación con confianza 0.8,
  y una tarea adicional "Web URLs en Docling" que no existe en la copia de `tecnico/`);
  de forma consistente, `RESUMEN_EJECUTIVO.md` en `docs/tecnico/` incluye la fecha de
  corte "Junio 2026" y una tabla de "Estado vs arquitectura objetivo" (80% implementado)
  ausente en la copia de `tecnico/`, que en cambio describe un estado de "Fases
  completadas" anterior
- **AND** por lo tanto `ticket-agent/tecnico/` es un snapshot desactualizado que quedó
  sin eliminar tras mover/actualizar la documentación técnica a `docs/tecnico/`, no una
  fuente alternativa o complementaria válida; cualquier limpieza futura SHOULD eliminar
  `ticket-agent/tecnico/` y conservar `ticket-agent/docs/tecnico/` como fuente de verdad
