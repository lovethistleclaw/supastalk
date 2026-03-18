import os
import json
import time

from base_plugin import BasePlugin, MethodHook, HookResult, HookStrategy
from client_utils import (
    run_on_queue,
    send_request, RequestCallback,
    get_messages_controller,
    get_user_config,
    get_last_fragment,
)
from android_utils import run_on_ui_thread, log
from ui.bulletin import BulletinHelper
from ui.settings import Text, Header, Switch, Divider, Input
from ui.alert import AlertDialogBuilder
from hook_utils import find_class

from org.telegram.messenger import ApplicationLoader
from java import jclass

__id__ = "supastalk"
__name__ = "SupaStalk"
__description__ = "Silently captures messages, edits and deletions from up to 5 users across all shared chats."
__author__ = "@ASTER0N"
__version__ = "1.0.0"
__min_version__ = "12.1.1"

MAX_SLOTS       = 5
MAX_MESSAGES    = 5000
SHOWN_LIMIT     = 60


# ─────────────────────────────────────────────────────────────────────────────
# Helpers
# ─────────────────────────────────────────────────────────────────────────────

def _display(user):
    if user is None:
        return "Unknown"
    parts = []
    try:
        fn = user.first_name
        if fn: parts.append(str(fn))
    except Exception: pass
    try:
        ln = user.last_name
        if ln: parts.append(str(ln))
    except Exception: pass
    name = " ".join(parts).strip()
    if not name:
        try: name = f"ID:{int(user.id)}"
        except Exception: name = "Unknown"
    try:
        un = user.username
        if un: name += f" @{un}"
    except Exception: pass
    return name


def _display_d(d):
    parts = []
    fn = d.get("first_name", "")
    ln = d.get("last_name", "")
    if fn: parts.append(fn)
    if ln: parts.append(ln)
    name = " ".join(parts).strip() or f"ID:{d.get('user_id','?')}"
    un = d.get("username", "")
    if un: name += f" @{un}"
    return name


def _media_label(media):
    if media is None: return None
    cls = type(media).__name__
    MAP = {
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
        return MAP.get(cls, "📦 Media")
    try:
        for attr in media.document.attributes:
            an = type(attr).__name__
            if "Video" in an:
                try:
                    if attr.round_message: return "🎥 Video Message"
                except Exception: pass
                return "🎬 Video"
            if "Audio" in an:
                try:
                    if attr.voice: return "🎤 Voice"
                except Exception: pass
                return "🎵 Audio"
            if "Sticker" in an: return "🎭 Sticker"
            if "Animated" in an: return "👾 GIF"
    except Exception: pass
    return "📎 Document"


def _make_link(peer, msg_id):
    try:
        cls = type(peer).__name__
        if cls == "TL_peerUser":
            return f"tg://openmessage?user_id={int(peer.user_id)}&message_id={msg_id}"
        if cls == "TL_peerChannel":
            return f"https://t.me/c/{int(peer.channel_id)}/{msg_id}"
    except Exception: pass
    return None


def _chat_id(peer):
    try:
        cls = type(peer).__name__
        if cls == "TL_peerUser":    return int(peer.user_id)
        if cls == "TL_peerChat":    return -int(peer.chat_id)
        if cls == "TL_peerChannel": return -(int(peer.channel_id) + 1_000_000_000_000)
    except Exception: pass
    return 0


def _chat_category(peer):
    """Returns 'DM', 'Group', or 'Channel' based on peer type."""
    try:
        cls = type(peer).__name__
        if cls == "TL_peerUser":    return "DM"
        if cls == "TL_peerChat":    return "Group"
        if cls == "TL_peerChannel": return "Channel"
    except Exception: pass
    return "Unknown"


def _entities(entities):
    result = []
    if not entities: return result
    try:
        for e in entities:
            result.append({"type": type(e).__name__, "offset": int(e.offset), "length": int(e.length)})
    except Exception: pass
    return result


def _dt(ts):
    try: return time.strftime("%H:%M  %d.%m", time.localtime(ts))
    except Exception: return str(ts)

def _dtl(ts):
    try: return time.strftime("%H:%M  %d.%m.%Y", time.localtime(ts))
    except Exception: return str(ts)

def _own_id():
    try:
        u = get_user_config().getCurrentUser()
        if u: return int(u.id)
    except Exception: pass
    return None


# ─────────────────────────────────────────────────────────────────────────────
# Xposed hook — fires for every processed message, regardless of UI state
# ─────────────────────────────────────────────────────────────────────────────

class _NewMessageHook(MethodHook):
    """
    Hooks MessagesController.processNewDifferenceParams / didReceivedNotification
    is complex. Instead we hook the method that ultimately stores every
    incoming message: MessagesController.updateInterfaceWithMessages.
    Signature (one of the overloads):
      void updateInterfaceWithMessages(long dialog_id, ArrayList<MessageObject> messages, boolean isForum)
    We use hook_all_methods to catch all overloads.
    """
    def __init__(self, plugin):
        self.plugin = plugin

    def after_hooked_method(self, param):
        try:
            plugin = self.plugin
            if not plugin.get_setting("enabled", True):
                return
            # args[1] should be ArrayList<MessageObject>
            msgs = param.args[1] if param.args and len(param.args) > 1 else None
            if msgs is None:
                return
            try:
                size = msgs.size()
            except Exception:
                return
            for i in range(size):
                try:
                    mo = msgs.get(i)
                    raw = mo.messageOwner
                    if raw is None:
                        continue
                    sender_id = plugin._sender_id(raw)
                    if sender_id is None or sender_id not in plugin._tracked:
                        continue
                    slot = plugin._tracked[sender_id]
                    run_on_queue(lambda s=slot, m=raw: plugin._capture(s, m, "message"))
                except Exception as e:
                    log(f"[SupaStalk] _NewMessageHook inner: {e}")
        except Exception as e:
            log(f"[SupaStalk] _NewMessageHook: {e}")


class _EditMessageHook(MethodHook):
    """
    Hook MessagesController.updateEditedMessage.
    Signature: void updateEditedMessage(long dialog_id, TLRPC.Message msg)
    """
    def __init__(self, plugin):
        self.plugin = plugin

    def after_hooked_method(self, param):
        try:
            plugin = self.plugin
            if not plugin.get_setting("enabled", True):
                return
            if not plugin.get_setting("track_edits", True):
                return
            raw = param.args[1] if param.args and len(param.args) > 1 else None
            if raw is None:
                return
            sender_id = plugin._sender_id(raw)
            if sender_id is None or sender_id not in plugin._tracked:
                return
            slot = plugin._tracked[sender_id]
            run_on_queue(lambda s=slot, m=raw: plugin._capture(s, m, "edit"))
        except Exception as e:
            log(f"[SupaStalk] _EditMessageHook: {e}")


class _DeleteMessageHook(MethodHook):
    """
    Hook MessagesController.deleteMessages.
    Signature: void deleteMessages(ArrayList<Integer> ids, ...)
    We hook all overloads.
    """
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        try:
            plugin = self.plugin
            if not plugin.get_setting("enabled", True):
                return
            if not plugin.get_setting("track_deletes", True):
                return
            ids_obj = param.args[0] if param.args else None
            if ids_obj is None:
                return
            try:
                ids = [int(ids_obj.get(i)) for i in range(ids_obj.size())]
            except Exception:
                return
            if not ids:
                return
            for slot in range(MAX_SLOTS):
                run_on_queue(lambda s=slot, d=ids: plugin._mark_deleted(s, d))
        except Exception as e:
            log(f"[SupaStalk] _DeleteMessageHook: {e}")


# ─────────────────────────────────────────────────────────────────────────────
# Plugin
# ─────────────────────────────────────────────────────────────────────────────

class SupaStalkPlugin(BasePlugin):

    def __init__(self):
        super().__init__()
        self._tracked: dict  = {}   # {uid: slot}
        self._storage_dir    = ""
        self._pending: dict  = {}   # {slot: {user_dict, display}}
        self._query_buf: dict = {}  # {slot: str}  live input buffer

    # ── Lifecycle ─────────────────────────────────────────────────────────────

    def on_plugin_load(self):
        self._storage_dir = self._resolve_storage()
        self._reload_tracked()
        self._setup_hooks()
        log(f"[SupaStalk] loaded, tracking {len(self._tracked)} user(s)")

    def on_plugin_unload(self):
        self._tracked.clear()
        self._pending.clear()
        self._query_buf.clear()

    def _setup_hooks(self):
        try:
            MC = find_class("org.telegram.messenger.MessagesController")
            if MC is None:
                log("[SupaStalk] MessagesController not found!")
                return

            # New messages
            try:
                self.hook_all_methods(MC, "updateInterfaceWithMessages", _NewMessageHook(self))
                log("[SupaStalk] hooked updateInterfaceWithMessages")
            except Exception as e:
                log(f"[SupaStalk] hook new-msg failed: {e}")

            # Edited messages
            try:
                self.hook_all_methods(MC, "updateEditedMessage", _EditMessageHook(self))
                log("[SupaStalk] hooked updateEditedMessage")
            except Exception as e:
                log(f"[SupaStalk] hook edit-msg failed: {e}")

            # Deleted messages
            try:
                self.hook_all_methods(MC, "deleteMessages", _DeleteMessageHook(self))
                log("[SupaStalk] hooked deleteMessages")
            except Exception as e:
                log(f"[SupaStalk] hook delete-msg failed: {e}")

        except Exception as e:
            log(f"[SupaStalk] _setup_hooks error: {e}")

    # ── Sender extraction ──────────────────────────────────────────────────────

    @staticmethod
    def _sender_id(msg):
        try:
            fi = msg.from_id
            if fi is not None and type(fi).__name__ == "TL_peerUser":
                return int(fi.user_id)
        except Exception: pass
        try:
            peer = msg.peer_id
            if peer is not None and type(peer).__name__ == "TL_peerUser":
                return int(peer.user_id)
        except Exception: pass
        return None

    # ── Capture ───────────────────────────────────────────────────────────────

    def _capture(self, slot, msg, kind):
        try:
            sd = self._load_slot(slot)
            if sd is None: return

            peer = None
            chat_id = 0
            chat_title = ""
            category = "Unknown"
            link = None
            try:
                peer = msg.peer_id
                chat_id = _chat_id(peer)
                category = _chat_category(peer)
                link = _make_link(peer, msg.id)
                mc = get_messages_controller()
                cls = type(peer).__name__
                if cls == "TL_peerUser":
                    u = mc.getUser(int(peer.user_id))
                    if u: chat_title = _display(u)
                elif cls == "TL_peerChat":
                    c = mc.getChat(int(peer.chat_id))
                    if c: chat_title = str(c.title or "")
                elif cls == "TL_peerChannel":
                    c = mc.getChat(int(peer.channel_id))
                    if c: chat_title = str(c.title or "")
            except Exception: pass

            text = ""
            try: text = str(msg.message or "")
            except Exception: pass

            media = None
            try: media = _media_label(msg.media)
            except Exception: pass

            ents = []
            try: ents = _entities(msg.entities)
            except Exception: pass

            fwd = None
            try:
                if msg.fwd_from:
                    try:
                        fn = msg.fwd_from.from_name
                        fwd = str(fn) if fn else "forwarded"
                    except Exception: fwd = "forwarded"
            except Exception: pass

            reply_to = None
            try:
                if msg.reply_to:
                    reply_to = int(msg.reply_to.reply_to_msg_id)
            except Exception: pass

            entry = {
                "id":          int(msg.id),
                "kind":        kind,
                "date":        int(msg.date),
                "chat_id":     chat_id,
                "chat_title":  chat_title,
                "category":    category,
                "text":        text,
                "entities":    ents,
                "media":       media,
                "fwd_from":    fwd,
                "reply_to":    reply_to,
                "link":        link,
                "captured_at": int(time.time()),
            }

            msgs = sd.get("messages", [])

            # If it's an edit, try to save original text
            if kind == "edit":
                for m in msgs:
                    if m["id"] == entry["id"] and m["chat_id"] == entry["chat_id"] and m.get("kind") == "message":
                        entry["original_text"] = m.get("text", "")
                        break

            # Dedup for new messages (edits are always appended as separate entries)
            if kind == "message":
                key = (entry["id"], entry["chat_id"])
                if any((m["id"], m["chat_id"]) == key for m in msgs):
                    return

            msgs.insert(0, entry)
            if len(msgs) > MAX_MESSAGES:
                msgs = msgs[:MAX_MESSAGES]
            sd["messages"] = msgs
            self._save_slot(slot, sd)
            log(f"[SupaStalk] slot {slot}: {kind} #{entry['id']} [{category}] {chat_title or 'private'}")
        except Exception as e:
            log(f"[SupaStalk] capture error: {e}")

    def _mark_deleted(self, slot, del_ids):
        try:
            sd = self._load_slot(slot)
            if not sd: return
            msgs = sd.get("messages", [])
            changed = False
            del_set = set(del_ids)
            for m in msgs:
                if m["id"] in del_set and m.get("kind") == "message" and not m.get("deleted"):
                    m["deleted"] = True
                    changed = True
            if changed:
                sd["messages"] = msgs
                self._save_slot(slot, sd)
        except Exception as e:
            log(f"[SupaStalk] mark_deleted error: {e}")

    # ── Storage ───────────────────────────────────────────────────────────────

    def _resolve_storage(self):
        try:
            ctx = ApplicationLoader.applicationContext
            base = None
            try: base = ctx.getExternalFilesDir(None)
            except Exception: pass
            if base is None:
                try: base = ctx.getFilesDir()
                except Exception: pass
            if base:
                path = os.path.join(str(base.getAbsolutePath()), "supastalk")
                os.makedirs(path, exist_ok=True)
                return path
        except Exception: pass
        path = os.path.join(os.environ.get("HOME", os.getcwd()), "supastalk")
        os.makedirs(path, exist_ok=True)
        return path

    def _slot_file(self, slot):
        return os.path.join(self._storage_dir, f"slot_{slot}.json")

    def _load_slot(self, slot):
        p = self._slot_file(slot)
        if not os.path.exists(p): return None
        try:
            with open(p, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception: return None

    def _save_slot(self, slot, data):
        p = self._slot_file(slot)
        tmp = p + ".tmp"
        try:
            with open(tmp, "w", encoding="utf-8") as f:
                json.dump(data, f, ensure_ascii=False, indent=2)
            os.replace(tmp, p)
        except Exception as e:
            log(f"[SupaStalk] save_slot error: {e}")

    def _del_slot(self, slot):
        p = self._slot_file(slot)
        try:
            if os.path.exists(p): os.remove(p)
        except Exception: pass

    def _reload_tracked(self):
        self._tracked.clear()
        for s in range(MAX_SLOTS):
            d = self._load_slot(s)
            if d and "user_id" in d:
                self._tracked[int(d["user_id"])] = s

    # ── User resolution ───────────────────────────────────────────────────────

    def _input_change(self, slot, val):
        self._query_buf[slot] = str(val).strip()

    def _search(self, slot):
        query = self._query_buf.get(slot, "").strip()
        if not query:
            query = self.get_setting(f"slot_{slot}_query", "").strip()
        if not query:
            BulletinHelper.show_error("Type a user ID or @username first.")
            return
        self._pending.pop(slot, None)
        BulletinHelper.show_info("Searching…")
        run_on_queue(lambda: self._do_search(slot, query))

    def _do_search(self, slot, query):
        try:
            TLRPC = jclass("org.telegram.tgnet.TLRPC")
            clean = query.strip().lstrip("@")

            if clean.lstrip("-").isdigit():
                uid = int(clean)
                if _own_id() == uid:
                    run_on_ui_thread(lambda: BulletinHelper.show_error("You can't track yourself."))
                    return
                mc = get_messages_controller()
                try:
                    u = mc.getUser(uid)
                    if u and int(u.id) == uid:
                        self._on_found(slot, u)
                        return
                except Exception: pass
                req = TLRPC.TL_users_getUsers()
                inp = TLRPC.TL_inputUser()
                inp.user_id = uid
                inp.access_hash = 0
                req.id.add(inp)
                def on_r(response, error, _s=slot):
                    if error or not response or len(response) == 0:
                        run_on_ui_thread(lambda: BulletinHelper.show_error(
                            "Not found. Try @username instead."))
                        return
                    try: u = response.get(0)
                    except Exception: u = None
                    if u: self._on_found(_s, u)
                    else: run_on_ui_thread(lambda: BulletinHelper.show_error("User not found."))
                send_request(req, RequestCallback(on_r))

            else:
                req = TLRPC.TL_contacts_resolveUsername()
                req.username = clean
                def on_u(response, error, _s=slot):
                    if error:
                        run_on_ui_thread(lambda: BulletinHelper.show_error(
                            f"Error: {error.text}"))
                        return
                    if not response:
                        run_on_ui_thread(lambda: BulletinHelper.show_error("Not found."))
                        return
                    try:
                        users = response.users
                        if not users or len(users) == 0:
                            # try peer fallback
                            try:
                                p = response.peer
                                if type(p).__name__ == "TL_peerUser":
                                    u = get_messages_controller().getUser(int(p.user_id))
                                    if u:
                                        self._on_found(_s, u)
                                        return
                            except Exception: pass
                            run_on_ui_thread(lambda: BulletinHelper.show_error("User not found."))
                            return
                        u = users.get(0)
                        if u is None:
                            run_on_ui_thread(lambda: BulletinHelper.show_error("User not found."))
                            return
                        if _own_id() == int(u.id):
                            run_on_ui_thread(lambda: BulletinHelper.show_error("Can't track yourself."))
                            return
                        self._on_found(_s, u)
                    except Exception as ex:
                        run_on_ui_thread(lambda: BulletinHelper.show_error(f"Error: {ex}"))
                send_request(req, RequestCallback(on_u))

        except Exception as e:
            run_on_ui_thread(lambda: BulletinHelper.show_error(f"Search error: {e}"))

    def _on_found(self, slot, user):
        try:
            ud = {
                "user_id":    int(user.id),
                "first_name": str(user.first_name or ""),
                "last_name":  str(user.last_name or ""),
                "username":   str(user.username or ""),
            }
            disp = _display(user)
            self._pending[slot] = {"user_dict": ud, "display": disp, "user": user}
            run_on_ui_thread(lambda: BulletinHelper.show_success(
                f"Found: {disp} — tap Add to tracking"))
        except Exception as e:
            run_on_ui_thread(lambda: BulletinHelper.show_error(f"Error: {e}"))

    def _open_profile(self, slot):
        """Open native Telegram ProfileActivity for the found/tracked user."""
        try:
            pending = self._pending.get(slot)
            uid = None
            if pending:
                uid = int(pending["user_dict"]["user_id"])
            else:
                sd = self._load_slot(slot)
                if sd: uid = int(sd.get("user_id", 0))
            if not uid:
                BulletinHelper.show_error("No user to open.")
                return
            def _do():
                try:
                    frag = get_last_fragment()
                    if not frag:
                        BulletinHelper.show_error("No active screen.")
                        return
                    ProfileActivity = jclass("org.telegram.ui.ProfileActivity")
                    args = jclass("android.os.Bundle")()
                    args.putLong("user_id", uid)
                    profile = ProfileActivity(args)
                    frag.presentFragment(profile)
                except Exception as ex:
                    BulletinHelper.show_error(f"Can't open profile: {ex}")
            run_on_ui_thread(_do)
        except Exception as e:
            BulletinHelper.show_error(f"Error: {e}")

    def _confirm_add(self, slot):
        pending = self._pending.get(slot)
        if not pending:
            BulletinHelper.show_error("Search for a user first.")
            return
        ud = pending["user_dict"]
        uid = int(ud["user_id"])
        if _own_id() == uid:
            BulletinHelper.show_error("Can't track yourself.")
            return
        for s in range(MAX_SLOTS):
            d = self._load_slot(s)
            if d and int(d.get("user_id", 0)) == uid:
                BulletinHelper.show_error(f"Already in Slot {s + 1}.")
                return
        sd = {
            "user_id":    uid,
            "first_name": ud["first_name"],
            "last_name":  ud["last_name"],
            "username":   ud["username"],
            "added_at":   int(time.time()),
            "messages":   [],
        }
        self._save_slot(slot, sd)
        self._reload_tracked()
        self._pending.pop(slot, None)
        self._query_buf.pop(slot, None)
        self.set_setting(f"slot_{slot}_query", "", reload_settings=False)
        disp = _display_d(ud)
        self.reload_settings()
        BulletinHelper.show_success(f"Now tracking: {disp}")

    def _remove_slot(self, slot):
        sd = self._load_slot(slot)
        name = _display_d(sd) if sd else f"Slot {slot + 1}"
        self._del_slot(slot)
        self._pending.pop(slot, None)
        self._query_buf.pop(slot, None)
        self._reload_tracked()
        self.reload_settings()
        BulletinHelper.show_info(f"Stopped tracking: {name}")

    def _clear_messages(self, slot):
        sd = self._load_slot(slot)
        if not sd: return
        sd["messages"] = []
        self._save_slot(slot, sd)
        self.reload_settings()
        BulletinHelper.show_info("Messages cleared.")

    def _clear_all(self):
        for s in range(MAX_SLOTS):
            self._del_slot(s)
        self._pending.clear()
        self._query_buf.clear()
        self._reload_tracked()
        self.reload_settings()
        BulletinHelper.show_info("All slots cleared.")

    def _export(self, slot):
        def _do():
            try:
                sd = self._load_slot(slot)
                if not sd:
                    run_on_ui_thread(lambda: BulletinHelper.show_error("Empty slot."))
                    return
                name = _display_d(sd)
                msgs = sd.get("messages", [])
                lines = [
                    f"SupaStalk  —  {name}",
                    f"Exported:   {time.strftime('%Y-%m-%d %H:%M')}",
                    f"Total:      {len(msgs)} entries",
                    "═" * 60, "",
                ]
                for m in msgs:
                    kind = m.get("kind", "message")
                    deleted = m.get("deleted", False)
                    cat = m.get("category", "")
                    chat = m.get("chat_title", "") or "Private"
                    d_str = _dtl(m.get("date", 0))
                    prefix = "🗑 DELETED" if deleted else ("✏️ EDIT" if kind == "edit" else "💬")
                    lines.append(f"{prefix}  [{d_str}]  [{cat}]  {chat}")
                    orig = m.get("original_text")
                    if orig is not None:
                        lines.append(f"  ORIGINAL: {orig}")
                    fwd = m.get("fwd_from")
                    if fwd: lines.append(f"  ↪ Fwd: {fwd}")
                    reply = m.get("reply_to")
                    if reply: lines.append(f"  ↩ Reply #{reply}")
                    text = m.get("text", "")
                    if text:
                        for tl in text.splitlines():
                            lines.append(f"  {tl}")
                    media = m.get("media")
                    if media: lines.append(f"  {media}")
                    lnk = m.get("link")
                    if lnk: lines.append(f"  → {lnk}")
                    lines.append("")
                path = os.path.join(self._storage_dir,
                                    f"slot_{slot}_{int(time.time())}.txt")
                with open(path, "w", encoding="utf-8") as f:
                    f.write("\n".join(lines))
                run_on_ui_thread(lambda: BulletinHelper.show_success(f"Saved: {path}"))
            except Exception as e:
                run_on_ui_thread(lambda: BulletinHelper.show_error(f"Export error: {e}"))
        run_on_queue(_do)

    # ── Chat viewer dialog ────────────────────────────────────────────────────

    def _show_chat_dialog(self, slot):
        """Open an AlertDialog with a native-style scrollable chat view."""
        def _do():
            try:
                sd = self._load_slot(slot)
                if not sd:
                    BulletinHelper.show_error("No data.")
                    return
                frag = get_last_fragment()
                if not frag:
                    BulletinHelper.show_error("No active screen.")
                    return
                activity = frag.getParentActivity()
                if not activity:
                    BulletinHelper.show_error("No activity.")
                    return

                # Android view classes
                Context      = jclass("android.content.Context")
                LinearLayout = jclass("android.widget.LinearLayout")
                ScrollView   = jclass("android.widget.ScrollView")
                TextView     = jclass("android.widget.TextView")
                View         = jclass("android.view.View")
                Color        = jclass("android.graphics.Color")
                GradientDrawable = jclass("android.graphics.drawable.GradientDrawable")
                AndroidU     = jclass("org.telegram.messenger.AndroidUtilities")

                dp = lambda x: AndroidU.dp(x)

                # Theme colours
                try:
                    Theme = jclass("org.telegram.ui.ActionBar.Theme")
                    bg_color  = Theme.getColor("windowBackgroundWhite")
                    txt_color = Theme.getColor("windowBackgroundWhiteBlackText")
                    sub_color = Theme.getColor("windowBackgroundWhiteGrayText2")
                    bubble_out = Theme.getColor("chat_outBubble")
                    bubble_in  = Theme.getColor("chat_inBubble")
                    hint_color = Theme.getColor("windowBackgroundWhiteHintText")
                except Exception:
                    bg_color   = Color.parseColor("#FFFFFF")
                    txt_color  = Color.parseColor("#000000")
                    sub_color  = Color.parseColor("#888888")
                    bubble_out = Color.parseColor("#EFFDDE")
                    bubble_in  = Color.parseColor("#FFFFFF")
                    hint_color = Color.parseColor("#AAAAAA")

                # Root scroll + container
                scroll = ScrollView(activity)
                container = LinearLayout(activity)
                container.setOrientation(LinearLayout.VERTICAL)
                container.setPadding(dp(8), dp(8), dp(8), dp(8))
                scroll.addView(container)

                messages = sd.get("messages", [])
                shown = messages[:SHOWN_LIMIT]

                if not shown:
                    tv = TextView(activity)
                    tv.setText("No messages captured yet.")
                    tv.setTextColor(sub_color)
                    tv.setPadding(dp(16), dp(16), dp(16), dp(16))
                    container.addView(tv)
                else:
                    # Group by date
                    last_day = ""
                    for m in reversed(shown):
                        # Day separator
                        try:
                            day = time.strftime("%d %B %Y", time.localtime(m.get("date", 0)))
                        except Exception:
                            day = ""
                        if day != last_day:
                            last_day = day
                            sep = TextView(activity)
                            sep.setText(day)
                            sep.setTextColor(hint_color)
                            sep.setTextSize(11)
                            sep.setGravity(0x11)  # CENTER
                            sep.setPadding(dp(4), dp(8), dp(4), dp(4))
                            container.addView(sep)

                        kind    = m.get("kind", "message")
                        deleted = m.get("deleted", False)
                        text    = m.get("text", "")
                        media   = m.get("media") or ""
                        fwd     = m.get("fwd_from") or ""
                        orig    = m.get("original_text")
                        cat     = m.get("category", "")
                        chat    = m.get("chat_title", "") or "Private"
                        lnk     = m.get("link") or ""
                        ts      = _dt(m.get("date", 0))

                        # Bubble layout
                        row = LinearLayout(activity)
                        row.setOrientation(LinearLayout.HORIZONTAL)
                        row_lp = LinearLayout.LayoutParams(
                            LinearLayout.LayoutParams.MATCH_PARENT,
                            LinearLayout.LayoutParams.WRAP_CONTENT
                        )
                        row_lp.setMargins(0, dp(2), 0, dp(2))
                        row.setLayoutParams(row_lp)

                        bubble = LinearLayout(activity)
                        bubble.setOrientation(LinearLayout.VERTICAL)
                        bubble.setPadding(dp(10), dp(6), dp(10), dp(6))

                        # Bubble shape + colour
                        drawable = GradientDrawable()
                        drawable.setCornerRadius(dp(12))
                        drawable.setColor(bubble_in)
                        bubble.setBackground(drawable)

                        bubble_lp = LinearLayout.LayoutParams(
                            LinearLayout.LayoutParams.WRAP_CONTENT,
                            LinearLayout.LayoutParams.WRAP_CONTENT
                        )
                        bubble_lp.weight = 0
                        # Max width ~75% of screen
                        try:
                            dm = activity.getResources().getDisplayMetrics()
                            bubble_lp.width = int(dm.widthPixels * 0.75)
                        except Exception:
                            pass

                        # Header: chat label + category
                        header_tv = TextView(activity)
                        label_parts = []
                        if cat: label_parts.append(f"[{cat}]")
                        if chat and chat != "Private": label_parts.append(chat)
                        if deleted: label_parts.append("🗑 DELETED")
                        elif kind == "edit": label_parts.append("✏️ EDITED")
                        if fwd: label_parts.append(f"↪ {fwd}")
                        header_tv.setText("  ".join(label_parts) if label_parts else "Private DM")
                        header_tv.setTextColor(sub_color)
                        header_tv.setTextSize(11)
                        bubble.addView(header_tv)

                        # Original text (for edits)
                        if orig is not None:
                            orig_tv = TextView(activity)
                            orig_tv.setText(f"~~{orig}~~" if orig else "(empty)")
                            orig_tv.setTextColor(hint_color)
                            orig_tv.setTextSize(13)
                            bubble.addView(orig_tv)

                        # Main content
                        content_parts = []
                        if text: content_parts.append(text)
                        if media: content_parts.append(media)
                        if lnk: content_parts.append(f"→ {lnk}")
                        content = "\n".join(content_parts) if content_parts else "(no text)"

                        content_tv = TextView(activity)
                        content_tv.setText(content)
                        content_tv.setTextColor(txt_color if not deleted else hint_color)
                        content_tv.setTextSize(14)
                        content_tv.setTextIsSelectable(True)
                        bubble.addView(content_tv)

                        # Timestamp
                        time_tv = TextView(activity)
                        time_tv.setText(ts)
                        time_tv.setTextColor(hint_color)
                        time_tv.setTextSize(10)
                        time_tv.setGravity(0x05)  # RIGHT
                        bubble.addView(time_tv)

                        # Spacer to push bubble left
                        spacer = View(activity)
                        spacer_lp = LinearLayout.LayoutParams(0, 1, 1.0)
                        spacer.setLayoutParams(spacer_lp)

                        row.addView(bubble, bubble_lp)
                        row.addView(spacer)
                        container.addView(row)

                    if len(messages) > SHOWN_LIMIT:
                        more_tv = TextView(activity)
                        more_tv.setText(f"…{len(messages) - SHOWN_LIMIT} more. Export to see all.")
                        more_tv.setTextColor(hint_color)
                        more_tv.setTextSize(11)
                        more_tv.setGravity(0x11)
                        more_tv.setPadding(dp(8), dp(8), dp(8), dp(8))
                        container.addView(more_tv)

                name = _display_d(sd)
                count = len(messages)

                builder = AlertDialogBuilder(activity)
                builder.set_title(f"{name}  ({count})")
                builder.set_view(scroll, -2)
                builder.set_positive_button("Close", lambda b, w: b.dismiss())
                builder.set_negative_button(
                    "Export",
                    lambda b, w, s=slot: (b.dismiss(), self._export(s))
                )
                builder.show()

            except Exception as e:
                log(f"[SupaStalk] chat dialog error: {e}")
                BulletinHelper.show_error(f"Chat view error: {e}")

        run_on_ui_thread(_do)

    # ── Settings UI ───────────────────────────────────────────────────────────

    def create_settings(self):
        try:
            active = len(self._tracked)
            total_msgs = sum(
                len((self._load_slot(s) or {}).get("messages", []))
                for s in range(MAX_SLOTS)
            )
            items = [
                Header(text="SupaStalk"),
                Switch(
                    key="enabled",
                    text="Enable Tracking",
                    subtext="Capture incoming messages from tracked users",
                    default=True,
                    icon="msg_eye",
                    # No on_change — avoid accidental reload loop
                ),
                Switch(
                    key="track_edits",
                    text="Track Edits",
                    subtext="Record original text before edits",
                    default=True,
                    icon="msg_edit",
                ),
                Switch(
                    key="track_deletes",
                    text="Track Deletions",
                    subtext="Flag messages that get deleted",
                    default=True,
                    icon="msg_delete",
                ),
                Divider(text=f"{active} / {MAX_SLOTS} slots active  •  {total_msgs} messages total"),
            ]

            for slot in range(MAX_SLOTS):
                sd = self._load_slot(slot)
                if sd:
                    name = _display_d(sd)
                    msgs = sd.get("messages", [])
                    count = len(msgs)
                    edits = sum(1 for m in msgs if m.get("kind") == "edit")
                    dels  = sum(1 for m in msgs if m.get("deleted"))
                    cats  = {}
                    for m in msgs:
                        c = m.get("category", "?")
                        cats[c] = cats.get(c, 0) + 1
                    cat_str = "  ".join(f"{k}:{v}" for k, v in cats.items()) if cats else ""
                    detail = f"{count} msgs"
                    if edits: detail += f"  •  {edits} edits"
                    if dels:  detail += f"  •  {dels} deleted"
                    if cat_str: detail += f"  •  {cat_str}"
                    items.append(Text(
                        text=f"● {name}",
                        icon="msg_contact",
                        create_sub_fragment=lambda s=slot: self._slot_page(s),
                    ))
                else:
                    items.append(Text(
                        text=f"○ Slot {slot + 1} — empty",
                        icon="msg_add",
                        create_sub_fragment=lambda s=slot: self._add_page(s),
                    ))

            items += [
                Divider(),
                Text(
                    text="🗑  Clear all slots",
                    icon="msg_delete",
                    red=True,
                    on_click=lambda v: self._clear_all(),
                ),
            ]
            return items
        except Exception as e:
            return [Header(text=f"SupaStalk — Error: {e}")]

    # ── Sub-page: add user ─────────────────────────────────────────────────────

    def _add_page(self, slot):
        try:
            pending = self._pending.get(slot)
            items = [
                Header(text=f"Add User — Slot {slot + 1}"),
                Input(
                    key=f"slot_{slot}_query",
                    text="User ID or @username",
                    default=self._query_buf.get(slot, ""),
                    icon="msg_search_filter",
                    on_change=lambda val, s=slot: self._input_change(s, val),
                ),
                Text(
                    text="🔍  Search",
                    icon="msg_search",
                    accent=True,
                    on_click=lambda v, s=slot: self._search(s),
                ),
            ]
            if pending:
                disp = pending.get("display", "?")
                items += [
                    Divider(),
                    Text(
                        text=f"✅  {disp}",
                        icon="msg_contact",
                    ),
                    Text(
                        text="👤  Open profile",
                        icon="msg_contact",
                        accent=True,
                        on_click=lambda v, s=slot: self._open_profile(s),
                    ),
                    Text(
                        text="➕  Add to tracking",
                        icon="msg_check",
                        accent=True,
                        on_click=lambda v, s=slot: self._confirm_add(s),
                    ),
                ]
            else:
                items.append(Divider(text="Type ID / @username → Search → Add"))
            return items
        except Exception as e:
            return [Header(text=f"Error: {e}")]

    # ── Sub-page: slot detail ──────────────────────────────────────────────────

    def _slot_page(self, slot):
        try:
            sd = self._load_slot(slot)
            if not sd:
                return [Header(text="Empty"), Text(text="Slot is empty.", icon="msg_info")]

            name   = _display_d(sd)
            msgs   = sd.get("messages", [])
            count  = len(msgs)
            added  = sd.get("added_at", 0)

            # Stats per category
            cats: dict = {}
            edits = dels = 0
            for m in msgs:
                c = m.get("category", "?")
                cats[c] = cats.get(c, 0) + 1
                if m.get("kind") == "edit": edits += 1
                if m.get("deleted"): dels += 1

            cat_lines = "  •  ".join(f"{k}: {v}" for k, v in cats.items())

            items = [
                Header(text=name),
                Text(
                    text=f"Tracking since  {_dtl(added) if added else '—'}",
                    icon="msg_contacts_time_remix",
                ),
                Text(
                    text=f"📊  {count} total  •  {edits} edits  •  {dels} deleted",
                    icon="msg_info",
                ),
            ]
            if cat_lines:
                items.append(Text(text=f"📂  {cat_lines}", icon="msg_info"))

            items += [
                Divider(),
                Text(
                    text="💬  Open chat view",
                    icon="msg_msgbubble3",
                    accent=True,
                    on_click=lambda v, s=slot: self._show_chat_dialog(s),
                ),
                Text(
                    text="👤  Open profile",
                    icon="msg_contact",
                    on_click=lambda v, s=slot: self._open_profile(s),
                ),
                Text(
                    text="📋  Export to .txt",
                    icon="msg_share",
                    on_click=lambda v, s=slot: self._export(s),
                ),
                Text(
                    text="🗑  Clear messages",
                    icon="msg_delete",
                    on_click=lambda v, s=slot: self._clear_messages(s),
                ),
                Text(
                    text="Remove from tracking",
                    icon="msg_leave",
                    red=True,
                    on_click=lambda v, s=slot: self._remove_slot(s),
                ),
                Divider(),
                Header(text=f"Last {min(count, SHOWN_LIMIT)} messages"),
            ]

            if not msgs:
                items.append(Text(text="No messages yet.", icon="msg_info"))
            else:
                for m in msgs[:SHOWN_LIMIT]:
                    items.append(self._msg_item(m))
                if count > SHOWN_LIMIT:
                    items.append(Text(
                        text=f"…{count - SHOWN_LIMIT} more — use Chat View or Export",
                        icon="msg_info",
                    ))
            return items
        except Exception as e:
            return [Header(text=f"Error: {e}")]

    def _msg_item(self, m):
        kind    = m.get("kind", "message")
        deleted = m.get("deleted", False)
        date_s  = _dt(m.get("date", 0))
        cat     = m.get("category", "")
        chat    = m.get("chat_title", "") or "Private"
        text    = m.get("text", "")
        media   = m.get("media", "")
        fwd     = m.get("fwd_from")

        preview = text.replace("\n", " ")[:50]
        if len(text) > 50: preview += "…"
        if not preview: preview = media or "(no text)"
        elif media: preview = f"{media}  {preview}"
        if fwd: preview = f"↪ {preview}"

        badge = ""
        if deleted: badge = "🗑 "
        elif kind == "edit": badge = "✏️ "

        cat_tag = f"[{cat}] " if cat else ""
        label = f"{badge}{cat_tag}[{date_s}]  {chat}\n{preview}"

        def on_tap(v, msg=m):
            full   = msg.get("text", "")
            med    = msg.get("media", "")
            fwd_i  = msg.get("fwd_from", "")
            rep    = msg.get("reply_to")
            lnk    = msg.get("link", "")
            orig   = msg.get("original_text")
            ch     = msg.get("chat_title", "") or "Private"
            ca     = msg.get("category", "")
            d      = _dtl(msg.get("date", 0))
            k      = msg.get("kind", "message")
            dl     = msg.get("deleted", False)

            parts = []
            if dl:   parts.append("🗑 DELETED")
            elif k == "edit": parts.append("✏️ EDITED")
            if ca: parts.append(f"[{ca}]")
            parts.append(f"[{d}]  {ch}")
            if fwd_i: parts.append(f"↪ Fwd: {fwd_i}")
            if rep:   parts.append(f"↩ Reply #{rep}")
            if orig is not None:
                parts.append(f"ORIGINAL: {orig[:120]}")
                parts.append(f"NEW:      {full[:120]}")
            elif full: parts.append(full[:300])
            if med:  parts.append(med)
            if lnk:  parts.append(f"→ {lnk}")

            BulletinHelper.show_info("\n".join(parts)[:250])

        return Text(text=label, icon="msg_msgbubble3", on_click=on_tap)