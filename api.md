# API Endpoints для Django Admin Panel

## 1. GET_FILTERS_URL - Получение фильтров для сайдбара

**Endpoint:** `GET /api/filters/`

**Описание:** Возвращает структуру фильтров для левого сайдбара

**Response format:**

```json
[
  {
    "sectionTitle": "Authentication and Authorization",
    "list": [
      {
        "itemTitle": "Groups",
        "titleLink": "/admin/auth/groups/",
        "addLink": "/admin/auth/groups/add/"
      },
      {
        "itemTitle": "Users",
        "titleLink": "/admin/auth/users/",
        "addLink": "/admin/auth/users/add/"
      }
    ]
  },
  {
    "sectionTitle": "LLM Integration",
    "list": [
      {
        "itemTitle": "Messages",
        "titleLink": "/admin/llm/messages/",
        "addLink": "/admin/llm/messages/add/"
      }
    ]
  }
]
```

## 2. GET_CHATS_URL - Получение списка чатов с пагинацией и поиском

**Endpoint:** `GET /api/chats/?page={page}&messageSearch={messageSearch}&emailSearch={emailSearch}`

**Parameters:**

- `page` (int) - номер страницы для пагинации
- `messageSearch` (string, optional) - поиск по содержимому сообщений в чатах
- `emailSearch` (string, optional) - поиск по email пользователей

**Примеры запросов:**

- `GET /api/chats/?page=1` - получить первую страницу всех чатов
- `GET /api/chats/?page=1&messageSearch=hello` - поиск чатов содержащих "hello" в сообщениях
- `GET /api/chats/?page=1&emailSearch=admin@test.com` - поиск чатов пользователя с email "admin@test.com"
- `GET /api/chats/?page=1&messageSearch=hello&emailSearch=user@example.com` - комбинированный поиск

**Response format:**

```json
{
  "pagesAmount": 29,
  "activePage": 1,
  "data": [
    {
      "uid": "c7d229bd-26c4-4757-9edb-cbe5f7765ca4",
      "email": "da000shi@gmail.com",
      "session": "New chat"
    },
    {
      "uid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "email": "user@example.com",
      "session": "Previous chat"
    }
  ]
}
```

## 3. GET_CHATS_MESSAGES_URL - Получение сообщений конкретного чата

**Endpoint:** `GET /api/chats/messages/?chatId={chatId}`

**Parameters:**

- `chatId` (string) - UUID чата для получения сообщений

**Response format:**

```json
[
  {
    "role": "user",
    "content": "Здравствуйте!"
  },
  {
    "role": "assistant",
    "content": "Добрый день!"
  },
  {
    "role": "user",
    "content": "Расскажи что-нибудь"
  }
]
```

## Обработка ошибок

Все endpoints должны возвращать соответствующие HTTP статус коды:

- `200` - успешный запрос
- `404` - данные не найдены
- `500` - внутренняя ошибка сервера

При ошибках фронтенд логирует в консоль, но продолжает работу.
