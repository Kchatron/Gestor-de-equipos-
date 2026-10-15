# Fase 0 — Fundamentos de Python y diseño del esquema de datos

> Proyecto: Sistema de Diagnóstico de Equipos | Fecha de apertura: 2026-10-15 | Estado: **EN CURSO, NO VALIDADA**
>
> Audiencia: autor del proyecto, revisores técnicos, Codex y otros asistentes IA. Documento de contexto y evidencia de trabajo; no equivale a una fase ejecutada.

## 0. Instrucciones para una IA que reciba este documento

Actúa como **revisor crítico, mentor y colaborador**, no como autor autónomo de las decisiones de ingeniería. Antes de recomendar cambios, distingue: (a) decisiones aprobadas, (b) hipótesis pendientes, (c) implementación verificada, (d) propuestas tuyas. No presentes tareas pendientes como ejecutadas. No agregues tablas o tecnologías porque sí. Examina riesgos, consistencia, integridad referencial, extensibilidad estrictamente necesaria y valor para portafolio Data Engineer. Propón alternativas con costos/beneficios, sin modificar archivos o ejecutar comandos destructivos sin autorización. Si faltan evidencias de pruebas, indícalo.

## 1. Propósito del sistema y orientación profesional

Construir un MVP local y auditable para diagnosticar el estado de equipos Linux con **datos reales**, historial de métricas y reglas explicables. Público objetivo: técnicos y usuarios con manejo de CLI. No sustituye diagnóstico físico ni juicio de un profesional. Prioridad profesional: demostrar diseño relacional, calidad de datos, adquisición, normalización, trazabilidad, ETL e idempotencia (estas últimas en fase 2), junto con capacidad de defender decisiones técnicas en entrevistas para **Data Engineer**.

**Alcance de v1 según documentación existente:** una máquina EndeavourOS/Arch, aunque con esquema preparado para varias máquinas; señales de CPU, RAM, temperatura, procesos y almacenamiento; SQL en PostgreSQL, CLI y diagnóstico basado en reglas y baseline estadístico. No construir todavía Kafka, ML predictivo, sistema multi-OS, multi-tenant ni dashboard complejo. La narración con IA corresponde a fase 4, no a esta fase.

## 2. Fuentes y precedencia

1. `roadmap-diagnostico-equipos.md` (agosto 2026): fases 0-5, división por tipo de trabajo y checkpoints.
2. `documentacion-proyecto-gestiondeequipos.md`: alcance, reglas SMART, límites, pruebas reales y decisiones anteriores.
3. Conversación de diseño del 2026-10-15: propuesta de `STORAGE_DEVICE` y claves foráneas opcionales para métricas y diagnósticos, evidencia en JSONB, vocabulario `NORMAL/SOSPECHA/PROBLEMA`.
4. Preferencias explícitas actuales: comenzar Fase 0; sin repositorio, PostgreSQL ni SQL creados; Python principiante; usar Codex intensivamente pero revisando cada cambio; documentación exhaustiva; cierre estricto con excepciones justificadas.

**Conflictos/versionado a vigilar, sin ocultarlos:**
- El roadmap inicial enumera 4 tablas (`machine`, `snapshot`, `metric_reading`, `diagnosis`), pero el diseño conceptual posterior propone una quinta (`storage_device`) para separar múltiples discos: **adoptar propuesta revisada y comprobarla durante Fase 0**.
- El roadmap dice `machine_id` en todo; la propuesta relacional discutida prioriza relaciones transitivas vía snapshot y dispositivo. **Definir explícitamente si se materializa machine_id adicional en cada tabla, justificando redundancia y reglas de consistencia**, antes del DDL.
- Documento anterior utiliza `UNKNOWN` para autotest nunca ejecutado; conversación reciente propone `NOT_TESTED` para separar ausencia real de prueba frente a error de lectura. **Pendiente de actualizar contrato de normalización**, no bloquea SQL si ese estado no queda persistido como enumeración todavía.
- Documentación vigente delimita normalizador MVP SATA; NVMe fue caso de contraste, no implementación comprometida para v1.
- El proyecto preveía Claude Code + Antigravity; a partir de ahora **Codex** es el asistente operativo preferente. No hay necesidad de coordinar varios agentes por defecto.

## 3. Objetivo y criterio de finalización de Fase 0

**Objetivo original:** reforzar `subprocess`, `dataclasses`/JSON y POO básica; diseñar y escribir personalmente las sentencias PostgreSQL `CREATE TABLE`, y explicar sus claves, relaciones, tipos y restricciones. **Objetivo operacional añadido:** disponer de un repositorio Git y entorno PostgreSQL para comprobar el esquema.

**No se cierra** solo por contar con archivos entregados por Codex. Debe existir evidencia ejecutable y explicación propia.

### Entregables

- [ ] Repositorio Git inicializado y primer commit propio; remoto GitHub opcional pero recomendado.
- [ ] Estructura reproducible, `.gitignore`, `README.md`, `AGENTS.md`, `compose.yaml` o instalación local equivalente para PostgreSQL; herramientas revisadas antes de ejecutarlas.
- [ ] Ejercicios cortos del autor en Python: un `subprocess.run` seguro, conversión `dataclass` a JSON y una clase sencilla; cada ejercicio explica qué hace y cómo fallaría.
- [ ] Diagrama actualizado de cinco entidades, cardinalidades, claves y nulabilidad.
- [ ] DDL `sql/001_initial_schema.sql` **escrito inicialmente por el autor**; Codex puede hacer comentarios, no sustituir sin aprobación.
- [ ] Arranque reproducible de PostgreSQL y aplicación del DDL sin errores.
- [ ] Prueba de integridad: una máquina con dos discos, mismo snapshot, dos lecturas de igual nombre diferenciadas por disco y dos diagnósticos `STORAGE` atribuibles a discos distintos.
- [ ] Prueba negativa de integridad referencial, clave ajena inválida y validación de consistencia equipo/snapshot/dispositivo.
- [ ] Registro de decisiones (ADR o sección de decisiones), incidencias, comandos usados, evidencias verificables y reflexión de aprendizaje.
- [ ] Revisión externa vía Codex + aceptación/rechazo razonado de sus recomendaciones.

## 4. Modelo conceptual de partida (propuesta por validar en SQL)

```text
MACHINE (1) ──── (N) SNAPSHOT
  │                   ├──── (N) METRIC_READING
  │                   └──── (N) DIAGNOSIS
  └──── (N) STORAGE_DEVICE
                    ├──── (N) METRIC_READING [FK opcional desde la lectura]
                    └──── (N) DIAGNOSIS      [FK opcional desde el diagnóstico]
```

- `MACHINE`: identidad de una computadora, distinta de sus capturas.
- `SNAPSHOT`: ejecución/captura puntual del colector, fecha/hora asociada a la máquina.
- `STORAGE_DEVICE`: identidad de un disco físico ligado a la máquina. No crear una entidad genérica `DEVICE` hasta tener un requerimiento real.
- `METRIC_READING`: lectura histórica de una métrica en un snapshot; si es específica de disco, FK nullable a `STORAGE_DEVICE`; si es global (CPU/RAM), FK NULL.
- `DIAGNOSIS`: hallazgo/conclusión asociado a snapshot y componente (`STORAGE`, `CPU`, `RAM`, etc.); si corresponde a disco físico, FK nullable a `STORAGE_DEVICE`. Evidencia justificativa copiada en JSONB: se distingue de la serie de medidas históricas en `METRIC_READING`. No crear de entrada una tabla de unión diagnóstico-lecturas.
- Categorías preliminares de severidad: `NORMAL`, `SOSPECHA`, `PROBLEMA`; independientes de confianza y triage. Revisar vocabulario antes de DDL si se impone `CHECK` o tipo ENUM.

**Invariantes a diseñar, no asumir implementadas:**
1. Ninguna lectura/diagnóstico debe asociarse a un disco de otra máquina diferente a la de su snapshot.
2. Un diagnóstico de disco individual debe identificar disco y componente STORAGE; CPU/RAM global no debe aparentar ser de un disco.
3. No confundir identidad estable de equipo/disco con snapshots temporales; evitar usar rutas Linux inestables (`/dev/sda`, etc.) como único identificador persistente.
4. Valorar cuál es la clave de idempotencia/captura, sin anticipar un pipeline completo de fase 2.
5. Determinar si una lectura global carece de `storage_device_id` y cómo se representa un dato no disponible (no como cero real).

**Preguntas concretas para el autor antes de escribir SQL:**
- ¿Cuáles son PK, FK, tipos y reglas de nulabilidad de cada entidad?
- ¿Cómo identificaremos de manera estable un disco aun si cambia la ruta `/dev/*`? ¿Qué hacemos si no trae serial?
- ¿En qué tablas es estrictamente necesario materializar `machine_id`? Si se duplica, ¿cómo impides inconsistencias?
- ¿Quieres una sola fila por hallazgo en `DIAGNOSIS` o un resumen por componente? ¿Qué cardinalidad permite múltiples hallazgos?
- ¿Cómo impondrías la condición de mismo equipo usando claves compuestas o una validación apropiada?
- ¿Qué índices justificas por consultas reales: histórico por máquina, métricas por disco, diagnósticos por componente?

## 5. División de responsabilidades

| Actividad | Responsable | Justificación |
|---|---|---|
| Explicar entidades y relaciones | Autor | Capacidad de diseño de datos para entrevista |
| Primer DDL, PK/FK, índices y restricciones | Autor | Competencia central Data Engineer |
| Ejercicios de Python y primera versión del colector (fase 1) | Autor | Entender código y operaciones reales |
| Configuración de repositorio y herramientas no críticas | Codex, con revisión | Delegar trabajo repetitivo |
| `.gitignore`, entorno virtual, `compose.yaml`, README inicial, plantilla AGENTS | Codex, con revisión | Automatización de scaffolding |
| Análisis de casos borde del DDL | Codex como revisor | El autor acepta o rechaza cambios con razones |
| Tests SQL basados en escenarios fijados por el autor | Codex puede implementarlos | El autor define resultado esperado y ejecuta pruebas |
| Documentación final de fase | Autor + IA asistente | Registrar hechos comprobados; no inventar evidencia |

### Política de Codex

- Leer contexto del proyecto y `AGENTS.md` antes de modificar archivos.
- Trabajar por una tarea acotada. Primero plan breve, luego modificaciones; confirmar para acciones potencialmente destructivas.
- No inventar métricas, diagnósticos, salidas de comandos, capturas, commits, resultados de tests ni versiones.
- No escribir/reemplazar el esquema final por iniciativa propia.
- Mostrar archivos cambiados, comandos usados, pruebas y riesgos.
- Revisar siempre `git diff` y ejecutar pruebas pertinentes antes del commit.
- No subir seriales reales, hostname, claves, credenciales, reportes SMART sin anonimizar ni `.env`.

### Primer encargo recomendado para Codex (copiar y pegar)

```text
Contexto: proyecto académico/portafolio Data Engineer de diagnóstico de equipos Linux. Estamos en la FASE 0. Soy autor principiante en Python, pero las decisiones de modelo y DDL deben ser mías. Lee FASE-00-fundamentos-y-schema.md y el roadmap antes de actuar.
Tarea: crea SOLAMENTE scaffolding: estructura mínima de carpetas, .gitignore para Python/PostgreSQL, README inicial con comandos por completar, AGENTS.md con normas de contribución y un compose.yaml básico para PostgreSQL local usando variables de entorno sin credenciales reales. No crees tablas ni SQL, ni colector, ni motor de reglas.
Antes de escribir: enseña el árbol propuesto y explica cada archivo. Después: informa cada cambio, cómo arrancar y detener el servicio, cómo verificar que está saludable, y riesgos o dependencias. No ejecutes comandos destructivos; no hagas commit automáticamente. Mantén cambios pequeños y revisables.
```

### Comprobaciones manuales del autor (ejemplos, adaptar tras confirmar herramientas)

```bash
python --version
git --version
codex --version
docker --version
docker compose version
```

No es obligatorio Docker si PostgreSQL se configura localmente; decidir a partir del sistema real y herramientas disponibles. Para cambios del asistente:

```bash
git status
git diff --check
git diff
```

Para comprobar JSON/Python y conexión SQL se escribirán los comandos **reales usados** cuando exista entorno, no antes.

## 6. Microplan de aprendizaje

1. **Python A — `subprocess`:** ejecutar un comando de solo lectura con lista de argumentos, capturar stdout/stderr, identificar `returncode`, gestionar comandos ausentes; explicar por qué no usar `shell=True` sin motivo.
2. **Python B — `dataclasses` + JSON:** representar una lectura con nombre, valor, unidad y timestamp; serializar a JSON; distinguir número, texto y `null`.
3. **Python C — POO básica:** clase `Collector` mínima con método `collect()` y salida dict, sin abstracciones prematuras.
4. **PostgreSQL A — diseño:** entidades, PK, FK, `NULL`, `CHECK`, `UNIQUE`, tipos y cardinalidad.
5. **PostgreSQL B — DDL:** construir las cinco tablas en orden de dependencias y explicar índices/restricciones.
6. **PostgreSQL C — prueba:** insertar máquina, dos discos, snapshot, lecturas y diagnósticos; consultas SELECT con JOIN y pruebas negativas.

**Regla de aprendizaje:** para cada ejercicio, explicar qué dato entra, qué dato sale, cómo detectar un fallo y por qué se eligió el diseño.

## 7. Criterios de aceptación / checkpoint estricto

- [ ] `CREATE TABLE` redactados inicialmente por el autor, ejecutados correctamente y versionados en Git.
- [ ] El autor puede justificar cada entidad, PK/FK, `NULL`, tipos y cardinalidad sin leer una respuesta de IA.
- [ ] El caso de **dos discos** funciona y permite filtrar diagnósticos de un disco sin mezclar lecturas del otro.
- [ ] No se permite asociación cruzada entre máquina del snapshot y máquina del almacenamiento.
- [ ] El autor realizó ejercicios de Python y puede interpretarlos.
- [ ] Cambios de Codex revisados y documentados; no hay datos sensibles en el repo.
- [ ] Evidencias (comandos/salidas pertinentes, commit, pruebas) agregadas abajo.

**Regla de excepción:** cada punto no logrado requiere `ID`, motivo, riesgo, mitigación, responsable y fecha de revisión. No marcar fase como completada sin expresar las excepciones.

## 8. Bitácora y evidencias (por completar durante la fase)

| Fecha | Actividad | Autor/Codex | Evidencia (archivo/commit/comando) | Resultado | Pendiente |
|---|---|---|---|---|---|
| 2026-10-15 | Apertura documental Fase 0 y reorganización de responsabilidades | Asistencia IA, pendiente revisión del autor | Este Markdown | Plan creado; NO hay repositorio ni SQL ejecutado | Empezar ejercicios y scaffolding |

### Pruebas ejecutadas

**NINGUNA al abrir la fase.** No existen todavía resultados verificables de PostgreSQL ni commits del proyecto.

### Registro de decisiones

| ID | Estado | Decisión | Motivo | Consecuencia |
|---|---|---|---|---|
| D-001 | Propuesta a validar | Quinta entidad `STORAGE_DEVICE` | Soportar varios discos en un snapshot | FK opcional desde lecturas y diagnósticos |
| D-002 | Propuesta a validar | Evidencia de `DIAGNOSIS` en JSONB | Auditoría de justificación sin tabla puente | Necesita contrato de evidencia |
| D-003 | Propuesta a validar | Severidad NORMAL/SOSPECHA/PROBLEMA | Consistencia con lenguaje del MVP | Fijar semántica antes de CHECK/ENUM |
| D-004 | Acordada | Codex sustituye a Claude Code/Antigravity como herramienta principal | Preferencia expresa del autor | Cambia asignación operativa, no dueño de decisiones |
| D-005 | Pendiente | `machine_id` materializado en todas las tablas vs transitivo | Roadmap original y modelo reciente difieren | Cerrar antes del DDL |

### Lecciones aprendidas (por completar)

- ¿Qué error de diseño descubrí y cómo lo corregí?
- ¿Qué hice manualmente, qué delegué y cómo verifiqué el trabajo de Codex?
- ¿Cómo explicaría el modelo en 90 segundos a un entrevistador?
- ¿Qué decisión sería diferente si hubiera 100 máquinas? ¿Por qué no implementarla ya?

## 9. Cierre formal de la fase (no rellenar anticipadamente)

- Estado final: **PENDIENTE**
- Fecha de cierre: **PENDIENTE**
- Commit/tag de cierre: **PENDIENTE**
- Pruebas verificadas: **PENDIENTE**
- Excepciones justificadas: **PENDIENTE**
- Lecciones del autor: **PENDIENTE**
- Revisión crítica de otra IA: **PENDIENTE**
- Siguiente fase autorizada: **NO; Fase 1 depende del cierre verificable de Fase 0**

## 10. Handoff hacia Fase 1 (solo cuando Fase 0 esté cerrada)

Entregar a quien implemente: diagrama/DDL validados, criterios de identificación de equipos y discos, esquema JSON preliminar, convenciones de timestamp/unidades/null, lista de sensores disponibles y reglas de acceso, configuración reproducible, pruebas y decisiones pendientes. La Fase 1 produce un colector CLI real de EndeavourOS con `psutil`, `lm-sensors`, `smartctl` y `subprocess`, capaz de emitir JSON válido y consistente. **No anticipar implementación ni afirmar completitud.**
