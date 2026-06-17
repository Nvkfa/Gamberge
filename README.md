<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0"/>
<title>index.html</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
<style>
  *{box-sizing:border-box;margin:0;padding:0;-webkit-tap-highlight-color:transparent;}
  body{background:#f1f5f9;font-family:'Inter',system-ui,sans-serif;overscroll-behavior:none;}
  input,select,button{font-family:inherit;}
  ::-webkit-scrollbar{width:0;}
  input[type=range]{accent-color:#6d28d9;}
</style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const {useState,useEffect,useRef} = React;

// ─── DATA ────────────────────────────────────────────────────────────────────
const DEFAULT_HABITS = [
  {id:"prayer_morning",label:"🌅 Prière du matin",  category:"spirituel",weight:15},
  {id:"prayer_evening",label:"🌙 Prière du soir",   category:"spirituel",weight:15},
  {id:"sport",         label:"🏋️ Sport / Exercice",      category:"sante",weight:15},
  {id:"stretching",    label:"🧘 Étirements / Mobilité 10min",category:"sante",weight:10},
  {id:"diet",          label:"🥗 Alimentation saine",     category:"sante",weight:10},
  {id:"sleep",         label:"😴 Couché avant minuit",     category:"sante",weight:5},
  {id:"nosocial",      label:"📵 Réseaux sociaux < 3h",    category:"sante",weight:10},
  {id:"study",         label:"📚 Révisions du jour",       category:"revisions",weight:15},
  {id:"daily_theme",   label:"🧠 Thème du jour appris",    category:"revisions",weight:15},
];

const CATEGORIES = [
  {id:"spirituel", label:"🕌 Spirituel", color:"#8b5cf6", bg:"#f5f3ff"},
  {id:"sante",     label:"💪 Santé",     color:"#22c55e", bg:"#f0fdf4"},
  {id:"revisions", label:"📖 Révisions", color:"#3b82f6", bg:"#eff6ff"},
];

const DAYS_FR   = ["Dim","Lun","Mar","Mer","Jeu","Ven","Sam"];
const MONTHS_FR = ["Janvier","Février","Mars","Avril","Mai","Juin","Juillet","Août","Septembre","Octobre","Novembre","Décembre"];
const MONTHS_SHORT = ["Jan","Fév","Mar","Avr","Mai","Jun","Jul","Aoû","Sep","Oct","Nov","Déc"];

// ─── STORAGE (localStorage) ───────────────────────────────────────────────────
const LS = {
  get:(k)=>{ try{const v=localStorage.getItem(k); return v?JSON.parse(v):null;}catch{return null;}},
  set:(k,v)=>{ try{localStorage.setItem(k,JSON.stringify(v));}catch{}},
};

// ─── UTILS ───────────────────────────────────────────────────────────────────
function toDateKey(d){ return d.toISOString().slice(0,10); }

function getWeekDays(ref){
  const d=new Date(ref);
  const mon=new Date(d);
  mon.setDate(d.getDate()-((d.getDay()+6)%7));
  return Array.from({length:7},(_,i)=>{const dd=new Date(mon);dd.setDate(mon.getDate()+i);return dd;});
}

function getMonthDays(y,m){
  const days=[];const d=new Date(y,m,1);
  while(d.getMonth()===m){days.push(new Date(d));d.setDate(d.getDate()+1);}
  return days;
}

function scoreForDay(log,habits){
  if(!log||Object.keys(log).length===0) return null;
  const total=habits.reduce((s,h)=>s+h.weight,0);
  const earned=habits.reduce((s,h)=>s+(log[h.id]?h.weight:0),0);
  return total>0?Math.round((earned/total)*100):0;
}

function catScore(log,habits,catId){
  if(!log) return null;
  const ch=habits.filter(h=>h.category===catId);
  const total=ch.reduce((s,h)=>s+h.weight,0);
  if(!total) return null;
  const earned=ch.reduce((s,h)=>s+(log[h.id]?h.weight:0),0);
  return Math.round((earned/total)*100);
}

// ─── COMPONENTS ──────────────────────────────────────────────────────────────
function ScoreRing({pct,size=80,stroke=8,color}){
  if(pct===null) return React.createElement('div',{
    style:{width:size,height:size,borderRadius:'50%',border:`${stroke}px solid #e5e7eb`,
      display:'flex',alignItems:'center',justifyContent:'center'}},
    React.createElement('span',{style:{color:'#9ca3af',fontSize:11}},'—'));
  const r=( size-stroke)/2, circ=2*Math.PI*r, dash=(pct/100)*circ;
  const col=color||(pct>=80?'#22c55e':pct>=50?'#eab308':'#ef4444');
  return React.createElement('div',{style:{position:'relative',width:size,height:size}},
    React.createElement('svg',{width:size,height:size,style:{transform:'rotate(-90deg)'}},
      React.createElement('circle',{cx:size/2,cy:size/2,r,fill:'none',stroke:'#e5e7eb',strokeWidth:stroke}),
      React.createElement('circle',{cx:size/2,cy:size/2,r,fill:'none',stroke:col,strokeWidth:stroke,
        strokeDasharray:`${dash} ${circ}`,strokeLinecap:'round'})),
    React.createElement('div',{style:{position:'absolute',inset:0,display:'flex',alignItems:'center',justifyContent:'center'}},
      React.createElement('span',{style:{fontWeight:700,fontSize:size*0.18,color:col}},`${pct}%`)));
}

function BarChart({days,logs,habits}){
  return React.createElement('div',{style:{display:'flex',alignItems:'flex-end',gap:4,height:80,padding:'0 4px'}},
    days.map((d,i)=>{
      const pct=scoreForDay(logs[toDateKey(d)],habits);
      const h=pct!==null?Math.max(4,(pct/100)*70):0;
      const col=pct===null?'#e5e7eb':pct>=80?'#22c55e':pct>=50?'#eab308':'#ef4444';
      return React.createElement('div',{key:i,style:{flex:1,display:'flex',flexDirection:'column',alignItems:'center',gap:2}},
        React.createElement('div',{style:{width:'100%',height:h,backgroundColor:col,borderRadius:3,transition:'height 0.3s'}}),
        React.createElement('span',{style:{fontSize:9,color:'#9ca3af'}},DAYS_FR[d.getDay()]));
    }));
}

// ─── MAIN APP (JSX) ──────────────────────────────────────────────────────────
function App(){
  const now = new Date();
  const [view,setView]           = useState('today');
  const [logs,setLogs]           = useState(()=>LS.get('pt_logs')||{});
  const [habits,setHabits]       = useState(()=>LS.get('pt_habits')||DEFAULT_HABITS);
  const [themes,setThemes]       = useState(()=>LS.get('pt_themes')||{});
  const [today]                  = useState(now);
  const [selDate,setSelDate]     = useState(now);
  const [newLabel,setNewLabel]   = useState('');
  const [newWeight,setNewWeight] = useState(10);
  const [newCat,setNewCat]       = useState('sante');
  const [themeInput,setThemeInput] = useState('');
  const [editTheme,setEditTheme] = useState(false);
  const [monthOffset,setMonthOffset] = useState(0);

  function saveLogs(v)  {setLogs(v);  LS.set('pt_logs',v);}
  function saveHabits(v){setHabits(v);LS.set('pt_habits',v);}
  function saveThemes(v){setThemes(v);LS.set('pt_themes',v);}

  function toggleHabit(dateKey,id){
    const prev=logs[dateKey]||{};
    saveLogs({...logs,[dateKey]:{...prev,[id]:!prev[id]}});
  }

  function confirmTheme(dateKey){
    if(!themeInput.trim()) return;
    saveThemes({...themes,[dateKey]:themeInput.trim()});
    setEditTheme(false);
  }

  function addHabit(){
    if(!newLabel.trim()) return;
    saveHabits([...habits,{id:`custom_${Date.now()}`,label:newLabel.trim(),category:newCat,weight:newWeight}]);
    setNewLabel(''); setNewWeight(10);
  }

  const todayKey  = toDateKey(today);
  const selKey    = toDateKey(selDate);
  const isToday   = selKey===todayKey;

  const weekDays  = getWeekDays(today);
  const weekScores= weekDays.map(d=>scoreForDay(logs[toDateKey(d)],habits)).filter(s=>s!==null);
  const weekAvg   = weekScores.length?Math.round(weekScores.reduce((a,b)=>a+b,0)/weekScores.length):null;

  // month with offset
  const displayMonth = new Date(today.getFullYear(), today.getMonth()+monthOffset, 1);
  const monthDays    = getMonthDays(displayMonth.getFullYear(), displayMonth.getMonth());
  const monthScores  = monthDays.map(d=>scoreForDay(logs[toDateKey(d)],habits)).filter(s=>s!==null);
  const monthAvg     = monthScores.length?Math.round(monthScores.reduce((a,b)=>a+b,0)/monthScores.length):null;
  const daysLogged   = monthScores.length;

  let streak=0;
  const tmp=new Date(today);
  while(true){
    const sc=scoreForDay(logs[toDateKey(tmp)],habits);
    if(sc!==null&&sc>=50){streak++;tmp.setDate(tmp.getDate()-1);}else break;
  }

  const S={
    app:{background:'#f1f5f9',minHeight:'100vh',maxWidth:430,margin:'0 auto',paddingBottom:80},
    header:{background:'#0f172a',color:'white',padding:'16px 18px 12px',position:'sticky',top:0,zIndex:10},
    hTitle:{fontSize:17,fontWeight:800,margin:0,letterSpacing:-0.5},
    hSub:{fontSize:11,color:'#94a3b8',marginTop:3},
    nav:{position:'fixed',bottom:0,left:'50%',transform:'translateX(-50%)',width:'100%',maxWidth:430,
      background:'white',borderTop:'1px solid #e2e8f0',display:'flex',zIndex:20},
    navBtn:(a)=>({flex:1,padding:'9px 0 5px',border:'none',background:'none',cursor:'pointer',
      fontSize:10,fontWeight:a?700:400,color:a?'#6d28d9':'#94a3b8',display:'flex',flexDirection:'column',alignItems:'center',gap:2}),
    card:{background:'white',borderRadius:16,margin:'10px 12px',padding:'14px 14px',boxShadow:'0 1px 4px rgba(0,0,0,0.07)'},
    catCard:(c)=>({background:c.bg,borderRadius:14,margin:'10px 12px',padding:'12px 14px',border:`1px solid ${c.color}33`}),
    catTitle:(c)=>({fontSize:12,fontWeight:800,color:c.color,marginBottom:8,textTransform:'uppercase',letterSpacing:0.8,display:'flex',alignItems:'center',gap:6}),
    sTitle:{fontSize:11,fontWeight:700,color:'#64748b',marginBottom:8,textTransform:'uppercase',letterSpacing:0.6},
    row:{display:'flex',alignItems:'center',justifyContent:'space-between',padding:'9px 0',borderBottom:'1px solid #f1f5f9'},
    lastRow:{display:'flex',alignItems:'center',justifyContent:'space-between',padding:'9px 0'},
    check:(c)=>({width:24,height:24,borderRadius:7,border:`2px solid ${c?'#22c55e':'#cbd5e1'}`,
      background:c?'#22c55e':'white',display:'flex',alignItems:'center',justifyContent:'center',
      cursor:'pointer',transition:'all 0.15s',flexShrink:0,userSelect:'none'}),
    tag:(p)=>({fontSize:10,fontWeight:700,padding:'2px 8px',borderRadius:20,
      background:p>=80?'#dcfce7':p>=50?'#fef9c3':'#fee2e2',color:p>=80?'#15803d':p>=50?'#92400e':'#b91c1c'}),
    statRow:{display:'flex',gap:8,marginBottom:8},
    stat:{flex:1,background:'#f8fafc',borderRadius:12,padding:'10px 6px',textAlign:'center'},
    statN:{fontSize:20,fontWeight:800,color:'#0f172a'},
    statL:{fontSize:9,color:'#94a3b8',marginTop:2},
    input:{border:'1px solid #e2e8f0',borderRadius:9,padding:'9px 12px',fontSize:14,
      width:'100%',background:'#f8fafc',outline:'none'},
    select:{border:'1px solid #e2e8f0',borderRadius:9,padding:'9px 12px',fontSize:13,
      background:'#f8fafc',outline:'none',width:'100%'},
    btn:(col='#6d28d9')=>({background:col,color:'white',border:'none',borderRadius:9,
      padding:'11px 16px',fontSize:13,fontWeight:700,cursor:'pointer',width:'100%'}),
    rmBtn:{background:'#fee2e2',color:'#ef4444',border:'none',borderRadius:6,
      padding:'3px 8px',fontSize:11,cursor:'pointer',fontWeight:600},
    arrowBtn:{background:'none',border:'none',cursor:'pointer',fontSize:22,color:'#334155',padding:'4px 10px'},
    themeBox:{background:'#fafafa',borderRadius:10,padding:'10px 12px',border:'1px dashed #cbd5e1',marginTop:4},
    dayCell:(p)=>({aspectRatio:'1',borderRadius:5,
      background:p===null?'#e2e8f0':p>=80?'#22c55e':p>=50?'#eab308':'#ef4444',
      opacity:p===null?0.4:1,display:'flex',alignItems:'center',justifyContent:'center'}),
    dateNav:{display:'flex',alignItems:'center',justifyContent:'space-between'},
  };

  const groupedHabits = CATEGORIES.map(cat=>({...cat,habits:habits.filter(h=>h.category===cat.id)}))
    .filter(g=>g.habits.length>0);

  const mKey = `${displayMonth.getFullYear()}-${String(displayMonth.getMonth()+1).padStart(2,'0')}`;

  return (
    <div style={S.app}>
      {/* HEADER */}
      <div style={S.header}>
        <p style={S.hTitle}>⚡ Life Tracker</p>
        <p style={S.hSub}>
          {view==='today' ? `${selDate.getDate()} ${MONTHS_SHORT[selDate.getMonth()]} ${selDate.getFullYear()}`
          :view==='week'  ? `Sem. du ${weekDays[0].getDate()} ${MONTHS_SHORT[weekDays[0].getMonth()]} → ${weekDays[6].getDate()} ${MONTHS_SHORT[weekDays[6].getMonth()]}`
          :view==='month' ? `${MONTHS_FR[displayMonth.getMonth()]} ${displayMonth.getFullYear()}`
          :'Paramètres'}
        </p>
      </div>

      {/* ══ TODAY ══ */}
      {view==='today' && <>
        {/* Date nav */}
        <div style={{...S.card,paddingTop:10,paddingBottom:10}}>
          <div style={S.dateNav}>
            <button style={S.arrowBtn} onClick={()=>{const d=new Date(selDate);d.setDate(d.getDate()-1);setSelDate(d);}}>‹</button>
            <div style={{textAlign:'center'}}>
              <div style={{fontSize:15,fontWeight:700,color:'#0f172a'}}>
                {DAYS_FR[selDate.getDay()]} {selDate.getDate()} {MONTHS_SHORT[selDate.getMonth()]}
              </div>
              {isToday && <div style={{fontSize:10,color:'#6d28d9',fontWeight:700}}>Aujourd'hui</div>}
            </div>
            <button style={{...S.arrowBtn,opacity:isToday?0.2:1}}
              onClick={()=>{if(!isToday){const d=new Date(selDate);d.setDate(d.getDate()+1);setSelDate(d);}}}>›</button>
          </div>
        </div>

        {/* Score global */}
        <div style={{...S.card,display:'flex',alignItems:'center',gap:14}}>
          <ScoreRing pct={scoreForDay(logs[selKey],habits)} size={90} stroke={9}/>
          <div style={{flex:1}}>
            <div style={{fontSize:14,fontWeight:800,color:'#0f172a'}}>Score du jour</div>
            <div style={{fontSize:12,color:'#64748b',marginTop:3}}>
              {habits.filter(h=>logs[selKey]?.[h.id]).length}/{habits.length} habitudes
            </div>
            {streak>0&&<div style={{marginTop:5,fontSize:12,color:'#f59e0b',fontWeight:700}}>
              🔥 {streak} jour{streak>1?'s':''} consécutifs ≥50%
            </div>}
            <div style={{display:'flex',gap:8,marginTop:8}}>
              {CATEGORIES.map(cat=>(
                <div key={cat.id} style={{textAlign:'center'}}>
                  <ScoreRing pct={catScore(logs[selKey],habits,cat.id)} size={34} stroke={4} color={cat.color}/>
                  <div style={{fontSize:8,color:cat.color,fontWeight:700,marginTop:2}}>{cat.label.split(' ')[0]}</div>
                </div>
              ))}
            </div>
          </div>
        </div>

        {/* Thème du jour */}
        <div style={S.card}>
          <div style={S.sTitle}>🧠 Thème du jour à apprendre</div>
          {!editTheme && themes[selKey] ? (
            <div style={S.themeBox}>
              <div style={{fontSize:14,color:'#1e293b',fontWeight:600}}>{themes[selKey]}</div>
              <button onClick={()=>{setThemeInput(themes[selKey]||'');setEditTheme(true);}}
                style={{marginTop:6,fontSize:11,color:'#94a3b8',background:'none',border:'none',cursor:'pointer',padding:0}}>
                ✏️ Modifier
              </button>
            </div>
          ) : (
            <div style={{display:'flex',flexDirection:'column',gap:8}}>
              <input style={S.input} placeholder="Ex: La loi des sinus, Le système nerveux…"
                value={themeInput} onChange={e=>setThemeInput(e.target.value)}
                onKeyDown={e=>{if(e.key==='Enter') confirmTheme(selKey);}}/>
              <button style={S.btn()} onClick={()=>confirmTheme(selKey)}>✓ Valider le thème</button>
            </div>
          )}
        </div>

        {/* Habitudes par catégorie */}
        {groupedHabits.map(cat=>(
          <div key={cat.id} style={S.catCard(cat)}>
            <div style={S.catTitle(cat)}>
              <span>{cat.label}</span>
              <span style={{marginLeft:'auto',fontSize:11,fontWeight:600,color:cat.color}}>
                {catScore(logs[selKey],habits,cat.id)!==null ? catScore(logs[selKey],habits,cat.id)+'%' : '—'}
              </span>
            </div>
            {cat.habits.map((h,i)=>{
              const checked=!!logs[selKey]?.[h.id];
              const isLast=i===cat.habits.length-1;
              return (
                <div key={h.id} style={isLast?S.lastRow:S.row}>
                  <div style={{display:'flex',alignItems:'center',gap:10}}>
                    <div style={S.check(checked)} onClick={()=>toggleHabit(selKey,h.id)}>
                      {checked&&<span style={{color:'white',fontSize:13,lineHeight:1}}>✓</span>}
                    </div>
                    <span style={{fontSize:13,color:checked?'#0f172a':'#64748b',fontWeight:checked?600:400}}>{h.label}</span>
                  </div>
                  <span style={{fontSize:11,color:cat.color,fontWeight:700}}>+{h.weight}%</span>
                </div>
              );
            })}
          </div>
        ))}
      </>}

      {/* ══ WEEK ══ */}
      {view==='week' && <>
        <div style={S.card}>
          <div style={S.sTitle}>Résumé de la semaine</div>
          <div style={S.statRow}>
            <div style={S.stat}>
              <div style={{display:'flex',justifyContent:'center'}}><ScoreRing pct={weekAvg} size={68} stroke={7}/></div>
              <div style={{...S.statL,marginTop:6}}>Moyenne</div>
            </div>
            <div style={S.stat}>
              <div style={S.statN}>{weekScores.filter(s=>s>=80).length}</div>
              <div style={S.statL}>Jours ≥80%</div>
            </div>
            <div style={S.stat}>
              <div style={S.statN}>{weekScores.length}</div>
              <div style={S.statL}>Jours trackés</div>
            </div>
          </div>
          <BarChart days={weekDays} logs={logs} habits={habits}/>
        </div>

        <div style={S.card}>
          <div style={S.sTitle}>Détail par jour</div>
          {weekDays.map((d,i)=>{
            const key=toDateKey(d);
            const pct=scoreForDay(logs[key],habits);
            const isT=key===todayKey;
            const isLast=i===6;
            return (
              <div key={i} style={isLast?S.lastRow:S.row}>
                <div>
                  <span style={{fontSize:13,fontWeight:isT?700:400,color:isT?'#6d28d9':'#374151'}}>
                    {DAYS_FR[d.getDay()]} {d.getDate()}
                  </span>
                  {themes[key]&&<div style={{fontSize:10,color:'#94a3b8',marginTop:1}}>🧠 {themes[key]}</div>}
                </div>
                {pct!==null?<span style={S.tag(pct)}>{pct}%</span>:<span style={{fontSize:12,color:'#d1d5db'}}>—</span>}
              </div>
            );
          })}
        </div>

        {CATEGORIES.map(cat=>{
          const ch=habits.filter(h=>h.category===cat.id);
          if(!ch.length) return null;
          return (
            <div key={cat.id} style={S.catCard(cat)}>
              <div style={S.catTitle(cat)}>{cat.label}</div>
              {ch.map((h,i)=>{
                const count=weekDays.filter(d=>logs[toDateKey(d)]?.[h.id]).length;
                const pct=Math.round((count/7)*100);
                const isLast=i===ch.length-1;
                return (
                  <div key={h.id} style={isLast?S.lastRow:S.row}>
                    <span style={{fontSize:13,color:'#374151'}}>{h.label}</span>
                    <div style={{display:'flex',alignItems:'center',gap:8}}>
                      <div style={{width:55,height:5,background:'#e5e7eb',borderRadius:3,overflow:'hidden'}}>
                        <div style={{width:`${pct}%`,height:'100%',background:cat.color,borderRadius:3}}/>
                      </div>
                      <span style={{fontSize:11,color:'#64748b',minWidth:28,textAlign:'right'}}>{count}/7</span>
                    </div>
                  </div>
                );
              })}
            </div>
          );
        })}
      </>}

      {/* ══ MONTH ══ */}
      {view==='month' && <>
        {/* Month nav */}
        <div style={{...S.card,paddingTop:10,paddingBottom:10}}>
          <div style={S.dateNav}>
            <button style={S.arrowBtn} onClick={()=>setMonthOffset(o=>o-1)}>‹</button>
            <div style={{fontSize:14,fontWeight:700,color:'#0f172a'}}>
              {MONTHS_FR[displayMonth.getMonth()]} {displayMonth.getFullYear()}
            </div>
            <button style={{...S.arrowBtn,opacity:monthOffset===0?0.2:1}}
              onClick={()=>{if(monthOffset<0)setMonthOffset(o=>o+1);}}>›</button>
          </div>
        </div>

        <div style={S.card}>
          <div style={S.statRow}>
            <div style={S.stat}>
              <div style={{display:'flex',justifyContent:'center'}}><ScoreRing pct={monthAvg} size={68} stroke={7}/></div>
              <div style={{...S.statL,marginTop:6}}>Moy. mois</div>
            </div>
            <div style={S.stat}>
              <div style={S.statN}>{daysLogged}</div>
              <div style={S.statL}>Jours trackés</div>
            </div>
            <div style={S.stat}>
              <div style={S.statN}>{monthScores.filter(s=>s>=80).length}</div>
              <div style={S.statL}>Jours ≥80%</div>
            </div>
          </div>
          <div style={{display:'flex',justifyContent:'space-around',marginTop:6}}>
            {CATEGORIES.map(cat=>{
              const scores=monthDays.map(d=>catScore(logs[toDateKey(d)],habits,cat.id)).filter(s=>s!==null);
              const avg=scores.length?Math.round(scores.reduce((a,b)=>a+b,0)/scores.length):null;
              return (
                <div key={cat.id} style={{textAlign:'center'}}>
                  <ScoreRing pct={avg} size={52} stroke={5} color={cat.color}/>
                  <div style={{fontSize:9,color:cat.color,fontWeight:700,marginTop:3}}>{cat.label}</div>
                </div>
              );
            })}
          </div>
        </div>

        {/* Heatmap */}
        <div style={S.card}>
          <div style={S.sTitle}>Heatmap</div>
          <div style={{display:'grid',gridTemplateColumns:'repeat(7,1fr)',gap:3,marginBottom:4}}>
            {DAYS_FR.map(d=><div key={d} style={{textAlign:'center',fontSize:8,color:'#94a3b8'}}>{d}</div>)}
          </div>
          <div style={{display:'grid',gridTemplateColumns:'repeat(7,1fr)',gap:3}}>
            {Array.from({length:(monthDays[0].getDay()+6)%7}).map((_,i)=><div key={`p${i}`}/>)}
            {monthDays.map((d,i)=>{
              const key=toDateKey(d);
              const pct=scoreForDay(logs[key],habits);
              const isT=key===todayKey;
              return (
                <div key={i} style={{...S.dayCell(pct),border:isT?'2px solid #0f172a':'none'}}>
                  <span style={{fontSize:7,color:'white',fontWeight:700,opacity:0.9}}>{d.getDate()}</span>
                </div>
              );
            })}
          </div>
          <div style={{display:'flex',gap:10,marginTop:8,flexWrap:'wrap'}}>
            {[['#e2e8f0','—'],['#ef4444','<50%'],['#eab308','50–79%'],['#22c55e','≥80%']].map(([c,l])=>(
              <div key={l} style={{display:'flex',alignItems:'center',gap:4}}>
                <div style={{width:10,height:10,background:c,borderRadius:2}}/>
                <span style={{fontSize:9,color:'#64748b'}}>{l}</span>
              </div>
            ))}
          </div>
        </div>

        {/* Thèmes du mois */}
        {Object.entries(themes).filter(([k])=>k.startsWith(mKey)).length>0&&(
          <div style={S.card}>
            <div style={S.sTitle}>🧠 Thèmes appris</div>
            {Object.entries(themes).filter(([k])=>k.startsWith(mKey)).sort(([a],[b])=>a.localeCompare(b)).map(([k,v])=>(
              <div key={k} style={S.row}>
                <span style={{fontSize:11,color:'#64748b'}}>{k.slice(8)}/{k.slice(5,7)}</span>
                <span style={{fontSize:12,color:'#1e293b',fontWeight:500,maxWidth:'75%',textAlign:'right'}}>{v}</span>
              </div>
            ))}
          </div>
        )}

        {/* Top habitudes */}
        <div style={S.card}>
          <div style={S.sTitle}>Top habitudes du mois</div>
          {[...habits].sort((a,b)=>{
            const ca=monthDays.filter(d=>logs[toDateKey(d)]?.[a.id]).length;
            const cb=monthDays.filter(d=>logs[toDateKey(d)]?.[b.id]).length;
            return cb-ca;
          }).map((h,i)=>{
            const count=monthDays.filter(d=>logs[toDateKey(d)]?.[h.id]).length;
            const pct=daysLogged?Math.round((count/daysLogged)*100):0;
            const cat=CATEGORIES.find(c=>c.id===h.category);
            const isLast=i===habits.length-1;
            return (
              <div key={h.id} style={isLast?S.lastRow:S.row}>
                <span style={{fontSize:12,color:'#374151'}}>{h.label}</span>
                <div style={{display:'flex',alignItems:'center',gap:6}}>
                  <div style={{width:50,height:5,background:'#e5e7eb',borderRadius:3,overflow:'hidden'}}>
                    <div style={{width:`${pct}%`,height:'100%',background:cat?.color||'#22c55e',borderRadius:3}}/>
                  </div>
                  <span style={{fontSize:11,color:'#64748b',minWidth:38,textAlign:'right'}}>{count}/{daysLogged}j</span>
                </div>
              </div>
            );
          })}
        </div>
      </>}

      {/* ══ SETTINGS ══ */}
      {view==='settings' && <>
        {CATEGORIES.map(cat=>{
          const ch=habits.filter(h=>h.category===cat.id);
          const total=ch.reduce((s,h)=>s+h.weight,0);
          return (
            <div key={cat.id} style={S.catCard(cat)}>
              <div style={S.catTitle(cat)}>
                {cat.label}<span style={{fontSize:10,fontWeight:500,color:cat.color,marginLeft:4}}>({total}pts)</span>
              </div>
              {ch.map((h,i)=>{
                const isLast=i===ch.length-1;
                const isDefault=DEFAULT_HABITS.find(d=>d.id===h.id);
                return (
                  <div key={h.id} style={isLast?S.lastRow:S.row}>
                    <span style={{fontSize:13,color:'#374151'}}>{h.label}</span>
                    <div style={{display:'flex',alignItems:'center',gap:8}}>
                      <span style={{fontSize:12,color:cat.color,fontWeight:700}}>+{h.weight}%</span>
                      {!isDefault&&(
                        <button style={S.rmBtn} onClick={()=>saveHabits(habits.filter(x=>x.id!==h.id))}>✕</button>
                      )}
                    </div>
                  </div>
                );
              })}
            </div>
          );
        })}

        <div style={S.card}>
          <div style={S.sTitle}>+ Ajouter une habitude</div>
          <div style={{display:'flex',flexDirection:'column',gap:10}}>
            <input style={S.input} placeholder="Ex: 📖 Lecture 30 min" value={newLabel}
              onChange={e=>setNewLabel(e.target.value)}/>
            <select style={S.select} value={newCat} onChange={e=>setNewCat(e.target.value)}>
              {CATEGORIES.map(c=><option key={c.id} value={c.id}>{c.label}</option>)}
            </select>
            <div style={{display:'flex',alignItems:'center',gap:10}}>
              <label style={{fontSize:13,color:'#374151',whiteSpace:'nowrap'}}>Poids : {newWeight}%</label>
              <input type="range" min={5} max={50} step={5} value={newWeight}
                onChange={e=>setNewWeight(Number(e.target.value))} style={{flex:1}}/>
            </div>
            <button style={S.btn()} onClick={addHabit}>+ Ajouter</button>
          </div>
        </div>

        <div style={S.card}>
          <div style={S.sTitle}>Légende scores</div>
          <div style={{padding:10,background:'#f8fafc',borderRadius:10}}>
            {[['🟢','≥80%','Journée excellente'],['🟡','50–79%','Journée correcte'],['🔴','<50%','À améliorer']].map(([e,p,l])=>(
              <div key={p} style={{fontSize:12,color:'#374151',padding:'4px 0'}}>{e} {p} — {l}</div>
            ))}
          </div>
        </div>

        <div style={S.card}>
          <div style={S.sTitle}>Données</div>
          <p style={{fontSize:12,color:'#64748b',marginBottom:10}}>
            Tes données sont sauvegardées localement sur cet appareil/navigateur. Pour ne pas les perdre, évite de vider le cache de ce navigateur.
          </p>
          <button style={S.btn('#ef4444')} onClick={()=>{
            if(confirm('Effacer TOUTES les données ? Cette action est irréversible.')){
              localStorage.removeItem('pt_logs');localStorage.removeItem('pt_habits');localStorage.removeItem('pt_themes');
              setLogs({});setHabits(DEFAULT_HABITS);setThemes({});
            }
          }}>🗑️ Réinitialiser toutes les données</button>
        </div>
      </>}

      {/* NAV */}
      <nav style={S.nav}>
        {[{id:'today',icon:'📅',label:"Aujourd'hui"},{id:'week',icon:'📊',label:'Semaine'},
          {id:'month',icon:'📆',label:'Mois'},{id:'settings',icon:'⚙️',label:'Habitudes'}].map(({id,icon,label})=>(
          <button key={id} style={S.navBtn(view===id)} onClick={()=>setView(id)}>
            <span style={{fontSize:20}}>{icon}</span>{label}
          </button>
        ))}
      </nav>
    </div>
  );
}

ReactDOM.createRoot(document.getElementById('root')).render(React.createElement(App));
</script>
</body>
</html>
