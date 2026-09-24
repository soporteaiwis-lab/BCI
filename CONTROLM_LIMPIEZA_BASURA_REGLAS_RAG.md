# Control-M — Detección, consenso y limpieza de "basura" (jobs, mallas, condiciones, recursos)

> **Base de conocimiento para ArgOS** — reglas, umbrales, comandos API/CLI, queries SQL,
> código de remediación y hoja de ruta para pasar el módulo de Limpieza de *simulado* a *real*.
>
> Versión 1.0 — 2026-09-24. Autor: Claude Opus 5.5 para Armin Salazar (AIWIS / ArgOS).
> Pensado para ser *chunkeado* como RAG: cada regla tiene un ID estable (`R-XXX-NN`),
> y cada sección se puede leer sola.

---

## 0. Cómo leer este documento (y qué tan confiable es cada dato)

| Marca | Significado |
|---|---|
| ✅ **API** | Endpoint verificado contra el **Swagger oficial de Control-M Automation API v9.22.125** (copiado en `docs/referencia_controlm_api/controlm_swagger_9.22.125.json`, 300 endpoints). Es la fuente más confiable de este documento. |
| 🟡 **SQL** | Tablas y columnas documentadas por BMC y comunidad (DEF_JOB, DEF_LNKI_P, DEF_LNKO_P, CMS_JOBDEF, CMR_RUNINF, CMR_AJF, CMR_IOALOG). **Validar contra el Physical Data Model de la versión instalada en BCI** antes de correr en producción: los nombres de columna cambian entre versiones. |
| 🟠 **UTIL** | Utilitarios de línea de comando (ctmpsm, ctmcontb, emdef, IOA/z/OS). Existen y se usan así habitualmente, pero la sintaxis exacta depende de la versión: validar con `-h` o con el manual de la versión antes de automatizarlos. |
| 🔵 **CRITERIO** | Umbral o práctica de gobierno propuesta por AIWIS. No es un estándar de BMC; es la política que ArgOS propone y que BCI debe aprobar. |

**Regla de oro del documento:** *ningún job se elimina porque una sola señal lo diga*. La
basura se declara por **consenso de evidencias** (sección 3), y la IA **propone pero nunca
decide ni ejecuta** la eliminación.

Contexto de negocio: en la llamada de preventa (ver
`docs/ANALISIS MEET BCI-AIWIS PROYECTO ARGOS/PABLO_SANDOVAL_LLAMADA1_ARGOS_PREVENTA_TRANSCRIPCION.md`)
el pedido fue: *"quiero saber cómo lo hacen a mano, qué comandos corren (…) con esto logré
determinar que sí, candidatos a eliminar (…) pido la autorización, luego ya se elimina"*.
Este documento reconstruye justamente ese proceso manual para que ArgOS lo haga con IA.

---

## 1. Qué es "basura" en Control-M: taxonomía completa

Hay 9 familias. Cada regla trae: **qué es**, **cómo se detecta** (fuente), **peso** en el
puntaje (sección 2) y **falsos positivos típicos** (lo que NO hay que borrar aunque lo parezca).

### Familia A — Definiciones de jobs muertas (lo principal)

| ID | Basura | Cómo se detecta | Peso | Falsos positivos |
|---|---|---|---|---|
| **R-DEF-01** | **Job sin ejecución en N días** (N=90 alerta, 400 fuerte) | Última corrida en `CMR_RUNINF`/Archive/`/archive/search` vs. definición en `/deploy/jobs` | 25 | Jobs anuales, cierre fiscal, contingencia/DR, on-demand |
| **R-DEF-02** | **Job que nunca se ha ejecutado** desde que se creó | Sin registros en `CMR_RUNINF` ni en Archive + `CREATIONDATETIME` antigua (> 180 días) | 30 | Job recién creado para un proyecto que aún no sale a producción |
| **R-DEF-03** | **Job cuyo calendario nunca lo va a programar** en los próximos 24 meses | ✅ `GET /run/forecast/timeline` (rango −24..+24 meses) devuelve vacío | 30 | Jobs que se ordenan a mano (`ctmorder`, `/run/order`) o por `DO FORCEJOB` desde otro job |
| **R-DEF-04** | **Job con fecha de vigencia vencida** (Active From/Until, `ActiveRetentionPolicy`, `Until`) | Campo de vigencia en la definición < hoy | 35 | Ninguno relevante: si venció, ya no corre |
| **R-DEF-05** | **Job huérfano de script/JCL**: el `MEMNAME`/`FileName`/`Command` apunta a algo que no existe | z/OS: el miembro no está en `EXPOD.SPC20.JCLLIB`. Distribuido: el script no existe en el agente | 30 | Miembro generado dinámicamente, librería con concatenación (`MEMLIB` alternativo) |
| **R-DEF-06** | **Job apuntando a agente/host inexistente o deshabilitado** | `Host`/`NODEID` de la definición ∉ ✅ `GET /config/server/{server}/agents` o agente en estado `Disabled`/`Unavailable` | 25 | Agente caído temporalmente; hostgroup que se resuelve en tiempo de ejecución |
| **R-DEF-07** | **Run-as user inexistente** | `RunAs` ∉ ✅ `GET /config/server/{server}/runasusers` (o usuario dado de baja en el SO/RACF) | 15 | Usuarios con alias o resueltos por LDAP |
| **R-DEF-08** | **Duplicados**: 2+ jobs que ejecutan lo mismo (mismo `Command`/`MEMNAME`+`MEMLIB`+host, mismos parámetros) con distinto nombre | Hash de (tipo, command/memname, host, runAs, parámetros) agrupando en `/deploy/jobs` | 20 | Mismo programa con parámetros o calendarios distintos a propósito (p. ej. diario vs. mensual) |
| **R-DEF-09** | **Jobs de prueba en producción** (`TEST`, `PRUEBA`, `_OLD`, `_BKP`, `COPIA`, `_TMP`, `XX`, `ZZ`, iniciales de persona) | Regex sobre nombre/descripción/folder | 15 | Jobs productivos con nombre heredado mal puesto (confirmar con dueño) |
| **R-DEF-10** | **Dummy sin propósito**: job tipo Dummy sin in-conditions, sin out-conditions, sin `DO` actions | Tipo `Job:Dummy` + listas vacías | 20 | Dummy usado como "hito" en un servicio BIM/SLA |
| **R-DEF-11** | **Nomenclatura inválida** (dominio TDC: `TDCO` + `D/E/M` + 3 caracteres = 8 fijos) | Regex `^TDCO[DEM][A-Z0-9]{3}$` sobre el `MEMNAME`/`JobName` de la aplicación TDC | 5 | Jobs de otra aplicación dentro del mismo folder. **Solo señal de gobierno, no de basura por sí sola** |
| **R-DEF-12** | **Sin dueño**: `Application`/`SubApplication` que ya no existe en el inventario, `CreatedBy` dado de baja, descripción vacía | Cruce con el inventario de aplicaciones/CMDB de BCI | 10 | Sistemas vivos con documentación mala |
| **R-DEF-13** | **Job "siempre OK en ~0 s"**: termina OK en < 2 s en el 100% de las corridas y con output vacío | ✅ `GET /run/job/{jobId}/statistics` + `/archive/search?outputContains=` / `CMR_RUNINF.ELAPTIME` | 15 | Jobs de *touch*/marca, jobs de validación rápida legítimos |
| **R-DEF-14** | **Job forzado a mano de forma sistemática** (`Set To OK`, `Bypass`, `Force OK`) en la mayoría de sus corridas | `CMR_IOALOG` / Audit / log de operación | 20 | Jobs con bug conocido en espera de corrección (eso es deuda, no basura: se clasifica aparte) |

### Familia B — Folders / mallas / sub-folders

| ID | Basura | Cómo se detecta | Peso |
|---|---|---|---|
| **R-FLD-01** | **Folder vacío** (0 jobs) | ✅ `GET /deploy/folders` vs. `GET /deploy/jobs?folder=` | 40 |
| **R-FLD-02** | **Sub-folder vacío** | idem, en SMART folders | 35 |
| **R-FLD-03** | **Folder cuyo 100 % de jobs ya es basura** (malla muerta completa) | Agregación de puntajes: todos los jobs ≥ 60 | 40 |
| **R-FLD-04** | **Folder que nunca se ordena** (SMART folder sin reglas de scheduling válidas / User Daily inexistente) | ✅ `/run/forecast/timeline?folder=` vacío; ✅ `/run/userDaily/{userDaily}/missing/list` | 30 |
| **R-FLD-05** | **Versiones viejas** de la definición acumuladas en el EM (`DEF_VER_*`) | 🟡 historial de versiones en tablas `DEF_VER_*` | 5 (higiene de BD) |
| **R-FLD-06** | **Folder de proyecto cerrado** (nombre con código de proyecto/ticket ya finalizado) | Cruce con el portafolio de proyectos | 15 |

### Familia C — Condiciones (la basura más peligrosa: es la que rompe o cuelga mallas)

| ID | Basura | Cómo se detecta | Peso |
|---|---|---|---|
| **R-CND-01** | **Out-condition huérfana**: un job la agrega (+) y **ningún** job la espera | 🟡 `DEF_LNKO_P` (SIGN='+') sin match en `DEF_LNKI_P` | 10 (higiene) |
| **R-CND-02** | **In-condition imposible**: un job espera una condición que **ningún** job produce → el job queda en *Wait Condition* para siempre | 🟡 `DEF_LNKI_P` sin match en `DEF_LNKO_P`; ✅ `/run/jobs/status?status=Wait Condition` | 35 |
| **R-CND-03** | **Condiciones antiguas acumuladas** en la tabla de condiciones (ODATE vieja, nunca borradas) | 🟠 `ctmcontb -LIST` (distribuido) / IOACND (z/OS); ✅ `GET /run/events` | 10 |
| **R-CND-04** | **Condiciones hacia/desde jobs eliminados** (quedaron colgando tras un borrado anterior) | Cruce de nombres de condición contra jobs vivos | 25 |
| **R-CND-05** | **Ciclos de condiciones** (A espera B, B espera A) | Grafo de dependencias (ArgOS ya tiene `dependency_graph.py`) — detección de ciclos | 30 |

### Familia D — Calendarios

| ID | Basura | Cómo se detecta | Peso |
|---|---|---|---|
| **R-CAL-01** | **Calendario no referenciado** por ningún job ni folder | ✅ `GET /deploy/calendars` vs. referencias en `/deploy/jobs` | 30 |
| **R-CAL-02** | **Calendario vencido** (sin días marcados para el año en curso ni el siguiente) | Contenido del calendario | 25 |
| **R-CAL-03** | **Rule-Based Calendar (RBC) sin uso** | ✅ `/deploy/jobs` + `/run/forecast/timeline?rbc=` | 25 |

### Familia E — Recursos, variables y eventos

| ID | Basura | Cómo se detecta | Peso |
|---|---|---|---|
| **R-RES-01** | **Recurso cuantitativo (pool) sin jobs que lo usen** | ✅ `GET /run/resources` vs. `Resources` en `/deploy/jobs` | 25 |
| **R-RES-02** | **Control resource (mutex/semáforo) sin uso** | idem (`resourceMutex`, `resourceSemaphore` en `/run/jobs/status`) | 25 |
| **R-RES-03** | **Variable global/pool no referenciada** (`%%VAR` que nadie lee) | ✅ `GET /run/variables` vs. texto de definiciones | 15 |
| **R-RES-04** | **Workload Policy inactiva o sin jobs** | ✅ `GET /run/workloadpolicies/detailed` | 20 |

### Familia F — Infraestructura

| ID | Basura | Cómo se detecta | Peso |
|---|---|---|---|
| **R-INF-01** | **Agente sin jobs asignados** | ✅ `/config/server/{server}/agents` vs. hosts en `/deploy/jobs` | 20 |
| **R-INF-02** | **Agente deshabilitado/desconectado con jobs** (los jobs no pueden correr) | ✅ `/config/server/{server}/agents` (estado) + ✅ `POST .../agent/{agent}/ping` | 30 |
| **R-INF-03** | **Hostgroup vacío o sin uso** | ✅ `GET /config/server/{server}/hostgroups` + `.../hostgroup/{hg}/agents` | 20 |
| **R-INF-04** | **Connection profile sin uso** (DB, FTP, SAP, Databricks…) | ✅ `GET /deploy/connectionprofiles/centralized` y `/local` vs. `ConnectionProfile` en jobs | 20 |
| **R-INF-05** | **Run-as user sin uso** | ✅ `GET /config/server/{server}/runasusers` vs. `RunAs` en jobs | 15 |
| **R-INF-06** | **Remote host / agentless host sin uso** | ✅ `/config/server/{server}/remotehosts`, `/agentlesshosts` | 15 |

### Familia G — Estado en ejecución (Active Jobs File / AJF — "lo que cuelga hoy")

| ID | Basura | Cómo se detecta | Peso |
|---|---|---|---|
| **R-RUN-01** | **Job en Wait eterno**: ODATE vieja (> 3 días) y sigue esperando condición/usuario/recurso | ✅ `GET /run/jobs/status?status=Wait Condition&orderDateTo=<hoy-3>` | 20 |
| **R-RUN-02** | **Job en HOLD por más de N días** (N=15) | ✅ `/run/jobs/status?held=true` + fecha de orden | 20 |
| **R-RUN-03** | **Ended Not OK abandonado**: falló y nadie lo reejecutó ni lo cerró en N días | ✅ `/run/jobs/status?status=Ended Not OK` + ODATE | 20 |
| **R-RUN-04** | **Job cíclico improductivo**: corre cada X minutos y siempre termina en segundos sin hacer nada | ✅ `cyclic=true` + estadísticas | 15 |
| **R-RUN-05** | **Jobs marcados "deleted" que siguen en el AJF** | ✅ `/run/jobs/status?deleted=true` | 10 |
| **R-RUN-06** | **Confirm pendiente** (Wait User) que nadie confirma | ✅ `status=Wait User` + ODATE vieja | 20 |

> Importante: la Familia G es **basura operacional** (se limpia sacando el job del día con
> `delete`/`setToOk`), no necesariamente basura de **definición**. Pero si el mismo job cae
> en G varios días seguidos, eso **suma evidencia** a las reglas A.

### Familia H — Históricos y datos (higiene de base de datos)

| ID | Basura | Cómo se detecta | Acción |
|---|---|---|---|
| **R-HIS-01** | Archive (Workload Archiving) con retención excedida | ✅ `GET /config/archive/statistics`, `/config/archive/rules` | ✅ `DELETE /config/archive/cleanup` con filtros y excepciones |
| **R-HIS-02** | `CMR_IOALOG` y logs crecidos sin purgar | 🟡 tamaño de tabla / 🟠 parámetros de retención (`ctm_menu`, `ctmdiskspace`) | Ajustar retención, no borrar a mano |
| **R-HIS-03** | Sysouts/outputs de jobs antiguos en los agentes | Filesystem de los agentes (`proclog`, `sysout`) | Purga por política |
| **R-HIS-04** | Reportes/netreports viejos | Directorios de reportes del EM | Purga por política |

### Familia I — Mainframe z/OS (específico dominio TDC de BCI)

| ID | Basura | Cómo se detecta | Peso |
|---|---|---|---|
| **R-ZOS-01** | **Miembro JCL huérfano**: existe en `EXPOD.SPC20.JCLLIB` pero **ningún** job de Control-M lo referencia en `MEMNAME` | Listado de miembros de la librería (ISPF 3.4 / `LISTDS ... MEMBERS`, extractor py3270 de ArgOS) vs. `MEMNAME` de todas las definiciones | 30 |
| **R-ZOS-02** | **Programa COBOL huérfano** (`CCTSF.PASO`): ningún JCL hace `EXEC PGM=` sobre él | Parser JCL de ArgOS (`malla_catalog._agregar_steps_jcl`) sobre todos los JCL vivos | 25 |
| **R-ZOS-03** | **Tarjeta SORT huérfana** en `EXPOD.SORTLIB` (ningún `SYSIN DD DSN=EXPOD.SORTLIB(xxx)`) | Parser JCL | 20 |
| **R-ZOS-04** | **Dataset que se escribe y nadie lee** (DSN solo con `DISP=(NEW,CATLG)` y sin lectores en ningún JCL) | Grafo DSN productor→consumidor (ArgOS ya arma aristas `lee`/`escribe`) | 20 |
| **R-ZOS-05** | **GDG con generaciones que nadie consume** | Grafo DSN + catálogo | 15 |
| **R-ZOS-06** | **Tablas Datacom sin acceso** (el dominio TDC usa Datacom, no DB2) | Cruce programas COBOL ↔ tablas Datacom vs. programas vivos | 15 |

> La Familia I es donde ArgOS tiene **ventaja real frente a las herramientas de BMC**:
> Control-M no sabe si el JCL o el COBOL al que apunta está vivo o si su dataset de salida
> lo lee alguien; ArgOS sí, porque parsea JCL y COBOL. Es el argumento comercial más fuerte.

---

## 2. Puntaje de basura (Índice de Basura 0–100) 🔵 CRITERIO

### 2.1 Fórmula

```
score_bruto(job) = Σ peso(regla) para cada regla que dispara
score(job)       = min(100, score_bruto × factor_confianza × factor_exclusion)

factor_confianza = 1.0  si la historia de ejecución cubre ≥ 13 meses
                   0.6  si cubre entre 3 y 13 meses
                   0.3  si cubre < 3 meses   (no se puede opinar sobre jobs anuales)

factor_exclusion = 0    si el job está en la LISTA DE PROTECCIÓN (2.3)
                   1    en otro caso
```

**¿Por qué 13 meses?** Porque existen jobs **anuales** (cierre contable, cierre fiscal, informes
regulatorios CMF de fin de año, recálculo anual de tasas/cupos de tarjeta). Con menos de un año
de historia, un job anual es indistinguible de uno muerto. Este es el falso positivo número uno
en limpiezas de Control-M.

### 2.2 Bandas → estados de ArgOS (ya existen en `malla_catalog.ESTADOS_LIMPIEZA`)

| Score | Estado ArgOS | Qué significa | Qué se hace |
|---|---|---|---|
| 0–29 | `activa` | Sin hallazgos relevantes | Nada |
| 30–59 | `en_observacion` | Hay señales, pero no alcanzan | Se revisa en el siguiente ciclo; se puede pedir info al dueño |
| 60–79 | `candidata_limpieza` | Evidencia suficiente para proponer | Entra al flujo de consenso (sección 3) |
| 80–100 | `candidata_limpieza` + prioridad alta | Evidencia fuerte y de varias fuentes | Idem, primero en la cola |
| — | `suspendida` | En cuarentena, reversible | Sección 3, paso 6 |
| — | `en_limpieza` | Respaldada, a la espera de la eliminación final | Sección 3, paso 7 |
| — | `limpiada` | Eliminada de Control-M, con respaldo y bitácora | Fin |

### 2.3 Lista de protección (nunca se proponen, aunque el puntaje dé 100)

1. Jobs marcados como **contingencia / DR / recuperación** (por folder, prefijo o etiqueta acordada con BCI).
2. Jobs **regulatorios** (CMF, SII, UAF, Banco Central) aunque corran una vez al año.
3. Jobs **on-demand** documentados (los ordena otro sistema, un operador o `DO FORCEJOB`).
4. Jobs que son **destino de `DO FORCEJOB`/`Run Job`** de otro job vivo.
5. Jobs con **ventana de congelamiento** (freeze de fin de año, cambios en curso).
6. Jobs creados hace **< 180 días** (proyecto en implantación).
7. Todo lo que el dueño de la aplicación haya marcado como "no tocar" en una revisión anterior (esa respuesta queda guardada y se respeta en los ciclos siguientes, con fecha de vencimiento).

### 2.4 Regla de las dos fuentes

Una propuesta de limpieza **exige al menos 2 fuentes independientes** que coincidan. Por ejemplo:

- *Definición* (`/deploy/jobs`) **+** *historia* (`CMR_RUNINF`/Archive) **+** *forecast* (`/run/forecast/timeline`), o
- *Definición* **+** *z/OS* (JCL inexistente en la librería).

Una sola fuente (p. ej. "no corrió en 90 días") deja el job como máximo en `en_observacion`.

---

## 3. Proceso de consenso: cómo se llega a "esto es basura y se elimina"

Es el proceso manual que describió BCI, formalizado. Cada paso deja registro (ArgOS ya lo hace
en `historial_limpieza` y `bitacora_limpieza.json`: *nunca una baja silenciosa*).

```
 1. EXTRAER       →  2. DETECTAR     →  3. PUNTUAR      →  4. DICTAMEN IA
 (API/SQL/z/OS,       (reglas R-*,        (índice 0-100,     (explica la evidencia,
  solo lectura)        deterministas)      protección)        no decide)
        ↓
 5. REVISIÓN DUEÑO  →  6. CUARENTENA   →  7. RESPALDO +    →  8. VERIFICACIÓN
 (dueño de aplicación   (30-90 días,        ELIMINACIÓN        (nada se rompió,
  + control batch)       reversible)         (orden de cambio)  bitácora cerrada)
```

### Paso 1 — Extraer (solo lectura)
Usuario de servicio **de solo lectura** en Control-M (perfil de consulta, sin permisos de
deploy ni delete). Se extrae: definiciones completas (`/deploy/jobs?format=json&useArrayFormat=true`),
folders, calendarios, recursos, agentes, estado activo, historia (Archive/CMR_RUNINF), forecast
y, en z/OS, los miembros de las librerías.

### Paso 2 — Detectar
Reglas deterministas de la sección 1 (código, no IA). Salida: lista de hallazgos
`{job, regla, evidencia, fuente, fecha}`.

### Paso 3 — Puntuar
Fórmula de la sección 2 + lista de protección + regla de las dos fuentes.

### Paso 4 — Dictamen IA (RAG)
El LLM recibe los hallazgos, la definición del job, su vecindario en el grafo (quién lo
dispara, a quién dispara) y este documento como contexto recuperado. Produce un **dictamen
explicado** en lenguaje de negocio: *"TDCOD123 no se ejecuta desde el 2025-03-02; su JCL no
existe en EXPOD.SPC20.JCLLIB; la condición TDCOD123-OK no la espera nadie; forecast vacío 24
meses. Riesgo de eliminar: bajo. Impacto aguas abajo: ninguno."* **El LLM no cambia el
puntaje ni el estado.** Si el LLM contradice las reglas, se marca para revisión humana.

### Paso 5 — Revisión del dueño
ArgOS genera el correo/ticket al **dueño de la aplicación** (ya existe
`solicitudes_malla.borrador_correo` como patrón). Respuestas posibles:
`aprobar` / `rechazar (proteger)` / `necesita más info`. El rechazo alimenta la lista de
protección 2.3 (punto 7).

### Paso 6 — Cuarentena (reversible) — **nunca borrar directo**
Opciones, de menos a más invasiva:

| Opción | Cómo | Reversión |
|---|---|---|
| a) Sacar del scheduling | Redeploy del job con `"When": {"Schedule": "Never"}` 🟠 (validar el valor soportado en la versión de BCI con `ctm build`) o calendario vacío | Redeploy de la definición respaldada |
| b) Mover a folder de cuarentena | Folder `ZZ_CUARENTENA_<AAAAMM>` sin reglas de orden | Mover de vuelta |
| c) Hold del job ordenado | ✅ `POST /run/job/{jobId}/hold` | ✅ `POST /run/job/{jobId}/free` |

Duración recomendada: **30 días** para diarios/semanales, **1 ciclo completo** (hasta 13 meses)
si hay duda de que sea mensual/anual. Si nadie reclama en la cuarentena, pasa al paso 7.

### Paso 7 — Respaldo + eliminación (con orden de cambio)
1. Respaldo **obligatorio**: ✅ `GET /deploy/jobs?folder=<f>&format=json` y `format=xml` → se guarda
   en `data/<entorno>/respaldos_limpieza/<fecha>/<folder>.json|.xml` + commit git.
2. Eliminación: ✅ `DELETE /deploy/job/{jobPath}?server=<ctm>` (job) o
   ✅ `DELETE /deploy/folder/{server}/{folderName}` (folder completo) o
   ✅ `DELETE /deploy/subfolder/{subFolderPath}?server=`.
3. Limpieza asociada: calendarios (✅ `DELETE /deploy/calendar/{name}`), recursos
   (✅ `DELETE /run/resource/{server}/{name}`), variables (✅ `DELETE /run/variables/{server}`),
   eventos/condiciones (✅ `DELETE /run/event/{server}/{name}/{date}`), connection profiles
   (✅ `DELETE /deploy/connectionprofile/...`).
4. z/OS: mover el miembro JCL a una librería de retirados (`EXPOD.SPC20.JCLLIB.RETIRADOS`),
   no borrarlo. Lo ejecuta el equipo de BCI con su proceso de cambio.

### Paso 8 — Verificación
- Al día siguiente del New Day: ✅ `/run/jobs/status?status=Wait Condition` no debe mostrar jobs
  nuevos esperando condiciones que producía algo eliminado.
- ✅ `GET /usage/jobs` antes/después → **métrica de valor** para el informe (jobs eliminados,
  % de reducción, licencias liberadas si BCI paga por job).
- Bitácora cerrada con: quién aprobó, número de orden de cambio, respaldo, resultado.

### RACI mínimo 🔵

| Actividad | ArgOS (IA) | Analista AIWIS | Dueño aplicación BCI | Control Batch / Operación BCI | Gestión de cambios BCI |
|---|---|---|---|---|---|
| Detectar y puntuar | **R** | A | I | I | — |
| Dictamen | **R** | A | I | C | — |
| Aprobar eliminación | — | C | **A/R** | C | I |
| Cuarentena | C | R | I | **A** | I |
| Eliminar en producción | — | C | I | **R** | **A** |
| Verificar | **R** | A | I | C | I |

---

## 4. Comandos: Control-M Automation API (REST + CLI `ctm`)

### 4.1 Autenticación ✅

```bash
# Opción A — API Key (recomendada para servicios; Control-M 9.0.21+ / SaaS)
curl -H "x-api-key: $CTM_API_KEY" "$CTM_ENDPOINT/automation-api/deploy/folders?server=*&folder=*"

# Opción B — sesión con usuario/clave → token Bearer
TOKEN=$(curl -s -k -X POST "$CTM_ENDPOINT/automation-api/session/login" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$CTM_USER\",\"password\":\"$CTM_PASSWORD\"}" | jq -r .token)
curl -k -H "Authorization: Bearer $TOKEN" "$CTM_ENDPOINT/automation-api/..."
curl -k -X POST -H "Authorization: Bearer $TOKEN" "$CTM_ENDPOINT/automation-api/session/logout"

# CLI ctm (mismo backend REST)
ctm environment add bci_ro "$CTM_ENDPOINT/automation-api" "$CTM_USER" "$CTM_PASSWORD"
ctm environment set bci_ro
ctm session login
```

`$CTM_ENDPOINT` típico: `https://<host-em>:8443`. **Nunca** hardcodear: van en `.env`
(regla de CLAUDE.md).

### 4.2 Extracción (lectura) — mapa regla → endpoint ✅

| Para qué | REST | CLI `ctm` |
|---|---|---|
| Todas las definiciones (JSON) | `GET /deploy/jobs?ctm=*&folder=*&format=json&useArrayFormat=true` | `ctm deploy jobs::get -s "ctm=*&folder=*"` |
| Definiciones en XML (respaldo) | `GET /deploy/jobs?...&format=xml` | `ctm deploy jobs::get xml -s "..."` |
| Folders por aplicación | `GET /deploy/folders?server=*&folder=*&application=TDC*` | `ctm deploy folders::get -s "..."` |
| z/OS: filtrar por librería | parámetro `library=` en `/deploy/jobs` y `/deploy/folders` | idem |
| Calendarios | `GET /deploy/calendars?server=*&name=*` | `ctm deploy calendars::get -s "..."` |
| Estado del día (AJF) | `GET /run/jobs/status?limit=10000&folder=...&status=...&held=...&deleted=...&cyclic=...&orderDateFrom=...&orderDateTo=...` | `ctm run jobs:status::get -s "..."` |
| Log / output de un job | `GET /run/job/{jobId}/log`, `/output` | `ctm run job:log::get <jobId>` |
| Estadísticas de un job | `GET /run/job/{jobId}/statistics` | `ctm run job:statistics::get <jobId>` |
| Por qué espera | `GET /run/job/{jobId}/waitingInfo` | `ctm run job:waitingInfo::get <jobId>` |
| **Historia larga (Archive)** | `GET /archive/search?jobname=&folder=&fromTime=YYYY-MM-DD&toTime=&status=&numberOfRuns=&limit=` | `ctm archive jobs::get -s "..."` |
| **Forecast 24 meses** (R-DEF-03) | `GET /run/forecast/timeline?ctm=&folder=&jobs=*&filterType=relativeMonths&from=-24&to=24` → `pollId` → `GET /run/forecast/timeline/poll/{poll}` | 🟠 `ctm run forecast...` (validar nombre del subcomando) |
| Jobs no ordenados de un User Daily | `GET /run/userDaily/{userDaily}/missing/list?server=` | 🟠 |
| Recursos | `GET /run/resources?ctm=*&name=*` | `ctm run resources::get -s "..."` |
| Eventos/condiciones | `GET /run/events?ctm=*&name=*&date=*&limit=` | `ctm run events::get -s "..."` |
| Variables | `GET /run/variables?server=&pool=&variable=` | 🟠 |
| Servidores | `GET /config/servers` | `ctm config servers::get` |
| Agentes | `GET /config/server/{server}/agents` | `ctm config server:agents::get <server>` |
| Hostgroups | `GET /config/server/{server}/hostgroups` | `ctm config server:hostgroups::get <server>` |
| Run-as users | `GET /config/server/{server}/runasusers` | `ctm config server:runasusers::get <server>` |
| Connection profiles | `GET /deploy/connectionprofiles/centralized?type=*&name=*`, `/deploy/connectionprofiles/local` | `ctm deploy connectionprofiles:centralized::get -t "*"` |
| Workload policies | `GET /run/workloadpolicies/detailed` | `ctm run workloadpolicies::get` |
| Conteo de jobs (métrica de valor) | `GET /usage/jobs` | 🟠 |
| Reportes del EM (CSV) | `POST /reporting/report` body `{"name":"<reporte>","format":"CSV"}` → `GET /reporting/status/{reportId}` → `GET /reporting/download?...` | `ctm reporting report::get "<reporte>" -o CSV` |
| Estadísticas del Archive | `GET /config/archive/statistics` | 🟠 |

> Tip: crear en el EM un reporte compartido "ArgOS_JobDef_Completo" (tipo *Job Definition*
> con todas las columnas) y bajarlo por `POST /reporting/report` con `"name":"shared:ArgOS_JobDef_Completo"`.
> Es la forma más barata de sacar TODO el inventario de una vez sin tocar la base de datos.

### 4.3 Acciones (escritura) — solo en pasos 6 y 7, con aprobación ✅

| Acción | REST | CLI |
|---|---|---|
| Validar un JSON sin desplegar | `POST /build` (multipart `definitionsFile`) | `ctm build <archivo.json>` |
| Transformar con Deploy Descriptor | `POST /deploy/transform` (`definitionsFile` + `deployDescriptorFile`) | `ctm deploy transform <def.json> <descriptor.json>` |
| Desplegar (reprogramar) | `POST /deploy` (`definitionsFile` JSON/XML/zip + `deployDescriptorFile` opcional) | `ctm deploy <def.json> [descriptor.json]` |
| Hold / Free | `POST /run/job/{jobId}/hold` / `/free` | `ctm run job::hold <jobId>` / `job::free` |
| Borrar job del día (AJF) | `POST /run/job/{jobId}/delete` / `/undelete` | `ctm run job::delete <jobId>` |
| Set to OK / Rerun / Kill | `POST /run/job/{jobId}/setToOk` / `/rerun` / `/kill` | `ctm run job::setToOk <jobId>` |
| Ordenar un folder | `POST /run/order` body `{"ctm":"","folder":"","jobs":""}` | `ctm run order <ctm> <folder> [jobs]` |
| **Eliminar job (definición)** | `DELETE /deploy/job/{jobPath}?server=<ctm>` | 🟠 `ctm deploy job::delete ...` |
| **Eliminar folder** | `DELETE /deploy/folder/{server}/{folderName}` | `ctm deploy folder::delete <server> <folder>` |
| Eliminar sub-folder | `DELETE /deploy/subfolder/{subFolderPath}?server=` | 🟠 |
| Eliminar calendario | `DELETE /deploy/calendar/{calendarName}?server=&type=` | 🟠 |
| Eliminar recurso | `DELETE /run/resource/{server}/{name}` | `ctm run resource::delete <server> <name>` |
| Eliminar evento/condición | `DELETE /run/event/{server}/{name}/{date}` | `ctm run event::delete <server> <name> <date>` |
| Eliminar variables | `DELETE /run/variables/{server}` | 🟠 |
| Eliminar connection profile | `DELETE /deploy/connectionprofile/centralized/{type}/{name}` | 🟠 |
| Deshabilitar / eliminar agente | `POST /config/server/{server}/agent/{agent}/disable` / `DELETE .../agent/{agent}` | `ctm config server:agent::disable ...` |
| Purgar Archive | `DELETE /config/archive/cleanup?folder=&jobname=&jobStatus=&...Exceptions=` | 🟠 |

### 4.4 Ejemplos curl listos (lectura)

```bash
H="x-api-key: $CTM_API_KEY"; B="$CTM_ENDPOINT/automation-api"

# Inventario completo TDC en JSON (arrays siempre)
curl -sk -H "$H" "$B/deploy/jobs?ctm=*&folder=TDC*&format=json&useArrayFormat=true" > defs_tdc.json

# Jobs esperando condición desde hace más de 3 días
curl -sk -H "$H" "$B/run/jobs/status?limit=10000&status=Wait%20Condition&orderDateTo=$(date -d '-3 day' +%y%m%d)"

# Jobs en HOLD
curl -sk -H "$H" "$B/run/jobs/status?limit=10000&held=true"

# Historia de un job en el Archive (últimos 13 meses)
curl -sk -H "$H" "$B/archive/search?jobname=TDCOD123&fromTime=$(date -d '-13 month' +%F)&toTime=$(date +%F)&limit=500"

# Forecast 24 meses de un folder (asíncrono)
POLL=$(curl -sk -H "$H" "$B/run/forecast/timeline?ctm=CTMPROD&folder=TDC_DIARIO&jobs=*&filterType=relativeMonths&from=-24&to=24" | jq -r '.pollId // .poll')
curl -sk -H "$H" "$B/run/forecast/timeline/poll/$POLL"

# Agentes y su estado
curl -sk -H "$H" "$B/config/server/CTMPROD/agents"
```

> Formato de `orderDateFrom/To` y de estados (`Ended OK`, `Ended Not OK`, `Wait Condition`,
> `Wait User`, `Wait Resource`, `Wait Host`, `Executing`…): validar contra la instalación de BCI
> con un par de llamadas de prueba; la documentación de BMC los describe, pero el Swagger no
> los enumera.

---

## 5. Utilitarios de línea de comando (cuando no hay API o para validar) 🟠

### 5.1 Control-M/Server distribuido (Unix/Windows)

| Utilitario | Uso en limpieza |
|---|---|
| `ctmpsm` | Menú/listados del Active Jobs File (jobs del día, estado, hold). Útil para R-RUN-* |
| `ctmcontb -LIST "*" "*"` / `-DELETE <cond> <odate>` / `-DELETEALL`(con filtros) | Listar y borrar condiciones (R-CND-03) |
| `ctmruninf -list <from> <to> -JOBNAME <j>` | Historia de ejecuciones (lee `CMR_RUNINF`) → R-DEF-01/02/13 |
| `ctmjsa` | Acumulación de estadísticas de jobs |
| `ctmlog listjob ...` / `ctmlog list ...` | Log del servidor (quién forzó OK, reruns) → R-DEF-14 |
| `ctmvar -action list` | Variables → R-RES-03 |
| `ctmloadset`, `ctmldnrs` | Recursos cuantitativos y condiciones en lote |
| `ctmudly`, `ctmorder` | Ordenamiento manual (útil para probar cuarentena) |
| `ctmwhy <orderno>` | Por qué un job no arranca → R-RUN-01 |
| `ctmdefine` / `ctmcpt` | Definir/copiar tablas (reprogramación rápida) |
| `ctm_menu` → Database Maintenance | Purga de históricos y logs (Familia H) |

### 5.2 Control-M/Enterprise Manager

| Utilitario | Uso |
|---|---|
| `emdef exportdeffolder -u <u> -p <p> -s <em> -arg <args.xml> -out <out.xml>` | Respaldo XML de folders (paso 7) |
| `emdef exportdefjob ...` | Respaldo/extracción de jobs por filtro |
| `emdef deldefjob ...` | Borrado de definiciones por filtro (alternativa a la API) |
| `emdef deffolder ... -src <xml>` | Re-importar (rollback) |
| `emdef updatedef ...` | Cambios masivos de atributos (reprogramar calendarios, hosts) |
| `emdef duplicatedefjob ...` | Duplicar para pruebas |
| `emreportcli` | Generar reportes del EM por línea de comando |

### 5.3 Control-M for z/OS (IOA) — dominio TDC

| Utilitario | Uso |
|---|---|
| `CTMRPLN` | Plan de jobs por calendario (qué días corre cada job) → R-DEF-03 en mainframe |
| `CTMRFLW` | Reporte de flujo de dependencias (condiciones in/out) → Familia C |
| `CTMRNSC` | Reporte del night schedule |
| `CTMRAFL` | Flujo del Active Jobs File |
| `CTMJSA` | Estadísticas de ejecución |
| `IOACND` | Listar/agregar/borrar condiciones IOA → R-CND-03 |
| `CTMTBUPD` / `CTMBLT` | Actualizar/crear tablas por lote (reprogramar) |
| IOA Log (pantalla 5) / `IOALOG` | Quién forzó, holdeó, borró → R-DEF-14 |
| ISPF 3.4 / TSO `LISTDS 'EXPOD.SPC20.JCLLIB' MEMBERS` | Inventario de miembros JCL → R-ZOS-01 |
| SuperC (ISRSUPC) search-for `EXEC PGM=` | Programas referenciados → R-ZOS-02 |

> En BCI el extractor de ArgOS (Capa 3, py3270) debería automatizar justamente estas pantallas
> y reportes IOA cuando no haya Automation API disponible para el Control-M de z/OS.

---

## 6. Queries SQL de limpieza 🟡

Escritas para **Oracle** (la base del Control-M de BCI, según `src/backend/db_motor.py`).
Corren con un **usuario de solo lectura**. **Antes de usarlas:** validar nombres de tablas y
columnas contra el *Physical Data Model* de la versión instalada (BMC lo publica por versión).
Las marcadas `DEF_*` son del **EM**; las `CMS_*`/`CMR_*` son del **Control-M/Server**.

### 6.1 Control-M/Server

```sql
-- Q-01 · R-DEF-01/02 · Jobs definidos y su última ejecución exitosa conocida
-- OJO: CMR_RUNINF guarda solo las últimas ~20 corridas EXITOSAS por job.
--      Ausencia de filas = nunca terminó OK *o* la historia se purgó. Cruzar con Archive.
SELECT d.SCHEDTAB            AS folder,
       d.JOBNAME,
       d.MEMNAME,
       d.NODEID              AS host,
       d.CREATIONDATETIME,
       d.CHANGEDATETIME,
       MAX(r.TIMESTMP)       AS ultima_corrida_ok,
       COUNT(r.JOBNAME)      AS corridas_registradas
FROM   CMS_JOBDEF d
LEFT JOIN CMR_RUNINF r
       ON r.JOBNAME = d.JOBNAME AND r.MEMNAME = d.MEMNAME AND r.NODEID = d.NODEID
GROUP BY d.SCHEDTAB, d.JOBNAME, d.MEMNAME, d.NODEID, d.CREATIONDATETIME, d.CHANGEDATETIME
HAVING MAX(r.TIMESTMP) IS NULL
    OR MAX(r.TIMESTMP) < TO_CHAR(SYSDATE - 90, 'YYYYMMDDHH24MISS')
ORDER BY ultima_corrida_ok NULLS FIRST;
```

```sql
-- Q-02 · R-DEF-13 · Jobs que siempre terminan en menos de 2 segundos
SELECT JOBNAME, MEMNAME, NODEID,
       COUNT(*)              AS corridas,
       MAX(ELAPTIME)         AS max_elapsed,
       AVG(ELAPTIME)         AS avg_elapsed
FROM   CMR_RUNINF
GROUP BY JOBNAME, MEMNAME, NODEID
HAVING COUNT(*) >= 5 AND MAX(ELAPTIME) < 200     -- ELAPTIME en centésimas: validar unidad
ORDER BY corridas DESC;
```

```sql
-- Q-03 · R-RUN-01/02/03 · Jobs del AJF colgados (ODATE vieja)
-- STATE/STATUS/HOLDFLAG determinan el estado: mapear códigos con la doc de la versión.
SELECT ODATE, ORDERNO, JOBNAME, MEMNAME, STATE, STATUS, HOLDFLAG, STARTRUN, ENDRUN, OSCOMPMSG
FROM   CMR_AJF
WHERE  ODATE < TO_CHAR(SYSDATE - 3, 'YYYYMMDD')
ORDER BY ODATE;
```

```sql
-- Q-04 · R-DEF-14 · Jobs forzados a mano (IOA log)
-- Identificar primero los MSGID de "force OK / set to OK / bypass" en la versión instalada
-- (5133 = ended OK, 5134 = ended not OK son los más citados).
SELECT JOBNAME, COUNT(*) AS veces
FROM   CMR_IOALOG
WHERE  MSGID IN (:msgid_force_ok, :msgid_bypass)
  AND  LOGDATE >= TO_CHAR(SYSDATE - 365, 'YYYYMMDD')
GROUP BY JOBNAME
ORDER BY veces DESC;
```

```sql
-- Q-05 · Auditoría: quién creó / cambió por última vez (para encontrar al dueño, R-DEF-12)
SELECT JOBNAME, CREATIONUSERID, CREATIONDATETIME, CHANGEUSERID, CHANGEDATETIME
FROM   CMS_JOBDEF
ORDER BY CHANGEDATETIME;
```

### 6.2 Enterprise Manager (definiciones y condiciones)

```sql
-- Q-10 · R-CND-02 · IN-conditions que ningún job produce (espera eterna)
SELECT t.TABLE_NAME AS folder, j.JOB_NAME, i.CONDITION
FROM   DEF_LNKI_P i
JOIN   DEF_JOB    j ON j.TABLE_ID = i.TABLE_ID AND j.JOB_ID = i.JOB_ID
JOIN   DEF_TABLES t ON t.TABLE_ID = j.TABLE_ID
WHERE  NOT EXISTS (
         SELECT 1 FROM DEF_LNKO_P o
         WHERE  o.CONDITION = i.CONDITION
           AND  o.SIGN = '+')                 -- '+' agrega, '-' borra
ORDER BY folder, j.JOB_NAME;
```

```sql
-- Q-11 · R-CND-01 · OUT-conditions que nadie espera (huérfanas)
SELECT t.TABLE_NAME AS folder, j.JOB_NAME, o.CONDITION
FROM   DEF_LNKO_P o
JOIN   DEF_JOB    j ON j.TABLE_ID = o.TABLE_ID AND j.JOB_ID = o.JOB_ID
JOIN   DEF_TABLES t ON t.TABLE_ID = j.TABLE_ID
WHERE  o.SIGN = '+'
  AND  NOT EXISTS (SELECT 1 FROM DEF_LNKI_P i WHERE i.CONDITION = o.CONDITION)
ORDER BY folder, j.JOB_NAME;
```

```sql
-- Q-12 · R-FLD-01 · Folders sin jobs
SELECT t.TABLE_NAME, t.DATA_CENTER
FROM   DEF_TABLES t
WHERE  NOT EXISTS (SELECT 1 FROM DEF_JOB j WHERE j.TABLE_ID = t.TABLE_ID);
```

```sql
-- Q-13 · R-DEF-08 · Duplicados (mismo miembro + librería + host)
SELECT j.MEM_NAME, j.MEM_LIB, j.NODE_ID, COUNT(*) AS copias,
       LISTAGG(t.TABLE_NAME || '/' || j.JOB_NAME, ', ') WITHIN GROUP (ORDER BY j.JOB_NAME) AS jobs
FROM   DEF_JOB j JOIN DEF_TABLES t ON t.TABLE_ID = j.TABLE_ID
GROUP BY j.MEM_NAME, j.MEM_LIB, j.NODE_ID
HAVING COUNT(*) > 1
ORDER BY copias DESC;
```

```sql
-- Q-14 · R-DEF-09 · Nombres de prueba/obsoletos en producción
SELECT t.TABLE_NAME, j.JOB_NAME, j.DESCRIPTION
FROM   DEF_JOB j JOIN DEF_TABLES t ON t.TABLE_ID = j.TABLE_ID
WHERE  REGEXP_LIKE(j.JOB_NAME || ' ' || NVL(j.DESCRIPTION,''),
                   '(TEST|PRUEBA|_OLD|_BKP|BACKUP|COPIA|_TMP|TEMP|OBSOLET|NO USAR|BORRAR)', 'i');
```

```sql
-- Q-15 · R-DEF-11 · Nomenclatura TDC inválida (8 caracteres: TDCO + D/E/M + 3)
SELECT t.TABLE_NAME, j.JOB_NAME, j.MEM_NAME
FROM   DEF_JOB j JOIN DEF_TABLES t ON t.TABLE_ID = j.TABLE_ID
WHERE  j.APPLICATION LIKE 'TDC%'
  AND  NOT REGEXP_LIKE(j.MEM_NAME, '^TDCO[DEM][A-Z0-9]{3}$');
```

```sql
-- Q-16 · R-CAL-01 · Calendarios no referenciados
-- (la tabla de calendarios y las columnas de calendario en DEF_JOB — DAYS_CAL, WEEKS_CAL,
--  CONF_CAL — varían por versión; validar)
SELECT c.CALENDAR
FROM   DF_CALENDARS c        -- nombre a validar según versión
WHERE  NOT EXISTS (SELECT 1 FROM DEF_JOB j
                   WHERE c.CALENDAR IN (j.DAYS_CAL, j.WEEKS_CAL, j.CONF_CAL));
```

> Recomendación fuerte: **preferir la API sobre el SQL** siempre que se pueda. La API es un
> contrato estable y soportado por BMC; el esquema de la base no lo es, y a un banco le
> cuesta mucho más aprobar acceso directo a la base de Control-M que un usuario de API
> de solo lectura. El SQL queda como plan B y para validar.

---

## 7. Reprogramación de mallas (código real, Jobs-as-Code)

### 7.1 Poner un job en cuarentena (sacarlo del scheduling sin borrarlo)

```json
{
  "TDC_DIARIO": {
    "Type": "Folder",
    "ControlmServer": "CTMPROD",
    "TDCOD123": {
      "Type": "Job:Command",
      "Description": "[ARGOS-CUARENTENA 2026-09-24 hasta 2026-10-24] candidato R-DEF-01,R-DEF-05 score 85",
      "When": { "Schedule": "Never" }
    }
  }
}
```

> Se despliega con `ctm build` → `ctm deploy`. **Validar** que `"Schedule": "Never"` exista en la
> versión de BCI; si no, usar un calendario vacío `ARGOS_NUNCA` o mover el job al folder
> `ZZ_CUARENTENA_202609`.
> Ojo: el deploy de un folder **reemplaza el folder completo**. Siempre partir del JSON
> extraído con `GET /deploy/jobs` y modificar solo el job, nunca armar el folder a mano.

### 7.2 Deploy Descriptor: cambios masivos sin tocar el JSON original

Repuntar todos los jobs de un agente retirado a uno nuevo (R-DEF-06/R-INF-02) y marcar la
descripción:

```json
{
  "DeployDescriptor": [
    {
      "ApplyOn": { "@": "TDC_*", "Type": "Job:*" },
      "Property": "Host",
      "Replace": [ { "srvtdc01old": "srvtdc05" } ]
    },
    {
      "ApplyOn": { "@": "TDC_*" },
      "Property": "Description",
      "Replace": [ { "^(.*)$": "$1 [ARGOS: host migrado 2026-09]" } ]
    }
  ]
}
```

```bash
ctm deploy transform defs_tdc.json descriptor_host.json > defs_tdc_nuevo.json   # ver el resultado
ctm build  defs_tdc_nuevo.json                                                  # validar
ctm deploy defs_tdc.json descriptor_host.json                                   # aplicar
```

### 7.3 Consolidar duplicados (R-DEF-08)

1. Elegir el job "sobreviviente" (el que tiene más consumidores de sus out-conditions).
2. Redirigir las in-conditions de los consumidores del duplicado hacia las out-conditions del
   sobreviviente (edición del JSON extraído).
3. Duplicado → cuarentena (7.1) → eliminación en el paso 7.

### 7.4 Reparar una in-condition imposible (R-CND-02) en vez de borrar

Si el dueño confirma que el job **sí** debe correr: cambiar la condición esperada por la que
produce el predecesor real (o quitar la condición si el predecesor ya no existe):

```json
"TDCOD200": {
  "Type": "Job:Command",
  "WaitForEvents": [ { "Event": "TDCOD123-ENDED-OK" } ]
}
```

### 7.5 Rollback

```bash
# El respaldo del paso 7 es el mismo JSON/XML que devolvió GET /deploy/jobs
ctm deploy data/real/respaldos_limpieza/2026-09-24/TDC_DIARIO.json
# o con emdef:
emdef deffolder -u $U -p $P -s $EM -src TDC_DIARIO.xml
```

---

## 8. Código Python de referencia (adaptador real + motor de reglas)

Encaja en ArgOS respetando el ADR 002: **misma interfaz** que `db_motor.py` (simulado),
seleccionada por `config.es_real()`. Credenciales solo desde `.env`.

### 8.1 Adaptador real — `src/backend/controlm_real.py` (propuesta)

```python
"""
ArgOS — Adaptador REAL de Control-M (Automation API).

Contraparte de db_motor.py (simulado). Solo se instancia si config.es_real().
Credenciales: CTM_ENDPOINT, CTM_API_KEY (o CTM_USER/CTM_PASSWORD) desde .env.
Por defecto SOLO LECTURA: los métodos de escritura exigen allow_write=True y una
orden de cambio aprobada.
"""
from __future__ import annotations

import os
import time
import requests

class ControlMRealAdapter:
    def __init__(self, allow_write: bool = False, timeout: int = 60):
        self.base = os.environ["CTM_ENDPOINT"].rstrip("/") + "/automation-api"
        self.allow_write = allow_write
        self.timeout = timeout
        self.s = requests.Session()
        self.s.verify = os.getenv("CTM_CA_BUNDLE", True)   # nunca verify=False en producción
        if os.getenv("CTM_API_KEY"):
            self.s.headers["x-api-key"] = os.environ["CTM_API_KEY"]
        else:
            r = self.s.post(f"{self.base}/session/login", json={
                "username": os.environ["CTM_USER"], "password": os.environ["CTM_PASSWORD"]},
                timeout=self.timeout)
            r.raise_for_status()
            self.s.headers["Authorization"] = f"Bearer {r.json()['token']}"

    # ---------------- lectura ----------------
    def _get(self, path: str, **params):
        r = self.s.get(f"{self.base}{path}", params=params, timeout=self.timeout)
        r.raise_for_status()
        return r.json()

    def job_definitions(self, ctm="*", folder="*"):
        return self._get("/deploy/jobs", ctm=ctm, folder=folder, format="json", useArrayFormat="true")

    def folders(self, server="*", folder="*", application=None):
        return self._get("/deploy/folders", server=server, folder=folder, application=application)

    def active_jobs(self, **filters):
        return self._get("/run/jobs/status", limit=filters.pop("limit", 10000), **filters)

    def archive_runs(self, jobname: str, from_date: str, to_date: str, limit=500):
        return self._get("/archive/search", jobname=jobname, fromTime=from_date, toTime=to_date, limit=limit)

    def forecast(self, ctm: str, folder: str, jobs="*", months_back=-24, months_fwd=24, wait_s=120):
        start = self._get("/run/forecast/timeline", ctm=ctm, folder=folder, jobs=jobs,
                          filterType="relativeMonths", **{"from": months_back, "to": months_fwd})
        poll = start.get("pollId") or start.get("poll")
        deadline = time.time() + wait_s
        while time.time() < deadline:
            res = self._get(f"/run/forecast/timeline/poll/{poll}")
            if res.get("status", "").lower() in ("completed", "done", "") and "jobs" in res:
                return res
            time.sleep(3)
        raise TimeoutError(f"forecast {folder} no terminó en {wait_s}s")

    def agents(self, server: str):
        return self._get(f"/config/server/{server}/agents")

    def calendars(self, server="*"):
        return self._get("/deploy/calendars", server=server, name="*")

    def resources(self, ctm="*"):
        return self._get("/run/resources", ctm=ctm, name="*")

    def usage(self):
        return self._get("/usage/jobs")

    # ---------------- escritura (paso 6/7, con aprobación) ----------------
    def _guard(self, change_order: str | None):
        if not self.allow_write or not change_order:
            raise PermissionError("Escritura bloqueada: requiere allow_write=True y orden de cambio aprobada.")

    def hold(self, job_id: str, change_order: str):
        self._guard(change_order)
        return self.s.post(f"{self.base}/run/job/{job_id}/hold", timeout=self.timeout).json()

    def backup_folder(self, server: str, folder: str) -> dict:
        return {"json": self.job_definitions(ctm=server, folder=folder),
                "xml": self.s.get(f"{self.base}/deploy/jobs",
                                  params={"ctm": server, "folder": folder, "format": "xml"},
                                  timeout=self.timeout).text}

    def delete_folder(self, server: str, folder: str, change_order: str):
        self._guard(change_order)
        r = self.s.delete(f"{self.base}/deploy/folder/{server}/{folder}", timeout=self.timeout)
        r.raise_for_status()
        return r.json()

    def delete_job(self, job_path: str, server: str, change_order: str):
        self._guard(change_order)
        r = self.s.delete(f"{self.base}/deploy/job/{requests.utils.quote(job_path, safe='')}",
                          params={"server": server}, timeout=self.timeout)
        r.raise_for_status()
        return r.json()
```

> Alternativa oficial: **Control-M Python Client** (`ctm-python-client`, BMC, open source)
> — sirve para construir y desplegar definiciones como código Python. Para lectura/limpieza
> el adaptador REST directo es más simple y controlable.

### 8.2 Motor de reglas — `src/backend/limpieza_reglas.py` (propuesta)

Reglas como **datos** (YAML/JSON) + funciones puras. Así la misma regla corre sobre el
adaptador simulado y el real (ADR 002), y el RAG indexa el mismo archivo que ejecuta el motor.

```yaml
# data/reglas_limpieza/reglas.yaml
- id: R-DEF-01
  familia: definicion
  nombre: Job sin ejecución en N días
  peso: 25
  parametros: { dias_alerta: 90, dias_fuerte: 400 }
  fuentes: [definiciones, historia]
  falsos_positivos: [anual, contingencia, on_demand]
  accion_sugerida: cuarentena
- id: R-CND-02
  familia: condiciones
  nombre: In-condition que nadie produce
  peso: 35
  fuentes: [definiciones]
  accion_sugerida: reparar_o_eliminar
```

```python
from dataclasses import dataclass, field

@dataclass
class Finding:
    rule_id: str
    job: str
    folder: str
    weight: int
    source: str
    evidence: dict = field(default_factory=dict)

def rule_no_runs(job, last_run_days: int | None, params) -> Finding | None:
    if last_run_days is None or last_run_days > params["dias_alerta"]:
        w = 25 if (last_run_days or 10**6) > params["dias_fuerte"] else 15
        return Finding("R-DEF-01", job["name"], job["folder"], w, "historia",
                       {"dias_sin_ejecutar": last_run_days})

def rule_impossible_in_condition(job, produced_events: set[str]) -> Finding | None:
    missing = [e for e in job.get("wait_for_events", []) if e not in produced_events]
    if missing:
        return Finding("R-CND-02", job["name"], job["folder"], 35, "definiciones",
                       {"condiciones_sin_productor": missing})

def score(findings: list[Finding], history_months: float, protected: bool) -> int:
    if protected:
        return 0
    conf = 1.0 if history_months >= 13 else 0.6 if history_months >= 3 else 0.3
    distinct_sources = {f.source for f in findings}
    raw = sum(f.weight for f in findings)
    s = min(100, round(raw * conf))
    return s if len(distinct_sources) >= 2 else min(s, 59)   # regla de las dos fuentes
```

---

## 9. Diseño del RAG en ArgOS

### 9.1 Qué se indexa (corpus)
1. **Este documento**, partido por regla (`R-XXX-NN`) y por sección → cada chunk con metadatos
   `{rule_id, familia, fuente, peso, confianza: API|SQL|UTIL|CRITERIO}`.
2. **`reglas.yaml`** (8.2): la versión ejecutable de las reglas.
3. **Swagger oficial** (`docs/referencia_controlm_api/controlm_swagger_9.22.125.json`): un
   chunk por endpoint (`método + path + parámetros + descripción`) → el agente arma llamadas
   correctas sin inventar endpoints.
4. **Decisiones pasadas**: cada dictamen aprobado/rechazado por un dueño (con el motivo) →
   el RAG aprende las excepciones propias de BCI ("TDCOA01 es anual, no tocar").
5. **Nomenclatura TDC y catálogo de aplicaciones/dueños** de BCI.
6. **Manuales BMC** que BCI entregue (Physical Data Model, utilitarios de su versión).

### 9.2 Cómo se usa (patrón Planner/Coder/Validator/Documenter de la Capa 5)

| Agente | Hace | No hace |
|---|---|---|
| **Planner** | Decide qué extraer y qué reglas aplicar a una malla/folder | No toca Control-M |
| **Coder** | Genera las consultas (API/SQL) y los JSON de reprogramación/cuarentena | No despliega |
| **Validator** | Corre `ctm build` / `POST /build`, verifica regla de dos fuentes y lista de protección, compara el forecast antes/después | No aprueba |
| **Documenter** | Escribe el dictamen de negocio, el correo al dueño y la entrada de bitácora | No decide |

**Guardrails no negociables:**
- El puntaje lo calculan **reglas deterministas**; el LLM solo explica y prioriza.
- El LLM **no tiene credenciales de escritura**. Las acciones de escritura las ejecuta un
  humano de BCI, o ArgOS con `allow_write=True` + orden de cambio aprobada.
- Toda afirmación del dictamen debe **citar la evidencia** (regla + fuente + valor). Si no
  puede citarla, la omite.
- Prompt base del dictamen: *"Eres analista de Control Batch. Con los hallazgos adjuntos
  (JSON) y el contexto recuperado, redacta un dictamen de ≤ 120 palabras: qué es el job, por
  qué parece basura, qué se rompería si se elimina (vecinos en el grafo), nivel de riesgo y
  acción recomendada (reparar / cuarentena / eliminar / proteger). No inventes datos que no
  estén en la evidencia."*

### 9.3 Stack sugerido (simple, local, sin nube obligatoria)
- Embeddings + índice: **SQLite + sqlite-vec** o **Chroma** en disco (`data/<entorno>/rag/`).
- LLM: el `llm_client.py` que ya existe (Claude). Con datos de BCI, evaluar si el banco exige
  un modelo on-premise o una región específica: **preguntar antes de mandar definiciones
  reales a una API externa.**
- Sin framework pesado: el volumen (miles de chunks) no lo justifica (Algoritmo AIWIS:
  simplificar antes de automatizar).

---

## 10. Qué le falta a ArgOS para quedar 100 % real (hoja de ruta concreta)

> **Actualización 2026-09-24:** los puntos 1–10 y 12–13 ya están implementados en
> `src/backend/limpieza/` (ver `docs/decisiones/004-modulo-limpieza-real.md`). Lo que falta para
> producción es conectarse al Control-M de BCI (credenciales, Workbench, fechas de creación, JCLLIB, dueños reales).
> La tabla de abajo queda como registro del diagnóstico previo.

Estado actual verificado en el código (2026-09-24):
- `src/backend/db_motor.py` es **100 % simulado**: genera filas aleatorias
  (`generar_tabla`), y `limpiar()` solo marca `origen: "limpiada"` en un JSON.
- Filtro "no corren" = `estado_job ∈ {WAIT, HELD, NOTOK, ENDED_NOT_OK}` **o** `dias_sin_ejecutar > 3`.
  Es una sola señal y con umbral de 3 días: **generaría muchos falsos positivos en la realidad**
  (cualquier job mensual o semanal cae).
- `malla_catalog.ESTADOS_LIMPIEZA` ya tiene los estados correctos (activa → en_observacion →
  candidata_limpieza → en_limpieza → limpiada / suspendida) con historial. **Esto se mantiene.**
- `solicitudes_malla.buscar_en_control_m()` busca en el catálogo local, no en Control-M.
- No existe RAG ni embeddings en el backend. `requirements.txt` no tiene `requests`/`httpx`.
- `config.py` ya tiene el switch `es_real()` — es el punto de enganche.

### 10.1 Mejoras priorizadas

| # | Mejora | Archivos | Por qué |
|---|---|---|---|
| 1 | **Definir la interfaz `ControlMSource`** (`job_definitions`, `active_jobs`, `run_history`, `forecast`, `agents`, `calendars`, `resources`, `backup_folder`, `delete_*`) y hacer que `db_motor.py` la implemente como adaptador simulado | `db_motor.py`, nuevo `controlm_source.py` | ADR 002: dos adaptadores detrás de una interfaz |
| 2 | **Adaptador real** (sección 8.1) seleccionado por `config.es_real()` | nuevo `controlm_real.py`, `requirements.txt` (+`requests`) | Sin esto nada es real |
| 3 | **Enriquecer el modelo simulado** con los campos que el real sí tiene: `host`, `run_as`, `mem_name/mem_lib`, `wait_for_events`, `add_events`, `calendario`, `creado_en/por`, `historia_corridas[]`, `forecast_proximos_24m` | `db_motor.py` | Las reglas se prueban en simulado con la misma forma de datos que tendrá el real |
| 4 | **Motor de reglas** con `reglas.yaml` + scoring con las dos fuentes, confianza de 13 meses y lista de protección | nuevo `limpieza_reglas.py`, `data/reglas_limpieza/` | Reemplaza el filtro de una sola señal |
| 5 | **Flujo de consenso** conectado a los estados existentes + aprobación del dueño (reusar el patrón de `solicitudes_malla`: historial + borrador de correo) | `malla_catalog.py`, nuevo `limpieza_flujo.py` + API | Pasos 5–8 de la sección 3 |
| 6 | **Respaldo automático antes de cualquier baja** en `data/<entorno>/respaldos_limpieza/` + commit | adaptador + flujo | Rollback garantizado |
| 7 | **Escritura bloqueada por defecto** (`allow_write` + número de orden de cambio + usuario aprobador en bitácora) | adaptador real | Un banco no aceptará otra cosa |
| 8 | **Probar contra un Control-M de verdad sin esperar a BCI: Control-M Workbench** (imagen Docker gratuita de BMC con Automation API) | `scripts/`, `tests/` | Permite pasar de "simulado" a "real" en tu máquina **ya**, con endpoints reales |
| 9 | **Tests de contrato**: la misma suite corre contra el simulado y contra Workbench | `tests/test_controlm_source.py` | Garantiza que los dos adaptadores no se desalineen |
| 10 | **RAG** (sección 9): indexar este doc + `reglas.yaml` + Swagger + decisiones | nuevo `rag/` | Dictámenes explicados y citables |
| 11 | **Extractor z/OS** (Capa 3, py3270) para R-ZOS-*: listar miembros JCL y cruzarlos con `MEMNAME` | `src/extraction/` | Ventaja diferencial vs. BMC |
| 12 | **Métrica de valor**: `GET /usage/jobs` antes/después + horas-hombre ahorradas → alimenta el informe comercial (5.000 UF piloto) | reportes | Justifica el precio |
| 13 | **Cambiar el umbral de 3 días** en `db_motor.consultar/estadisticas` por el scoring real (o como mínimo por frecuencia: diario > 3 días, mensual > 35, anual > 400) | `db_motor.py` | Evita que la demo muestre números que la realidad desmiente |

### 10.2 Preguntas para BCI (para validar el proceso manual actual)
1. ¿Qué versión exacta de Control-M (EM/Server) y de Control-M for z/OS usan? ¿Automation API habilitada? ¿Workload Archiving licenciado?
2. ¿Pueden darnos un **usuario de API de solo lectura** (o API key) en certificación primero?
3. ¿Cuántos meses de historia guardan (Archive, `CMR_RUNINF`, IOA Log)? → define el `factor_confianza`.
4. ¿Cómo marcan hoy los jobs de contingencia, regulatorios y on-demand? (lista de protección)
5. ¿Qué comandos/reportes usa hoy el equipo cuando "limpia a mano"? (CTMRPLN, reportes del EM, SQL…) → se replican 1:1 en ArgOS.
6. ¿Quién aprueba una baja? ¿Qué sistema de órdenes de cambio usan (Helix)? ¿Cuánto dura una cuarentena?
7. ¿Se puede mandar metadata de definiciones (no datos de clientes) a un LLM externo, o debe ser on-premise?
8. ¿Cobran por job (licencia por tarea)? → la reducción de jobs se traduce directo en ahorro.

---

## 11. Checklist operativo (una página)

- [ ] Usuario/API key de **solo lectura** configurado en `.env` (`CTM_ENDPOINT`, `CTM_API_KEY`).
- [ ] Extracción completa: definiciones (JSON+XML), folders, calendarios, recursos, agentes, AJF, Archive ≥ 13 meses, forecast 24 meses, miembros JCL z/OS.
- [ ] Reglas R-* corridas; hallazgos con fuente y evidencia.
- [ ] Puntaje con factor de confianza, lista de protección y regla de dos fuentes.
- [ ] Dictamen IA con citas; contradicciones → revisión humana.
- [ ] Correo/ticket al dueño; respuesta registrada.
- [ ] Cuarentena aplicada (Schedule Never / folder ZZ / hold) y fecha de fin.
- [ ] Respaldo JSON+XML versionado en git.
- [ ] Eliminación con orden de cambio aprobada (API `DELETE` o `emdef`).
- [ ] Verificación post New Day (sin Wait Condition nuevos) + métrica `/usage/jobs`.
- [ ] Bitácora cerrada (quién, cuándo, orden de cambio, respaldo, resultado).

---

## 12. Fuentes

- **Swagger oficial Control-M Automation API v9.22.125** — <http://aapi-swagger-doc.s3-website-us-west-2.amazonaws.com/> (copia local: `docs/referencia_controlm_api/controlm_swagger_9.22.125.json`)
- Control-M Automation API — portal: <https://controlm.github.io/>
- Ejemplos oficiales: <https://github.com/controlm/automation-api-quickstart>
- Control-M Python Client: <https://controlm.github.io/ctm-python-client>
- Documentación Automation API (servicios deploy/run): <https://docs.bmc.com/docs/automation-api/monthly/services-1116950323.html>, <https://docs.bmc.com/docs/automation-api/920/deploy-service-887941140.html>
- Tablas DEF_JOB / DEF_LNKI_P / DEF_LNKO_P (condiciones en definiciones): <https://scheduler-usage.com/forum/viewtopic.php?t=1224>, <http://www.ctmguru.com/2012/02/prerequisite-conditions.html>
- CMR_RUNINF (últimas ~20 corridas exitosas por job): <https://scheduler-usage.com/forum/viewtopic.php?t=2084>, <https://community.bmc.com/s/question/0D53n00007aDgvJCAS/info-required-from-table-cmrruninf>
- Queries sobre CMS_JOBDEF / CMR_AJF / CMR_IOALOG: <http://ithowtoit.blogspot.com/2012/10/useful-sql-request-for-control-m.html>
- Esquema de base Control-M/Server 9: <https://www.scribd.com/document/839651161/471972-BMC-Control-M-Server-9>
- Utilitario `em database` / estadísticas: <https://controlm.uconn.edu/help/CTMHelp/en-US/Documentation/Utilities/em_database_menu.htm>, <https://controlm.uconn.edu/help/CTMHelp/en-US/Documentation/Utilities/Statistics_and_reporting.htm>
