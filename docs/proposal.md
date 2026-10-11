# Технічна специфікація кваліфікаційного проєкту (Thesis Proposal)

---

## 1. Попередня робоча назва проєкту

* **Українською мовою:** Розподілена система автоматизованого вилучення, семантичного аналізу та контекстного експорту лексичних одиниць із потокового медіаконтенту.
* **Англійською мовою (English Title):** *Distributed Pipeline for Automated Lexical Unit Extraction, Semantic Disambiguation, and Contextual Flashcard Generation from Streaming Media*.
* **Спеціальність:** F2 Інженерія програмного забезпечення (Software Engineering).
* **Виконавець:** Пастух Андрій Юрійович.
* **Науковий керівник:** Янушевський Сергій Володимирович.
* **Інженерне спрямування:** Перетин напрямів *«Розподілені системи та асинхронні конвеєри обробки даних»* і *«Інженерія даних, комп'ютерна лінгвістика (Applied NLP) та ШІ-екосистеми»*.

---

## 2. Актуальність та проблематика

### 2.1. Прикладна та бізнес-проблематика
Засвоєння іноземної мови на основі автентичних матеріалів (технічні відеодоповіді, документація, спеціалізовані статті, подкасти) супроводжується трьома системними бар'єрами:
1. **Високі транзакційні витрати часу на ручну обробку лексики:** Пошук незнайомих лексичних одиниць, вилучення речення з вихідного контексту, визначення релевантного значення слова та ручне формування карток для систем інтервального повторення займають до 75–80% навчального часу користувача.
2. **Невизначеність лексичної складності вхідного матеріалу:** Відсутність попередньої кількісної оцінки частки незнайомої лексики призводить або до когнітивного перевантаження (коли частка нових слів перевищує 20–25%), або до зниження ефективності навчання (коли частка нових слів становить менше ніж 2%).
3. **Втрата семантичного контексту та полісемія:** Стандартні засоби перекладу оперують ізольованими словниковими значеннями, ігноруючи багатозначність слів (полісемію), конверсію частин мови та стійкі багатослівні вирази (*Multi-Word Expressions*, зокрема фразові дієслова).

### 2.2. Інженерна складність та обмеження прямих викликів великих мовних моделей (LLM)
Пряма передача неструктурованих транскриптів медіаконтенту до комерційних великих мовних моделей не вирішує проблему в промислових умовах через такі фактори:
* **Обчислювальна та економічна неефективність (Token Cost & Latency):** Транскрипт 40-хвилинного відеоматеріалу містить 6 000–8 000 слів. Обробка повного масиву тексту через зовнішні LLM API створює високу затримку відповіді (20–35 секунд) та нелінійне зростання вартості інфраструктури при масштабуванні кількості користувачів.
* **Деградація уваги на довгому контексті (Lost in the Middle):** При аналізі великих текстових масивів генеративні моделі схильні пропускати лексичні одиниці в середині документа або генерувати неіснуючі в першоджерелі цитати.
* **Відсутність детермінованого стану знань користувача (Statelessness):** Зовнішня мовна модель не володіє інформацією про тисячі вже засвоєних користувачем лексичних значень, а передача повного профілю користувача у кожному запиті є архітектурно недоцільною через обмеження контекстного вікна.

**Інженерне рішення:** Побудова гібридного розподіленого конвеєра обробки даних, у якому локальний детермінований NLP-рівень та швидкий типізований класифікатор відсікають до 95% відомої або нерелевантної лексики за мілісекунди, делегуючи великій мовній моделі виключно пакетну семантичну обробку відібраних топ-$N$ кандидатів.

---

## 3. Сформована ідея продукту (Архітектурна концепція системи)

Програмний продукт проєктується як **розподілена асинхронна бекенд-платформа за архітектурним стилем Modular Monolith із виділеним пулом фонових воркерів (Asynchronous Task Processing Pipeline)**.

Система складається з таких архітектурних рівнів:
1. **Шлюз прикладного програмування (API Gateway Layer):** Асинхронний REST API на базі FastAPI, що забезпечує автентифікацію (JWT), обмеження частоти запитів (Rate Limiting), строгу валідацію вхідних контрактів (Pydantic v2) та неблокуючу диспетчеризацію обчислювальних задач.
2. **Шар брокера повідомлень та сесійного кешування (Message Broker & Session Cache):** Використання Redis 7 та менеджера черг ARQ із розділенням задач на стандартну чергу (`default_queue` для текстових джерел і субтитрів) та ізольовану ресурсоємну чергу (`heavy_transcribe_queue` із обмеженням паралелізму для ASR-транскрибації аудіофайлів).
3. **Гібридний конвеєр обробки природної мови (Hybrid NLP & AI Pipeline):** Поєднання локального морфологічного аналізу (`spaCy` з лінивим завантаженням мовних моделей у пам'ять воркера), детермінованого SQL-порівняння зі станом знань користувача, типізованого класифікатора рішень System-1 (**Jev / TypeSafe AI**) та генеративного оцінювача System-2 (**LLM із контрактами JSON Schema / Structured Outputs**).
4. **Шар персистентності та файлових адаптерів (Persistence & Interoperability Layer):** Реляційне сховище PostgreSQL 16 (із розділенням сутностей лем `global_lemmas` та конкретних семантичних значень `lexical_senses` для розв'язання проблеми полісемії) та набір експортних адаптерів для генерації бінарних і текстових пакетів карток (`.apkg`, `.tsv`, `.csv`, `.md`).

```text
[Client Layer: Web UI / Telegram Bot]
                 │ HTTPS (REST API)
                 ▼
      ┌──────────────────────┐
      │ FastAPI Gateway API  │
      └──────────┬───────────┘
                 │ Job Dispatch / Session State
                 ▼
      ┌──────────────────────┐
      │   Redis 7 + ARQ      │ (default_queue / heavy_transcribe_queue)
      └──────────┬───────────┘
                 │
                 ▼
      ┌────────────────────────────────────────────────────────┐
      │               ARQ Asynchronous Workers                 │
      │  1. Ingestion Engine (yt-dlp / trafilatura / whisper)  │
      │  2. Content Difficulty Radar (Coverage & i+1 Model)    │
      │  3. Deterministic NLP & MWE Matcher (spaCy Lazy Load)  │
      │  4. System-1 Gate (Jev: Noise Triage & Sense Match)    │
      │  5. System-2 Evaluator (LLM Structured Outputs)        │
      │  6. Format Adapters (Anki .apkg, Quizlet, Obsidian)    │
      └──────────┬─────────────────────────────────┬───────────┘
                 │                                 │
                 ▼                                 ▼
      ┌──────────────────────┐          ┌──────────────────────┐
      │    PostgreSQL 16     │          │   MinIO / S3 Store   │
      └──────────────────────┘          └──────────────────────┘
```

---

## 4. Детальний опис функціоналу (Core Features)

У програмному коді системи підлягають обов'язковій реалізації такі **6 ключових функціональних модулів**:

1. **Багатоканальний модуль інгестії та нормалізації джерел (Omnichannel Ingestion Engine):**
   * Автоматизоване вилучення субтитрів із потокового відео YouTube (`.vtt`, `.srt`), очищення основного тексту веб-публікацій та технічної документації від HTML-розмітки (`trafilatura`), обробка текстових документів і фонова ASR-транскрибація аудіофайлів через модель `faster-whisper`.
   * Ідемпотентне кешування публічних джерел за криптографічним хешем (`SHA-256`) для уникнення повторних мережевих запитів та обчислень.

2. **Аналізатор лексичної складності контенту (Content Difficulty Radar):**
   * Кількісна оцінка придатності матеріалу до вивчення згідно з моделлю *Comprehensible Input ($i+1$)*.
   * Розрахунок метрики покриття словника користувача (*Vocabulary Coverage Rate*) із урахуванням зважених коефіцієнтів стану знань (`KNOWN = 1.0`, `LEARNING = 0.5`) та побудова розподілу незнайомої лексики за рівнями шкали CEFR (A1–C2).
   * Збереження результатів первинного розбору в сесійному кеші Redis (`SourceAnalysisSession`, TTL = 1 година) для миттєвого переходу до етапу вилучення слів.

3. **Детермінований багатомовний NLP-конвеєр із детекцією багатослівних одиниць (NLP & MWE Pipeline):**
   * Підтримка трьох цільових мов джерела (англійська `en`, німецька `de`, іспанська `es`) із механізмом лінивого завантаження моделей `spaCy` (Lazy Loading з LRU-кешуванням) для оптимізації споживання оперативної пам'яті.
   * Токенізація, лематизація, визначення частин мови (POS-tagging) та синтаксичне виявлення фразових дієслів і багатослівних виразів (`spaCy Matcher`), що запобігає помилковому розбиттю ідіоматичних конструкцій на елементарні слова рівня A1.

4. **Алгоритмічний скоринг та двоетапна семантична дезамбігуація (Hybrid Scoring & WSD Engine):**
   * Ранжування незнайомих лексичних одиниць за багатофакторною математичною функцією корисності:
     $$\text{Score}(w) = w_1 \cdot \text{CEFR\_Match}(w, U_{\text{level}}) + w_2 \cdot \text{TF}_{\text{source}}(w) + w_3 \cdot \text{Freq}_{\text{global}}(w)$$
   * **Етап 1 (System-1 Decision Gate — Jev):** Швидка типізована фільтрація артефактів субтитрів і власних назв (`is_valid_word`), вибір репрезентативного контекстного речення та семантична прив'язка значення слова.
   * **Етап 2 (System-2 Evaluator — LLM Structured Outputs):** Пакетна генерація точних контекстних перекладів цільовою рідною мовою користувача (зокрема українською), дефініцій та динамічна оцінка рівня CEFR для позасловникових технічних термінів і неологізмів (`origin = 'DYNAMIC_EVALUATED'`) із суворою валідацією за схемами Pydantic v2.

5. **Підсистема інтерактивного тріажу та управління життєвим циклом знань (Interactive Triage & Vocabulary State Machine):**
   * Надання API для попереднього перегляду відібраних кандидатів перед фінальною генерацією колоди.
   * Керування станами лексичних значень у профілі користувача (`NEW` $\rightarrow$ `LEARNING` $\rightarrow$ `KNOWN` або `IGNORED`) із можливістю миттєвої заміни відомих слів на наступні за рейтингом кандидати з кешованої сесії розбору.

6. **Мультиплатформний шар файлових адаптерів експорту та імпорту (Format Adapters Layer):**
   * Реалізація патерна проєктування *Adapter* для формування готових навчальних пакетів: програмна генерація бінарних архівів **Anki (`.apkg`)** з внутрішньою базою SQLite та HTML/CSS-розміткою карток (`genanki`), а також валідованих текстових форматів для **Quizlet (`.tsv`)**, **Brainscape (`.csv`)**, **RemNote** та **Obsidian (`.md`)**.
   * Підтримка зворотного імпорту користувацьких списків слів для первинного наповнення профілю знань (Cold Start Onboarding).

---

## 5. Попередній інструментарій та технологічний стек

Технологічний стек системи структуровано за функціональними категоріями відповідно до вимог спеціальності F2 «Інженерія програмного забезпечення»:

* **Мови програмування:**
  * Python 3.12 (основна мова розробки серверної логіки, асинхронних воркерів та NLP-конвеєра).
  * SQL (декларативні запити, індексація та агрегація даних у PostgreSQL).

* **Архітектурні стилі та патерни проєктування:**
  * *Modular Monolith* та *Clean / Layered Architecture* (ізоляція доменної логіки від інфраструктурних адаптерів).
  * *Asynchronous Task Queue / Producer-Consumer* (неблокуюча обробка медіаконтенту через пул фонових воркерів).
  * *Strategy / Provider Pattern* (уніфікація каналів отримання контенту та можливість резервного перемикання AI-провайдерів — Graceful Degradation).
  * *Adapter Pattern* (конвертація внутрішньої доменної моделі карток у специфічні бінарні та текстові формати 5 зовнішніх платформ).
  * *State Machine* (керування життєвим циклом засвоєння лексичних значень користувачем).

* **Фреймворки та спеціалізовані бібліотеки:**
  * *Веб-рівень та валідація:* FastAPI, Pydantic v2, Uvicorn, HTTPX (асинхронний мережевий клієнт).
  * *ORM та міграції:* SQLAlchemy 2.0 (AsyncIO Engine), Asyncpg, Alembic.
  * *Асинхронні черги:* ARQ (Asyncio Redis Queue).
  * *Комп'ютерна лінгвістика та обробка медіа:* `spaCy` (`en_core_web_md`, `de_core_news_md`, `es_core_news_md`), `faster-whisper` (локальна ASR-транскрибація), `trafilatura` (екстракція веб-контенту), `yt-dlp` / `youtube-transcript-api` (парсинг субтитрів), `genanki` (генерація пакетів `.apkg`).
  * *ШІ-інтеграція:* Jev (TypeSafe AI SDK для швидкої типізованої класифікації), OpenAI / Gemini API (Structured Outputs за контрактами JSON Schema).

* **Бази даних, кешування та сховища даних:**
  * *Реляційна СУБД:* PostgreSQL 16 (зберігання глобальних лем, семантичних синсетів/значень, матриці станів користувачів та кешованих транскриптів).
  * *In-Memory СУБД та брокер:* Redis 7 (черги задач ARQ, сесійний кеш `SourceAnalysisSession`, Rate Limiting).
  * *Об'єктне сховище:* MinIO / S3-compatible Object Storage (зберігання згенерованих файлових пакетів експорту).

* **DevOps-інструменти, безпека та інженерія якості (QA & Observability):**
  * *Контейнеризація та оркестрація:* Docker, Docker Compose (ізольоване розгортання API, воркерів, PostgreSQL, Redis та MinIO).
  * *CI/CD та контроль якості коду:* GitHub Actions, Ruff (лінтер і форматер), Mypy (статична перевірка типів).
  * *Автоматизоване та навантажувальне тестування:* Pytest, Pytest-Asyncio (модульні й інтеграційні тести), Grafana k6 (стрес-тестування пропускної здатності асинхронних черг).

---

## 6. Thesis Proposal Specification (English Summary)

* **Title:** *Distributed Pipeline for Automated Lexical Unit Extraction, Semantic Disambiguation, and Contextual Flashcard Generation from Streaming Media*.
* **Problem Statement:** Language learners consuming authentic media (technical videos, articles, podcasts) spend up to 80% of their study time manually extracting unknown vocabulary, identifying contextual meanings, and formatting flashcards. Direct processing of full media transcripts via commercial LLMs is economically inefficient due to high token consumption, high latency (20–35s per transcript), the *Lost in the Middle* phenomenon on long contexts, and the lack of persistent user vocabulary state.
* **System Architecture:** An asynchronous **Modular Monolith** built with **FastAPI**, **PostgreSQL 16**, and **Redis 7 (ARQ)** background worker pools (`default_queue` and `heavy_transcribe_queue`). The pipeline combines local deterministic NLP (`spaCy` with lazy model loading and Multi-Word Expression detection) with a two-stage AI contour: a low-latency System-1 decision gate (**Jev / TypeSafe AI**) for noise filtering and sense selection, and a System-2 **LLM** with **Pydantic v2 Structured Outputs** for batch contextual translation and dynamic CEFR evaluation of out-of-vocabulary terms.
* **Core Features:**
  1. Omnichannel media and text ingestion (`yt-dlp`, `trafilatura`, `faster-whisper`) with SHA-256 idempotent source caching.
  2. Content Difficulty Radar computing vocabulary coverage and *Comprehensible Input ($i+1$)* suitability scores.
  3. Multilingual deterministic NLP pipeline (`en`, `de`, `es`) with syntactic phrasal verb (MWE) detection.
  4. Multi-factor mathematical lexical scoring and two-stage Word Sense Disambiguation (WSD).
  5. Interactive vocabulary triage and weighted knowledge state machine (`KNOWN`, `LEARNING`, `IGNORED`) linked to disambiguated `lexical_senses`.
  6. Multi-platform export and import file adapters (`genanki` `.apkg` binary packages, Quizlet `.tsv`, Brainscape `.csv`, RemNote, and Obsidian `.md`).
