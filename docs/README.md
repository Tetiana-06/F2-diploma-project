# Documentation Dashboard — Qualification Thesis Project

**Спеціальність:** 121 «Інженерія програмного забезпечення»  
**Тема проєкту:** Підсистема динамічного планування та оптимізації багатофазних процесів в умовах гетерогенних ресурсних обмежень  
**Студентка:** Шостак Тетяна  

---

##  Робочий контур документації

* **Хмарний робочий файл Розділу 1 (Google Docs):** [F2_Diploma_Chapter1_Shostak (Режим коментування)](https://docs.google.com/document/d/1hMwIJXCw0ZT-kP-8Pf7YQW8NmCeXGNGTXLlkAFiOemY/edit?usp=sharing)
* **Структура Markdown-документації:**
  * [`/docs/0_intro.md`](./0_intro.md) — Вступ, науковий апарат, актуальність та об'єкт/предмет дослідження.
  * [`/docs/1_analysis.md`](./1_analysis.md) — Аналіз предметної області, огляд аналогів, модель As-Is/To-Be, функціональні та нефункціональні вимоги (Core Features & NFRs).

---

##  Зведена матриця нефункціональних вимог (NFRs Summary Matrix)

| Категорія NFR | Ключова інженерна метрика / Індикатор | Технологічне забезпечення (Tech Stack) |
| :--- | :--- | :--- |
| **Performance (REST API)** | $Latency \le 100$ мс ($p_{95}$) при навантаженні 500 RPS | FastAPI, Pydantic v2, `asyncpg`, uvicorn |
| **Solver Performance** | $Latency \le 1000$ мс ($p_{95}$) для CP-SAT оптимізації | Google OR-Tools (CP-SAT Core), time-budgeting |
| **Concurrency & Integrity** | $0\%$ фантомних бронювань (Phantom Bookings) | PostgreSQL 16+ (`tstzrange` + `EXCLUDE USING gist`) |
| **Async Isolation** | Повна ізоляція CPU-bound обчислень від I/O | Task-Worker Pattern (Taskiq + Redis Worker Pool) |
| **Real-time Event Streaming** | Атомарна трансляція подій мутації розкладу | Transactional Outbox Pattern + Redis Pub/Sub + SSE |
| **Security & Privacy** | OAuth 2.0 / OIDC (JWT RS256), PII-знеособлення (GDPR) | FastAPI Security, UUID-деперсоналізація в CP-SAT |
| **Quality Engineering** | $100\%$ покриття інваріантів алгоритму тестами | Property-based testing (`Hypothesis`), `testcontainers` |