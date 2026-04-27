const { useState, useRef, useEffect } = React;

// ── Datos ────────────────────────────────────────────────────────────────────
const MOOD_EMOJIS = [
  { emoji: "🌟", label: "Increíble", color: "#F9C74F" },
  { emoji: "😊", label: "Feliz",     color: "#90BE6D" },
  { emoji: "😐", label: "Neutral",   color: "#A8DADC" },
  { emoji: "😔", label: "Triste",    color: "#6D8DA8" },
  { emoji: "😤", label: "Enojado",   color: "#E07A5F" },
  { emoji: "😰", label: "Ansioso",   color: "#B5838D" },
];

const FAQ = [
  { q: "¿Cómo puedo manejar la ansiedad?",
    a: "La ansiedad es una señal de que algo importa. Prueba respirar 4-7-8: inhala 4s, retén 7s, exhala 8s. Repítelo 3 veces. También ayuda nombrar 5 cosas que ves, 4 que tocas, 3 que escuchas." },
  { q: "No me siento bien conmigo mismo/a",
    a: "Es valiente reconocerlo. Intenta escribir 3 cosas pequeñas que hiciste bien hoy. El cerebro tiende a ignorar lo positivo; entrenarlo lleva tiempo pero funciona." },
  { q: "Estoy muy estresado/a",
    a: "Escribe todo lo que te preocupa, luego sepáralo en 'puedo controlar' y 'no puedo controlar'. Soltar lo segundo libera energía real." },
  { q: "¿Cómo duermo mejor?",
    a: "Limita la cafeína después de las 2pm, baja la temperatura del cuarto, y 30 min antes apaga pantallas. Al acostarte, piensa en 3 momentos buenos del día." },
  { q: "Me siento solo/a",
    a: "La soledad es profundamente humana. A veces empezar pequeño ayuda: un mensaje a alguien que aprecias, o simplemente escribir aquí lo que sientes." },
  { q: "Quiero sentirme más motivado/a",
    a: "La motivación llega actuando. Empieza ridículamente pequeño: 2 minutos, una sola tarea. El movimiento crea el impulso." },
  { q: "¿Cómo manejo pensamientos negativos?",
    a: "Los pensamientos no son hechos. Cuando llegue uno negativo, pregúntate: ¿tengo evidencia real de esto? Escríbelo — externalizar el pensamiento le quita poder." },
];

// ── Helpers ──────────────────────────────────────────────────────────────────
const today = () => {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
};
const fmt = (dateStr) => {
  const [y,m,d] = dateStr.split('-');
  return new Date(y,m-1,d).toLocaleDateString('es-ES',{weekday:'long',day:'numeric',month:'long'});
};

// ── App ──────────────────────────────────────────────────────────────────────
function App() {
  const [tab, setTab]         = useState("chat");
  const [calOpen, setCalOpen] = useState(false);
  const [calPage, setCalPage] = useState({ y: new Date().getFullYear(), m: new Date().getMonth() });
  const [moods, setMoods]     = useState(() => { try { return JSON.parse(localStorage.getItem("moods")||"{}") } catch { return {} } });
  const [pickingMood, setPickingMood] = useState(null);

  // Chat
  const [messages, setMessages] = useState([
    { role:"ai", text:"Hola 🌿 Soy tu espacio seguro. Puedes preguntarme algo tocando una opción, o simplemente escribirme lo que sientes." }
  ]);
  const [input, setInput]   = useState("");
  const [loading, setLoading] = useState(false);
  const msgEnd = useRef(null);

  // Diary
  const [entries, setEntries]   = useState(() => { try { return JSON.parse(localStorage.getItem("diary")||"{}") } catch { return {} } });
  const [diaryDate, setDiaryDate] = useState(today());
  const [draftText, setDraftText] = useState("");
  const [saved, setSaved]       = useState(false);

  useEffect(() => { msgEnd.current?.scrollIntoView({behavior:"smooth"}) }, [messages]);
  useEffect(() => { setDraftText(entries[diaryDate]||"") }, [diaryDate]);

  // Mood
  const saveMood = (date, emoji) => {
    const next = emoji ? {...moods, [date]: emoji} : (() => { const c={...moods}; delete c[date]; return c; })();
    setMoods(next);
    localStorage.setItem("moods", JSON.stringify(next));
    setPickingMood(null);
  };

  // Diary
  const saveDiary = () => {
    const next = {...entries, [diaryDate]: draftText};
    setEntries(next);
    localStorage.setItem("diary", JSON.stringify(next));
    setSaved(true);
    setTimeout(()=>setSaved(false), 2000);
  };

  // AI chat — REEMPLAZÁ la API key real en producción
  const sendMsg = async (text) => {
    if (!text.trim()) return;
    setMessages(m=>[...m, {role:"user", text}]);
    setInput("");
    setLoading(true);
    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method:"POST",
        headers:{"Content-Type":"application/json"},
        body: JSON.stringify({
          model:"claude-sonnet-4-20250514",
          max_tokens:1000,
          system:`Eres un acompañante emocional cálido, empático y sabio. Respondes en español rioplatense, con frases cortas y directas. No eres terapeuta pero orientas con compasión. Usa lenguaje sencillo y humano. Máximo 4 oraciones por respuesta. No uses listas ni bullets.`,
          messages:[
            ...messages.filter((_,i)=>i>0).map(x=>({role:x.role==="ai"?"assistant":"user",content:x.text})),
            {role:"user",content:text}
          ]
        })
      });
      const data = await res.json();
      const reply = data.content?.map(c=>c.text||"").join("") || "Lo siento, no pude responder.";
      setMessages(m=>[...m, {role:"ai", text:reply}]);
    } catch {
      setMessages(m=>[...m, {role:"ai", text:"Hubo un problema de conexión. Intenta de nuevo 🌿"}]);
    }
    setLoading(false);
  };

  // Calendario
  const renderCal = () => {
    const { y, m } = calPage;
    const firstDay = new Date(y,m,1).getDay();
    const daysInMonth = new Date(y,m+1,0).getDate();
    const cells = [];
    for(let i=0;i<firstDay;i++) cells.push(null);
    for(let d=1;d<=daysInMonth;d++) cells.push(d);
    const mNames=["Ene","Feb","Mar","Abr","May","Jun","Jul","Ago","Sep","Oct","Nov","Dic"];
    const todayStr = today();
    return (
      <div style={s.calPopup}>
        <div style={s.calHeader}>
          <button style={s.calNav} onClick={()=>setCalPage(p=>p.m-1<0?{y:p.y-1,m:11}:{y:p.y,m:p.m-1})}>‹</button>
          <span style={s.calTitle}>{mNames[m]} {y}</span>
          <button style={s.calNav} onClick={()=>setCalPage(p=>p.m+1>11?{y:p.y+1,m:0}:{y:p.y,m:p.m+1})}>›</button>
        </div>
        <div style={s.calGrid}>
          {["D","L","M","X","J","V","S"].map(d=><div key={d} style={s.calWeekday}>{d}</div>)}
          {cells.map((d,i)=>{
            if(!d) return <div key={`e${i}`}/>;
            const ds=`${y}-${String(m+1).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
            return (
              <div key={ds} style={{...s.calDay,...(ds===todayStr?s.calToday:{})}} onClick={()=>setPickingMood(ds)}>
                <span style={{fontSize:"10px",opacity:.7}}>{d}</span>
                {moods[ds]&&<span style={{fontSize:"14px"}}>{moods[ds]}</span>}
              </div>
            );
          })}
        </div>
      </div>
    );
  };

  return (
    <div style={s.root}>
      {/* Header */}
      <div style={s.header}>
        <div style={{position:"relative"}}>
          <button style={s.calBtn} onClick={()=>setCalOpen(o=>!o)}>
            📅 <span style={{fontSize:"11px",opacity:.8}}>Estado</span>
          </button>
          {calOpen&&(
            <>
              <div style={s.overlay} onClick={()=>{setCalOpen(false);setPickingMood(null)}}/>
              {renderCal()}
            </>
          )}
        </div>
        <div style={s.headerTitle}>
          <span style={s.logo}>Alma</span>
          <span style={s.logoSub}>tu espacio interior</span>
        </div>
        <div style={s.tabs}>
          <button style={{...s.tab,...(tab==="chat"?s.tabActive:{})}} onClick={()=>setTab("chat")}>IA</button>
          <button style={{...s.tab,...(tab==="diary"?s.tabActive:{})}} onClick={()=>setTab("diary")}>Diario</button>
        </div>
      </div>

      {/* Mood picker */}
      {pickingMood&&(
        <div style={s.moodModal} onClick={()=>setPickingMood(null)}>
          <div style={s.moodBox} onClick={e=>e.stopPropagation()}>
            <p style={s.moodTitle}>¿Cómo te sentiste?</p>
            <p style={{fontSize:"12px",opacity:.6,marginBottom:"12px"}}>{fmt(pickingMood)}</p>
            <div style={s.moodGrid}>
              {MOOD_EMOJIS.map(({emoji,label,color})=>(
                <button key={emoji} style={{...s.moodBtn,background:moods[pickingMood]===emoji?color+"33":"transparent",borderColor:color+"66"}}
                  onClick={()=>saveMood(pickingMood,emoji)}>
                  <span style={{fontSize:"22px"}}>{emoji}</span>
                  <span style={{fontSize:"10px",opacity:.7}}>{label}</span>
                </button>
              ))}
            </div>
            {moods[pickingMood]&&(
              <button style={s.clearMood} onClick={()=>saveMood(pickingMood,undefined)}>Quitar registro</button>
            )}
          </div>
        </div>
      )}

      {/* Main */}
      <div style={s.main}>
        {tab==="chat" ? (
          <div style={s.chatWrap}>
            <div style={s.msgList}>
              {messages.map((msg,i)=>(
                <div key={i} style={{...s.bubble,...(msg.role==="user"?s.bubbleUser:s.bubbleAI)}}>
                  {msg.role==="ai"&&<span style={s.aiDot}>🌿</span>}
                  <p style={{lineHeight:1.6,fontSize:"14px"}}>{msg.text}</p>
                </div>
              ))}
              {loading&&(
                <div style={{...s.bubble,...s.bubbleAI}}>
                  <span style={s.aiDot}>🌿</span>
                  <div className="typing"><span/><span/><span/></div>
                </div>
              )}
              <div ref={msgEnd}/>
            </div>
            <div style={s.chips}>
              <p style={s.chipsLabel}>Preguntas frecuentes</p>
              <div style={s.chipsScroll}>
                {FAQ.map((f,i)=>(
                  <button key={i} style={s.chip} onClick={()=>sendMsg(f.q)}>{f.q}</button>
                ))}
              </div>
            </div>
            <div style={{...s.inputRow}} className="safe-bottom">
              <input style={s.input} placeholder="Escribe lo que sientes..." value={input}
                onChange={e=>setInput(e.target.value)}
                onKeyDown={e=>e.key==="Enter"&&sendMsg(input)}/>
              <button style={s.sendBtn} onClick={()=>sendMsg(input)} disabled={!input.trim()||loading}>↑</button>
            </div>
          </div>
        ) : (
          <div style={s.diaryWrap}>
            <div style={s.dateRow}>
              <button style={s.dateNav} onClick={()=>{
                const d=new Date(diaryDate+"T12:00:00"); d.setDate(d.getDate()-1);
                setDiaryDate(`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`);
              }}>‹</button>
              <div style={{textAlign:"center"}}>
                <p style={s.datePrimary}>{fmt(diaryDate)}</p>
                {moods[diaryDate]&&<span style={{fontSize:"18px"}}>{moods[diaryDate]}</span>}
              </div>
              <button style={{...s.dateNav,...(diaryDate>=today()?{opacity:.3,cursor:"default"}:{})}}
                onClick={()=>{
                  const d=new Date(diaryDate+"T12:00:00"); d.setDate(d.getDate()+1);
                  const next=`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
                  if(next<=today()) setDiaryDate(next);
                }}>›</button>
            </div>
            {!draftText&&(
              <div style={s.prompts}>
                {["¿Qué me alegró hoy?","¿Qué me pesó?","¿Qué agradezco?","¿Qué quiero soltar?"].map(p=>(
                  <button key={p} style={s.promptChip} onClick={()=>setDraftText(p+" ")}>{p}</button>
                ))}
              </div>
            )}
            <textarea style={s.textarea}
              placeholder="Este espacio es solo tuyo. Escribe sin filtros, sin reglas, sin juicio..."
              value={draftText} onChange={e=>setDraftText(e.target.value)}/>
            <div style={s.diaryFooter}>
              <span style={{fontSize:"12px",opacity:.4}}>{draftText.length} caracteres</span>
              <button style={{...s.saveBtn,...(saved?s.savedBtn:{})}} onClick={saveDiary}>
                {saved?"✓ Guardado":"Guardar"}
              </button>
            </div>
            {Object.keys(entries).filter(d=>d!==diaryDate&&entries[d]).sort().reverse().slice(0,3).map(d=>(
              <button key={d} style={s.prevEntry} onClick={()=>setDiaryDate(d)}>
                <span style={{fontSize:"13px",opacity:.6}}>{fmt(d)}</span>
                {moods[d]&&<span style={{marginLeft:"6px"}}>{moods[d]}</span>}
                <p style={{fontSize:"13px",opacity:.8,marginTop:"2px",overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap"}}>
                  {entries[d].slice(0,60)}…
                </p>
              </button>
            ))}
          </div>
        )}
      </div>
    </div>
  );
}

const s = {
  root:{ height:"100vh", display:"flex", flexDirection:"column", background:"#0f1117", maxWidth:"480px", margin:"0 auto", position:"relative", overflow:"hidden" },
  header:{ display:"flex", alignItems:"center", justifyContent:"space-between", padding:"14px 16px 10px", borderBottom:"1px solid #1e1e2e", background:"#0f1117", flexShrink:0, paddingTop:"calc(14px + env(safe-area-inset-top))" },
  logo:{ fontFamily:"'Playfair Display',serif", fontSize:"20px", color:"#c8b89a", letterSpacing:"1px" },
  logoSub:{ display:"block", fontSize:"10px", opacity:.5, letterSpacing:"1.5px", textTransform:"uppercase" },
  headerTitle:{ textAlign:"center", flex:1 },
  tabs:{ display:"flex", gap:"4px" },
  tab:{ background:"transparent", border:"1px solid #2a2a3a", color:"#888", padding:"5px 14px", borderRadius:"20px", cursor:"pointer", fontSize:"13px" },
  tabActive:{ background:"#c8b89a22", borderColor:"#c8b89a88", color:"#c8b89a" },
  calBtn:{ background:"transparent", border:"1px solid #2a2a3a", color:"#aaa", padding:"5px 10px", borderRadius:"20px", cursor:"pointer", fontSize:"13px", display:"flex", alignItems:"center", gap:"5px" },
  calPopup:{ position:"absolute", top:"50px", left:0, background:"#1a1a2e", border:"1px solid #2a2a3a", borderRadius:"12px", padding:"14px", width:"240px", zIndex:100, boxShadow:"0 8px 32px #00000088" },
  calHeader:{ display:"flex", justifyContent:"space-between", alignItems:"center", marginBottom:"10px" },
  calTitle:{ fontSize:"13px", color:"#c8b89a", fontFamily:"'Playfair Display',serif" },
  calNav:{ background:"transparent", border:"none", color:"#aaa", fontSize:"18px", cursor:"pointer", padding:"0 6px" },
  calGrid:{ display:"grid", gridTemplateColumns:"repeat(7,1fr)", gap:"2px" },
  calWeekday:{ fontSize:"9px", textAlign:"center", opacity:.4, padding:"2px 0" },
  calDay:{ display:"flex", flexDirection:"column", alignItems:"center", justifyContent:"center", padding:"2px", borderRadius:"6px", cursor:"pointer", minHeight:"30px", border:"1px solid transparent" },
  calToday:{ border:"1px solid #c8b89a55", background:"#c8b89a11" },
  overlay:{ position:"fixed", inset:0, zIndex:99 },
  moodModal:{ position:"fixed", inset:0, background:"#00000088", zIndex:200, display:"flex", alignItems:"center", justifyContent:"center" },
  moodBox:{ background:"#1a1a2e", border:"1px solid #2a2a3a", borderRadius:"16px", padding:"24px", width:"300px", textAlign:"center" },
  moodTitle:{ fontFamily:"'Playfair Display',serif", fontSize:"18px", color:"#c8b89a", marginBottom:"6px" },
  moodGrid:{ display:"grid", gridTemplateColumns:"repeat(3,1fr)", gap:"8px", marginBottom:"12px" },
  moodBtn:{ background:"transparent", border:"1px solid #3a3a4a", borderRadius:"10px", padding:"10px 6px", cursor:"pointer", display:"flex", flexDirection:"column", alignItems:"center", gap:"4px", color:"#e8e4dc" },
  clearMood:{ background:"transparent", border:"none", color:"#666", fontSize:"11px", cursor:"pointer", textDecoration:"underline" },
  main:{ flex:1, overflow:"hidden", display:"flex", flexDirection:"column" },
  chatWrap:{ flex:1, display:"flex", flexDirection:"column", overflow:"hidden" },
  msgList:{ flex:1, overflowY:"auto", padding:"16px", display:"flex", flexDirection:"column", gap:"10px" },
  bubble:{ maxWidth:"85%", padding:"12px 14px", borderRadius:"14px" },
  bubbleAI:{ alignSelf:"flex-start", background:"#1e1e2e", borderRadius:"4px 14px 14px 14px", display:"flex", gap:"8px", alignItems:"flex-start" },
  bubbleUser:{ alignSelf:"flex-end", background:"#c8b89a22", border:"1px solid #c8b89a44", borderRadius:"14px 4px 14px 14px" },
  aiDot:{ fontSize:"16px", flexShrink:0, marginTop:"1px" },
  chips:{ padding:"8px 16px", borderTop:"1px solid #1e1e2e", flexShrink:0 },
  chipsLabel:{ fontSize:"10px", opacity:.4, textTransform:"uppercase", letterSpacing:"1px", marginBottom:"6px" },
  chipsScroll:{ display:"flex", gap:"6px", overflowX:"auto", paddingBottom:"4px" },
  chip:{ flexShrink:0, background:"transparent", border:"1px solid #2a2a3a", color:"#bbb", padding:"6px 12px", borderRadius:"20px", fontSize:"12px", cursor:"pointer", whiteSpace:"nowrap" },
  inputRow:{ display:"flex", gap:"8px", padding:"10px 16px", borderTop:"1px solid #1e1e2e", flexShrink:0 },
  input:{ flex:1, background:"#1a1a2e", border:"1px solid #2a2a3a", borderRadius:"24px", padding:"10px 16px", color:"#e8e4dc", fontSize:"14px", outline:"none" },
  sendBtn:{ width:"40px", height:"40px", borderRadius:"50%", background:"#c8b89a", border:"none", color:"#0f1117", fontSize:"18px", cursor:"pointer", fontWeight:"bold" },
  diaryWrap:{ flex:1, overflowY:"auto", padding:"16px", display:"flex", flexDirection:"column", gap:"12px" },
  dateRow:{ display:"flex", justifyContent:"space-between", alignItems:"center" },
  dateNav:{ background:"transparent", border:"1px solid #2a2a3a", color:"#aaa", width:"34px", height:"34px", borderRadius:"50%", cursor:"pointer", fontSize:"18px" },
  datePrimary:{ fontFamily:"'Playfair Display',serif", fontSize:"16px", color:"#c8b89a", textTransform:"capitalize" },
  prompts:{ display:"flex", flexWrap:"wrap", gap:"6px" },
  promptChip:{ background:"transparent", border:"1px dashed #3a3a4a", color:"#aaa", padding:"5px 12px", borderRadius:"20px", fontSize:"12px", cursor:"pointer" },
  textarea:{ width:"100%", minHeight:"200px", background:"#1a1a2e", border:"1px solid #2a2a3a", borderRadius:"12px", padding:"14px", color:"#e8e4dc", fontSize:"14px", lineHeight:1.7, resize:"none", outline:"none" },
  diaryFooter:{ display:"flex", justifyContent:"space-between", alignItems:"center" },
  saveBtn:{ background:"#c8b89a", color:"#0f1117", border:"none", padding:"8px 20px", borderRadius:"20px", cursor:"pointer", fontSize:"13px", fontWeight:"500" },
  savedBtn:{ background:"#90BE6D" },
  prevEntry:{ width:"100%", background:"#1a1a2e", border:"1px solid #2a2a3a", borderRadius:"10px", padding:"10px 12px", cursor:"pointer", color:"#e8e4dc", textAlign:"left", marginBottom:"6px" },
};

ReactDOM.createRoot(document.getElementById('app')).render(<App/>);
