# Розділ 2. Проєктування архітектури системи

**Проєкт:** Підсистема динамічного планування та оптимізації багатофазних процесів у б'юті-індустрії  
**Методологія моделювання:** C4 Model (Simon Brown) / Clean Architecture / Domain-Driven Design (DDD)  
**Контрактна база вимог:** [Специфікація вимог (SRS — ISO/IEC/IEEE 29148)](./srs.md)  
**Журнал рішень:** [ADR-001: Контроль конкурентного доступу та неперетин інтервалів](./adr/0001-choice-of-technology.md)

---

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model)

### 2.1.1. Обґрунтування макроархітектурного стилю та патернів проєктування

Для реалізації підсистеми динамічного планування та оптимізації багатофазних процесів обрано парадигму **розподіленого модульного моноліту з виділеними обчислювальними та подієвими воркерами (Shared-Database Multi-Process Architecture)**, спроєктовану за принципами **Clean Architecture** та **Domain-Driven Design (DDD)**. Свідома відмова від класичного мікросервісного підходу з роздільними базами даних (*Database-per-Service*) на користь єдиного транзакційного ядра **PostgreSQL 16+** зумовлена необхідністю виконання суворих ACID-інваріантів неперетину часових інтервалів між усіма фазами послуг без накладних витрат і складності розподілених транзакцій (2PC / Saga).

Декомпозиція системи на контейнери безпосередньо забезпечує виконання кількісних нефункціональних вимог специфікації SRS:

1. **Ізоляція CPU-bound обчислень для захисту циклу подій (`SRS-NFR-PERF-01`, `SRS-NFR-PERF-02`):** Комбінаторний пошук оптимального розкладу задачі RCPSP засобами `Google OR-Tools CP-SAT` є ресурсомісткою CPU-bound операцією з верхньою межею часу виконання $T_{max} = 800\text{ мс}$. Запуск таких обчислень безпосередньо всередині процесу `Web API Application (FastAPI)` заблокував би однопотоковий асинхронний цикл подій (`asyncio` Event Loop) сервера `uvicorn`, деградувавши обслуговування паралельних HTTP-запитів та активних SSE-потоків. Винесення обчислень у пули ізольованих контейнерів `CP-SAT Solver Worker` та `Baseline Heuristic Worker` через черги брокера `Redis 7.x / Taskiq` дозволяє I/O-контуру API стабільно обслуговувати навантаження $500\text{ RPS}$ із затримкою $p_{95} \le 100\text{ мс}$, тоді як обчислювальний контур масштабується горизонтально збільшенням кількості реплік воркерів.
2. **Детермінований контроль конкурентного доступу та захист від пікових навантажень (`SRS-FUN-DB-01`, `SRS-NFR-REL-01`):** Оскільки розрахунок варіантів розкладу (`Taskiq`) та фіксація бронювання (`Web API`) відбуваються у різних конкурентних потоках, два клієнти можуть одночасно спробувати зарезервувати однаковий часовий слот (*Race Condition*). Захист від подвійних бронювань винесено на рівень рушія баз даних PostgreSQL 16+ через діапазонний тип `TSTZRANGE` та індексне обмеження `EXCLUDE USING gist` (розширення `btree_gist`, рішення [ADR-001](./adr/0001-choice-of-technology.md)). При одночасному надходженні конкуруючих запитів СУБД серіалізує перевірку перетинів на рівні GiST-індексу за логарифмічний час $O(\log K)$, фіксуючи першу транзакцію та миттєво відхиляючи конфліктуючі запити з кодом `SQLSTATE 23P01` (`HTTP 409 Conflict`). Для захисту пулу з'єднань `asyncpg` від дублювання мережевих запитів перед відкриттям транзакції використовується кеш ключів ідепотентності `Idempotency-Key` у Redis 7.x (TTL 24 год).
3. **Надійність реактивної доставки подій та узгодженість протоколів (`SRS-FUN-STREAM-01`, `SRS-NFR-REL-02`):** Внутрішній брокер `Redis Pub/Sub` функціонує за семантикою *Fire-and-Forget* і не зберігає повідомлення при тимчасовій втраті з'єднання. Для усунення ризику втрати подій реалізовано патерн **Transactional Outbox**: подія мутації розкладу атомарно записується в таблицю `OUTBOX_EVENTS` у тій самій транзакції БД, що й бронювання, звідки вичитується фоновим процесом `Outbox Publisher Worker` (ізольованим за допомогою `SELECT ... FOR UPDATE SKIP LOCKED`) та транслюється в `Redis Pub/Sub` і зовнішній `Notification Gateway`. Доставка змін у клієнтський `Single Page Application (Vue.js 3)` стандартизована за протоколом **W3C Server-Sent Events (SSE)** поверх HTTP/2, де кожна подія містить монотонний ідентифікатор `id` для автоматичної ресинхронізації пропущених станів через заголовок `Last-Event-ID`.
На мікрорівні всередині сервісів застосовано патерни **Unit of Work (UoW)** спільно з **Repository Pattern** (атомарний запис агрегатів та подій Outbox під керуванням спільного `AsyncSession` та ізоляція специфіки `tstzrange` / `SQLSTATE 23P01`), **Distributed Fallback Orchestrator** (перемикання з `taskiq:solver` на `taskiq:baseline` при перевищенні $800\text{ мс}$) та **Dependency Injection (`Depends`)**.

---

### 2.1.2. C4 Level 1: System Context Diagram (Системний контекст) {#diagram-level1}

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

#### Технічний опис системного контексту та трасування вимог (Level 1)

- **Зовнішні актори (Person):**
  - **Клієнт (Споживач):** Взаємодіє з підсистемою через шифрований канал `HTTPS / WSS` для вибору композитних послуг та фіксації часових слотів відповідно до контексту системи ([System Context](https://www.google.com/search?q=./srs.md%231-system-context-%D0%BA%D0%BE%D0%BD%D1%82%D0%B5%D0%BA%D1%81%D1%82-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D0%B8)).
  - **Майстер (Виконавець) та Диспетчер / Адміністратор:** Отримують персоніфіковані розклади та керують сіткою ресурсів із розмежуванням прав доступу за ролями (RBAC), що трасується до вимоги [SRS-FUN-AUTH-02](https://www.google.com/search?q=./srs.md%2321-%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D1%8C-%D0%B0%D0%B2%D1%82%D0%B5%D0%BD%D1%82%D0%B8%D1%84%D1%96%D0%BA%D0%B0%D1%86%D1%96%D1%97-%D1%82%D0%B0-%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D1%8E-%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF%D1%83-auth).
  - **Зовнішні програмні системи (`System_Ext`):**
  - **`Identity Provider`:** Забезпечує федеративну автентифікацію за протоколами OAuth 2.0 / OIDC та верифікацію криптографічних підписів токенів `JWT RS256` ([SRS-FUN-AUTH-01](https://www.google.com/search?q=./srs.md%2321-%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D1%8C-%D0%B0%D0%B2%D1%82%D0%B5%D0%BD%D1%82%D0%B8%D1%84%D1%96%D0%BA%D0%B0%D1%86%D1%96%D1%97-%D1%82%D0%B0-%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D1%8E-%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF%D1%83-auth), [SRS-NFR-SEC-01](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8)).
  - **`Notification Gateway`:** Виконує асинхронну доставку зовнішніх сповіщень про зміну статусу візиту через протокол `AMQP / HTTPS`.

---

### 2.1.3. C4 Level 2: Container Diagram (Діаграма контейнерів) {#diagram-level2}

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

#### Технічний опис контейнерів, мережевих протоколів та трасування (Level 2)

1. **`Reverse Proxy / Gateway (Nginx / Envoy)`:** Вхідний шлюз периметра. Забезпечує термінацію `TLS 1.3` ([SRS-NFR-SEC-01](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8)) та маршрутизацію трафіку до SPA (`HTTP / 1.1`) і бекенд-API (`HTTP / 2`).
2. **`Single Page Application (Vue.js 3)`:** Клієнтський інтерфейс диспетчеризації. Виконує запити мутацій через `REST / JSON (HTTPS)` та утримує однонапрямний потік оновлень розкладу за протоколом `HTTP EventStream (W3C SSE)` ([SRS-FUN-STREAM-02](https://www.google.com/search?q=./srs.md%2325-%D1%80%D0%B5%D0%B0%D0%BA%D1%82%D0%B8%D0%B2%D0%BD%D0%B8%D0%B9-%D1%81%D0%B5%D1%80%D0%B2%D1%96%D1%81-%D1%82%D1%80%D0%B0%D0%BD%D1%81%D0%BB%D1%8F%D1%86%D1%96%D1%97-%D1%81%D1%82%D0%B0%D0%BD%D1%96%D0%B2-stream)).
3. **`Web API Application (FastAPI)`:** Асинхронний сервіс бізнес-логіки та валідації Pydantic v2, розрахований на затримку $p_{95} \le 100$ мс при $500$ RPS ([SRS-NFR-PERF-01](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8)). Взаємодіє з СУБД через `PostgreSQL Wire Protocol / TCP (asyncpg)`.
4. **`CP-SAT Solver Worker` та `Baseline Heuristic Worker`:** Ізольовані у dedicated Docker-контейнерах ([SRS-NFR-DEVOPS-01](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8)) фонові воркери. Реалізують лексикографічну оптимізацію RCPSP (`Google OR-Tools`) з жорстким time-budgeting 800 мс та автоматичним переключенням на інтервальний евристичний аналізатор (`portion` / `intervaltree`) згідно з [SRS-FUN-SOLV-01 — SRS-FUN-SOLV-03](https://www.google.com/search?q=./srs.md%2323-%D0%B0%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC%D1%96%D1%87%D0%BD%D0%B5-%D1%8F%D0%B4%D1%80%D0%BE-%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D1%96%D0%B7%D0%B0%D1%86%D1%96%D1%97-solv) та [SRS-NFR-PERF-02](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8).
5. **`Outbox Publisher Worker (Taskiq)`:** Вичитує невідправлені доменні події з таблиці Outbox у PostgreSQL та публікує їх у канали `Redis Pub/Sub` за протоколом `RESP3 Protocol / TCP`, ліквідуючи аномалії Dual-Write та забезпечуючи відновлення після збоїв брокера ([SRS-FUN-STREAM-01](https://www.google.com/search?q=./srs.md%2325-%D1%80%D0%B5%D0%B0%D0%BA%D1%82%D0%B8%D0%B2%D0%BD%D0%B8%D0%B9-%D1%81%D0%B5%D1%80%D0%B2%D1%96%D1%81-%D1%82%D1%80%D0%B0%D0%BD%D1%81%D0%BB%D1%8F%D1%86%D1%96%D1%97-%D1%81%D1%82%D0%B0%D0%BD%D1%96%D0%B2-stream), [SRS-NFR-REL-02](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8)).
6. **`Main Relational Database (PostgreSQL 16+)`:** Єдине джерело істини (Single Source of Truth). Використовує темпоральний тип `tstzrange` та індексне обмеження `EXCLUDE USING gist` (розширення `btree_gist`) для гарантування $0\%$ подвійних бронювань ([SRS-FUN-DB-01](https://www.google.com/search?q=./srs.md%2324-%D1%81%D0%B5%D1%80%D0%B2%D1%96%D1%81-%D1%82%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D1%96%D0%B9%D0%BD%D0%BE%D0%B3%D0%BE-%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D1%8E-%D0%BA%D0%BE%D0%BB%D1%96%D0%B7%D1%96%D0%B9-db), [SRS-NFR-REL-01](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8)). Архітектурне обґрунтування вибору зафіксовано в [ADR-001](https://www.google.com/search?q=./adr/0001-choice-of-technology.md).
7. **`In-Memory Broker & Cache (Redis 7.x)`:** Обслуговує асинхронні черги задач `Taskiq` (включно з механізмом вивільнення тимчасових холдів слотів за [SRS-FUN-DB-02](https://www.google.com/search?q=./srs.md%2324-%D1%81%D0%B5%D1%80%D0%B2%D1%96%D1%81-%D1%82%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D1%96%D0%B9%D0%BD%D0%BE%D0%B3%D0%BE-%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D1%8E-%D0%BA%D0%BE%D0%BB%D1%96%D0%B7%D1%96%D0%B9-db)) та шину подій `Pub/Sub`.

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

### Деталізований опис внутрішніх компонентів та наскрізне трасування (Level 3)

1. **Шар вхідних адаптерів (`Inbound Adaptation Layer`):**
   - **`Taskiq Queue Consumer`:** Асинхронний консюмер подій, що отримує завдання з черги Redis (`RESP3`) за патерном *Event-Driven Consumer* та передає необроблений пейлоад у шар валідації без прямого доступу до інфраструктури БД.
2. **Шар безпеки та валідації (`Security & Validation Layer`):**
   - **`DTO Validation Guard (Pydantic v2)`:** Виконує сувору перевірку вхідних структур даних перед їх потраплянням у доменне ядро.
   - **`UUID PII Anonymizer Guard`:** Гарантує очищення вхідного контексту від персональних даних (PII), залишаючи лише знеособлені ідентифікатори `UUID` та часові параметри на виконання вимоги [SRS-NFR-SEC-02](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8).
3. **Шар доменного ядра та координації (`Domain Core & Orchestration Layer`):**
   - **`Scheduling Pipeline Orchestration Service`:** Центральний координатор процесу (патерн *Mediator / Orchestrator*). Керує життєвим циклом розрахунку, контролює жорсткий часовий бюджет 800 мс ([SRS-FUN-SOLV-03](https://www.google.com/search?q=./srs.md%2323-%D0%B0%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC%D1%96%D1%87%D0%BD%D0%B5-%D1%8F%D0%B4%D1%80%D0%BE-%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D1%96%D0%B7%D0%B0%D1%86%D1%96%D1%97-solv)) та ізолює алгоритмічні рушії від побічних ефектів запису в БД.
   - **`DAG Process Decomposition Service`:** Перетворює композитну послугу у спрямований ациклічний граф (DAG) із розділенням на активні та пасивні фази та перевіркою на ациклічність згідно з вимогами [SRS-FUN-DAG-01](https://www.google.com/search?q=./srs.md%2322-%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D1%8C-%D0%B4%D0%B5%D0%BA%D0%BE%D0%BC%D0%BF%D0%BE%D0%B7%D0%B8%D1%86%D1%96%D1%97-%D1%82%D0%B0-%D0%B3%D1%80%D0%B0%D1%84-%D0%BC%D0%BE%D0%B4%D0%B5%D0%BB%D1%8E%D0%B2%D0%B0%D0%BD%D0%BD%D1%8F-dag) та [SRS-FUN-DAG-02](https://www.google.com/search?q=./srs.md%2322-%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D1%8C-%D0%B4%D0%B5%D0%BA%D0%BE%D0%BC%D0%BF%D0%BE%D0%B7%D0%B8%D1%86%D1%96%D1%97-%D1%82%D0%B0-%D0%B3%D1%80%D0%B0%D1%84-%D0%BC%D0%BE%D0%B4%D0%B5%D0%BB%D1%8E%D0%B2%D0%B0%D0%BD%D0%BD%D1%8F-dag).
   - **`Global Exception & Fallback Handler`:** Компонент підвищення відмовостійкості (*Resilience Manager*), що перехоплює таймаути або винятки основного солвера та делегує розрахунок резервному евристичному рушію.
   - **`Telemetry & Metrics Collector`:** Збирає операційні метрики затримок ($p_{95}$ Latency) для верифікації вимоги продуктивності [SRS-NFR-PERF-02](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8).
4. **Алгоритмічне ядро оптимізації (`Algorithmic Optimization Engine`):**
   - **`CP-SAT Solver Engine (Google OR-Tools)`:** Математичне ядро оптимізації (чиста доменна функція), що будує модель із змінними `NewOptionalIntervalVar`, обмеженнями `AddNoOverlap` ([SRS-FUN-SOLV-01](https://www.google.com/search?q=./srs.md%2323-%D0%B0%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC%D1%96%D1%87%D0%BD%D0%B5-%D1%8F%D0%B4%D1%80%D0%BE-%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D1%96%D0%B7%D0%B0%D1%86%D1%96%D1%97-solv)) та виконує лексикографічну мінімізацію цільової функції ([SRS-FUN-SOLV-02](https://www.google.com/search?q=./srs.md%2323-%D0%B0%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC%D1%96%D1%87%D0%BD%D0%B5-%D1%8F%D0%B4%D1%80%D0%BE-%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D1%96%D0%B7%D0%B0%D1%86%D1%96%D1%97-solv)).
   - **`Baseline Interval Engine (portion / intervaltree)`:** Резервний евристичний аналізатор інтервальних дерев для миттєвого пошуку допустимого розкладу у разі перевищення ліміту 800 мс ([SRS-FUN-SOLV-03](https://www.google.com/search?q=./srs.md%2323-%D0%B0%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC%D1%96%D1%87%D0%BD%D0%B5-%D1%8F%D0%B4%D1%80%D0%BE-%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D1%96%D0%B7%D0%B0%D1%86%D1%96%D1%97-solv)).
5. **Шар вихідної інфраструктури (`Outbound Infrastructure Layer`):**
   - **`Schedule Range Repository`:** Реалізує патерн *Repository* поверх `SQLAlchemy 2.0 (asyncpg)`. Перетворює часові слоти у напіввідкриті діапазони `tstzrange` (`[start_time, end_time)`) та перехоплює винятки індексу `EXCLUDE USING gist` (`SQLSTATE 23P01`), трансформуючи їх у доменний виняток `ScheduleConflictException`.
   - **`Outbox Event Repository`:** Забезпечує реалізацію патерну *Transactional Outbox* (Алгоритм 3), атомарно зберігаючи події мутації розкладу (`BOOKING_HELD`, `BOOKING_CONFIRMED`, `SCHEDULE_MUTATED`) у таблицю `OUTBOX_EVENTS` у межах спільної сесії `AsyncSession`.

---

## 2.3. Контур безпеки програмного рішення (Application Security) {#appsec-contour}

Захист серверного ядра спроєктовано відповідно до вимог специфікації (`SRS-NFR-SEC-01..03`) та рекомендацій **OWASP API Security Top 10 (2023)** і **OWASP ASVS v4.0** за принципом глибокого ешелонованого захисту (*Defense-in-Depth*).

### 2.3.1. Криптографічна автентифікація та управління сесіями (`OWASP API2:2023`)

- **Асиметрична верифікація токенів (`JWT RS256`):** Автентифікація делегована зовнішньому `Identity Provider` (OAuth 2.0 / OIDC). Middleware серверного ядра верифікує цифровий підпис `RS256` за допомогою кешованих публічних ключів (JWKS), суворо фіксуючи дозволений алгоритм (`algorithms=["RS256"]` для захисту від атак підміни алгоритму `alg: none` або `HS256`), та перевіряє клейми `iss`, `aud` і термін дії `exp` (TTL Access-токена — $15\text{ хвилин}$).
- **Захист зберігання токенів на клієнті:** У клієнтському `Vue.js 3 SPA` Access-токен утримується виключно в оперативній пам'яті (`Pinia` store) без запису в `localStorage`/`sessionStorage` (захист від XSS), а Refresh-токен передається через `HttpOnly; Secure; SameSite=Strict` cookie. Для внутрішніх криптографічних операцій (хешування сервісних ключів та підпис вебхуків) використовується алгоритм **`Argon2id`** (RFC 9106, параметри: $m=65536\text{ KiB}, t=3, p=4$) та **`HMAC-SHA256`**.

### 2.3.2. Авторизаційний контроль об'єктів та блокування ін'єкцій (`OWASP API1 / API3`)

- **Захист від BOLA / IDOR (`RBAC + Resource Ownership Guard`):** Окрім перевірки ролі користувача (`CLIENT`, `MASTER`, `DISPATCHER`, `ADMIN`) на рівні маршруту, сервіс управління життєвим циклом бронювання виконує обов'язкову перевірку належності об'єкта: при створенні або підтвердженні бронювання ідентифікатор суб'єкта з токена (`jwt.sub`) звіряється зі значенням `client_id` у запиті або записі БД. Будь-яка спроба звернення до чужого агрегату блокується з кодом `HTTP 403 Forbidden` та фіксується в журналі аудиту.
- **Блокування Mass Assignment (`OWASP API3:2023 — BOPLA`):** Усі вхідні моделі `Pydantic v2` конфігуруються з параметром `model_config = ConfigDict(extra="forbid")`, що автоматично відхиляє будь-які непередбачені поля в JSON-запиті (`HTTP 422 Unprocessable Entity`). Мапінг даних із DTO у ORM-сутності `SQLAlchemy 2.0` виконується виключно через явне перерахування дозволених атрибутів, унеможливлюючи підміну системних полів (`status`, `version`, `total_price`, `role`).
- **Захист від SQL-ін'єкцій та логічних атак на граф:** Усі запити в `Schedule Range Repository` та `Outbox Event Repository` виконуються виключно через параметризовані вирази `SQLAlchemy Core / ORM` (`asyncpg` binary protocol) без конкатенації рядків. Топологічна структура графа послуги попередньо проходить перевірку на відсутність циклів за алгоритмом Кана (`CYCLE_DETECTED_IN_DAG`), блокуючи спроби викликати нескінченну рекурсію під час планування.

### 2.3.3. Захист від виснаження ресурсів, Audit Logging та DevSecOps-контроль (`OWASP API4`)

- **Багаторівневий Rate Limiting та ідепотентність:** На рівні шлюзу діє обмежувач частоти запитів `Token Bucket`, а на рівні FastAPI для CPU-bound ендпоінту `POST /api/v1/schedules/optimize` застосовується лімітер на базі `Redis Sliding Window` (не більше $10\text{ запитів/хв}$ на одного користувача) із жорстким обмеженням `max_candidates <= 10`. Мутуючі запити `POST /api/v1/bookings` захищені обов'язковим заголовком `Idempotency-Key` (UUID v4) для запобігання повторному виконанню транзакцій.
- **Незмінний журнал аудиту безпеки (`Security Audit Logging`):** Усі критичні безпекові події (невдала валідація JWT `401`, спроби несанкціонованого доступу до чужих об'єктів `403`, колізії блокувань `409`, перевищення квот `429`) записуються у структурований потік логів (`JSON`) із фіксацією `timestamp`, `trace_id`, `ip_hash`, `endpoint` та знеособленого через `UUID PII Anonymizer Guard` ідентифікатора суб'єкта.
- **Управління секретами та автоматизація перевірок (DevSecOps):** Конфіденційні параметри конфігурації (рядки підключення PostgreSQL/Redis, публічні ключі JWKS) передаються виключно через змінні середовища Docker Secrets / Vault (файли `.env` додані до `.gitignore`). У конвеєрі CI/CD інтегровано статичний аналіз безпеки коду Python (**`Bandit`** / **`Ruff S-rules`**) та сканування вразливостей сторонніх пакетів (**`pip-audit`** / **`Trivy`**).

---

## 2.4. Проєктування та моделювання баз даних {#database-design}

Логіко-фізична модель збереження даних побудована для реляційної СУБД **PostgreSQL 16+** у Третій нормальній формі (**3NF**) із використанням діапазонного типу `TSTZRANGE` та індексного розширення `btree_gist` для забезпечення декларативного інваріанта неперетину інтервалів ([SRS-FUN-DB-01](https://www.google.com/search?q=./srs.md%2324-%D1%81%D0%B5%D1%80%D0%B2%D1%96%D1%81-%D1%82%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D1%96%D0%B9%D0%BD%D0%BE%D0%B3%D0%BE-%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D1%8E-%D0%BA%D0%BE%D0%BB%D1%96%D0%B7%D1%96%D0%B9-db), [ADR-001](https://www.google.com/search?q=./adr/0001-choice-of-technology.md)).

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

### Опис ключових контурів та індексів БД

- **Довідниковий та графовий контур (`USERS`, `SERVICES`, `SERVICE_STAGES`, `STAGE_DEPENDENCIES`, `MASTER_SKILLS`):** Забезпечує декомпозицію композитних послуг у спрямований ациклічний граф (**DAG**) із розділенням на активні (`ACTIVE`) та пасивні (`PASSIVE_WAIT`) фази ([SRS-FUN-DAG-01](https://www.google.com/search?q=./srs.md%2322-%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D1%8C-%D0%B4%D0%B5%D0%BA%D0%BE%D0%BC%D0%BF%D0%BE%D0%B7%D0%B8%D1%86%D1%96%D1%97-%D1%82%D0%B0-%D0%B3%D1%80%D0%B0%D1%84-%D0%BC%D0%BE%D0%B4%D0%B5%D0%BB%D1%8E%D0%B2%D0%B0%D0%BD%D0%BD%D1%8F-dag)) та формує матрицю компетенцій для розв'язувача CP-SAT.
- **Транзакційний контур (`BOOKINGS`, `BOOKING_PHASES`):** Підтримує двоетапне резервування (`HOLD` із TTL 10 хв та оптимістичним блокуванням `version` за [SRS-FUN-DB-02](https://www.google.com/search?q=./srs.md%2324-%D1%81%D0%B5%D1%80%D0%B2%D1%96%D1%81-%D1%82%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D1%96%D0%B9%D0%BD%D0%BE%D0%B3%D0%BE-%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D1%8E-%D0%BA%D0%BE%D0%BB%D1%96%D0%B7%D1%96%D0%B9-db)) і декларативне обмеження `EXCLUDE USING gist (master_id WITH =, time_slot WITH &&)` для гарантування $0\%$ подвійних бронювань ([SRS-NFR-REL-01](https://www.google.com/search?q=./srs.md%233-non-functional-requirements-%D0%BD%D0%B5%D1%84%D1%83%D0%BD%D0%BA%D1%86%D1%96%D0%BE%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%96-%D0%B2%D0%B8%D0%BC%D0%BE%D0%B3%D0%B8)).
- **Подієвий контур (`OUTBOX_EVENTS`):** Реалізує патерн *Transactional Outbox* із частковим покриваючим індексом `WHERE processed_at IS NULL` для атомарного запису подій мутації в одній ACID-транзакції з бронюванням ([SRS-FUN-STREAM-01](https://www.google.com/search?q=./srs.md%2325-%D1%80%D0%B5%D0%B0%D0%BA%D1%82%D0%B8%D0%B2%D0%BD%D0%B8%D0%B9-%D1%81%D0%B5%D1%80%D0%B2%D1%96%D1%81-%D1%82%D1%80%D0%B0%D0%BD%D1%81%D0%BB%D1%8F%D1%86%D1%96%D1%97-%D1%81%D1%82%D0%B0%D0%BD%D1%96%D0%B2-stream)).

---

## 2.5. Специфікація міжсервісних контрактів (API) та топологія маршрутів {#api-contracts}

Взаємодія між клієнтським застосунком **`Single Page Application (Vue.js 3)`** та сервісом **`Web API Application (FastAPI)`** спроєктована за підходом **API-First Design** для забезпечення наскрізної типізації через автогенерацію SDK (`@hey-api/openapi-ts`) та валідацію моделей `Pydantic v2`.

Деталізовану специфікацію ендпоінтів, моделей даних (`components/schemas`), схем безпеки `JWT RS256` (`BearerAuth`) та стандартизованих відповідей про помилки (`RFC 9457 ProblemDetails`, включно з `409 Conflict` при спрацюванні GiST-обмеження) у форматі **OpenAPI 3.0.3** зафіксовано у файлі **[openapi.yaml](https://www.google.com/search?q=./api/openapi.yaml)**.

### 2.5.1. Карта інтерфейсних маршрутів системи

Усі ендпоінти функціонують під базовим префіксом `/api/v1` за протоколом **HTTPS (TLS 1.3)**, вимагають заголовка автентифікації `Authorization: Bearer <JWT_RS256>` (схема безпеки `BearerAuth`) і повертають помилки у стандартизованому форматі **RFC 9457 (`application/problem+json`)**.

| ID маршруту | HTTP-метод | Топологічний шлях (URI) | Вхідні параметри / Payload | Призначення та Контейнер-обробник (C4 Model) |
| --- | --- | --- | --- | --- |
| **API-REQ-01** | `POST` | `/api/v1/schedules/optimize` | **Headers:** `Authorization: Bearer <JWT>`<br><br>**Body (`ScheduleOptimizationRequest`):** `service_id: uuid`, `target_date: date`, `preferred_start_time: datetime\|null`, `preferred_master_id: uuid\|null`, `max_candidates: int [1..10]` | Ініціює топологічну декомпозицію DAG послуги та запуск лексикографічної оптимізації RCPSP із тайм-бюджетом $800\text{ мс}$.<br><br>**Контейнери:** `Reverse Proxy` $\to$ `Web API (FastAPI)` $\to$ `Redis (Taskiq)` $\to$ `CP-SAT Solver Worker` / `Baseline Heuristic Worker`. |
| **API-REQ-02** | `POST` | `/api/v1/bookings` | **Headers:** `Authorization: Bearer <JWT>`, `Idempotency-Key: uuid` (обов'язковий)<br><br>**Body (`CreateBookingRequest`):** `client_id: uuid`, `service_id: uuid`, `phases: PhaseSlotDTO[]` (`minItems: 1`) | Атомарно резервує фази послуги у статусі `HOLD` (TTL $600\text{ с}$), перевіряє темпоральний інваріант `EXCLUDE USING gist` та фіксує подію `BOOKING_HELD`. Повертає `201 Created` або `409 Conflict`.<br><br>**Контейнери:** `Reverse Proxy` $\to$ `Web API (FastAPI)` $\to$ `Redis (Idempotency)` + `PostgreSQL 16+ (GiST & Outbox)`. |
| **API-REQ-03** | `PATCH` | `/api/v1/bookings/{booking_id}/confirm` | **Headers:** `Authorization: Bearer <JWT>`<br><br>**Path:** `booking_id: uuid`<br><br>**Body (`ConfirmBookingRequest`):** `version: int >= 1` | Переводить бронювання зі стану `HOLD` у `CONFIRMED` з перевіркою оптимістичного блокування (`version`) та строку дії холду (`hold_expires_at`).<br><br>**Контейнери:** `Reverse Proxy` $\to$ `Web API (FastAPI)` $\to$ `PostgreSQL 16+` $\to$ `Outbox Publisher Worker`. |
| **API-REQ-04** | `GET` | `/api/v1/schedules/stream` | **Headers:** `Authorization: Bearer <JWT>`, `Last-Event-ID: string` (опціонально при реконекті)<br><br>**Query:** `target_date: date` | Відкриває однонапрямний потік реактивної синхронізації матриці розкладу (`Content-Type: text/event-stream`, подія `SCHEDULE_MUTATED`).<br><br>**Контейнери:** `Reverse Proxy (HTTP/2)` $\to$ `Web API (FastAPI)` $\leftarrow$ `Redis 7.x (Pub/Sub)` $\leftarrow$ `Outbox Publisher Worker`. |

### 2.5.2. Специфікація наскрізних потоків даних (Data Flows) та інваріантів мережевого рівня

Наскрізний потік даних **DF-01** (атомарна фіксація багатофазного бронювання та реактивна синхронізація `API-REQ-02` $\to$ `API-REQ-04`) реалізується у 5 послідовних етапів:

1. **Формування запиту на клієнті (`Single Page Application — Vue.js 3`):** Після вибору варіанту розкладу клієнтський застосунок генерує криптографічно випадковий ключ ідепотентності `Idempotency-Key` (UUID v4) та надсилає JSON-пакет `CreateBookingRequest` методом `HTTPS POST` на `/api/v1/bookings`, передаючи токен доступу виключно у заголовку `Authorization: Bearer <JWT>` (передача секретів у Query String суворо заборонена).
2. **Мережева фільтрація на периметрі (`Reverse Proxy / Gateway — Nginx / Envoy`):** Виконується термінація TLS 1.3 та перевірка **CORS-політик** (дозволено виключно довірений `Origin` клієнтського SPA із підтримкою `Access-Control-Allow-Credentials: true`). Застосовується жорстке обмеження розміру тіла запиту **`Max Body Size = 64 KB`** (`client_max_body_size 64k`) для захисту від DoS-атак великими JSON-пейлоадами та обмежувач частоти запитів **Rate Limiter (Token Bucket)** (`HTTP 429 Too Many Requests`).
3. **Автентифікація, захист від IDOR та контроль ідепотентності (`Web API Application — FastAPI`):** Middleware валідує підпис `JWT RS256` за кешованим публічним ключем `Identity Provider` (JWKS). Сервер порівнює `jwt.sub` із полем `payload.client_id` (захист від IDOR — у разі невідповідності повертається `HTTP 403 Forbidden`), валідує схеми через `Pydantic v2` та перевіряє ключ ідепотентності у **Redis 7.x** (`GET idemp:{Idempotency-Key}`).
4. **ACID-транзакція та контроль колізій (`Main Relational Database — PostgreSQL 16+`):** У межах єдиної транзакції `Unit of Work` створюється запис у `BOOKINGS` (`status = 'HOLD'`, `hold_expires_at = NowUTC() + 600 s`, `version = 1`), вставляються фази у `BOOKING_PHASES` як напіввідкриті діапазони `tstzrange` (`[start_time, end_time)`) та записується подія `BOOKING_HELD` у таблицю `OUTBOX_EVENTS`. При накладанні інтервалів індекс `EXCLUDE USING gist` генерує `SQLSTATE 23P01` $\to$ виконується `ROLLBACK` і повертається `HTTP 409 Conflict`. При успіху виконується `COMMIT`, ключ зберігається в Redis (`EX 86400`), а клієнту повертається `HTTP 201 Created`.
5. **Асинхронна ретрансляція події (`Outbox Publisher Worker` $\to$ `Redis Pub/Sub` $\to$ `SSE`):** Фоновий воркер вичитує необроблені записи з `OUTBOX_EVENTS` (`SELECT ... FOR UPDATE SKIP LOCKED`), публікує знеособлену дельту змін у канал `Redis Pub/Sub` і проставляє `processed_at`. Відкриті HTTP/2 з'єднання `GET /api/v1/schedules/stream` отримують фрейм `event: SCHEDULE_MUTATED` із монотонним `id` події (для підтримки ресинхронізації через `Last-Event-ID`).

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
   - Якщо `DAG[i].Type = 'ACTIVE'`, формується підмножина кваліфікованих майстрів `Qualified[i] := FilterBySkill(Masters, DAG[i])`. Якщо `Length(Qualified[i]) = 0`, встановлюється `HasResourceGap := TRUE`. Інакше для кожного `m` (`0 <= m < Length(Qualified[i])`) створюється опціональний інтервал `OptVar[i][m] := NewOptionalIntervalVar(DAG[i], Qualified[i][m])` та накладається обмеження вибору рівно одного виконавця `AddExactlyOne(OptVar[i][0..K-1])`.
   - Якщо `DAG[i].Type <> 'ACTIVE'` (фаза `PASSIVE_WAIT`), реєструється фіксований часовий інтервал без прив'язки до виконавця `PassiveVar[i] := NewIntervalVar(DAG[i])`.
3. **Перевірка доступності ресурсів:** Якщо `HasResourceGap = TRUE`, виконання припиняється зі статусом `NO_AVAILABLE_RESOURCES`.
4. **Оптимізація та резервування (Fallback):** Накладаються констрейнти неперетину `AddNoOverlapConstraints(Model, OptVar)`, задається векторна лексикографічна цільова функція `SetLexicographicObjective(Model)` та запускається розв'язувач із обмеженням `Solver.MaxTimeInSeconds := T_max / 1000`.
   - Якщо `Status = 'OPTIMAL' OR Status = 'FEASIBLE'`, формується результат `Candidates := ExtractCPSATSchedule(Model)`.
   - Інакше (при перевищенні бюджету часу або помилці) управління передається на резервний рушій інтервальних дерев `Candidates := RunBaselineIntervalTree(DAG, Masters)`, `Mode := 'BASELINE_HEURISTIC_FALLBACK'`. Якщо `Length(Candidates) = 0`, повертається статус `NO_SLOTS_FOUND`.
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
   - Якщо `DbError <> NULL`, виконується `RollbackTransaction()`. При коді `DbError = '23P01'` (спрацювання індексу `EXCLUDE USING gist`) повертається `HTTP 409 Conflict`; при інших системних збоях СУБД — `HTTP 503 Service Unavailable`.
   - Якщо `DbError = NULL`, у межах тієї ж транзакції генерується `EventID := GenerateUUIDv4()` та записується подія `BOOKING_HELD` у таблицю `OUTBOX_EVENTS`, після чого виконується `CommitTransaction()`, збереження ключа ідепотентності та повернення статусу `HTTP 201 Created (BookingID, ExpiresAt)`.

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
| --- | --- | --- | --- |
| 1 | Узгодженість типів ключів (`UUID`), фінансів (`NUMERIC(12,2)`) та дат (`ISO 8601` $\to$ `TSTZRANGE`) | **PASS** | Повна типова сумісність між [openapi.yaml](https://www.google.com/search?q=./api/openapi.yaml), Pydantic v2 та PostgreSQL 16+ |
| 2 | Покриття захистом від стану гонитви (Race Conditions) та подвійних бронювань | **PASS** | `EXCLUDE USING gist` у БД $\leftrightarrow$ `SQLSTATE 23P01` в Алгоритмі 3 $\leftrightarrow$ `409 Conflict` в API |
| 3 | Узгодженість механізму підтвердження холду (`HOLD` $\to$ `CONFIRMED`) та оптимістичного блокування | **PASS** | `hold_expires_at` + `version` у `BOOKINGS` $\leftrightarrow$ `PATCH /bookings/{booking_id}/confirm` |
| 4 | Атомарність подійної синхронізації (Transactional Outbox $\to$ Redis Pub/Sub $\to$ SSE) | **PASS** | Таблиця `OUTBOX_EVENTS` + Алгоритм 3 $\leftrightarrow$ `GET /schedules/stream` |
| 5 | Академічний нормоконтроль формалізації алгоритмів (`:=`, `AND/OR/NOT`, індекси) | **PASS** | Усі 3 алгоритми відповідають вимогам ДСТУ / ISO 5807 |

**Висновок комісії:** Архітектурний, даталогічний та алгоритмічний контури Розділу 2 є повністю узгодженими, не містять інтеграційних блокерів і затверджуються для переходу до програмної реалізації.

---

## 2.8. Моделювання користувацького досвіду та транзакційних сценаріїв (Use Cases) {#use-cases}

Межі системи, ролі акторів та поведінкові сценарії взаємодії формалізовано на діаграмі прецедентів UML та у деталізованих специфікаціях транзакційних сценаріїв.

### 2.8.1. Специфікація прецеденту UC-02: Оформлення та підтвердження багатофазного бронювання

- **Первинний Актор:** Клієнт (`CLIENT`) | **Вторинні Актори:** `Identity Provider`, `Notification Gateway`
- **Продуктові передумови:** Користувач автентифікований (`JWT RS256`, роль `CLIENT`), сформовано валідний кандидат розкладу `ScheduleCandidateDTO` (`UC-01`).
- **Інфраструктурні передумови:** Пул `asyncpg` до `PostgreSQL 16+` активний, індекс `EXCLUDE USING gist` у стані `VALID`, `Redis 7.x` та `Outbox Publisher Worker` доступні.

| Крок | Дія Актора (Actor Action) | Реакція та відповідь системи (System Response) |
| --- | --- | --- |
| **1.** | Клієнт ініціює резервування обраного слоту («Забронювати»). | SPA генерує `Idempotency-Key` (UUID) і надсилає `POST /api/v1/bookings`. Backend валідує `JWT RS256`, перевіряє відсутність ключа в Redis (`GET idemp:{key}`) і валідує `CreateBookingRequest` через `Pydantic v2`. |
| **2.** | *(Очікування створення холду)* | Backend у транзакції `READ COMMITTED` створює запис `BOOKINGS` (`HOLD`, `hold_expires_at := NowUTC() + 600 s`, `version := 1`), вставляє фази в `BOOKING_PHASES` (`tstzrange`) під контролем індексу `GiST`, записує подію `BOOKING_HELD` в `OUTBOX_EVENTS`, виконує `COMMIT`, кешує ключ у Redis (`EX 86400`) і повертає `HTTP 201 Created`. |
| **3.** | Клієнт підтверджує деталі замовлення. | SPA надсилає `PATCH /api/v1/bookings/{booking_id}/confirm` із поточною версією `{"version": 1}`. |
| **4.** | *(Очікування фінального підтвердження)* | Backend перевіряє `status = 'HOLD'`, `NowUTC() <= hold_expires_at` та `version = 1`. Оновлює статус на `CONFIRMED` (`version := version + 1`), записує подію `BOOKING_CONFIRMED` в `OUTBOX_EVENTS`, виконує `COMMIT` та повертає `HTTP 200 OK`. `Outbox Publisher Worker` транслює оновлення в `Redis Pub/Sub` (SSE) і ставить задачу сповіщення в чергу `Notification Gateway`. |

- **Постумови:** Запис `BOOKINGS` перебуває у статусі `CONFIRMED` (`hold_expires_at = NULL`, `version = 2`), темпоральні інтервали заблоковані в `BOOKING_PHASES`, подія оброблена воркером Outbox.
- **Альтернативні потоки та виключення:**
  - **AF-02A (Колізія слотів — старт із Кроку №2, повернення до `UC-01`):** При спрацюванні індексу `EXCLUDE USING gist` (`SQLSTATE 23P01`) сервер виконує `ROLLBACK` і повертає `HTTP 409 Conflict` (`RFC 9457`), а SPA ініціює перерахунок вільних слотів.
  - **AF-02B (Повторний запит `Idempotency-Key` — старт із Кроку №1, перехід до Кроку №3):** При знаходженні ключа в Redis сервер повертає кешовану відповідь `HTTP 200 OK` без дублювання запису в БД.
  - **AF-02C (Сплив TTL холду — старт із Кроку №4, повернення до `UC-01`):** Якщо `NowUTC() > hold_expires_at`, сервер відхиляє підтвердження з кодом `HTTP 409 Conflict (HoldExpiredError)`.
  - **EF-02A (Збій з'єднання з PostgreSQL):** Сервер виконує гарантований `ROLLBACK` транзакції, не зберігає ключ ідепотентності та повертає `HTTP 503 Service Unavailable`.
  - **EF-02B (Тимчасова недоступність Redis / Notification Gateway після `COMMIT`):** Подія зберігається в `OUTBOX_EVENTS` (`processed_at = NULL`) і доставляється воркером за алгоритмом *Exponential Backoff* після відновлення зв'язку.

### 2.8.2. Специфікація прецеденту UC-03: Диспетчерське перенесення фази (Drag-and-Drop)

При перетягуванні диспетчером блоку фази на матриці часу (`PATCH /api/v1/bookings/{booking_id}/phases/{phase_id}`) сервер перевіряє права ролі `DISPATCHER`/`ADMIN`, сумісність навичок нового майстра (`MASTER_SKILLS`), топологічні лаги (`min_lag_minutes`) та оновлює `tstzrange` під контролем індексу `GiST` та оптимістичного блокування (`version`). При конфлікті версій (`AF-03A`) повертається `HTTP 409 Conflict`, а SPA повертає блок на вихідну позицію.

---

## Висновки до розділу 2

У результаті виконання проєктного етапу розроблено цілісну архітектуру підсистеми динамічного планування та оптимізації багатофазних процесів, побудовану за парадигмою розподіленого модульного моноліту з виділеними обчислювальними та подієвими воркерами (`FastAPI`, `Google OR-Tools CP-SAT`, `PostgreSQL 16+`, `Redis 7.x`, `Vue.js 3`). Декомпозиція системи за методологією **C4 Model (Levels 1–3)** забезпечила чітке розмежування I/O-контуру вебсервера та ресурсомістких CPU-bound обчислень оптимізації розкладу з жорстким бюджетом часу $T_{max} = 800\text{ мс}$ і резервним евристичним алгоритмом на інтервальних деревах, що гарантує дотримання нефункціональних вимог щодо пропускної здатності ($500\text{ RPS}$) та затримок ($p_{95} \le 100\text{ мс}$).

Спроєктовано нормалізовану реляційну модель даних (3NF) із застосуванням діапазонного типу `TSTZRANGE` та декларативного індексного обмеження `EXCLUDE USING gist` (`btree_gist`), що на рівні ядра СУБД унеможливлює виникнення подвійних бронювань за умов високої конкуренції запитів. Поєднання патернів **Unit of Work**, **Transactional Outbox** та протоколу **W3C Server-Sent Events (SSE)** із підтримкою `Last-Event-ID` усунуло ризики аномалії *Dual-Write* і втрати подій при мережевих збоях, а впровадження багаторівневого контуру **Application Security** (`JWT RS256`, `RBAC + Ownership Guard`, `Pydantic v2 extra="forbid"`, `Idempotency-Key` та `Redis Rate Limiting`) закрило критичні вектори атак за стандартом OWASP API Top 10.

Проведений наскрізний інтеграційний аудит підтвердив повну узгодженість між схемою бази даних, контрактами `OpenAPI 3.0.3`, поведінковими специфікаціями транзакційних сценаріїв (`Use Cases`) та математичними алгоритмами критичних вузлів. Отриманий комплекс інженерних артефактів не містить архітектурних суперечностей і є повністю готовим до переходу на етап програмної реалізації та тестування системи (Розділ 3 кваліфікаційної роботи).
