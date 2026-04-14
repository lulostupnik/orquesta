# Jarvis PM — El Agente que Elimina Reuniones Innecesarias

> "NO NECESITAS ESTA REUNION." — Jarvis PM te dice por que, cuanto te costaria, y que hacer en su lugar.

## 1. Vision

**El problema no es que las meetings son malas. El problema es que existen cuando no deberian.**

Jarvis PM es un sistema multi-agente anti-meeting. Se conecta a todas las herramientas del equipo (GitHub, Slack, Calendar, Jira) y su trabajo principal es **IMPEDIR que se pierda tiempo**:

- **Calcula el COSTO en plata de cada reunion** antes de que ocurra (salarios * tiempo * personas)
- **Dice NO**: "No necesitas esta reunion. Las 4 personas ya entendieron el tema. Manda este mensaje async."
- **Detecta QUIEN necesita mentorship** vs quien ya entiende — en vez de meter a todos en una call
- **Biohacking de productividad**: sabe cuando cada persona rinde mas y protege esas horas
- **Si la reunion ocurre igual**: la analiza, mide cuanto valio, y usa eso para bloquear futuras reuniones inutiles

**Categoria**: Open
**Stack AI**: Claude Opus (orchestrator) + Claude Sonnet (sub-agentes) + Claude Haiku (comms)
**Diferenciador**: No optimiza meetings. Las ELIMINA. Y cuando no puede eliminarlas, te dice exactamente cuanto te costo en plata.

### 1.1 El Pitch en Una Frase

Alguien quiere agendar una reunion. Jarvis responde:

> "NO NECESITAS ESTA REUNION.
> - Te va a costar $450 USD en salarios (5 personas x 1h x salario promedio)
> - 3 de 5 personas ya entienden el tema (basado en sus commits y Slack activity)
> - Mejor mandamos este resumen async a todos, EXCEPTO a Carlos
> - Carlos necesita una sesion de mentorship 1:1 de 15 min con Juan (que es quien mas toco ese codigo)
> - Ahorro estimado: $360 y 4 horas-persona de deep work protegidas"

---

## 2. Arquitectura Multi-Agente

### 2.1 Diagrama General

```
                         ┌──────────────────────────┐
                         │    JARVIS ORCHESTRATOR    │
                         │    Claude Opus            │
                         │    Decision Engine +      │
                         │    Extended Thinking      │
                         └─────────────┬────────────┘
                                       │
        ┌──────────┬───────────┬───────┼────────┬──────────┐
        ▼          ▼           ▼       ▼        ▼          ▼
  ┌──────────┐┌──────────┐┌────────┐┌────────┐┌────────┐┌────────┐
  │ PULSE    ││ FLOW     ││MEETING ││ TASK   ││ COMMS  ││INSIGHT │
  │ AGENT    ││ DETECTOR ││ INTEL  ││ AGENT  ││ AGENT  ││ AGENT  │
  │Productiv.││Blockers  ││Analiza ││Asigna  ││Standups││Reportes│
  │& Ritmos  ││& Friction││reunions││tareas  ││& Notif ││& KPIs  │
  └────┬─────┘└────┬─────┘└───┬────┘└───┬────┘└───┬────┘└───┬────┘
       │           │          │         │         │         │
  ┌────▼───────────▼──────────▼─────────▼─────────▼─────────▼────┐
  │                    MCP SERVER LAYER                            │
  │  GitHub · Google Calendar · Slack · Linear · Google Drive     │
  │  CI/CD  · Clockify · Notion · Confluence                      │
  └──────────────────────────────────────────────────────────────┘
```

### 2.2 Agentes y sus roles

| Agente | Modelo | Rol |
|--------|--------|-----|
| **Jarvis Orchestrator** | Opus | Cerebro central. Recibe requests, decide que agente actua, sintetiza resultados, toma decisiones con Extended Thinking |
| **Pulse Agent** | Sonnet | Analiza ritmos de productividad, peak hours, impacto de meetings en output |
| **Flow Detector Agent** | Sonnet | Detecta bloqueos, friccion en PRs, patrones de Slack que indican problemas |
| **Meeting Intel Agent** | Sonnet | Analiza meetings existentes, decide si una meeting es necesaria, sugiere cancelaciones |
| **Task Agent** | Sonnet | Crea, asigna y prioriza tareas basandose en carga, skills y disponibilidad |
| **Comms Agent** | Haiku | Maneja standups asincronos, notificaciones, resumenes |
| **Insight Agent** | Sonnet | Genera reportes, dashboards, KPIs compuestos cruzando multiples fuentes |

---

## 3. Agentes en Detalle

### 3.1 PULSE Agent — Ritmos de productividad

**Conectado a**: GitHub MCP + Google Calendar MCP

Analiza POR CADA developer:

- **Commits por hora del dia** — detecta peak hours (ej: "Juan commitea 3x mas entre 10am-1pm")
- **Commits por dia de la semana** — detecta patrones semanales
- **Tiempo entre commits** — detecta flow states vs fragmentacion
- **Horas en meetings vs horas de codigo** — ratio meeting/code per dev
- **Impacto de meetings en output** — "despues de meetings de >1hr, el output de Juan cae 40% por 2 horas"
- **Deep work windows** — identifica bloques donde NUNCA deberia haber meetings

**KPIs generados:**

| KPI | Que mide |
|-----|----------|
| `productivity_score_by_hour` | Output relativo por franja horaria por dev |
| `meeting_cost_index` | Horas de codigo perdidas por cada hora de meeting |
| `deep_work_ratio` | % del dia en bloques ininterrumpidos de >2hrs |
| `flow_fragmentation_score` | Cuantas veces se interrumpe un flow state por meetings |
| `post_meeting_recovery_time` | Minutos hasta volver al ritmo normal de commits |

---

### 3.2 FLOW DETECTOR Agent — Deteccion de bloqueos e interacciones

**Conectado a**: GitHub MCP + Slack MCP

Detecta cuando algo anda mal:

- **Bloqueo por codigo**: un dev lleva X tiempo sin commits en un branch, o tiene muchos commits revertidos — esta trabado
- **Bloqueo por review**: un PR lleva >N horas sin review — bottleneck
- **Friction en PRs**: muchos comment threads, muchas rondas de review — hay desacuerdo tecnico, puede necesitar meeting
- **Patrones de Slack**: si dos devs estan yendo y viniendo en un thread por >10 mensajes — "esto necesita una call de 15 min, no mas Slack"
- **Silos de interaccion**: mapea quien habla con quien en Slack y PRs — detecta si alguien esta aislado o si hay sub-equipos que no se comunican
- **Sentiment en interacciones**: tono de los mensajes en Slack y PR comments — detecta frustracion temprana

**KPIs generados:**

| KPI | Que mide |
|-----|----------|
| `blocker_detection_time` | Tiempo desde que un dev se traba hasta que Jarvis lo detecta |
| `pr_review_bottleneck_hours` | Tiempo promedio de PR esperando review |
| `slack_to_meeting_escalation_rate` | % de threads largos que terminaron necesitando call |
| `interaction_graph_density` | Que tan conectado esta el equipo (vs silos) |
| `friction_score_per_pr` | Nivel de friccion en cada PR (comments, re-requests, tone) |

---

### 3.3 MEETING KILLER Agent — Eliminacion de reuniones innecesarias

**Conectado a**: Google Calendar MCP + Slack MCP + GitHub MCP + Transcription Service

**Filosofia**: La reunion por defecto NO deberia existir. Este agente actua como un GUARDIAN que bloquea reuniones y solo las deja pasar si realmente se justifican.

#### 3.3.1 PRECIO vs VALOR — Estimacion pre-reunion

Cuando alguien intenta agendar una reunion, Jarvis calcula ANTES:

**COSTO de la reunion:**
- Salario/hora de cada asistente * duracion estimada = costo en USD
- Horas de deep work perdidas (contando recovery time post-meeting)
- Costo de oportunidad: que podrian estar haciendo esas personas

**VALOR estimado de la reunion:**
- Es un tema que requiere debate en tiempo real? O es informativo?
- Cuantas personas REALMENTE necesitan estar? (basado en quien toco el codigo/tema)
- Se puede resolver con un mensaje async?

**Decision de Jarvis:**

```
REUNION SOLICITADA: "Sprint planning" - 5 personas - 1 hora
    │
    ▼
COSTO ESTIMADO: $450 USD en salarios
    │
    ▼
ANALISIS:
    ├── 3 de 5 personas ya entienden el scope (commits recientes + Slack)
    ├── 1 persona (Carlos) necesita contexto — NO necesita reunion, necesita MENTORSHIP
    ├── 1 persona (Ana) no toco nada relacionado — NO necesita estar
    │
    ▼
JARVIS RESPONDE: "NO NECESITAS ESTA REUNION"
    ├── Mando resumen async a Juan, Maria, Pedro (ya entienden)
    ├── Agendo mentorship 1:1 Carlos con Juan (15 min) — Juan es quien mas toco ese codigo
    ├── Ana no necesita estar — la saco
    ├── Ahorro: $360 USD + 4 horas-persona de deep work
    └── Si insistis en hacerla: la agenda optimizada es [X] y solo necesita 20 min, no 1h
```

#### 3.3.2 Deteccion de quien necesita MENTORSHIP vs quien ya sabe

Esto es clave — en vez de meter a todos en una call, Jarvis identifica:

- **Quien ya entiende**: tiene commits recientes en ese area, participo en PRs relevantes, respondio preguntas sobre el tema en Slack
- **Quien necesita mentorship**: no toco ese codigo, hizo preguntas basicas sobre el tema, tiene PRs con muchas correcciones en esa area
- **Quien es el mejor mentor**: el dev que mas contribuyo a esa area, que tiene reviews aprobados rapido, que otros le consultan

**Resultado**: en vez de 1 reunion de 1h con 5 personas, Jarvis genera:
- 1 mensaje async para los que ya saben
- 1 sesion de mentorship 1:1 de 15 min para quien lo necesita
- 0 tiempo perdido para quien no tiene nada que ver

#### 3.3.3 Anti-meeting intelligence (core del producto)

- "NO: esta meeting se puede cancelar — nadie tiene updates"
- "NO: esto se resuelve con un mensaje async. Aca esta el mensaje listo para enviar."
- "NO: solo Carlos necesita entender esto. Le agendo 15 min con Juan, no una hora con todos."
- "NO: es mas barato que todos lean este doc de 2 paginas que pagar $600 en salarios por esta call."
- "SI, pero solo 20 min con Juan y Maria. Los otros 3 no necesitan estar."

#### 3.3.4 Trigger de reuniones (solo cuando es REALMENTE necesario)

Jarvis solo sugiere reunion cuando el costo de NO hacerla es mayor que hacerla:

- 2+ devs trabados en el mismo problema y ninguno lo sabe — necesitan hablar, no hay forma async
- Conflicto tecnico que lleva >3 dias sin resolverse en PRs — necesita decision en vivo
- Deadline en riesgo y el equipo no esta alineado — sync de emergencia

#### 3.3.1 Transcripcion y Analisis en Tiempo Real de Meetings

Jarvis graba y transcribe cada meeting (via integracion con Google Meet, Zoom, o un bot de recording) y corre analisis en dos momentos:

**DURANTE la meeting (tiempo real via streaming de transcript):**

- **Deteccion de off-topic**: Jarvis conoce la agenda (del calendar invite o Jira tickets vinculados). Si la conversacion se desvia del tema, genera una alerta silenciosa al host: "Llevan 8 minutos hablando de infra cuando la meeting es sobre el feature de pagos. Quedan 12 minutos."
- **Deteccion de rabbit holes**: si dos personas estan debatiendo un detalle tecnico por >5 min sin llegar a conclusion — "Esto se puede resolver offline en un PR. Sugerencia: seguir con el siguiente punto."
- **Deteccion de temas que no deberian discutirse en vivo**: temas sensibles (performance reviews, compensation), o temas que requieren preparacion previa (architecture decisions sin RFC) — "Este tema necesita un doc escrito antes de discutirse. Agendo follow-up."
- **Participation tracking**: quien habla cuanto. Si una persona monopoliza >60% del tiempo, alerta. Si alguien no hablo en toda la meeting, flagea que probablemente no necesitaba estar.
- **Decision capture**: detecta cuando se toma una decision ("ok, entonces vamos con X") y la registra automaticamente como action item.

**DESPUES de la meeting (analisis post-mortem):**

- **Meeting score (0-100)**: basado en:
  - % del tiempo en tema vs off-topic
  - Cantidad de action items generados
  - Cantidad de decisiones tomadas
  - Distribucion de participacion (equitativa = mejor)
  - Duracion real vs duracion agendada
- **Topic drift map**: timeline visual de de que se hablo minuto a minuto, marcando donde se desvio la conversacion
- **"Esta meeting debio ser un email"**: score de si el outcome se podria haber logrado asincrono
- **Action items automaticos**: extrae todos los compromisos, los asigna en Jira/Linear, y hace follow-up
- **Resumen ejecutivo**: genera un resumen de 3-5 bullets que se postea automaticamente en Slack
- **Pattern detection across meetings**: despues de N meetings, detecta patrones como:
  - "Las reuniones de planning siempre se desvian en el minuto 20 hacia deuda tecnica"
  - "Las 1:1 de Juan con Maria siempre terminan 15 min antes — reducir a 30 min"
  - "Las retros nunca generan action items — cambiar formato"

**Integracion de transcripcion:**

| Plataforma | Metodo |
|-----------|--------|
| Google Meet | Bot que se une a la call + Speech-to-Text API |
| Zoom | Zoom Recording API + transcript |
| Microsoft Teams | Teams Graph API + transcription |
| Alternativa ligera | Otter.ai / Fireflies.ai API como fuente de transcripts |

**KPIs generados:**

| KPI | Que mide |
|-----|----------|
| `meetings_killed` | Reuniones eliminadas por Jarvis esta semana/mes |
| `money_saved_usd` | Dolares ahorrados en salarios por reuniones eliminadas |
| `hours_returned_to_deep_work` | Horas devueltas al equipo por no tener meetings inutiles |
| `meeting_cost_avg` | Costo promedio en USD de cada reunion que SI se hizo |
| `meeting_roi_score` | Valor generado (action items, decisions) / costo en USD |
| `mentorship_sessions_created` | Sesiones 1:1 creadas en lugar de meetings grupales |
| `async_resolution_rate` | % de problemas resueltos sin necesidad de meeting |
| `unnecessary_meeting_rate` | % de meetings propuestas que Jarvis bloqueo |
| `meeting_score` | Score 0-100 de las meetings que SI ocurrieron |
| `on_topic_ratio` | % del tiempo de meeting dedicado al tema agendado |
| `topic_drift_frequency` | Veces por meeting que la conversacion se desvia |
| `participation_balance` | Distribucion de tiempo de habla entre asistentes (Gini coefficient) |
| `decision_rate` | Decisiones tomadas por hora de meeting |
| `should_have_been_async_rate` | % de meetings que no necesitaban ser sincronicas (post-mortem) |
| `mentorship_accuracy` | Devs identificados como "necesita mentorship" que efectivamente mejoraron despues |
| `who_knows_what_map` | Mapa de expertise: quien sabe que, basado en commits + PRs + Slack |

---

### 3.4 TASK Agent — Asignacion inteligente de tareas

**Conectado a**: Linear/Jira MCP + GitHub MCP

- Asigna tareas basandose en: skills del dev, carga actual, disponibilidad, peak hours
- Re-prioriza automaticamente cuando detecta bloqueos
- Sugiere redistribucion si un dev esta sobrecargado
- Estima tiempos basandose en historico del equipo

**KPIs generados:**

| KPI | Que mide |
|-----|----------|
| `sprint_velocity` | Story points completados por sprint |
| `task_completion_rate` | Tareas completadas / tareas planeadas |
| `task_age` | Dias que una tarea lleva en "In Progress" sin movimiento |
| `scope_creep_index` | Tareas agregadas al sprint despues del planning |
| `blocked_task_duration` | Tiempo promedio que una tarea esta en "Blocked" |
| `workload_balance` | Distribucion de tareas/puntos entre devs |
| `estimation_accuracy` | Story points estimados vs tiempo real |

---

### 3.5 COMMS Agent — Comunicacion asincrona

**Conectado a**: Slack MCP + Google Calendar MCP

- Ejecuta standups asincronos: le pregunta a cada dev su status en Slack
- Consolida respuestas y alerta si hay blockers
- Genera resumenes de meetings a partir de transcripts
- Notifica decisiones importantes al equipo
- Envio de alertas cuando Jarvis detecta algo

---

### 3.6 INSIGHT Agent — Reportes y dashboards

**Conectado a**: Todos los MCP servers

Genera reportes automaticos consolidando data de todas las fuentes. Es el agente que alimenta el dashboard.

---

## 4. Plataformas e Integraciones

### 4.1 Mapa completo de conexiones

```
  CODE & ENGINEERING          COMMUNICATION
  ├── GitHub/GitLab           ├── Slack
  ├── Bitbucket               ├── Microsoft Teams
  ├── Jira                    ├── Discord
  └── Linear                  └── Email (Gmail/Outlook)

  CALENDAR & TIME             DOCS & KNOWLEDGE
  ├── Google Calendar         ├── Google Drive
  ├── Outlook Calendar        ├── Notion
  └── Clockify/Toggl          ├── Confluence
                               └── Google Docs

  CI/CD & INFRA               PROJECT MANAGEMENT
  ├── GitHub Actions          ├── ClickUp
  ├── Vercel                  ├── Asana
  └── AWS CloudWatch          ├── Trello
                               └── Monday.com
```

### 4.2 KPIs automaticos por fuente

#### De GitHub/GitLab (automatico):

| KPI | Formula | Frecuencia |
|-----|---------|------------|
| `commits_per_hour_heatmap` | Commits agrupados por hora y dia per dev | Tiempo real |
| `pr_cycle_time` | Tiempo desde PR abierto hasta mergeado | Por PR |
| `review_turnaround` | Tiempo desde PR asignado a reviewer hasta primer comment | Por PR |
| `code_churn_rate` | Lineas agregadas y luego borradas en <48h (indica retrabajo) | Diario |
| `bus_factor` | Cuantos devs tocan cada area del codigo (1 = riesgo) | Semanal |
| `merge_conflict_frequency` | Conflictos de merge por branch — indica falta de coordinacion | Por merge |
| `first_commit_time` | Hora del primer commit del dia per dev | Diario |
| `last_commit_time` | Hora del ultimo commit del dia per dev | Diario |
| `avg_pr_size` | Lineas cambiadas promedio por PR (PRs grandes = riesgo) | Por PR |

#### De Slack/Teams (automatico):

| KPI | Formula | Frecuencia |
|-----|---------|------------|
| `thread_escalation_score` | Threads de >10 msgs que deberian ser call | Tiempo real |
| `response_time_avg` | Tiempo promedio de respuesta por persona | Diario |
| `interaction_matrix` | Quien habla con quien — mapa de conexiones del equipo | Semanal |
| `after_hours_messages` | Mensajes fuera de horario laboral per dev | Diario |
| `blocker_mention_rate` | Frecuencia de palabras como "blocked", "stuck", "waiting" | Tiempo real |
| `channel_noise_ratio` | Mensajes informativos vs ruido por canal | Semanal |
| `decision_latency` | Tiempo desde pregunta hasta decision en un thread | Por thread |

#### De Google Calendar/Outlook (automatico):

| KPI | Formula | Frecuencia |
|-----|---------|------------|
| `meeting_load_hours` | Horas totales en meetings per dev per semana | Semanal |
| `meeting_free_blocks` | Bloques de >2h sin meetings (deep work posible) | Diario |
| `recurring_meeting_roi` | Meetings recurrentes que no generan action items | Mensual |
| `meeting_overlap_with_peak` | % de meetings en las horas de mayor productividad | Semanal |
| `meeting_attendee_necessity` | Asistentes que nunca hablan en la meeting (no necesitan estar) | Por meeting |
| `schedule_density` | % del dia ocupado en meetings per dev | Diario |

#### De Jira/Linear/ClickUp (automatico):

| KPI | Formula | Frecuencia |
|-----|---------|------------|
| `sprint_velocity` | Story points completados por sprint | Por sprint |
| `task_completion_rate` | Tareas completadas / tareas planeadas | Semanal |
| `task_age` | Dias que una tarea lleva en "In Progress" sin movimiento | Tiempo real |
| `scope_creep_index` | Tareas agregadas al sprint despues del planning | Por sprint |
| `blocked_task_duration` | Tiempo promedio que una tarea esta en "Blocked" | Por tarea |
| `workload_balance` | Distribucion de tareas/puntos entre devs | Tiempo real |
| `estimation_accuracy` | Story points estimados vs tiempo real | Por sprint |

#### De CI/CD — GitHub Actions/Vercel (automatico):

| KPI | Formula | Frecuencia |
|-----|---------|------------|
| `build_success_rate` | Builds exitosos / total | Diario |
| `deploy_frequency` | Deploys a produccion por semana | Semanal |
| `build_fix_time` | Tiempo desde build roto hasta fix | Por build |
| `test_flakiness` | Tests que fallan intermitentemente | Semanal |

---

## 5. KPIs Compuestos — Cruzando Fuentes

Estos son los killer KPIs — Jarvis los calcula cruzando data de multiples plataformas:

| KPI | Cruza | Que revela |
|-----|-------|------------|
| `productivity_impact_of_meetings` | Calendar + GitHub | "Despues de meetings >1h, el output de commits cae X% por N horas" |
| `collaboration_health_score` | Slack + GitHub + Calendar | Score 0-100 de que tan sano es el trabajo en equipo |
| `burnout_risk_indicator` | GitHub (commits nocturnos) + Slack (after hours) + Calendar (meeting overload) | Alerta temprana de burnout per dev |
| `optimal_meeting_windows` | Calendar + GitHub (peak hours) | "El mejor horario para reunir a todo el equipo sin matar productividad es Martes 3-4pm" |
| `blocker_cost_in_hours` | Jira (blocked duration) + GitHub (idle branches) + Slack (help requests) | Cuantas horas de productividad se perdieron por blockers |
| `team_sync_score` | Slack (interactions) + GitHub (co-authored PRs) + Calendar (shared meetings) | Que tan alineado esta el equipo |
| `meeting_necessity_predictor` | Slack (thread length) + GitHub (PR friction) + Jira (stale tasks) | Predice cuando se VA a necesitar una meeting antes de que sea urgente |
| `developer_experience_score` | Build times + PR review speed + blocker duration + meeting load | Score general de la experiencia del dev |
| `meeting_effectiveness_trend` | Transcripts (meeting scores) + Calendar (recurrence) | Tendencia de calidad de meetings en el tiempo — estan mejorando o empeorando? |
| `topic_drift_cost` | Transcripts (off-topic time) + Calendar (meeting duration) | Horas por semana perdidas en conversaciones off-topic |
| `async_potential_score` | Transcripts (decision count, complexity) + Slack (async resolution history) | "El 40% de tus meetings podrian ser async basado en el tipo de decisiones que se toman" |
| `meeting_tax_rate` | Calendar (meeting hours) + GitHub (salaries estimate) | % del presupuesto de salarios que se va en reuniones |
| `mentorship_vs_meeting_efficiency` | Transcripts + GitHub (post-mentorship improvement) | "Las sesiones 1:1 de 15 min mejoran el output 3x mas que meetings grupales de 1h" |
| `expertise_coverage` | GitHub (code ownership) + Slack (who answers what) | Mapa de quien sabe que — detecta bus factor y gaps de conocimiento |

---

## 6. Dashboard — Vista del Usuario

```
┌──────────────────────────────────────────────────────────────┐
│  JARVIS PM — Meeting Killer Dashboard                   Live │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  $ SAVED THIS MONTH: $4,230 USD        Meetings killed: 23  │
│  Hours returned to deep work: 47h      Team Health: 82/100  │
│                                                              │
│  ┌─ BLOCKED BY JARVIS (recent) ───────────────────────────┐ │
│  │ X "Sprint Planning" — $450 cost, 3/5 already know      │ │
│  │   -> Sent async summary + 1:1 mentorship Carlos<>Juan  │ │
│  │ X "Design Review" — $300 cost, PR already approved      │ │
│  │   -> Sent Slack summary, saved 1h for 4 people         │ │
│  │ X "Standup Wed" — $150 cost, no blockers detected      │ │
│  │   -> Nobody had updates. Skipped.                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─ Biohacking: Peak Hours ────┐ ┌─ Money Tracker ────────┐ │
│  │ Juan:   10am-1pm  [=====  ] │ │ Meetings this week: 3  │ │
│  │ Maria:   9am-12pm [====   ] │ │ Meetings killed: 7     │ │
│  │ Carlos:  2pm-5pm  [  ====]  │ │ Cost of remaining: $540│ │
│  │ Ana:     8am-11am [====   ] │ │ Cost avoided: $2,100   │ │
│  │ Pedro:   1pm-4pm  [ ====  ] │ │ ROI of meetings: 72/100│ │
│  │ PROTECTED: no meetings in   │ │ "Should be async": 1   │ │
│  │ peak hours this week        │ └────────────────────────┘ │
│  └─────────────────────────────┘                            │
│                                                              │
│  ┌─ Who Knows What (Expertise Map) ───────────────────────┐ │
│  │ auth-system:  Juan (expert), Maria (knows), Carlos (!) │ │
│  │ payments:     Maria (expert), Pedro (knows)            │ │
│  │ frontend:     Ana (expert), Juan (knows)               │ │
│  │ infra:        Pedro (expert), Carlos (knows)           │ │
│  │                                                         │ │
│  │ (!) Carlos needs mentorship on auth — suggest 1:1 Juan │ │
│  │ (!) No backup for Pedro on infra — bus factor risk     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─ Last Meeting That DID Happen: Incident Review (30m) ──┐ │
│  │ Cost: $270 | Score: 88/100 | JUSTIFIED                  │ │
│  │ On-topic: 92% | Decisions: 4 | Action items: 6         │ │
│  │ All participants needed to be there                     │ │
│  │ Jarvis verdict: "This meeting earned its cost."         │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─ Mentorship Radar ─────────────────────────────────────┐ │
│  │ Carlos -> needs help with auth (3 reverted commits)    │ │
│  │   Best mentor: Juan (15 min 1:1 > 1h group meeting)   │ │
│  │ Pedro -> isolated on infra (low interaction score)     │ │
│  │   Suggestion: pair with Maria on next infra task       │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─ Jarvis Actions Today ─────────────────────────────────┐ │
│  │ KILLED: 3 meetings (saved $720, returned 6h)           │ │
│  │ CREATED: 1 mentorship session Carlos<>Juan             │ │
│  │ SENT: 2 async summaries instead of meetings            │ │
│  │ PROTECTED: all peak hours free from meetings           │ │
│  │ ALERTED: Juan stuck on auth-branch — helped async      │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. Flujos de Decision del Orchestrator

### 7.1 Deteccion de bloqueo

```
GitHub: no commits on branch for >3h
    │
    ▼
Flow Detector: confirm blocker
    ├── Check Slack: is dev asking for help? 
    ├── Check PRs: is dev waiting on review?
    └── Check Calendar: was dev in meetings?
    │
    ▼
Orchestrator decides:
    ├── Dev in meetings all morning → "Not blocked, just busy. Will check again at 2pm"
    ├── Dev asking in Slack with no response → Alert team + suggest reviewer
    ├── Dev quiet everywhere → DM via Comms Agent: "Need help with [branch]?"
    └── PR blocked on review → Notify reviewer + escalate if >8h
```

### 7.2 Meeting necessity decision

```
Signal detected (stale task / long thread / PR friction)
    │
    ▼
Meeting Intel evaluates:
    ├── Can this be resolved async? (Slack message, PR comment)
    ├── How many people need to be involved?
    ├── What is the cost of NOT meeting? (blocked work, timeline risk)
    │
    ▼
Decision:
    ├── LOW urgency + 1-2 people → Slack DM suggestion
    ├── MED urgency + 2-3 people → Suggest 15min call at optimal time
    ├── HIGH urgency + team-wide → Schedule sync, protect deep work hours
    └── NO meeting needed → Send async summary instead
```

### 7.3 Schedule optimization

```
New meeting request received
    │
    ▼
Scheduler checks per attendee:
    ├── Peak productivity hours (from Pulse Agent)
    ├── Current meeting load this week
    ├── Deep work blocks already scheduled
    ├── Time zones (if distributed team)
    │
    ▼
Scheduler proposes:
    ├── Best slot: minimizes productivity impact across all attendees
    ├── Alt slot: if best is unavailable
    └── Recommendation: "Consider async instead" (if meeting score is low)
```

---

## 8. Features de Claude Utilizados

| Feature | Donde se usa |
|---------|-------------|
| **Multi-agent (Agent SDK)** | Orchestrator coordina 6 sub-agentes especializados |
| **MCP Servers** | Conexion a GitHub, Slack, Calendar, Jira, Drive |
| **Extended Thinking** | Orchestrator razona sobre decisiones complejas (cancelar meeting? escalar bloqueo?) |
| **Tool Use** | Cada agente usa herramientas especificas de su dominio |
| **Claude Opus** | Orchestrator — decisiones de alto nivel |
| **Claude Sonnet** | Sub-agentes de analisis — balance velocidad/calidad |
| **Claude Haiku** | Comms agent — respuestas rapidas, notificaciones |

---

## 9. Tech Stack

| Capa | Tecnologia | Razon |
|------|-----------|-------|
| **Orchestrator** | Python + Claude Agent SDK | SDK oficial para multi-agent |
| **MCP Servers** | MCP protocol (stdio/SSE) | Protocolo estandar de Claude para integraciones |
| **Backend API** | FastAPI (Python) | Rapido de levantar, async nativo, buen match con Agent SDK |
| **Frontend Dashboard** | Next.js + Tailwind + Recharts | Dashboard reactivo con graficos de KPIs |
| **Base de datos** | SQLite (hackaton) / PostgreSQL (prod) | Simple para la demo, migrable despues |
| **Real-time updates** | WebSockets (FastAPI -> Next.js) | Dashboard en tiempo real |

---

## 10. Guion de Demo (2 minutos)

### 0:00 - 0:15 — Hook
"Tu equipo gasto $4,200 dolares este mes en reuniones. El 60% eran innecesarias. Jarvis las hubiera eliminado."

### 0:15 - 0:50 — Dashboard: el dinero habla (35 seg)
Abrir el dashboard. Lo primero que se ve: **"$4,230 SAVED THIS MONTH"**.
- Mostrar las meetings bloqueadas: "Sprint planning — $450 cost — 3 de 5 ya sabian"
- "Jarvis mando un resumen async y agendo mentorship 1:1 en vez de una reunion grupal"
- Mostrar el Expertise Map: "Jarvis sabe quien sabe que, y quien necesita ayuda"

### 0:50 - 1:25 — El Momento Wow: Jarvis dice NO en vivo (35 seg)
Alguien intenta agendar una reunion. Jarvis responde EN VIVO:
- "NO NECESITAS ESTA REUNION"
- "Te va a costar $300 en salarios"
- "Maria y Juan ya entienden — les mando resumen"
- "Carlos necesita mentorship, le agendo 15 min con Juan"
- "Ahorro: $240 y 3 horas de deep work"

### 1:25 - 1:50 — Biohacking + Transcripts (25 seg)
- Mostrar peak hours por dev: "Jarvis protege estas horas — cero meetings"
- Mostrar analisis de una meeting que SI ocurrio: score 88/100, "esta SI valio la pena"
- Contraste: otra que no — score 35/100, "esto debio ser un email"

### 1:50 - 2:00 — Cierre (10 seg)
"Las reuniones son el impuesto mas caro de tu empresa. Jarvis las audita. 6 agentes Claude, 5 MCP servers, cero reuniones innecesarias."

---

## 11. Scope para la Hackaton (6 horas)

### Must-have (demo funcional):

1. **GitHub MCP** conectado — commits por hora, expertise map, blocker detection
2. **Meeting Killer Agent** — recibe meeting request, calcula costo en USD, dice NO con razones
3. **Mentorship detection** — identifica quien necesita help vs quien ya sabe (basado en GitHub data)
4. **Dashboard web** — mostrando dinero ahorrado, meetings killed, expertise map, peak hours
5. **Orchestrator** diciendo NO a una reunion en vivo

### Nice-to-have:

6. Google Calendar MCP — data real de meetings para calcular costos
7. Slack MCP — analisis de interacciones para expertise map
8. Transcript analysis — score de meetings que SI ocurrieron
9. Pulse Agent completo — biohacking de peak hours con data real

### Demo wow-factor:

10. Alguien intenta agendar meeting → Jarvis dice NO en vivo con costo en USD
11. Jarvis sugiere mentorship 1:1 en vez de meeting grupal
12. Dashboard mostrando "$4,230 saved this month"
