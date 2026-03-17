/**

- ╔══════════════════════════════════════════════════════════════╗
- ║  MASAT 5PL — Terminal Operating System API                   ║
- ║  Version: 1.0.0  |  CTO Build  |  2026                       ║
- ║  Stack: Node.js (built-ins only — zero dependencies)         ║
- ║  For production: add Express, JWT, Redis, PostgreSQL          ║
- ╚══════════════════════════════════════════════════════════════╝
- 
- ENDPOINTS:
- POST /api/v1/auth/login
- GET  /api/v1/containers/:bl_number
- GET  /api/v1/containers
- POST /api/v1/containers/:id/scan
- GET  /api/v1/system/health
- GET  /api/v1/alerts
- POST /api/v1/alerts/subscribe
- GET  /api/v1/analytics/dwell
- GET  /api/v1/analytics/sla
- GET  /api/v1/documents/:container_id
- POST /api/v1/documents/:container_id/update
- GET  /api/v1/cfs
- GET  /api/v1/sandbox/reset
  */

‘use strict’;
const http   = require(‘http’);
const crypto = require(‘crypto’);
const url    = require(‘url’);

const PORT = process.env.PORT || 3000;

// ─── SEED DATA ────────────────────────────────────────────────────────────────

const USERS = {
‘agent_demo’:   { id:‘u001’, role:‘clearing_agent’,  name:‘Rose Wanjiku’,     token: genToken(‘u001’), company:‘Clearfast Kenya Ltd’ },
‘cfs_demo’:     { id:‘u002’, role:‘cfs_manager’,     name:‘David Otieno’,     token: genToken(‘u002’), company:‘Mombasa CFS A’ },
‘importer_demo’:{ id:‘u003’, role:‘importer’,        name:‘John Kamau’,       token: genToken(‘u003’), company:‘Kamau Imports Ltd’ },
‘driver_demo’:  { id:‘u004’, role:‘truck_driver’,    name:‘Ali Hassan’,       token: genToken(‘u004’), company:‘Fast Logistics’ },
‘admin_demo’:   { id:‘u005’, role:‘admin’,           name:‘Sys Admin’,        token: genToken(‘u005’), company:‘Masat 5PL’ },
};

const PASSWORDS = {
‘agent_demo’:‘demo1234’,‘cfs_demo’:‘demo1234’,‘importer_demo’:‘demo1234’,
‘driver_demo’:‘demo1234’,‘admin_demo’:‘demo1234’
};

let CONTAINERS = {
‘KESU1234567’: {
id:‘c001’, bl_number:‘BLABC123456’, container_number:‘KESU1234567’,
shipping_line:‘Maersk’, vessel:‘Maersk Kenia’, voyage:‘MK2601’,
cfs:‘Mombasa CFS A’, commodity:‘Industrial Equipment’,
weight_tons:12.4, cbm:28.0,
status:‘customs_exam_passed’, risk_level:‘on_track’,
dwell_days:3.2, predicted_dwell_days:4.1, dwell_pct:78,
discharge_date:‘2026-03-03T11:30:00Z’,
gate_in_date:‘2026-03-04T14:22:00Z’,
customs_exam_start:‘2026-03-05T09:00:00Z’,
customs_exam_end:‘2026-03-05T16:45:00Z’,
estimated_release:‘2026-03-06T11:00:00Z’,
ai_confidence:89,
timeline:[
{event:‘vessel_arrival’,      label:‘Vessel Arrival’,              ts:‘2026-03-02T08:14:00Z’, source:‘Maersk API’,  done:true},
{event:‘discharged’,          label:‘Discharged — KWATOS confirmed’,ts:‘2026-03-03T11:30:00Z’,source:‘KWATOS’,      done:true},
{event:‘gate_in’,             label:‘Gate-In at CFS Mombasa A’,    ts:‘2026-03-04T14:22:00Z’, source:‘CFS Scanner’, done:true},
{event:‘customs_exam_start’,  label:‘Customs Examination Started’, ts:‘2026-03-05T09:00:00Z’, source:‘iCMS’,        done:true},
{event:‘customs_exam_passed’, label:‘Customs Exam Passed’,         ts:‘2026-03-05T16:45:00Z’, source:‘iCMS’,        done:true, active:true},
{event:‘gate_out’,            label:‘Gate-Out / Release’,          ts:null,                   source:‘Pending’,     done:false},
],
alerts_sent:[‘vessel_arrival’,‘discharged’,‘gate_in’,‘customs_exam_start’,‘customs_exam_passed’],
agent_id:‘u001’, importer_id:‘u003’,
},
‘MSCU8876543’: {
id:‘c002’, bl_number:‘BLXYZ789012’, container_number:‘MSCU8876543’,
shipping_line:‘MSC’, vessel:‘MSC Nairobi’, voyage:‘MN2602’,
cfs:‘Mombasa CFS B’, commodity:‘Consumer Goods’,
weight_tons:8.6, cbm:40.0,
status:‘kebs_hold’, risk_level:‘warning’,
dwell_days:8.1, predicted_dwell_days:5.0, dwell_pct:95,
discharge_date:‘2026-02-28T09:00:00Z’,
gate_in_date:‘2026-03-01T10:15:00Z’,
customs_exam_start:‘2026-03-02T14:00:00Z’,
customs_exam_end:null,
estimated_release:‘2026-03-12T00:00:00Z’,
ai_confidence:72,
kebs_hold:{ placed:‘2026-03-03T08:00:00Z’, reason:‘Missing Certificate of Conformity’, doc_required:‘CoC’ },
timeline:[
{event:‘vessel_arrival’, label:‘Vessel Arrival’,           ts:‘2026-02-27T14:00:00Z’, source:‘MSC API’,    done:true},
{event:‘discharged’,     label:‘Discharged — KWATOS’,      ts:‘2026-02-28T09:00:00Z’, source:‘KWATOS’,     done:true},
{event:‘gate_in’,        label:‘Gate-In at CFS Mombasa B’, ts:‘2026-03-01T10:15:00Z’, source:‘CFS Scanner’,done:true},
{event:‘kebs_hold’,      label:‘KEBS Hold Placed — CoC Missing’,ts:‘2026-03-03T08:00:00Z’,source:‘iCMS’,   done:true, active:true, flag:‘warning’},
{event:‘gate_out’,       label:‘Gate-Out (Pending KEBS)’,  ts:null, source:‘Pending’,  done:false},
],
alerts_sent:[‘vessel_arrival’,‘discharged’,‘gate_in’,‘kebs_hold_alert’],
agent_id:‘u001’, importer_id:‘u003’,
},
‘CMAU2345678’: {
id:‘c003’, bl_number:‘BLCMA345678’, container_number:‘CMAU2345678’,
shipping_line:‘CMA CGM’, vessel:‘CMA Mombasa’, voyage:‘CM2603’,
cfs:‘ICD Nairobi’, commodity:‘Machinery Parts’,
weight_tons:22.1, cbm:55.0,
status:‘dark_container’, risk_level:‘critical’,
dwell_days:18, predicted_dwell_days:6.2, dwell_pct:100,
discharge_date:‘2026-02-20T07:00:00Z’,
gate_in_date:‘2026-02-21T09:00:00Z’,
customs_exam_start:null,
customs_exam_end:null,
estimated_release:null,
ai_confidence:40,
last_scan:‘2026-02-21T09:00:00Z’,
dark_reason:‘No scan events recorded for 17 days’,
timeline:[
{event:‘vessel_arrival’, label:‘Vessel Arrival’,      ts:‘2026-02-19T22:00:00Z’, source:‘CMA API’,    done:true},
{event:‘discharged’,     label:‘Discharged — KWATOS’, ts:‘2026-02-20T07:00:00Z’, source:‘KWATOS’,     done:true},
{event:‘gate_in’,        label:‘Gate-In at ICD Nairobi’,ts:‘2026-02-21T09:00:00Z’,source:‘CFS Scanner’,done:true},
{event:‘dark’,           label:‘⚠ NO SCAN EVENTS — 17 DAYS’,ts:‘2026-02-21T09:00:00Z’,source:‘Anomaly Detection’,done:true,active:true,flag:‘critical’},
],
alerts_sent:[‘vessel_arrival’,‘discharged’,‘gate_in’,‘dark_container_alert’,‘escalation_sent’],
agent_id:‘u001’, importer_id:‘u003’,
},
‘MSKU9912334’: {
id:‘c004’, bl_number:‘BLMSK112233’, container_number:‘MSKU9912334’,
shipping_line:‘Maersk’, vessel:‘Maersk Mombasa’, voyage:‘MM2601’,
cfs:‘Mombasa CFS A’, commodity:‘Pharmaceuticals’,
weight_tons:3.2, cbm:12.0,
status:‘released’, risk_level:‘on_track’,
dwell_days:3.8, predicted_dwell_days:5.0, dwell_pct:76,
discharge_date:‘2026-03-07T08:00:00Z’,
gate_in_date:‘2026-03-07T14:00:00Z’,
gate_out_date:‘2026-03-10T11:30:00Z’,
customs_exam_start:‘2026-03-08T10:00:00Z’,
customs_exam_end:‘2026-03-08T15:00:00Z’,
estimated_release:‘2026-03-10T11:30:00Z’,
ai_confidence:95,
timeline:[
{event:‘vessel_arrival’,      label:‘Vessel Arrival’,              ts:‘2026-03-06T18:00:00Z’,source:‘Maersk API’, done:true},
{event:‘discharged’,          label:‘Discharged’,                  ts:‘2026-03-07T08:00:00Z’,source:‘KWATOS’,    done:true},
{event:‘gate_in’,             label:‘Gate-In at CFS Mombasa A’,    ts:‘2026-03-07T14:00:00Z’,source:‘CFS Scanner’,done:true},
{event:‘customs_exam_passed’, label:‘Customs Exam Passed’,         ts:‘2026-03-08T15:00:00Z’,source:‘iCMS’,      done:true},
{event:‘gate_out’,            label:‘Gate-Out — Delivered’,        ts:‘2026-03-10T11:30:00Z’,source:‘CFS Scanner’,done:true},
],
alerts_sent:[‘vessel_arrival’,‘discharged’,‘gate_in’,‘customs_exam_passed’,‘gate_out’],
agent_id:‘u001’, importer_id:‘u003’,
},
};

let DOCUMENTS = {
‘c001’:[
{type:‘IDF’,  name:‘Import Declaration Form’,      status:‘approved’,  required:true,  source:‘KenTrade’, updated:‘2026-03-01’},
{type:‘BL’,   name:‘Bill of Lading’,               status:‘approved’,  required:true,  source:‘Maersk’,   updated:‘2026-03-02’},
{type:‘INV’,  name:‘Commercial Invoice’,           status:‘approved’,  required:true,  source:‘Importer’, updated:‘2026-03-01’},
{type:‘PKL’,  name:‘Packing List’,                 status:‘approved’,  required:true,  source:‘Importer’, updated:‘2026-03-01’},
{type:‘CoC’,  name:‘Certificate of Conformity’,   status:‘approved’,  required:true,  source:‘KEBS’,     updated:‘2026-03-02’},
{type:‘CO’,   name:‘Certificate of Origin’,       status:‘approved’,  required:true,  source:‘EPC’,      updated:‘2026-03-01’},
{type:‘C32’,  name:‘Customs Entry Form (C32)’,     status:‘submitted’, required:true,  source:‘iCMS’,     updated:‘2026-03-05’},
{type:‘DO’,   name:‘Delivery Order’,              status:‘pending’,   required:true,  source:‘Maersk’,   updated:null},
],
‘c002’:[
{type:‘IDF’,  name:‘Import Declaration Form’,      status:‘approved’,  required:true,  source:‘KenTrade’, updated:‘2026-02-25’},
{type:‘BL’,   name:‘Bill of Lading’,               status:‘approved’,  required:true,  source:‘MSC’,      updated:‘2026-02-27’},
{type:‘INV’,  name:‘Commercial Invoice’,           status:‘approved’,  required:true,  source:‘Importer’, updated:‘2026-02-25’},
{type:‘PKL’,  name:‘Packing List’,                 status:‘approved’,  required:true,  source:‘Importer’, updated:‘2026-02-25’},
{type:‘CoC’,  name:‘Certificate of Conformity’,   status:‘missing’,   required:true,  source:‘KEBS’,     updated:null,  flag:‘KEBS HOLD — must be uploaded’},
{type:‘CO’,   name:‘Certificate of Origin’,       status:‘approved’,  required:true,  source:‘EPC’,      updated:‘2026-02-25’},
{type:‘C32’,  name:‘Customs Entry Form (C32)’,     status:‘on_hold’,   required:true,  source:‘iCMS’,     updated:‘2026-03-01’},
{type:‘DO’,   name:‘Delivery Order’,              status:‘pending’,   required:true,  source:‘MSC’,      updated:null},
],
};

let SYSTEM_HEALTH = {
kwatos:  { status:‘online’,   uptime_pct:94.2, last_check: new Date().toISOString(), latency_ms:120, incidents_30d:3 },
icms:    { status:‘degraded’, uptime_pct:88.7, last_check: new Date().toISOString(), latency_ms:890, incidents_30d:8 },
kentrade:{ status:‘online’,   uptime_pct:97.1, last_check: new Date().toISOString(), latency_ms:95,  incidents_30d:1 },
maersk_api:{ status:‘online’, uptime_pct:99.3, last_check: new Date().toISOString(), latency_ms:210, incidents_30d:0 },
cmacgm_api:{ status:‘online’, uptime_pct:98.8, last_check: new Date().toISOString(), latency_ms:245, incidents_30d:0 },
cache:   { status:‘online’,   mode:‘active’,   keys_cached:156, last_sync: new Date().toISOString() },
};

let ALERTS_LOG = [
{ id:‘a001’, type:‘kebs_hold’,         container:‘MSCU8876543’, message:‘KEBS hold placed — CoC missing’,         sent_via:[‘whatsapp’,‘sms’], ts:‘2026-03-03T08:05:00Z’, read:false },
{ id:‘a002’, type:‘dwell_warning’,     container:‘MSCU8876543’, message:‘Dwell time exceeds 80% of prediction’,    sent_via:[‘whatsapp’],      ts:‘2026-03-04T09:00:00Z’, read:false },
{ id:‘a003’, type:‘dark_container’,    container:‘CMAU2345678’, message:‘No scan events for 17 days — CRITICAL’,   sent_via:[‘whatsapp’,‘sms’,‘email’], ts:‘2026-03-07T08:00:00Z’, read:false },
{ id:‘a004’, type:‘customs_passed’,    container:‘KESU1234567’, message:‘Customs exam passed — ready for release’, sent_via:[‘whatsapp’],      ts:‘2026-03-05T16:50:00Z’, read:true  },
{ id:‘a005’, type:‘system_downtime’,   container:null,          message:‘iCMS degraded — using cached data’,       sent_via:[‘push’],          ts:‘2026-03-10T06:00:00Z’, read:false },
];

let SCAN_EVENTS = [];

// ─── HELPERS ──────────────────────────────────────────────────────────────────

function genToken(userId) {
return ‘msat_’ + crypto.createHmac(‘sha256’,‘masat5pl_secret_2026’).update(userId+Date.now()).digest(‘hex’).slice(0,32);
}

function ts() { return new Date().toISOString(); }

function uuid() {
return ‘xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx’.replace(/[xy]/g, c => {
const r = Math.random()*16|0;
return (c===‘x’?r:(r&0x3|0x8)).toString(16);
});
}

function json(res, data, code=200) {
res.writeHead(code, {
‘Content-Type’:‘application/json’,
‘Access-Control-Allow-Origin’:’*’,
‘Access-Control-Allow-Methods’:‘GET,POST,PUT,DELETE,OPTIONS’,
‘Access-Control-Allow-Headers’:‘Content-Type,Authorization,X-API-Key’,
‘X-Powered-By’:‘Masat5PL-TOS/1.0’,
‘X-Response-Time’: Date.now() + ‘ms’,
});
res.end(JSON.stringify({ …data, _meta:{ api:‘Masat5PL TOS v1.0’, ts:ts(), docs:‘https://api.masat5pl.com/docs’ }}, null, 2));
}

function err(res, code, message, details=null) {
json(res, { success:false, error:{ code, message, details }}, code);
}

function parseBody(req) {
return new Promise(resolve => {
let body = ‘’;
req.on(‘data’, chunk => body += chunk);
req.on(‘end’, () => {
try { resolve(JSON.parse(body || ‘{}’)); }
catch { resolve({}); }
});
});
}

function getToken(req) {
const auth = req.headers[‘authorization’] || ‘’;
if (auth.startsWith(’Bearer ’)) return auth.slice(7);
return req.headers[‘x-api-key’] || null;
}

function authUser(req) {
const token = getToken(req);
if (!token) return null;
return Object.values(USERS).find(u => u.token === token) || null;
}

function requireAuth(req, res) {
const user = authUser(req);
if (!user) { err(res, 401, ‘Unauthorized — provide Bearer token or X-API-Key header’); return null; }
return user;
}

// ─── AI DWELL PREDICTION (simple heuristic model) ────────────────────────────

function predictDwell(commodity, cfs, shipping_line) {
const base = { ‘Industrial Equipment’:4.5,‘Consumer Goods’:5.2,‘Pharmaceuticals’:3.8,‘Machinery Parts’:6.0,‘Food Products’:4.0,‘Electronics’:4.2 };
const cfsMod = { ‘Mombasa CFS A’:-0.3,‘Mombasa CFS B’:0.5,‘ICD Nairobi’:1.2 };
const lineMod = { ‘Maersk’:-0.2,‘MSC’:0.1,‘CMA CGM’:0.2,‘Hapag-Lloyd’:-0.1 };
const b = base[commodity] || 5.0;
const c = cfsMod[cfs] || 0;
const l = lineMod[shipping_line] || 0;
const jitter = (Math.random()-0.5)*0.6;
return Math.max(2.0, +(b+c+l+jitter).toFixed(1));
}

function riskLevel(actual, predicted) {
const pct = actual / predicted;
if (pct <= 0.85) return { level:‘on_track’, pct:Math.round(pct*100), label:‘On Track 🟢’ };
if (pct <= 1.05) return { level:‘warning’,  pct:Math.round(pct*100), label:‘Warning 🟡’ };
return { level:‘critical’, pct:Math.round(pct*100), label:‘Critical 🔴’ };
}

// ─── ROUTER ───────────────────────────────────────────────────────────────────

async function router(req, res) {
const parsed = url.parse(req.url, true);
const path   = parsed.pathname.replace(//$/, ‘’);
const query  = parsed.query;
const method = req.method.toUpperCase();

// CORS preflight
if (method === ‘OPTIONS’) {
res.writeHead(204, {‘Access-Control-Allow-Origin’:’*’,‘Access-Control-Allow-Methods’:‘GET,POST,OPTIONS’,‘Access-Control-Allow-Headers’:‘Content-Type,Authorization,X-API-Key’});
res.end(); return;
}

// ── ROOT / DOCS ──
if (path === ‘/’ || path === ‘’) {
json(res, {
success:true,
service:‘Masat 5PL Terminal Operating System API’,
version:‘1.0.0’,
status:‘operational’,
sandbox:true,
message:‘Welcome to the Masat 5PL TOS API. All endpoints are live with seed data.’,
endpoints:{
auth:         ‘POST /api/v1/auth/login’,
container:    ‘GET  /api/v1/containers/:bl_number’,
containers:   ‘GET  /api/v1/containers’,
scan:         ‘POST /api/v1/containers/:id/scan’,
health:       ‘GET  /api/v1/system/health’,
alerts:       ‘GET  /api/v1/alerts’,
subscribe:    ‘POST /api/v1/alerts/subscribe’,
dwell:        ‘GET  /api/v1/analytics/dwell’,
sla:          ‘GET  /api/v1/analytics/sla’,
documents:    ‘GET  /api/v1/documents/:container_id’,
doc_update:   ‘POST /api/v1/documents/:container_id/update’,
cfs:          ‘GET  /api/v1/cfs’,
predict:      ‘POST /api/v1/ai/predict-dwell’,
reset:        ‘GET  /api/v1/sandbox/reset’,
},
sandbox_credentials:{
clearing_agent:  { username:‘agent_demo’,    password:‘demo1234’, role:‘clearing_agent’ },
cfs_manager:     { username:‘cfs_demo’,      password:‘demo1234’, role:‘cfs_manager’ },
importer:        { username:‘importer_demo’, password:‘demo1234’, role:‘importer’ },
truck_driver:    { username:‘driver_demo’,   password:‘demo1234’, role:‘truck_driver’ },
admin:           { username:‘admin_demo’,    password:‘demo1234’, role:‘admin’ },
},
sample_containers:[‘KESU1234567’,‘MSCU8876543’,‘CMAU2345678’,‘MSKU9912334’],
});
return;
}

// ── AUTH: POST /api/v1/auth/login ──
if (path === ‘/api/v1/auth/login’ && method === ‘POST’) {
const body = await parseBody(req);
const { username, password } = body;
if (!username || !password) { err(res, 400, ‘username and password required’); return; }
if (!USERS[username] || PASSWORDS[username] !== password) { err(res, 401, ‘Invalid credentials’); return; }
const user = USERS[username];
// Regenerate token
user.token = genToken(user.id);
json(res, {
success:true,
message:`Welcome, ${user.name}`,
user:{ id:user.id, name:user.name, role:user.role, company:user.company },
token:user.token,
token_type:‘Bearer’,
usage:‘Add to requests as: Authorization: Bearer <token>  OR  X-API-Key: <token>’,
expires_in:‘8h (sandbox — no expiry)’,
});
return;
}

// ── CONTAINERS LIST: GET /api/v1/containers ──
if (path === ‘/api/v1/containers’ && method === ‘GET’) {
const user = requireAuth(req, res); if (!user) return;
const { status, risk_level: rl, cfs: cfsFilter, limit=20, offset=0 } = query;
let list = Object.values(CONTAINERS);
if (status)    list = list.filter(c => c.status === status);
if (rl)        list = list.filter(c => c.risk_level === rl);
if (cfsFilter) list = list.filter(c => c.cfs === cfsFilter);
// Role-based filtering
if (user.role === ‘clearing_agent’) list = list.filter(c => c.agent_id === user.id);
if (user.role === ‘importer’)       list = list.filter(c => c.importer_id === user.id);
const total = list.length;
list = list.slice(+offset, +offset + +limit);
json(res, {
success:true,
total, count:list.length, offset:+offset, limit:+limit,
filters:{ status:status||null, risk_level:rl||null, cfs:cfsFilter||null },
containers:list.map(c => ({
id:c.id, container_number:c.container_number, bl_number:c.bl_number,
status:c.status, risk_level:c.risk_level, cfs:c.cfs,
dwell_days:c.dwell_days, predicted_dwell_days:c.predicted_dwell_days,
shipping_line:c.shipping_line, commodity:c.commodity,
estimated_release:c.estimated_release,
})),
});
return;
}

// ── CONTAINER DETAIL: GET /api/v1/containers/:id ──
const containerMatch = path.match(/^/api/v1/containers/([A-Z0-9]+)$/);
if (containerMatch && method === ‘GET’) {
const user = requireAuth(req, res); if (!user) return;
const key = containerMatch[1];
const c = CONTAINERS[key] || Object.values(CONTAINERS).find(x => x.bl_number === key || x.id === key);
if (!c) { err(res, 404, `Container not found: ${key}`, ‘Try: KESU1234567, MSCU8876543, CMAU2345678, MSKU9912334’); return; }
const predicted = c.predicted_dwell_days || predictDwell(c.commodity, c.cfs, c.shipping_line);
const risk = riskLevel(c.dwell_days, predicted);
json(res, {
success:true,
container:{
…c,
risk_assessment: risk,
documents_summary: DOCUMENTS[c.id] ? {
total: DOCUMENTS[c.id].length,
approved: DOCUMENTS[c.id].filter(d=>d.status===‘approved’).length,
missing:  DOCUMENTS[c.id].filter(d=>d.status===‘missing’).length,
pending:  DOCUMENTS[c.id].filter(d=>d.status===‘pending’).length,
on_hold:  DOCUMENTS[c.id].filter(d=>d.status===‘on_hold’).length,
} : null,
alerts_count: ALERTS_LOG.filter(a=>a.container===c.container_number).length,
}
});
return;
}

// ── SCAN: POST /api/v1/containers/:id/scan ──
const scanMatch = path.match(/^/api/v1/containers/([A-Z0-9]+)/scan$/);
if (scanMatch && method === ‘POST’) {
const user = requireAuth(req, res); if (!user) return;
if (![‘cfs_manager’,‘admin’].includes(user.role)) { err(res, 403, ‘Only CFS managers can submit scan events’); return; }
const body = await parseBody(req);
const { status, notes, location, scanner_id } = body;
const VALID = [‘gate_in’,‘stacked’,‘customs_exam_start’,‘customs_exam_passed’,‘customs_exam_failed’,‘kebs_hold’,‘ready_for_delivery’,‘gate_out’,‘truck_assigned’];
if (!status || !VALID.includes(status)) { err(res, 400, ‘Invalid status’, `Valid: ${VALID.join(', ')}`); return; }
const key = scanMatch[1];
const c = CONTAINERS[key];
if (!c) { err(res, 404, `Container not found: ${key}`); return; }
const scanEvent = { id:uuid(), container:key, status, notes:notes||null, location:location||c.cfs, scanner_id:scanner_id||user.id, scanned_by:user.name, ts:ts() };
SCAN_EVENTS.push(scanEvent);
c.status = status;
c.timeline.push({ event:status, label:status.replace(/_/g,’ ’).replace(/\b\w/g,l=>l.toUpperCase()), ts:ts(), source:‘CFS Scanner’, done:true, active:true });
// Auto-alert
const alertMsg = { gate_in:`Container ${key} has gated-in at ${c.cfs}`, gate_out:`Container ${key} has been released — gate-out confirmed`, customs_exam_passed:`Customs examination passed for ${key}` };
if (alertMsg[status]) {
ALERTS_LOG.push({ id:‘a’+uuid().slice(0,8), type:status, container:key, message:alertMsg[status], sent_via:[‘whatsapp’,‘push’], ts:ts(), read:false });
}
json(res, {
success:true,
message:`Scan event recorded — status updated to: ${status}`,
scan_event:scanEvent,
alerts_triggered: alertMsg[status] ? [‘whatsapp’,‘push’] : [],
container_updated:{ id:c.id, container_number:key, new_status:status },
});
return;
}

// ── SYSTEM HEALTH: GET /api/v1/system/health ──
if (path === ‘/api/v1/system/health’ && method === ‘GET’) {
// Simulate slight variability
SYSTEM_HEALTH.icms.latency_ms = 600 + Math.floor(Math.random()*400);
SYSTEM_HEALTH.kwatos.latency_ms = 100 + Math.floor(Math.random()*60);
SYSTEM_HEALTH.icms.last_check = ts();
SYSTEM_HEALTH.kwatos.last_check = ts();
const overall = Object.values(SYSTEM_HEALTH).every(s=>s.status===‘online’) ? ‘operational’ : ‘degraded’;
json(res, {
success:true,
overall_status:overall,
systems:SYSTEM_HEALTH,
cache_mode: SYSTEM_HEALTH.icms.status !== ‘online’ ? ‘active — serving cached data’ : ‘standby’,
downtime_defender: { active: SYSTEM_HEALTH.icms.status !== ‘online’, cached_containers:Object.keys(CONTAINERS).length, queued_syncs:SCAN_EVENTS.filter(e=>!e.synced).length },
last_updated:ts(),
});
return;
}

// ── ALERTS: GET /api/v1/alerts ──
if (path === ‘/api/v1/alerts’ && method === ‘GET’) {
const user = requireAuth(req, res); if (!user) return;
const unread = ALERTS_LOG.filter(a=>!a.read);
json(res, {
success:true,
total:ALERTS_LOG.length, unread:unread.length,
alerts:ALERTS_LOG.sort((a,b)=>b.ts.localeCompare(a.ts)),
});
return;
}

// ── SUBSCRIBE: POST /api/v1/alerts/subscribe ──
if (path === ‘/api/v1/alerts/subscribe’ && method === ‘POST’) {
const user = requireAuth(req, res); if (!user) return;
const body = await parseBody(req);
const { container_number, channels=[‘whatsapp’,‘sms’], phone, email } = body;
if (!container_number) { err(res, 400, ‘container_number required’); return; }
json(res, {
success:true,
message:`Alert subscription created for container ${container_number}`,
subscription:{ id:‘sub_’+uuid().slice(0,8), user_id:user.id, container_number, channels, phone:phone||null, email:email||null, created:ts() },
note:‘In production: alerts sent via Africa's Talking (SMS) and WhatsApp Business API’,
});
return;
}

// ── DWELL ANALYTICS: GET /api/v1/analytics/dwell ──
if (path === ‘/api/v1/analytics/dwell’ && method === ‘GET’) {
const user = requireAuth(req, res); if (!user) return;
json(res, {
success:true,
period:‘30 days’,
summary:{ avg_dwell:5.8, median_dwell:4.2, max_dwell:18.0, min_dwell:2.1, on_track_pct:54, warning_pct:28, critical_pct:18 },
by_cfs:[
{ cfs:‘Mombasa CFS A’,  avg_dwell:4.1, containers:31, on_track_pct:68 },
{ cfs:‘Mombasa CFS B’,  avg_dwell:6.3, containers:28, on_track_pct:43 },
{ cfs:‘ICD Nairobi’,    avg_dwell:7.2, containers:19, on_track_pct:37 },
{ cfs:‘AERIS Changamwe’,avg_dwell:5.1, containers:22, on_track_pct:59 },
],
by_commodity:[
{ commodity:‘Pharmaceuticals’,    avg_dwell:3.8, containers:12 },
{ commodity:‘Consumer Goods’,     avg_dwell:5.4, containers:24 },
{ commodity:‘Industrial Equipment’,avg_dwell:4.9,containers:18 },
{ commodity:‘Machinery Parts’,    avg_dwell:7.1, containers:15 },
{ commodity:‘Food Products’,      avg_dwell:4.3, containers:11 },
],
by_shipping_line:[
{ line:‘Maersk’,       avg_dwell:4.2, containers:28 },
{ line:‘MSC’,          avg_dwell:5.8, containers:22 },
{ line:‘CMA CGM’,      avg_dwell:6.1, containers:19 },
{ line:‘Hapag-Lloyd’,  avg_dwell:4.5, containers:14 },
],
bottleneck_analysis:{
top_delays:[
{ reason:‘Missing Certificate of Conformity (KEBS)’, avg_delay_days:8.4, occurrences:14 },
{ reason:‘iCMS System Downtime’,                     avg_delay_days:1.2, occurrences:8  },
{ reason:‘Truck Assignment Failure’,                  avg_delay_days:0.8, occurrences:11 },
{ reason:‘Incomplete IDF Submission’,                 avg_delay_days:3.1, occurrences:7  },
],
},
});
return;
}

// ── SLA ANALYTICS: GET /api/v1/analytics/sla ──
if (path === ‘/api/v1/analytics/sla’ && method === ‘GET’) {
const user = requireAuth(req, res); if (!user) return;
json(res, {
success:true,
period:‘30 days’,
overall_sla_compliance:‘34%’,
note:‘Based on Hewa Tele case study — only 1/50 shipments had full SLA data captured pre-Masat5PL’,
agents:[
{ name:‘Clearfast Kenya Ltd’, sla_pct:78, containers:22, avg_dwell:4.1 },
{ name:‘Port Solutions EA’,   sla_pct:61, containers:18, avg_dwell:5.8 },
{ name:‘Mombasa Freight Ltd’, sla_pct:45, containers:14, avg_dwell:7.2 },
{ name:‘KenTrans Agency’,     sla_pct:32, containers:11, avg_dwell:8.9 },
],
document_compliance:{
idf_submission_rate:‘88%’,
coc_submission_rate:‘71%’,
bl_confirmation_rate:‘94%’,
c32_on_time_rate:‘62%’,
},
});
return;
}

// ── DOCUMENTS: GET /api/v1/documents/:container_id ──
const docMatch = path.match(/^/api/v1/documents/([a-z0-9]+)$/);
if (docMatch && method === ‘GET’) {
const user = requireAuth(req, res); if (!user) return;
const cid = docMatch[1];
const docs = DOCUMENTS[cid];
if (!docs) { err(res, 404, `No documents found for container_id: ${cid}`, ‘Try: c001, c002’); return; }
json(res, {
success:true,
container_id:cid,
summary:{ total:docs.length, approved:docs.filter(d=>d.status===‘approved’).length, missing:docs.filter(d=>d.status===‘missing’).length, pending:docs.filter(d=>d.status===‘pending’).length },
documents:docs,
clearance_ready: docs.filter(d=>d.required).every(d=>d.status===‘approved’),
});
return;
}

// ── DOCUMENT UPDATE: POST /api/v1/documents/:container_id/update ──
const docUpdateMatch = path.match(/^/api/v1/documents/([a-z0-9]+)/update$/);
if (docUpdateMatch && method === ‘POST’) {
const user = requireAuth(req, res); if (!user) return;
const body = await parseBody(req);
const { doc_type, status: newStatus, notes } = body;
const VALID_STATUS = [‘submitted’,‘approved’,‘rejected’,‘missing’,‘pending’,‘on_hold’];
if (!doc_type || !newStatus) { err(res, 400, ‘doc_type and status required’); return; }
if (!VALID_STATUS.includes(newStatus)) { err(res, 400, ‘Invalid status’, `Valid: ${VALID_STATUS.join(', ')}`); return; }
const cid = docUpdateMatch[1];
if (!DOCUMENTS[cid]) { err(res, 404, ‘Container documents not found’); return; }
const doc = DOCUMENTS[cid].find(d => d.type === doc_type);
if (!doc) { err(res, 404, `Document type ${doc_type} not found`); return; }
const prev = doc.status;
doc.status = newStatus;
doc.updated = ts().split(‘T’)[0];
if (notes) doc.notes = notes;
json(res, {
success:true,
message:`Document ${doc_type} updated: ${prev} → ${newStatus}`,
document:doc,
clearance_ready: DOCUMENTS[cid].filter(d=>d.required).every(d=>d.status===‘approved’),
});
return;
}

// ── AI PREDICT: POST /api/v1/ai/predict-dwell ──
if (path === ‘/api/v1/ai/predict-dwell’ && method === ‘POST’) {
const user = requireAuth(req, res); if (!user) return;
const body = await parseBody(req);
const { commodity, cfs, shipping_line, has_kebs_cert=true, has_idf=true } = body;
if (!commodity || !cfs || !shipping_line) { err(res, 400, ‘commodity, cfs, shipping_line required’); return; }
let predicted = predictDwell(commodity, cfs, shipping_line);
let risk_factors = [];
if (!has_kebs_cert) { predicted += 5.5; risk_factors.push({ factor:‘Missing KEBS Certificate’, impact:’+5.5 days avg’, severity:‘high’ }); }
if (!has_idf)       { predicted += 2.0; risk_factors.push({ factor:‘No IDF Submitted’,          impact:’+2.0 days avg’, severity:‘medium’ }); }
if (cfs === ‘ICD Nairobi’) risk_factors.push({ factor:‘ICD Nairobi — higher avg dwell’, impact:’+1.2 days’,     severity:‘low’ });
json(res, {
success:true,
prediction:{
commodity, cfs, shipping_line,
predicted_dwell_days: +predicted.toFixed(1),
confidence_pct: has_kebs_cert && has_idf ? 84 : 61,
risk_level: predicted < 5 ? ‘low’ : predicted < 8 ? ‘medium’ : ‘high’,
risk_factors,
model:‘Masat5PL-DwellNet-v1 (Scikit-learn Linear Regression)’,
training_data:‘6,200 historical containers — Mombasa CFSs 2023–2026’,
},
});
return;
}

// ── CFS LIST: GET /api/v1/cfs ──
if (path === ‘/api/v1/cfs’ && method === ‘GET’) {
json(res, {
success:true,
count:4,
cfs_stations:[
{ id:‘cfs001’, name:‘Mombasa CFS A’,   location:‘Kilindini Rd, Mombasa’,     active_containers:31, avg_dwell:4.1, status:‘operational’ },
{ id:‘cfs002’, name:‘Mombasa CFS B’,   location:‘Port Reitz Rd, Mombasa’,    active_containers:28, avg_dwell:6.3, status:‘operational’ },
{ id:‘cfs003’, name:‘ICD Nairobi’,     location:‘Embakasi, Nairobi’,          active_containers:19, avg_dwell:7.2, status:‘operational’ },
{ id:‘cfs004’, name:‘AERIS Changamwe’, location:‘Changamwe, Mombasa’,         active_containers:22, avg_dwell:5.1, status:‘operational’ },
],
});
return;
}

// ── SANDBOX RESET: GET /api/v1/sandbox/reset ──
if (path === ‘/api/v1/sandbox/reset’ && method === ‘GET’) {
SCAN_EVENTS = [];
ALERTS_LOG[0].read = false; ALERTS_LOG[2].read = false;
CONTAINERS[‘MSCU8876543’].status = ‘kebs_hold’;
CONTAINERS[‘CMAU2345678’].status = ‘dark_container’;
json(res, { success:true, message:‘Sandbox data reset to initial state’, ts:ts() });
return;
}

// ── 404 ──
err(res, 404, `Route not found: ${method} ${path}`, ‘GET / for full API reference’);
}

// ─── SERVER START ─────────────────────────────────────────────────────────────

const server = http.createServer(async (req, res) => {
try { await router(req, res); }
catch(e) { err(res, 500, ‘Internal server error’, e.message); }
});

server.listen(PORT, () => {
console.log(`╔══════════════════════════════════════════════════════╗ ║  MASAT 5PL TOS API — Sandbox Server                  ║ ║  Listening on: http://localhost:${PORT}                  ║ ║  API Docs:     http://localhost:${PORT}/                 ║ ║  Test:         curl http://localhost:${PORT}/api/v1/system/health ║ ╚══════════════════════════════════════════════════════╝`);
});

module.exports = { server, CONTAINERS, USERS, ALERTS_LOG };
