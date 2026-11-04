name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.11"}
      - run: pip install -r requirements-dev.txt
      - run: python -m pytest -q

      """Entry point: Slack Bolt (Socket Mode) + Claude. Run: python app.py"""
import logging

import anthropic
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler

from bot import flow, responder
from bot.config import Config

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")


def build_app(cfg: Config) -> App:
    app = App(token=cfg.slack_bot_token)
    claude = anthropic.Anthropic(api_key=cfg.anthropic_api_key)
    bot_id = app.client.auth_test()["user_id"]

    def ask(system: str, user_text: str) -> str:
        return responder.answer(claude, cfg.model, system, user_text, cfg.max_tokens)

    @app.event("message")
    def on_message(event, client):
        try:
            logging.getLogger("fpus_dx_bot").info("message -> %s", flow.handle_message(cfg, client, bot_id, event, ask))
        except Exception:
            logging.getLogger("fpus_dx_bot").exception("message handling failed")

    @app.event("reaction_added")
    def on_reaction(event, client):
        try:
            logging.getLogger("fpus_dx_bot").info("reaction -> %s", flow.handle_reaction(cfg, client, bot_id, event))
        except Exception:
            logging.getLogger("fpus_dx_bot").exception("reaction handling failed")

    @app.event("app_mention")
    def on_mention():
        pass

    return app


if __name__ == "__main__":
    config = Config()
    config.validate()
    SocketModeHandler(build_app(config), config.slack_app_token).start()
    """Settings from environment variables only. No secret is ever stored in code."""
import os
from dataclasses import dataclass, field
from typing import List


def _list(name: str) -> List[str]:
    return [x.strip() for x in os.environ.get(name, "").split(",") if x.strip()]


@dataclass(frozen=True)
class Config:
    slack_bot_token: str = field(default_factory=lambda: os.environ.get("SLACK_BOT_TOKEN", ""))
    slack_app_token: str = field(default_factory=lambda: os.environ.get("SLACK_APP_TOKEN", ""))
    anthropic_api_key: str = field(default_factory=lambda: os.environ.get("ANTHROPIC_API_KEY", ""))
    model: str = field(default_factory=lambda: os.environ.get("ANTHROPIC_MODEL", "claude-sonnet-4-5"))
    max_tokens: int = field(default_factory=lambda: int(os.environ.get("MAX_TOKENS", "1500")))
    channels: List[str] = field(default_factory=lambda: _list("ALLOWED_CHANNELS"))
    reviewers: List[str] = field(default_factory=lambda: _list("REVIEWER_USER_IDS"))
    handoff: str = field(default_factory=lambda: os.environ.get("HANDOFF", "<#C01RBHX6UP2>"))
    trial_banner: bool = field(default_factory=lambda: os.environ.get("TRIAL_BANNER", "true").lower() == "true")
    dry_run: bool = field(default_factory=lambda: os.environ.get("DRY_RUN", "false").lower() == "true")
    max_bot_replies: int = field(default_factory=lambda: int(os.environ.get("MAX_BOT_REPLIES", "3")))
    log_path: str = field(default_factory=lambda: os.environ.get("REVIEW_LOG", "logs/review_log.jsonl"))

    def validate(self) -> None:
        missing = [n for n, v in (("SLACK_BOT_TOKEN", self.slack_bot_token), ("SLACK_APP_TOKEN", self.slack_app_token),
                                  ("ANTHROPIC_API_KEY", self.anthropic_api_key), ("ALLOWED_CHANNELS", self.channels)) if not v]
        if missing:
            raise SystemExit(f"Missing required settings: {', '.join(missing)} (see .env.example)")

            """Master knowledge: split on '## ' headings, searched by keyword/bigram overlap.
The file is the ONLY source the bot may answer from."""
import re
from pathlib import Path
from typing import Dict, List, Tuple

KB_PATH = Path(__file__).parent / "data" / "master_knowledge.md"
_SECTIONS: List[Tuple[str, str]] = []


def _load() -> List[Tuple[str, str]]:
    global _SECTIONS
    if not _SECTIONS and KB_PATH.exists():
        text = KB_PATH.read_text(encoding="utf-8")
        parts = re.split(r"(?m)^(?=## )", text)
        _SECTIONS = [(p.splitlines()[0].lstrip("# ").strip(), p) for p in parts if p.startswith("## ")]
    return _SECTIONS


def _grams(s: str) -> set:
    s = re.sub(r"\s+", "", s.lower())
    return set(re.findall(r"[a-z0-9_\-]{2,}", s)) | {s[i:i + 2] for i in range(len(s) - 1)}


def available() -> bool:
    return bool(_load())


def search(query: str, top: int = 4, chunk: int = 1800) -> List[Dict[str, str]]:
    q = _grams(query)
    if not q:
        return []
    scored = []
    for title, body in _load():
        for para in re.split(r"\n\s*\n", body):
            if len(para.strip()) < 8:
                continue
            score = len(q & _grams(para + title)) / (len(q) ** 0.5)
            if score > 0:
                scored.append((score, title, para.strip()))
    scored.sort(key=lambda x: -x[0])
    out, seen = [], set()
    for _, title, para in scored:
        if para not in seen:
            seen.add(para)
            out.append({"section": title, "excerpt": para[:chunk]})
        if len(out) >= top:
            break
    return out


def read_section(key: str, limit: int = 6000) -> str:
    key = (key or "").strip()
    for title, body in _load():
        if key and (title.startswith(f"{key}.") or key in title):
            return body[:limit] + (" …(truncated)" if len(body) > limit else "")
    return "該当する章が見つかりません。"

    """Mechanical reply policy: fixed frame, silence sentinel, sanitising. Pure functions."""
import re
from typing import Optional

TRIAL_BANNER = ("🧪 *試運転中* ｜このFPuS自動回答機能は現在試験運用中です。"
                "内容は参考情報としてご活用いただき、正式な確認は引き続きDXチームまでお願いします。")
HEADING = "🤖 DX Assistant"
FOOTER = "※自動生成です。回答後にDXチームが内容を確認します。"
CORRECTION_NOTICE = ("先ほどの自動回答に誤りがあるとの指摘がありました。上記の内容は参考にせず、"
                     "DXチームの回答をお待ちください。このスレッドでの自動回答は停止します。")
CORRECTION_MARKER = "このスレッドでの自動回答は停止します"

_SENTINEL = re.compile(r"^\s*[*_~`]*\s*NO_USEFUL_ANSWER\s*[*_~`]*\s*:?\s*(.*)", re.I | re.S)
_BAD_URL = re.compile(r"https?://[^\s)]*(?:outlook\.office(?:365)?\.com|-my\.sharepoint\.com/personal)[^\s)]*", re.I)
_SECRET = re.compile(r"(xox[abprs]-[\w-]+|sk-ant-[\w-]+|AKIA[0-9A-Z]{16})")


def silence_reason(text: str) -> Optional[str]:
    m = _SENTINEL.match(text or "")
    if not m:
        return None
    return (re.sub(r"\s+", " ", m.group(1)).strip()[:200]) or "no useful answer"


def sanitize(text: str) -> str:
    text = (text or "").replace("&lt;", "<").replace("&gt;", ">")
    text = _BAD_URL.sub("(社内リンクは省略)", text)
    return _SECRET.sub("[redacted]", text)


def frame(body: str, asker_id: str = "", banner: bool = True) -> str:
    parts = [f"<@{asker_id}>"] if asker_id else []
    if banner:
        parts.append(TRIAL_BANNER)
    parts += [HEADING, sanitize(body).strip(), FOOTER]
    return "\n\n".join(parts)


def correction(asker_id: str = "") -> str:
    return frame(CORRECTION_NOTICE, asker_id, banner=False)

    """Claude tool-use loop over the master knowledge."""
import json
from typing import Callable, List

from . import kb

TOOLS = [
    {"name": "search_knowledge",
     "description": "DXマスターナレッジを検索する。query は投稿者の困りごと（エラー文言・画面名を残す）。"
                    "結果は {section, excerpt} の配列（最大4件）。空配列は「ナレッジに無い」。回答の唯一の根拠。",
     "input_schema": {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]}},
    {"name": "read_section",
     "description": "ナレッジの1章を丸ごと読む。section は章番号（例 \"10\"）または題名の一部。",
     "input_schema": {"type": "object", "properties": {"section": {"type": "string"}}, "required": ["section"]}},
]
MAX_ROUNDS = 6
MAX_TOOL_CHARS = 6000


def _run_tool(name: str, args: dict) -> str:
    try:
        if name == "search_knowledge":
            out = json.dumps(kb.search(str(args.get("query", ""))), ensure_ascii=False)
        elif name == "read_section":
            out = kb.read_section(str(args.get("section", "")))
        else:
            out = "unknown tool"
    except Exception as exc:
        out = f"tool error: {type(exc).__name__}"
    return out[:MAX_TOOL_CHARS]


def answer(client, model: str, system: str, user_text: str, max_tokens: int = 1500,
           tool_runner: Callable[[str, dict], str] = _run_tool) -> str:
    messages: List[dict] = [{"role": "user", "content": user_text}]
    for _ in range(MAX_ROUNDS):
        resp = client.messages.create(model=model, max_tokens=max_tokens, system=system, tools=TOOLS, messages=messages)
        if resp.stop_reason != "tool_use":
            return "".join(b.text for b in resp.content if getattr(b, "type", "") == "text").strip()
        messages.append({"role": "assistant", "content": resp.content})
        results = [{"type": "tool_result", "tool_use_id": b.id, "content": tool_runner(b.name, b.input)}
                   for b in resp.content if getattr(b, "type", "") == "tool_use"]
        messages.append({"role": "user", "content": results})
    return ""

    """Claude tool-use loop over the master knowledge."""
import json
from typing import Callable, List

from . import kb

TOOLS = [
    {"name": "search_knowledge",
     "description": "DXマスターナレッジを検索する。query は投稿者の困りごと（エラー文言・画面名を残す）。"
                    "結果は {section, excerpt} の配列（最大4件）。空配列は「ナレッジに無い」。回答の唯一の根拠。",
     "input_schema": {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]}},
    {"name": "read_section",
     "description": "ナレッジの1章を丸ごと読む。section は章番号（例 \"10\"）または題名の一部。",
     "input_schema": {"type": "object", "properties": {"section": {"type": "string"}}, "required": ["section"]}},
]
MAX_ROUNDS = 6
MAX_TOOL_CHARS = 6000


def _run_tool(name: str, args: dict) -> str:
    try:
        if name == "search_knowledge":
            out = json.dumps(kb.search(str(args.get("query", ""))), ensure_ascii=False)
        elif name == "read_section":
            out = kb.read_section(str(args.get("section", "")))
        else:
            out = "unknown tool"
    except Exception as exc:
        out = f"tool error: {type(exc).__name__}"
    return out[:MAX_TOOL_CHARS]


def answer(client, model: str, system: str, user_text: str, max_tokens: int = 1500,
           tool_runner: Callable[[str, dict], str] = _run_tool) -> str:
    messages: List[dict] = [{"role": "user", "content": user_text}]
    for _ in range(MAX_ROUNDS):
        resp = client.messages.create(model=model, max_tokens=max_tokens, system=system, tools=TOOLS, messages=messages)
        if resp.stop_reason != "tool_use":
            return "".join(b.text for b in resp.content if getattr(b, "type", "") == "text").strip()
        messages.append({"role": "assistant", "content": resp.content})
        results = [{"type": "tool_result", "tool_use_id": b.id, "content": tool_runner(b.name, b.input)}
                   for b in resp.content if getattr(b, "type", "") == "tool_use"]
        messages.append({"role": "user", "content": results})
    return ""

    """Claude tool-use loop over the master knowledge."""
import json
from typing import Callable, List

from . import kb

TOOLS = [
    {"name": "search_knowledge",
     "description": "DXマスターナレッジを検索する。query は投稿者の困りごと（エラー文言・画面名を残す）。"
                    "結果は {section, excerpt} の配列（最大4件）。空配列は「ナレッジに無い」。回答の唯一の根拠。",
     "input_schema": {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]}},
    {"name": "read_section",
     "description": "ナレッジの1章を丸ごと読む。section は章番号（例 \"10\"）または題名の一部。",
     "input_schema": {"type": "object", "properties": {"section": {"type": "string"}}, "required": ["section"]}},
]
MAX_ROUNDS = 6
MAX_TOOL_CHARS = 6000


def _run_tool(name: str, args: dict) -> str:
    try:
        if name == "search_knowledge":
            out = json.dumps(kb.search(str(args.get("query", ""))), ensure_ascii=False)
        elif name == "read_section":
            out = kb.read_section(str(args.get("section", "")))
        else:
            out = "unknown tool"
    except Exception as exc:
        out = f"tool error: {type(exc).__name__}"
    return out[:MAX_TOOL_CHARS]


def answer(client, model: str, system: str, user_text: str, max_tokens: int = 1500,
           tool_runner: Callable[[str, dict], str] = _run_tool) -> str:
    messages: List[dict] = [{"role": "user", "content": user_text}]
    for _ in range(MAX_ROUNDS):
        resp = client.messages.create(model=model, max_tokens=max_tokens, system=system, tools=TOOLS, messages=messages)
        if resp.stop_reason != "tool_use":
            return "".join(b.text for b in resp.content if getattr(b, "type", "") == "text").strip()
        messages.append({"role": "assistant", "content": resp.content})
        results = [{"type": "tool_result", "tool_use_id": b.id, "content": tool_runner(b.name, b.input)}
                   for b in resp.content if getattr(b, "type", "") == "tool_use"]
        messages.append({"role": "user", "content": results})
    return ""

    """Decision flow, independent of Bolt so it is unit-testable."""
import json
import logging
import threading
import time
from pathlib import Path
from typing import Callable

from . import policy, prompts
from .config import Config

log = logging.getLogger("fpus_dx_bot")
NEGATIVE_REACTIONS = {"x", "negative_squared_cross_mark"}
_seen = set()
_lock = threading.Lock()


def _claim(key) -> bool:
    with _lock:
        if key in _seen:
            return False
        _seen.add(key)
        if len(_seen) > 5000:
            _seen.clear()
        return True


def _record(cfg: Config, kind: str, **fields) -> None:
    try:
        p = Path(cfg.log_path)
        p.parent.mkdir(parents=True, exist_ok=True)
        with p.open("a", encoding="utf-8") as f:
            f.write(json.dumps({"t": int(time.time()), "kind": kind, **fields}, ensure_ascii=False) + "\n")
    except OSError:
        log.warning("could not write review log")


def _transcript(msgs, bot_id: str) -> str:
    lines = []
    for m in msgs:
        who = "Bot" if m.get("user") == bot_id else f"<@{m.get('user', 'unknown')}>"
        lines.append(f"{who}: {m.get('text', '')}")
    return "\n".join(lines)


def _post(cfg: Config, slack, channel: str, thread_ts: str, text: str) -> bool:
    if cfg.dry_run:
        log.info("DRY_RUN reply to %s/%s:\n%s", channel, thread_ts, text)
        return True
    try:
        slack.chat_postMessage(channel=channel, thread_ts=thread_ts, text=text)
        return True
    except Exception as exc:
        log.error("post failed (%s) channel=%s", type(exc).__name__, channel)
        _record(cfg, "post_failed", channel=channel, thread_ts=thread_ts, error=type(exc).__name__)
        return False


def handle_message(cfg: Config, slack, bot_id: str, event: dict, ask: Callable[[str, str], str]) -> str:
    if event.get("channel") not in cfg.channels:
        return "ignored:channel"
    if event.get("subtype") not in (None, "thread_broadcast", "file_share") or event.get("bot_id") or event.get("user") == bot_id:
        return "ignored:not_human_message"
    ts, channel = event["ts"], event["channel"]
    thread_ts = event.get("thread_ts") or ts
    if not _claim((channel, ts)):
        return "ignored:duplicate"

    msgs = slack.conversations_replies(channel=channel, ts=thread_ts, limit=50).get("messages", [])
    bot_msgs = [m for m in msgs if m.get("user") == bot_id]
    if any(policy.CORRECTION_MARKER in m.get("text", "") for m in bot_msgs):
        return "ignored:corrected_thread"
    is_root = thread_ts == ts
    root_user = msgs[0].get("user", "") if msgs else event.get("user", "")
    if is_root and bot_msgs:
        return "ignored:already_replied"
    if not is_root and (not bot_msgs or event.get("user") != root_user):
        return "ignored:not_a_follow_up"
    if len(bot_msgs) >= cfg.max_bot_replies:
        _record(cfg, "silent", channel=channel, thread_ts=thread_ts, reason="max_bot_replies")
        return "silent:max_bot_replies"

    body = ask(prompts.system_prompt(cfg.handoff),
               prompts.user_message(_transcript(msgs, bot_id), len(bot_msgs), cfg.max_bot_replies))
    reason = policy.silence_reason(body)
    if reason is not None or not body.strip():
        _record(cfg, "silent", channel=channel, thread_ts=thread_ts, reason=reason or "empty")
        return f"silent:{reason or 'empty'}"
    text = policy.frame(body, root_user, cfg.trial_banner)
    ok = _post(cfg, slack, channel, thread_ts, text)
    if ok:
        _record(cfg, "reply", channel=channel, thread_ts=thread_ts)
    return "replied" if ok else "ignored:post_failed"


def handle_reaction(cfg: Config, slack, bot_id: str, event: dict) -> str:
    if event.get("reaction") not in NEGATIVE_REACTIONS:
        return "ignored:reaction"
    if cfg.reviewers and event.get("user") not in cfg.reviewers:
        return "ignored:not_reviewer"
    item = event.get("item", {})
    if item.get("type") != "message" or item.get("channel") not in cfg.channels:
        return "ignored:item"
    msgs = slack.conversations_replies(channel=item["channel"], ts=item["ts"], limit=50, inclusive=True).get("messages", [])
    target = next((m for m in msgs if m.get("ts") == item["ts"]), None)
    if not target or target.get("user") != bot_id:
        return "ignored:not_bot_message"
    thread_ts = target.get("thread_ts") or target["ts"]
    thread = slack.conversations_replies(channel=item["channel"], ts=thread_ts, limit=50).get("messages", [])
    if any(policy.CORRECTION_MARKER in m.get("text", "") for m in thread):
        return "ignored:already_corrected"
    asker = thread[0].get("user", "") if thread else ""
    ok = _post(cfg, slack, item["channel"], thread_ts, policy.correction(asker))
    _record(cfg, "correction", channel=item["channel"], thread_ts=thread_ts)
    return "corrected" if ok else "ignored:post_failed"
