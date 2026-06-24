# linear-resumen

Agente de **solo lectura** que, bajo demanda, te dice en qué issues estás metido en Linear y cómo vas. Cruza tus tareas asignadas del equipo **Back-end** con el ciclo actual y el calendario laboral, y produce un resumen a tres niveles —**hoy**, **ciclo** y **backlog**— ordenado por lo que tienes en marcha, señalando qué deberías haber empezado ya, qué está en riesgo, qué es de soporte y si vas bien o mal.

No modifica nada en Linear. Solo lee, calcula y te lo cuenta.

Se lanza bajo demanda con cualquiera de:

```bash
claude "cómo voy"
claude "qué tengo que hacer hoy"
claude "resúmeme mis issues"
claude "¿tengo menciones pendientes?"   # foco solo en menciones; admite "barrido amplio" para el alcance B
```

## Skills requeridas

- `linear-rules-backend` — convenciones del equipo Back-end: estados del workflow, estimaciones válidas (`0,1,2,4,8,16`), prioridades y su significado temporal, proyectos (`Platform Engineering`, `Engineering Debt`, `Support issues`), flujo de soporte y labels temáticos. Es lo que da sentido a "deberías haberla empezado" y a "esto es soporte".

## Herramientas (MCPs)

| MCP | URL / origen | Uso |
|---|---|---|
| **Linear** | `https://mcp.linear.app/mcp` | Listar mis issues, leer detalle y relaciones, consultar el ciclo actual. **Solo lectura.** |

Tools que usa (todas de lectura):

| Tool | Para qué |
|---|---|
| `list_issues` | Traer mis issues con `assignee="me"`, `team` Back-end. Devuelve estado, prioridad, estimación, ciclo, proyecto, labels, due date. |
| `get_issue` (`includeRelations=true`) | Detalle de una issue concreta: relaciones `Relates to`/`Blocks` (detección de soporte y bloqueos) y campos que no vengan en el listado. |
| `list_cycles` (`type="current"`) | Fechas y número del ciclo en curso para calcular el % transcurrido y la proyección. |
| `list_comments` | Comentarios de un issue concreto, para el barrido de @-menciones (ver "Menciones"). Es **por issue**, no global. |

> **Identidad — más simple que en Productive.** Aquí no hay que resolver la persona por email (ni hay colisiones de nombre): `list_issues` acepta `assignee="me"` y resuelve automáticamente al usuario autenticado por el MCP. Tu `userId` aparece además como `assigneeId` en cualquier resultado (sirve para autoría de comentarios), y tu handle `@jaime.hernandez` se resuelve con `get_user` para detectar @-menciones en comentarios (ver "Menciones").

## Qué significa "participo" (alcance)

**`participo` = issues asignadas a mí** (`assignee="me"`) en el equipo Back-end.

> ⚠️ **Limitación real del MCP.** `list_issues` **no** permite filtrar por *subscriptor*, *comentarista* ni *creador*. Por eso "participo" se limita a **asignadas**. Issues donde solo estás suscrito o has comentado no son recuperables de forma eficiente y quedan **fuera de alcance** (ampliable a futuro si el MCP añade ese filtro, o escaneando issue a issue —caro, no recomendado por defecto—).

> 💡 **Aprendizaje del uso real.** En la práctica, casi todo el trabajo del ciclo está **asignado** al usuario (en el run de prueba, 58 de 60 puntos del ciclo eran suyos). Por eso la limitación de no ver suscripciones apenas afecta: con `assignee="me"` se cubre prácticamente todo el trabajo propio.

## Conceptos clave

### Conversión puntos → tiempo

El equipo estima en `0,1,2,4,8,16` puntos y ninguna tarea supera **2 días**. Se asume **1 punto ≈ 1 hora** (misma convención que el agente de horas), con jornada de 8 h:

| Puntos | Tiempo | |
|---|---|---|
| 16 | 2 días | máximo permitido |
| 8 | 1 día | |
| 4 | ½ día | |
| 2 | ¼ día | |
| 1 | ~1 h | |
| 0 | trivial | |

Para "¿llego?" se cuentan **días hábiles** (lun–vie menos festivos del calendario de abajo), no días naturales.

### Orden de visualización (por estado)

Las issues se ordenan poniendo primero lo que tienes entre manos:

1. **In progress** — lo que estás haciendo ahora
2. **Blocked** — en marcha pero parado
3. **To do** — comprometido al ciclo, sin empezar
4. **Backlog** — no comprometido todavía
5. Estados de soporte (`Triage`, `Analyzing`, `Hotfix`, `Next release`) se intercalan según equivalgan a "en marcha" o "pendiente"

`Done` / `Canceled` / `Duplicate` no se listan, salvo las **cerradas hoy / en el ciclo** que se usan como contexto para el veredicto ("vas bien").

Dentro de cada estado, ordenar por **prioridad** (Urgent → Low) y luego por **due date** más cercano.

> **Prioridad en Linear es numérica e invertida**: `1=Urgent, 2=High, 3=Medium, 4=Low, 0=None`. Menor número = más urgente. Significado temporal (de `linear-rules-backend`): Urgent = hoy · High = esta semana · Medium = ciclo normal · Low = cuando haya hueco.

### Semáforo "¿debería haberla empezado?" (issues SIN empezar: To do / Backlog)

- 🔴 **Deberías haberla empezado ya** si se cumple cualquiera:
  - `priority = Urgent` y no está In progress (Urgent = hoy)
  - `dueDate < hoy` (vencida) y no está cerrada
  - **no llegas**: días hábiles hasta el `dueDate` (o el fin de ciclo si no hay due) **<** estimación en días
- 🟡 **Empiézala pronto** si:
  - `priority = High` (esta semana) y due/ciclo aprietan
  - vas **pasado de la mitad del ciclo** y la tarea sigue sin empezar
  - `dueDate` cae dentro de 1–2 días hábiles
- 🟢 **Va con margen** si hay holgura (días hábiles hasta due ≥ estimación) y prioridad Medium/Low

> 💡 **Aprendizaje del uso real.** En la práctica las tareas del ciclo **casi nunca llevan due date** (en el run de prueba, ninguna lo tenía). Por eso el semáforo se apoya sobre todo en **prioridad + posición en el ciclo + fin de ciclo como fecha límite implícita**, no en el `dueDate`. El due, cuando existe, refuerza el cálculo; cuando falta, el fin de ciclo es la referencia (no tratar la ausencia de due como "sin urgencia").

### Semáforo "¿vas bien o mal?" (issues IN progress)

- 🔴 **Mal**: `dueDate` vencida, **o** lleva en progreso bastante más que su estimación (p. ej. una de 8 pt / 1 día abierta 3+ días), **o** está `Blocked`
- 🟡 **Justo**: `dueDate` hoy/mañana y sin cerrar, o el tiempo en progreso ya iguala la estimación
- 🟢 **Bien**: dentro de estimación y con margen hasta el due

> Si el listado no trae `startedAt`, aproximar la antigüedad en el estado por `updatedAt` o leer la issue con `get_issue`. Indicar cuando el dato sea aproximado.

### Detección de soporte ("¿tiene soporte?")

Se marca **Soporte: sí** si se cumple cualquiera (de más fuerte a más débil):

1. El **nombre del proyecto** empieza por **`[SOPORTE]`** (o es `Support issues` / `Customer Support`). **Señal principal y más fiable en este equipo.**
2. La issue tiene una relación **`Relates to`** con un issue del proyecto/equipo **Support** (requiere `get_issue includeRelations=true`) — señal canónica del flujo de soporte, pero **en la práctica muchas tareas de soporte no la llevan**, así que no basta con confiar en ella.
3. Está en un **estado del workflow de soporte**: `Triage`, `Analyzing`, `Hotfix`, `Next release`.
4. El **título** empieza por `Soporte` / `[Soporte]` (señal débil de respaldo).

Si no se cumple ninguna → **Soporte: no**. Las de soporte se resaltan porque suelen llevar SLA y prioridad implícita.

> 💡 **Aprendizaje del uso real.** En el run de prueba, BACK-141 era soporte y **no** tenía relación `Relates to`; lo que la delataba era el proyecto `[SOPORTE] Error en la carga de información estructural vía SFTP`. Por eso el prefijo `[SOPORTE]` en el nombre del proyecto pasa a ser la regla nº1, por delante de la relación.

### Los niveles del resumen

- **HOY (diario)** — lo accionable de la jornada, **con detalle por tarea**: In progress (con antigüedad vs estimación y qué hacer), Urgent/vence hoy (con el porqué), y Blocked (con a quién escalar). Cierra con un resumen de "las N cosas de hoy".
- **CICLO (quincena = `cycle` de Linear)** — cómo va el ciclo en curso, **listando todas tus tareas**: puntos (hechos/en curso/pendientes), días hábiles y proyección, ritmo (hecho vs tiempo consumido), las pendientes en orden de ataque, en curso, las que deberías haber empezado y las ya cerradas.
- **BACKLOG** — lo asignado fuera del ciclo actual, priorizado (compacto), más avisos de higiene (sin estimación, sin due, sin label) que incumplen las reglas de Back-end.
- **MENCIONES** — @-menciones en comentarios que esperan respuesta tuya, para que no se pierdan (ver sección "Menciones").

Cada nivel lleva su **veredicto** (🟢/🟡/🔴) y al final hay un **veredicto global**.

### Menciones (seguimiento de @-menciones en comentarios)

Objetivo: que ninguna @-mención en un comentario se quede sin respuesta y puedas darle seguimiento.

> ⚠️ **Limitación de partida (honesta).** El MCP de Linear **no** expone la bandeja de entrada (Inbox) ni una búsqueda global de comentarios. `list_comments` funciona **por issue**: hay que saber dónde mirar. No existe "dame todos los comentarios que me mencionan". Por eso el agente hace un **barrido best-effort** sobre un conjunto acotado de issues y **declara siempre su alcance** (qué miró y qué no).

**Cómo detecta una mención tuya.** En el cuerpo de cada comentario las menciones aparecen como **texto plano `@handle`** (p. ej. `@jaime.hernandez`), **no** como markup `<user id="…">`. Por eso:

- Para saber si **te mencionan**, buscar tu **handle** (`mention_handle`, p. ej. `@jaime.hernandez`) en el `body` del comentario. Resolverlo una vez con `get_user` (o `list_users`) y cachearlo; no fiarse del nombre suelto.
- Para saber **quién escribió** cada comentario y si **ya respondiste**, usar `author.id` (tu `userId` es `d62e52b3-7d35-4673-9168-91499345cdb6`), que `list_comments` sí devuelve de forma fiable junto a `parentId` y `createdAt`.

> 💡 **Aprendizaje del uso real.** Los dos formatos conviven: las **descripciones** de issue traen las menciones como markup `<user id="…">email</user>`, pero los **comentarios** (vía `list_comments`) las traen como `@handle` en texto plano. La detección de menciones en comentarios va por handle; la de autoría/respuesta, por `author.id`.

**Qué cuenta como "pendiente de responder".** Un comentario que (a) contiene tu `mention_handle`, (b) lo escribió otra persona (`author.id` ≠ el tuyo), y (c) **no** tiene un comentario posterior tuyo en el mismo hilo (mismo `parentId` raíz, `createdAt` mayor y `author.id` = el tuyo). Orden: lo **más antiguo sin responder primero** (es lo que más riesgo tiene de perderse).

**Alcance del barrido — elige el compromiso (coste vs cobertura):**

- **A · Solo mis issues (por defecto, barato).** `list_comments` sobre mis issues activas (In progress / Blocked / To do + las actualizadas en los últimos `mention_lookback_days`). Pocas llamadas. **Punto ciego:** menciones en issues donde **no** soy asignado (un `FUNC-…` de otro equipo que me arrastra), que son las más fáciles de perder.
- **B · Issues recientes del equipo (cobertura media).** Además, `list_issues(team="Back-end", orderBy="updatedAt", updatedAt="-P{N}D")` y barrer sus comentarios. Cubre menciones cross-issue dentro de Back-end, a cambio de una llamada `list_comments` por issue reciente.
- **C · Bandeja nativa de Linear (cobertura total, fuera del MCP).** La fuente autoritativa de menciones es el **Inbox de Linear** (y sus avisos por email/Slack), que captura menciones en cualquier issue de cualquier equipo. El MCP no lo lee, así que aquí el agente **no sustituye** al Inbox: lo recomienda como hábito y usa A/B como red de seguridad/resumen. (Si algún día el MCP expone *notifications*, esta pasa a ser la vía principal y A/B quedan de respaldo.)

**Recomendación:** **A** por defecto; ofrecer "barrido amplio" (**B**) cuando el usuario pida una pasada a fondo; recordar **C** como hábito complementario. El alcance usado se indica en la salida.

**Semáforo de menciones:** 🔴 si hay alguna mención sin responder de **> 3 días hábiles**, o en issue **Urgent/soporte**; 🟡 si hay alguna de 1–3 días; 🟢 si ninguna pendiente.

**Seguimiento (siendo solo lectura).** El agente **no responde por ti**: lista cada mención pendiente con issue, quién te menciona, antigüedad, un fragmento corto y el **enlace directo al comentario**, en orden de ataque. Si quieres responder, reaccionar o mover el issue desde aquí, eso se delega en un agente de **escritura** (`save_comment`) aparte — nunca este.

---

## Flujo del agente

### Paso 0 — Contexto

- Resolver el equipo Back-end (nombre o key `BACK`). Cachear su `team_id` en el fichero de estado.
- `list_cycles(teamId, type="current")` → fechas y número del ciclo; calcular **% transcurrido** y **días hábiles restantes**.
- Fijar "hoy" y la semana en curso.

### Paso 1 — Recopilación (Linear, en paralelo)

- `list_issues(assignee="me", team="Back-end", limit=250)` sin filtrar por estado → traer todas mis issues abiertas y clasificarlas localmente por estado. (Filtrar por estado en varias llamadas también vale; una sola llamada amplia suele bastar.) **El nombre del proyecto viene en el listado**, así que el soporte (prefijo `[SOPORTE]`) se detecta ya aquí, sin llamadas extra.
- Solo para las **In progress** (y para confirmar bloqueos), `get_issue(id, includeRelations=true)` → `Blocks`/`Blocked by`, `stateHistory` (antigüedad en el estado) y relaciones. No hace falta para detectar soporte.
- Opcional: `list_issues(assignee="me", team="Back-end", state="Done", updatedAt="-P14D")` → cerradas del ciclo, como contexto del veredicto.
- **Menciones** (ver sección "Menciones"): cachear tu `mention_handle` (`@jaime.hernandez`, vía `get_user`) y tu `user_id`; luego `list_comments` sobre el conjunto del alcance elegido (A por defecto). Quedarse con los comentarios cuyo `body` contiene tu handle, escritos por otra persona y sin respuesta tuya posterior en el hilo. Es la parte más cara en llamadas: limitar por `mention_lookback_days` y por issues activas.

### Paso 2 — Clasificación y cálculo (local, sin red)

Por cada issue: estado, prioridad, estimación (→ días), due date, ciclo, proyecto, labels, soporte sí/no, semáforo correspondiente (empezar / vas-bien). A nivel ciclo: puntos totales asignados, hechos, en curso, pendientes; proyección = ¿caben los puntos pendientes en los días hábiles restantes?

### Paso 3 — Presentación

Mostrar en texto plano, en este orden y formato:

```
═══ CÓMO VOY · {fecha} · ciclo {N} ({inicio}–{fin}, {%} transcurrido) ═══

▶ HOY                                                        [🟢/🟡/🔴]
En progreso (sigue con esto):
  • BACK-123 {título}                                        vas {🟢/🟡/🔴}
      {pts}pt (~{tiempo}) · {prioridad} · {proyecto} · Soporte: {sí/no}
      Lleva {n} días en curso (estimada en {tiempo}). {Due {due} | sin due → fin de ciclo}.
      {nota de estado: depende de Infra / a punto de cerrar / se está alargando}
      → {qué hacer hoy con ella}
Empieza hoy (Urgent o vence hoy):
  • BACK-130 {título}                                        🔴 deberías haberla empezado
      {pts}pt (~{tiempo}) · Urgent · {proyecto} · Soporte: {sí/no}
      {por qué es de hoy: prioridad Urgent / vence hoy / no llegas si no empiezas}
Bloqueadas (desbloquear o escalar):
  • BACK-118 {título} — bloqueada por {issue/motivo} · {prioridad}
      → {a quién escalar / qué desbloquea}
{Resumen del día: "3 cosas hoy — cierra BACK-123, arranca BACK-130, desbloquea BACK-118"}

▶ CICLO {N}                                                  [🟢/🟡/🔴]
Puntos (tuyos): {hechos}/{total} (~{%}) · en curso {x} · pendientes {y}
Días hábiles restantes: {d} (con hoy {d+1}) · Trabajo restante: {y}pt ≈ ~{días} → {cabe con holgura / justo / no cabe}
Ritmo: {% hecho} vs {% días hábiles consumidos} → {por delante / a la par / por detrás}

En curso ({x}):
  • BACK-140 {título} — {pts}pt · {prioridad} · {proyecto} · vas {🟢/🟡/🔴} · {nota corta}
Deberías haber empezado ({n}):
  • BACK-127 {título} — {pts}pt · {prioridad} · {due/—} · {motivo}  🔴
Pendientes, por orden de ataque sugerido ({y}):
  • BACK-141 {título} — 8pt · Medium · [SOPORTE] SFTP · Soporte: SÍ   ← no dejar para el final
  • BACK-130 {título} — 8pt · Medium · Platform Engineering
  • BACK-144 {título} — 8pt · Medium · {proyecto}
  • BACK-143 {título} — 8pt · Low · {proyecto}
  • BACK-111 {título} — 2pt · Medium · {proyecto}
Hechas este ciclo ({n}): BACK-142, BACK-145, BACK-139   ({pts} pt cerrados)

▶ BACKLOG                                                    [🟢/🟡/🔴]
  • BACK-140 {título} ({pts}pt · {prioridad})  {soporte?}
Higiene (incumplen reglas Back-end):
  • BACK-141 sin estimación   • BACK-142 sin due date   • BACK-143 sin label

▶ MENCIONES (pendientes de responder · alcance: {A/B/C})     [🟢/🟡/🔴]
  • BACK-88  hace {n} días · de {quién}
      "…{fragmento corto del comentario}"
      → {enlace directo al comentario}
  • FUNC-75  hace {n} días · de {quién}   (no eres asignado)
      "…{fragmento}"
      → {enlace}
{si vacío} Nadie espera respuesta tuya en comentarios. 🟢

═══ VEREDICTO GLOBAL: {🟢 vas bien / 🟡 justo / 🔴 vas mal} ═══
{una línea de resumen accionable: "céntrate en BACK-130 y BACK-127 hoy"}
```

Reglas de presentación:
- **HOY y CICLO van detallados.** En HOY, cada tarea lleva 2–4 líneas: puntos (y su equivalente en tiempo), prioridad, proyecto, soporte, antigüedad en el estado vs estimación, una nota de situación y el "qué hacer". En CICLO se listan **todas** las pendientes (no solo un ejemplo), en orden de ataque sugerido, además de en curso, las que deberías haber empezado, el ritmo y las cerradas.
- **BACKLOG y MENCIONES van compactos**: una línea por ítem (las menciones añaden fragmento + enlace).
- Si un bloque está vacío, decirlo en una línea ("HOY: nada urgente, sigue con lo de In progress") en vez de omitirlo en silencio.
- Resaltar siempre soporte y bloqueos.
- Detallado **no** es volcar la descripción entera: resumir en una nota corta y propia, nunca pegar párrafos largos del issue.

---

## Reglas de negocio

### Qué cuenta y cómo

- **Estados activos** que se analizan: `In progress`, `Blocked`, `To do`, `Backlog` + estados de soporte (`Triage`, `Analyzing`, `Hotfix`, `Next release`).
- **`Done`** → no se lista (sirve solo de contexto para el veredicto del ciclo).
- **`Canceled` / `Duplicate`** → se ignoran.
- **Sin estimación** → no se puede calcular "¿llego?"; se marca en **Higiene** y se trata como riesgo (no se asume 0).
- **Sin due date** → usar el **fin de ciclo** como fecha límite implícita para los cálculos, y marcarlo en Higiene (el due lo asignan los POs al inicio del ciclo).
- **Sin label temático** → marcar en Higiene (toda tarea Back-end lleva al menos uno).

### Días hábiles

`días hábiles entre A y B = días lun–vie en el rango − festivos del calendario que caigan en día laborable`. Se usa para "¿llego al due?" y para la proyección del ciclo, de modo que un festivo no cuente como día disponible.

### Veredicto por nivel

- **HOY** → 🔴 si hay algo Urgent/vencido sin empezar o bloqueos sin gestionar; 🟡 si hay In progress en estado "justo"; 🟢 si todo lo de hoy va con margen.
- **CICLO** → comparar puntos pendientes con días hábiles restantes: 🟢 caben con holgura · 🟡 caben justo · 🔴 no caben o hay tareas que deberían haber empezado.
- **BACKLOG** → 🟢 limpio · 🟡 acumulando o con higiene pendiente · 🔴 hay Urgent/High esperando en backlog sin ciclo.

---

## Restricciones

- **SOLO LECTURA.** El agente **nunca** llama a `save_issue`, `save_document`, `save_comment` ni a ningún tool de escritura de Linear. Solo informa.
- Si detecta algo que convendría cambiar (mover a `Blocked`, estimar, poner due, etc.), lo **sugiere en el resumen** pero **no lo ejecuta**. Para actuar, derivar a un agente de escritura aparte o pedírselo explícitamente al usuario.
- **No inventa datos**: si falta estimación o due, lo señala; no los rellena ni los asume.
- **No psicoanaliza el ritmo del usuario**: el semáforo se basa solo en datos de Linear + calendario, no en juicios sobre la persona.
- Marcar como **aproximado** cualquier cálculo basado en campos no presentes en el listado (p. ej. antigüedad en estado si no hay `startedAt`).

---

## Instalación en Claude Code

### MCP requerido en `~/.claude.json` (o `~/.claude/settings.json`)

```json
{
  "mcpServers": {
    "linear": {
      "type": "http",
      "url": "https://mcp.linear.app/mcp"
    }
  }
}
```

La primera vez, Linear pedirá autorizar por OAuth en el navegador. `assignee="me"` queda ligado a esa cuenta.

### Skill en `~/repos/skills-ai/skills/` (o donde tengas las skills)

- `linear-rules-backend/SKILL.md`

### Fichero de estado (`~/.claude/linear-resumen.json`)

Opcional pero recomendado: cachea el `team_id` y guarda tus preferencias para no tener que repetirlas. Si no existe, el agente lo crea con valores por defecto.

```json
{
  "team_key": "BACK",
  "team_id": null,
  "hours_per_day": 8,
  "work_days": [1, 2, 3, 4, 5],
  "points_to_hours": 1,
  "participacion": "assignee",
  "user_id": null,
  "mention_handle": "@jaime.hernandez",
  "mention_scope": "A",
  "mention_lookback_days": 14,
  "soporte": {
    "project_prefixes": ["[SOPORTE]"],
    "projects": ["Support issues", "Customer Support"],
    "states": ["Triage", "Analyzing", "Hotfix", "Next release"],
    "title_prefixes": ["Soporte", "[Soporte]"],
    "related_via": "Relates to"
  },
  "ciclo_dias": 14
}
```

- `team_id` — se rellena en el primer run para no resolver el equipo cada vez.
- `points_to_hours` / `hours_per_day` / `work_days` — convierten estimación en días hábiles. Permiten jornada o convención distintas sin tocar la lógica.
- `participacion` — hoy `"assignee"` (única vía fiable del MCP; en la práctica cubre casi todo tu trabajo). Reservado por si más adelante se amplía a suscriptor.
- `user_id` — tu id de Linear, para saber **quién** escribió cada comentario y si ya respondiste (`author.id`). Se rellena solo desde `assigneeId` en el primer run. El tuyo es `d62e52b3-7d35-4673-9168-91499345cdb6`.
- `mention_handle` — tu handle para detectar menciones en comentarios (texto plano `@handle`). Resolver con `get_user` y cachear. El tuyo es `@jaime.hernandez`.
- `mention_scope` / `mention_lookback_days` — alcance del barrido de menciones (`"A"`, `"B"` o `"C"`) y ventana de días hacia atrás. Ver sección "Menciones".
- `soporte` — cómo se detecta el soporte. El más fiable es `project_prefixes` (`[SOPORTE]`); el resto son señales de respaldo. Editable si cambian proyectos/estados.

### Calendario laboral — festivos Madrid capital (para "días hábiles")

Reutilizado del agente de horas. Solo afectan los que caen en **día laborable** (los de fin de semana no descuentan). Mantener actualizado cada año.

**2026** (festivos en día laborable + días de empresa):

| Fecha | Día | Festivo |
|---|---|---|
| 2026-01-01 | jue | Año Nuevo |
| 2026-01-06 | mar | Epifanía |
| 2026-04-02 | jue | Jueves Santo |
| 2026-04-03 | vie | Viernes Santo |
| 2026-05-01 | vie | Fiesta del Trabajo |
| 2026-05-15 | vie | San Isidro (local) |
| 2026-10-12 | lun | Fiesta Nacional |
| 2026-11-02 | lun | Traslado Todos los Santos |
| 2026-11-09 | lun | La Almudena (local) |
| 2026-12-07 | lun | Traslado Constitución |
| 2026-12-08 | mar | Inmaculada |
| 2026-12-24 | jue | Nochebuena (empresa) |
| 2026-12-25 | vie | Natividad |
| 2026-12-31 | jue | Nochevieja (empresa) |

> Las 2 fiestas locales (San Isidro, La Almudena) y el 24/31-dic (empresa) no las trae ninguna API genérica de festivos; por eso el calendario vive aquí.

### Uso

```bash
claude "cómo voy"
```
