# Стратегия добавления VK как нативного канала в Chatwoot

## Цель
Добавить в Chatwoot полноценный канал VK (как Telegram/WhatsApp):
- входящие сообщения из VK Callback API,
- исходящие сообщения из Chatwoot в VK,
- поддержка вложений, базовых кнопок и статусов,
- надежность, безопасность и управляемый rollout.

---

## 1) Объем работ

### MVP (этап 1)
- Новый тип канала: `Channel::Vk`.
- Создание inbox типа `vk` через существующий API inboxes.
- Webhook endpoint для VK Callback API (включая `confirmation`).
- Входящие: текст + базовые вложения.
- Исходящие: текст + базовые вложения.
- Идемпотентность входящих/исходящих.
- Логирование ошибок + retries.

### Этап 2
- Кнопки/keyboard VK.
- Статусы delivery/read (где доступны события VK).
- Health-check канала и UX для reauthorization.

### Этап 3
- Расширенные типы вложений и тонкая настройка лимитов.
- Улучшенная аналитика по событиям VK.

---

## 2) Архитектура

### Ключевые компоненты
1. **Модель канала**: `Channel::Vk` (channelable).
2. **Webhook контроллер**: `Webhooks::VkController`.
3. **Job-слой**: `Webhooks::VkEventsJob` / `Vk::ProcessWebhookEventJob`.
4. **Inbound-сервисы**: роутер событий + сервис обработки `message_new`.
5. **Outbound-сервисы**: отправка сообщений и вложений в VK API.
6. **Dedup слой**: защита от дублей callback и повторной отправки.
7. **Frontend мастер**: подключение VK inbox в настройках.

### Принцип
- HTTP webhook отвечает быстро (`ok`), тяжелая обработка всегда в Sidekiq.
- Любая обработка событий и отправок должна быть идемпотентной.

---

## 3) Данные и миграции

### Таблица `channel_vk`
Рекомендуемые поля:
- `account_id` (FK)
- `group_id` (ID сообщества)
- `group_name`
- `access_token` (encrypted)
- `callback_secret` (encrypted)
- `confirmation_code` (encrypted)
- `api_version` (default, например `5.199`)
- `enabled` (bool)
- timestamps

Индексы:
- unique `(account_id, group_id)`
- index `(enabled)`

### Таблица идемпотентности (рекомендуется)
`vk_event_receipts`:
- `channel_vk_id`
- `event_type`
- `event_unique_key`
- `status`
- `processed_at`
- timestamps

Индекс:
- unique `(channel_vk_id, event_unique_key)`

---

## 4) Backend API и роуты

### Добавления в `InboxesController`
- Добавить `vk` в `allowed_channel_types`.
- Добавить `'vk' => Channel::Vk` в `channel_type_from_params`.
- Определить `Channel::Vk::EDITABLE_ATTRS`.

### Webhook роут
- `POST /webhooks/vk/:group_id` → `webhooks/vk#events`

### Логика `Webhooks::VkController`
1. Найти `Channel::Vk` по `group_id`.
2. Если `type=confirmation` — вернуть `confirmation_code`.
3. Проверить `secret` из payload.
4. Проверить активность канала.
5. Поставить событие в job.
6. Вернуть `ok`.

---

## 5) Inbound pipeline (VK → Chatwoot)

### События для MVP
- `message_new` (обязательно)
- `message_reply` (по необходимости)

### Алгоритм обработки
1. Dedup по `event_id`/`message_id` + `group_id`.
2. Найти/создать Contact по `vk_user_id`.
3. Найти/создать ContactInbox (`source_id: vk:<user_id>`).
4. Найти/создать Conversation.
5. Создать входящее Message с `external_source_id`.
6. Для вложений: скачать/валидировать/сохранить в ActiveStorage.

---

## 6) Outbound pipeline (Chatwoot → VK)

### Алгоритм
1. На исходящее сообщение канала `vk` запускать `Vk::SendMessageJob`.
2. Формировать `random_id` детерминированно (идемпотентность VK).
3. Отправлять текст через `messages.send`.
4. Вложения: upload flow VK (photo/doc) + attach IDs.
5. Сохранять mapping `chatwoot_message_id -> vk_message_id`.
6. При ошибке — статус `failed`, reason в лог.

### Retry политика
- Сетевые/5xx/429 → retry с экспоненциальным backoff + jitter.
- Невалидный токен/доступ → fail fast + флаг degraded/reauthorization.

---

## 7) Безопасность

- Проверка callback `secret` обязательна.
- `confirmation` handshake обязателен.
- Токены/секреты хранить только encrypted.
- Не логировать секреты и токены.
- Ограничения на размер/тип вложений.
- Защита от replay через dedup.

---

## 8) Frontend (настройка канала)

Добавить UI-мастер VK inbox:
- Group ID
- Access Token
- Callback Secret
- Confirmation Code
- Кнопка "Проверить подключение"

Плюс:
- понятные ошибки (invalid token, secret mismatch),
- состояние канала (active/degraded).

---

## 9) Тесты

### Unit
- валидации `Channel::Vk`
- secret verification
- dedup key builder
- outbound random_id generation

### Integration
- webhook confirmation
- webhook с valid/invalid secret
- `message_new` end-to-end
- duplicate event ignored
- outbound retry при 429

### E2E
- входящее VK сообщение появляется в inbox
- ответ агента уходит в VK
- вложение проходит в обе стороны

---

## 10) Rollout и фича-флаги

Флаги:
- `vk_channel_integration`
- `vk_channel_outbound_enabled`
- `vk_channel_read_receipts_enabled`

План rollout:
1. Dark launch backend.
2. Staging + тестовое VK сообщество.
3. Canary на 1-2 inbox.
4. Постепенное расширение.

---

## 11) Риски и mitigation

1. **Дубли сообщений**
   - unique dedup + идемпотентные jobs.
2. **Лимиты VK / 429**
   - throttling + backoff.
3. **Протухший токен**
   - health-check + reauth UX.
4. **Регрессии в sender pipeline**
   - изоляция adapter + regression suite.
5. **Нестабильность вложений**
   - поэтапная поддержка типов + fallback.

---

## 12) Definition of Done (DoD)

- [ ] Канал `vk` создается через API/UI.
- [ ] Webhook проходит confirmation и secret-проверку.
- [ ] Входящие текст/вложения корректно создаются в Chatwoot.
- [ ] Исходящие текст/вложения доставляются в VK.
- [ ] Дедуп и retries работают.
- [ ] Метрики/логи/алерты заведены.
- [ ] Фича-флаги позволяют быстро отключить канал.
- [ ] Документация и runbook готовы.

---

## 13) Порядок реализации (чеклист)

1. Миграции + `Channel::Vk`.
2. Подключение типа `vk` в inbox creation pipeline.
3. Webhook route/controller + confirmation/secret.
4. Job router + inbound `message_new`.
5. Outbound send service.
6. Attachments in/out.
7. Retry/throttle/idempotency hardening.
8. Frontend setup flow.
9. Тесты + staging e2e.
10. Canary rollout.
