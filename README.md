# SupaStalk

> **exteraGram plugin** — silently capture every message from up to 5 tracked users across all shared chats.

---

## Features

- Track up to **5 users simultaneously**, each with their own local storage slot
- Captures **all message types**: text, photos, videos, voice, stickers, polls, locations, forwards, replies, and more
- Stores **full formatting** (entity offsets/lengths for bold, italic, links, code blocks, etc.)
- Generates **deep links** back to the original message (`tg://openmessage` for DMs, `t.me/c/…` for channels)
- **Export** any slot to a `.txt` file with timestamps, chat names, links, and media labels
- User lookup by **numeric ID** or **@username** with live preview before confirming
- All data stored **locally on device** — nothing leaves your phone

---

## Installation

1. Open **exteraGram** → Settings → Plugins
2. Tap **+** and paste the raw plugin URL:
   ```
   https://raw.githubusercontent.com/lovethistleclaw/supastalk/main/supastalk.py
   ```
3. Enable the plugin

---

## Usage

1. Open plugin settings → tap an **empty slot**
2. Enter a user ID (e.g. `123456789`) or `@username`
3. Tap **Search** — a preview appears with the found user's name
4. Tap **Add to tracking** to confirm
5. From now on every message this user sends in any shared chat is captured automatically

### Viewing captured messages

- Tap an **active slot** in settings to open its detail page
- Each message shows: timestamp, chat name, text preview, media type
- Tap any message item for the full content + link
- Tap **Export … messages to .txt** to save a full report to device storage

### Removing a user

Open the slot → tap **Remove from tracking** (data is deleted from disk).

---

## Data format

Each slot is stored as `slot_N.json` in the app's external files directory under `supastalk/`.  
Each message entry contains:

| Field | Description |
|---|---|
| `id` | Telegram message ID |
| `date` | Unix timestamp |
| `chat_id` | Numeric chat ID |
| `chat_title` | Human-readable chat name |
| `text` | Raw message text |
| `entities` | Formatting entities (type, offset, length) |
| `media` | Media type label (e.g. `📷 Photo`, `🎤 Voice`) |
| `fwd_from` | Forward origin name (if forwarded) |
| `reply_to_id` | ID of the replied-to message |
| `link` | Deep link to the original message |
| `captured_at` | Unix timestamp when captured |

---

## Requirements

- exteraGram **≥ 12.1.1**

---

## Author

[@Asteron](https://t.me/Asteron)

## Version

1.0.0