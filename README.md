# Friendship Almanac — Web frontend

A mobile-first, almanac-style view of the Friend Suggester. Replaces the Google Sheets Dashboard tab with a slicker, on-the-go interface.

## View locally

Just open `index.html` in a browser. Currently shows mock data (the top 15 friends from your tab as of mid-May 2026).

## Wire up real data

The frontend is ready to fetch a JSON payload from an Apps Script Web App.

### 1. Add `doGet()` to your Apps Script (Code.gs)

Paste this at the bottom of the file. It returns the same data the Dashboard tab uses, in JSON form.

```javascript
function doGet(e) {
  const token = e && e.parameter && e.parameter.t;
  // Optional: gate by a secret token in the URL.
  // if (token !== PropertiesService.getScriptProperties().getProperty('SECRET')) {
  //   return ContentService.createTextOutput('forbidden').setMimeType(ContentService.MimeType.TEXT);
  // }

  const friends = readTab(TAB_FRIENDS);
  const meets   = readTab(TAB_MEETS);
  const groups  = readTab(TAB_GROUPS);

  const groupMembers = {};
  groups.forEach(g => {
    if (!g.group_name) return;
    groupMembers[String(g.group_name).toLowerCase()] =
      String(g.members || '').split(',').map(s => s.trim().toLowerCase()).filter(Boolean);
  });

  const today = new Date();
  const stats = {};
  meets.forEach(m => {
    if (!m.date) return;
    const d = (m.date instanceof Date) ? m.date : new Date(m.date);
    if (isNaN(d)) return;
    let attendees = [];
    if (m.group_name) attendees = groupMembers[String(m.group_name).toLowerCase()] || [];
    else attendees = String(m.friend_names || '').split(',').map(s => s.trim().toLowerCase()).filter(Boolean);
    attendees.forEach(name => {
      if (!stats[name]) stats[name] = { total: 0, lastDate: null, lastLocation: '', lastNotes: '', them: 0, me: 0 };
      stats[name].total++;
      if (!stats[name].lastDate || d > stats[name].lastDate) {
        stats[name].lastDate = d;
        stats[name].lastLocation = m.location || '';
        stats[name].lastNotes = m.notes || '';
      }
      const i = String(m.initiator || '').toLowerCase();
      if (i === 'me') stats[name].me++;
      else if (i === 'them') stats[name].them++;
    });
  });

  const payload = friends.map(f => {
    const nameKey = String(f.name || '').toLowerCase();
    const s = stats[nameKey] || { total: 0, lastDate: null, lastLocation: '', lastNotes: '', them: 0, me: 0 };
    return {
      name: f.name,
      meets: s.total,
      avgWeeks: f['average meet up frequency'] || '',
      lastMeet: s.lastDate ? Utilities.formatDate(s.lastDate, 'Asia/Singapore', 'yyyy-MM-dd') : '',
      lastLocation: s.lastLocation,
      lastNotes: s.lastNotes,
      mrt_home: f.mrt_home || '',
      mrt_work: f.mrt_work || '',
      home_area: f.home_area || '',
      work_area: f.work_area || '',
      target: Number(f.target_cadence_weeks) || 52,
      status: f.status || 'Active',
      them: s.them,
      me: s.me,
    };
  });

  return ContentService
    .createTextOutput(JSON.stringify({ generatedAt: new Date().toISOString(), friends: payload }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### 2. Deploy as Web App

In Apps Script editor:
- **Deploy → New deployment**
- Type: **Web app**
- Execute as: **Me (colin.ng89@gmail.com)**
- Who has access: **Anyone** (data is gated by the obscure URL — fine for single-user. Add the SECRET token check above for stricter privacy.)
- Click **Deploy**, copy the Web app URL.

### 3. Wire the frontend

In `index.html`, find `loadFriends()` near the bottom of the script. Replace the body with:

```javascript
async function loadFriends() {
  const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/AKfyc.../exec';
  const res = await fetch(APPS_SCRIPT_URL);
  const data = await res.json();
  return data.friends;
}
```

Also remove the `today` override (`new Date('2026-05-16')`) — let it default to actual today.

## Deploy to GitHub Pages

```bash
cd "side-projects/friend-suggester/web"
git init
gh auth switch --user colin1201
gh repo create colin1201/friendship-almanac --public --source=. --remote=origin
git add . && git commit -m "Initial commit"
git push -u origin master
# In repo Settings → Pages → deploy from main branch / root
```

After Pages publishes, you'll get a URL like `https://colin1201.github.io/friendship-almanac/`. Add to home screen on your phone for app-like access.

## Design notes

- **Aesthetic**: editorial almanac. Fraunces (italic) for display, IBM Plex Sans for body, IBM Plex Mono for marginalia.
- **Palette**: warm cream paper, ink, terracotta + umber accents.
- **Algorithm mirrors Code.gs**: overdue = `weeks_since_last / target_cadence_weeks > 1.5`. Active + 3+ meets only.
- **No notifications, no auth flow** — single-user, secret URL.
