<!DOCTYPE html>
<html>
<head>
<title>Six Easy Pieces</title>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<style>
body { font-family: Arial, Helvetica, sans-serif; margin: 20px; color: #000; background: #fff; }
h1 { font-size: 22px; margin-bottom: 2px; }
h2 { font-size: 17px; margin-top: 26px; margin-bottom: 4px; }
p { margin: 4px 0 8px 0; font-size: 14px; }
canvas { border: 1px solid #000; background: #fff; max-width: 100%; height: auto; }
.controls { margin: 6px 0; font-size: 14px; }
.readout { font-size: 14px; margin: 4px 0; }
#sciInfo { margin-top: 8px; font-size: 14px; min-height: 90px; }
#sciInfo img { width: 80px; height: 80px; border: 1px solid #999; vertical-align: middle; margin-right: 8px; }
.note { color: #555; font-size: 13px; }
input[type=range] { vertical-align: middle; }
</style>
</head>
<body>

<h1>Six Easy Pieces</h1>

<h2>1. Atoms in Motion</h2>
<p>Atoms pull on each other and shift around. Cold makes a solid, warm makes a liquid, hot makes a gas.</p>
<canvas id="atoms" width="400" height="250"></canvas>
<div class="controls">
  Temperature: <input type="range" id="tempSlider" min="0" max="100" value="8" oninput="setTemp()">
  <button onclick="resetAtoms()">Reset</button>
</div>
<div class="readout" id="atomState">State: solid</div>

<h2>2. Basic Physics</h2>
<p>Zoom in to see what matter is made of and what force holds each level together.</p>
<canvas id="basic" width="400" height="250"></canvas>
<div class="controls">
  <button onclick="zoomOut()">Zoom out</button>
  <button onclick="zoomIn()">Zoom in</button>
</div>
<div class="readout" id="basicInfo"></div>

<h2>3. Physics and Other Sciences</h2>
<p>Click a science to see how it connects to physics.</p>
<div class="controls">
  <button onclick="sci('Chemistry')">Chemistry</button>
  <button onclick="sci('Biology')">Biology</button>
  <button onclick="sci('Astronomy')">Astronomy</button>
  <button onclick="sci('Geology')">Geology</button>
  <button onclick="sci('Psychology')">Psychology</button>
</div>
<div id="sciInfo"></div>

<h2>4. Conservation of Energy</h2>
<p>Energy is not lost, it changes form. Turn on friction and motion energy leaks away as heat.</p>
<canvas id="energy" width="400" height="250"></canvas>
<div class="controls">
  Release angle: <input type="range" id="angleSlider" min="10" max="170" value="120" oninput="setAngle()">
  <button onclick="releasePendulum()">Release</button>
  <button onclick="toggleFriction()" id="fricBtn">Friction: off</button>
</div>

<h2>5. The Theory of Gravitation</h2>
<p>The Sun's gravity pulls the planet. The speed decides if you get a circle, an ellipse, or an escape.</p>
<canvas id="gravity" width="400" height="250"></canvas>
<div class="controls">
  Launch speed: <input type="range" id="speedSlider" min="40" max="150" value="100" oninput="setSpeed()">
  <button onclick="launchPlanet()">Launch</button>
  <button onclick="clearTrail()">Clear trail</button>
</div>
<div class="readout" id="orbitInfo">Orbit: circular</div>

<h2>6. Quantum Behavior</h2>
<p>Waves spread out and converge, electrons make stripes, and bullets make two clumps, and watching which slit removes the stripes.</p>
<canvas id="quantum" width="400" height="250"></canvas>
<div class="controls">
  <button onclick="setMode('electron')">Electrons</button>
  <button onclick="setMode('bullet')">Bullets</button>
  <button onclick="setMode('wave')">Waves</button>
  <button onclick="toggleDetector()" id="detBtn">Detector: off</button>
  <button onclick="clearGun()">Clear</button>
</div>
<div class="readout" id="quantumInfo">Mode: electrons</div>

<script>
// ===================== 1. ATOMS (solid / liquid / gas) =====================
var ax = document.getElementById("atoms").getContext("2d");
var N = 64;
var px = [], py = [], vx = [], vy = [];
var D = 28, CUT = D * 1.5, KREP = 0.9, KATT = 0.05, DAMP = 0.985;
var temp = 0.08;
function seedAtoms() {
  px = []; py = []; vx = []; vy = [];
  var cols = 8, rows = 8;
  var startX = (400 - (cols - 1) * D) / 2;
  var startY = (250 - (rows - 1) * D) / 2;
  for (var r = 0; r < rows; r++) {
    for (var c = 0; c < cols; c++) {
      px.push(startX + c * D + (r % 2 ? D / 2 : 0));
      py.push(startY + r * D);
      vx.push(0); vy.push(0);
    }
  }
}
seedAtoms();
function drawAtoms() {
  // forces
  for (var i = 0; i < N; i++) {
    for (var j = i + 1; j < N; j++) {
      var dx = px[j] - px[i], dy = py[j] - py[i];
      var r2 = dx * dx + dy * dy;
      if (r2 > CUT * CUT || r2 < 0.01) continue;
      var r = Math.sqrt(r2), nx = dx / r, ny = dy / r, f;
      if (r < D) { f = -KREP * (D - r); } else { f = KATT * (r - D); }
      vx[i] += f * nx; vy[i] += f * ny;
      vx[j] -= f * nx; vy[j] -= f * ny;
    }
  }
  var kick = temp * temp * 3.2;
  for (var i = 0; i < N; i++) {
    vx[i] += (Math.random() * 2 - 1) * kick;
    vy[i] += (Math.random() * 2 - 1) * kick;
    vx[i] *= DAMP; vy[i] *= DAMP;
    px[i] += vx[i]; py[i] += vy[i];
    if (px[i] < 10) { px[i] = 10; vx[i] = Math.abs(vx[i]); }
    if (px[i] > 390) { px[i] = 390; vx[i] = -Math.abs(vx[i]); }
    if (py[i] < 10) { py[i] = 10; vy[i] = Math.abs(vy[i]); }
    if (py[i] > 240) { py[i] = 240; vy[i] = -Math.abs(vy[i]); }
  }
  // draw
  ax.clearRect(0, 0, 400, 250);
  ax.strokeStyle = "#bbb"; ax.lineWidth = 1;
  for (var i = 0; i < N; i++) {
    for (var j = i + 1; j < N; j++) {
      var dx = px[j] - px[i], dy = py[j] - py[i];
      if (dx * dx + dy * dy < (D * 1.25) * (D * 1.25)) {
        ax.beginPath(); ax.moveTo(px[i], py[i]); ax.lineTo(px[j], py[j]); ax.stroke();
      }
    }
  }
  ax.fillStyle = "blue";
  for (var i = 0; i < N; i++) {
    ax.beginPath(); ax.arc(px[i], py[i], 8, 0, 7); ax.fill();
  }
}
function setTemp() {
  var v = Number(document.getElementById("tempSlider").value);
  temp = Math.pow(v / 100, 1.4) + 0.02;
  var label;
  if (v < 22) { label = "solid"; } else if (v < 55) { label = "liquid"; } else { label = "gas"; }
  document.getElementById("atomState").innerHTML = "State: " + label;
}
function resetAtoms() {
  seedAtoms();
  document.getElementById("tempSlider").value = 8;
  setTemp();
}
setTemp();
setInterval(drawAtoms, 30);

// ===================== 2. BASIC PHYSICS (zoom + scale + force) =====================
var bx = document.getElementById("basic").getContext("2d");
var level = 0;
var levels = [
  { name: "A rock", scale: "about 1 m", force: "electromagnetism" },
  { name: "Molecules", scale: "about 1 nm", force: "electromagnetism" },
  { name: "One atom", scale: "about 0.1 nm", force: "the electric force" },
  { name: "The nucleus", scale: "about 0.00001 nm", force: "the strong force" },
  { name: "Quarks", scale: "even smaller", force: "the strong force" }
];
function drawBasic() {
  bx.clearRect(0, 0, 400, 250);
  bx.fillStyle = "black"; bx.font = "16px Arial"; bx.textAlign = "center";
  bx.fillText(levels[level].name, 200, 28);
  if (level == 0) {
    bx.fillStyle = "gray"; bx.fillRect(150, 90, 100, 80);
  } else if (level == 1) {
    bx.fillStyle = "blue";
    for (var i = 0; i < 12; i++) {
      bx.beginPath(); bx.arc(120 + (i % 4) * 50, 100 + Math.floor(i / 4) * 40, 12, 0, 7); bx.fill();
    }
  } else if (level == 2) {
    bx.fillStyle = "blue"; bx.beginPath(); bx.arc(200, 130, 16, 0, 7); bx.fill();
    bx.strokeStyle = "gray"; bx.lineWidth = 1; bx.beginPath(); bx.arc(200, 130, 60, 0, 7); bx.stroke();
    bx.fillStyle = "red"; bx.beginPath(); bx.arc(260, 130, 6, 0, 7); bx.fill();
  } else if (level == 3) {
    bx.fillStyle = "red";
    bx.beginPath(); bx.arc(185, 120, 15, 0, 7); bx.fill();
    bx.beginPath(); bx.arc(215, 135, 15, 0, 7); bx.fill();
    bx.fillStyle = "blue";
    bx.beginPath(); bx.arc(210, 110, 15, 0, 7); bx.fill();
    bx.beginPath(); bx.arc(185, 145, 15, 0, 7); bx.fill();
  } else if (level == 4) {
    bx.fillStyle = "green";
    bx.beginPath(); bx.arc(200, 110, 11, 0, 7); bx.fill();
    bx.beginPath(); bx.arc(182, 142, 11, 0, 7); bx.fill();
    bx.beginPath(); bx.arc(218, 142, 11, 0, 7); bx.fill();
  }
  document.getElementById("basicInfo").innerHTML =
    "Scale: " + levels[level].scale + ". Held together by: " + levels[level].force + ".";
}
function zoomIn() { level += 1; if (level >= levels.length) { level = levels.length - 1; } drawBasic(); }
function zoomOut() { level -= 1; if (level < 0) { level = 0; } drawBasic(); }
drawBasic();

// ===================== 3. OTHER SCIENCES (text + small image) =====================
var sciSvg = {
  Chemistry: "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 64 64'><rect width='64' height='64' fill='#fff'/><path d='M27 9h10v15l11 24a4 4 0 0 1-3.6 6H19.6A4 4 0 0 1 16 48l11-24z' fill='none' stroke='#000' stroke-width='2'/><path d='M23 39h18l6.5 13.5a2 2 0 0 1-1.8 3H18.3a2 2 0 0 1-1.8-3z' fill='#9cf' /><line x1='25' y1='9' x2='39' y2='9' stroke='#000' stroke-width='2'/></svg>",
  Biology: "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 64 64'><rect width='64' height='64' fill='#fff'/><g stroke='#000' stroke-width='2' fill='none'><path d='M24 8c0 12 16 12 16 24s-16 12-16 24'/><path d='M40 8c0 12-16 12-16 24s16 12 16 24'/></g><g stroke='#888' stroke-width='1.5'><line x1='26' y1='16' x2='38' y2='16'/><line x1='29' y1='24' x2='35' y2='24'/><line x1='29' y1='40' x2='35' y2='40'/><line x1='26' y1='48' x2='38' y2='48'/></g></svg>",
  Astronomy: "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 64 64'><rect width='64' height='64' fill='#fff'/><circle cx='30' cy='34' r='13' fill='#fc6'/><ellipse cx='30' cy='34' rx='22' ry='7' fill='none' stroke='#000' stroke-width='1.5'/><circle cx='50' cy='13' r='1.6' fill='#000'/><circle cx='14' cy='15' r='1.4' fill='#000'/></svg>",
  Geology: "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 64 64'><rect width='64' height='64' fill='#fff'/><path d='M4 50 22 24 33 38 41 28 60 50Z' fill='#999'/><path d='M22 24l5 7-4 6-6-8z' fill='#fff'/></svg>",
  Psychology: "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 64 64'><rect width='64' height='64' fill='#fff'/><path d='M40 14c7 0 11 6 9 12 4 3 3 10-2 12 1 6-5 11-11 9-5 3-12-1-12-7-6-1-8-9-3-13-3-7 4-13 10-11 1-2 4-3 2-3z' fill='none' stroke='#000' stroke-width='2'/></svg>"
};
function sci(name) {
  var text = "";
  if (name == "Chemistry") { text = "Atoms joining up to make new substances."; }
  if (name == "Biology") { text = "Living things are chemistry that is alive."; }
  if (name == "Astronomy") { text = "Stars are made of the same atoms as we are."; }
  if (name == "Geology") { text = "Mountains and volcanoes are heat and force on a planet."; }
  if (name == "Psychology") { text = "Even thinking may come down to physics in the brain."; }
  var img = "data:image/svg+xml;utf8," + encodeURIComponent(sciSvg[name]);
  document.getElementById("sciInfo").innerHTML =
    "<img src='" + img + "' alt=''><b>" + name + ":</b> " + text;
}

// ===================== 4. ENERGY (KE / PE / heat / total, friction) =====================
var ex = document.getElementById("energy").getContext("2d");
var angle = 120 * Math.PI / 180, vel = 0, k = 0.01, friction = false, heat = 0;
var E0 = k * (1 - Math.cos(angle));
function drawEnergy() {
  ex.clearRect(0, 0, 400, 250);
  vel -= k * Math.sin(angle);
  if (friction) {
    var before = 0.5 * vel * vel;
    vel *= 0.999;
    var after = 0.5 * vel * vel;
    heat += (before - after);
  }
  angle += vel;
  var pivotX = 200, pivotY = 20, len = 150;
  var bxx = pivotX + Math.sin(angle) * len;
  var byy = pivotY + Math.cos(angle) * len;
  ex.strokeStyle = "black"; ex.lineWidth = 1;
  ex.beginPath(); ex.moveTo(pivotX, pivotY); ex.lineTo(bxx, byy); ex.stroke();
  ex.fillStyle = "red"; ex.beginPath(); ex.arc(bxx, byy, 14, 0, 7); ex.fill();
  var ke = 0.5 * vel * vel;
  var pe = k * (1 - Math.cos(angle));
  var tot = ke + pe + heat;
  var bars = [[ke, "green", "KE"], [pe, "blue", "PE"], [heat, "gray", "Ht"], [tot, "orange", "Tot"]];
  for (var i = 0; i < bars.length; i++) {
    var h = bars[i][0] / E0 * 200;
    if (h > 210) { h = 210; }
    var x = 30 + i * 35;
    ex.fillStyle = bars[i][1];
    ex.fillRect(x, 240 - h, 25, h);
    ex.fillStyle = "black"; ex.font = "12px Arial"; ex.textAlign = "center";
    ex.fillText(bars[i][2], x + 12, 248);
  }
}
function setAngle() {
  angle = Number(document.getElementById("angleSlider").value) * Math.PI / 180;
  vel = 0; heat = 0; E0 = k * (1 - Math.cos(angle));
}
function releasePendulum() { setAngle(); }
function toggleFriction() {
  friction = !friction;
  document.getElementById("fricBtn").innerHTML = "Friction: " + (friction ? "on" : "off");
}
setInterval(drawEnergy, 30);

// ===================== 5. GRAVITY (launch speed, orbit type) =====================
var gx = document.getElementById("gravity").getContext("2d");
var sunX = 200, sunY = 125, GM = 120;
var startX = 310, startR = startX - sunX;
var planet = { x: startX, y: 125, dx: 0, dy: 0 };
var trail = [];
function launchPlanet() {
  var factor = Number(document.getElementById("speedSlider").value) / 100;
  var vcirc = Math.sqrt(GM / startR);
  planet = { x: startX, y: 125, dx: 0, dy: -factor * vcirc };
  trail = [];
}
function setSpeed() {
  document.getElementById("orbitInfo");
}
function clearTrail() { trail = []; }
function drawGravity() {
  gx.clearRect(0, 0, 400, 250);
  var dx = sunX - planet.x, dy = sunY - planet.y;
  var dist = Math.sqrt(dx * dx + dy * dy);
  if (dist < 12) { dist = 12; }
  var force = GM / (dist * dist);
  planet.dx += force * dx / dist;
  planet.dy += force * dy / dist;
  planet.x += planet.dx;
  planet.y += planet.dy;
  trail.push({ x: planet.x, y: planet.y });
  if (trail.length > 50) { trail.shift(); }
  gx.fillStyle = "gray";
  for (var i = 0; i < trail.length; i++) { gx.fillRect(trail[i].x, trail[i].y, 2, 2); }
  gx.fillStyle = "orange"; gx.beginPath(); gx.arc(sunX, sunY, 16, 0, 7); gx.fill();
  gx.fillStyle = "blue"; gx.beginPath(); gx.arc(planet.x, planet.y, 8, 0, 7); gx.fill();
  // orbit type
  var speed = Math.sqrt(planet.dx * planet.dx + planet.dy * planet.dy);
  var vEsc = Math.sqrt(2 * GM / dist);
  var factor = Number(document.getElementById("speedSlider").value) / 100;
  var label;
  if (speed > vEsc * 0.99) { label = "escape"; }
  else if (Math.abs(factor - 1) < 0.06) { label = "circular"; }
  else { label = "ellipse"; }
  document.getElementById("orbitInfo").innerHTML = "Orbit: " + label;
}
launchPlanet();
setInterval(drawGravity, 30);

// ===================== 6. QUANTUM (full model, plain colors) =====================
var qx = document.getElementById("quantum").getContext("2d");
var mode = "electron";
var detector = false;
var dots = [];     // landed: {x,y}
var fliers = [];   // travelling particles
var qt = 0, fireAccum = 0;
var SRCX = 30, BARX = 150, SCREENX = 290, MIDY = 125;
var SEP = 30, SLITH = 12, SPAN = 95;

function lerpQ(a, b, t) { return a + (b - a) * t; }
function intensity(yy) {
  var k = 0.16, d = SEP, a = SLITH;
  var beta = k * a * yy / 120;
  var env = Math.abs(beta) < 0.0001 ? 1 : Math.pow(Math.sin(beta) / beta, 2);
  var inter = Math.pow(Math.cos(k * d * yy / 40), 2);
  return env * inter;
}
function sampleWaveY() {
  for (var t = 0; t < 60; t++) {
    var yy = (Math.random() * 2 - 1) * SPAN;
    if (Math.random() < intensity(yy)) { return MIDY + yy; }
  }
  return MIDY;
}
function sampleBulletY() {
  var side = Math.random() < 0.5 ? -1 : 1;
  var g = 0; for (var i = 0; i < 6; i++) { g += Math.random(); } g = (g / 6 - 0.5);
  return MIDY + side * SEP + g * 40;
}
function targetY() {
  if (mode == "bullet" || (mode == "electron" && detector)) { return sampleBulletY(); }
  return sampleWaveY();
}
function spawnFlier() { fliers.push({ t: 0, ty: targetY(), tx: SCREENX + Math.random() * 60 }); }

function updateQuantum() {
  qt++;
  if (mode != "wave") {
    fireAccum++;
    if (fireAccum >= 3) { fireAccum = 0; spawnFlier(); }
    for (var i = fliers.length - 1; i >= 0; i--) {
      var f = fliers[i];
      f.t += 0.03;
      if (f.t >= 1) {
        dots.push({ x: f.tx, y: f.ty });
        if (dots.length > 2500) { dots.shift(); }
        fliers.splice(i, 1);
      }
    }
  }
  drawQuantum();
}

function drawQuantum() {
  qx.clearRect(0, 0, 400, 250);
  // source
  qx.fillStyle = "red"; qx.beginPath(); qx.arc(SRCX, MIDY, 7, 0, 7); qx.fill();
  // barrier with two slits
  qx.fillStyle = "black";
  qx.fillRect(BARX, 0, 8, MIDY - SEP - SLITH);
  qx.fillRect(BARX, MIDY - SEP + SLITH, 8, (MIDY + SEP - SLITH) - (MIDY - SEP + SLITH));
  qx.fillRect(BARX, MIDY + SEP + SLITH, 8, 250 - (MIDY + SEP + SLITH));
  // detector marks
  if (detector && mode == "electron") {
    qx.fillStyle = "red";
    qx.beginPath(); qx.arc(BARX + 4, MIDY - SEP, 5, 0, 7); qx.fill();
    qx.beginPath(); qx.arc(BARX + 4, MIDY + SEP, 5, 0, 7); qx.fill();
  }
  // screen line
  qx.strokeStyle = "#999"; qx.lineWidth = 1;
  qx.beginPath(); qx.moveTo(SCREENX, MIDY - SPAN); qx.lineTo(SCREENX, MIDY + SPAN); qx.stroke();

  if (mode == "wave") {
    // moving wavefronts: from the source, then from each slit
    qx.strokeStyle = "#ccc"; qx.lineWidth = 1;
    var p1 = (qt * 1.2) % 40;
    for (var r = p1; r < BARX - SRCX; r += 40) { qx.beginPath(); qx.arc(SRCX, MIDY, r, -1, 1); qx.stroke(); }
    var p2 = (qt * 1.2) % 34;
    var slits = [-SEP, SEP];
    for (var sIdx = 0; sIdx < slits.length; sIdx++) {
      for (var r2 = p2; r2 < SCREENX - BARX; r2 += 34) {
        qx.beginPath(); qx.arc(BARX, MIDY + slits[sIdx], r2, -1.2, 1.2); qx.stroke();
      }
    }
    // analytic intensity curve
    qx.strokeStyle = "black"; qx.lineWidth = 2; qx.beginPath();
    for (var yy = -SPAN; yy <= SPAN; yy += 2) {
      var cx = SCREENX + intensity(yy) * 66;
      if (yy == -SPAN) { qx.moveTo(cx, MIDY + yy); } else { qx.lineTo(cx, MIDY + yy); }
    }
    qx.stroke();
  } else {
    // histogram of landed dots
    var bins = 90, arr = [];
    for (var b = 0; b < bins; b++) { arr[b] = 0; }
    for (var i = 0; i < dots.length; i++) {
      var bb = Math.floor((dots[i].y - (MIDY - SPAN)) / (2 * SPAN) * bins);
      if (bb >= 0 && bb < bins) { arr[bb]++; }
    }
    var mx = 1; for (var b = 0; b < bins; b++) { if (arr[b] > mx) { mx = arr[b]; } }
    qx.fillStyle = "#ddd";
    for (var b = 0; b < bins; b++) {
      var yb = MIDY - SPAN + b * (2 * SPAN / bins);
      qx.fillRect(SCREENX, yb, arr[b] / mx * 66, (2 * SPAN / bins) + 0.6);
    }
    // landed dots
    qx.fillStyle = "black";
    for (var i = 0; i < dots.length; i++) { qx.fillRect(dots[i].x, dots[i].y, 2, 2); }
    // travelling particles
    for (var i = 0; i < fliers.length; i++) {
      var f = fliers[i];
      var slitY = MIDY + (f.ty < MIDY ? -SEP : SEP), x, y, u;
      if (f.t < 0.5) { u = f.t / 0.5; x = lerpQ(SRCX, BARX, u); y = lerpQ(MIDY, slitY, u); }
      else { u = (f.t - 0.5) / 0.5; x = lerpQ(BARX, f.tx, u); y = lerpQ(slitY, f.ty, u); }
      qx.fillStyle = (mode == "bullet") ? "gray" : "black";
      qx.beginPath(); qx.arc(x, y, 3, 0, 7); qx.fill();
    }
  }
  // readout
  var names = { electron: "electrons", bullet: "bullets", wave: "waves" };
  var info = "Mode: " + names[mode];
  if (mode != "wave") { info += ". Hits: " + dots.length; }
  if (mode == "electron") { info += ". Detector: " + (detector ? "on" : "off"); }
  document.getElementById("quantumInfo").innerHTML = info;
}

function setMode(m) { mode = m; dots = []; fliers = []; drawQuantum(); }
function toggleDetector() {
  detector = !detector;
  document.getElementById("detBtn").innerHTML = "Detector: " + (detector ? "on" : "off");
  dots = []; fliers = []; drawQuantum();
}
function clearGun() { dots = []; fliers = []; drawQuantum(); }
setInterval(updateQuantum, 30);
drawQuantum();
</script>

</body>
</html>
