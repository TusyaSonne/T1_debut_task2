
<img width="1454" height="811" alt="Микросервис-библиотека drawio (2)" src="https://github.com/user-attachments/assets/5a704037-975c-4d8b-9bbc-66f2d07c0226" />

# ⚙️ Общая архитектура
## Микросервисы
1. Gateway API – входная точка для всех запросов от фронтенда и мобильных приложений. Запрашивает регистрацию у Auth Service и получая JWT, проксирует запросы клиента на другие сервисы.
2. Auth Service – аутентификация и авторизация (JWT + OAuth2).
3. User Service – управление пользователями, учетной записью.
4. Book Service – каталог книг: хранение, поиск, фильтрация.
5. Lending Service – учет выдачи и возврата книг, формирование графика выдачи.
6. Delivery Service – логистика: доставка и возврат книг.
7. Payment Service – безналичная оплата штрафов: просрочки даты возврата, порчи (или утери) книг.
8. Notification Service – отправка уведомлений (e-mail, push, SMS).

## Хранилище данных
| Сервис               | Хранилище                  | Подробности                                                   |
| -------------------- | -------------------------- | ------------------------------------------------------------- |
| User Service         | PostgreSQL                 | Пользователи                                                  |
| Book Service         | PostgreSQL / Elasticsearch | Основные данные в PostgreSQL, поиск по книгам — Elasticsearch |
| Lending Service      | PostgreSQL                 | История выдач                                                 |
| Notification Service | Redis                      | Очереди и история уведомлений                                 |
| Delivery Service     | PostgreSQL                 | Доставка и возвращение книг                                   |
| Payment Service      | PostgreSQL                 | Выставление штрафов, транзакции                               |

---
## Диаграмма связи
| Отправитель     | Получатель      | Тип     | Событие / Операция                     |
| --------------- | --------------- | ------- | -------------------------------------- |
| API Gateway     | Все сервисы (кроме Notification)    | REST    | Проксирование запросов (с имеющимся JWT)                |
| API Gateway     | Auth Service   | REST    | Запрос на авторизацию пользователя                |
| Auth Service    | User Service    | REST    | Получение данных пользователя          |
| User Service    | Payment Service | REST    | Проверка блокировки (за неуплаченные штрафы)                    |
| Lending Service | Book Catalog    | REST    | Проверка доступности книги и обновление статуса доступности            |
| Lending Service | Delivery Service | Kafka   | `DeliveryRequestedEvent`, `ReturnRequestedEvent` - запрос на выполнение доставки и возврата             |
| Lending Service | Notification    | Kafka   | `BookIssuedEvent`, `BookReturnedEvent` - книга выдана, книга принята |
| Lending Service | Payment Service | Kafka   | `FineIssuedEvent` - выставление штрафа пользователю (за порчу, просрочку, утерю)                     |
| Payment Service | User Service    | Kafka   | `PaymentSuccessEvent` - успешная оплата штрафа               |
| Payment Service | Notification    | Kafka   | `PaymentResultEvent` - результат оплаты                  |
| Delivery Service | Lending Service    | Kafka   | `BookReturnedEvent` - книга доставлена          |
| Delivery Service | Notification    | Kafka   | `DeliveryStatusChangedEvent` - смена статуса доставки           |


# 📊 Набор метрик для системы цифровизации библиотек

## 1. Общесистемные метрики (инфраструктурные)

### API-Gateway
- `http_requests_total{status="5xx"}` — количество ошибок на шлюзе
- `request_duration_seconds{service="api-gateway"}` — время обработки запроса
- `jwt_validation_failures_total` — число невалидных JWT

### Kafka (взаимодействие между сервисами)
- `kafka_server_BrokerTopicMetrics_MessagesInPerSec` — входящие сообщения в Kafka
- `kafka_server_BrokerTopicMetrics_BytesInPerSec` — объем входящего трафика
- `kafka_consumer_lag{group="consumer-group"}` — лаг консюмеров
- `dead_letter_queue_messages_total` — сообщения, попавшие в DLQ (Dead Letter Queue)

##  2. Метрики микросервисов

### User Service
- `user_created_total` — общее количество зарегистрированных пользователей
- `user_lookup_duration_seconds` — среднее время поиска пользователя
- `failed_user_fetch_total` — количество ошибок при получении пользователя

### Book Service (каталог книг)
- `books_catalog_size` — общее количество книг в базе
- `book_search_requests_total` — количество запросов на поиск
- `book_update_duration_seconds` — среднее время обновления данных книги

### Lending Service (выдача и возврат книг)
- `books_issued_total` — количество выданных книг
- `books_returned_total` — количество возвращённых книг
- `overdue_books_total` — количество просроченных книг
- `lending_request_duration_seconds` — время обработки запроса на выдачу

### Delivery Service (доставка)
- `delivery_requests_total` — общее количество запросов на доставку
- `delivery_success_total` — успешные доставки
- `delivery_failures_total` — сбои доставки
- `delivery_duration_seconds` — среднее время доставки

### Payment Service
- `payments_processed_total` — общее число обработанных платежей
- `payment_failures_total` — количество неудачных платежей
- `average_payment_amount` — средний чек
- `fine_payments_total` — количество оплат штрафов

### Notification Service
- `notifications_sent_total` — отправленные уведомления
- `notification_failures_total` — неудачные попытки отправки
- `notification_latency_seconds` — задержка между событием и уведомлением

## 3. Метрики безопасности (Auth Service)

- `auth_login_attempts_total` — общее число попыток входа
- `auth_login_failures_total` — неудачные попытки входа
- `auth_token_issued_total` — количество выданных JWT
- `invalid_jwt_requests_total` — количество запросов с просроченными/некорректными токенами

## 4. Метрики нагрузки на систему

- `service_instance_count{service=...}` — количество активных экземпляров сервиса
- `cpu_usage{service=...}` — загрузка процессора
- `memory_usage{service=...}` — использование памяти
- `request_rate{service=...}` — количество входящих запросов в секунду

## 5. Метрики ошибок

- `exceptions_total{exception=...}` — общее количество исключений
- `http_response_errors_total{status="4xx|5xx"}` — HTTP-ошибки
- `timeout_errors_total` — количество таймаутов запросов

## 6. Пользовательские бизнес-метрики

- `active_users_total` — число активных пользователей в день/час
- `top_requested_books` — самые популярные книги
- `avg_book_issue_duration` — среднее время от заказа до получения
- `repeat_borrow_rate` — доля пользователей, которые возвращаются
- `fine_rate` — отношение количества штрафов к числу выдач

## 🛠 Инструменты мониторинга и трейсинга

| Инструмент     | Назначение                                   |
|----------------|----------------------------------------------|
| **Prometheus** | Сбор и агрегация метрик                      |
| **Grafana**    | Визуализация: графики, алерты, дашборды      |
| **Jaeger**     | Трассировка межсервисных вызовов             |
