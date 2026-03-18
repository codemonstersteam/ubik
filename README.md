# Ubik — платформа обмена знаниями

Открытая децентрализованная платформа для создания и распространения образовательного контента в современных форматах. Минималистичный интерфейс в эстетике терминалов будущего.

## Идея

Знания распространяются быстрее, когда инфраструктура не принадлежит одному центру. Ubik — это сеть независимых серверов (нод), которые обмениваются контентом между собой по открытому протоколу. Каждый может поднять свою ноду, каждая нода видит контент остальных.

Не социальная сеть. Платформа для структурированной передачи знаний в трёх форматах, заточенных под мобильное потребление.

## Форматы контента

**Рилсы** — вертикальное видео до 3 минут. Концентрированное объяснение одной идеи, одного приёма, одного инструмента. Без воды, без вступлений. Три минуты — достаточно для демонстрации, недостаточно для размазывания.

**Голосовые заметки** — аудио до 5 минут. Разбор, комментарий, рассуждение вслух. Формат для ситуаций, когда экран недоступен: дорога, прогулка, тренировка. Потребляется как подкаст-фрагмент.

**Слайды** — текст, разбитый на вертикальные карточки (до 7 штук). Листаются свайпом. Одна мысль — одна карточка. Структура вынуждает автора сжимать материал до сути. Формат для пошаговых инструкций, чеклистов, разборов, конспектов.

Все три формата — вертикальные, листаются одним жестом, потребляются за минуты.

## Децентрализация

```
    Нода A               Нода B               Нода C
 ┌──────────┐        ┌──────────┐        ┌──────────┐
 │ Go Server│◄──────►│ Go Server│◄──────►│ Go Server│
 │ Users    │  sync  │ Users    │  sync  │ Users    │
 │ Content  │        │ Content  │        │ Content  │
 └──────────┘        └──────────┘        └──────────┘
      ▲                    ▲                    ▲
      │                    │                    │
   Flutter              Flutter              Flutter
   clients              clients              clients
```

Каждая нода — самостоятельный Go-сервер со своей базой пользователей и контента. Ноды обмениваются контентом по протоколу синхронизации (вдохновлён ActivityPub / AT Protocol). Пользователь регистрируется на одной ноде, но видит контент со всех. Handle включает домен ноды: `@neo@learn.ubik.net`.

Что это даёт: ни одна компания не контролирует ленту; нода упала — остальные работают; цензура одной ноды не затрагивает сеть; каждое сообщество может настроить свою ноду под себя.

## Технологический стек

| Слой | Технология | Почему |
|------|-----------|--------|
| Frontend | Flutter | Кроссплатформа: iOS, Android, Web из одной кодовой базы |
| Backend | Go | Скорость, один бинарник, легко поднять ноду |
| Авторизация | Passkeys (WebAuthn/FIDO2) | Без паролей, фишинг невозможен конструктивно |
| Токены | JWT (Ed25519) | Stateless-валидация на любой ноде |
| Синхронизация | Открытый протокол (federation) | Ноды обмениваются контентом без центрального сервера |

## Архитектура ноды (целевая)

```
Flutter Client → API Gateway → Auth Domain
                     │              │
                     │         ┌────┴────────────────┐
                     │         │  Auth Service        │
                     │         │  (WebAuthn RP)       │
                     │         ├──────┬───────┬───────┤
                     │         │Token │Cred   │Challenge│
                     │         │Issuer│Store  │Manager  │
                     │         └──┬───┴───┬───┴───┬────┘
                     │            │       │       │
                     │         Signing  Postgres  Redis
                     │         Keys
                     │
                     ├──── JWT ──── Content Service (рилсы, голос, слайды)
                     ├──── JWT ──── Feed Service (поток контента, рекомендации)
                     ├──── JWT ──── Profile Service
                     ├──── JWT ──── Media Service (хранение видео/аудио)
                     └──── JWT ──── Federation Service (синхронизация с другими нодами)
```

Federation Service — ключевой компонент для децентрализации. Он получает контент от других нод, отдаёт локальный контент наружу и разрешает федеративные handle.

---

## Шаг 1: Passkey Demo

Минимальный сервис для демонстрации полного цикла passkey-авторизации. Один бинарник Go, одна база SQLite, три экрана Flutter. Первый кирпич будущей ноды.

### Что умеет демо

- Регистрация пользователя по handle + биометрия
- Вход по handle + биометрия
- Выход (инвалидация сессии)
- Проверка текущей сессии

### Архитектура демо

```
┌─────────────────────────────────┐
│  Flutter App                    │
│  3 экрана: register/login/home  │
└──────────────┬──────────────────┘
               │ HTTPS / JSON
┌──────────────▼──────────────────┐
│  Go Passkey Service             │
│  ┌────────────┐ ┌─────────────┐ │
│  │  WebAuthn   │ │  Session    │ │
│  │  module     │ │  module     │ │
│  └──────┬─────┘ └──────┬──────┘ │
└─────────┼──────────────┼────────┘
          │              │
   ┌──────▼──────────────▼──────┐
   │  SQLite                    │
   │  users | credentials |     │
   │  sessions                  │
   └────────────────────────────┘
```

### REST API

| Метод | Ресурс | Действие |
|-------|--------|----------|
| `POST` | `/registrations` | Начать регистрацию — сервер возвращает challenge и options |
| `PUT` | `/registrations` | Завершить регистрацию — клиент отправляет attestation, сервер выдаёт JWT |
| `POST` | `/sessions` | Начать вход — сервер возвращает challenge и options |
| `PUT` | `/sessions` | Завершить вход — клиент отправляет assertion, сервер выдаёт JWT |
| `DELETE` | `/sessions/current` | Выход — инвалидация refresh token |
| `GET` | `/users/me` | Текущий пользователь — требует валидный access token |

### Как работает Passkey

Passkey — замена паролей на криптографию с открытым ключом.

**Регистрация:** устройство генерирует пару ключей. Приватный ключ остаётся в защищённом чипе (Secure Enclave / TPM). Публичный ключ отправляется серверу. Сервер никогда не видит приватный ключ.

**Вход:** сервер отправляет случайный challenge (32 байта). Устройство просит подтвердить личность (отпечаток / Face ID) — это разблокирует приватный ключ локально. Приватный ключ подписывает challenge. Сервер проверяет подпись публичным ключом.

**Challenge** — одноразовое случайное число с TTL 5 минут. Защита от replay-атаки: перехваченная подпись бесполезна, потому что challenge каждый раз новый.

**Привязка к домену:** passkey привязан к origin. Фишинговый сайт не сможет запросить ключ — устройство его просто не найдёт.

### Стек демо

| Компонент | Технология | Пакет |
|-----------|-----------|-------|
| WebAuthn Relying Party | Go | `github.com/go-webauthn/webauthn` |
| JWT | Go | `github.com/golang-jwt/jwt/v5` |
| HTTP Router | Go | `github.com/go-chi/chi/v5` |
| База данных | SQLite | `github.com/mattn/go-sqlite3` |
| Passkey клиент | Flutter | `passkeys` (pub.dev) |

### Модель данных (SQLite)

```sql
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    handle TEXT UNIQUE NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE credentials (
    id BLOB PRIMARY KEY,
    user_id TEXT REFERENCES users(id) ON DELETE CASCADE,
    public_key BLOB NOT NULL,
    sign_count INTEGER DEFAULT 0,
    transports TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_used_at DATETIME
);

CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    user_id TEXT REFERENCES users(id) ON DELETE CASCADE,
    refresh_token_hash BLOB UNIQUE NOT NULL,
    expires_at DATETIME NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE challenges (
    challenge BLOB PRIMARY KEY,
    user_id TEXT,
    type TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    expires_at DATETIME NOT NULL
);
```

### Структура проекта

```
passkey-demo/
├── server/
│   ├── main.go
│   ├── webauthn.go
│   ├── session.go
│   ├── store.go
│   └── go.mod
├── app/
│   └── lib/
│       ├── main.dart
│       ├── api.dart
│       ├── passkey.dart
│       └── screens/
│           ├── register.dart
│           ├── login.dart
│           └── home.dart
├── README.md
└── passkey.db
```

### Запуск

```bash
# Backend
cd server
go run .

# Frontend
cd app
flutter run
```
