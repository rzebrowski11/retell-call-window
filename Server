// retell-call-window — Retell inbound call webhook for SCS
// Tells the inbound agent, before it speaks, whether SCS is open, closed for the night/weekend,
// or closed for a holiday, and hands it the matching greeting.
//
// POST /            Retell inbound webhook (event: call_inbound) -> { call_inbound: { dynamic_variables } }
// GET  /preview     Browser check. Optional ?at=2026-11-26T10:00:00-05:00 to test any moment.
// GET  /health      Railway health check.
//
// Hours and holidays come from the SCS Employee Handbook (Nov 2026):
//   Office hours Mon–Fri 7:00 a.m.–7:00 p.m. Eastern.
//   Holidays: New Year's Day, Memorial Day, Independence Day, Labor Day, Thanksgiving,
//   Day after Thanksgiving, Christmas. Saturday holidays observed Friday, Sunday holidays Monday.
// Holidays are calculated for any year, so this needs no annual update unless the handbook changes.

const http = require('http');

const TZ = 'America/New_York';
const OPEN_HOUR = 7;    // 7:00 a.m.
const CLOSE_HOUR = 19;  // 7:00 p.m.
const COMPANY = 'Solid Commercial Solutions';

// ---------- date helpers (all "civil" dates are plain {y, m, d}, months 1–12) ----------

const pad = n => String(n).padStart(2, '0');
const key = ({ y, m, d }) => `${y}-${pad(m)}-${pad(d)}`;
const dow = ({ y, m, d }) => new Date(Date.UTC(y, m - 1, d)).getUTCDay(); // 0=Sun
const addDays = ({ y, m, d }, n) => {
  const t = new Date(Date.UTC(y, m - 1, d + n));
  return { y: t.getUTCFullYear(), m: t.getUTCMonth() + 1, d: t.getUTCDate() };
};
const nthWeekday = (y, m, weekday, n) => {          // e.g. 4th Thursday of November
  let date = { y, m, d: 1 };
  while (dow(date) !== weekday) date = addDays(date, 1);
  return addDays(date, 7 * (n - 1));
};
const lastWeekday = (y, m, weekday) => {            // e.g. last Monday of May
  let date = addDays({ y: m === 12 ? y + 1 : y, m: m === 12 ? 1 : m + 1, d: 1 }, -1);
  while (dow(date) !== weekday) date = addDays(date, -1);
  return date;
};
const observed = date => {                           // Sat -> Fri, Sun -> Mon
  const w = dow(date);
  return w === 6 ? addDays(date, -1) : w === 0 ? addDays(date, 1) : date;
};

function holidaysForYear(y) {
  const thanksgiving = nthWeekday(y, 11, 4, 4);
  return [
    ["New Year's Day", observed({ y, m: 1, d: 1 })],
    ['Memorial Day', lastWeekday(y, 5, 1)],
    ['Independence Day', observed({ y, m: 7, d: 4 })],
    ['Labor Day', nthWeekday(y, 9, 1, 1)],
    ['Thanksgiving', thanksgiving],
    ['the Thanksgiving holiday', addDays(thanksgiving, 1)],
    ['Christmas', observed({ y, m: 12, d: 25 })],
  ];
}

// Map of 'YYYY-MM-DD' -> holiday name. Includes neighbouring years so an observed date that
// crosses a year boundary (New Year's on a Saturday -> Friday Dec 31) is caught.
function holidayMap(y) {
  const map = {};
  for (const yr of [y - 1, y, y + 1]) for (const [name, date] of holidaysForYear(yr)) map[key(date)] = name;
  return map;
}

function easternNow(at) {
  const parts = Object.fromEntries(
    new Intl.DateTimeFormat('en-US', {
      timeZone: TZ, year: 'numeric', month: 'numeric', day: 'numeric',
      hour: 'numeric', minute: 'numeric', hour12: false, weekday: 'long',
    }).formatToParts(at).map(p => [p.type, p.value])
  );
  return {
    date: { y: +parts.year, m: +parts.month, d: +parts.day },
    hour: +parts.hour % 24,
    minute: +parts.minute,
    weekday: parts.weekday,
  };
}

const isBusinessDay = (date, holidays) => dow(date) >= 1 && dow(date) <= 5 && !holidays[key(date)];
const spokenDate = date =>
  new Date(Date.UTC(date.y, date.m - 1, date.d)).toLocaleDateString('en-US', {
    timeZone: 'UTC', weekday: 'long', month: 'long', day: 'numeric',
  });

// ---------- the decision ----------

function callWindow(at = new Date()) {
  const now = easternNow(at);
  const holidays = holidayMap(now.date.y);
  const holidayName = holidays[key(now.date)] || '';
  const businessDay = isBusinessDay(now.date, holidays);
  const open = businessDay && now.hour >= OPEN_HOUR && now.hour < CLOSE_HOUR;

  // When does the office next open?
  let nextOpen, nextOpenPhrase;
  if (businessDay && now.hour < OPEN_HOUR) {
    nextOpen = now.date;
    nextOpenPhrase = 'when the office opens at 7 this morning';
  } else {
    let d = addDays(now.date, 1);
    while (!isBusinessDay(d, holidays)) d = addDays(d, 1);
    nextOpen = d;
    const tomorrow = key(d) === key(addDays(now.date, 1));
    nextOpenPhrase = tomorrow ? 'first thing tomorrow morning' : `first thing ${spokenDate(d)}`;
  }

  let window, greeting;
  if (open) {
    window = 'business_hours';
    greeting = `Thanks for calling ${COMPANY}. Our team is helping other customers right now, ` +
      `but I'm SCS's automated assistant, and I can take your information so the right person ` +
      `gets back to you as soon as they're off the phone.`;
  } else if (holidayName) {
    window = 'holiday';
    greeting = `Thanks for calling ${COMPANY}. Our office is closed today for ${holidayName}, ` +
      `but I'm SCS's automated assistant. I can take a message, and the team will get back to you ${nextOpenPhrase}.`;
  } else {
    window = 'after_hours';
    greeting = `Thanks for calling ${COMPANY}. Our office is closed right now, ` +
      `but I'm SCS's automated assistant. I can take a message, and the team will get back to you ${nextOpenPhrase}.`;
  }

  // Retell requires every dynamic variable value to be a string.
  return {
    call_window: window,
    greeting,
    holiday_name: holidayName,
    next_business_day: spokenDate(nextOpen),
    callback_expectation: open ? 'as soon as the team is off the phone' : nextOpenPhrase,
    current_time_et: `${now.weekday}, ${spokenDate(now.date).split(', ').slice(1).join(', ')} ${now.date.y}, ` +
      `${((now.hour + 11) % 12) + 1}:${pad(now.minute)} ${now.hour < 12 ? 'a.m.' : 'p.m.'} Eastern`,
  };
}

// ---------- HTTP ----------

function send(res, status, body) {
  res.writeHead(status, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(body, null, 2));
}

const server = http.createServer((req, res) => {
  const url = new URL(req.url, 'http://localhost');

  if (req.method === 'GET' && url.pathname === '/health') return send(res, 200, { ok: true });

  if (req.method === 'GET' && url.pathname === '/preview') {
    const at = url.searchParams.get('at') ? new Date(url.searchParams.get('at')) : new Date();
    if (isNaN(at)) return send(res, 400, { error: 'Bad ?at= value. Example: 2026-11-26T10:00:00-05:00' });
    return send(res, 200, callWindow(at));
  }

  if (req.method === 'POST') {
    let raw = '';
    req.on('data', c => { raw += c; if (raw.length > 1e6) req.destroy(); });
    req.on('end', () => {
      let event = '';
      try { event = JSON.parse(raw || '{}').event || ''; } catch { /* ignore bad JSON */ }
      // Anything that isn't an inbound call gets an empty 200 so Retell does nothing special.
      if (event && event !== 'call_inbound') return send(res, 200, {});
      try {
        const vars = callWindow();
        console.log(new Date().toISOString(), event || '(no event)', vars.call_window);
        return send(res, 200, { call_inbound: { dynamic_variables: vars } });
      } catch (e) {
        // On any error, return 200 with nothing: Retell connects the call and the agent's
        // default dynamic variable values (generic greeting) are used.
        console.error('callWindow failed', e);
        return send(res, 200, {});
      }
    });
    return;
  }

  send(res, 404, { error: 'Not found' });
});

if (require.main === module) {
  const port = process.env.PORT || 3000;
  server.listen(port, () => console.log(`retell-call-window listening on ${port}`));
}

module.exports = { callWindow, holidaysForYear };
