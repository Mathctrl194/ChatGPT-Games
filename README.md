import { useState, useEffect, useRef } from "react";

const MEMORY_KEY = "sat_coach_by_jr";
const CHATS_LIST_KEY = "sat_coach_chats_index";
const CHAT_PREFIX = "sat_coach_chat_";

function SAT101Logo({ size = 44 }) {
  return (
    <svg width={size} height={size} viewBox="0 0 100 100" xmlns="http://www.w3.org/2000/svg">
      <circle cx="50" cy="50" r="49" fill="#1a3a6b" />
      <circle cx="50" cy="50" r="44" fill="none" stroke="#f5a800" strokeWidth="3" />
      <g fill="#f5a800">
        <ellipse cx="18" cy="55" rx="4" ry="7" transform="rotate(-40 18 55)" />
        <ellipse cx="14" cy="46" rx="4" ry="7" transform="rotate(-30 14 46)" />
        <ellipse cx="13" cy="36" rx="4" ry="7" transform="rotate(-15 13 36)" />
        <ellipse cx="16" cy="27" rx="4" ry="7" transform="rotate(5 16 27)" />
      </g>
      <g fill="#f5a800">
        <ellipse cx="82" cy="55" rx="4" ry="7" transform="rotate(40 82 55)" />
        <ellipse cx="86" cy="46" rx="4" ry="7" transform="rotate(30 86 46)" />
        <ellipse cx="87" cy="36" rx="4" ry="7" transform="rotate(15 87 36)" />
        <ellipse cx="84" cy="27" rx="4" ry="7" transform="rotate(-5 84 27)" />
      </g>
      <path d="M28 22 L72 22 L72 62 Q50 78 28 62 Z" fill="white" stroke="#1a3a6b" strokeWidth="1.5" />
      <rect x="38" y="14" width="24" height="3" rx="1" fill="#1a3a6b" />
      <polygon points="50,8 62,14 50,17 38,14" fill="#1a3a6b" />
      <line x1="62" y1="14" x2="65" y2="19" stroke="#f5a800" strokeWidth="1.5" />
      <circle cx="65" cy="20" r="1.5" fill="#f5a800" />
      <text x="50" y="44" textAnchor="middle" fontSize="16" fontWeight="900" fill="#1a3a6b" fontFamily="Arial Black, sans-serif">SAT</text>
      <polygon points="50,48 51.5,52 55,52 52.5,54.5 53.5,58 50,56 46.5,58 47.5,54.5 45,52 48.5,52" fill="#f5a800" transform="scale(0.6) translate(33,33)" />
      <text x="50" y="64" textAnchor="middle" fontSize="16" fontWeight="900" fill="#f5a800" fontFamily="Arial Black, sans-serif">101</text>
      <path d="M28 70 Q50 65 72 70 Q50 75 28 70Z" fill="#4a90d9" opacity="0.7" />
      <circle cx="72" cy="76" r="8" fill="#3b82f6" />
      <polyline points="68,76 71,79 76,73" stroke="white" strokeWidth="2" fill="none" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  );
}

const RW_PROMPT = `You are Coach JR, an SAT coach with 20 years of experience. The student has pasted screenshots of their WRONG answers from the Reading & Writing section of a full-length SAT on the Bluebook app (not PSAT or NMSQT).
Analyze ONLY the Reading & Writing section errors. Identify weak R&W topics, patterns, and predict a R&W score range. Give a focused expert R&W breakdown in Coach JR's voice. Tell the student you're ready for their Math screenshots next.
After your response, append '---MEMORY---' with a 3-5 sentence summary of R&W weak topics and predicted score.
Prior session memory:\nMEMORY_PLACEHOLDER`;

const MATH_PROMPT = `You are Coach JR, an SAT coach with 20 years of experience. The student has pasted screenshots of their WRONG answers from the Math section of a full-length SAT on the Bluebook app (not PSAT or NMSQT).
Analyze ONLY the Math section errors. Then provide a FULL COMBINED ANALYSIS covering both sections:
## 📊 Full SAT Analysis & Summary
- Overall predicted SAT score range
- Top 3 priority topics across both sections
- Key patterns
- Personalized study roadmap
- Remind them to type "Make a study guide" for a full exported guide
After your response, append '---MEMORY---' with an updated 4-6 sentence summary covering both sections.
Prior session memory:\nMEMORY_PLACEHOLDER`;

const STUDY_GUIDE_PROMPT = `You are Coach JR, an SAT coach with 20 years of experience. Create a comprehensive personalized SAT study guide including:
1. Personalized intro, 2. Score estimate & target, 3. Top priority topics ranked by impact, 4. For each topic: 🎯 why it matters, 💡 core principle (deep explanation), 📝 2-3 fully worked example problems, ✅ full solutions, ⚠️ common mistakes, practice strategies, 5. 📅 4-week study schedule, 6. Resources, 7. Test-day tips, 8. 🏆 Motivational closing.
Write as Coach JR — direct, expert, encouraging. Use emojis throughout.
Student history:\nMEMORY_PLACEHOLDER`;

function formatDate(iso) {
  if (!iso) return "";
  const d = new Date(iso);
  return d.toLocaleDateString("en-US", { month: "short", day: "numeric", hour: "2-digit", minute: "2-digit" });
}

function genId() { return Date.now().toString(36) + Math.random().toString(36).slice(2, 6); }

const WELCOME_MSG = {
  role: "assistant",
  text: "Hey, I'm SAT 101 — backed by Coach JR's 20 years of SAT expertise, and I've seen every mistake in the book. 📚\n\nHere's how this works:\n\n1️⃣ Paste your **Reading & Writing** wrong-answer screenshots (Ctrl+V / Cmd+V), then tap **Done (R&W)**\n2️⃣ Paste your **Math** wrong-answer screenshots, then tap **Done (Math)**\n\nI'll analyze each section separately, then give you a full combined breakdown.\n\nWant a full personalized study guide exported to Google Drive? Type **Make a study guide** anytime."
};

export default function SATCoach() {
  const [view, setView] = useState("chat"); // "chat" | "history"
  const [chatId, setChatId] = useState(null);
  const [messages, setMessages] = useState([WELCOME_MSG]);
  const [input, setInput] = useState("");
  const [images, setImages] = useState([]);
  const [loading, setLoading] = useState(false);
  const [memory, setMemory] = useState(null);
  const [pasteHint, setPasteHint] = useState(false);
  const [doneCount, setDoneCount] = useState(0);
  const [allUserMessages, setAllUserMessages] = useState([]);
  const [exportStatus, setExportStatus] = useState(null);
  const [chatsList, setChatsList] = useState([]);
  const [appReady, setAppReady] = useState(false);
  const bottomRef = useRef(null);
  const inputRef = useRef(null);
  const saveTimer = useRef(null);

  useEffect(() => { init(); }, []);
  useEffect(() => { bottomRef.current?.scrollIntoView({ behavior: "smooth" }); }, [messages, loading]);

  // Autosave on messages change
  useEffect(() => {
    if (!appReady || !chatId) return;
    clearTimeout(saveTimer.current);
    saveTimer.current = setTimeout(() => saveCurrentChat(chatId, messages, doneCount), 800);
  }, [messages, chatId, appReady]);

  async function init() {
    try {
      const memRes = await window.storage.get(MEMORY_KEY, true).catch(() => null);
      if (memRes) setMemory(JSON.parse(memRes.value));
      const listRes = await window.storage.get(CHATS_LIST_KEY, true).catch(() => null);
      const list = listRes ? JSON.parse(listRes.value) : [];
      setChatsList(list);
      if (list.length > 0) {
        await loadChat(list[0].id, false);
      } else {
        setChatId(genId());
      }
    } catch (e) { console.error(e); setChatId(genId()); }
    setAppReady(true);
  }

  async function loadChat(id, switchView = true) {
    try {
      const res = await window.storage.get(CHAT_PREFIX + id, true);
      if (res) {
        const data = JSON.parse(res.value);
        setMessages(data.messages || [WELCOME_MSG]);
        setDoneCount(data.doneCount || 0);
        setAllUserMessages(data.allUserMessages || []);
      } else {
        setMessages([WELCOME_MSG]);
        setDoneCount(0);
        setAllUserMessages([]);
      }
      setChatId(id);
      if (switchView) setView("chat");
    } catch { setMessages([WELCOME_MSG]); setChatId(id); if (switchView) setView("chat"); }
  }

  async function saveCurrentChat(id, msgs, dc) {
    if (!id || msgs.length <= 1) return;
    try {
      const title = msgs.find(m => m.role === "user" && m.text && m.text !== "(pasted screenshots)")?.text?.slice(0, 40) || "SAT Session";
      const payload = { messages: msgs, doneCount: dc, allUserMessages, updatedAt: new Date().toISOString(), title };
      await window.storage.set(CHAT_PREFIX + id, JSON.stringify(payload), true);
      const listRes = await window.storage.get(CHATS_LIST_KEY, true).catch(() => null);
      let list = listRes ? JSON.parse(listRes.value) : [];
      list = list.filter(c => c.id !== id);
      list.unshift({ id, title, updatedAt: payload.updatedAt });
      list = list.slice(0, 30);
      await window.storage.set(CHATS_LIST_KEY, JSON.stringify(list), true);
      setChatsList(list);
    } catch (e) { console.error("Save failed", e); }
  }

  async function startNewChat() {
    if (chatId) await saveCurrentChat(chatId, messages, doneCount);
    const id = genId();
    setChatId(id);
    setMessages([WELCOME_MSG]);
    setDoneCount(0);
    setAllUserMessages([]);
    setImages([]);
    setInput("");
    setExportStatus(null);
    setView("chat");
  }

  async function deleteChat(id, e) {
    e.stopPropagation();
    try {
      await window.storage.delete(CHAT_PREFIX + id, true);
      const listRes = await window.storage.get(CHATS_LIST_KEY, true).catch(() => null);
      let list = listRes ? JSON.parse(listRes.value) : [];
      list = list.filter(c => c.id !== id);
      await window.storage.set(CHATS_LIST_KEY, JSON.stringify(list), true);
      setChatsList(list);
      if (id === chatId) startNewChat();
    } catch (e) { console.error(e); }
  }

  async function saveMemory(text) {
    try {
      await window.storage.set(MEMORY_KEY, JSON.stringify({ summary: text, updatedAt: new Date().toISOString() }), true);
      setMemory({ summary: text });
    } catch (e) { console.error(e); }
  }

  function removeImage(i) { setImages(prev => prev.filter((_, idx) => idx !== i)); }

  function handlePaste(e) {
    const items = e.clipboardData?.items;
    if (!items) return;
    let found = false;
    for (const item of items) {
      if (item.type.startsWith("image/")) {
        found = true;
        const file = item.getAsFile();
        const reader = new FileReader();
        reader.onload = ev => {
          setImages(prev => [...prev, { dataUrl: ev.target.result, name: "screenshot.png", type: item.type }]);
          setPasteHint(true);
          setTimeout(() => setPasteHint(false), 2000);
        };
        reader.readAsDataURL(file);
      }
    }
    if (found) e.preventDefault();
  }

  async function generateAndExportStudyGuide() {
    setExportStatus("exporting");
    const memText = memory?.summary || "No prior sessions yet.";
    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST", headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ model: "claude-sonnet-4-20250514", max_tokens: 1000, messages: [{ role: "user", content: STUDY_GUIDE_PROMPT.replace("MEMORY_PLACEHOLDER", memText) }] })
      });
      const data = await res.json();
      const guideText = data.content.map(b => b.text || "").join("").trim();
      const driveMessages = [{ role: "user", content: `Use the create_file tool to create a Google Doc titled "SAT Study Guide by Coach JR" with this content:\n\n${guideText}` }];
      const driveRes = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST", headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ model: "claude-sonnet-4-20250514", max_tokens: 1000, mcp_servers: [{ type: "url", url: "https://drivemcp.googleapis.com/mcp/v1", name: "google-drive" }], messages: driveMessages })
      });
      const driveData = await driveRes.json();
      let fileUrl = null;
      for (const block of driveData.content) {
        const raw = block.type === "mcp_tool_result" ? JSON.stringify(block.content) : block.type === "text" ? block.text : "";
        const m = raw.match(/https:\/\/docs\.google\.com\/[^\s"'\\)]+|https:\/\/drive\.google\.com\/[^\s"'\\)]+/);
        if (m) { fileUrl = m[0]; break; }
      }
      if (!fileUrl) {
        const fu = await fetch("https://api.anthropic.com/v1/messages", {
          method: "POST", headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ model: "claude-sonnet-4-20250514", max_tokens: 500, mcp_servers: [{ type: "url", url: "https://drivemcp.googleapis.com/mcp/v1", name: "google-drive" }], messages: [...driveMessages, { role: "assistant", content: driveData.content }, { role: "user", content: "Return only the webViewLink URL of the file you just created." }] })
        });
        const fd = await fu.json();
        const ft = fd.content.map(b => b.text || "").join("");
        const m = ft.match(/https:\/\/docs\.google\.com\/[^\s"'\\)]+|https:\/\/drive\.google\.com\/[^\s"'\\)]+/);
        if (m) fileUrl = m[0];
      }
      if (fileUrl) {
        setExportStatus({ url: fileUrl });
        setMessages(prev => [...prev, { role: "assistant", text: "Your personalized study guide is ready and saved to Google Drive! 🎉\n\nClick the link below to open it. Packed with priority topics, worked examples, a 4-week schedule, and my best tips. Now get to work! 💪", driveUrl: fileUrl }]);
      } else {
        setExportStatus("error");
        setMessages(prev => [...prev, { role: "assistant", text: `Generated your study guide but couldn't get the Drive link. Here it is:\n\n${guideText}` }]);
      }
    } catch (e) {
      setExportStatus("error");
      setMessages(prev => [...prev, { role: "assistant", text: `Study guide error: ${e?.message}` }]);
    }
  }

  async function generatePracticeQuiz() {
    if (!memory?.summary) {
      setMessages(prev => [...prev, { role: "assistant", text: "I need to analyze your screenshots first before making a quiz! Upload your wrong answers and type Done to get started. 📝" }]);
      return;
    }
    setLoading(true);
    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST", headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514", max_tokens: 1000,
          messages: [{ role: "user", content: `You are Coach JR, a 20-year SAT expert. Based on this student's weak topics, create a practice quiz with exactly 3 SAT-style problems — one per general weak topic. Each must be realistic and match actual SAT difficulty.\n\nFor each problem:\n- Label: "Problem 1 — [Topic Name]"\n- Full SAT-style question with A/B/C/D choices\n\nAfter all 3, add "--- ANSWERS ---" with correct answer letter, full step-by-step explanation, and a Coach JR tip. Use emojis. Speak as Coach JR.\n\nStudent's weak topics:\n${memory.summary}` }]
        })
      });
      const data = await res.json();
      const quizText = data.content.map(b => b.text || "").join("").trim();
      setMessages(prev => [...prev, { role: "assistant", text: quizText, sectionBadge: "🧪 Practice Quiz" }]);
    } catch (e) { setMessages(prev => [...prev, { role: "assistant", text: `Quiz error: ${e?.message}` }]); }
    setLoading(false);
  }

  async function sendMessage() {
    const text = input.trim();
    if (!text && images.length === 0) return;
    if (loading) return;
    const isDone = text.toLowerCase() === "done";
    const isStudyGuide = text.toLowerCase() === "make a study guide";
    const userMsg = { role: "user", text: text || "(pasted screenshots)", images: [...images] };
    const newAllUser = [...allUserMessages, userMsg];
    setAllUserMessages(newAllUser);
    setMessages(prev => [...prev, userMsg]);
    setInput(""); setImages([]);
    if (isStudyGuide) { await generateAndExportStudyGuide(); return; }
    if (!isDone) return;
    const newDoneCount = doneCount + 1;
    setDoneCount(newDoneCount);
    setLoading(true);
    const memText = memory?.summary || "No prior sessions yet.";
    const isRW = newDoneCount === 1;
    const sysPrompt = (isRW ? RW_PROMPT : MATH_PROMPT).replace("MEMORY_PLACEHOLDER", memText);
    const lastDoneIdx = allUserMessages.map(m => m.text?.toLowerCase()).lastIndexOf("done");
    const batch = lastDoneIdx === -1 ? newAllUser : newAllUser.slice(lastDoneIdx + 1);
    const batchImages = batch.flatMap(m => m.images || []);
    const userContent = [];
    batchImages.forEach(img => userContent.push({ type: "image", source: { type: "base64", media_type: img.type || "image/png", data: img.dataUrl.split(",")[1] } }));
    userContent.push({ type: "text", text: `I'm done uploading my ${isRW ? "Reading & Writing" : "Math"} wrong-answer screenshots. Please analyze them.` });
    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST", headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ model: "claude-sonnet-4-20250514", max_tokens: 1000, system: sysPrompt, messages: [{ role: "user", content: userContent }] })
      });
      const data = await res.json();
      const fullText = data.content.map(b => b.text || "").join("");
      const memIdx = fullText.indexOf("---MEMORY---");
      let reply = fullText, newMem = null;
      if (memIdx !== -1) { reply = fullText.slice(0, memIdx).trim(); newMem = fullText.slice(memIdx + 12).trim(); }
      setMessages(prev => [...prev, { role: "assistant", text: reply, sectionBadge: isRW ? "📖 R&W Analysis" : "🔢 Math Analysis + Full Summary" }]);
      if (newMem) await saveMemory(newMem);
    } catch (e) { setMessages(prev => [...prev, { role: "assistant", text: `Coach error: ${e?.message}` }]); }
    setLoading(false);
  }

  function handleKey(e) { if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); sendMessage(); } }

  const doneLabel = doneCount === 0 ? "✅ Done (R&W)" : doneCount === 1 ? "✅ Done (Math)" : "✅ Done";

  // ── HISTORY VIEW ──
  if (view === "history") return (
    <div style={{ display:"flex", flexDirection:"column", height:"100vh", background:"#0f172a", fontFamily:"'Inter',sans-serif", color:"#e2e8f0" }}>
      <div style={{ background:"#1e293b", borderBottom:"1px solid #334155", padding:"16px 20px", display:"flex", alignItems:"center", gap:12 }}>
        <button onClick={() => setView("chat")} style={{ background:"#334155", border:"none", color:"#94a3b8", borderRadius:10, padding:"6px 12px", cursor:"pointer", fontSize:13 }}>← Back</button>
        <div style={{ fontWeight:700, fontSize:16, color:"#f1f5f9" }}>Chat History</div>
        <button onClick={startNewChat} style={{ marginLeft:"auto", background:"linear-gradient(135deg,#3b82f6,#1d4ed8)", border:"none", color:"#fff", borderRadius:10, padding:"6px 14px", cursor:"pointer", fontSize:13, fontWeight:600 }}>+ New Chat</button>
      </div>
      <div style={{ flex:1, overflowY:"auto", padding:16, display:"flex", flexDirection:"column", gap:10 }}>
        {chatsList.length === 0 && <div style={{ color:"#64748b", textAlign:"center", marginTop:40 }}>No saved chats yet. Start a session!</div>}
        {chatsList.map(c => (
          <div key={c.id} onClick={() => loadChat(c.id)} style={{ background:c.id===chatId?"#1e3a5f":"#1e293b", border:`1px solid ${c.id===chatId?"#3b82f6":"#334155"}`, borderRadius:12, padding:"12px 16px", cursor:"pointer", display:"flex", alignItems:"center", gap:10 }}>
            <div style={{ flex:1 }}>
              <div style={{ fontSize:14, fontWeight:600, color:"#f1f5f9", marginBottom:3 }}>{c.title || "SAT Session"}</div>
              <div style={{ fontSize:11, color:"#64748b" }}>{formatDate(c.updatedAt)}</div>
            </div>
            {c.id === chatId && <div style={{ fontSize:11, color:"#3b82f6", fontWeight:600 }}>Current</div>}
            <button onClick={e => deleteChat(c.id, e)} style={{ background:"#ef444420", border:"1px solid #ef444440", color:"#ef4444", borderRadius:8, padding:"4px 8px", cursor:"pointer", fontSize:11 }}>Delete</button>
          </div>
        ))}
      </div>
    </div>
  );

  // ── CHAT VIEW ──
  return (
    <div style={{ display:"flex", flexDirection:"column", height:"100vh", background:"#0f172a", fontFamily:"'Inter',sans-serif", color:"#e2e8f0" }}>
      {/* Header */}
      <div style={{ background:"#1e293b", borderBottom:"1px solid #334155", padding:"12px 16px", display:"flex", alignItems:"center", gap:10 }}>
        <div style={{ flexShrink:0 }}><SAT101Logo size={40} /></div>
        <div style={{ flex:1, minWidth:0 }}>
          <div style={{ fontWeight:700, fontSize:15, color:"#f1f5f9" }}>SAT 101</div>
          <div style={{ fontSize:11, color:"#64748b", whiteSpace:"nowrap", overflow:"hidden", textOverflow:"ellipsis" }}>SAT Expert · 20 Years Experience · By ChanHou Jeremiah Ruan</div>
        </div>
        <div style={{ display:"flex", gap:6, alignItems:"center", flexShrink:0 }}>
          {memory?.summary && <div style={{ fontSize:10, color:"#3b82f6", background:"#1e3a5f", padding:"3px 8px", borderRadius:20, border:"1px solid #1d4ed8" }}>📖 History</div>}
          <button onClick={() => setView("history")} style={{ background:"#334155", border:"none", color:"#94a3b8", borderRadius:10, padding:"6px 10px", cursor:"pointer", fontSize:12 }}>🕘 Chats</button>
          <button onClick={startNewChat} style={{ background:"linear-gradient(135deg,#3b82f6,#1d4ed8)", border:"none", color:"#fff", borderRadius:10, padding:"6px 10px", cursor:"pointer", fontSize:12, fontWeight:600 }}>+ New</button>
        </div>
      </div>

      {/* Section status */}
      <div style={{ background:"#0f172a", borderBottom:"1px solid #1e293b", padding:"6px 16px", fontSize:11, color:"#64748b" }}>
        {doneCount===0?"📖 Step 1: Paste R&W wrong-answer screenshots":doneCount===1?"🔢 Step 2: Paste Math wrong-answer screenshots":"✅ Both sections analyzed"}
      </div>

      {/* Messages */}
      <div style={{ flex:1, overflowY:"auto", padding:"20px 16px", display:"flex", flexDirection:"column", gap:14 }}>
        {messages.map((m, i) => (
          <div key={i} style={{ display:"flex", justifyContent:m.role==="user"?"flex-end":"flex-start", gap:8, alignItems:"flex-end" }}>
            {m.role==="assistant" && <div style={{ flexShrink:0 }}><SAT101Logo size={28} /></div>}
            <div style={{ maxWidth:"78%", display:"flex", flexDirection:"column", gap:5 }}>
              {m.sectionBadge && <div style={{ fontSize:11, fontWeight:700, color:"#3b82f6", padding:"3px 10px", background:"#1e3a5f", borderRadius:20, border:"1px solid #1d4ed8", alignSelf:"flex-start" }}>{m.sectionBadge}</div>}
              <div style={{ background:m.role==="user"?"linear-gradient(135deg,#3b82f6,#1d4ed8)":"#1e293b", color:"#f1f5f9", borderRadius:m.role==="user"?"18px 18px 4px 18px":"18px 18px 18px 4px", padding:"11px 15px", fontSize:14, lineHeight:1.65, border:m.role==="assistant"?"1px solid #334155":"none", whiteSpace:"pre-wrap" }}>
                {m.images?.length > 0 && <div style={{ display:"flex", flexWrap:"wrap", gap:6, marginBottom:8 }}>{m.images.map((img,j) => <img key={j} src={img.dataUrl} alt="" style={{ width:72, height:72, objectFit:"cover", borderRadius:8, border:"2px solid rgba(255,255,255,0.2)" }} />)}</div>}
                {m.text !== "(pasted screenshots)" && m.text}
                {m.driveUrl && <a href={m.driveUrl} target="_blank" rel="noopener noreferrer" style={{ display:"inline-flex", alignItems:"center", gap:6, marginTop:10, background:"#ffffff15", border:"1px solid #ffffff30", borderRadius:10, padding:"8px 14px", color:"#93c5fd", fontSize:13, textDecoration:"none", fontWeight:600 }}>📄 Open Study Guide in Google Drive ↗</a>}
              </div>
            </div>
          </div>
        ))}
        {(loading || exportStatus==="exporting") && (
          <div style={{ display:"flex", alignItems:"flex-end", gap:8 }}>
            <SAT101Logo size={28} />
            <div style={{ background:"#1e293b", border:"1px solid #334155", borderRadius:"18px 18px 18px 4px", padding:"14px 18px", display:"flex", flexDirection:"column", gap:6 }}>
              <div style={{ display:"flex", gap:6 }}>{[0,1,2].map(i => <div key={i} style={{ width:8, height:8, borderRadius:"50%", background:"#3b82f6", animation:"bounce 1.2s infinite", animationDelay:`${i*0.2}s` }} />)}</div>
              {exportStatus==="exporting" && <div style={{ fontSize:12, color:"#64748b" }}>Generating & exporting to Google Drive...</div>}
            </div>
          </div>
        )}
        <div ref={bottomRef} />
      </div>

      {/* Image previews */}
      {images.length > 0 && (
        <div style={{ padding:"8px 16px", background:"#1e293b", borderTop:"1px solid #334155", display:"flex", gap:8, flexWrap:"wrap", alignItems:"center" }}>
          <span style={{ fontSize:12, color:"#64748b" }}>{images.length} screenshot{images.length>1?"s":""} ready</span>
          {images.map((img,i) => (
            <div key={i} style={{ position:"relative" }}>
              <img src={img.dataUrl} alt="" style={{ width:52, height:52, objectFit:"cover", borderRadius:8, border:"2px solid #3b82f6" }} />
              <button onClick={() => removeImage(i)} style={{ position:"absolute", top:-5, right:-5, width:16, height:16, borderRadius:"50%", background:"#ef4444", border:"none", color:"#fff", fontSize:10, cursor:"pointer", display:"flex", alignItems:"center", justifyContent:"center" }}>✕</button>
            </div>
          ))}
        </div>
      )}

      {/* Quick actions */}
      <div style={{ background:"#1e293b", padding:"6px 12px", display:"flex", gap:6, flexWrap:"wrap" }}>
        <button onClick={() => setInput("done")} style={{ fontSize:11, color:"#94a3b8", background:"#0f172a", border:"1px solid #334155", borderRadius:20, padding:"4px 10px", cursor:"pointer" }}>{doneLabel}</button>
        <button onClick={() => setInput("Make a study guide")} style={{ fontSize:11, color:"#94a3b8", background:"#0f172a", border:"1px solid #334155", borderRadius:20, padding:"4px 10px", cursor:"pointer" }}>📘 Study Guide</button>
        <button onClick={generatePracticeQuiz} style={{ fontSize:11, color:"#94a3b8", background:"#0f172a", border:"1px solid #334155", borderRadius:20, padding:"4px 10px", cursor:"pointer" }}>🧪 Practice Quiz</button>
      </div>

      {/* Input */}
      <div style={{ background:"#1e293b", borderTop:"1px solid #334155", padding:"10px 14px", display:"flex", gap:8, alignItems:"flex-end" }}>
        <div style={{ flex:1, position:"relative" }}>
          <textarea ref={inputRef} value={input} onChange={e => setInput(e.target.value)} onKeyDown={handleKey} onPaste={handlePaste}
            placeholder={doneCount===0?'Paste R&W screenshots (Ctrl+V), then click Done (R&W)...':doneCount===1?'Paste Math screenshots (Ctrl+V), then click Done (Math)...':'Ask Coach JR anything or type "Make a study guide"...'}
            rows={1} style={{ width:"100%", boxSizing:"border-box", background:"#0f172a", border:`1px solid ${pasteHint?"#3b82f6":"#334155"}`, borderRadius:12, padding:"10px 14px", color:"#f1f5f9", fontSize:14, resize:"none", outline:"none", lineHeight:1.5, fontFamily:"inherit", transition:"border-color 0.2s" }} />
          {pasteHint && <div style={{ position:"absolute", top:-30, left:0, background:"#3b82f6", color:"#fff", fontSize:12, padding:"4px 10px", borderRadius:8, pointerEvents:"none" }}>✅ Screenshot added!</div>}
        </div>
        <button onClick={sendMessage} disabled={loading||exportStatus==="exporting"||(!input.trim()&&images.length===0)}
          style={{ width:40, height:40, borderRadius:12, background:(loading||exportStatus==="exporting")?"#334155":"linear-gradient(135deg,#3b82f6,#1d4ed8)", border:"none", color:"#fff", fontSize:17, cursor:(loading||exportStatus==="exporting")?"not-allowed":"pointer", flexShrink:0, display:"flex", alignItems:"center", justifyContent:"center" }}>➤</button>
      </div>
      <style>{`@keyframes bounce{0%,80%,100%{transform:translateY(0);opacity:0.4}40%{transform:translateY(-6px);opacity:1}}textarea::-webkit-scrollbar{width:4px}::-webkit-scrollbar{width:6px}::-webkit-scrollbar-track{background:#0f172a}::-webkit-scrollbar-thumb{background:#334155;border-radius:4px}`}</style>
    </div>
  );
} 

