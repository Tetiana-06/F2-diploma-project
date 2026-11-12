# Розділ 2. Проєктування архітектури системи

**Проєкт:** Підсистема динамічного планування та оптимізації багатофазних процесів у б'юті-індустрії  
**Методологія моделювання:** C4 Model (Simon Brown) / Clean Architecture / Domain-Driven Design (DDD)  
**Контрактна база вимог:** [Специфікація вимог (SRS — ISO/IEC/IEEE 29148)](./srs.md)  
**Журнал рішень:** [ADR-001: Контроль конкурентного доступу та неперетин інтервалів](./adr/0001-choice-of-technology.md)

---

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model)

### 2.1.1. C4 Level 1: System Context Diagram (Системний контекст) {#diagram-level1}

```mermaid
%%{init: {'flowchart': {'curve': 'stepAfter', 'nodeSpacing': 60, 'rankSpacing': 75}}}%%
flowchart TD
    %% Офіційна палітра стилів C4 Model
    classDef person fill:#08427b,stroke:#052e56,stroke-width:2px,color:#ffffff;
    classDef system fill:#1168bd,stroke:#0b4884,stroke-width:2px,color:#ffffff;
    classDef external fill:#999999,stroke:#6b6b6b,stroke-width:2px,color:#ffffff;

    %% Верхній рівень: Актори (Person)
    subgraph actors ["Користувачі системи (Actors)"]
        direction LR
        client["<b>Клієнт (Споживач)</b><br/>[Person]<br/>Формує запити на складні послуги,<br/>обирає бажані часові слоти"]:::person
        master["<b>Майстер (Виконавець)</b><br/>[Person]<br/>Переглядає персональний таймлайн,<br/>змінює фактичний статус фаз"]:::person
        admin["<b>Диспетчер / Адміністратор</b><br/>[Person]<br/>Оперує сіткою бронювань, вирішує<br/>колізії в реальному часі (Drag-and-Drop)"]:::person
    end

    %% Центральний рівень: Проєктована система (Software System)
    system["<b>Підсистема динамічного планування</b><br/>[Software System]<br/>Автономне обчислювально-аналітичне ядро<br/>планування та оптимізації багатофазних процесів"]:::system

    %% Нижній рівень: Зовнішні системи (External Systems)
    subgraph externals ["Зовнішні сервіси (External Systems)"]
        direction LR
        idp["<b>Identity Provider</b><br/>[External System]<br/>Зовнішній сервіс автентифікації<br/>(OAuth 2.0 / OIDC, JWT RS256)"]:::external
        notify["<b>Notification Gateway</b><br/>[External System]<br/>Сервіс розсилки зовнішніх повідомлень<br/>(Email / SMS / Push)"]:::external
    end

    %% Зв'язки зі стрілками під прямим кутом (90°)
    client -->|"Обирає композиційні послуги,<br/>ініціює бронювання<br/>[HTTPS / WSS]"| system
    master -->|"Отримує деталізацію активних фаз,<br/>змінює статуси<br/>[HTTPS]"| system
    admin -->|"Здійснює графічне маніпулювання<br/>таймлайном, керує ресурсами<br/>[HTTPS / WSS]"| system

    system -->|"Валідує криптографічні<br/>токени доступу<br/>[HTTPS / REST]"| idp
    system -->|"Асинхронно публікує події<br/>для сповіщень<br/>[AMQP / HTTPS]"| notify

    %% Прозорі рамки для вирівнювання рівнів
    style actors fill:none,stroke:none
    style externals fill:none,stroke:none
```

#### Технічний опис системного контексту та трасування вимог (Level 1):
* **Зовнішні актори (Person):**
  * **Клієнт (Споживач):** Взаємодіє з підсистемою через шифрований канал `HTTPS / WSS` для вибору композитних послуг та фіксації часових слотів відповідно до контексту системи ([System Context](./srs.md#1-system-context-контекст-системи)).
  * **Майстер (Виконавець) та Диспетчер / Адміністратор:** Отримують персоніфіковані розклади та керують сіткою ресурсів із розмежуванням прав доступу за ролями (RBAC), що трасується до вимоги [SRS-FUN-AUTH-02](./srs.md#21-модуль-автентифікації-та-контролю-доступу-auth).
* **Зовнішні програмні системи (`System_Ext`):**
  * **`Identity Provider`:** Забезпечує федеративну автентифікацію за протоколами OAuth 2.0 / OIDC та верифікацію криптографічних підписів токенів `JWT RS256` ([SRS-FUN-AUTH-01](./srs.md#21-модуль-автентифікації-та-контролю-доступу-auth), [SRS-NFR-SEC-01](./srs.md#3-non-functional-requirements-нефункціональні-вимоги)).
  * **`Notification Gateway`:** Виконує асинхронну доставку зовнішніх сповіщень про зміну статусу візиту через протокол `AMQP / HTTPS`.

---

### 2.1.2. C4 Level 2: Container Diagram (Діаграма контейнерів) {#diagram-level2}

```mermaid
%%{init: {'flowchart': {'curve': 'stepAfter', 'nodeSpacing': 50, 'rankSpacing': 65}}}%%
flowchart TD
    %% Офіційна палітра стилів C4 Model
    classDef person fill:#08427b,stroke:#052e56,stroke-width:2px,color:#ffffff;
    classDef external fill:#999999,stroke:#6b6b6b,stroke-width:2px,color:#ffffff;
    classDef container fill:#438dd5,stroke:#2e6295,stroke-width:2px,color:#ffffff;
    classDef db fill:#438dd5,stroke:#2e6295,stroke-width:2px,color:#ffffff;

    %% Зовнішні актори та системи
    client["<b>Клієнт</b><br/>[Person]<br/>Споживач послуг"]:::person
    admin["<b>Диспетчер</b><br/>[Person]<br/>Адміністратор розкладу"]:::person
    idp["<b>Identity Provider</b><br/>[External System]<br/>OAuth 2.0 / OIDC (JWT RS256)"]:::external

    %% Контейнерний контур системи
    subgraph b_system ["Підсистема динамічного планування (Docker Container Boundary)"]
        direction TB

        proxy["<b>Reverse Proxy / Gateway</b><br/>[Container: Nginx / Envoy]<br/>Маршрутизація, SSL Termination,<br/>Rate Limiting (Token Bucket)"]:::container

        subgraph tier_app ["Рівень клієнта та Web API"]
            direction LR
            spa["<b>Single Page Application (SPA)</b><br/>[Container: Vue.js 3, Vite, Pinia]<br/>Клієнтський інтерфейс та багатодоріжкова<br/>матриця диспетчеризації"]:::container
            api["<b>Web API Application</b><br/>[Container: FastAPI, Python 3.12]<br/>Приймає REST-запити, реалізує<br/>DDD-логіку, валідує схеми"]:::container
        end

        subgraph tier_workers ["Рівень фонових воркерів (Taskiq)"]
            direction LR
            worker_cpsat["<b>CP-SAT Solver Worker</b><br/>[Container: Python 3.12, OR-Tools]<br/>CPU-bound оптимізація розкладу<br/>RCPSP (ліміт 800ms)"]:::container
            worker_base["<b>Baseline Heuristic Worker</b><br/>[Container: Python, portion]<br/>Fallback-воркер евристичного<br/>обчислення розкладу"]:::container
            outbox_pub["<b>Outbox Publisher Worker</b><br/>[Container: Python, Taskiq]<br/>Фонове вичитання подій із БД<br/>та трансляція в брокер"]:::container
        end

        subgraph tier_data ["Рівень зберігання та брокерів"]
            direction LR
            db_pg[("<b>Main Relational Database</b><br/>[Container: PostgreSQL 16+]<br/>Зберігає сутності, DAG-послуги,<br/>Outbox-події та tstzrange")]:::db
            broker_redis[("<b>In-Memory Broker & Cache</b><br/>[Container: Redis 7.x]<br/>Черги фонових задач Taskiq<br/>та Pub/Sub шина подій")]:::db
        end
    end

    %% Зв'язки зі стрілками під прямим кутом (90°)
    client -->|"Створює бронювання<br/>[HTTPS / TLS 1.3]"| proxy
    admin -->|"Drag-and-Drop мутації<br/>[HTTPS / TLS 1.3]"| proxy

    proxy -->|"Віддає статику SPA<br/>[HTTP / 1.1]"| spa
    proxy -->|"Маршрутизує API та SSE<br/>[HTTP / 2]"| api

    spa -->|"Виклики REST API<br/>[REST / JSON]"| api
    api -->|"Події в реальному часі<br/>[HTTP EventStream / SSE]"| spa

    api -->|"Валідує JWT-токени<br/>[HTTPS / REST]"| idp
    api -->|"ACID-транзакції (GiST)<br/>[PostgreSQL / asyncpg]"| db_pg
    api -->|"Ставить задачі розрахунку<br/>[RESP3 / TCP]"| broker_redis

    worker_cpsat -->|"Читає граф / пише розклад<br/>[PostgreSQL / TCP]"| db_pg
    worker_cpsat -->|"Вичитує taskiq:solver<br/>[RESP3 / TCP]"| broker_redis

    worker_base -->|"Забирає fallback-задачі<br/>[RESP3 / TCP]"| broker_redis

    outbox_pub -->|"Вичитує таблицю Outbox<br/>[PostgreSQL / TCP]"| db_pg
    outbox_pub -->|"Публікує в Pub/Sub<br/>[RESP3 / TCP]"| broker_redis
    broker_redis -.->|"Передає події розкладу<br/>[RESP3 Pub/Sub]"| api

    %% Стиль рамки контуру
    style b_system fill:none,stroke:#666666,stroke-width:2px,stroke-dasharray: 5 5
    style tier_app fill:none,stroke:none
    style tier_workers fill:none,stroke:none
    style tier_data fill:none,stroke:none
```

#### Технічний опис контейнерів, мережевих протоколів та трасування (Level 2):
1. **`Reverse Proxy / Gateway (Nginx / Envoy)`:** Вхідний шлюз периметра. Забезпечує термінацію `TLS 1.3` ([SRS-NFR-SEC-01](./srs.md#3-non-functional-requirements-нефункціональні-вимоги)) та маршрутизацію трафіку до SPA (`HTTP / 1.1`) і бекенд-API (`HTTP / 2`).
2. **`Single Page Application (Vue.js 3)`:** Клієнтський інтерфейс диспетчеризації. Виконує запити мутацій через `REST / JSON (HTTPS)` та утримує однонапрямний потік оновлень розкладу за протоколом `HTTP EventStream (W3C SSE)` ([SRS-FUN-STREAM-02](./srs.md#25-реактивний-сервіс-трансляції-станів-stream)).
3. **`Web API Application (FastAPI)`:** Асинхронний сервіс бізнес-логіки та валідації Pydantic v2, розрахований на затримку $p_{95} \le 100$ мс при $500$ RPS ([SRS-NFR-PERF-01](./srs.md#3-non-functional-requirements-нефункціональні-вимоги)). Взаємодіє з СУБД через `PostgreSQL Wire Protocol / TCP (asyncpg)`.
4. **`CP-SAT Solver Worker` та `Baseline Heuristic Worker`:** Ізольовані у dedicated Docker-контейнерах ([SRS-NFR-DEVOPS-01](./srs.md#3-non-functional-requirements-нефункціональні-вимоги)) фонові воркери. Реалізують лексикографічну оптимізацію RCPSP (`Google OR-Tools`) з жорстким time-budgeting 800 мс та автоматичним переключенням на інтервальний евристичний аналізатор (`portion` / `intervaltree`) згідно з [SRS-FUN-SOLV-01 — SRS-FUN-SOLV-03](./srs.md#23-алгоритмічне-ядро-оптимізації-solv) та [SRS-NFR-PERF-02](./srs.md#3-non-functional-requirements-нефункціональні-вимоги).
5. **`Outbox Publisher Worker (Taskiq)`:** Вичитує невідправлені доменні події з таблиці Outbox у PostgreSQL та публікує їх у канали `Redis Pub/Sub` за протоколом `RESP3 Protocol / TCP`, ліквідуючи аномалії Dual-Write та забезпечуючи відновлення після збоїв брокера ([SRS-FUN-STREAM-01](./srs.md#25-реактивний-сервіс-трансляції-станів-stream), [SRS-NFR-REL-02](./srs.md#3-non-functional-requirements-нефункціональні-вимоги)).
6. **`Main Relational Database (PostgreSQL 16+)`:** Єдине джерело істини (Single Source of Truth). Використовує темпоральний тип `tstzrange` та індексне обмеження `EXCLUDE USING gist` (розширення `btree_gist`) для гарантування $0\%$ подвійних бронювань ([SRS-FUN-DB-01](./srs.md#24-сервіс-транзакційного-контролю-колізій-db), [SRS-NFR-REL-01](./srs.md#3-non-functional-requirements-нефункціональні-вимоги)). Архітектурне обґрунтування вибору зафіксовано в [ADR-001](./adr/0001-choice-of-technology.md).
7. **`In-Memory Broker & Cache (Redis 7.x)`:** Обслуговує асинхронні черги задач `Taskiq` (включно з механізмом вивільнення тимчасових холдів слотів за [SRS-FUN-DB-02](./srs.md#24-сервіс-транзакційного-контролю-колізій-db)) та шину подій `Pub/Sub`.

---

## 2.2. Компонентне моделювання критичного сервісу системи {#diagram-level3}

Як критичний сервіс для деталізації на рівень **C4 Component (Level 3)** обрано контейнер **`Optimization & Scheduling Service (Taskiq Worker Pool + CP-SAT Core)`**, спроєктований за принципами **Clean Architecture** та **Dependency Inversion**.

```mermaid
%%{init: {'flowchart': {'curve': 'stepAfter'}}}%%
graph TD

    subgraph TITLE ["C4 Component Diagram (Level 3) - Optimization Worker Core (Clean Architecture Final)"]
        direction TD
        style TITLE fill:none,stroke:none,font-weight:bold
    end

    %% CSS Styling for Layer Visual Isolation
    classDef external fill:#e1f5fe,stroke:#0288d1,stroke-width:2px,color:#01579b;
    classDef inbound fill:#fff3e0,stroke:#f57c00,stroke-width:2px,color:#e65100;
    classDef security fill:#ffebee,stroke:#c62828,stroke-width:2px,color:#b71c1c;
    classDef core fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#1b5e20;
    classDef algo fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#4a148c;
    classDef outbound fill:#eceff1,stroke:#455a64,stroke-width:2px,color:#263238;

    subgraph EXT ["Зовнішнє Середовище (External Containers)"]
        Broker["Broker / Redis Taskiq Queue<br/>(RESP3 Protocol)"]:::external
        MainDB["Main Database PostgreSQL 16+<br/>(tstzrange + btree_gist)"]:::external
    end

    subgraph CONTAINER ["Optimization & Scheduling Service Container"]
        
        subgraph LAYER_INBOUND ["1. Inbound Adaptation Layer"]
            Consumer["Taskiq Queue Consumer<br/>(Async Worker Process)"]:::inbound
        end

        subgraph LAYER_SECURITY ["2. Security & Validation Layer"]
            DTOValidator["DTO Validation Guard<br/>(Pydantic v2 Schema Inspector)"]:::security
            Anonymizer["UUID PII Anonymizer Guard<br/>(GDPR Compliance Sanitizer)"]:::security
        end

        subgraph LAYER_CORE ["3. Domain Core & Orchestration Layer"]
            Orchestrator["Scheduling Pipeline Orchestration Service<br/>(Time-Budgeting Coordinator)"]:::core
            DAGDecomposer["DAG Process Decomposition Service<br/>(Graph Topology Analyzer)"]:::core
            ErrorHandler["Global Exception & Fallback Handler<br/>(Resilience Manager)"]:::core
            Telemetry["Telemetry & Metrics Collector<br/>(Prometheus / Loggers)"]:::core
        end

        subgraph LAYER_ALGO ["4. Algorithmic Optimization Engine"]
            CPSATCore["CP-SAT Solver Engine<br/>(Google OR-Tools / Time Limit 800ms)"]:::algo
            BaselineEngine["Baseline Interval Engine<br/>(portion / intervaltree Fallback)"]:::algo
        end

        subgraph LAYER_OUTBOUND ["5. Outbound Infrastructure Layer"]
            ScheduleRepo["Schedule Range Repository<br/>(SQLAlchemy Async Engine / asyncpg)"]:::outbound
            EventOutboxRepo["Outbox Event Repository<br/>(Transactional Event Persistence)"]:::outbound
        end

    end

    %% Flow Relationships
    Broker -->|1. Pop Task Payload<br/>Async Queue Pull| Consumer
    Consumer -->|2. Raw Payload<br/>In-Process Call| DTOValidator
    DTOValidator -->|3. Validated DTO<br/>Internal Injection| Anonymizer
    Anonymizer -->|4. Anonymized Context<br/>Domain DTO Transfer| Orchestrator

    Orchestrator -->|5. Build Service Graph<br/>Internal Call| DAGDecomposer
    DAGDecomposer -->|6. Verified Service DAG<br/>Domain Model| Orchestrator

    Orchestrator -->|7a. Execute Primary Search<br/>Time-budget 800ms| CPSATCore
    CPSATCore -.->|7b. Timeout / Exception Fallback<br/>Circuit Signal| ErrorHandler
    ErrorHandler -->|7c. Trigger Fallback Solver<br/>Emergency Delegate| BaselineEngine

    CPSATCore -->|8a. Primary Schedule Solution<br/>Pure Result Model| Orchestrator
    BaselineEngine -->|8b. Fallback Schedule Solution<br/>Pure Result Model| Orchestrator

    Orchestrator -->|9. Track Performance Metrics<br/>Async Event Push| Telemetry

    Orchestrator -->|10a. Persist Interval Ranges<br/>Repository Pattern| ScheduleRepo
    Orchestrator -->|10b. Save Outbox Event<br/>Transactional Pattern| EventOutboxRepo

    ScheduleRepo -->|11a. ACID Range Insertion<br/>PostgreSQL Wire / asyncpg| MainDB
    EventOutboxRepo -->|11b. Write Outbox Record<br/>SQL Transaction| MainDB
```

#### Деталізований опис внутрішніх компонентів та наскрізне трасування (Level 3):

1. **Шар вхідних адаптерів (`Inbound Adaptation Layer`):**
   * **`Taskiq Queue Consumer`:** Асинхронний консюмер подій, що отримує завдання з черги Redis (`RESP3`) за патерном *Event-Driven Consumer* та передає необроблений пейлоад у шар валідації без прямого доступу до інфраструктури БД.
2. **Шар безпеки та валідації (`Security & Validation Layer`):**
   * **`DTO Validation Guard (Pydantic v2)`:** Виконує сувору перевірку вхідних структур даних перед їх потраплянням у доменне ядро.
   * **`UUID PII Anonymizer Guard`:** Гарантує очищення вхідного контексту від персональних даних (PII), залишаючи лише знеособлені ідентифікатори `UUID` та часові параметри на виконання вимоги [SRS-NFR-SEC-02](./srs.md#3-non-functional-requirements-нефункціональні-вимоги).
3. **Шар доменного ядра та координації (`Domain Core & Orchestration Layer`):**
   * **`Scheduling Pipeline Orchestration Service`:** Центральний координатор процесу (патерн *Mediator / Orchestrator*). Керує життєвим циклом розрахунку, контролює жорсткий часовий бюджет 800 мс ([SRS-FUN-SOLV-03](./srs.md#23-алгоритмічне-ядро-оптимізації-solv)) та ізолює алгоритмічні рушії від побічних ефектів запису в БД.
   * **`DAG Process Decomposition Service`:** Перетворює композитну послугу у спрямований ациклічний граф (DAG) із розділенням на активні та пасивні фази та перевіркою на ациклічність згідно з вимогами [SRS-FUN-DAG-01](./srs.md#22-модуль-декомпозиції-та-граф-моделювання-dag) та [SRS-FUN-DAG-02](./srs.md#22-модуль-декомпозиції-та-граф-моделювання-dag).
   * **`Global Exception & Fallback Handler`:** Компонент підвищення відмовостійкості (*Resilience Manager*), що перехоплює таймаути або винятки основного солвера та делегує розрахунок резервному евристичному рушію.
   * **`Telemetry & Metrics Collector`:** Збирає операційні метрики затримок ($p_{95}$ Latency) для верифікації вимоги продуктивності [SRS-NFR-PERF-02](./srs.md#3-non-functional-requirements-нефункціональні-вимоги).
4. **Алгоритмічне ядро оптимізації (`Algorithmic Optimization Engine`):**
   * **`CP-SAT Solver Engine (Google OR-Tools)`:** Математичне ядро оптимізації (чиста доменна функція), що будує модель із змінними `NewOptionalIntervalVar`, обмеженнями `AddNoOverlap` ([SRS-FUN-SOLV-01](./srs.md#23-алгоритмічне-ядро-оптимізації-solv)) та виконує лексикографічну мінімізацію цільової функції ([SRS-FUN-SOLV-02](./srs.md#23-алгоритмічне-ядро-оптимізації-solv)).
   * **`Baseline Interval Engine (portion / intervaltree)`:** Резервний евристичний аналізатор інтервальних дерев для миттєвого пошуку допустимого розкладу у разі перевищення ліміту 800 мс ([SRS-FUN-SOLV-03](./srs.md#23-алгоритмічне-ядро-оптимізації-solv)).
5. **Шар вихідної інфраструктури (`Outbound Infrastructure Layer`):**
   * **`Schedule Range Repository`:** Реалізує патерн *Repository* поверх `SQLAlchemy 2.0 (asyncpg)`. Фіксує розраховані часові діапазони `tstz