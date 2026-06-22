<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Six Easy Pieces, Interactive</title>
<style>
*{box-sizing:border-box}
html,body{margin:0;padding:0}
body{font-family:Georgia,'Times New Roman',serif;color:#000;background:#fff;line-height:1.5;max-width:820px;margin:0 auto;padding:22px 18px 44px}
h1{font-size:1.9em;font-weight:bold;margin:0 0 2px}
h2{font-size:1.3em;margin:1.5em 0 .25em}
.byline{font-style:italic;color:#333;margin:0 0 1.1em}
.intro{margin:0 0 .35em}
#nav{line-height:1.95}
#nav a{color:#0000cc}
hr{border:0;border-top:1px solid #aaa;margin:1.1em 0}
.lead{margin:.25em 0 1em}
.lead .em,.notebox .q{font-style:italic}
canvas{display:block;background:#0b0b0b;border:1px solid #000;max-width:100%;width:100%;margin:.6em 0}
.controls{margin:.5em 0 .2em}
.controls .ctl{display:inline-block;vertical-align:top;margin:0 1.6em .7em 0}
.controls label{display:block;font-size:.82em;margin-bottom:2px}
.controls input[type=range]{width:200px;vertical-align:middle}
.controls .val{font-weight:bold}
.btnrow{margin-top:2px}
button{font-family:inherit;font-size:.95em;margin:0 4px 0 0;padding:1px 7px}
.seg{display:inline-block}
.seg button[aria-pressed=true]{font-weight:bold;text-decoration:underline}
button.on{font-weight:bold}
.readouts{font-family:monospace;font-size:.88em;margin:.3em 0 .2em}
.readouts .ro{display:inline-block;margin-right:1.6em}
.readouts .ro .k::after{content:': '}
.readouts .ro .v{font-weight:bold}
.readouts .ro .v .u{font-weight:normal;margin-left:2px}
.statepill{display:inline-block;font-family:monospace;font-size:.88em;font-weight:bold;margin-right:1.6em}
.notebox{border-left:3px solid #000;padding-left:12px;margin:1em 0;font-size:.95em}
.ladder-meta{font-family:monospace;font-size:.88em;margin:.3em 0}
.ladder-meta .scale{font-weight:bold}
.sim-wrap{position:relative}
.sidepanel{border:1px solid #000;padding:12px;margin:.6em 0;display:none}
.sidepanel.show{display:block}
.sidepanel h3{margin:0 0 4px;font-size:1.1em}
.sidepanel .tag{font-family:monospace;font-size:.78em;text-transform:uppercase;color:#333;margin-bottom:5px}
.sidepanel .hint{font-size:.85em;color:#333}
.foot{font-size:.85em;color:#333;margin-top:1.2em}
.vis-hidden{position:absolute;left:-9999px}
.menutoggle,.scrim{display:none}
.lead,.notebox,.readouts,.ladder-meta,.sidepanel,.sidepanel.show,.controls label{display:none!important}
.controls{margin-top:.4em}
</style>
</head>
<body>
<h1>Six Easy Pieces</h1>
<nav id="nav"></nav>
<hr>
<main id="stage"></main>

<script>
"use strict";
/* =====================================================================
   SIX EASY PIECES, INTERACTIVE
   Each piece is a self-contained module: {meta, render(stage), stop()}.
   The router mounts one at a time and stops the previous animation loop.
   ===================================================================== */

const REDUCED = window.matchMedia && window.matchMedia('(prefers-reduced-motion: reduce)').matches;

/* ---- small helpers ------------------------------------------------- */
function el(tag, attrs, kids){
  const n = document.createElement(tag);
  if(attrs) for(const k in attrs){
    if(k==='class') n.className = attrs[k];
    else if(k==='html') n.innerHTML = attrs[k];
    else if(k==='text') n.textContent = attrs[k];
    else if(k.startsWith('on')&&typeof attrs[k]==='function') n.addEventListener(k.slice(2),attrs[k]);
    else n.setAttribute(k, attrs[k]);
  }
  if(kids) (Array.isArray(kids)?kids:[kids]).forEach(c=>{ if(c!=null) n.appendChild(typeof c==='string'?document.createTextNode(c):c); });
  return n;
}
function fitCanvas(canvas, h){
  const dpr = Math.min(window.devicePixelRatio||1, 2);
  const w = canvas.clientWidth || canvas.parentElement.clientWidth;
  canvas.width = Math.round(w*dpr);
  canvas.height = Math.round(h*dpr);
  canvas.style.height = h+'px';
  const ctx = canvas.getContext('2d');
  ctx.setTransform(dpr,0,0,dpr,0,0);
  return {ctx, w, h, dpr};
}
function clamp(v,a,b){return v<a?a:v>b?b:v;}
function lerp(a,b,t){return a+(b-a)*t;}

/* RAF manager so only the active piece animates */
const Loop = (function(){
  let id=null, fn=null;
  function frame(t){ if(fn){ fn(t); id=requestAnimationFrame(frame);} }
  return {
    start(f){ this.stop(); fn=f; id=requestAnimationFrame(frame); },
    stop(){ if(id) cancelAnimationFrame(id); id=null; fn=null; }
  };
})();

/* Build the shared header for a piece, returns the stage container */
function pieceHeader(p){
  const stage = document.getElementById('stage');
  stage.innerHTML='';
  return stage;
}

/* =====================================================================
   PIECE I  -  ATOMS IN MOTION
   A 2D molecular-dynamics box. A bond model (rest-length springs within
   a cutoff) plus temperature-scaled thermal kicks gives a real
   solid -> liquid -> gas transition as you raise the temperature.
   ===================================================================== */
const PieceAtoms = {
  color:'var(--c1)', roman:'I', eyebrow:'Piece One',
  title:'Atoms in Motion',
  lead:'Heat is just atoms <span class="em">jiggling</span>. Turn up the temperature: solid, liquid, gas.',
  raf:null,
  render(){
    const stage = pieceHeader(this);
    const panel = el('div',{class:'panel'});
    const wrap = el('div',{class:'sim-wrap'});
    const canvas = el('canvas',{class:'sim-canvas'});
    wrap.appendChild(canvas);
    panel.appendChild(wrap);

    // readouts
    const roTemp = el('span',{class:'v'}); const roState=el('span',{class:'statepill'});
    const roPress = el('span',{class:'v'});
    const readouts = el('div',{class:'readouts'},[
      el('div',{class:'ro'},[el('span',{class:'k',text:'Temperature'}),roTemp]),
      el('div',{class:'ro'},[el('span',{class:'k',text:'Wall pressure'}),roPress]),
      roState
    ]);
    panel.appendChild(readouts);

    // controls
    const tempVal = el('span',{class:'val'});
    const slider = el('input',{type:'range',min:'0',max:'100',value:'8'});
    const slabel = el('label',[document.createTextNode('Temperature  '),tempVal]);
    const reset = el('button',{class:'action',text:'Reset'});
    panel.appendChild(el('div',{class:'controls'},[
      el('div',{class:'ctl'},[slabel,slider]),
      el('div',{class:'ctl',style:'min-width:auto'},[el('label',{text:'Reset'}),el('div',{class:'btnrow'},reset)])
    ]));
    stage.appendChild(panel);

    stage.appendChild(el('div',{class:'notebox',html:'Cold locks atoms into a lattice; heat melts it, then boils it into a wall-pounding gas. Pressure is those collisions.'}));

    // ---- simulation state ----
    let W=0,H=0; const N=90;
    const px=new Float32Array(N),py=new Float32Array(N),vx=new Float32Array(N),vy=new Float32Array(N);
    const D=26;          // rest spacing
    const CUT=D*1.6;     // bond cutoff
    const KREP=0.9, KATT=0.06, DAMP=0.985;
    let temp=0.08, wallHits=0, pressEMA=0;

    function seedCrystal(){
      // triangular lattice packed near center
      let i=0; const cols=10, rows=9, ox=0, oy=0;
      const startX=()=> (W- (cols-1)*D)/2;
      for(let r=0;r<rows&&i<N;r++){
        for(let c=0;c<cols&&i<N;c++){
          px[i]=startX()+c*D + (r%2?D/2:0) + ox;
          py[i]=(H-(rows-1)*D*0.92)/2 + r*D*0.92 + oy;
          vx[i]=0; vy[i]=0; i++;
        }
      }
      for(;i<N;i++){px[i]=W/2;py[i]=H/2;vx[i]=0;vy[i]=0;}
    }
    function size(){ const m=fitCanvas(canvas, Math.min(520, Math.max(360, canvas.clientWidth*0.62))); W=m.w; H=m.h; }
    size(); seedCrystal();
    const onResize=()=>{ const oldW=W; size(); if(Math.abs(oldW-W)>2){ /* keep positions in bounds */ for(let i=0;i<N;i++){px[i]=clamp(px[i],8,W-8);py[i]=clamp(py[i],8,H-8);} } };
    window.addEventListener('resize',onResize); this._onResize=onResize;

    function step(){
      // pairwise bond forces (O(N^2), fine for N=90)
      for(let i=0;i<N;i++){
        for(let j=i+1;j<N;j++){
          let dx=px[j]-px[i], dy=py[j]-py[i];
          let r2=dx*dx+dy*dy; if(r2>CUT*CUT||r2<0.0001) continue;
          let r=Math.sqrt(r2), nx=dx/r, ny=dy/r, f;
          if(r<D){ f=-KREP*(D-r); }          // repel when squeezed
          else  { f= KATT*(r-D); }            // attract when stretched (within cutoff)
          const fx=f*nx, fy=f*ny;
          vx[i]+=fx; vy[i]+=fy; vx[j]-=fx; vy[j]-=fy;
        }
      }
      // thermal kicks scaled by temperature + damping
      const kick=temp*temp*3.4;
      for(let i=0;i<N;i++){
        vx[i]+= (Math.random()*2-1)*kick;
        vy[i]+= (Math.random()*2-1)*kick;
        vx[i]*=DAMP; vy[i]*=DAMP;
        px[i]+=vx[i]; py[i]+=vy[i];
        // walls
        if(px[i]<8){px[i]=8;vx[i]=Math.abs(vx[i]);wallHits++;}
        if(px[i]>W-8){px[i]=W-8;vx[i]=-Math.abs(vx[i]);wallHits++;}
        if(py[i]<8){py[i]=8;vy[i]=Math.abs(vy[i]);wallHits++;}
        if(py[i]>H-8){py[i]=H-8;vy[i]=-Math.abs(vy[i]);wallHits++;}
      }
    }

    function speedColor(s){
      // cold blue -> warm coral -> hot yellow
      const t=clamp(s/4.5,0,1);
      const r=Math.round(lerp(90,255,t)), g=Math.round(lerp(150,200,t*0.7)), b=Math.round(lerp(255,70,t));
      return `rgb(${r},${g},${b})`;
    }

    let frame=0;
    function draw(){
      const ctx=canvas.getContext('2d');
      ctx.clearRect(0,0,W,H);
      // bonds (only in solid/liquid-ish range, faint)
      ctx.lineWidth=1.4;
      for(let i=0;i<N;i++){
        for(let j=i+1;j<N;j++){
          let dx=px[j]-px[i],dy=py[j]-py[i],r2=dx*dx+dy*dy;
          if(r2<(D*1.25)*(D*1.25)){
            const a=clamp(1-(Math.sqrt(r2)/(D*1.25)),0,1)*0.4;
            ctx.strokeStyle=`rgba(255,160,120,${a})`;
            ctx.beginPath();ctx.moveTo(px[i],py[i]);ctx.lineTo(px[j],py[j]);ctx.stroke();
          }
        }
      }
      for(let i=0;i<N;i++){
        const s=Math.hypot(vx[i],vy[i]);
        const col=speedColor(s);
        ctx.beginPath();ctx.arc(px[i],py[i],8,0,7);
        ctx.fillStyle=col;ctx.fill();
      }
      // readouts (throttled)
      if((frame++ & 7)===0){
        let ke=0; for(let i=0;i<N;i++) ke+=vx[i]*vx[i]+vy[i]*vy[i];
        const tShown=Math.round(ke/N*120);
        roTemp.innerHTML = tShown+'<span class="u">arb. K</span>';
        pressEMA = pressEMA*0.9 + wallHits*0.1; wallHits=0;
        roPress.innerHTML = (pressEMA*4).toFixed(0)+'<span class="u">hits/s</span>';
        // state classification by temperature setting + mobility
        let label;
        if(temp<0.22) label='solid';
        else if(temp<0.55) label='liquid';
        else label='gas';
        roState.textContent='State: '+label;
      }
    }
    function tick(){ step(); draw(); }
    if(!REDUCED) Loop.start(tick); else { for(let k=0;k<60;k++) step(); draw(); }

    slider.oninput=()=>{
      temp = Math.pow(slider.value/100,1.4)*1.0 + 0.02;
      tempVal.textContent = slider.value;
      slider.style.setProperty('--fill', slider.value+'%');
      if(REDUCED){ for(let k=0;k<30;k++) step(); draw(); }
    };
    slider.oninput(); 
    reset.onclick=()=>{ seedCrystal(); slider.value=8; slider.oninput(); };

    this.raf=true;
  },
  stop(){ Loop.stop(); if(this._onResize) window.removeEventListener('resize',this._onResize); }
};

/* =====================================================================
   PIECE II  -  BASIC PHYSICS
   Physics tries to compress everything into a few fundamental laws and
   forces. Here you climb down the ladder of scale, from a solid object
   to the quarks inside a proton, and at each rung you see what force
   holds that level together.
   ===================================================================== */
const PieceBasic = {
  color:'var(--c2)', roman:'II', eyebrow:'Piece Two',
  title:'Basic Physics',
  lead:'All of physics is a few rules. Zoom down to the <span class="em">forces</span> holding each scale together.',
  levels:[
    {scale:'1 m', name:'A solid object', force:'Electromagnetism',
     blurb:'Electric forces between atoms make solids hold their shape.', draw:'object'},
    {scale:'1 nm', name:'Molecules', force:'Electromagnetism',
     blurb:'Atoms bond into molecules. The bonds are electric.', draw:'molecule'},
    {scale:'0.1 nm', name:'A single atom', force:'Electric attraction',
     blurb:'Electrons held to the nucleus by electric attraction.', draw:'atom'},
    {scale:'10\u207B\u00B9\u2074 m', name:'The nucleus', force:'The strong force',
     blurb:'Protons and neutrons glued by the strong nuclear force.', draw:'nucleus'},
    {scale:'10\u207B\u00B9\u2075 m', name:'Quarks', force:'The strong force',
     blurb:'Three quarks per proton, bound so tight you cannot pull one out.', draw:'quark'}
  ],
  render(){
    const stage=pieceHeader(this);
    const panel=el('div',{class:'panel'});
    const wrap=el('div',{class:'sim-wrap'});
    const canvas=el('canvas',{class:'sim-canvas'});
    wrap.appendChild(canvas); panel.appendChild(wrap);

    const scaleEl=el('span',{class:'scale'});
    const forceEl=el('span',{style:'color:var(--text);font-weight:700'});
    panel.appendChild(el('div',{class:'ladder-meta'},[
      el('span',[document.createTextNode('Scale  '),scaleEl]),
      el('span',[document.createTextNode('Held together by  '),forceEl])
    ]));

    const zoomIn=el('button',{class:'action primary',text:'Zoom in \u2193'});
    const zoomOut=el('button',{class:'action',text:'\u2191 Zoom out'});
    panel.appendChild(el('div',{class:'controls'},[
      el('div',{class:'ctl',style:'min-width:auto'},[el('label',{text:'Descend the ladder of scale'}),el('div',{class:'btnrow'},[zoomOut,zoomIn])])
    ]));
    stage.appendChild(panel);
    const note=el('div',{class:'notebox'});
    stage.appendChild(note);

    let W=0,H=0,idx=0,anim=0,target=0;
    const size=()=>{const m=fitCanvas(canvas,Math.min(480,Math.max(340,canvas.clientWidth*0.56)));W=m.w;H=m.h;};
    size(); const onR=()=>size(); window.addEventListener('resize',onR); this._onR=onR;
    const L=this.levels;

    function update(){
      const lv=L[idx];
      scaleEl.textContent=lv.scale; forceEl.textContent=lv.force;
      note.innerHTML='<b>'+lv.name+'.</b> '+lv.blurb;
      zoomIn.disabled=idx>=L.length-1; zoomOut.disabled=idx<=0;
      zoomIn.style.opacity=zoomIn.disabled?0.4:1; zoomOut.style.opacity=zoomOut.disabled?0.4:1;
    }
    function drawScene(ctx,kind,cx,cy,scale){
      ctx.save(); ctx.translate(cx,cy); ctx.scale(scale,scale);
      const C='90,169,255';
      if(kind==='object'){
        ctx.fillStyle='rgba(90,169,255,.18)';ctx.strokeStyle='rgba(90,169,255,.8)';ctx.lineWidth=2;
        ctx.beginPath();ctx.rect(-70,-46,140,92);ctx.fill();ctx.stroke();
        ctx.fillStyle='rgba(255,255,255,.5)';ctx.font='13px Space Mono';ctx.textAlign='center';
        ctx.fillText('solid matter',0,5);
      } else if(kind==='molecule'){
        const pts=[[-46,-18],[-16,8],[16,-12],[44,14],[-30,30],[20,32]];
        ctx.strokeStyle='rgba(90,169,255,.5)';ctx.lineWidth=2.5;
        for(let i=0;i<pts.length-1;i++){ctx.beginPath();ctx.moveTo(pts[i][0],pts[i][1]);ctx.lineTo(pts[i+1][0],pts[i+1][1]);ctx.stroke();}
        pts.forEach((p,i)=>{ctx.beginPath();ctx.arc(p[0],p[1],i%2?9:13,0,7);ctx.fillStyle=i%2?'#9cc2ff':'#5aa9ff';ctx.fill();});
      } else if(kind==='atom'){
        ctx.strokeStyle='rgba(90,169,255,.35)';ctx.lineWidth=1.5;
        for(let k=0;k<3;k++){ctx.beginPath();ctx.ellipse(0,0,40+k*16,18+k*9,k*1.05,0,7);ctx.stroke();}
        ctx.beginPath();ctx.arc(0,0,12,0,7);ctx.fillStyle='#5aa9ff';ctx.fill();
        for(let k=0;k<3;k++){const a=anim*0.04+k*2.1;const rx=40+k*16,ry=18+k*9,rot=k*1.05;
          const ex=Math.cos(a)*rx,ey=Math.sin(a)*ry;
          const x=ex*Math.cos(rot)-ey*Math.sin(rot),y=ex*Math.sin(rot)+ey*Math.cos(rot);
          ctx.beginPath();ctx.arc(x,y,5,0,7);ctx.fillStyle='#cfe2ff';ctx.fill();}
      } else if(kind==='nucleus'){
        const nuc=[[-14,-8,'#ff6f59'],[10,-12,'#9cc2ff'],[-6,12,'#ff6f59'],[16,8,'#9cc2ff'],[2,-2,'#ff6f59'],[-18,6,'#9cc2ff']];
        nuc.forEach(p=>{const j=Math.sin(anim*0.06+p[0])*1.5;
          ctx.beginPath();ctx.arc(p[0]+j,p[1]+j,13,0,7);ctx.fillStyle=p[2];ctx.fill();
          ctx.strokeStyle='rgba(255,255,255,.25)';ctx.lineWidth=1;ctx.stroke();});
      } else if(kind==='quark'){
        ctx.strokeStyle='rgba(90,169,255,.4)';ctx.lineWidth=2;
        ctx.beginPath();ctx.arc(0,0,52,0,7);ctx.setLineDash([5,6]);ctx.stroke();ctx.setLineDash([]);
        const q=[[0,-22,'#ff6f59'],[-20,16,'#4fd38a'],[20,16,'#5aa9ff']];
        q.forEach(p=>{const j=Math.sin(anim*0.08+p[1])*2;
          ctx.beginPath();ctx.arc(p[0]+j,p[1],14,0,7);ctx.fillStyle=p[2];ctx.fill();});
      }
      ctx.restore();
    }
    function frame(){
      anim++;
      const ctx=canvas.getContext('2d');ctx.clearRect(0,0,W,H);
      // transition zoom feel
      target += (idx-target)*0.12;
      const fade=1-Math.min(Math.abs(idx-target)*1.4,1);
      ctx.globalAlpha=clamp(fade,0.2,1);
      drawScene(ctx,L[idx].draw,W/2,H/2, Math.min(W,H)/260 * lerp(0.78,1,fade));
      ctx.globalAlpha=1;
    }
    if(!REDUCED) Loop.start(frame); else { update(); frame(); }
    update();
    zoomIn.onclick=()=>{ if(idx<L.length-1){idx++;update(); if(REDUCED) frame();} };
    zoomOut.onclick=()=>{ if(idx>0){idx--;update(); if(REDUCED) frame();} };
  },
  stop(){ Loop.stop(); if(this._onR) window.removeEventListener('resize',this._onR); }
};

/* =====================================================================
   PIECE III  -  THE RELATION OF PHYSICS TO OTHER SCIENCES
   A small web with physics at the hub. Click any science to read how,
   in Feynman's telling, it ultimately rests on physics.
   ===================================================================== */
const PieceSciences = {
  color:'var(--c3)', roman:'III', eyebrow:'Piece Three',
  title:'The Relation of Physics to Other Sciences',
  lead:'Every science rests on physics. <span class="em">Tap a field</span> to see how.',
  nodes:[
    {name:'Chemistry', tag:'Atoms, arranged',
     text:'Atoms rearranging partners: chemistry is physics for many atoms at once.'},
    {name:'Biology', tag:'Chemistry, alive',
     text:'The same atoms and forces, arranged into chemistry that lives.'},
    {name:'Astronomy', tag:'Physics, enlarged',
     text:'The stars are made of the same atoms we study on Earth.'},
    {name:'Geology', tag:'Forces on a planet',
     text:'Mountains, volcanoes, drifting continents: heat and force on a planet.'},
    {name:'Psychology', tag:'The hardest frontier',
     text:'Even thought may trace down to the physics of the brain. We are far from it.'}
  ],
  render(){
    const stage=pieceHeader(this);
    const panel=el('div',{class:'panel'});
    const wrap=el('div',{class:'sim-wrap'});
    const canvas=el('canvas',{class:'sim-canvas'});
    wrap.appendChild(canvas);
    const side=el('div',{class:'sidepanel'});
    const sTag=el('div',{class:'tag'}),sTitle=el('h3'),sText=el('p');
    const sClose=el('button',{class:'action',text:'Close',style:'margin-top:8px'});
    side.appendChild(sTag);side.appendChild(sTitle);side.appendChild(sText);side.appendChild(sClose);
    wrap.appendChild(side);
    panel.appendChild(wrap);
    stage.appendChild(panel);
    stage.appendChild(el('div',{class:'notebox',html:'Physics sits underneath the rest: they are layers of the same structure.'}));

    let W=0,H=0,t=0,sel=-1,hover=-1;
    const nodes=this.nodes; const pos=nodes.map((_,i)=>({a:i/nodes.length*Math.PI*2,r:0,x:0,y:0}));
    const size=()=>{const m=fitCanvas(canvas,Math.min(500,Math.max(360,canvas.clientWidth*0.6)));W=m.w;H=m.h;};
    size(); const onR=()=>size(); window.addEventListener('resize',onR); this._onR=onR;

    function layout(){
      const cx=W/2,cy=H/2,R=Math.min(W,H)*0.33;
      pos.forEach((p,i)=>{const a=p.a + (REDUCED?0:t*0.0016);p.x=cx+Math.cos(a)*R;p.y=cy+Math.sin(a)*R*0.82;});
      return {cx,cy};
    }
    function frame(){
      t++; const ctx=canvas.getContext('2d');ctx.clearRect(0,0,W,H);
      const {cx,cy}=layout();
      // links
      pos.forEach((p,i)=>{
        const active=(i===sel||i===hover);
        ctx.strokeStyle=active?'rgba(79,211,138,.85)':'rgba(79,211,138,.22)';
        ctx.lineWidth=active?2.4:1.2;
        ctx.beginPath();ctx.moveTo(cx,cy);ctx.lineTo(p.x,p.y);ctx.stroke();
        if(active){ // travelling pulse
          const tt=(t%70)/70;const mx=lerp(cx,p.x,tt),my=lerp(cy,p.y,tt);
          ctx.beginPath();ctx.arc(mx,my,3.5,0,7);ctx.fillStyle='#9bf3c4';ctx.fill();
        }
      });
      // hub
      ctx.beginPath();ctx.arc(cx,cy,30,0,7);
      ctx.fillStyle='#4fd38a';ctx.fill();
      ctx.fillStyle='#06351f';ctx.font='700 13px Space Grotesk';ctx.textAlign='center';ctx.textBaseline='middle';
      ctx.fillText('PHYSICS',cx,cy);
      // nodes
      pos.forEach((p,i)=>{
        const active=(i===sel||i===hover);const rad=active?24:20;
        ctx.beginPath();ctx.arc(p.x,p.y,rad,0,7);
        ctx.fillStyle=active?'#4fd38a':'rgba(79,211,138,.16)';
        ctx.fill();ctx.lineWidth=2;ctx.strokeStyle='#4fd38a';ctx.stroke();
        ctx.fillStyle=active?'#06351f':'#d6f5e4';ctx.font='600 12px Space Grotesk';
        ctx.fillText(nodes[i].name,p.x,p.y+(active?0:0));
      });
    }
    if(!REDUCED) Loop.start(frame); else frame();

    function pick(mx,my){ for(let i=0;i<pos.length;i++){ if(Math.hypot(pos[i].x-mx,pos[i].y-my)<26) return i; } return -1; }
    function evtPos(e){const r=canvas.getBoundingClientRect();const cx=(e.touches?e.touches[0].clientX:e.clientX)-r.left;const cy=(e.touches?e.touches[0].clientY:e.clientY)-r.top;return [cx,cy];}
    canvas.style.cursor='pointer';
    canvas.onmousemove=(e)=>{const[mx,my]=evtPos(e);hover=pick(mx,my);canvas.style.cursor=hover>=0?'pointer':'default';if(REDUCED)frame();};
    function open(i){ if(i<0)return; sel=i; const n=nodes[i];sTag.textContent=n.tag;sTitle.textContent=n.name;sText.textContent=n.text;side.classList.add('show'); if(REDUCED)frame(); }
    canvas.onclick=(e)=>{const[mx,my]=evtPos(e);open(pick(mx,my));};
    sClose.onclick=()=>{side.classList.remove('show');sel=-1;if(REDUCED)frame();};
  },
  stop(){ Loop.stop(); if(this._onR) window.removeEventListener('resize',this._onR); }
};

/* =====================================================================
   PIECE IV  -  CONSERVATION OF ENERGY
   A nonlinear pendulum with live energy bars. Total mechanical energy
   holds flat with no friction; turn friction on and you watch it leak
   into "heat", the modern version of Feynman's blocks under the rug.
   ===================================================================== */
const PieceEnergy = {
  color:'var(--c4)', roman:'IV', eyebrow:'Piece Four',
  title:'Conservation of Energy',
  lead:'Energy changes form but the <span class="em">total never changes</span>. Watch motion and height trade off.',
  render(){
    const stage=pieceHeader(this);
    const panel=el('div',{class:'panel'});
    const wrap=el('div',{class:'sim-wrap'});
    const canvas=el('canvas',{class:'sim-canvas'});
    wrap.appendChild(canvas);panel.appendChild(wrap);

    const roKE=el('span',{class:'v'}),roPE=el('span',{class:'v'}),roTot=el('span',{class:'v'}),roHeat=el('span',{class:'v'});
    panel.appendChild(el('div',{class:'readouts'},[
      el('div',{class:'ro'},[el('span',{class:'k',text:'Kinetic'}),roKE]),
      el('div',{class:'ro'},[el('span',{class:'k',text:'Potential'}),roPE]),
      el('div',{class:'ro'},[el('span',{class:'k',text:'Heat (lost)'}),roHeat]),
      el('div',{class:'ro'},[el('span',{class:'k',text:'Total'}),roTot])
    ]));

    const angVal=el('span',{class:'val'});
    const angSlider=el('input',{type:'range',min:'10',max:'170',value:'120'});
    const fricBtn=el('button',{class:'action',text:'Friction: off'});
    const release=el('button',{class:'action primary',text:'Release'});
    panel.appendChild(el('div',{class:'controls'},[
      el('div',{class:'ctl'},[el('label',[document.createTextNode('Release angle  '),angVal]),angSlider]),
      el('div',{class:'ctl',style:'min-width:auto'},[el('label',{text:'Energy leak'}),el('div',{class:'btnrow'},fricBtn)]),
      el('div',{class:'ctl',style:'min-width:auto'},[el('label',{text:'Drop the bob'}),el('div',{class:'btnrow'},release)])
    ]));
    stage.appendChild(panel);
    stage.appendChild(el('div',{class:'notebox',html:'Friction off: the Total bar holds flat. Friction on: energy leaks into Heat, never lost.'}));

    let W=0,H=0;
    const size=()=>{const m=fitCanvas(canvas,Math.min(460,Math.max(340,canvas.clientWidth*0.5)));W=m.w;H=m.h;};
    size();const onR=()=>size();window.addEventListener('resize',onR);this._onR=onR;

    let theta=Math.PI*120/180, omega=0, friction=false, heat=0;
    const g=0.0006, L=1.0; // arbitrary units
    let E0=0;
    function pivot(){return {x:W/2, y:H*0.16, len:Math.min(W,H)*0.42};}
    function recompute0(){ E0 = g*L*(1-Math.cos(theta)); } // PE at release, omega=0

    function step(dt){
      // nonlinear pendulum
      const a = -g*Math.sin(theta);
      omega += a*dt;
      if(friction){ const before=0.5*L*omega*omega; omega*=0.9985; const after=0.5*L*omega*omega; heat+=(before-after); }
      theta += omega*dt;
    }
    function energies(){
      const ke=0.5*L*omega*omega;
      const pe=g*L*(1-Math.cos(theta));
      return {ke,pe,heat,tot:ke+pe+heat};
    }
    let frameN=0;
    function draw(){
      const ctx=canvas.getContext('2d');ctx.clearRect(0,0,W,H);
      const p=pivot();
      const bx=p.x+Math.sin(theta)*p.len, by=p.y+Math.cos(theta)*p.len;
      // arc guide
      ctx.strokeStyle='rgba(255,255,255,.08)';ctx.lineWidth=1;
      ctx.beginPath();ctx.arc(p.x,p.y,p.len,Math.PI*0.5-1.9,Math.PI*0.5+1.9);ctx.stroke();
      // rod
      ctx.strokeStyle='rgba(255,194,75,.55)';ctx.lineWidth=2.5;
      ctx.beginPath();ctx.moveTo(p.x,p.y);ctx.lineTo(bx,by);ctx.stroke();
      // pivot
      ctx.fillStyle='#8b94ac';ctx.beginPath();ctx.arc(p.x,p.y,5,0,7);ctx.fill();
      // bob, color shifts toward red as it heats up if friction
      ctx.beginPath();ctx.arc(bx,by,17,0,7);
      ctx.fillStyle='#ffc24b';ctx.fill();

      // energy bars on the right
      const E=energies();
      const ref=Math.max(E0,1e-9);
      const bx0=W-150, bw=26, gap=38, bh=H*0.6, by0=H*0.2;
      const items=[['KE',E.ke/ref,'#ff6f59'],['PE',E.pe/ref,'#5aa9ff'],['Heat',E.heat/ref,'#b388ff'],['Total',E.tot/ref,'#ffc24b']];
      items.forEach((it,i)=>{
        const x=bx0+i*gap;
        ctx.fillStyle='rgba(255,255,255,.07)';ctx.fillRect(x,by0,bw,bh);
        const fr=clamp(it[1],0,1.05);
        ctx.fillStyle=it[2];ctx.fillRect(x,by0+bh*(1-fr),bw,bh*fr);
        ctx.fillStyle='#8b94ac';ctx.font='10px Space Mono';ctx.textAlign='center';
        ctx.fillText(it[0],x+bw/2,by0+bh+15);
      });
      // total reference line
      ctx.strokeStyle='rgba(255,194,75,.45)';ctx.setLineDash([4,4]);ctx.lineWidth=1;
      ctx.beginPath();ctx.moveTo(bx0-8,by0);ctx.lineTo(bx0+3*gap+bw+8,by0);ctx.stroke();ctx.setLineDash([]);
      ctx.fillStyle='rgba(255,194,75,.7)';ctx.font='9px Space Mono';ctx.textAlign='right';
      ctx.fillText('release total',bx0-12,by0+3);

      if((frameN++ &7)===0){
        const pct=(x)=>Math.round(x/ref*100);
        roKE.innerHTML=pct(E.ke)+'<span class="u">%</span>';
        roPE.innerHTML=pct(E.pe)+'<span class="u">%</span>';
        roHeat.innerHTML=pct(E.heat)+'<span class="u">%</span>';
        roTot.innerHTML=pct(E.tot)+'<span class="u">%</span>';
      }
    }
    function tick(){ for(let s=0;s<6;s++) step(1.0); draw(); }
    function reset(){ theta=Math.PI*angSlider.value/180; omega=0; heat=0; recompute0(); draw(); }
    recompute0();
    if(!REDUCED) Loop.start(tick); else draw();

    angSlider.oninput=()=>{angVal.textContent=angSlider.value+'\u00B0';angSlider.style.setProperty('--fill',((angSlider.value-10)/160*100)+'%');theta=Math.PI*angSlider.value/180;omega=0;heat=0;recompute0();if(REDUCED)draw();};
    angSlider.oninput();
    fricBtn.onclick=()=>{friction=!friction;fricBtn.textContent='Friction: '+(friction?'on':'off');fricBtn.classList.toggle('on',friction);};
    release.onclick=reset;
    if(REDUCED){ // give reduced-motion users a static "in motion" snapshot
      for(let k=0;k<140;k++) step(1.0); draw();
    }
  },
  stop(){ Loop.stop(); if(this._onR) window.removeEventListener('resize',this._onR); }
};

/* =====================================================================
   PIECE V  -  THE THEORY OF GRAVITATION
   Newtonian two-body orbit. Set the launch speed and watch circles,
   ellipses, and escapes. The trail thins where the planet moves slow
   and bunches where it moves fast: Kepler's equal-area law, visible.
   ===================================================================== */
const PieceGravity = {
  color:'var(--c5)', roman:'V', eyebrow:'Piece Five',
  title:'The Theory of Gravitation',
  lead:'Gravity falls off as <span class="em">distance squared</span>. Set a launch speed: circle, ellipse, or escape.',
  render(){
    const stage=pieceHeader(this);
    const panel=el('div',{class:'panel'});
    const wrap=el('div',{class:'sim-wrap'});
    const canvas=el('canvas',{class:'sim-canvas'});
    wrap.appendChild(canvas);panel.appendChild(wrap);

    const roSpeed=el('span',{class:'v'}),roDist=el('span',{class:'v'}),roType=el('span',{class:'statepill'});
    panel.appendChild(el('div',{class:'readouts'},[
      el('div',{class:'ro'},[el('span',{class:'k',text:'Orbital speed'}),roSpeed]),
      el('div',{class:'ro'},[el('span',{class:'k',text:'Distance'}),roDist]),
      roType
    ]));

    const vVal=el('span',{class:'val'});
    const vSlider=el('input',{type:'range',min:'30',max:'150',value:'82'});
    const launch=el('button',{class:'action primary',text:'Launch'});
    const clearBtn=el('button',{class:'action',text:'Clear'});
    panel.appendChild(el('div',{class:'controls'},[
      el('div',{class:'ctl'},[el('label',[document.createTextNode('Launch speed  '),vVal]),vSlider]),
      el('div',{class:'ctl',style:'min-width:auto'},[el('label',{text:'Set the planet going'}),el('div',{class:'btnrow'},[launch,clearBtn])])
    ]));
    stage.appendChild(panel);
    stage.appendChild(el('div',{class:'notebox',html:'The trail bunches where the planet moves fast and spreads where it moves slow: Kepler\u2019s equal-area law.'}));

    let W=0,H=0;
    const size=()=>{const m=fitCanvas(canvas,Math.min(520,Math.max(380,canvas.clientWidth*0.62)));W=m.w;H=m.h;};
    size();const onR=()=>{const o=W;size();};window.addEventListener('resize',onR);this._onR=onR;

    const GM=46; // gravitational parameter, tuned for the view
    let px,py,vx,vy,trail=[],launched=false;
    function star(){return {x:W/2,y:H/2};}
    function setup(){
      const s=star(); const r0=Math.min(W,H)*0.32;
      px=s.x+r0; py=s.y; const sp=vSlider.value/100 * Math.sqrt(GM/r0);
      vx=0; vy=-sp; trail=[]; launched=true;
    }
    function step(dt){
      const s=star();
      for(let k=0;k<2;k++){
        let dx=s.x-px, dy=s.y-py; let r2=dx*dx+dy*dy; let r=Math.sqrt(r2);
        const soft=Math.max(r,14);
        const a=GM/(soft*soft);
        vx+=a*dx/r*dt; vy+=a*dy/r*dt;
        px+=vx*dt; py+=vy*dt;
      }
      trail.push([px,py]); if(trail.length>900) trail.shift();
    }
    let frameN=0;
    function draw(){
      const ctx=canvas.getContext('2d');ctx.clearRect(0,0,W,H);
      const s=star();
      // trail
      ctx.lineWidth=2;
      for(let i=1;i<trail.length;i++){
        const a=i/trail.length;
        ctx.strokeStyle='rgba(179,136,255,'+(a*0.8)+')';
        ctx.beginPath();ctx.moveTo(trail[i-1][0],trail[i-1][1]);ctx.lineTo(trail[i][0],trail[i][1]);ctx.stroke();
      }
      // star with glow
      ctx.fillStyle='#e8b54a';ctx.beginPath();ctx.arc(s.x,s.y,13,0,7);ctx.fill();
      if(launched){
        ctx.beginPath();ctx.arc(px,py,6,0,7);
        ctx.fillStyle='#b388ff';ctx.fill();
      }
      if((frameN++ &7)===0 && launched){
        const sp=Math.hypot(vx,vy); const dx=s.x-px,dy=s.y-py;const r=Math.hypot(dx,dy);
        roSpeed.innerHTML=(sp*30).toFixed(0)+'<span class="u">arb.</span>';
        roDist.innerHTML=(r/Math.min(W,H)*1000|0)+'<span class="u">arb.</span>';
        const vEsc=Math.sqrt(2*GM/Math.max(r,14));
        let label;
        if(sp>vEsc*0.995) label='escape';
        else if(Math.abs(vSlider.value-100)<6) label='~circular';
        else label='ellipse';
        roType.textContent='Orbit: '+label;
      }
    }
    function tick(){ if(launched) step(1.0); draw(); }
    setup();
    if(!REDUCED) Loop.start(tick); else { for(let k=0;k<400;k++) step(1.0); draw(); }
    vSlider.oninput=()=>{vVal.textContent=(vSlider.value/100).toFixed(2)+'\u00D7';vSlider.style.setProperty('--fill',((vSlider.value-30)/120*100)+'%');};
    vSlider.oninput();
    launch.onclick=()=>{ setup(); if(REDUCED){for(let k=0;k<400;k++)step(1.0);draw();} };
    clearBtn.onclick=()=>{ trail=[]; if(REDUCED)draw(); };
  },
  stop(){ Loop.stop(); if(this._onR) window.removeEventListener('resize',this._onR); }
};

/* =====================================================================
   PIECE VI  -  QUANTUM BEHAVIOR
   The double slit, the heart of the chapter. Fire bullets (two bumps),
   waves (smooth fringes), or electrons (lumps that pile into fringes).
   Switch on the detector and the electron interference collapses.
   ===================================================================== */
const PieceQuantum = {
  color:'var(--c6)', roman:'VI', eyebrow:'Piece Six',
  title:'Quantum Behavior',
  lead:'Two slits, <span class="em">one mystery</span>. Fire bullets, waves, or electrons, then switch on the detector.',
  render(){
    const stage=pieceHeader(this);
    const panel=el('div',{class:'panel'});
    const wrap=el('div',{class:'sim-wrap'});
    const canvas=el('canvas',{class:'sim-canvas'});
    wrap.appendChild(canvas);panel.appendChild(wrap);

    const roCount=el('span',{class:'v'});
    const roMode=el('span',{class:'statepill'});
    panel.appendChild(el('div',{class:'readouts'},[
      el('div',{class:'ro'},[el('span',{class:'k',text:'Detections'}),roCount]),
      roMode
    ]));

    // segmented source control
    let mode='electrons', detector=false;
    const seg=el('div',{class:'seg'});
    const modes=[['bullets','Bullets'],['waves','Waves'],['electrons','Electrons']];
    const segBtns={};
    modes.forEach(m=>{const b=el('button',{text:m[1],'aria-pressed':String(m[0]===mode)});b.onclick=()=>setMode(m[0]);segBtns[m[0]]=b;seg.appendChild(b);});
    const detBtn=el('button',{class:'action',text:'Detector: off'});
    const clearBtn=el('button',{class:'action',text:'Clear'});
    panel.appendChild(el('div',{class:'controls'},[
      el('div',{class:'ctl',style:'min-width:auto'},[el('label',{text:'Fire from the source'}),seg]),
      el('div',{class:'ctl',style:'min-width:auto'},[el('label',{text:'Which-path measurement'}),el('div',{class:'btnrow'},detBtn)]),
      el('div',{class:'ctl',style:'min-width:auto'},[el('label',{text:'Reset'}),el('div',{class:'btnrow'},clearBtn)])
    ]));
    stage.appendChild(panel);
    const note=el('div',{class:'notebox'});
    stage.appendChild(note);

    let W=0,H=0;
    const size=()=>{const m=fitCanvas(canvas,Math.min(500,Math.max(360,canvas.clientWidth*0.6)));W=m.w;H=m.h;};
    size();const onR=()=>size();window.addEventListener('resize',onR);this._onR=onR;

    // geometry
    const slitSep=46, slitHalf=11;
    function geom(){return {sx:W*0.5, screenX:W*0.86, srcX:W*0.1, midY:H/2};}
    // intensity along screen y (relative to mid). Two-slit pattern with envelope.
    function intensity(y){
      const k=0.16, d=slitSep*0.5, a=slitHalf;
      const beta=k*a*y/120;
      const env=Math.abs(beta)<1e-4?1:Math.pow(Math.sin(beta)/beta,2);
      const inter=Math.pow(Math.cos(k*d*y/40),2);
      return env*inter;
    }
    // sample y by rejection against the interference distribution
    function sampleWaveY(){
      const span=H*0.42;
      for(let tries=0;tries<60;tries++){
        const y=(Math.random()*2-1)*span;
        if(Math.random()<intensity(y)) return y;
      }
      return (Math.random()*2-1)*span*0.3;
    }
    function sampleBulletY(){ // two gaussian bumps at the slit projections
      const which=Math.random()<0.5?-1:1;
      let g=0;for(let i=0;i<6;i++)g+=Math.random();g=(g/6-0.5);
      return which*slitSep*1.6 + g*H*0.16;
    }

    const dots=[]; const MAXD=4000;
    function addDot(){
      let y;
      if(mode==='bullets'||(mode==='electrons'&&detector)) y=sampleBulletY();
      else y=sampleWaveY();
      dots.push(y);
      if(dots.length>MAXD) dots.shift();
    }
    // a few flying particles for animation
    const fliers=[];
    function spawnFlier(){
      const g=geom();
      const targetY = (mode==='bullets'||(mode==='electrons'&&detector))?sampleBulletY():sampleWaveY();
      fliers.push({x:g.srcX,y:g.midY,t:0,ty:targetY});
    }

    function drawWaveCurve(ctx,g){
      // analytic intensity curve for waves mode
      ctx.strokeStyle='rgba(41,224,208,.85)';ctx.lineWidth=2;ctx.beginPath();
      const span=H*0.42;
      for(let yy=-span;yy<=span;yy+=2){
        const I=intensity(yy);
        const x=g.screenX+ I*70;
        const y=g.midY+yy;
        if(yy===-span)ctx.moveTo(x,y);else ctx.lineTo(x,y);
      }
      ctx.stroke();
      // fill
      ctx.lineTo(g.screenX,g.midY+span);ctx.lineTo(g.screenX,g.midY-span);ctx.closePath();
      ctx.fillStyle='rgba(41,224,208,.14)';ctx.fill();
    }
    function drawHistogram(ctx,g){
      const span=H*0.42, bins=90; const arr=new Array(bins).fill(0);
      dots.forEach(y=>{const b=Math.floor((y+span)/(2*span)*bins);if(b>=0&&b<bins)arr[b]++;});
      const mx=Math.max(1,...arr);
      ctx.fillStyle='rgba(41,224,208,.16)';
      for(let b=0;b<bins;b++){
        const y=g.midY-span + b*(2*span/bins);
        const w=arr[b]/mx*70;
        ctx.fillRect(g.screenX,y,w,(2*span/bins)+0.6);
      }
      // recent dots as individual lumps
      ctx.fillStyle='#9bf6ee';
      const recent=dots.slice(-700);
      recent.forEach(y=>{ const jitter=(Math.random()*2-1)*6;
        ctx.fillRect(g.screenX+2+Math.random()*66, g.midY+y+jitter*0, 1.6,1.6); });
    }

    let frameN=0, fireAccum=0;
    function draw(){
      const ctx=canvas.getContext('2d');ctx.clearRect(0,0,W,H);
      const g=geom();
      // source
      ctx.fillStyle='#2ee6d6';ctx.beginPath();ctx.arc(g.srcX,g.midY,7,0,7);ctx.fill();
      ctx.fillStyle='rgba(255,255,255,.4)';ctx.font='10px Space Mono';ctx.textAlign='center';
      ctx.fillText('source',g.srcX,g.midY+26);
      // barrier with two slits
      ctx.fillStyle='rgba(255,255,255,.16)';
      const bx=g.sx, top=g.midY-H*0.42, bot=g.midY+H*0.42;
      ctx.fillRect(bx-4,top, 8, (g.midY-slitSep-slitHalf)-top);
      ctx.fillRect(bx-4,g.midY-slitSep+slitHalf, 8, (g.midY+slitSep-slitHalf)-(g.midY-slitSep+slitHalf));
      ctx.fillRect(bx-4,g.midY+slitSep+slitHalf, 8, bot-(g.midY+slitSep+slitHalf));
      // detector glow at slits if on
      if(detector && mode==='electrons'){
        [-slitSep,slitSep].forEach(o=>{ctx.fillStyle='rgba(255,111,89,.5)';ctx.beginPath();ctx.arc(bx,g.midY+o,9,0,7);ctx.fill();});
        ctx.fillStyle='rgba(255,111,89,.8)';ctx.font='10px Space Mono';ctx.fillText('watching',bx,top-6);
      }
      // screen line
      ctx.strokeStyle='rgba(255,255,255,.18)';ctx.lineWidth=2;ctx.beginPath();ctx.moveTo(g.screenX,top);ctx.lineTo(g.screenX,bot);ctx.stroke();

      if(mode==='waves'){
        // animated wavefronts + analytic pattern
        if(!REDUCED){ctx.strokeStyle='rgba(41,224,208,.12)';ctx.lineWidth=1.4;
          for(let r=( (frameN*1.2)%40);r<g.sx-g.srcX;r+=40){ctx.beginPath();ctx.arc(g.srcX,g.midY,r,-1,1);ctx.stroke();}
          [-slitSep,slitSep].forEach(o=>{for(let r=((frameN*1.2)%34);r<(g.screenX-g.sx);r+=34){ctx.beginPath();ctx.arc(g.sx,g.midY+o,r,-1.2,1.2);ctx.stroke();}});
        }
        drawWaveCurve(ctx,g);
      } else {
        drawHistogram(ctx,g);
        // flying particles
        for(let i=fliers.length-1;i>=0;i--){
          const f=fliers[i]; f.t+=0.03;
          if(f.t>=1){ addDot.__y=f.ty; dots.push(f.ty); if(dots.length>MAXD)dots.shift(); fliers.splice(i,1); continue; }
          const t=f.t;
          // path: source -> slit -> screen target
          let x,y;
          if(t<0.5){const u=t/0.5;const slitY=g.midY + (f.ty<0?-slitSep:slitSep)*(detector||mode==='bullets'?1:(Math.random()<0?-1:1));
            x=lerp(g.srcX,g.sx,u);y=lerp(g.midY, g.midY+(f.ty<0?-slitSep:slitSep),u);}
          else {const u=(t-0.5)/0.5;x=lerp(g.sx,g.screenX,u);y=lerp(g.midY+(f.ty<0?-slitSep:slitSep), g.midY+f.ty,u);}
          ctx.fillStyle=mode==='bullets'?'#ffd27a':'#9bf6ee';
          ctx.beginPath();ctx.arc(x,y,mode==='bullets'?4:3,0,7);ctx.fill();
        }
      }

      // fire new particles over time
      if(!REDUCED){
        fireAccum+=1;
        if(mode!=='waves'){ if(fireAccum>=3){fireAccum=0; spawnFlier();} }
      }
      frameN++;
      if((frameN&7)===0){ roCount.innerHTML = (mode==='waves'?'\u2014':dots.length); }
    }
    function tick(){ draw(); }

    function setModeNote(){
      let msg;
      if(mode==='bullets') msg='Each goes through one slit. Two piles, no stripes.';
      else if(mode==='waves') msg='A wave passes through both slits and interferes: stripes.';
      else if(mode==='electrons'&&!detector) msg='Single lumps that pile into stripes. Each goes through both slits.';
      else msg='Measure the path and the stripes collapse to two piles.';
      note.innerHTML=msg;
      const labels={bullets:'classical',waves:'interference',electrons:detector?'collapsed':'quantum'};
      roMode.textContent='Pattern: '+labels[mode];
    }
    function setMode(m){ mode=m; dots.length=0; fliers.length=0;
      Object.keys(segBtns).forEach(k=>segBtns[k].setAttribute('aria-pressed',String(k===m)));
      detBtn.style.opacity = (m==='electrons')?1:0.4;
      setModeNote(); if(REDUCED) staticFill();
    }
    function staticFill(){ // reduced motion: precompute pattern
      dots.length=0; if(mode!=='waves'){ for(let i=0;i<1400;i++) addDot(); } draw();
    }
    setMode('electrons');
    if(!REDUCED) Loop.start(tick); else staticFill();
    detBtn.onclick=()=>{ if(mode!=='electrons')return; detector=!detector; detBtn.textContent='Detector: '+(detector?'on':'off'); detBtn.classList.toggle('on',detector); dots.length=0;fliers.length=0; setModeNote(); if(REDUCED) staticFill(); };
    clearBtn.onclick=()=>{ dots.length=0; fliers.length=0; if(REDUCED) staticFill(); };
  },
  stop(){ Loop.stop(); if(this._onR) window.removeEventListener('resize',this._onR); }
};

/* =====================================================================
   ROUTER
   ===================================================================== */
const PIECES=[PieceAtoms,PieceBasic,PieceSciences,PieceEnergy,PieceGravity,PieceQuantum];
let current=null;

function buildNav(){
  const nav=document.getElementById('nav');
  nav.innerHTML='';
  PIECES.forEach((p,i)=>{
    const a=el('a',{href:'#',text:p.roman+'. '+p.title});
    a.onclick=(e)=>{ e.preventDefault(); go(i); };
    nav.appendChild(a);
    if(i<PIECES.length-1) nav.appendChild(document.createTextNode('   |   '));
  });
}
function go(i){
  if(current && current.stop) current.stop();
  const p=PIECES[i]; current=p; p.render();
  const nav=document.getElementById('nav');
  Array.prototype.forEach.call(nav.querySelectorAll('a'),(a,j)=>{ a.style.fontWeight=(j===i)?'bold':'normal'; });
  window.scrollTo(0,0);
  try{ history.replaceState(null,'','#'+(i+1)); }catch(e){}
}
buildNav();
const startIdx = Math.min(5, Math.max(0, (parseInt((location.hash||'').replace('#',''))||1)-1));
go(startIdx);
</script>
</body>
</html>
