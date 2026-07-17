import { useState, useEffect, useRef, useCallback } from "react";
import { Send, Smile, Users, Newspaper, Lock, X } from "lucide-react";

const COLORS = [
  { name: "blue", hex: "#2F6FED" },
  { name: "coral", hex: "#FF6552" },
  { name: "amber", hex: "#E2A400" },
  { name: "teal", hex: "#0F9B8E" },
  { name: "violet", hex: "#7C5CFC" },
];

const EMOJIS = ["❤️", "😂", "👍", "😮", "🔥"];
const ROOM_KEY = "breakit:messages:v1";
const PROFILE_KEY = "breakit:profile:v1";
const LAST_SEEN_NEWS_KEY = "breakit:lastSeenNews:v1";
const MAX_MESSAGES = 150;
const POLL_MS = 3500;
const NEWS_USER = "BreakitNews";
const NEWS_PASSWORD = "0405";

function timeFmt(ts) {
  try {
    return new Date(ts).toLocaleTimeString("ja-JP", { hour: "2-digit", minute: "2-digit" });
  } catch {
    return "";
  }
}

function uid() {
  return Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
}

// Signature: jagged "crack" divider, evokes the brand's "break" motif
function CrackDivider() {
  return (
    <svg viewBox="0 0 400 14" preserveAspectRatio="none" className="w-full h-[10px]" aria-hidden="true">
      <polyline
        points="0,7 30,7 42,1 56,13 70,3 86,10 104,4 122,11 140,2 160,9 180,5 200,12 220,3 240,9 262,2 284,11 306,4 328,10 350,3 372,9 400,7"
        fill="none"
        stroke="url(#crackGrad)"
        strokeWidth="2"
        strokeLinecap="round"
        strokeLinejoin="round"
      />
      <defs>
        <linearGradient id="crackGrad" x1="0" y1="0" x2="400" y2="0" gradientUnits="userSpaceOnUse">
          <stop offset="0%" stopColor="#FF6552" />
          <stop offset="50%" stopColor="#E2A400" />
          <stop offset="100%" stopColor="#2F6FED" />
        </linearGradient>
      </defs>
    </svg>
  );
}

function Avatar({ name, color, size = 34 }) {
  const initial = (name || "?").trim().charAt(0).toUpperCase() || "?";
  return (
    <div
      style={{
        width: size,
        height: size,
        background: color,
        borderRadius: "10px 10px 10px 3px",
        fontFamily: "'Space Grotesk', sans-serif",
        fontSize: size * 0.42,
      }}
      className="flex items-center justify-center text-white font-bold shrink-0 shadow-sm"
    >
      {initial}
    </div>
  );
}

export default function Breakit() {
  const [profile, setProfile] = useState(null);
  const [profileLoading, setProfileLoading] = useState(true);
  const [nameInput, setNameInput] = useState("");
  const [colorInput, setColorInput] = useState(COLORS[0].hex);

  const [messages, setMessages] = useState([]);
  const [loadingFeed, setLoadingFeed] = useState(true);
  const [feedError, setFeedError] = useState(false);
  const [text, setText] = useState("");
  const [pickerFor, setPickerFor] = useState(null);
  const [sending, setSending] = useState(false);

  const [adminOpen, setAdminOpen] = useState(false);
  const [adminUnlocked, setAdminUnlocked] = useState(false);
  const [pwInput, setPwInput] = useState("");
  const [pwError, setPwError] = useState(false);
  const [newsText, setNewsText] = useState("");
  const [postingNews, setPostingNews] = useState(false);
  const [toast, setToast] = useState(null);

  const scrollRef = useRef(null);
  const bottomRef = useRef(null);
  const pollRef = useRef(null);
  const lastSeenNewsRef = useRef(0);
  const newsInitRef = useRef(false);

  // load personal profile
  useEffect(() => {
    (async () => {
      try {
        const res = await window.storage.get(PROFILE_KEY, false);
        if (res && res.value) {
          setProfile(JSON.parse(res.value));
        }
      } catch {
        // no profile yet
      } finally {
        setProfileLoading(false);
      }
      try {
        const res = await window.storage.get(LAST_SEEN_NEWS_KEY, false);
        if (res && res.value) lastSeenNewsRef.current = Number(res.value) || 0;
      } catch {
        // never seen news yet
      } finally {
        newsInitRef.current = true;
      }
    })();
  }, []);

  const fetchMessages = useCallback(async (isInitial) => {
    try {
      const res = await window.storage.get(ROOM_KEY, true);
      const list = res && res.value ? JSON.parse(res.value) : [];
      setMessages(list);
      setFeedError(false);

      const newsItems = list.filter((m) => m.type === "news");
      if (newsItems.length) {
        const latest = newsItems.reduce((a, b) => (b.ts > a.ts ? b : a));
        if (latest.ts > lastSeenNewsRef.current) {
          if (!isInitial) {
            setToast(latest);
          }
          lastSeenNewsRef.current = latest.ts;
          window.storage.set(LAST_SEEN_NEWS_KEY, String(latest.ts), false).catch(() => {});
        }
      }
    } catch {
      setMessages((prev) => prev); // key may not exist yet
      setFeedError(false);
    } finally {
      if (isInitial) setLoadingFeed(false);
    }
  }, []);

  useEffect(() => {
    if (!toast) return;
    const t = setTimeout(() => setToast(null), 6000);
    return () => clearTimeout(t);
  }, [toast]);

  useEffect(() => {
    fetchMessages(true);
    pollRef.current = setInterval(() => fetchMessages(false), POLL_MS);
    return () => clearInterval(pollRef.current);
  }, [fetchMessages]);

  useEffect(() => {
    if (bottomRef.current) {
      bottomRef.current.scrollIntoView({ behavior: "smooth", block: "end" });
    }
  }, [messages.length]);

  const saveProfile = async () => {
    const name = nameInput.trim().slice(0, 16);
    if (!name) return;
    const p = { name, color: colorInput };
    setProfile(p);
    try {
      await window.storage.set(PROFILE_KEY, JSON.stringify(p), false);
    } catch {
      // keep local even if save fails
    }
  };

  const send = async () => {
    const body = text.trim();
    if (!body || !profile || sending) return;
    setSending(true);
    const newMsg = {
      id: uid(),
      user: profile.name,
      color: profile.color,
      text: body.slice(0, 500),
      ts: Date.now(),
      reactions: {},
    };
    setText("");
    try {
      const res = await window.storage.get(ROOM_KEY, true).catch(() => null);
      const current = res && res.value ? JSON.parse(res.value) : [];
      const next = [...current, newMsg].slice(-MAX_MESSAGES);
      await window.storage.set(ROOM_KEY, JSON.stringify(next), true);
      setMessages(next);
    } catch {
      setFeedError(true);
    } finally {
      setSending(false);
    }
  };

  const react = async (msgId, emoji) => {
    setPickerFor(null);
    try {
      const res = await window.storage.get(ROOM_KEY, true).catch(() => null);
      const current = res && res.value ? JSON.parse(res.value) : messages;
      const next = current.map((m) => {
        if (m.id !== msgId) return m;
        const reactions = { ...(m.reactions || {}) };
        reactions[emoji] = (reactions[emoji] || 0) + 1;
        return { ...m, reactions };
      });
      await window.storage.set(ROOM_KEY, JSON.stringify(next), true);
      setMessages(next);
    } catch {
      setFeedError(true);
    }
  };

  const checkPassword = () => {
    if (pwInput === NEWS_PASSWORD) {
      setAdminUnlocked(true);
      setPwError(false);
    } else {
      setPwError(true);
    }
  };

  const closeAdmin = () => {
    setAdminOpen(false);
    setAdminUnlocked(false);
    setPwInput("");
    setPwError(false);
    setNewsText("");
  };

  const postNews = async () => {
    const body = newsText.trim();
    if (!body || postingNews) return;
    setPostingNews(true);
    const newMsg = {
      id: uid(),
      user: NEWS_USER,
      color: "#FF6552",
      text: body.slice(0, 500),
      ts: Date.now(),
      type: "news",
      reactions: {},
    };
    try {
      const res = await window.storage.get(ROOM_KEY, true).catch(() => null);
      const current = res && res.value ? JSON.parse(res.value) : [];
      const next = [...current, newMsg].slice(-MAX_MESSAGES);
      await window.storage.set(ROOM_KEY, JSON.stringify(next), true);
      setMessages(next);
      lastSeenNewsRef.current = newMsg.ts;
      window.storage.set(LAST_SEEN_NEWS_KEY, String(newMsg.ts), false).catch(() => {});
      closeAdmin();
    } catch {
      setFeedError(true);
    } finally {
      setPostingNews(false);
    }
  };

  const memberCount = new Set(messages.filter((m) => m.type !== "news").map((m) => m.user)).size;

  // ---------- Onboarding ----------
  if (!profileLoading && !profile) {
    return (
      <div
        className="min-h-screen w-full flex items-center justify-center px-5"
        style={{ background: "#EEF2F5", fontFamily: "'Inter', sans-serif" }}
      >
        <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;700&family=Inter:wght@400;500;600&family=IBM+Plex+Mono:wght@500&display=swap" />
        <div className="w-full max-w-sm bg-white rounded-2xl shadow-lg p-6" style={{ borderRadius: "20px 20px 20px 6px" }}>
          <div
            style={{ fontFamily: "'Space Grotesk', sans-serif", color: "#131B23" }}
            className="text-2xl font-bold tracking-tight mb-1"
          >
            Breakit
          </div>
          <CrackDivider />
          <p className="text-sm mt-3 mb-5" style={{ color: "#5B6B76" }}>
            ニックネームを決めて、みんなの公開チャットに参加しよう。誰でも見れる、誰でも話せるオープンな場所です。
          </p>
          <label className="text-xs font-semibold" style={{ color: "#5B6B76" }}>
            ニックネーム
          </label>
          <input
            value={nameInput}
            onChange={(e) => setNameInput(e.target.value)}
            maxLength={16}
            placeholder="例：さかな"
            className="w-full mt-1 mb-4 px-3 py-2.5 rounded-lg border outline-none text-[15px]"
            style={{ borderColor: "#D8E0E6", color: "#131B23" }}
            onKeyDown={(e) => e.key === "Enter" && saveProfile()}
          />
          <label className="text-xs font-semibold" style={{ color: "#5B6B76" }}>
            カラー
          </label>
          <div className="flex gap-2 mt-2 mb-6">
            {COLORS.map((c) => (
              <button
                key={c.hex}
                onClick={() => setColorInput(c.hex)}
                aria-label={c.name}
                className="w-9 h-9 rounded-full flex items-center justify-center transition-transform"
                style={{
                  background: c.hex,
                  outline: colorInput === c.hex ? "3px solid #131B23" : "none",
                  outlineOffset: "2px",
                  transform: colorInput === c.hex ? "scale(1.05)" : "scale(1)",
                }}
              />
            ))}
          </div>
          <button
            onClick={saveProfile}
            disabled={!nameInput.trim()}
            className="w-full py-3 rounded-lg font-semibold text-white text-[15px] disabled:opacity-40 transition-opacity"
            style={{ background: "#131B23", borderRadius: "12px 12px 12px 4px" }}
          >
            参加する
          </button>
        </div>
      </div>
    );
  }

  // ---------- Main chat ----------
  return (
    <div
      className="min-h-screen w-full flex flex-col"
      style={{ background: "#EEF2F5", fontFamily: "'Inter', sans-serif" }}
    >
      <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;700&family=Inter:wght@400;500;600&family=IBM+Plex+Mono:wght@500&display=swap" />

      {/* Header */}
      <div className="sticky top-0 z-10 bg-white shadow-sm">
        <div className="px-4 pt-4 pb-2 flex items-center justify-between">
          <div>
            <div style={{ fontFamily: "'Space Grotesk', sans-serif", color: "#131B23" }} className="text-xl font-bold tracking-tight">
              Breakit
            </div>
            <div className="text-xs mt-0.5" style={{ color: "#5B6B76" }}>
              みんなの公開チャット
            </div>
          </div>
          <div className="flex items-center gap-2">
            <div
              className="flex items-center gap-1.5 px-2.5 py-1.5 rounded-full text-xs font-semibold"
              style={{ background: "#EEF2F5", color: "#5B6B76", fontFamily: "'IBM Plex Mono', monospace" }}
            >
              <Users size={13} />
              {memberCount}
            </div>
            <button
              onClick={() => setAdminOpen(true)}
              aria-label="BreakitNews"
              className="w-8 h-8 rounded-full flex items-center justify-center"
              style={{ background: "#EEF2F5", color: "#5B6B76" }}
            >
              <Newspaper size={15} />
            </button>
          </div>
        </div>
        <CrackDivider />
      </div>

      {/* News toast */}
      {toast && (
        <div className="fixed top-3 left-1/2 -translate-x-1/2 z-30 w-[92%] max-w-sm animate-[fadeIn_0.2s_ease-out]">
          <button
            onClick={() => setToast(null)}
            className="w-full text-left flex items-start gap-2.5 px-4 py-3 shadow-lg"
            style={{
              background: "#131B23",
              color: "#FFFFFF",
              borderRadius: "16px 16px 16px 4px",
              borderLeft: "3px solid #FF6552",
            }}
          >
            <Newspaper size={16} className="mt-0.5 shrink-0" style={{ color: "#FF6552" }} />
            <div className="min-w-0">
              <div className="text-[11px] font-semibold tracking-wide" style={{ color: "#FF6552" }}>
                BreakitNews
              </div>
              <div className="text-[13px] leading-snug line-clamp-2">{toast.text}</div>
            </div>
          </button>
        </div>
      )}

      {/* Admin / news posting modal */}
      {adminOpen && (
        <div className="fixed inset-0 z-40 flex items-center justify-center px-5" style={{ background: "rgba(19,27,35,0.5)" }}>
          <div className="w-full max-w-sm bg-white p-5" style={{ borderRadius: "20px 20px 20px 6px" }}>
            <div className="flex items-center justify-between mb-3">
              <div className="flex items-center gap-2" style={{ fontFamily: "'Space Grotesk', sans-serif", color: "#131B23" }}>
                <Newspaper size={18} style={{ color: "#FF6552" }} />
                <span className="font-bold">BreakitNews</span>
              </div>
              <button onClick={closeAdmin} aria-label="閉じる" style={{ color: "#8A97A1" }}>
                <X size={18} />
              </button>
            </div>

            {!adminUnlocked ? (
              <>
                <p className="text-xs mb-3" style={{ color: "#5B6B76" }}>
                  配信にはパスワードが必要です。
                </p>
                <div className="flex items-center gap-2 px-3 py-2.5 rounded-lg border mb-2" style={{ borderColor: pwError ? "#FF6552" : "#D8E0E6" }}>
                  <Lock size={14} style={{ color: "#8A97A1" }} />
                  <input
                    type="password"
                    value={pwInput}
                    onChange={(e) => {
                      setPwInput(e.target.value);
                      setPwError(false);
                    }}
                    onKeyDown={(e) => e.key === "Enter" && checkPassword()}
                    placeholder="パスワード"
                    className="flex-1 outline-none text-[15px]"
                    style={{ color: "#131B23" }}
                    autoFocus
                  />
                </div>
                {pwError && (
                  <p className="text-xs mb-3" style={{ color: "#FF6552" }}>
                    パスワードが違います
                  </p>
                )}
                <button
                  onClick={checkPassword}
                  disabled={!pwInput}
                  className="w-full py-3 rounded-lg font-semibold text-white text-[15px] disabled:opacity-40"
                  style={{ background: "#131B23", borderRadius: "12px 12px 12px 4px" }}
                >
                  ロック解除
                </button>
              </>
            ) : (
              <>
                <p className="text-xs mb-3" style={{ color: "#5B6B76" }}>
                  投稿すると全員のタイムラインに通知が表示されます。
                </p>
                <textarea
                  value={newsText}
                  onChange={(e) => setNewsText(e.target.value)}
                  rows={4}
                  maxLength={500}
                  placeholder="お知らせの内容を入力…"
                  className="w-full px-3.5 py-2.5 rounded-lg border outline-none text-[14px] leading-relaxed resize-none mb-3"
                  style={{ borderColor: "#D8E0E6", color: "#131B23" }}
                  autoFocus
                />
                <button
                  onClick={postNews}
                  disabled={!newsText.trim() || postingNews}
                  className="w-full py-3 rounded-lg font-semibold text-white text-[15px] disabled:opacity-40 flex items-center justify-center gap-2"
                  style={{ background: "#FF6552", borderRadius: "12px 12px 12px 4px" }}
                >
                  <Send size={15} />
                  配信する
                </button>
              </>
            )}
          </div>
        </div>
      )}

      {/* Feed */}
      <div ref={scrollRef} className="flex-1 overflow-y-auto px-4 py-4 space-y-3">
        {loadingFeed && (
          <div className="text-center text-sm py-10" style={{ color: "#5B6B76" }}>
            読み込み中…
          </div>
        )}

        {!loadingFeed && messages.length === 0 && (
          <div className="text-center py-16">
            <div className="text-sm font-semibold mb-1" style={{ color: "#131B23" }}>
              まだ誰も話していません
            </div>
            <div className="text-xs" style={{ color: "#5B6B76" }}>
              最初のひとことを投稿してみよう
            </div>
          </div>
        )}

        {messages.map((m) => {
          if (m.type === "news") {
            return (
              <div key={m.id} className="w-full flex justify-center py-1">
                <div
                  className="w-full px-4 py-3"
                  style={{
                    background: "#131B23",
                    borderRadius: "16px 16px 16px 4px",
                    borderLeft: "3px solid #FF6552",
                  }}
                >
                  <div className="flex items-center gap-2 mb-1">
                    <Newspaper size={13} style={{ color: "#FF6552" }} />
                    <span className="text-[11px] font-bold tracking-wide" style={{ color: "#FF6552" }}>
                      BreakitNews
                    </span>
                    <span style={{ fontFamily: "'IBM Plex Mono', monospace", color: "#8A97A1" }} className="text-[10px]">
                      {timeFmt(m.ts)}
                    </span>
                  </div>
                  <div className="text-[14px] leading-relaxed break-words" style={{ color: "#FFFFFF" }}>
                    {m.text}
                  </div>
                </div>
              </div>
            );
          }
          const isMe = profile && m.user === profile.name && m.color === profile.color;
          return (
            <div key={m.id} className={`flex gap-2 ${isMe ? "flex-row-reverse" : "flex-row"}`}>
              <Avatar name={m.user} color={m.color} />
              <div className={`flex flex-col ${isMe ? "items-end" : "items-start"} max-w-[75%]`}>
                <div className="flex items-baseline gap-2 mb-1 px-1">
                  <span className="text-xs font-semibold" style={{ color: m.color }}>
                    {m.user}
                  </span>
                  <span style={{ fontFamily: "'IBM Plex Mono', monospace", color: "#8A97A1" }} className="text-[10px]">
                    {timeFmt(m.ts)}
                  </span>
                </div>
                <div
                  className="px-3.5 py-2.5 text-[14px] leading-relaxed break-words"
                  style={{
                    background: isMe ? "#131B23" : "#FFFFFF",
                    color: isMe ? "#FFFFFF" : "#131B23",
                    borderRadius: isMe ? "16px 16px 4px 16px" : "16px 16px 16px 4px",
                    boxShadow: "0 1px 2px rgba(19,27,35,0.06)",
                  }}
                >
                  {m.text}
                </div>

                <div className="flex items-center gap-1 mt-1 px-1 relative">
                  {Object.entries(m.reactions || {}).map(([emoji, count]) =>
                    count > 0 ? (
                      <button
                        key={emoji}
                        onClick={() => react(m.id, emoji)}
                        className="text-[11px] px-1.5 py-0.5 rounded-full bg-white border flex items-center gap-1"
                        style={{ borderColor: "#D8E0E6" }}
                      >
                        <span>{emoji}</span>
                        <span style={{ fontFamily: "'IBM Plex Mono', monospace", color: "#5B6B76" }}>{count}</span>
                      </button>
                    ) : null
                  )}
                  <button
                    onClick={() => setPickerFor(pickerFor === m.id ? null : m.id)}
                    className="text-[11px] w-5 h-5 rounded-full flex items-center justify-center"
                    style={{ color: "#8A97A1" }}
                    aria-label="リアクションを追加"
                  >
                    <Smile size={14} />
                  </button>

                  {pickerFor === m.id && (
                    <div
                      className="absolute bottom-6 flex gap-1 bg-white px-2 py-1.5 rounded-full shadow-md border"
                      style={{ borderColor: "#D8E0E6", [isMe ? "right" : "left"]: 0 }}
                    >
                      {EMOJIS.map((e) => (
                        <button key={e} onClick={() => react(m.id, e)} className="text-base leading-none px-0.5 hover:scale-125 transition-transform">
                          {e}
                        </button>
                      ))}
                    </div>
                  )}
                </div>
              </div>
            </div>
          );
        })}
        <div ref={bottomRef} />
      </div>

      {feedError && (
        <div className="px-4 py-1.5 text-center text-xs" style={{ color: "#FF6552" }}>
          通信エラーが発生しました。もう一度お試しください。
        </div>
      )}

      {/* Composer */}
      <div className="sticky bottom-0 bg-white border-t px-3 py-3" style={{ borderColor: "#D8E0E6" }}>
        <div className="flex items-end gap-2">
          <div className="flex items-center gap-1.5 px-2 shrink-0">
            <Avatar name={profile?.name} color={profile?.color} size={28} />
          </div>
          <textarea
            value={text}
            onChange={(e) => setText(e.target.value)}
            onKeyDown={(e) => {
              if (e.key === "Enter" && !e.shiftKey) {
                e.preventDefault();
                send();
              }
            }}
            placeholder="なにか話そう…"
            rows={1}
            maxLength={500}
            className="flex-1 resize-none px-3.5 py-2.5 rounded-2xl border outline-none text-[14px] leading-snug max-h-24"
            style={{ borderColor: "#D8E0E6", color: "#131B23", borderRadius: "18px 18px 18px 6px" }}
          />
          <button
            onClick={send}
            disabled={!text.trim() || sending}
            className="w-10 h-10 rounded-full flex items-center justify-center text-white shrink-0 disabled:opacity-30 transition-opacity"
            style={{ background: "#2F6FED", borderRadius: "14px 14px 14px 4px" }}
            aria-label="送信"
          >
            <Send size={17} />
          </button>
        </div>
      </div>
    </div>
  );
}
