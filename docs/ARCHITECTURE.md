# Архитектура и наблюдаемость

## Рекомендуемая форма системы

Начать с модульного монолита: React-клиент, FastAPI-backend, PostgreSQL и отдельный worker только для фоновых интеграционных операций. Это проще разворачивать на собственном сервере и надёжнее микросервисов при ещё не подтверждённой нагрузке.

```text
Bitrix24 iframe / browser
          |
       HTTPS
          v
   Caddy or Nginx
      |         |
   React     FastAPI -------- Bitrix24 REST API
                 |
          PostgreSQL
                 |
        outbox / job queue
                 |
          Redis + worker       (после подтверждения необходимости)
                 |
       S3-compatible storage   (вложения)
```

## Модули backend

- identity & access;
- projects & taxonomies;
- intake;
- triage & duplicates;
- issue lifecycle;
- assignments & SLA;
- Bitrix24 integration;
- analytics read models;
- notifications;
- audit trail;
- administration.

Между модулями — явные сервисные интерфейсы и доменные события. Внешние действия выполняются после фиксации транзакции через transactional outbox, чтобы проблема не потерялась при сбое Bitrix24.

## Данные

Основные таблицы: `projects`, `components`, `control_points`, `reports`, `issues`, `issue_reports`, `issue_relations`, `classifications`, `assignments`, `issue_transitions`, `bitrix_tasks`, `attachments`, `comments`, `audit_events`, `outbox_events`.

Для аналитики сначала использовать SQL-представления/материализованные представления и агрегаты PostgreSQL. Отдельное аналитическое хранилище добавлять только после измерения объёма и задержек.

## Наблюдаемость: что брать

### Берём с первого production-релиза

- **OpenTelemetry SDK + Collector** — единый формат traces/metrics/log correlation;
- **Prometheus** — технические метрики приложения и инфраструктуры;
- **Grafana** — технические дашборды и алерты;
- **Loki + Grafana Alloy/OTel Collector** — структурированные логи;
- **Sentry** или self-hosted GlitchTip — ошибки frontend/backend с релизами;
- health/readiness endpoints и алерты на доступность, ошибки, latency и очередь.

### Добавляем по необходимости

- **Tempo** — распределённые трассировки, когда появляются worker и сложные внешние цепочки;
- **Redis + Celery** — фоновые задачи, повторы, импорты и уведомления;
- **Elasticsearch/OpenSearch** — только если PostgreSQL FTS/GIN не покрывает поиск или объём логов оправдывает отдельный кластер;
- **Logstash** — только для сложного ETL из множества несовместимых источников. Для приложения с OTel он избыточен;
- **Kibana** — только вместе с Elasticsearch/OpenSearch; продуктовые графики в ней не строить.

## Разделение аналитики

- продуктовые графики для сотрудников — собственный UI и API над PostgreSQL;
- техническое здоровье — Grafana/Prometheus/Loki/Tempo;
- ошибки разработчиков — Sentry/GlitchTip;
- события аудита — PostgreSQL с ограниченным доступом и политикой хранения.

Нельзя напрямую отдавать сотрудникам Grafana/Kibana как интерфейс продукта: это раскрывает технические детали, усложняет права и делает UX зависимым от инструмента эксплуатации.

## Безопасность

- токены Bitrix24 только на backend, шифрование at rest и ротация;
- server-side RBAC с проверкой проекта/подразделения;
- signed URLs или защищённая прокси-выдача вложений;
- CSRF/session hardening с учётом iframe Bitrix24;
- аудит назначения ответственности, объединения, экспорта и изменения ролей;
- редактирование событий аудита запрещено обычным пользователям;
- политика хранения и удаления персональных данных определяется до запуска персональных графиков.

