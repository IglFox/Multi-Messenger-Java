# -мессенджер
---

## О проекте

**Мультимессенджер-агрегатор** — веб-приложение, объединяющее чаты из нескольких
внешних мессенджеров в едином интерфейсе.

**Возможности:**

- единый список чатов из всех подключённых мессенджеров;
- отправка/получение сообщений без перехода в родные клиенты;
- подключение аккаунтов мессенджеров через Bot API (Telegram, Discord);
- пересылка сообщений между мессенджерами (Telegram → Discord и обратно);
- поиск по всем чатам сразу;
- real-time доставка через WebSocket;
- вложения (текст, файлы).

Архитектура коннекторов (`MessengerConnector` + `ConnectorRegistry`)
позволяет добавить любой мессенджер без изменения ядра приложения.

## Стек

| Слой | Технологии |
|---|---|
| Backend | Java 17, Spring Boot 3.2, Spring MVC, Security, Data JPA, WebSocket (STOMP) |
| Интеграции | Telegram Bot API (long polling), JDA для Discord Bot API |
| БД | PostgreSQL 16, Flyway |
| Frontend | HTML/CSS/JS, SockJS + STOMP |
| Сборка | Maven |

## Архитектура

```
┌──────────────┐    REST (JSON)     ┌───────────────┐
│   Browser    │◄──────────────────►│  Spring Boot  │
│  (HTML/JS)   │                    │     API       │
│              │◄──── WebSocket ───►│  (STOMP/WS)   │
└──────────────┘                    └───────┬───────┘
                                            │ JPA
                                    ┌───────▼───────┐
                                    │  PostgreSQL   │
                                    └───────────────┘
```

**Разделение транспортов:**

| Задача | Транспорт |
|---|---|
| Аутентификация, CRUD чатов | REST |
| История сообщений, пагинация | REST |
| Отправка сообщения | REST (запись) + WS (рассылка) |
| Загрузка файлов | REST (multipart) |
| Поиск | REST |
| Новое сообщение в реальном времени | WS |
| Статус прочтения | WS |
| «Печатает…», online | WS |

---

## Структура репозитория

```
messenger/
├── backend/
│   ├── pom.xml
│   └── src/main/
│       ├── java/ru/net/messenger/
│       │   ├── config/        # Security, WebSocket, JWT-filter
│       │   ├── domain/        # JPA-сущности
│       │   ├── repo/          # Spring Data репозитории
│       │   ├── dto/           # DTO для REST и WS
│       │   ├── service/       # Бизнес-логика
│       │   ├── web/           # REST-контроллеры
│       │   └── web/ws/        # STOMP-контроллеры
│       └── resources/
│           ├── application.yml
│           └── db/migration/  # Flyway SQL
├── frontend/
│   ├── index.html
│   ├── app.js
│   └── style.css
├── docs/
│   ├── use-case.puml
│   ├── er.puml
│   ├── bpmn/
│   └── report.pdf
└── README.md
```

---

## Быстрый старт

### 1. Поднять PostgreSQL

```bash
docker run -d --name messenger-pg \
  -e POSTGRES_USER=messenger \
  -e POSTGRES_PASSWORD=messenger \
  -e POSTGRES_DB=messenger \
  -p 5432:5432 postgres:16
```

### 2. Запустить backend

```bash
cd backend
./mvnw spring-boot:run
```

Сервер: `http://localhost:8080`. Миграции применятся автоматически.

### 3. Открыть frontend

```bash
cd frontend
python3 -m http.server 5500
```

Открыть `http://localhost:5500`.

---

## HTTP API

Все ответы — `application/json`. Авторизация: `Authorization: Bearer <jwt>`.

### Аутентификация

| Метод | Путь | Тело | Ответ |
|---|---|---|---|
| POST | `/api/auth/register` | `{username, email, password}` | `201 {userId}` |
| POST | `/api/auth/login` | `{login, password}` | `200 {accessToken, userId, username}` |

### Чаты

| Метод | Путь | Тело | Ответ |
|---|---|---|---|
| GET | `/api/chats` | — | `200 [ChatDto]` |
| POST | `/api/chats` | `{type, title?, memberIds:[]}` | `201 ChatDto` |
| GET | `/api/chats/{id}` | — | `200 ChatDto` |
| GET | `/api/chats/{id}/members` | — | `200 [MemberDto]` |
| POST | `/api/chats/{id}/members` | `{userIds:[]}` | `200` |
| DELETE | `/api/chats/{id}/members/{uid}` | — | `204` |

### Сообщения

| Метод | Путь | Тело | Ответ |
|---|---|---|---|
| GET | `/api/chats/{id}/messages?before=&limit=` | — | `200 [MessageDto]` |
| POST | `/api/chats/{id}/messages` | `{body}` | `201 MessageDto` |
| PATCH | `/api/messages/{id}` | `{body}` | `200 MessageDto` |
| DELETE | `/api/messages/{id}` | — | `204` |
| GET | `/api/messages/search?q=` | — | `200 [MessageDto]` |

### Файлы

| Метод | Путь | Тело | Ответ |
|---|---|---|---|
| POST | `/api/files` | `multipart/form-data` | `201 {fileId, url}` |
| GET | `/api/files/{id}` | — | `200 binary` |

### Коды ошибок

| Код | Когда |
|---|---|
| 400 | Невалидные данные |
| 401 | Нет/просрочен токен |
| 403 | Нет прав (не участник чата) |
| 404 | Объект не найден |
| 409 | Конфликт (username/email занят) |
| 413 | Файл больше лимита |

---

## WebSocket (STOMP)

Endpoint: `/ws` (SockJS). Авторизация — JWT в заголовке коннекта.

| Направление | Destination | Payload |
|---|---|---|
| C → S | `/app/chats/{id}/typing` | `{}` |
| C → S | `/app/messages/{id}/read` | `{}` |
| S → C | `/topic/chats/{id}/messages` | `MessageDto` |
| S → C | `/topic/chats/{id}/read` | `{messageId, userId}` |
| S → C | `/topic/chats/{id}/typing` | `{userId, displayName}` |

---

## Схема БД (6 таблиц)

| Таблица | Назначение |
|---|---|
| `users` | Пользователи системы |
| `chats` | Чаты (PRIVATE/GROUP) |
| `chat_members` | Участники чата (M:N) |
| `messages` | Сообщения |
| `attachments` | Вложения к сообщениям |
| `message_reads` | Статусы прочтения (M:N) |

ER-диаграмма — в `docs/er.puml`.

---

## Соответствие требованиям курсовой

| Требование | Где реализовано |
|---|---|
| Область применения | см. раздел «О проекте» |
| Бизнес-сущности и use cases | `docs/use-case.puml` + раздел «Use cases» в отчёте |
| Схема БД ≥ 5 таблиц | 6 таблиц, `V1__init.sql` |
| Веб-интерфейс | `frontend/` |
| HTTP-методы с JSON | раздел «HTTP API» |
| Java backend | `backend/`, Spring Boot |
| ≥ 7 бизнес-процессов | UC-1…UC-14 (см. диаграмму) |
| Код в Git | этот репозиторий |
| Оформление по ГОСТ | `docs/report.pdf` |

---

## План развития

- [ ] E2E-шифрование сообщений
- [ ] Голосовые/видеозвонки (WebRTC)
- [ ] Push-уведомления
- [ ] Мобильное приложение
- [ ] Реактивный фронтенд (Vue/React)