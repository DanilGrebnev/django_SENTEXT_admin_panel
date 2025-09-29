# 1. Назначение и архитектура

Страница отображает список чатов с пагинацией, позволяет открывать конкретный чат и просматривать его сообщения, содержит хлебные крошки и левое меню с разделами. Весь рендер в DOM выполняется через публичные методы PageRenderer.

Координация приложения построена на едином EventEmitter (далее — bus). Слой страницы (наш код) только слушает и эмитит события bus и вызывает публичные методы PageRenderer, не трогая их внутренности. Внутренняя реализация EventEmitter и PageRenderer вне рамок этого документа; здесь описано ровно когда и зачем вызываются их публичные методы.

# 2. Публичный контракт «слоя страницы»

Ниже — публичные функции слоя страницы. Они связывают bus (события) и PageRenderer (отрисовка), а также вызываются внешним кодом/инициализацией.

## start()

- Назначение: запуск страницы и первичная отрисовка (UX: пользователь видит список, пагинацию, хлебные крошки, левую колонку).
- События: bus.emit('app:init'). Подписки на события bus настраиваются здесь же.
- Вызовы PageRenderer: по результатам загрузки данных — renderChatsList, renderPagination, renderBreadcrumbs, renderLeftColumn.

## destroy()

- Назначение: корректное завершение, отписки от bus и очистка любых внешних подписок.
- События: — (может эмитить 'app:destroy' при необходимости).
- Вызовы PageRenderer: — (ничего не рендерит; задача — снять подписки).

## loadChats(pageNumber)

- Назначение: загрузить и отрисовать список чатов для указанной страницы.
- События: слушает bus.on('chats:load', { page }); может эмитить внутренние статусы (опционально), а также bus.emit('pagination:change', { page }) при синхронизации.
- Вызовы PageRenderer: renderChatsList(items) — таблица чатов; renderPagination({ pagesAmount, activePage }) — пагинация; при необходимости renderBreadcrumbs(items) — обновление навигации.

## openChat(chatId)

- Назначение: открыть чат по ID и отрисовать его сообщения.
- События: слушает bus.on('chat:open', { chatId }).
- Вызовы PageRenderer: setMessageContent(messages) — сохранить текущие сообщения; openChatById(chatId, messages) — открыть нужный ряд таблицы и показать чат. Альтернативно, если чат уже найден в DOM, может вызываться chatMessageRender(chatId, messages).

## updateBreadcrumbs(items)

- Назначение: обновить хлебные крошки на странице.
- События: слушает bus.on('breadcrumbs:set', { items }).
- Вызовы PageRenderer: renderBreadcrumbs(items) — перерисовать навигационную цепочку.

## updateLeft(sections)

- Назначение: обновить левое меню (разделы и пункты).
- События: слушает bus.on('left:update', { sections }).
- Вызовы PageRenderer: renderLeftColumn(sections) — перерисовать левую колонку.

# 3. Публичные методы PageRenderer, которые ИСПОЛЬЗУЮТСЯ

- renderLeftColumn(sections) — отрисовать левую колонку
- renderChatsList(items) — отрисовать таблицу чатов
- renderPagination({ pagesAmount, activePage }) — отрисовать пагинацию
- renderBreadcrumbs(items) — отрисовать хлебные крошки
- chatMessageRender(chatId, messages) — отрисовать сообщения для уже найденного чата
- setMessageContent(messages) — сохранить текущие сообщения (для последующего рендера)
- openChatById(chatId, messages) — открыть чат по ID и отрисовать сообщения

# 4. Событийный контракт (карта bus-событий)

| Событие           | Кто эмитит            | Пэйлоад                                                           | Кто слушает                | Что вызывается у PageRenderer                                                                                  |
| ----------------- | --------------------- | ----------------------------------------------------------------- | -------------------------- | -------------------------------------------------------------------------------------------------------------- |
| app:init          | start()               | —                                                                 | слой страницы              | (последовательно после загрузки данных) renderChatsList, renderPagination, renderBreadcrumbs, renderLeftColumn |
| chats:load        | UI/пагинация/start    | { page:number }                                                   | loadChats                  | renderChatsList, renderPagination (и при необходимости renderBreadcrumbs)                                      |
| chat:open         | UI (клик по UID)      | { chatId:string }                                                 | openChat                   | setMessageContent, openChatById (или chatMessageRender)                                                        |
| pagination:change | UI (клик по странице) | { page:number }                                                   | loadChats                  | renderChatsList, renderPagination                                                                              |
| selection:change  | UI (чекбоксы)         | { selectedIds?:string[], counts?:{selected:number,total:number} } | (если нужно) слой страницы | (опционально) повторный renderChatsList для обновления счётчиков                                               |
| breadcrumbs:set   | Data/навигация        | { items:Array<{text,link}> }                                      | updateBreadcrumbs          | renderBreadcrumbs                                                                                              |
| left:update       | Data/инициализация    | { sections:Array }                                                | updateLeft                 | renderLeftColumn                                                                                               |

# 5. Карта делегирования DOM-событий (UI → bus → renderer)

| Пользовательское действие              | bus.emit(...)              | Публичная функция слоя                | Вызовы PageRenderer                                                        |
| -------------------------------------- | -------------------------- | ------------------------------------- | -------------------------------------------------------------------------- |
| Клик по `.table__uid`                  | chat:open { chatId }       | openChat(chatId)                      | setMessageContent + openChatById (либо chatMessageRender)                  |
| Клик по `.pagination__item[data-page]` | pagination:change { page } | loadChats(page)                       | renderChatsList + renderPagination (+ при необходимости renderBreadcrumbs) |
| Чекбокс select-all/строка              | selection:change {...}     | (по требованию)                       | (опционально) renderChatsList для обновления счётчика выбранных            |
| Клик по `.breadcrumbs__link`           | chats:load/breadcrumbs:set | loadChats(...)/updateBreadcrumbs(...) | renderChatsList/renderBreadcrumbs                                          |

# 6. Основные сценарии (пошагово)

## Открытие страницы

1. start() создаёт bus, регистрирует подписки, инициирует первичные загрузки, затем bus.emit('app:init').
2. loadChats(1) → после получения данных: renderChatsList + renderPagination + renderBreadcrumbs.
3. updateLeft(sections) → renderLeftColumn.

## Переход на страницу N

1. Клик по кнопке пагинации → bus.emit('pagination:change', { page: N }).
2. loadChats(N) → renderChatsList + renderPagination (+ при необходимости обновление renderBreadcrumbs).

## Открытие чата по UID

1. Клик по `.table__uid` → bus.emit('chat:open', { chatId }).
2. openChat(chatId) → setMessageContent(messages) → openChatById(chatId, messages) (или chatMessageRender).

## Пустой чат

1. openChat(chatId) получает [].
2. Вызывает setMessageContent([]) и openChatById — PageRenderer сам отображает заглушку.

# 7. Состояния и данные (минимально)

- Текущая страница: number (page)
- Текущий чат: string (chatId)
- Текущие сообщения: через PageRenderer.setMessageContent(messages)
- Источники данных (в памяти/через API):
  - getChats({ page }) → { pagesAmount:number, activePage:number, data:Array<ChatItem> }
  - getChatMessages(chatId) → Array<Message>
  - getBreadcrumbs() → Array<Breadcrumb>
  - getLeftSections() → Array<Section>

Ожидаемые форматы:

- ChatItem: { uid:string, email:string, session:string }
- Message: { role:'user'|'assistant', content:string }
- Pagination meta: { pagesAmount:number, activePage:number }
- Breadcrumb: { text:string, link:string }
- Section: { sectionTitle:string, list:Array<{ itemTitle:string, titleLink:string, addLink:string }> }

# 8. Обработка ошибок и защита от неконсистентности

- Не найден chatId → слой не падает; openChat вызывает PageRenderer.openChatById(chatId, messages). Если элемент не найден в DOM, визуально ничего не меняется (или используется chatMessageRender при наличии).
- Пустые messages → setMessageContent([]) допустимо; PageRenderer отображает пустое состояние.
- Некорректная пагинация → при отсутствии данных вызывается renderChatsList([]) и renderPagination с безопасными значениями.
- Любые ошибки загрузки → слой логирует, но продолжает работу, повторные вызовы PageRenderer возможны с пустыми массивами.

# 9. Псевдокод ТОЛЬКО уровня использования

```js
function start() {
  bus.on("pagination:change", ({ page }) => loadChats(page))
  bus.on("chat:open", ({ chatId }) => openChat(chatId))
  bus.on("breadcrumbs:set", ({ items }) => updateBreadcrumbs(items))
  bus.on("left:update", ({ sections }) => updateLeft(sections))
  bus.emit("app:init")
  loadChats(1)
}
```

```js
function loadChats(page) {
  const { data, pagesAmount, activePage } = getChats({ page })
  renderer.renderChatsList(data)
  renderer.renderPagination({ pagesAmount, activePage })
}
```

```js
function openChat(chatId) {
  const messages = getChatMessages(chatId)
  renderer.setMessageContent(messages)
  renderer.openChatById(chatId, messages)
}
```

```js
function updateBreadcrumbs(items) {
  renderer.renderBreadcrumbs(items)
}
```

```js
function updateLeft(sections) {
  renderer.renderLeftColumn(sections)
}
```

# 10. Чек-лист ручного теста

- Клик по UID в таблице → ожидается setMessageContent + openChatById.
- Клик по странице «3» в пагинации → ожидается renderChatsList + renderPagination.
- Вызов updateBreadcrumbs([...]) → ожидается renderBreadcrumbs с новыми элементами.
- Вызов updateLeft(sections) → ожидается renderLeftColumn с секциями.
- Открытие пустого чата → ожидается setMessageContent([]) + openChatById и отображение пустого состояния.
- Первая загрузка (start) → после данных ожидаются renderChatsList + renderPagination + renderBreadcrumbs + renderLeftColumn.

