# РОЗДІЛ 2. АРХІТЕКТУРНЕ ПРОЄКТУВАННЯ, МОДЕЛЮВАННЯ ДАНИХ ТА АЛГОРИТМІЗАЦІЯ NLP/AI-КОНВЕЄРА

> **Статус документа:** Розроблено підрозділ `2.1` (модель C4 рівнів 1–3, трасування до [SRS](./srs.md) та [ADR-001](./adr/0001-choice-of-technology.md)). Підрозділи `2.2–2.3` перебувають на наступних етапах календарного плану.

---

## 2.1. Контекстне, контейнерне та компонентне моделювання системи (C4 Model)

Проєктування архітектури виконано за методологією **C4 Model** Саймона Брауна з використанням підходу *Diagrams-as-Code* (`Mermaid.js`) на основі затвердженого контракту вимог [SRS](./srs.md) та архітектурного рішення [ADR-001](./adr/0001-choice-of-technology.md).

---

### 2.1.1. Контекстне та контейнерне моделювання системи (C4 Levels 1–2)

<a id="diagram-level1"></a>
#### Діаграма C4 Level 1 — System Context Diagram

```mermaid
C4Context
    title System Context Diagram — Lexical Extraction & WSD Platform

    Person(user, "Кінцевий користувач", "Вивчає мови (en, de, es), передає URL або аудіофайли, проводить тріаж слів і завантажує колоди карток")
    Person(operator, "Оператор моніторингу (SRE)", "Контролює метрики продуктивності /metrics, довжину черг ARQ, DLQ та журнали X-Correlation-ID")

    System(lex_system, "Lexical Extraction & WSD System", "Modular Monolith для автоматизованого вилучення лексики, оцінки складності (i+1 / CEFR), двоконтурної дезамбігуації та експорту карток")

    System_Ext(media_src, "Media & Web Providers", "YouTube, подкасти, вебстатті (субтитри, аудіопотоки, HTML-контент)")
    System_Ext(llm_api, "System-2 Cloud LLM API", "OpenAI-сумісний сервіс пакетної дезамбігуації (Structured Outputs, Zero Data Retention)")
    System_Ext(srs_app, "Anki / SRS Platforms", "Зовнішні клієнти інтервального повторення (імпорт .apkg, .tsv, .csv, .md)")

    Rel(user, lex_system, "Передає джерела, виконує тріаж, завантажує картки", "HTTPS / TLS 1.3 (JWT Bearer)")
    Rel(operator, lex_system, "Аудит метрик черг і стану сервісів", "HTTPS / Prometheus & JSON Logs")
    Rel(lex_system, media_src, "Вилучення субтитрів, вебтексту й аудіо (із захистом SSRF)", "HTTPS / TLS 1.3")
    Rel(lex_system, llm_api, "Пакетна дезамбігуація (під захистом Circuit Breaker)", "HTTPS / JSON Schema REST")
    Rel(user, srs_app, "Імпортує експортовані пакети карток", "Локальні файли (.apkg, .tsv, .csv, .md)")
```

##### Технічний опис вузлів, акторів та протоколів рівня System Context
- **Кінцевий користувач:** взаємодіє із системою за протоколом `HTTPS / TLS 1.3` (`JWT Bearer`), реалізуючи сценарії інгестії ([`SRS-FUN-01`](./srs.md#srs-fun-01), [`SRS-FUN-02`](./srs.md#srs-fun-02)), тріажу та експорту колод ([`SRS-FUN-14`](./srs.md#srs-fun-14)).
- **Оператор моніторингу (`SRE`):** отримує доступ до зібраних метрик і структурованих журналів (`X-Correlation-ID`), реалізуючи вимоги спостережуваності [`SRS-NFR-OBS-01`](./srs.md#srs-nfr-obs-01) та контролю збоїв [`SRS-FUN-13`](./srs.md#srs-fun-13).
- **`Media & Web Providers`:** зовнішні веб- і медіаджерела, звернення до яких виконується через `HTTPS / TLS 1.3` із перевіркою IP-діапазонів (`SSRF Guard`, [`SRS-FUN-01`](./srs.md#srs-fun-01), [`SRS-NFR-SEC-02`](./srs.md#srs-nfr-sec-02)).
- **`System-2 Cloud LLM API`:** зовнішній провайдер дезамбігуації, взаємодія з яким відбувається через `HTTPS / JSON Schema` із маскуванням `PII` ([`SRS-FUN-11`](./srs.md#srs-fun-11)) і захистом `Circuit Breaker` ([`SRS-NFR-REL-01`](./srs.md#srs-nfr-rel-01)).
- **`Anki / SRS Platforms`:** зовнішні автономні клієнти, у які користувач імпортує сформовані пакети `.apkg`, `.tsv`, `.csv` та `.md` ([`SRS-FUN-14`](./srs.md#srs-fun-14)).

---

<a id="diagram-level2"></a>
#### Діаграма C4 Level 2 — Container Diagram

```mermaid
C4Container
    title Container Diagram — Lexical Extraction & WSD System (Docker Compose)

    Person(user, "Кінцевий користувач", "Клієнт вебзастосунку / REST API")
    Person(operator, "Оператор моніторингу (SRE)", "Інженер супроводу")

    System_Boundary(c1, "Lexical Extraction & WSD Platform (Docker Compose)") {
        Container(proxy, "Reverse Proxy", "Nginx 1.25 (Alpine)", "Термінація TLS 1.3, ліміт завантаження client_max_body_size 500M, генерація X-Correlation-ID, захист внутрішніх точок")
        Container(api, "FastAPI Backend API", "Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0", "Автентифікація OAuth 2.0 / JWT, валідація magic bytes, SSRF Guard, ідемпотентність, маршрутизація черг, експорт карток")
        Container(nlp_worker, "NLP Background Worker", "Python 3.12, ARQ, spaCy, Jev, HTTPX (<= 512 MiB)", "Обслуговує default_queue: вебвилучення, лематизація, MWE, оцінка i+1, двоконтурний WSD (System-1 / System-2), GDPR")
        Container(transcribe_worker, "ASR Background Worker", "Python 3.12, ARQ, faster-whisper INT8, yt-dlp, FFmpeg (<= 2.0 GiB)", "Обслуговує heavy_transcribe_queue: завантаження медіа, локальна транскрибація до 120 хв, запис у transcribe_dlq")
        ContainerQueue(redis, "Redis 7 Broker & Cache", "Redis 7.2 (In-Memory)", "Черги default_queue, heavy_transcribe_queue, default_dlq, transcribe_dlq; кеш ідемпотентності, Token Bucket, JTI Blacklist")
        ContainerDb(pg, "PostgreSQL 16 Database", "PostgreSQL 16 (Relational DB)", "Користувачі, MediaSource, Transcript, global_lemmas, lexical_senses, user_vocabulary, SourceAnalysisSession")
        ContainerDb(minio, "MinIO Object Storage", "MinIO (S3 Compatible)", "Транзитне зберігання завантажених аудіофайлів (автоочищення за Lifecycle Rule <= 24 год)")
        Container(prom, "Prometheus", "Prometheus TSDB", "Періодичний збір метрик продуктивності та глибини черг ARQ")
    }

    System_Ext(media_src, "Media & Web Providers", "YouTube / Подкасти / Вебсторінки")
    System_Ext(llm_api, "System-2 Cloud LLM API", "Зовнішній OpenAI-сумісний API")

    Rel(user, proxy, "Запити API, завантаження медіа (<= 500 MB), експорт", "HTTPS / TLS 1.3")
    Rel(operator, proxy, "Перегляд метрик і стану черг", "HTTPS / TLS 1.3 (Basic Auth)")
    Rel(proxy, api, "Проксування API-запитів із X-Correlation-ID", "HTTP / Reverse Proxy (TCP 8000)")
    Rel(proxy, prom, "Доступ до панелі метрик", "HTTP (TCP 9090)")

    Rel(api, redis, "Постановка задач, Token Bucket, X-Idempotency-Key, JTI Blacklist", "RESP3 / TCP (порт 6379)")
    Rel(api, pg, "CRUD, автентифікація, перевірка SHA-256, вибірка для експорту", "PostgreSQL Wire / AsyncPG (TCP 5432)")
    Rel(api, minio, "Збереження перевірених аудіофайлів", "S3 REST API / HTTP (порт 9000)")
    Rel(prom, api, "Періодичний збір метрик /metrics (раз на 15 с)", "HTTP (TCP 8000)")

    Rel(nlp_worker, redis, "Вилучення задач (BLPOP) із default_queue, запис у default_dlq", "RESP3 / TCP (порт 6379)")
    Rel(transcribe_worker, redis, "Вилучення задач із heavy_transcribe_queue, запис у transcribe_dlq", "RESP3 / TCP (порт 6379)")
    Rel(nlp_worker, pg, "Збереження лем, сенсів, статусів та асинхронне GDPR-очищення", "PostgreSQL Wire / AsyncPG (TCP 5432)")
    Rel(transcribe_worker, pg, "Збереження транскриптів і часових міток", "PostgreSQL Wire / AsyncPG (TCP 5432)")
    Rel(transcribe_worker, minio, "Читання транзитних аудіофайлів", "S3 REST API / HTTP (порт 9000)")

    Rel(nlp_worker, media_src, "Отримання HTML вебстатей із перевіркою SSRF", "HTTPS / TLS 1.3")
    Rel(transcribe_worker, media_src, "Завантаження медіапотоків та субтитрів із перевіркою SSRF", "HTTPS / TLS 1.3")
    Rel(nlp_worker, llm_api, "Пакетна дезамбігуація (Circuit Breaker + Token Bucket)", "HTTPS / JSON Schema REST")
```

##### Технічний опис контейнерів та наскрізне трасування до вимог SRS і ADR
1. **`Reverse Proxy` (`Nginx 1.25`):** забезпечує термінацію `HTTPS / TLS 1.3`, обмеження максимального розміру тіла запиту `500 MB`, захист від повільних клієнтів і генерацію наскрізного ідентифікатора `X-Correlation-ID` ([`SRS-NFR-SEC-02`](./srs.md#srs-nfr-sec-02), [`SRS-NFR-OBS-01`](./srs.md#srs-nfr-obs-01)).
2. **`FastAPI Backend API`:** реалізує автентифікацію `JWT Bearer / Argon2id` ([`SRS-NFR-SEC-01`](./srs.md#srs-nfr-sec-01)), прийом `URL` і файлів з перевіркою *magic bytes* та `SSRF` ([`SRS-FUN-01`](./srs.md#srs-fun-01), [`SRS-FUN-02`](./srs.md#srs-fun-02)), ідемпотентну постановку задач ([`SRS-FUN-03`](./srs.md#srs-fun-03), [`SRS-NFR-PERF-02`](./srs.md#srs-nfr-perf-02)) та генерацію файлів експорту ([`SRS-FUN-14`](./srs.md#srs-fun-14)).
3. **`NLP Background Worker` (`nlp_worker`, ліміт `<= 512 MiB`):** фізично відокремлений процес для обслуговування `default_queue`, що реалізує розбір тексту `spaCy` (`LRU`-кеш на `1` модель), оцінку `i+1 / CEFR`, локальну фільтрацію `System-1 Jev`, пакетні звернення до `System-2 LLM` ([`SRS-FUN-06`](./srs.md#srs-fun-06)–[`SRS-FUN-12`](./srs.md#srs-fun-12)) та асинхронне очищення профілів за `GDPR` ([`SRS-NFR-COMP-01`](./srs.md#srs-nfr-comp-01)). Вибір ізоляції обґрунтовано в [`ADR-001`](./adr/0001-choice-of-technology.md).
4. **`ASR Background Worker` (`transcribe_worker`, ліміт `<= 2.0 GiB`):** ізольований контейнер для `heavy_transcribe_queue`, що завантажує субтитри/аудіо й виконує локальну транскрибацію через `faster-whisper-small (INT8)` до `120 хв` без фрагментації `25 MB` ([`SRS-FUN-04`](./srs.md#srs-fun-04), [`SRS-FUN-05`](./srs.md#srs-fun-05), [`SRS-NFR-PERF-04`](./srs.md#srs-nfr-perf-04), [`SRS-NFR-INFRA-01`](./srs.md#srs-nfr-infra-01)).
5. **`Redis 7 Broker & Cache`:** виконує роль брокера черг (`default_queue`, `heavy_transcribe_queue`, `default_dlq`, `transcribe_dlq`) за протоколом `RESP3 / TCP (6379)`, кешу ідемпотентності `X-Idempotency-Key`, лічильників `Token Bucket` та чорного списку `JTI` ([`SRS-FUN-03`](./srs.md#srs-fun-03), [`SRS-FUN-13`](./srs.md#srs-fun-13), [`ADR-001`](./adr/0001-choice-of-technology.md)).
6. **`PostgreSQL 16 Database`:** центральне реляційне сховище (`PostgreSQL Wire / AsyncPG`, порт `5432`), що підтримує індексний пошук дублікатів за `SHA-256` ([`SRS-NFR-PERF-03`](./srs.md#srs-nfr-perf-03)) та атомарне збереження `global_lemmas` і `lexical_senses` ([`SRS-FUN-12`](./srs.md#srs-fun-12), [`ADR-001`](./adr/0001-choice-of-technology.md)).
7. **`MinIO Object Storage`:** S3-сумісне транзитне сховище (`S3 REST API / HTTP`, порт `9000`) з політикою автоматичного видалення файлів через `24 год` ([`SRS-FUN-02`](./srs.md#srs-fun-02), [`SRS-NFR-COMP-01`](./srs.md#srs-nfr-comp-01)).
8. **`Prometheus`:** внутрішній сервіс моніторингу, що опитує `/metrics` кожні `15 с` ([`SRS-NFR-OBS-01`](./srs.md#srs-nfr-obs-01)).

---

### 2.1.2. Компонентне моделювання критичного сервісу (C4 Level 3 — Component Diagram)

Для поглибленого проєктування обрано найскладніший серверний контейнер — **`NLP & Semantic Disambiguation Worker (nlp_worker)`**, який поєднує `CPU`-лінгвістику, керування пам'яттю, захист персональних даних і відмовостійку взаємодію із зовнішнім `LLM API`. Внутрішню структуру побудовано за принципами **Clean Architecture** та **Dependency Inversion (DIP)** через контракти `typing.Protocol`.

<a id="diagram-level3"></a>
#### Діаграма C4 Level 3 — Component Diagram (`NLP & Semantic Disambiguation Worker`)

```mermaid
C4Component
    title Component Diagram — NLP & Semantic Disambiguation Worker (Clean Architecture)

    ContainerQueue(redis_q, "Redis 7 (Queues & Token Bucket)", "Redis 7.2", "Черги default_queue, default_dlq та лічильники Token Bucket")
    ContainerDb(pg_db, "PostgreSQL 16", "PostgreSQL 16", "Таблиці MediaSource, Transcript, global_lemmas, lexical_senses, user_vocabulary")
    System_Ext(web_ext, "External Web Sites", "Зовнішні вебстатті")
    System_Ext(llm_ext, "System-2 Cloud LLM API", "OpenAI-сумісний сервіс дезамбігуації")

    Container_Boundary(nlp_worker, "NLP & Semantic Disambiguation Worker (nlp_worker)") {
        Component(consumer, "ARQ Task Consumer", "ARQ Worker Entrypoint", "Отримує завдання з default_queue і передає його у шар валідації")
        Component(val_ctx, "Task Validator & Tenant Guard", "Pydantic v2 + ContextVars", "Валідує DTO задачі, відновлює X-Correlation-ID і перевіряє права власника user_id")
        Component(err_handler, "Worker Fault & DLQ Handler", "Resilience Handler", "Перехоплює критичні винятки, переводить сесію у FAILED та маршрутизує в default_dlq")

        Component(use_case, "ProcessAnalysisPipelineUseCase", "Application Service", "Координує сценарій: завантаження -> NLP -> оцінка i+1 -> PII-санітизація -> WSD -> збереження")

        Component(ling_svc, "Linguistic Analysis Service", "spaCy Pipeline (LRU <= 1 model)", "Токенізація, лематизація, POS-розмітка і детекція MWE (Dependency Matcher)")
        Component(profiler, "Complexity Profiling Service", "Domain Service", "Розрахунок покриття i+1 / CEFR і формування рейтингу кандидатів")
        Component(pii_guard, "PII Sanitizer", "Domain Privacy Filter", "Маскування email, телефонів та імен у контекстах перед передачею в LLM")
        Component(wsd_engine, "WSD Strategy Engine", "Domain Strategy", "Двоконтурна маршрутизація: швидкий System-1 (Jev) vs пакетний System-2 (LLM)")
        Component(jev_clf, "System-1 Jev Strategy", "Local Classifier", "Локальна дезамбігуація однозначних та високовживаних лем")

        Component(ports, "Application Ports", "typing.Protocol", "Абстрактні інтерфейси: WebScraperPort, VocabRepoPort, SenseRepoPort, LLMProviderPort")

        Component(web_adapter, "Safe Web Scraper Adapter", "HTTPX + trafilatura + SSRF Guard", "Перевіряє DNS/IP проти приватних підмереж і вилучає HTML-текст")
        Component(llm_adapter, "Resilient LLM Adapter", "HTTPX + Tenacity + Circuit Breaker", "Перевіряє Token Bucket, виконує до 3 спроб (budget <= 8s) і перемикає на System-1 (<= 500 ms)")
        Component(pg_repo, "PostgreSQL Persistence Adapter", "SQLAlchemy 2.0 AsyncPG", "Індексна вибірка словника користувача і транзакційний запис INSERT ... ON CONFLICT")
    }

    Rel(consumer, redis_q, "Опитування задач (BLPOP)", "RESP3 / TCP")
    Rel(consumer, val_ctx, "Передача сирого повідомлення задачі", "Python Call")
    Rel(val_ctx, use_case, "Запуск валідованого сценарію", "Python DTO Call")
    Rel(consumer, err_handler, "Обробка фатальних винятків після max_tries", "Exception Trap")
    Rel(err_handler, ports, "Оновлення статусу FAILED через абстрактний порт", "Protocol Dispatch")
    Rel(err_handler, redis_q, "Запис діагностики у default_dlq", "RESP3 / TCP")

    Rel(use_case, ports, "Взаємодія із зовнішніми ресурсами через інтерфейси", "Protocol DI")
    Rel(use_case, ling_svc, "Морфосинтаксичний аналіз тексту", "Python Call")
    Rel(use_case, profiler, "Обчислення індексу i+1 / CEFR", "Python Call")
    Rel(use_case, pii_guard, "Очищення контекстів від персональних даних", "Python Call")
    Rel(use_case, wsd_engine, "Запуск дезамбігуації кандидатів", "Python Call")

    Rel(wsd_engine, jev_clf, "Первинна локальна класифікація", "Strategy Call")
    Rel(wsd_engine, ports, "Виклик LLMProviderPort для складних випадків", "Protocol DI")

    Rel(web_adapter, ports, "Реалізує WebScraperPort", "Implements Protocol")
    Rel(llm_adapter, ports, "Реалізує LLMProviderPort", "Implements Protocol")
    Rel(pg_repo, ports, "Реалізує VocabRepoPort та SenseRepoPort", "Implements Protocol")

    Rel(web_adapter, web_ext, "Отримання HTML-сторінки після SSRF-перевірки", "HTTPS / TLS 1.3")
    Rel(llm_adapter, redis_q, "Перевірка ліміту Token Bucket", "RESP3 / TCP")
    Rel(llm_adapter, llm_ext, "Пакетний запит (Structured Outputs)", "HTTPS / JSON Schema")
    Rel(pg_repo, pg_db, "Асинхронні транзакції AsyncSession", "PostgreSQL Wire / TCP 5432")
```

##### Деталізований опис внутрішніх компонентів сервісу, патернів і трасування до вимог

1. **Вхідний шар (`Inbound / Framework`):**
   - **`ARQ Task Consumer`:** точка входу фонового процесу, опитує `default_queue` у `Redis 7` (`RESP3 / TCP`), перехоплює винятки через `Worker Fault & DLQ Handler` і забезпечує виконання вимог [`SRS-FUN-03`](./srs.md#srs-fun-03), [`SRS-FUN-13`](./srs.md#srs-fun-13).
   - **`Task Validator & Tenant Guard`:** валідує вхідне повідомлення схемами `Pydantic v2`, встановлює контекст `X-Correlation-ID` ([`SRS-NFR-OBS-01`](./srs.md#srs-nfr-obs-01)) і перевіряє відповідність `user_id` власнику `SourceAnalysisSession` (ізоляція профілів користувачів).
   - **`Worker Fault & DLQ Handler`:** перехоплює критичні помилки обробки, оновлює статус сесії на `FAILED` через порт репозиторію (без прямого зв'язування з ORM) та записує діагностику в `default_dlq` ([`SRS-FUN-13`](./srs.md#srs-fun-13)).

2. **Шар координації (`Application Layer & Ports`):**
   - **`ProcessAnalysisPipelineUseCase`:** центральний оркестратор пайплайну, що ліквідує каскадну зв'язність між доменними сервісами; реалізує патерн **Pipeline / Application Service**.
   - **`Application Ports` (`typing.Protocol`):** реалізує принцип **Dependency Inversion (DIP)** — шар сценаріїв і домену залежить виключно від абстрактних інтерфейсів (`WebScraperPort`, `VocabRepoPort`, `SenseRepoPort`, `LLMProviderPort`), а не від конкретних клієнтів `SQLAlchemy` чи `HTTPX`. Залежності ін'єктуються під час старту процесу через `Composition Root`.

3. **Доменний шар (`Domain Layer`):**
   - **`Linguistic Analysis Service`:** виконує токенізацію, лематизацію й виявлення *split phrasal verbs* через `spaCy Dependency Matcher` для `en`, `de`, `es` із `LRU`-кешем (`<= 1` активна модель у `RAM`) — трасування до [`SRS-FUN-06`](./srs.md#srs-fun-06), [`SRS-FUN-07`](./srs.md#srs-fun-07), [`SRS-NFR-INFRA-01`](./srs.md#srs-nfr-infra-01).
   - **`Complexity Profiling Service`:** обчислює метрику *Comprehensible Input (`i+1`)* / `CEFR` ([`SRS-FUN-08`](./srs.md#srs-fun-08)) та відсікає відомі одиниці (`KNOWN`, `IGNORED`) без прямих запитів у БД ([`SRS-FUN-09`](./srs.md#srs-fun-09)).
   - **`PII Sanitizer`:** маскує персональні дані у прикладах речень перед формуванням запитів у зовнішній сервіс ([`SRS-FUN-11`](./srs.md#srs-fun-11), [`SRS-NFR-COMP-01`](./srs.md#srs-nfr-comp-01)).
   - **`WSD Strategy Engine` + `System-1 Jev Strategy`:** реалізує патерн **Strategy** для вибору між миттєвим локальним класифікатором [`System-1`](./srs.md#srs-fun-10) та пакетним викликом `System-2` ([`SRS-FUN-11`](./srs.md#srs-fun-11)).

4. **Вихідний шар інфраструктури (`Outbound Adapters`):**
   - **`Safe Web Scraper Adapter`:** завантажує `HTML`-статті через `trafilatura`, попередньо перевіряючи розв'язану DNS IP-адресу через `SSRF Guard` ([`SRS-FUN-01`](./srs.md#srs-fun-01), [`SRS-FUN-04`](./srs.md#srs-fun-04), [`SRS-NFR-SEC-02`](./srs.md#srs-nfr-sec-02)).
   - **`Resilient LLM Adapter`:** реалізує `LLMProviderPort`, перевіряє квоти `Token Bucket` у `Redis 7`, виконує до `3` повторних спроб у межах бюджету `<= 8.0 с` (`Exponential Backoff with Jitter`), а при стані `Circuit Breaker == OPEN` повертає управління `System-1` за `<= 500 мс` із `degraded_mode=true` ([`SRS-FUN-12`](./srs.md#srs-fun-12), [`SRS-NFR-REL-01`](./srs.md#srs-nfr-rel-01)).
   - **`PostgreSQL Persistence Adapter`:** виконує асинхронні транзакції через `SQLAlchemy 2.0 AsyncPG` із конструкцією `INSERT ... ON CONFLICT DO UPDATE / NOTHING` для підтримки унікальності `(lemma, part_of_speech)` при конкурентному записі ([`SRS-FUN-12`](./srs.md#srs-fun-12), [`ADR-001`](./adr/0001-choice-of-technology.md)).

---

### 2.1.3. Документування архітектурних рішень (`Architecture Decision Records`)

Ключовий інженерний компроміс проєкту — вибір топології **`Modular Monolith` (`FastAPI` + `Redis 7 / ARQ`)** замість розподілених мікросервісів на базі `Apache Kafka` та вбудованих `FastAPI BackgroundTasks` — зафіксовано за шаблоном Майкла Нігардза у документі [**`ADR-001`**](./adr/0001-choice-of-technology.md) (статус: **`Approved`**). Рішення кількісно збалансовує споживання `RAM` (`<= 50 MiB` для `Redis 7` проти `1.5–2.0 GiB` для `Kafka`), транзакційну `ACID`-цілісність словникового ядра, затримку диспетчеризації `p99 <= 25 мс` та чітко документує прийняті ризики (`AOF everysec` вікно `<= 1 с`, відсутність *Event Replay*, конкуренція за з'єднання `PostgreSQL 16` і вертикальна стеля односерверного розгортання).

---

## 2.2. Проєктування реляційної схеми бази даних PostgreSQL та структур оперативного сховища Redis

> *Підрозділ заповнюється на наступному етапі календарного плану (Етап №3).*

- Логічна та фізична ER-діаграма бази даних під керуванням `PostgreSQL 16`.
- Нормалізація зберігання лексичного ядра: розділення глобального словника лем (`global_lemmas`), семантичних значень полісемії (`lexical_senses`) та матриці станів вивчення користувача (`NEW`, `LEARNING`, `KNOWN`, `IGNORED`).
- Організація простору ключів у `Redis 7`: ізольовані черги задач `ARQ`, кеш `X-Idempotency-Key`, лімітер `Token Bucket` та чорний список `JTI`.

---

## 2.3. Алгоритмізація гібридного каскаду вилучення лексичних одиниць і семантичної дезамбігуації

> *Підрозділ заповнюється на етапі алгоритмізації (Етап №4).*

- Кількісна оцінка придатності матеріалу за моделлю *Comprehensible Input (`i+1`)* та морфосинтаксичний аналіз (`spaCy` із детектором `MWE`).
- Багатофакторна математична функція ранжування лем і швидка типізована фільтрація `System-1 (Jev)`.
- Контракти *Structured Outputs* (`Pydantic v2 / JSON Schema`) та алгоритм пакетного звернення до `System-2 (LLM)`.

---

## 2.4. Висновки до розділу 2

> *Підрозділ формується після завершення архітектурного та алгоритмічного проєктування.*
