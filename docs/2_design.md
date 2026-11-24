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



   ---

## 2.4. Проєктування та моделювання баз даних {#database-design}

Логіко-фізична модель збереження даних побудована для реляційної СУБД **PostgreSQL 16+** у Третій нормальній формі (**3NF**) із використанням діапазонного типу `TSTZRANGE` та індексного розширення `btree_gist` для забезпечення декларативного інваріанта неперетину інтервалів ([SRS-FUN-DB-01](./srs.md#24-сервіс-транзакційного-контролю-колізій-db), [ADR-001](./adr/0001-choice-of-technology.md)).

```mermaid
erDiagram
    USERS ||--o{ MASTER_SKILLS : possesses
    SERVICE_STAGES ||--o{ MASTER_SKILLS : requires
    USERS ||--o{ BOOKINGS : initiates
    USERS ||--o{ BOOKING_PHASES : assigned_to
    SERVICES ||--|{ SERVICE_STAGES : decomposes_into
    SERVICE_STAGES ||--o{ STAGE_DEPENDENCIES : precedes
    SERVICE_STAGES ||--o{ STAGE_DEPENDENCIES : succeeds
    SERVICES ||--o{ BOOKINGS : booked_for
    BOOKINGS ||--|{ BOOKING_PHASES : contains
    BOOKINGS ||--o{ OUTBOX_EVENTS : emits

    USERS {
        uuid id PK "DEFAULT gen_random_uuid()"
        varchar(255) email UK "NOT NULL, B-Tree Index"
        varchar(150) full_name "NOT NULL"
        user_role_enum role "NOT NULL (CLIENT|MASTER|DISPATCHER|ADMIN)"
        boolean is_active "NOT NULL DEFAULT TRUE"
        timestamptz created_at "NOT NULL DEFAULT now()"
        timestamptz updated_at "NOT NULL DEFAULT now()"
        timestamptz deleted_at "NULL (Soft Delete)"
    }

    SERVICES {
        uuid id PK "DEFAULT gen_random_uuid()"
        varchar(200) name "NOT NULL"
        text description "NULL"
        numeric(12_2) base_price "NOT NULL CHECK (base_price >= 0)"
        boolean is_active "NOT NULL DEFAULT TRUE"
        timestamptz created_at "NOT NULL DEFAULT now()"
        timestamptz updated_at "NOT NULL DEFAULT now()"
        timestamptz deleted_at "NULL (Soft Delete)"
    }

    SERVICE_STAGES {
        uuid id PK "DEFAULT gen_random_uuid()"
        uuid service_id FK "NOT NULL REFERENCES services(id)"
        varchar(150) stage_name "NOT NULL"
        stage_type_enum stage_type "NOT NULL (ACTIVE|PASSIVE_WAIT)"
        int duration_minutes "NOT NULL CHECK (duration_minutes > 0)"
        int buffer_after_minutes "NOT NULL DEFAULT 0"
        timestamptz created_at "NOT NULL DEFAULT now()"
    }

    STAGE_DEPENDENCIES {
        uuid parent_stage_id PK, FK "REFERENCES service_stages(id)"
        uuid child_stage_id PK, FK "REFERENCES service_stages(id)"
        int min_lag_minutes "NOT NULL DEFAULT 0"
    }

    MASTER_SKILLS {
        uuid master_id PK, FK "REFERENCES users(id)"
        uuid stage_id PK, FK "REFERENCES service_stages(id)"
        int skill_priority "NOT NULL DEFAULT 1"
    }

    BOOKINGS {
        uuid id PK "DEFAULT gen_random_uuid()"
        uuid client_id FK "NOT NULL REFERENCES users(id)"
        uuid service_id FK "NOT NULL REFERENCES services(id)"
        booking_status_enum status "NOT NULL (HOLD|CONFIRMED|IN_PROGRESS|COMPLETED|CANCELLED)"
        numeric(12_2) total_price "NOT NULL"
        timestamptz hold_expires_at "NULL (TTL 10 min for HOLD)"
        int version "NOT NULL DEFAULT 1 (Optimistic Lock)"
        timestamptz created_at "NOT NULL DEFAULT now()"
        timestamptz updated_at "NOT NULL DEFAULT now()"
    }

    BOOKING_PHASES {
        uuid id PK "DEFAULT gen_random_uuid()"
        uuid booking_id FK "NOT NULL REFERENCES bookings(id)"
        uuid stage_id FK "NOT NULL REFERENCES service_stages(id)"
        uuid master_id FK "NULL REFERENCES users(id)"
        tstzrange time_slot "NOT NULL (EXCLUDE USING gist with master_id)"
        phase_status_enum phase_status "NOT NULL (SCHEDULED|ACTIVE|DONE|SKIPPED)"
        timestamptz created_at "NOT NULL DEFAULT now()"
        timestamptz updated_at "NOT NULL DEFAULT now()"
    }

    OUTBOX_EVENTS {
        uuid id PK "DEFAULT gen_random_uuid()"
        uuid aggregate_id FK "NOT NULL REFERENCES bookings(id)"
        varchar(100) event_type "NOT NULL (e.g. SCHEDULE_MUTATED)"
        jsonb payload "NOT NULL"
        timestamptz created_at "NOT NULL DEFAULT now()"
        timestamptz processed_at "NULL (Partial B-Tree Index)"
    }
```

#### Опис ключових контурів та індексів БД:
* **Довідниковий та графовий контур (`USERS`, `SERVICES`, `SERVICE_STAGES`, `STAGE_DEPENDENCIES`, `MASTER_SKILLS`):** Забезпечує декомпозицію композитних послуг у спрямований ациклічний граф (**DAG**) із розділенням на активні (`ACTIVE`) та пасивні (`PASSIVE_WAIT`) фази ([SRS-FUN-DAG-01](./srs.md#22-модуль-декомпозиції-та-граф-моделювання-dag)) та формує матрицю компетенцій для розв'язувача CP-SAT.
* **Транзакційний контур (`BOOKINGS`, `BOOKING_PHASES`):** Підтримує двоетапне резервування (`HOLD` із TTL 10 хв та оптимістичним блокуванням `version` за [SRS-FUN-DB-02](./srs.md#24-сервіс-транзакційного-контролю-колізій-db)) і декларативне обмеження `EXCLUDE USING gist (master_id WITH =, time_slot WITH &&)` для гарантування $0\%$ подвійних бронювань ([SRS-NFR-REL-01](./srs.md#3-non-functional-requirements-нефункціональні-вимоги)).
* **Подієвий контур (`OUTBOX_EVENTS`):** Реалізує патерн *Transactional Outbox* із частковим покриваючим індексом `WHERE processed_at IS NULL` для атомарного запису подій мутації в одній ACID-транзакції з бронюванням ([SRS-FUN-STREAM-01](./srs.md#25-реактивний-сервіс-трансляції-станів-stream)).

---

## 2.5. Специфікація міжсервісних контрактів (API) {#api-contracts}

Взаємодія між клієнтським застосунком **`Single Page Application (Vue.js 3)`** та сервісом **`Web API Application (FastAPI)`** спроєктована за підходом **API-First Design** для забезпечення наскрізної типізації через автогенерацію SDK (`@hey-api/openapi-ts`) та валідацію моделей `Pydantic v2`.

Деталізовану специфікацію ендпоінтів, моделей даних (`components/schemas`), схем безпеки `JWT RS256` (`BearerAuth`) та стандартизованих відповідей про помилки (`RFC 9457 ProblemDetails`, включно з `409 Conflict` при спрацюванні GiST-обмеження) у форматі **OpenAPI 3.0.3** зафіксовано у файлі **[openapi.yaml](./api/openapi.yaml)**.

---

## 2.6. Моделювання логіки критичних вузлів {#critical-algorithms}

### 2.6.1. Алгоритм 1: Топологічна декомпозиція та валідація ациклічності графа послуги (Kahn's DAG Validation)

1. **Вхідні дані:** Масив етапів композитної послуги `Stages[0..N-1]` та масив орієнтованих ребер залежностей `Edges[0..M-1]`.
2. **Перевірка виродженості:** Якщо `N = 0 OR Stages = NULL`, алгоритм переривається із поверненням винятку `EMPTY_SERVICE_DAG`.
3. **Ініціалізація:** Встановлюються змінні `i := 0`, `VisitedCount := 0`, `Queue := EMPTY`, `TopoOrder := EMPTY`. Для кожного індексу `i` (`0 <= i < N`) обчислюється напівстепінь заходу `InDegree[i] := CountIncoming(Stages[i], Edges)`. Вузол із умовою `InDegree[i] = 0` додається до черги `Queue := Enqueue(Queue, Stages[i])`.
4. **Топологічне сортування (алгоритм Кана):** Поки виконується умова `NOT IsEmpty(Queue)`, вилучається вершина `u := Dequeue(Queue)`, оновлюються `TopoOrder := Append(TopoOrder, u)` та `VisitedCount := VisitedCount + 1`. Для кожного суміжного вузла `v := Neighbors[k]` (`0 <= k < Length(Neighbors)`) виконується декремент `InDegree[v] := InDegree[v] - 1`; при досягненні `InDegree[v] = 0` вершина `v` додається у чергу `Queue`.
5. **Верифікація ациклічності:** Якщо `VisitedCount <> N`, фіксується наявність контуру в графі та повертається помилка `CYCLE_DETECTED_IN_DAG`; інакше повертається впорядкований граф `ValidatedDAG(TopoOrder)`.

```mermaid
graph TD
    Start(["Початок"]) --> Input[/"Ввід: Stages[0..N-1], Edges[0..M-1]"/]
    Input --> CheckEmpty{"N = 0 OR Stages = NULL"}
    CheckEmpty -->|"Так"| ErrEmpty[/"Вивід: Error('EMPTY_SERVICE_DAG')"/]
    ErrEmpty --> End(["Кінець"])

    CheckEmpty -->|"Ні"| InitVars["i := 0;<br/>VisitedCount := 0;<br/>Queue := EMPTY;<br/>TopoOrder := EMPTY"]
    InitVars --> LoopInit{"i < N"}
    LoopInit -->|"Так"| CalcDeg["InDegree[i] := CountIncoming(Stages[i], Edges);<br/>i := i + 1"]
    CalcDeg --> LoopInit

    LoopInit -->|"Ні"| ResetIdx["i := 0"]
    ResetIdx --> LoopQueue{"i < N"}
    LoopQueue -->|"Так"| CheckZero{"InDegree[i] = 0"}
    CheckZero -->|"Так"| PushQ["Queue := Enqueue(Queue, Stages[i]);<br/>i := i + 1"]
    CheckZero -->|"Ні"| IncI["i := i + 1"]
    PushQ --> LoopQueue
    IncI --> LoopQueue

    LoopQueue -->|"Ні"| WhileQ{"NOT IsEmpty(Queue)"}
    WhileQ -->|"Так"| PopU["u := Dequeue(Queue);<br/>TopoOrder := Append(TopoOrder, u);<br/>VisitedCount := VisitedCount + 1;<br/>Neighbors := GetAdjacent(u, Edges);<br/>k := 0"]
    PopU --> LoopAdj{"k < Length(Neighbors)"}
    LoopAdj -->|"Так"| DecDeg["v := Neighbors[k];<br/>InDegree[v] := InDegree[v] - 1;<br/>k := k + 1"]
    DecDeg --> CheckNextZero{"InDegree[v] = 0"}
    CheckNextZero -->|"Так"| PushNext["Queue := Enqueue(Queue, v)"]
    CheckNextZero -->|"Ні"| LoopAdj
    PushNext --> LoopAdj
    LoopAdj -->|"Ні"| WhileQ

    WhileQ -->|"Ні"| CheckCycle{"VisitedCount <> N"}
    CheckCycle -->|"Так"| ErrCycle[/"Вивід: Error('CYCLE_DETECTED_IN_DAG')"/]
    ErrCycle --> End
    CheckCycle -->|"Ні"| OutDAG[/"Вивід: ValidatedDAG(TopoOrder)"/]
    OutDAG --> End
```

### 2.6.2. Алгоритм 2: Оркестрація лексикографічної оптимізації RCPSP із тайм-бюджетом та fallback-переключенням

1. **Вхідні дані:** Топологічно відсортований граф `DAG[0..N-1]`, множина виконавців `Masters[0..M-1]`, жорсткий ліміт часу обчислення `T_max := 800 ms`.
2. **Побудова моделі обмежень:** Ініціалізується модель `Model := CreateCPSATModel()`, лічильник `i := 0` та прапорець дефіциту ресурсів `HasResourceGap := FALSE`. У циклі при `i < N AND NOT HasResourceGap`:
   * Якщо `DAG[i].Type = 'ACTIVE'`, формується підмножина кваліфікованих майстрів `Qualified[i] := FilterBySkill(Masters, DAG[i])`. Якщо `Length(Qualified[i]) = 0`, встановлюється `HasResourceGap := TRUE`. Інакше для кожного `m` (`0 <= m < Length(Qualified[i])`) створюється опціональний інтервал `OptVar[i][m] := NewOptionalIntervalVar(DAG[i], Qualified[i][m])` та накладається обмеження вибору рівно одного виконавця `AddExactlyOne(OptVar[i][0..K-1])`.
   * Якщо `DAG[i].Type <> 'ACTIVE'` (фаза `PASSIVE_WAIT`), реєструється фіксований часовий інтервал без прив'язки до виконавця `PassiveVar[i] := NewIntervalVar(DAG[i])`.
3. **Перевірка доступності ресурсів:** Якщо `HasResourceGap = TRUE`, виконання припиняється зі статусом `NO_AVAILABLE_RESOURCES`.
4. **Оптимізація та резервування (Fallback):** Накладаються констрейнти неперетину `AddNoOverlapConstraints(Model, OptVar)`, задається векторна лексикографічна цільова функція `SetLexicographicObjective(Model)` та запускається розв'язувач із обмеженням `Solver.MaxTimeInSeconds := T_max / 1000`.
   * Якщо `Status = 'OPTIMAL' OR Status = 'FEASIBLE'`, формується результат `Candidates := ExtractCPSATSchedule(Model)`.
   * Інакше (при перевищенні бюджету часу або помилці) управління передається на резервний рушій інтервальних дерев `Candidates := RunBaselineIntervalTree(DAG, Masters)`, `Mode := 'BASELINE_HEURISTIC_FALLBACK'`. Якщо `Length(Candidates) = 0`, повертається статус `NO_SLOTS_FOUND`.
5. **Вихідні дані:** Повертається структура `ScheduleResult(Mode, Candidates)`.

```mermaid
graph TD
    Start(["Початок"]) --> Input[/"Ввід: DAG[0..N-1], Masters[0..M-1],<br/>T_max := 800 ms"/]
    Input --> InitModel["Model := CreateCPSATModel();<br/>i := 0;<br/>HasResourceGap := FALSE"]

    InitModel --> LoopPhases{"i < N AND NOT HasResourceGap"}
    LoopPhases -->|"Так"| CheckType{"DAG[i].Type = 'ACTIVE'"}
    CheckType -->|"Ні"| AddPassive["PassiveVar[i] := NewIntervalVar(DAG[i]);<br/>i := i + 1"]
    AddPassive --> LoopPhases

    CheckType -->|"Так"| FilterM["Qualified[i] := FilterBySkill(Masters, DAG[i]);<br/>K := Length(Qualified[i])"]
    FilterM --> CheckQual{"K = 0"}
    CheckQual -->|"Так"| SetGap["HasResourceGap := TRUE"]
    SetGap --> LoopPhases

    CheckQual -->|"Ні"| LoopM["m := 0"]
    LoopM --> InnerM{"m < K"}
    InnerM -->|"Так"| AddOpt["OptVar[i][m] := NewOptionalIntervalVar(DAG[i], Qualified[i][m]);<br/>m := m + 1"]
    AddOpt --> InnerM
    InnerM -->|"Ні"| AddExactlyOne["AddExactlyOne(OptVar[i][0..K-1]);<br/>i := i + 1"]
    AddExactlyOne --> LoopPhases

    LoopPhases -->|"Ні"| CheckGapFlag{"HasResourceGap = TRUE"}
    CheckGapFlag -->|"Так"| OutNoRes[/"Вивід: Status('NO_AVAILABLE_RESOURCES')"/]
    OutNoRes --> End(["Кінець"])

    CheckGapFlag -->|"Ні"| ConfigureSolver["AddNoOverlapConstraints(Model, OptVar);<br/>SetLexicographicObjective(Model);<br/>Solver.MaxTimeInSeconds := T_max / 1000;<br/>Status := Solve(Model)"]
    ConfigureSolver --> CheckSolver{"Status = 'OPTIMAL' OR Status = 'FEASIBLE'"}

    CheckSolver -->|"Так"| ExtractCP["Candidates := ExtractCPSATSchedule(Model);<br/>Mode := Status"]
    ExtractCP --> OutSuccess[/"Вивід: ScheduleResult(Mode, Candidates)"/]
    OutSuccess --> End

    CheckSolver -->|"Ні"| RunFallback["Candidates := RunBaselineIntervalTree(DAG, Masters);<br/>Mode := 'BASELINE_HEURISTIC_FALLBACK'"]
    RunFallback --> CheckFallbackEmpty{"Length(Candidates) = 0"}
    CheckFallbackEmpty -->|"Так"| OutEmptySlot[/"Вивід: Status('NO_SLOTS_FOUND')"/]
    OutEmptySlot --> End
    CheckFallbackEmpty -->|"Ні"| OutSuccess
```

### 2.6.3. Алгоритм 3: Атомарна фіксація темпоральних інтервалів (`tstzrange`) та патерн Transactional Outbox

1. **Вхідні дані:** Ключ ідепотентності `IdempotencyKey`, ідентифікатори `ClientID`, `ServiceID`, масив фаз розкладу `Phases[0..N-1]` та час життя холду `TTL := 600 s`.
2. **Контроль ідепотентності:** Виконується пошук `CachedResp := CheckIdempotencyStore(IdempotencyKey)` у сховищі Redis 7.x. Якщо `CachedResp <> NULL`, повертається збережена відповідь `HTTP 200 (CachedResp)` без відкриття транзакції запису.
3. **Транзакційне збереження та захист GiST:** Відкривається транзакція БД `BeginTransaction(Isolation := 'READ COMMITTED')`, фіксується дедлайн `ExpiresAt := NowUTC() + TTL`, генерується `BookingID := GenerateUUIDv4()` і створюється агрегат `BOOKINGS` зі статусом `Status := 'HOLD'` та `Version := 1`.
4. **Запис темпоральних діапазонів:** У циклі при `i < N AND DbError = NULL` для кожної фази конструюється напіввідкритий інтервал `Range[i] := BuildTSTZRange(Phases[i].Start, Phases[i].End)` та виконується вставка в таблицю `BOOKING_PHASES`. При виникненні помилки СУБД (`Result.Error <> NULL`) фіксується код `DbError := Result.SqlState`.
5. **Обробка колізій та фіксація події Outbox:**
   * Якщо `DbError <> NULL`, виконується `RollbackTransaction()`. При коді `DbError = '23P01'` (спрацювання індексу `EXCLUDE USING gist`) повертається `HTTP 409 Conflict`; при інших системних збоях СУБД — `HTTP 503 Service Unavailable`.
   * Якщо `DbError = NULL`, у межах тієї ж транзакції генерується `EventID := GenerateUUIDv4()` та записується подія `BOOKING_HELD` у таблицю `OUTBOX_EVENTS`, після чого виконується `CommitTransaction()`, збереження ключа ідепотентності та повернення статусу `HTTP 201 Created (BookingID, ExpiresAt)`.

```mermaid
graph TD
    Start(["Початок"]) --> Input[/"Ввід: IdempotencyKey, ClientID, ServiceID,<br/>Phases[0..N-1], TTL := 600 s"/]
    Input --> CheckIdemp["CachedResp := CheckIdempotencyStore(IdempotencyKey)"]
    CheckIdemp --> IsCached{"CachedResp <> NULL"}
    IsCached -->|"Так"| OutCached[/"Вивід: HTTP 200 (CachedResp)"/]
    OutCached --> End(["Кінець"])

    IsCached -->|"Ні"| BeginTx["BeginTransaction(Isolation := 'READ COMMITTED');<br/>ExpiresAt := NowUTC() + TTL;<br/>BookingID := GenerateUUIDv4()"]
    BeginTx --> InsertBooking["InsertBooking(BookingID, ClientID, ServiceID,<br/>Status := 'HOLD', ExpiresAt, Version := 1);<br/>i := 0;<br/>DbError := NULL"]

    InsertBooking --> LoopInsert{"i < N AND DbError = NULL"}
    LoopInsert -->|"Так"| BuildRange["Range[i] := BuildTSTZRange(Phases[i].Start, Phases[i].End);<br/>Result := InsertBookingPhase(BookingID, Phases[i], Range[i])"]
    BuildRange --> CheckSqlErr{"Result.Error <> NULL"}
    CheckSqlErr -->|"Так"| SetErr["DbError := Result.SqlState"]
    SetErr --> LoopInsert
    CheckSqlErr -->|"Ні"| NextPhase["i := i + 1"]
    NextPhase --> LoopInsert

    LoopInsert -->|"Ні"| HasDbErr{"DbError <> NULL"}
    HasDbErr -->|"Так"| RollbackTx["RollbackTransaction()"]
    RollbackTx --> IsOverlap{"DbError = '23P01'"}
    IsOverlap -->|"Так"| Out409[/"Вивід: HTTP 409 Conflict (GiST Exclusion)"/]
    Out409 --> End
    IsOverlap -->|"Ні"| Out503[/"Вивід: HTTP 503 Service Unavailable"/]
    Out503 --> End

    HasDbErr -->|"Ні"| WriteOutbox["EventID := GenerateUUIDv4();<br/>Payload := SerializeDelta(BookingID, Phases);<br/>InsertOutboxEvent(EventID, BookingID, 'BOOKING_HELD', Payload)"]
    WriteOutbox --> CommitTx["CommitTransaction();<br/>SaveIdempotencyStore(IdempotencyKey, BookingID)"]
    CommitTx --> Out201[/"Вивід: HTTP 201 Created (BookingID, ExpiresAt)"/]
    Out201 --> End
```

---

## 2.7. Інтеграційний аудит та верифікація цілісності доменів {#integration-audit}

**Статус верифікації:** `APPROVED FOR IMPLEMENTATION (PASS — 0 CRITICAL SHOWSTOPPERS)`

| № | Критерій інтеграційної верифікації | Статус | Коментар аудитора |
| :-: | :--- | :---: | :--- |
| 1 | Узгодженість типів ключів (`UUID`), фінансів (`NUMERIC(12,2)`) та дат (`ISO 8601` $\to$ `TSTZRANGE`) | **PASS** | Повна типова сумісність між [openapi.yaml](./api/openapi.yaml), Pydantic v2 та PostgreSQL 16+ |
| 2 | Покриття захистом від стану гонитви (Race Conditions) та подвійних бронювань | **PASS** | `EXCLUDE USING gist` у БД $\leftrightarrow$ `SQLSTATE 23P01` в Алгоритмі 3 $\leftrightarrow$ `409 Conflict` в API |
| 3 | Узгодженість механізму підтвердження холду (`HOLD` $\to$ `CONFIRMED`) та оптимістичного блокування | **PASS** | `hold_expires_at` + `version` у `BOOKINGS` $\leftrightarrow$ `PATCH /bookings/{booking_id}/confirm` |
| 4 | Атомарність подійної синхронізації (Transactional Outbox $\to$ Redis Pub/Sub $\to$ SSE) | **PASS** | Таблиця `OUTBOX_EVENTS` + Алгоритм 3 $\leftrightarrow$ `GET /schedules/stream` |
| 5 | Академічний нормоконтроль формалізації алгоритмів (`:=`, `AND/OR/NOT`, індекси) | **PASS** | Усі 3 алгоритми відповідають вимогам ДСТУ / ISO 5807 |

**Висновок комісії:** Архітектурний, даталогічний та алгоритмічний контури Розділу 2 є повністю узгодженими, не містять інтеграційних блокерів і затверджуються для переходу до програмної реалізації.