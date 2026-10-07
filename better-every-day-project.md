# Better Every Day: full project source

Create each file below at the given path, then follow the README steps.

## README.md

```md
# Better Every Day
Small habits. Better days. Better you.

React + TypeScript + Vite + Tailwind frontend, Express + TypeScript REST API, PostgreSQL. Username + password only (no email).

## Setup
1. Create the database and load the schema (creates tables and seeds the 10 activities):
   `createdb better_every_day && psql better_every_day -f database/schema.sql`
2. Backend: `cd backend && cp .env.example .env` (edit DATABASE_URL and JWT_SECRET), then `npm install && npm run dev` (port 4000).
3. Frontend: `cd frontend && npm install && npm run dev` (port 5173, proxies /api to the backend).

## API (Bearer token auth except register/login)
| Method | Path | Notes |
|---|---|---|
| POST | /api/auth/register | {username, password, confirmPassword} |
| POST | /api/auth/login | {username, password} |
| POST | /api/auth/logout | client discards token |
| GET | /api/tasks | the 10 activities |
| GET | /api/tasks/today?date=YYYY-MM-DD | activities + completed flag |
| GET | /api/progress/today?date= | {completed, total, percent} |
| PUT | /api/tasks/:taskId/complete | body {date} |
| PUT | /api/tasks/:taskId/uncomplete | body {date} |
| GET | /api/history | per-date completed counts |
| GET | /api/history/:date | activities for that date |
| GET | /api/statistics?date= | weekly/monthly, streaks, totals |
| GET | /api/streak?date= | {current, longest} |

The client sends its local date so "today" follows the user's timezone. Each date is tracked separately; a day is "full" at 10/10.
```

## backend/.env.example

```
DATABASE_URL=postgres://postgres:password@localhost:5432/better_every_day
JWT_SECRET=change-me-to-a-long-random-string
PORT=4000
CLIENT_ORIGIN=http://localhost:5173
```

## backend/package.json

```json
{"name":"bed-backend","private":true,"scripts":{"dev":"tsx watch src/index.ts","build":"tsc","start":"node dist/index.js"},
"dependencies":{"bcryptjs":"^2.4.3","cors":"^2.8.5","dotenv":"^16.4.5","express":"^4.19.2","jsonwebtoken":"^9.0.2","pg":"^8.12.0"},
"devDependencies":{"@types/bcryptjs":"^2.4.6","@types/cors":"^2.8.17","@types/express":"^4.17.21","@types/jsonwebtoken":"^9.0.6","@types/node":"^20.14.0","@types/pg":"^8.11.6","tsx":"^4.16.0","typescript":"^5.5.0"}}
```

## backend/src/db.ts

```ts
import pg from 'pg';
export const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });
```

## backend/src/index.ts

```ts
import 'dotenv/config';
import express, { RequestHandler } from 'express';
import cors from 'cors';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { pool } from './db';
import { DATE_RE, addDays, todayOr, streaks, Row } from './utils';

if (!process.env.JWT_SECRET) throw new Error('JWT_SECRET is required');
const SECRET = process.env.JWT_SECRET;
const app = express();
app.use(cors({ origin: process.env.CLIENT_ORIGIN || 'http://localhost:5173' }));
app.use(express.json());

const auth: RequestHandler = (req, res, next) => {
  try {
    const t = (req.headers.authorization || '').replace('Bearer ', '');
    (req as any).uid = (jwt.verify(t, SECRET) as any).uid; next();
  } catch { res.status(401).json({ error: 'Please log in again.' }); }
};
const wrap = (fn: RequestHandler): RequestHandler => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
const uid = (req: any): number => req.uid;
const sign = (u: { id: number; username: string }) =>
  ({ token: jwt.sign({ uid: u.id }, SECRET, { expiresIn: '30d' }), user: { id: u.id, username: u.username } });

app.post('/api/auth/register', wrap(async (req, res) => {
  const { username, password, confirmPassword } = req.body || {};
  if (typeof username !== 'string' || !/^[A-Za-z0-9_]{3,30}$/.test(username)) return res.status(400).json({ error: 'Username must be 3-30 letters, numbers or underscores.' });
  if (typeof password !== 'string' || password.length < 6) return res.status(400).json({ error: 'Password must be at least 6 characters.' });
  if (password !== confirmPassword) return res.status(400).json({ error: "Passwords don't match." });
  try {
    const r = await pool.query('INSERT INTO users(username,password_hash) VALUES($1,$2) RETURNING id,username', [username, await bcrypt.hash(password, 12)]);
    res.status(201).json(sign(r.rows[0]));
  } catch (e: any) {
    if (e.code === '23505') return res.status(409).json({ error: 'That username is taken.' });
    throw e;
  }
}));
app.post('/api/auth/login', wrap(async (req, res) => {
  const { username, password } = req.body || {};
  const r = await pool.query('SELECT id,username,password_hash FROM users WHERE lower(username)=lower($1)', [String(username || '')]);
  const u = r.rows[0];
  if (!u || !(await bcrypt.compare(String(password || ''), u.password_hash))) return res.status(401).json({ error: 'Username or password is incorrect.' });
  res.json(sign(u));
}));
app.post('/api/auth/logout', (_req, res) => res.json({ ok: true }));

app.get('/api/tasks', auth, wrap(async (_req, res) => {
  res.json((await pool.query('SELECT id,task_number,task_text FROM tasks ORDER BY task_number')).rows);
}));

const forDate = async (user: number, date: string) => (await pool.query(
  `SELECT t.id,t.task_number,t.task_text,COALESCE(d.completed,false) AS completed
   FROM tasks t LEFT JOIN daily_tasks d ON d.task_id=t.id AND d.user_id=$1 AND d.date=$2 ORDER BY t.task_number`, [user, date])).rows;

app.get('/api/tasks/today', auth, wrap(async (req, res) => res.json(await forDate(uid(req), todayOr(req.query.date)))));
app.get('/api/progress/today', auth, wrap(async (req, res) => {
  const t = await forDate(uid(req), todayOr(req.query.date));
  const completed = t.filter(x => x.completed).length;
  res.json({ completed, total: t.length, percent: Math.round((completed / t.length) * 100) });
}));

const setDone = (done: boolean): RequestHandler => wrap(async (req, res) => {
  const taskId = Number(req.params.taskId), date = req.body?.date;
  if (!Number.isInteger(taskId) || !DATE_RE.test(date || '')) return res.status(400).json({ error: 'Invalid task or date.' });
  if (date > addDays(todayOr(undefined), 1)) return res.status(400).json({ error: 'You cannot complete future days.' });
  const r = await pool.query(
    `INSERT INTO daily_tasks(user_id,task_id,date,completed,completed_at)
     SELECT $1,id,$3,$4::boolean,CASE WHEN $4::boolean THEN now() END FROM tasks WHERE id=$2
     ON CONFLICT(user_id,task_id,date) DO UPDATE SET completed=$4::boolean,completed_at=CASE WHEN $4::boolean THEN now() END
     RETURNING task_id,completed`, [uid(req), taskId, date, done]);
  if (!r.rowCount) return res.status(404).json({ error: 'Task not found.' });
  res.json(r.rows[0]);
});
app.put('/api/tasks/:taskId/complete', auth, setDone(true));
app.put('/api/tasks/:taskId/uncomplete', auth, setDone(false));

const histRows = async (user: number): Promise<Row[]> => (await pool.query(
  `SELECT to_char(date,'YYYY-MM-DD') AS date,(count(*) FILTER (WHERE completed))::int AS done
   FROM daily_tasks WHERE user_id=$1 GROUP BY date ORDER BY date`, [user])).rows;

app.get('/api/history', auth, wrap(async (req, res) => res.json(await histRows(uid(req)))));
app.get('/api/history/:date', auth, wrap(async (req, res) => {
  if (!DATE_RE.test(req.params.date)) return res.status(400).json({ error: 'Invalid date.' });
  const tasks = await forDate(uid(req), req.params.date);
  const done = tasks.filter(t => t.completed).length;
  res.json({ date: req.params.date, tasks, completed: done, percent: done * 10 });
}));
app.get('/api/streak', auth, wrap(async (req, res) => res.json(streaks(await histRows(uid(req)), todayOr(req.query.date)))));
app.get('/api/statistics', auth, wrap(async (req, res) => {
  const today = todayOr(req.query.date), rows = await histRows(uid(req)), m = new Map(rows.map(r => [r.date, r.done]));
  const range = (n: number) => Array.from({ length: n }, (_, i) => { const d = addDays(today, i - n + 1); return { date: d, percent: (m.get(d) || 0) * 10 }; });
  const avg = (r: { percent: number }[]) => Math.round(r.reduce((s, x) => s + x.percent, 0) / r.length);
  const week = range(7), month = range(30);
  res.json({ today: (m.get(today) || 0) * 10, week, month, weekAvg: avg(week), monthAvg: avg(month), ...streaks(rows, today),
    totalCompleted: rows.reduce((s, r) => s + r.done, 0), activeDays: rows.filter(r => r.done > 0).length });
}));

app.use((err: any, _req: any, res: any, _next: any) => { console.error(err); res.status(500).json({ error: 'Something went wrong on the server.' }); });
app.listen(Number(process.env.PORT) || 4000, () => console.log('API running'));
```

## backend/src/utils.ts

```ts
export const DATE_RE = /^\d{4}-\d{2}-\d{2}$/;
export const addDays = (s: string, n: number) => {
  const d = new Date(s + 'T00:00:00Z'); d.setUTCDate(d.getUTCDate() + n); return d.toISOString().slice(0, 10);
};
export const todayOr = (q: unknown) => (typeof q === 'string' && DATE_RE.test(q) ? q : new Date().toISOString().slice(0, 10));
export type Row = { date: string; done: number };
export function streaks(rows: Row[], today: string) {
  const full = new Set(rows.filter(r => r.done === 10).map(r => r.date));
  let longest = 0, run = 0, prev = '';
  [...full].sort().forEach(d => { run = prev && addDays(prev, 1) === d ? run + 1 : 1; longest = Math.max(longest, run); prev = d; });
  let current = 0, c = full.has(today) ? today : addDays(today, -1);
  while (full.has(c)) { current++; c = addDays(c, -1); }
  return { current, longest };
}
```

## backend/tsconfig.json

```json
{"compilerOptions":{"target":"ES2020","module":"commonjs","outDir":"dist","rootDir":"src","strict":true,"esModuleInterop":true,"skipLibCheck":true}}
```

## database/schema.sql

```sql
CREATE TABLE IF NOT EXISTS users(
  id SERIAL PRIMARY KEY,
  username VARCHAR(30) NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now());
CREATE TABLE IF NOT EXISTS tasks(
  id SERIAL PRIMARY KEY,
  task_number INT NOT NULL UNIQUE,
  task_text TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now());
CREATE TABLE IF NOT EXISTS daily_tasks(
  id SERIAL PRIMARY KEY,
  user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  task_id INT NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  date DATE NOT NULL,
  completed BOOLEAN NOT NULL DEFAULT false,
  completed_at TIMESTAMPTZ,
  UNIQUE(user_id,task_id,date));
CREATE INDEX IF NOT EXISTS idx_daily_user_date ON daily_tasks(user_id,date);
INSERT INTO tasks(task_number,task_text) VALUES
(1,'Limit your screen time.'),(2,'Learn something new.'),(3,'Read for 10 minutes.'),
(4,'Practice speaking English.'),(5,'Learn 5 new words.'),(6,'Listen carefully when others speak.'),
(7,'Speak politely with everyone.'),(8,'Keep your surroundings clean.'),(9,'Exercise for 30 minutes.'),
(10,'Reflect on your day before sleeping.') ON CONFLICT DO NOTHING;
```

## frontend/index.html

```html
<!doctype html><html lang="en" class="dark"><head><meta charset="UTF-8"/><meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Better Every Day</title>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&family=Poppins:wght@600;700&display=swap" rel="stylesheet"/>
</head><body><div id="root"></div><script type="module" src="/src/main.tsx"></script></body></html>
```

## frontend/package.json

```json
{"name":"bed-frontend","private":true,"type":"module","scripts":{"dev":"vite","build":"tsc && vite build","preview":"vite preview"},
"dependencies":{"lucide-react":"^0.453.0","react":"^18.3.1","react-dom":"^18.3.1"},
"devDependencies":{"@types/react":"^18.3.3","@types/react-dom":"^18.3.0","@vitejs/plugin-react":"^4.3.1","autoprefixer":"^10.4.19","postcss":"^8.4.39","tailwindcss":"^3.4.6","typescript":"^5.5.0","vite":"^5.3.0"}}
```

## frontend/postcss.config.js

```js
export default {plugins:{tailwindcss:{},autoprefixer:{}}}
```

## frontend/src/App.tsx

```tsx
import { useCallback, useEffect, useState } from 'react';
import { Check, Flame, Trophy, Moon, Sun, LogOut, Loader2, ChevronLeft, ChevronRight } from 'lucide-react';
import { api, iso, shift, fmt } from './api';
import Auth from './Auth';

type Task = { id: number; task_number: number; task_text: string; completed: boolean };
type Hist = { date: string; done: number };
const QUOTES = ['Small progress is still progress.', 'Build better habits, one day at a time.', 'Your daily actions shape your future.', 'Learn something. Do something. Improve something.'];
const card = 'rounded-2xl border border-slate-200 dark:border-slate-700 bg-white dark:bg-[#161d3d] p-5';
const today = () => iso(new Date());

function Err({ msg }: { msg: string }) { return msg ? <div role="alert" className="mb-4 rounded-xl bg-rose-500/10 text-rose-500 px-4 py-3 text-sm">{msg}</div> : null; }
function Stat({ v, l, c = '' }: { v: React.ReactNode; l: string; c?: string }) { return <div className={card}><div className={`font-display text-4xl font-bold ${c}`}>{v}</div><div className="text-sm text-slate-500 dark:text-slate-400 mt-1">{l}</div></div>; }

function Dashboard() {
  const [date, setDate] = useState(today()), [tasks, setTasks] = useState<Task[]>([]), [streak, setStreak] = useState({ current: 0, longest: 0 }), [total, setTotal] = useState(0), [err, setErr] = useState(''), [loading, setLoading] = useState(true);
  const refresh = useCallback(async () => {
    try {
      const [t, s, st] = await Promise.all([api<Task[]>(`/tasks/today?date=${date}`), api(`/streak?date=${today()}`), api(`/statistics?date=${today()}`)]);
      setTasks(t); setStreak(s); setTotal(st.totalCompleted); setErr('');
    } catch (e: any) { setErr(e.message); } finally { setLoading(false); }
  }, [date]);
  useEffect(() => { setLoading(true); refresh(); }, [refresh]);
  async function toggle(t: Task) {
    setTasks(ts => ts.map(x => x.id === t.id ? { ...x, completed: !x.completed } : x));
    try { await api(`/tasks/${t.id}/${t.completed ? 'uncomplete' : 'complete'}`, 'PUT', { date }); refresh(); }
    catch (e: any) { setErr(e.message); refresh(); }
  }
  const done = tasks.filter(t => t.completed).length, pct = tasks.length ? Math.round(done / tasks.length * 100) : 0;
  const q = QUOTES[Math.floor(Date.now() / 864e5) % QUOTES.length];
  return (
    <div className="fade">
      <p className="text-sm text-slate-500">{fmt(today())}</p>
      <h1 className="font-display text-3xl font-bold mt-1">Hello, {localStorage.getItem('bed_user')}</h1>
      <p className="italic text-slate-500 dark:text-slate-400 mb-4">“{q}”</p>
      <div className="flex items-center gap-2 mb-4">
        <button aria-label="Previous day" onClick={() => setDate(shift(date, -1))} className="p-2 rounded-lg hover:bg-slate-500/10"><ChevronLeft size={18} /></button>
        <input type="date" max={today()} value={date} onChange={e => e.target.value && setDate(e.target.value)} className="rounded-lg border border-slate-300 dark:border-slate-700 bg-transparent px-2 py-1" />
        <button aria-label="Next day" disabled={date >= today()} onClick={() => setDate(shift(date, 1))} className="p-2 rounded-lg hover:bg-slate-500/10 disabled:opacity-30"><ChevronRight size={18} /></button>
      </div>
      <Err msg={err} />
      <div className="grid gap-4 md:grid-cols-3">
        <div className={card}>
          <div className="text-sm text-slate-500">{date === today() ? "Today's progress" : fmt(date, { month: 'short', day: 'numeric' }) + ' progress'}</div>
          <div className="font-display text-4xl font-bold mt-1">{done} / {tasks.length || 10}</div>
          <div className="text-sm text-slate-500">{pct}% completed</div>
          <div className="h-2.5 rounded-full bg-slate-500/15 mt-3 overflow-hidden"><div className="h-full rounded-full bg-gradient-to-r from-indigo-500 to-emerald-400 transition-all duration-700" style={{ width: pct + '%' }} /></div>
        </div>
        <Stat v={<><Flame className="inline -mt-1 mr-1" />{streak.current}</>} l="Current streak (days)" c="text-amber-400" />
        <Stat v={total} l="Total activities completed" />
      </div>
      {loading ? <div className="flex justify-center py-12"><Loader2 className="animate-spin" /></div> :
        <div className="grid gap-3 sm:grid-cols-2 mt-5">
          {tasks.map(t => (
            <button key={t.id} onClick={() => toggle(t)} aria-pressed={t.completed}
              className={`flex items-center gap-4 text-left rounded-2xl border p-4 transition hover:-translate-y-0.5 ${t.completed ? 'bg-emerald-500/10 border-emerald-500' : 'bg-white dark:bg-[#161d3d] border-slate-200 dark:border-slate-700 hover:border-indigo-400'}`}>
              <span className="w-10 h-10 rounded-xl bg-slate-500/10 grid place-items-center font-display font-semibold shrink-0">{t.task_number}</span>
              <span className={`flex-1 font-medium ${t.completed ? 'line-through decoration-emerald-500/60' : ''}`}>{t.task_text}</span>
              <span className={`w-8 h-8 rounded-full border-2 grid place-items-center shrink-0 transition ${t.completed ? 'bg-emerald-500 border-emerald-500 text-white pop' : 'border-slate-400/50'}`}>{t.completed && <Check size={18} strokeWidth={3} />}</span>
            </button>))}
        </div>}
    </div>
  );
}

function History() {
  const [rows, setRows] = useState<Hist[]>([]), [month, setMonth] = useState(today().slice(0, 7)), [sel, setSel] = useState(today()), [detail, setDetail] = useState<any>(null), [err, setErr] = useState('');
  useEffect(() => { api('/history').then(setRows).catch(e => setErr(e.message)); }, []);
  useEffect(() => { api(`/history/${sel}`).then(setDetail).catch(e => setErr(e.message)); }, [sel]);
  const m = new Map(rows.map(r => [r.date, r.done])), [y, mo] = month.split('-').map(Number), first = new Date(y, mo - 1, 1).getDay(), dim = new Date(y, mo, 0).getDate();
  const go = (n: number) => { const d = new Date(y, mo - 1 + n, 1); setMonth(iso(d).slice(0, 7)); };
  return (
    <div className="fade"><h1 className="font-display text-3xl font-bold mb-4">History</h1><Err msg={err} />
      <div className="grid gap-4 md:grid-cols-2">
        <div className={card}>
          <div className="flex justify-between items-center mb-3"><button aria-label="Previous month" onClick={() => go(-1)}><ChevronLeft /></button><h3 className="font-display font-semibold">{new Date(y, mo - 1).toLocaleDateString('en-US', { month: 'long', year: 'numeric' })}</h3><button aria-label="Next month" onClick={() => go(1)}><ChevronRight /></button></div>
          <div className="grid grid-cols-7 gap-1.5 text-center text-xs text-slate-500">
            {'SMTWTFS'.split('').map((d, i) => <b key={i}>{d}</b>)}{Array.from({ length: first }, (_, i) => <span key={'e' + i} />)}
            {Array.from({ length: dim }, (_, i) => { const s = `${month}-${String(i + 1).padStart(2, '0')}`, n = m.get(s) || 0;
              return <button key={s} disabled={s > today()} onClick={() => setSel(s)} className={`aspect-square rounded-lg border text-sm disabled:opacity-30 ${s === sel ? 'ring-2 ring-indigo-500' : ''} ${n === 10 ? 'bg-emerald-500/20 border-emerald-500' : n > 0 ? 'bg-amber-400/20 border-amber-400' : 'border-slate-300 dark:border-slate-700'}`}>{i + 1}</button>; })}
          </div>
          <div className="flex gap-4 text-xs text-slate-500 mt-4"><span>🟢 Fully completed</span><span>🟡 Partial</span><span>⚪ No activity</span></div>
        </div>
        <div className={card}>
          <h3 className="font-display font-semibold">{fmt(sel)}</h3>
          {detail && detail.completed === 0 && !m.has(sel) && <p className="text-slate-500 py-6 text-center">No activity recorded for this day.</p>}
          {detail && <><div className="font-display text-4xl font-bold mt-3">{detail.percent}%</div><div className="text-sm text-slate-500 mb-3">Score {detail.completed} / 10</div>
            <ul className="space-y-2 text-sm">{detail.tasks.map((t: Task) => <li key={t.id} className={t.completed ? '' : 'text-slate-500'}>{t.completed ? '✅' : '⬜'} {t.task_text}</li>)}</ul></>}
        </div>
      </div></div>
  );
}

function Bars({ data, label }: { data: { date: string; percent: number }[]; label: (d: string) => string }) {
  return <div className="flex items-end gap-1.5 h-40 mt-4">{data.map(d => <div key={d.date} title={`${d.date}: ${d.percent}%`} className="flex-1 h-full flex flex-col justify-end items-center gap-1 text-[10px] text-slate-500"><div className="w-full max-w-9 rounded-t-lg bg-gradient-to-t from-indigo-500 to-purple-500 transition-all duration-700" style={{ height: Math.max(d.percent, 2) + '%' }} />{label(d.date)}</div>)}</div>;
}
function Stats() {
  const [s, setS] = useState<any>(null), [err, setErr] = useState('');
  useEffect(() => { api(`/statistics?date=${today()}`).then(setS).catch(e => setErr(e.message)); }, []);
  if (!s) return err ? <Err msg={err} /> : <div className="flex justify-center py-12"><Loader2 className="animate-spin" /></div>;
  return (
    <div className="fade"><h1 className="font-display text-3xl font-bold mb-4">Statistics</h1>
      <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
        <Stat v={s.today + '%'} l="Today" /><Stat v={s.weekAvg + '%'} l="Weekly average" /><Stat v={s.monthAvg + '%'} l="Monthly average" /><Stat v={s.activeDays} l="Active days" />
        <Stat v={<><Flame className="inline -mt-1" /> {s.current}</>} l="Current streak" c="text-amber-400" /><Stat v={<><Trophy className="inline -mt-1" /> {s.longest}</>} l="Longest streak" /><div className="col-span-2"><Stat v={s.totalCompleted} l="Total completed activities" /></div>
      </div>
      <div className={card + ' mt-4'}><h3 className="font-display font-semibold">This week</h3><Bars data={s.week} label={d => fmt(d, { weekday: 'short' })} /></div>
      <div className={card + ' mt-4'}><h3 className="font-display font-semibold">Last 30 days</h3><Bars data={s.month} label={d => String(+d.slice(8))} /></div>
    </div>
  );
}

export default function App() {
  const [user, setUser] = useState<{ username: string } | null>(localStorage.getItem('bed_token') ? { username: localStorage.getItem('bed_user') || '' } : null);
  const [view, setView] = useState<'today' | 'history' | 'stats'>('today');
  const [dark, setDark] = useState(localStorage.getItem('bed_theme') !== 'light');
  useEffect(() => { document.documentElement.classList.toggle('dark', dark); localStorage.setItem('bed_theme', dark ? 'dark' : 'light'); }, [dark]);
  const logout = () => { api('/auth/logout', 'POST').catch(() => {}); localStorage.removeItem('bed_token'); setUser(null); };
  useEffect(() => { if (user) api('/tasks').catch(e => e.status === 401 && logout()); }, []);
  if (!user) return <Auth onAuth={u => { localStorage.setItem('bed_user', u.username); setUser(u); }} />;
  const tab = (v: typeof view, l: string) => <button onClick={() => setView(v)} className={`px-3 py-1.5 rounded-lg text-sm font-medium ${view === v ? 'bg-slate-500/15' : 'text-slate-500 hover:text-inherit'}`}>{l}</button>;
  return (
    <>
      <nav className="sticky top-0 z-10 backdrop-blur border-b border-slate-200 dark:border-slate-800">
        <div className="max-w-4xl mx-auto px-4 py-3 flex items-center gap-1 flex-wrap">
          <span className="font-display font-bold mr-auto bg-gradient-to-r from-indigo-400 to-purple-500 bg-clip-text text-transparent">Better Every Day</span>
          {tab('today', 'Today')}{tab('history', 'History')}{tab('stats', 'Statistics')}
          <button aria-label="Toggle theme" onClick={() => setDark(!dark)} className="p-2">{dark ? <Sun size={18} /> : <Moon size={18} />}</button>
          <button aria-label="Log out" onClick={logout} className="p-2"><LogOut size={18} /></button>
        </div>
      </nav>
      <main className="max-w-4xl mx-auto px-4 py-6">{view === 'today' ? <Dashboard /> : view === 'history' ? <History /> : <Stats />}</main>
      <footer className="text-center text-sm text-slate-500 pb-8">Small habits. Better days. Better you.</footer>
    </>
  );
}
```

## frontend/src/Auth.tsx

```tsx
import { useState } from 'react';
import { api } from './api';
export default function Auth({ onAuth }: { onAuth: (u: { username: string }) => void }) {
  const [reg, setReg] = useState(false), [u, setU] = useState(''), [p, setP] = useState(''), [c, setC] = useState(''), [err, setErr] = useState(''), [busy, setBusy] = useState(false);
  async function go(e: React.FormEvent) {
    e.preventDefault(); setErr('');
    if (reg && (u.trim().length < 3 || p.length < 6)) return setErr('Username needs 3+ characters and password 6+.');
    if (reg && p !== c) return setErr("Passwords don't match.");
    setBusy(true);
    try {
      const r = await api(reg ? '/auth/register' : '/auth/login', 'POST', { username: u.trim(), password: p, confirmPassword: c });
      localStorage.setItem('bed_token', r.token); onAuth(r.user);
    } catch (x: any) { setErr(x.message); } finally { setBusy(false); }
  }
  const inp = 'w-full rounded-xl border border-slate-300 dark:border-slate-700 bg-transparent px-3 py-3 mb-3 focus:outline-none focus:ring-2 focus:ring-indigo-500';
  return (
    <form onSubmit={go} className="fade mx-auto mt-[10vh] max-w-sm rounded-2xl border border-slate-200 dark:border-slate-700 bg-white dark:bg-[#161d3d] p-6 m-4">
      <h1 className="font-display text-3xl font-bold bg-gradient-to-r from-indigo-400 to-purple-500 bg-clip-text text-transparent">Better Every Day</h1>
      <p className="text-slate-500 dark:text-slate-400 mb-5">Small habits. Better days. Better you.</p>
      <input className={inp} placeholder="Username" autoComplete="username" value={u} onChange={e => setU(e.target.value)} />
      <input className={inp} type="password" placeholder="Password" autoComplete={reg ? 'new-password' : 'current-password'} value={p} onChange={e => setP(e.target.value)} />
      {reg && <input className={inp} type="password" placeholder="Confirm password" autoComplete="new-password" value={c} onChange={e => setC(e.target.value)} />}
      <p className="text-rose-500 text-sm min-h-5 mb-2" role="alert">{err}</p>
      <button disabled={busy} className="w-full rounded-xl bg-gradient-to-r from-indigo-500 to-purple-500 py-3 font-semibold text-white hover:-translate-y-0.5 transition disabled:opacity-60">{busy ? 'Please wait…' : reg ? 'Create account' : 'Log in'}</button>
      <p className="text-center text-sm text-slate-500 mt-4">{reg ? 'Already have an account?' : 'New here?'} <button type="button" className="font-semibold text-indigo-500" onClick={() => { setReg(!reg); setErr(''); }}>{reg ? 'Log in' : 'Create an account'}</button></p>
    </form>
  );
}
```

## frontend/src/api.ts

```ts
const BASE = import.meta.env.VITE_API_URL || '/api';
export async function api<T = any>(path: string, method = 'GET', body?: unknown): Promise<T> {
  const t = localStorage.getItem('bed_token');
  let r: Response;
  try {
    r = await fetch(BASE + path, { method, headers: { 'Content-Type': 'application/json', ...(t ? { Authorization: 'Bearer ' + t } : {}) }, body: body ? JSON.stringify(body) : undefined });
  } catch { throw new Error('Cannot reach the server. Check your connection and try again.'); }
  const j = await r.json().catch(() => ({}));
  if (!r.ok) throw Object.assign(new Error(j.error || 'Request failed.'), { status: r.status });
  return j;
}
export const iso = (d: Date) => `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`;
export const shift = (s: string, n: number) => { const d = new Date(s + 'T12:00:00'); d.setDate(d.getDate() + n); return iso(d); };
export const fmt = (s: string, o: Intl.DateTimeFormatOptions = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' }) => new Date(s + 'T12:00:00').toLocaleDateString('en-US', o);
```

## frontend/src/index.css

```css
@tailwind base;@tailwind components;@tailwind utilities;
body{@apply font-sans bg-slate-50 text-slate-900 transition-colors duration-300;}
.dark body{@apply bg-[#0b1020] text-slate-100;}
@keyframes pop{50%{transform:scale(1.25)}}
.pop{animation:pop .35s}
.fade{animation:fade .35s}
@keyframes fade{from{opacity:0;transform:translateY(6px)}}
@media (prefers-reduced-motion:reduce){*{animation:none!important;transition:none!important}}
```

## frontend/src/main.tsx

```tsx
import ReactDOM from 'react-dom/client';
import App from './App';
import './index.css';
ReactDOM.createRoot(document.getElementById('root')!).render(<App />);
```

## frontend/tailwind.config.js

```js
export default { darkMode: 'class', content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: { extend: { fontFamily: { sans: ['Inter', 'sans-serif'], display: ['Poppins', 'sans-serif'] } } } };
```

## frontend/tsconfig.json

```json
{"compilerOptions":{"target":"ES2020","lib":["ES2020","DOM"],"module":"ESNext","moduleResolution":"bundler","jsx":"react-jsx","strict":true,"skipLibCheck":true,"noEmit":true,"types":["vite/client"]},"include":["src"]}
```

## frontend/vite.config.ts

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
export default defineConfig({ plugins: [react()], server: { proxy: { '/api': 'http://localhost:4000' } } });
```

