# Online Support API Structure

This documents the HTTP and WebSocket shape that the app expects when `onlineSupport` is configured in `config.json`.

## Config Block
- Location: `config.json` -> `onlineSupport` (array).
- Fields per entry:
  - `url` (string): human friendly link that can be shown to users.
  - `description` (string): label for the entry.
  - `apiBaseUrl` (string): base HTTP URL used by all REST calls, e.g. `https://chat.example.com`.
  - `wsBaseUrl` (string): base WebSocket URL, e.g. `wss://chat.example.com`.
  - `metadata` (object, optional): free-form extra fields.

## Auth
- All HTTP requests include `Authorization: <token>` where the token comes from `XBoardSDK.instance.getToken()`.
- Content type for JSON requests: `application/json`.

## HTTP Endpoints (apiBaseUrl)
- `POST /messages/`
  - Body: `{ "content": "<text message>" }`
  - Response: `ChatMessage` JSON (see model below).
- `POST /files/send-with-attachment`
  - Body: `{ "content": "<text message>", "attachment_ids": [<int>, ...] }`
  - Response: `ChatMessage`.
- `GET /messages/?limit=<int>&offset=<int>`
  - Defaults: `limit=20`, `offset=0`.
  - Response: `{ "items": [ChatMessage], "total": <int> }`.
- `GET /messages/unread`
  - Response body should be the unread count (number or numeric string).
- `POST /messages/mark-read`
  - Body: `{ "message_ids": ["<id>", ...] }`.
  - If this path is not available the client will fall back to trying `/messages/read/`, `/messages/mark_read/`, `/messages/read-status/`, and `/mark-read/` with the same payload.
- File upload
  - `POST /files/upload` (multipart) with field name `file`.
  - `POST /files/upload-multiple` (multipart) with field name `files` (repeatable).
  - Response: a single `MessageAttachment` for `/files/upload`; `{ "attachments": [MessageAttachment, ...] }` for `/files/upload-multiple`.
  - `GET /files/{attachmentId}/info` -> `MessageAttachment`.
  - `DELETE /files/{attachmentId}` deletes the attachment.
  - File access helpers: `/files/{attachmentId}` (optionally `?size=<WxH>`), `/files/{attachmentId}?size=<WxH>` also used for thumbnails.

### Models
- `ChatMessage`
  - `id` (int)
  - `content` (string)
  - `sender_type` (`"agent"` | `"user"`)
  - `message_type` (`"text"` | `"image"` | `"file"`)
  - `created_at` (ISO8601 string)
  - `read` (bool)
  - `attachments` (array of `MessageAttachment`, optional)
- `MessageAttachment`
  - `id` (int)
  - `filename` (string)
  - `file_size` (int, bytes)
  - `mime_type` (string)
  - `thumbnail_url` (string, nullable)
  - `file_url` (string)

## WebSocket (wsBaseUrl)
- Connection URL: `${wsBaseUrl}/ws`.
- Immediately after opening, the client sends `{ "type": "auth", "token": "<token>" }`.
- Heartbeat: client sends `{ "type": "ping", "timestamp": <ms_since_epoch> }` every 30s.
- Outbound messages:
  - `{ "type": "message", "content": "<text>" }`
  - `{ "type": "message_with_attachments", "content": "<text>", "attachment_ids": [<int>, ...] }`
  - `{ "type": "mark_read", "message_ids": [<int>, ...] }`
- Expected inbound message types (field `type`):
  - `connection_established`, `new_message`, `message_sent`, `marked_read`, `pong`, `error`, or any other string (treated as `unknown`).
  - The payload is passed through in the `data` field of `WebSocketMessage`.
