import os
import json
import time

from base_plugin import BasePlugin, HookResult, HookStrategy
from client_utils import (
    run_on_queue,
    send_request, RequestCallback,
    get_messages_controller,
)
from android_utils import run_on_ui_thread, log
from ui.bulletin import BulletinHelper
from ui.settings import Text, Header, Switch, Divider, Input

from org.telegram.messenger import ApplicationLoader
from java import jclass

__id__ = "supastalk"
__name__ = "SupaStalk"
__description__ = "Track up to 5 users — captures every message they send in shared chats with links, media info and full formatting."
__author__ = "@ASTER0N"
__version__ = "1.0.0"
__min_version__ = "12.1.1"

MAX_SLOTS = 5
MAX_MESSAGES_PER_SLOT = 500
_SHOWN_MESSAGES_LIMIT = 100


# ─────────────────────────────────────────────────────────
# Module-level helpers
# ─────────────────────────────────────────────────────────

def _user_display(user):
    """Build 'First Last @username' from a TLRPC.User object."""
    if user is None:
        return "Unknown"
    parts = []
    try:
        fn = user.first_name
        if fn:
            parts.append(str(fn))
    except Exception:
        pass
    try:
        ln = user.last_name
        if ln:
            parts.append(str(ln))
    except Exception:
        pass
    name = " ".join(parts).strip()
    if not name:
        try:
            name = f"ID:{int(user.id)}"
        except Exception:
            name = "Unknown"
    try:
        un = user.username
        if un:
            name += f" @{un}"
    except Exception:
        pass
    return name


def _user_display_from_dict(d):
    """Build display name from a stored slot dict."""
    parts = []
    fn = d.get("first_name", "")
    ln = d.get("last_name", "")
    if fn:
        parts.append(fn)
    if ln:
        parts.append(ln)
    name = " ".join(parts).strip() or f"ID:{d.get('user_id', '?')}"
    un = d.get("username", "")
    if un:
        name += f" @{un}"
    return name


def _media_label(media):
    """Return an emoji-prefixed label for a TLRPC.MessageMedia object."""
    if media is None:
        return None
    cls = type(media).__name__
    base_labels = {
        "TL_messageMediaPhoto":   "📷 Photo",
        "TL_messageMediaGeo":     "📍 Location",
        "TL_messageMediaGeoLive": "📍 Live Location",
        "TL_messageMediaContact": "👤 Contact",
        "TL_messageMediaPoll":    "📊 Poll",
        "TL_messageMediaDice":    "🎲 Dice",
        "TL_messageMediaWebPage": "🔗 Web Preview",
        "TL_messageMediaGame":    "🎮 Game",
        "TL_messageMediaInvoice": "💰 Invoice",
        "TL_messageMediaStory":   "📖 Story",
    }
    if cls != "TL_messageMediaDocument":
        return base_labels.get(cls, f"📦 Media")
    # Determine document subtype from attributes
    try:
        for attr in media.document.attributes:
            an = type(attr).__name__
            if "Video" in an:
                try:
                    if attr.round_message:
                        return "🎥 Video Message"
                except Exception:
                    pass
                return "🎬 Video"
            if "Audio" in an:
                try:
                    if attr.voice:
                        return "🎤 Voice"
                except Exception:
                    pass
                return "🎵 Audio"
            if "Sticker" in an:
                return "🎭 Sticker"
            if "Animated" in an:
                return "👾 GIF"
    except Exception:
        pass
    return "📎 Document"


def _make_link(peer_id_obj, msg_id):
    """Build a deep link to a message. Returns None for basic groups."""
    try:
        cls = type(peer_id_obj).__name__
        if cls == "TL_peerUser":
            uid = int(peer_id_obj.user_id)
            return f"tg://openmessage?user_id={uid}&message_id={msg_id}"
        if cls == "TL_peerChannel":
            cid = int(peer_id_obj.channel_id)
            return f"https://t.me/c/{cid}/{msg_id}"
    except Exception:
        pass
    return None


def _chat_id_from_peer(peer_id_obj):
    """Convert TLRPC.Peer to a signed integer chat id."""
    try:
        cls = type(peer_id_obj).__name__
        if cls == "TL_peerUser":
            return int(peer_id_obj.user_id)
        if cls == "TL_peerChat":
            return -int(peer_id_obj.chat_id)
        if cls == "TL_peerChannel":
            return -(int(peer_id_obj.channel_id) + 1_000_000_000_000)
    except Exception:
        pass
    return 0


def _entities_list(entities):
    """Serialize TLRPC entity list to a JSON-safe list."""
    result = []
    if not entities:
        return result
    try:
        for ent in entities:
            result.append({
                "type":   type(ent).__name__,
                "offset": int(ent.offset),
                "length": int(ent.length),
            })
    except Exception:
        pass
    return result


def _fmt_date(ts):
    try:
        return time.strftime("%H:%M  %d.%m", time.localtime(ts))
    except Exception:
        return str(ts)


def _fmt_date_long(ts):
    try:
        return time.strftime("%H:%M  %d.%m.%Y", time.localtime(ts))
    except Exception:
        return str(ts)


# ─────────────────────────────────────────────────────────
# Plugin class
# ─────────────────────────────────────────────────────────

class SupaStalkPlugin(BasePlugin):

    def __init__(self):
        super().__init__()
        # {user_id_int: slot_index} — rebuilt from disk on load and after every change
        self._tracked: dict = {}
        self._storage_dir: str = ""
        # {slot_index: user_dict} — RAM-only staging area before the user confirms an add
        self._pending: dict = {}

    # ── Lifecycle ────────────────────────────────────────

    def on_plugin_load(self):
        self._storage_dir = self._resolve_storage_dir()
        self._reload_tracked()
        self.add_hook("TL_updateNewMessage")
        log(f"[SupaStalk] Loaded. Tracking {len(self._tracked)} user(s).")

    def on_plugin_unload(self):
        self._tracked.clear()
        self._pending.clear()

    # ── Hook ─────────────────────────────────────────────

    def on_update_hook(self, update_name, account, update):
        if update_name != "TL_updateNewMessage":
            return HookResult(strategy=HookStrategy.DEFAULT)
        if not self.get_setting("enabled", True):
            return HookResult(strategy=HookStrategy.DEFAULT)
        try:
            msg = update.message
            if msg is None:
                return HookResult(strategy=HookStrategy.DEFAULT)
            sender_id = self._extract_sender_id(msg)
            if sender_id is None or sender_id not in self._tracked:
                return HookResult(strategy=HookStrategy.DEFAULT)
            slot = self._tracked[sender_id]
            run_on_queue(lambda: self._capture(slot, msg))
        except Exception as e:
            log(f"[SupaStalk] Hook error: {e}")
        return HookResult(strategy=HookStrategy.DEFAULT)

    @staticmethod
    def _extract_sender_id(msg):
        """Return sender user_id as int, or None."""
        try:
            from_id = msg.from_id
            if from_id is not None and type(from_id).__name__ == "TL_peerUser":
                return int(from_id.user_id)
        except Exception:
            pass
        # Fallback: private chat where from_id is absent
        try:
            peer = msg.peer_id
            if peer is not None and type(peer).__name__ == "TL_peerUser":
                return int(peer.user_id)
        except Exception:
            pass
        return None

    # ── Message capture ──────────────────────────────────

    def _capture(self, slot, msg):
        """Runs on background queue. Extract data and append to slot JSON."""
        try:
            slot_data = self._load_slot(slot)
            if slot_data is None:
                return

            # Chat info
            chat_id = 0
            chat_title = ""
            link = None
            try:
                peer = msg.peer_id
                chat_id = _chat_id_from_peer(peer)
                link = _make_link(peer, msg.id)
                mc = get_messages_controller()
                cls = type(peer).__name__
                if cls == "TL_peerUser":
                    u = mc.getUser(int(peer.user_id))
                    if u:
                        chat_title = _user_display(u)
                elif cls == "TL_peerChat":
                    c = mc.getChat(int(peer.chat_id))
                    if c:
                        chat_title = str(c.title or "")
                elif cls == "TL_peerChannel":
                    c = mc.getChat(int(peer.channel_id))
                    if c:
                        chat_title = str(c.title or "")
            except Exception:
                pass

            # Text
            text = ""
            try:
                text = str(msg.message or "")
            except Exception:
                pass

            # Media label
            media = None
            try:
                media = _media_label(msg.media)
            except Exception:
                pass

            # Formatting entities
            entities = []
            try:
                entities = _entities_list(msg.entities)
            except Exception:
                pass

            # Forward origin
            fwd_from = None
            try:
                if msg.fwd_from:
                    try:
                        fn = msg.fwd_from.from_name
                        fwd_from = str(fn) if fn else "forwarded"
                    except Exception:
                        fwd_from = "forwarded"
            except Exception:
                pass

            # Reply-to message id
            reply_to_id = None
            try:
                if msg.reply_to:
                    reply_to_id = int(msg.reply_to.reply_to_msg_id)
            except Exception:
                pass

            entry = {
                "id":           int(msg.id),
                "date":         int(msg.date),
                "chat_id":      chat_id,
                "chat_title":   chat_title,
                "text":         text,
                "entities":     entities,
                "media":        media,
                "fwd_from":     fwd_from,
                "reply_to_id":  reply_to_id,
                "link":         link,
                "captured_at":  int(time.time()),
            }

            messages = slot_data.get("messages", [])

            # Deduplicate by (msg_id, chat_id)
            key = (entry["id"], entry["chat_id"])
            if any((m["id"], m["chat_id"]) == key for m in messages):
                return

            messages.insert(0, entry)
            if len(messages) > MAX_MESSAGES_PER_SLOT:
                messages = messages[:MAX_MESSAGES_PER_SLOT]
            slot_data["messages"] = messages
            self._save_slot(slot, slot_data)
            log(f"[SupaStalk] Slot {slot}: captured msg {entry['id']} in '{chat_title or 'private'}'")
        except Exception as e:
            log(f"[SupaStalk] Capture error slot {slot}: {e}")

    # ── Storage ──────────────────────────────────────────

    def _resolve_storage_dir(self):
        try:
            ctx = ApplicationLoader.applicationContext
            base = None
            try:
                base = ctx.getExternalFilesDir(None)
            except Exception:
                pass
            if base is None:
                try:
                    base = ctx.getFilesDir()
                except Exception:
                    pass
            if base:
                path = os.path.join(str(base.getAbsolutePath()), "supastalk")
                os.makedirs(path, exist_ok=True)
                return path
        except Exception:
            pass
        path = os.path.join(os.environ.get("HOME", os.getcwd()), "supastalk")
        os.makedirs(path, exist_ok=True)
        return path

    def _slot_file(self, slot):
        return os.path.join(self._storage_dir, f"slot_{slot}.json")

    def _load_slot(self, slot):
        path = self._slot_file(slot)
        if not os.path.exists(path):
            return None
        try:
            with open(path, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception:
            return None

    def _save_slot(self, slot, data):
        path = self._slot_file(slot)
        tmp = path + ".tmp"
        try:
            with open(tmp, "w", encoding="utf-8") as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
            os.replace(tmp, path)
        except Exception as e:
            log(f"[SupaStalk] _save_slot error: {e}")

    def _delete_slot_file(self, slot):
        path = self._slot_file(slot)
        try:
            if os.path.exists(path):
                os.remove(path)
        except Exception:
            pass

    def _reload_tracked(self):
        self._tracked.clear()
        for s in range(MAX_SLOTS):
            d = self._load_slot(s)
            if d and "user_id" in d:
                self._tracked[int(d["user_id"])] = s

    # ── User resolution ──────────────────────────────────

    def _resolve_user_query(self, slot):
        query = self.get_setting(f"slot_{slot}_query", "").strip()
        if not query:
            BulletinHelper.show_error("Enter a user ID or @username first.")
            return
        self._pending.pop(slot, None)
        self.set_setting(f"slot_{slot}_preview", "", reload_settings=False)
        run_on_queue(lambda: self._do_resolve(slot, query))

    def _do_resolve(self, slot, query):
        try:
            TLRPC = jclass("org.telegram.tgnet.TLRPC")

            if query.lstrip("-").isdigit():
                uid = int(query)
                # Check local cache first (fast path)
                mc = get_messages_controller()
                user = mc.getUser(uid)
                if user:
                    self._on_resolved(slot, user)
                    return
                # Fallback: network request (works if user shares a chat with us)
                req = TLRPC.TL_users_getUsers()
                inp = TLRPC.TL_inputUser()
                inp.user_id = uid
                inp.access_hash = 0
                req.id.add(inp)

                def on_id_result(response, error):
                    if error or not response or len(response) == 0:
                        run_on_ui_thread(lambda: BulletinHelper.show_error(
                            "Not found. Share a chat with this user, or try @username."))
                        return
                    u = response.get(0)
                    if u is None:
                        run_on_ui_thread(lambda: BulletinHelper.show_error("User not found."))
                        return
                    self._on_resolved(slot, u)

                send_request(req, RequestCallback(on_id_result))

            else:
                username = query.lstrip("@")
                req = TLRPC.TL_contacts_resolveUsername()
                req.username = username

                def on_username_result(response, error):
                    if error:
                        run_on_ui_thread(lambda: BulletinHelper.show_error(
                            f"Not found: {error.text}"))
                        return
                    if response is None:
                        run_on_ui_thread(lambda: BulletinHelper.show_error("User not found."))
                        return
                    try:
                        users = response.users
                        if not users or len(users) == 0:
                            run_on_ui_thread(lambda: BulletinHelper.show_error("User not found."))
                            return
                        self._on_resolved(slot, users.get(0))
                    except Exception as ex:
                        run_on_ui_thread(lambda: BulletinHelper.show_error(f"Parse error: {ex}"))

                send_request(req, RequestCallback(on_username_result))

        except Exception as e:
            run_on_ui_thread(lambda: BulletinHelper.show_error(f"Resolve error: {e}"))

    def _on_resolved(self, slot, user):
        try:
            user_dict = {
                "user_id":    int(user.id),
                "first_name": str(user.first_name or ""),
                "last_name":  str(user.last_name or ""),
                "username":   str(user.username or ""),
            }
            display = _user_display(user)
            self._pending[slot] = user_dict

            def update_ui():
                self.set_setting(f"slot_{slot}_preview", display, reload_settings=True)
                BulletinHelper.show_success(f"Found: {display}")

            run_on_ui_thread(update_ui)
        except Exception as e:
            run_on_ui_thread(lambda: BulletinHelper.show_error(f"Error: {e}"))

    def _confirm_add_user(self, slot):
        user_dict = self._pending.get(slot)
        if not user_dict:
            BulletinHelper.show_error("Search for a user first.")
            return
        uid = int(user_dict["user_id"])
        # Duplicate check
        for s in range(MAX_SLOTS):
            d = self._load_slot(s)
            if d and int(d.get("user_id", 0)) == uid:
                BulletinHelper.show_error(f"Already tracking this user in Slot {s + 1}.")
                return
        slot_data = {
            "user_id":    uid,
            "first_name": user_dict["first_name"],
            "last_name":  user_dict["last_name"],
            "username":   user_dict["username"],
            "added_at":   int(time.time()),
            "messages":   [],
        }
        self._save_slot(slot, slot_data)
        self._reload_tracked()
        self._pending.pop(slot, None)
        self.set_setting(f"slot_{slot}_query",   "", reload_settings=False)
        self.set_setting(f"slot_{slot}_preview", "", reload_settings=False)
        display = _user_display_from_dict(user_dict)
        self.reload_settings()
        BulletinHelper.show_success(f"Now tracking: {display}")

    def _remove_slot(self, slot):
        slot_data = self._load_slot(slot)
        name = _user_display_from_dict(slot_data) if slot_data else f"Slot {slot + 1}"
        self._delete_slot_file(slot)
        self._pending.pop(slot, None)
        self._reload_tracked()
        self.reload_settings()
        BulletinHelper.show_info(f"Stopped tracking: {name}")

    def _clear_all_slots(self):
        for s in range(MAX_SLOTS):
            self._delete_slot_file(s)
        self._pending.clear()
        self._reload_tracked()
        self.reload_settings()
        BulletinHelper.show_info("All slots cleared.")

    def _export_slot(self, slot):
        """Write a human-readable .txt report, show its path in a bulletin."""
        def _do():
            try:
                slot_data = self._load_slot(slot)
                if not slot_data:
                    run_on_ui_thread(lambda: BulletinHelper.show_error("Slot is empty."))
                    return
                name = _user_display_from_dict(slot_data)
                messages = slot_data.get("messages", [])
                lines = [
                    f"SupaStalk export — {name}",
                    f"Exported:  {time.strftime('%Y-%m-%d %H:%M')}",
                    f"Messages:  {len(messages)}",
                    "=" * 60,
                    "",
                ]
                for m in messages:
                    d_str = _fmt_date_long(m.get("date", 0))
                    chat = m.get("chat_title", "") or "Private"
                    text = m.get("text", "")
                    media = m.get("media", "")
                    fwd = m.get("fwd_from", "")
                    reply = m.get("reply_to_id")
                    link = m.get("link", "")
                    lines.append(f"[{d_str}]  {chat}")
                    if fwd:
                        lines.append(f"  ↪ Forwarded from: {fwd}")
                    if reply:
                        lines.append(f"  ↩ Reply to msg #{reply}")
                    if text:
                        for tl in text.splitlines():
                            lines.append(f"  {tl}")
                    if media:
                        lines.append(f"  {media}")
                    if link:
                        lines.append(f"  → {link}")
                    lines.append("")
                export_path = os.path.join(
                    self._storage_dir,
                    f"slot_{slot}_export_{int(time.time())}.txt",
                )
                with open(export_path, "w", encoding="utf-8") as f:
                    f.write("\n".join(lines))
                run_on_ui_thread(lambda: BulletinHelper.show_success(
                    f"Saved: {export_path}"))
            except Exception as e:
                run_on_ui_thread(lambda: BulletinHelper.show_error(f"Export error: {e}"))

        run_on_queue(_do)

    # ── Settings UI ──────────────────────────────────────

    def create_settings(self):
        try:
            active = len(self._tracked)
            items = [
                Header(text="SupaStalk"),
                Switch(
                    key="enabled",
                    text="Enable Tracking",
                    subtext="Intercept messages from tracked users",
                    default=True,
                    icon="msg_eye",
                ),
                Divider(),
                Header(text=f"Tracked Users  ({active} / {MAX_SLOTS} active)"),
            ]
            for slot in range(MAX_SLOTS):
                slot_data = self._load_slot(slot)
                if slot_data:
                    name = _user_display_from_dict(slot_data)
                    count = len(slot_data.get("messages", []))
                    plural = "s" if count != 1 else ""
                    items.append(Text(
                        text=f"Slot {slot + 1}:  {name}  [{count} msg{plural}]",
                        icon="msg_contact",
                        create_sub_fragment=lambda s=slot: self._slot_fragment(s),
                    ))
                else:
                    items.append(Text(
                        text=f"Slot {slot + 1}:  Empty — tap to add",
                        icon="msg_add",
                        create_sub_fragment=lambda s=slot: self._add_user_fragment(s),
                    ))
            items += [
                Divider(),
                Text(
                    text="Clear all slots",
                    icon="msg_delete",
                    red=True,
                    on_click=lambda v: self._clear_all_slots(),
                ),
            ]
            return items
        except Exception as e:
            return [Header(text=f"SupaStalk Error: {e}")]

    # ── Sub-fragment: add user ────────────────────────────

    def _add_user_fragment(self, slot):
        try:
            preview = self.get_setting(f"slot_{slot}_preview", "")
            items = [
                Header(text=f"Add User — Slot {slot + 1}"),
                Input(
                    key=f"slot_{slot}_query",
                    text="User ID or @username",
                    default="",
                    icon="msg_search_filter",
                ),
                Text(
                    text="Search",
                    icon="msg_search",
                    accent=True,
                    on_click=lambda v, s=slot: self._resolve_user_query(s),
                ),
            ]
            if preview:
                items += [
                    Divider(),
                    Text(
                        text=f"Found:  {preview}",
                        icon="msg_contact",
                    ),
                    Text(
                        text="Add to tracking",
                        icon="msg_check",
                        accent=True,
                        on_click=lambda v, s=slot: self._confirm_add_user(s),
                    ),
                ]
            else:
                items.append(Divider(text="Enter ID or @username above, then tap Search"))
            return items
        except Exception as e:
            return [Header(text=f"Error: {e}")]

    # ── Sub-fragment: slot detail ─────────────────────────

    def _slot_fragment(self, slot):
        try:
            slot_data = self._load_slot(slot)
            if not slot_data:
                return [
                    Header(text=f"Slot {slot + 1}"),
                    Text(text="Slot is empty.", icon="msg_info"),
                ]
            name = _user_display_from_dict(slot_data)
            messages = slot_data.get("messages", [])
            count = len(messages)
            added = slot_data.get("added_at", 0)
            added_str = _fmt_date_long(added) if added else "—"

            items = [
                Header(text=f"Slot {slot + 1}: {name}"),
                Text(
                    text=f"Tracking since:  {added_str}",
                    icon="msg_contacts_time_remix",
                ),
                Text(
                    text=f"Export {count} messages to .txt",
                    icon="msg_share",
                    on_click=lambda v, s=slot: self._export_slot(s),
                ),
                Text(
                    text="Remove from tracking",
                    icon="msg_delete",
                    red=True,
                    on_click=lambda v, s=slot: self._remove_slot(s),
                ),
                Divider(),
                Header(text=f"Captured Messages ({count})"),
            ]

            if not messages:
                items.append(Text(
                    text="No messages yet. They appear here as the user sends them.",
                    icon="msg_info",
                ))
            else:
                for msg in messages[:_SHOWN_MESSAGES_LIMIT]:
                    items.append(self._make_message_item(msg))
                if count > _SHOWN_MESSAGES_LIMIT:
                    items.append(Text(
                        text=f"…{count - _SHOWN_MESSAGES_LIMIT} more messages. Export to see all.",
                        icon="msg_info",
                    ))
            return items
        except Exception as e:
            return [Header(text=f"Error: {e}")]

    def _make_message_item(self, msg):
        """Build a single Text item for one captured message."""
        date_str = _fmt_date(msg.get("date", 0))
        chat = msg.get("chat_title", "") or "Private"
        text = msg.get("text", "")
        media = msg.get("media", "")
        fwd = msg.get("fwd_from")

        preview = text.replace("\n", " ")[:55]
        if len(text) > 55:
            preview += "…"
        if not preview:
            preview = media or "(empty)"
        elif media:
            preview = f"{media}  {preview}"
        if fwd:
            preview = f"↪ {preview}"

        label = f"[{date_str}]  {chat}\n{preview}"

        def on_tap(v, m=msg):
            full_text = m.get("text", "")
            media_info = m.get("media", "")
            fwd_info = m.get("fwd_from", "")
            reply_id = m.get("reply_to_id")
            link_info = m.get("link", "")
            chat_name = m.get("chat_title", "") or "Private"
            d = _fmt_date_long(m.get("date", 0))

            parts = [f"[{d}]  {chat_name}"]
            if fwd_info:
                parts.append(f"↪ Forwarded from: {fwd_info}")
            if reply_id:
                parts.append(f"↩ Reply to #{reply_id}")
            if full_text:
                parts.append(full_text[:300])
            if media_info:
                parts.append(media_info)
            if link_info:
                parts.append(f"→ {link_info}")

            BulletinHelper.show_info("\n".join(parts)[:250])

        return Text(
            text=label,
            icon="msg_msgbubble3",
            on_click=on_tap,
        )