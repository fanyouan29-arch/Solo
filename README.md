import { useState, useEffect, useRef } from "react";
import { Plus, Trash2, Flame, ShieldCheck, X } from "lucide-react";

// ---------- منطق المستويات ----------
const xpForLevel = (level) => 100 + (level - 1) * 40; // كلفة الانتقال من level إلى level+1

function levelFromTotalXP(totalXP) {
  let level = 1;
  let remaining = totalXP;
  while (remaining >= xpForLevel(level)) {
    remaining -= xpForLevel(level);
    level += 1;
  }
  return { level, xpIntoLevel: remaining, xpNeeded: xpForLevel(level) };
}

function rankFromLevel(level) {
  if (level >= 50) return { label: "S", color: "#facc15" };
  if (level >= 40) return { label: "A", color: "#f97316" };
  if (level >= 30) return { label: "B", color: "#a78bfa" };
  if (level >= 20) return { label: "C", color: "#38bdf8" };
  if (level >= 10) return { label: "D", color: "#4ade80" };
  return { label: "E", color: "#94a3b8" };
}

const todayStr = () => new Date().toISOString().slice(0, 10);
const daysBetween = (a, b) => Math.round((new Date(b) - new Date(a)) / 86400000);

const uid = () => Math.random().toString(36).slice(2, 10);

const DEFAULT_STATE = {
  totalXP: 0,
  streak: 0,
  lastActiveDate: todayStr(),
  fixedQuests: [
    { id: uid(), title: "شرب 2 لتر ماء", xp: 10, done: false },
    { id: uid(), title: "30 دقيقة رياضة", xp: 20, done: false },
    { id: uid(), title: "قراءة 15 صفحة", xp: 15, done: false },
  ],
  variableQuests: [],
};

export default function SoloSystem() {
  const [state, setState] = useState(null);
  const [loading, setLoading] = useState(true);
  const [newFixed, setNewFixed] = useState("");
  const [newVar, setNewVar] = useState("");
  const [showFixedInput, setShowFixedInput] = useState(false);
  const [showVarInput, setShowVarInput] = useState(false);
  const [levelUpInfo, setLevelUpInfo] = useState(null);
  const prevLevelRef = useRef(1);

  // تحميل الحالة من التخزين
  useEffect(() => {
    (async () => {
      let loaded = null;
      try {
        const res = await window.storage.get("solo-system-state", false);
        if (res && res.value) loaded = JSON.parse(res.value);
      } catch (e) {
        loaded = null;
      }
      if (!loaded) loaded = DEFAULT_STATE;

      // تصفير يومي
      const today = todayStr();
      if (loaded.lastActiveDate !== today) {
        const gap = daysBetween(loaded.lastActiveDate, today);
        const allDoneYesterday =
          loaded.fixedQuests.length > 0 &&
          loaded.fixedQuests.every((q) => q.done);
        loaded = {
          ...loaded,
          streak: gap === 1 && allDoneYesterday ? loaded.streak + 1 : 0,
          fixedQuests: loaded.fixedQuests.map((q) => ({ ...q, done: false })),
          variableQuests: [],
          lastActiveDate: today,
        };
      }
      prevLevelRef.current = levelFromTotalXP(loaded.totalXP).level;
      setState(loaded);
      setLoading(false);
    })();
  }, []);

  // حفظ الحالة عند أي تغيير
  useEffect(() => {
    if (!state) return;
    window.storage.set("solo-system-state", JSON.stringify(state), false).catch(() => {});
  }, [state]);

  if (loading || !state) {
    return (
      <div style={styles.loadingScreen}>
        <div style={styles.loadingText}>...جارِ تحميل النظام</div>
      </div>
    );
  }

  const { level, xpIntoLevel, xpNeeded } = levelFromTotalXP(state.totalXP);
  const rank = rankFromLevel(level);
  const progressPct = Math.min(100, Math.round((xpIntoLevel / xpNeeded) * 100));

  function applyXPChange(delta) {
    setState((prev) => {
      const newTotal = Math.max(0, prev.totalXP + delta);
      const newLevel = levelFromTotalXP(newTotal).level;
      if (newLevel > prevLevelRef.current) {
        setLevelUpInfo({ level: newLevel, rank: rankFromLevel(newLevel) });
      }
      prevLevelRef.current = newLevel;
      return { ...prev, totalXP: newTotal };
    });
  }

  function toggleFixed(id) {
    setState((prev) => {
      const q = prev.fixedQuests.find((x) => x.id === id);
      const delta = q.done ? -q.xp : q.xp;
      applyXPChange(delta);
      return {
        ...prev,
        fixedQuests: prev.fixedQuests.map((x) =>
          x.id === id ? { ...x, done: !x.done } : x
        ),
      };
    });
  }

  function toggleVar(id) {
    setState((prev) => {
      const q = prev.variableQuests.find((x) => x.id === id);
      const delta = q.done ? -q.xp : q.xp;
      applyXPChange(delta);
      return {
        ...prev,
        variableQuests: prev.variableQuests.map((x) =>
          x.id === id ? { ...x, done: !x.done } : x
        ),
      };
    });
  }

  function addFixed() {
    if (!newFixed.trim()) return;
    setState((prev) => ({
      ...prev,
      fixedQuests: [
        ...prev.fixedQuests,
        { id: uid(), title: newFixed.trim(), xp: 10, done: false },
      ],
    }));
    setNewFixed("");
    setShowFixedInput(false);
  }

  function addVar() {
    if (!newVar.trim()) return;
    setState((prev) => ({
      ...prev,
      variableQuests: [
        ...prev.variableQuests,
        { id: uid(), title: newVar.trim(), xp: 15, done: false },
      ],
    }));
    setNewVar("");
    setShowVarInput(false);
  }

  function removeFixed(id) {
    setState((prev) => ({
      ...prev,
      fixedQuests: prev.fixedQuests.filter((x) => x.id !== id),
    }));
  }

  function removeVar(id) {
    setState((prev) => ({
      ...prev,
      variableQuests: prev.variableQuests.filter((x) => x.id !== id),
    }));
  }

  return (
    <div dir="rtl" style={styles.app}>
      <style>{`
        @keyframes glowPulse {
          0%, 100% { box-shadow: 0 0 12px rgba(56,189,248,0.35), inset 0 0 20px rgba(56,189,248,0.05); }
          50% { box-shadow: 0 0 22px rgba(56,189,248,0.6), inset 0 0 30px rgba(56,189,248,0.1); }
        }
        @keyframes riseIn {
          from { opacity: 0; transform: translateY(10px) scale(0.98); }
          to { opacity: 1; transform: translateY(0) scale(1); }
        }
        @keyframes modalIn {
          from { opacity: 0; transform: scale(0.85); }
          to { opacity: 1; transform: scale(1); }
        }
        .quest-row { animation: riseIn 0.25s ease; }
        .quest-row:active { transform: scale(0.99); }
        * { box-sizing: border-box; }
        input::placeholder { color: #4b5c73; }
      `}</style>

      {/* بطاقة اللاعب */}
      <div style={{ ...styles.playerCard, animation: "glowPulse 4s ease-in-out infinite" }}>
        <div style={styles.playerTop}>
          <div>
            <div style={styles.eyebrow}>[ الصياد ]</div>
            <div style={styles.levelRow}>
              <span style={styles.levelNum}>Lv.{level}</span>
              <span style={{ ...styles.rankBadge, borderColor: rank.color, color: rank.color }}>
                رتبة {rank.label}
              </span>
            </div>
          </div>
          <div style={styles.streakBox}>
            <Flame size={16} color={state.streak > 0 ? "#fb923c" : "#4b5c73"} />
            <span style={{ color: state.streak > 0 ? "#fb923c" : "#4b5c73" }}>
              {state.streak}
            </span>
          </div>
        </div>

        <div style={styles.xpBarOuter}>
          <div style={{ ...styles.xpBarInner, width: `${progressPct}%` }} />
        </div>
        <div style={styles.xpLabel}>
          {xpIntoLevel} / {xpNeeded} XP
        </div>
      </div>

      {/* المهام الثابتة */}
      <Section
        title="المهام اليومية الثابتة"
        subtitle="تتكرر كل يوم — عادات تبنيها"
        onAdd={() => setShowFixedInput((s) => !s)}
      >
        {showFixedInput && (
          <AddRow
            value={newFixed}
            onChange={setNewFixed}
            onSubmit={addFixed}
            onCancel={() => setShowFixedInput(false)}
            placeholder="اسم المهمة الثابتة..."
          />
        )}
        {state.fixedQuests.length === 0 && !showFixedInput && (
          <EmptyHint text="لا توجد مهام ثابتة بعد. أضف أول عادة يومية." />
        )}
        {state.fixedQuests.map((q) => (
          <QuestRow key={q.id} quest={q} onToggle={() => toggleFixed(q.id)} onRemove={() => removeFixed(q.id)} />
        ))}
      </Section>

      {/* المهام المتغيرة */}
      <Section
        title="مهام اليوم"
        subtitle="تُحذف تلقائياً عند بداية يوم جديد"
        onAdd={() => setShowVarInput((s) => !s)}
      >
        {showVarInput && (
          <AddRow
            value={newVar}
            onChange={setNewVar}
            onSubmit={addVar}
            onCancel={() => setShowVarInput(false)}
            placeholder="مهمة اليوم..."
          />
        )}
        {state.variableQuests.length === 0 && !showVarInput && (
          <EmptyHint text="لا توجد مهام لهذا اليوم. أضف ما تريد إنجازه." />
        )}
        {state.variableQuests.map((q) => (
          <QuestRow key={q.id} quest={q} onToggle={() => toggleVar(q.id)} onRemove={() => removeVar(q.id)} />
        ))}
      </Section>

      {/* نافذة رفع المستوى */}
      {levelUpInfo && (
        <div style={styles.modalOverlay} onClick={() => setLevelUpInfo(null)}>
          <div style={{ ...styles.modal, animation: "modalIn 0.3s ease" }} onClick={(e) => e.stopPropagation()}>
            <ShieldCheck size={36} color="#38bdf8" style={{ marginBottom: 8 }} />
            <div style={styles.modalTitle}>[ ارتقاء بالمستوى ]</div>
            <div style={styles.modalLevel}>Lv.{levelUpInfo.level}</div>
            <div style={{ ...styles.modalRank, color: levelUpInfo.rank.color }}>
              الرتبة الحالية: {levelUpInfo.rank.label}
            </div>
            <button style={styles.modalBtn} onClick={() => setLevelUpInfo(null)}>
              متابعة
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

function Section({ title, subtitle, onAdd, children }) {
  return (
    <div style={styles.section}>
      <div style={styles.sectionHeader}>
        <div>
          <div style={styles.sectionTitle}>{title}</div>
          <div style={styles.sectionSubtitle}>{subtitle}</div>
        </div>
        <button style={styles.addBtn} onClick={onAdd}>
          <Plus size={18} color="#38bdf8" />
        </button>
      </div>
      <div>{children}</div>
    </div>
  );
}

function QuestRow({ quest, onToggle, onRemove }) {
  return (
    <div className="quest-row" style={{ ...styles.questRow, opacity: quest.done ? 0.55 : 1 }}>
      <button style={styles.checkBox} onClick={onToggle}>
        {quest.done ? (
          <span style={{ color: "#38bdf8" }}>[✓]</span>
        ) : (
          <span style={{ color: "#4b5c73" }}>[ ]</span>
        )}
      </button>
      <div
        style={{
          ...styles.questTitle,
          textDecoration: quest.done ? "line-through" : "none",
        }}
        onClick={onToggle}
      >
        {quest.title}
      </div>
      <div style={styles.questXp}>+{quest.xp}</div>
      <button style={styles.removeBtn} onClick={onRemove}>
        <Trash2 size={15} color="#5a6b82" />
      </button>
    </div>
  );
}

function AddRow({ value, onChange, onSubmit, onCancel, placeholder }) {
  return (
    <div style={styles.addRow}>
      <input
        autoFocus
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder={placeholder}
        style={styles.input}
        onKeyDown={(e) => {
          if (e.key === "Enter") onSubmit();
          if (e.key === "Escape") onCancel();
        }}
      />
      <button style={styles.iconBtnSmall} onClick={onSubmit}>
        <Plus size={16} color="#38bdf8" />
      </button>
      <button style={styles.iconBtnSmall} onClick={onCancel}>
        <X size={16} color="#5a6b82" />
      </button>
    </div>
  );
}

function EmptyHint({ text }) {
  return <div style={styles.emptyHint}>{text}</div>;
}

const styles = {
  app: {
    minHeight: "100vh",
    background: "radial-gradient(circle at 50% 0%, #0d1626 0%, #05070d 55%)",
    color: "#e2e8f0",
    fontFamily: "'Segoe UI', system-ui, -apple-system, sans-serif",
    padding: "16px 14px 40px",
    maxWidth: 480,
    margin: "0 auto",
  },
  loadingScreen: {
    minHeight: "100vh",
    background: "#05070d",
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    color: "#38bdf8",
    fontFamily: "monospace",
  },
  loadingText: { letterSpacing: 1 },
  playerCard: {
    border: "1px solid rgba(56,189,248,0.35)",
    borderRadius: 14,
    background: "linear-gradient(180deg, rgba(20,30,48,0.9), rgba(10,14,22,0.9))",
    padding: "16px 18px",
    marginBottom: 20,
  },
  playerTop: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "flex-start",
    marginBottom: 12,
  },
  eyebrow: {
    fontFamily: "monospace",
    fontSize: 11,
    color: "#5a86ad",
    letterSpacing: 1,
    marginBottom: 4,
  },
  levelRow: { display: "flex", alignItems: "center", gap: 10 },
  levelNum: { fontSize: 26, fontWeight: 700, color: "#f1f5f9", fontFamily: "monospace" },
  rankBadge: {
    fontSize: 12,
    fontFamily: "monospace",
    border: "1px solid",
    borderRadius: 6,
    padding: "2px 8px",
  },
  streakBox: {
    display: "flex",
    alignItems: "center",
    gap: 4,
    fontFamily: "monospace",
    fontSize: 14,
    fontWeight: 600,
  },
  xpBarOuter: {
    width: "100%",
    height: 10,
    borderRadius: 6,
    background: "#131c2c",
    overflow: "hidden",
    border: "1px solid rgba(56,189,248,0.2)",
  },
  xpBarInner: {
    height: "100%",
    background: "linear-gradient(90deg, #38bdf8, #a78bfa)",
    transition: "width 0.4s ease",
  },
  xpLabel: {
    marginTop: 6,
    fontSize: 11,
    fontFamily: "monospace",
    color: "#6b84a3",
    textAlign: "left",
  },
  section: { marginBottom: 22 },
  sectionHeader: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    marginBottom: 10,
  },
  sectionTitle: { fontSize: 15, fontWeight: 700, color: "#e2e8f0" },
  sectionSubtitle: { fontSize: 11.5, color: "#5a6b82", marginTop: 2 },
  addBtn: {
    background: "rgba(56,189,248,0.1)",
    border: "1px solid rgba(56,189,248,0.3)",
    borderRadius: 8,
    width: 32,
    height: 32,
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    cursor: "pointer",
  },
  questRow: {
    display: "flex",
    alignItems: "center",
    gap: 10,
    background: "rgba(15,22,35,0.7)",
    border: "1px solid rgba(56,189,248,0.12)",
    borderRadius: 10,
    padding: "10px 12px",
    marginBottom: 8,
  },
  checkBox: {
    background: "none",
    border: "none",
    fontFamily: "monospace",
    fontSize: 15,
    cursor: "pointer",
    padding: 0,
  },
  questTitle: { flex: 1, fontSize: 14, cursor: "pointer", color: "#dbe4ee" },
  questXp: { fontSize: 12, fontFamily: "monospace", color: "#a78bfa" },
  removeBtn: { background: "none", border: "none", cursor: "pointer", padding: 4 },
  addRow: { display: "flex", gap: 6, marginBottom: 8 },
  input: {
    flex: 1,
    background: "#0d1524",
    border: "1px solid rgba(56,189,248,0.3)",
    borderRadius: 8,
    padding: "8px 10px",
    color: "#e2e8f0",
    fontSize: 13,
    outline: "none",
  },
  iconBtnSmall: {
    background: "rgba(56,189,248,0.08)",
    border: "1px solid rgba(56,189,248,0.25)",
    borderRadius: 8,
    width: 34,
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    cursor: "pointer",
  },
  emptyHint: {
    fontSize: 12.5,
    color: "#4b5c73",
    padding: "10px 4px",
    fontStyle: "italic",
  },
  modalOverlay: {
    position: "fixed",
    inset: 0,
    background: "rgba(2,5,10,0.75)",
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    zIndex: 50,
    padding: 20,
  },
  modal: {
    background: "linear-gradient(180deg, #0f1a2c, #060a12)",
    border: "1px solid rgba(56,189,248,0.5)",
    borderRadius: 16,
    padding: "28px 30px",
    textAlign: "center",
    boxShadow: "0 0 40px rgba(56,189,248,0.25)",
    maxWidth: 320,
  },
  modalTitle: {
    fontFamily: "monospace",
    fontSize: 13,
    color: "#7dd3fc",
    letterSpacing: 1,
    marginBottom: 6,
  },
  modalLevel: { fontSize: 32, fontWeight: 800, color: "#f1f5f9", marginBottom: 4 },
  modalRank: { fontSize: 13, fontFamily: "monospace", marginBottom: 18 },
  modalBtn: {
    background: "rgba(56,189,248,0.15)",
    border: "1px solid rgba(56,189,248,0.5)",
    color: "#e2e8f0",
    borderRadius: 8,
    padding: "8px 24px",
    fontSize: 14,
    cursor: "pointer",
  },
};
