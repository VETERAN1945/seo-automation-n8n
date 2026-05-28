# SEO Automation Workflow (n8n)

## Описание

Автоматизация на базе n8n, которая анализирует выдачу Google по заданному ключевому слову, скрейпит сайты конкурентов и генерирует оптимизированную мету (H1, Meta Title, Meta Description) с помощью AI.

## Архитектура решения

### Общая схема
### Узлы workflow:

1. **Get row(s) in sheet** — читает входные данные (Keyword, GEO, Language) из Google Sheets
2. **HTTP Request (SerpApi)** — выполняет поиск в Google с учётом GEO, получает TOP-10 результатов
3. **Code in JavaScript** — фильтрует TOP-3 аффилиат-сайта, исключает google/youtube/wikipedia
4. **Split Out** — разбивает массив на отдельные items для параллельной обработки
5. **HTTP Request1** — скрейпит каждый сайт конкурента
6. **Code in JavaScript1** — парсит HTML, извлекает H1, Meta Title, Meta Description
7. **Basic LLM Chain (Groq)** — генерирует оптимизированную мету через llama-3.3-70b
8. **Code in JavaScript2** — парсит JSON от AI, группирует данные по keyword
9. **Create a document** — создаёт Google Doc с именем `{Keyword}-{GEO}`
10. **Update a document** — заполняет документ данными конкурентов и SEO контентом
11. **HTTP Request3** — применяет Heading 1 к заголовку документа
12. **HTTP Request2** — выставляет права Commenter на документ
13. **Update row in sheet** — записывает ссылку на документ в колонку Result

## Логика парсинга

- Поиск выполняется через **SerpApi** с параметрами `q=Keyword` и `gl=GEO`
- Из TOP-10 выбираются первые 3 сайта, не входящие в чёрный список (google, youtube, wikipedia, apple, trustpilot)
- Для каждого сайта выполняется HTTP запрос и парсинг HTML для извлечения H1, Meta Title, Meta Description
- При ошибке парсинга фиксируется сообщение "Ошибка парсинга" и обработка продолжается

## Логика генерации (AI)

- Модель: **llama-3.3-70b-versatile** через Groq API (бесплатно)
- Промпт включает данные конкурентов и строгие правила по ТЗ:
  - H1 и Meta Title начинаются с Keyword
  - Запрещены эмодзи и стоп-слова
  - Title: 40-60 символов
  - Description: < 160 символов
- Генерация на языке, указанном в колонке Language

## Инструкция по запуску

### Требования

- n8n (self-hosted или cloud)
- Аккаунты и API ключи:
  - **SerpApi** — для поиска Google (serpapi.com)
  - **Groq** — для AI генерации (console.groq.com)
  - **Google Account** — для Sheets, Docs, Drive

### Настройка

1. Импортируй JSON файл workflow в n8n
2. Настрой credentials:
   - `Google Sheets account` — OAuth2 для Google Sheets
   - `Google Docs account` — OAuth2 для Google Docs
   - `Google Drive account` — OAuth2 для Google Drive
   - `Groq account` — API ключ из console.groq.com
3. В узле **HTTP Request (SerpApi)** укажи свой API ключ SerpApi
4. В узле **Get row(s) in sheet** укажи URL своей Google Таблицы

### Запуск

1. Открой Google Таблицу, заполни колонки:
   - `Keyword` — ключевое слово
   - `GEO` — ISO-код страны (FR, IN, IE)
   - `Language` — язык генерации (fr, en, es)
2. В n8n нажми **Execute Workflow**
3. После выполнения в колонке `Result` появятся ссылки на Google Docs

### Структура входной таблицы

| Keyword | GEO | Language | Result |
|---|---|---|---|
| casino en ligne | FR | fr | (автозаполнение) |
| 1win | IN | en | (автозаполнение) |
| novibet | IE | en | (автозаполнение) |

### Структура выходного документа

Каждый Google Doc содержит:
- **Заголовок** `Analysis for [Keyword] - [GEO]` (Heading 1)
- **Блок Competitor Reports** — данные 3 конкурентов (URL, позиция, Title, Meta)
- **Блок Optimized SEO Content** — H1, Meta Title, Meta Description от AI

## Доступ к результатам

- Все созданные документы открыты по ссылке с правами **Commenter**
- Ссылки записываются в колонку **Result** исходной таблицы
