# Workout Backend API

> [English version](README.md)

RESTful бэкенд для приложения по отслеживанию тренировок. Написан на **Node.js / Express**, использует **PostgreSQL** через **Prisma ORM** и защищён **JWT-аутентификацией**.

---

## Содержание

- [Описание проекта](#описание-проекта)
- [Технологический стек](#технологический-стек)
- [Архитектура](#архитектура)
- [Схема базы данных](#схема-базы-данных)
- [Справка по API](#справка-по-api)
  - [Аутентификация](#аутентификация)
  - [Пользователи](#пользователи)
  - [Упражнения](#упражнения)
  - [Логи упражнений](#логи-упражнений)
  - [Тренировки](#тренировки)
  - [Логи тренировок](#логи-тренировок)
- [Запуск проекта](#запуск-проекта)
- [Структура проекта](#структура-проекта)
- [Контакты](#контакты)

---

## Описание проекта

Workout Backend предоставляет мобильным и веб-клиентам следующие возможности:

- **Регистрация и вход** - JWT-аутентификация с хешированием паролей через Argon2.
- **Управление библиотекой упражнений** - создание, редактирование и удаление упражнений с иконками.
- **Создание тренировок** - составление многоразовых планов тренировок из упражнений.
- **Запись тренировочных сессий** - запуск лога тренировки, отслеживание каждого подхода (вес × повторения), отметка подходов и упражнений как выполненных.
- **Отслеживание прогресса** - каждый лог упражнения показывает результаты предыдущего выполнения, что позволяет видеть динамику улучшений.
- **Просмотр статистики** - эндпоинт профиля пользователя агрегирует общее количество тренировок и другие сводные данные.

Приложение следует многоуровневой архитектуре: маршруты → контроллеры → Prisma-клиент → PostgreSQL. Все защищённые маршруты требуют валидного JWT Bearer-токена.

---

## Технологический стек

| Уровень | Технология |
|---|---|
| Среда выполнения | Node.js (ES Modules) |
| Фреймворк | Express.js 4 |
| ORM | Prisma 4 |
| База данных | PostgreSQL |
| Аутентификация | JWT (`jsonwebtoken`) |
| Хеширование паролей | Argon2 |
| HTTP-логирование | Morgan |
| Стиль кода | Prettier + `@trivago/prettier-plugin-sort-imports` |
| Dev-сервер | Nodemon |

---

## Архитектура

```
src/
├── app.js                  # Инициализация Express и цепочка middleware
├── controllers/            # По папке на каждый ресурс, по файлу на каждую операцию
│   ├── auth/               # register, login
│   ├── exercise/           # CRUD каталога упражнений
│   ├── exersice-log/       # создание, обновление, завершение лога упражнения
│   ├── user/               # профиль (со статистикой)
│   ├── workout/            # CRUD планов тренировок
│   └── workout-log/        # создание, обновление, завершение сессии тренировки
├── db/
│   └── prisma.js           # Singleton Prisma-клиент
├── middlewares/
│   ├── auth.js             # JWT protect middleware (прикрепляет req.user)
│   └── error.js            # Обработчик 404 и глобальный обработчик ошибок
├── routes/                 # Файлы Express Router, связанные с контроллерами
└── utils/
    ├── calc-minutes.js     # Оценка длительности тренировки (упражнения × 5 мин)
    ├── generate-token.js   # Подпись JWT на 10 дней
    ├── getUserWithoutPassword.js
    └── userDataWithoutPassword.js
prisma/
├── schema.prisma           # Модели данных и связи
└── migrations/             # История SQL-миграций (авто-генерация)
public/
└── images/exercises/       # Статические файлы иконок упражнений
```

**Цепочка middleware (app.js):**
Morgan → CORS → JSON body parser → статические файлы → маршруты → обработчик 404 → обработчик ошибок

---

## Схема базы данных

Пять моделей, все на PostgreSQL:

### User (Пользователь)
| Поле | Тип | Примечание |
|---|---|---|
| id | Int (PK) | auto-increment |
| email | String | уникальный |
| name | String | |
| password | String | хеш Argon2 |
| images | String[] | изображения профиля |
| createdAt / updatedAt | DateTime | |

Связи: один-ко-многим с `ExerciseLog`, `WorkoutLog`.

### Exercise (Упражнение)
| Поле | Тип | Примечание |
|---|---|---|
| id | Int (PK) | auto-increment |
| name | String | |
| times | Int | количество подходов по умолчанию |
| icon | String | путь к файлу иконки |
| createdAt / updatedAt | DateTime | |

Связи: многие-ко-многим с `Workout`; один-ко-многим с `ExerciseLog`.

### Workout (Тренировка)
| Поле | Тип | Примечание |
|---|---|---|
| id | Int (PK) | auto-increment |
| name | String | |
| createdAt / updatedAt | DateTime | |

Связи: многие-ко-многим с `Exercise`; один-ко-многим с `WorkoutLog`.

### WorkoutLog (Лог тренировки)
| Поле | Тип | Примечание |
|---|---|---|
| id | Int (PK) | auto-increment |
| isCompleted | Boolean | false по умолчанию |
| userId | Int | FK → User |
| workoutId | Int | FK → Workout |
| createdAt / updatedAt | DateTime | |

Связи: один-ко-многим с `ExerciseLog`.

### ExerciseLog (Лог упражнения)
| Поле | Тип | Примечание |
|---|---|---|
| id | Int (PK) | auto-increment |
| isCompleted | Boolean | false по умолчанию |
| userId | Int | FK → User |
| workoutLogId | Int | FK → WorkoutLog |
| exerciseId | Int | FK → Exercise |
| createdAt / updatedAt | DateTime | |

Связи: один-ко-многим с `ExerciseTime` (записи отдельных подходов).

### ExerciseTime (Подход)
| Поле | Тип | Примечание |
|---|---|---|
| id | Int (PK) | auto-increment |
| weight | Float | 0 по умолчанию |
| repeat | Int | 0 по умолчанию |
| isCompleted | Boolean | false по умолчанию |
| exerciseLogId | Int | FK → ExerciseLog |
| createdAt / updatedAt | DateTime | |

---

## Справка по API

Все защищённые маршруты требуют заголовка:
```
Authorization: Bearer <token>
```

### Аутентификация

| Метод | Путь | Авторизация | Описание |
|---|---|---|---|
| POST | `/api/auth/register` | - | Создать аккаунт |
| POST | `/api/auth/login` | - | Войти |

**POST /api/auth/register**
```json
// Тело запроса
{ "email": "user@example.com", "password": "secret", "name": "Алексей" }

// Ответ 201
{ "user": { "id": 1, "email": "...", "name": "Алексей", "images": [] }, "token": "<jwt>" }
```

**POST /api/auth/login**
```json
// Тело запроса
{ "email": "user@example.com", "password": "secret" }

// Ответ 200
{ "user": { ... }, "token": "<jwt>" }
```

---

### Пользователи

| Метод | Путь | Авторизация | Описание |
|---|---|---|---|
| GET | `/api/users/profile` | Обязательна | Получить профиль + статистику |

---

### Упражнения

| Метод | Путь | Авторизация | Описание |
|---|---|---|---|
| GET | `/api/exercises` | Обязательна | Список всех упражнений (новые сначала) |
| POST | `/api/exercises` | Обязательна | Создать упражнение |
| PUT | `/api/exercises/:id` | Обязательна | Обновить упражнение |
| DELETE | `/api/exercises/:id` | Обязательна | Удалить упражнение |

**POST /api/exercises** body: `{ "name": "Отжимания", "times": 3, "iconPath": "/images/pushup.png" }`

Статические файлы иконок доступны по пути `/public/images/exercises/*`.

---

### Логи упражнений

Лог упражнения - это запись одного упражнения в рамках тренировочной сессии. Каждый лог содержит строки `ExerciseTime` (по одной на каждый подход).

| Метод | Путь | Авторизация | Описание |
|---|---|---|---|
| GET | `/api/exercises/log/:id` | Обязательна | Получить лог с предыдущими значениями |
| POST | `/api/exercises/log/:id` | Обязательна | Создать лог для упражнения `:id` |
| PUT | `/api/exercises/log/time/:id` | Обязательна | Обновить отдельный подход |
| PATCH | `/api/exercises/log/complete/:id` | Обязательна | Отметить лог как выполненный |

**PUT /api/exercises/log/time/:id** body: `{ "weight": 60, "repeat": 10, "isCompleted": true }`

**PATCH /api/exercises/log/complete/:id** body: `{ "isCompleted": true }`

---

### Тренировки

| Метод | Путь | Авторизация | Описание |
|---|---|---|---|
| GET | `/api/workouts` | Обязательна | Список всех планов тренировок |
| POST | `/api/workouts` | Обязательна | Создать план тренировки |
| GET | `/api/workouts/:id` | Обязательна | Получить план (с расчётом времени) |
| PUT | `/api/workouts/:id` | Обязательна | Обновить план тренировки |
| DELETE | `/api/workouts/:id` | Обязательна | Удалить план тренировки |

**POST /api/workouts** body: `{ "name": "День груди", "exercises": [1, 3, 7] }`

Ответ `GET /api/workouts/:id` включает поле `minutes` - оценочная длительность тренировки: `ceil(количество упражнений × 5)`.

---

### Логи тренировок

Лог тренировки - это одна завершённая тренировочная сессия. При создании автоматически инициализируются логи упражнений и подходы для каждого упражнения из плана.

| Метод | Путь | Авторизация | Описание |
|---|---|---|---|
| GET | `/api/workouts/log/:id` | Обязательна | Получить сессию со всеми логами упражнений |
| POST | `/api/workouts/log/:id` | Обязательна | Начать сессию для тренировки `:id` |
| PATCH | `/api/workouts/log/complete/:id` | Обязательна | Отметить сессию как завершённую |

**PATCH /api/workouts/log/complete/:id** body: `{ "isCompleted": true }`

---

## Запуск проекта

### Требования

- Node.js ≥ 18
- База данных PostgreSQL

### Установка

```bash
# 1. Клонировать репозиторий
git clone <repo-url>
cd workout-backend/back

# 2. Установить зависимости
npm install

# 3. Создать файл окружения
cp .env.example .env   # или создать .env вручную

# 4. Заполнить .env
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DBNAME"
JWT_TOKEN="ваш-секретный-ключ"
PORT=5000
NODE_ENV=development

# 5. Применить миграции базы данных
npx prisma migrate deploy

# 6. Запустить dev-сервер
npm run dev
```

API будет доступно по адресу `http://localhost:5000`.

### Переменные окружения

| Переменная | Обязательная | Описание |
|---|---|---|
| `DATABASE_URL` | Да | Строка подключения к PostgreSQL |
| `JWT_TOKEN` | Да | Секрет для подписи JWT-токенов |
| `PORT` | Нет | HTTP-порт (по умолчанию: 5000) |
| `NODE_ENV` | Нет | Установить `development` для включения логов Morgan |

---

## Структура проекта

```
workout-backend/
├── back/                   # Корень приложения
│   ├── package.json
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   ├── public/
│   │   └── images/exercises/
│   └── src/
│       ├── app.js
│       ├── controllers/
│       ├── db/
│       ├── middlewares/
│       ├── routes/
│       └── utils/
└── README.md
```

---

## Контакты

Олег Мелех - [oleg.melekh.frontend@gmail.com](mailto:oleg.melekh.frontend@gmail.com)
