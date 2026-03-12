# Supsystem: HTML <-> Buttons <-> APIs <-> Rollen (inkl. API-Code)

Diese Doku erklaert **ausfuehrlich** fuer `supsystem.html`:

1. welches UI-Element welchen Request ausloest,
2. welche Rolle was im Frontend sieht,
3. welche Rolle es backendseitig wirklich darf,
4. und den konkreten API-Code (Route + relevante Logik).

## 1) Welche HTML-Datei ist gemeint?

- Seite: `public/supsystem.html`
- Eingebundene Skripte:
  - `public/js/admin.js`
  - `public/js/admin-accounts.js`

Die meisten API-Calls kommen aus diesen beiden JS-Dateien.

## 2) Rollenmodell (Backend)

```js
const ROLE_POWER = {
  owner: 4,
  admin: 3,
  dev: 3,
  supporter: 2,
  hoster: 1,
  user: 0
};
```

```js
function requireRole(role) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ success: false, error: 'unauthorized' });
    }
    const requiredPower = getRolePower(role);
    const userPower = getRolePower(req.user.role);
    if (userPower < requiredPower) {
      return res.status(403).json({ success: false, error: 'forbidden' });
    }
    next();
  };
}
```

Bedeutung:

- `requireRole('supporter')` erlaubt: `owner/admin/dev/supporter`
- `requireRole('admin')` erlaubt: `owner/admin/dev`
- `requireRole('owner')` erlaubt: nur `owner`

## 3) Button-/UI-Mapping aus `supsystem.html`

## 3.1 Top-Actions im Header

| HTML-Element | Zweck | Frontend-Verhalten | API/Backend |
|---|---|---|---|
| `Alles Wichtige` (Link) | Navigation | statischer Link auf `alles-wichtige.html` | keine API von `supsystem` |
| `#linkedSessionsBtn` | Navigation zu Sessions | sichtbar fuer `owner/admin/supporter`, Klick auf `/admin-bots.html` | keine direkte API im Klick; Zielseite nutzt eigene APIs |
| `#pm2DashboardBtn` | Navigation PM2 | sichtbar nur fuer `owner/admin` | keine direkte API im Klick; Zielseite nutzt eigene APIs |
| `#uploadPortalBtn` | Navigation Upload | sichtbar nur fuer `owner/admin` | keine direkte API im Klick; Zielseite nutzt eigene APIs |
| `#logoutBtn` | Logout | ruft `apiLogout()` | `POST /api/auth/logout` |

Relevanter Frontend-Code (`public/js/admin.js`):

```js
const uploadBtn = document.getElementById('uploadPortalBtn');
const linkedSessionsBtn = document.getElementById('linkedSessionsBtn');
const pm2DashboardBtn = document.getElementById('pm2DashboardBtn');

const role = (user.role || '').toLowerCase();
const canUpload = role === 'owner' || role === 'admin';
const canAccessLinkedSessions = role === 'owner' || role === 'admin' || role === 'supporter';
const canAccessPm2 = role === 'owner' || role === 'admin';
```

## 3.2 Support-Tabelle (dynamische Buttons)

Diese Buttons entstehen pro Ticket in `renderRequests()`:

- `Speichern` -> `PUT /api/admin/request/:id`
- `Uebernehmen` -> `PUT /api/admin/request/:id/assign`
- `Loeschen` -> `DELETE /api/admin/request/:id`

Wichtig:

- Frontend zeigt `Loeschen` immer.
- Backend blockt `supporter` dort mit `403`, weil delete `admin+` braucht.

## 3.3 Self-Hosting-Panel (dynamische Buttons)

- `Speichern` -> `PATCH /api/admin/self-hosting/:id`
- `Loeschen` -> nur im UI fuer `owner` sichtbar, ruft `DELETE /api/admin/self-hosting/:id`

## 3.4 Accounts-Panel (`admin-accounts.js`)

- fuer `supporter` komplett versteckt (`panel.classList.add('hidden')`)
- fuer `owner/admin` sicht- und nutzbar
- `dev` ist Backend `admin+`, aber im Frontend nicht als `canManage()` behandelt

Aktionen im Panel:

- `Passwort zuruecksetzen` -> `PUT /api/admin/accounts/:id/reset-password`
- `Rolle aendern` -> `PATCH /api/admin/accounts/:id`
- `Loeschen` -> `DELETE /api/admin/accounts/:id`

## 4) API-Mapping: Welche API gehoert zu welcher HTML/JS-Stelle?

| API | Aufruf-Ort in Frontend | Ausgeloest durch |
|---|---|---|
| `GET /api/auth/me` | `admin.js` -> `apiMe()` | Seitenstart (`DOMContentLoaded`) |
| `POST /api/auth/logout` | `admin.js` -> `apiLogout()` | Klick `#logoutBtn` |
| `GET /api/admin/requests` | `admin.js` -> `loadRequests()` | Initiales Laden/Reload |
| `PUT /api/admin/request/:id` | `admin.js` -> `attachRowActions()` | Button `Speichern` pro Ticket |
| `PUT /api/admin/request/:id/assign` | `admin.js` -> `attachRowActions()` | Button `Uebernehmen` pro Ticket |
| `DELETE /api/admin/request/:id` | `admin.js` -> `attachRowActions()` | Button `Loeschen` pro Ticket |
| `GET /api/admin/self-hosting` | `admin.js` -> `loadSelfHostAdmin()` | Initiales Laden/Reload |
| `PATCH /api/admin/self-hosting/:id` | `admin.js` -> `attachSelfHostActions()` | Button `Speichern` Self-Hosting |
| `DELETE /api/admin/self-hosting/:id` | `admin.js` -> `attachSelfHostActions()` | Button `Loeschen` Self-Hosting (owner UI) |
| `GET /api/admin/bots` | `admin.js` -> `loadBotAdmin()` | Dashboard-Bot-Widget laden |
| `POST /api/admin/create-user` | `admin.js` -> `initAdminUserForm()` | Submit `#createUserForm` |
| `GET /api/admin/accounts` | `admin-accounts.js` -> `fetchAccounts()` | Accounts-Panel laden / Filterwechsel |
| `PUT /api/admin/accounts/:id/reset-password` | `admin-accounts.js` | Button `Passwort zuruecksetzen` |
| `PATCH /api/admin/accounts/:id` | `admin-accounts.js` | Button `Rolle aendern` |
| `DELETE /api/admin/accounts/:id` | `admin-accounts.js` | Button `Loeschen` |

## 5) Rollen-Matrix: Was kann jede Rolle in supsystem?

| Funktion | owner | admin | dev | supporter | hoster/user |
|---|---|---|---|---|---|
| Seite laden (`/api/auth/me`) | Ja | Ja | Ja | Ja | Ja (falls Staff-Account) |
| Ticketliste sehen | Ja | Ja | Ja | Ja | Nein |
| Ticket uebernehmen/antworten | Ja | Ja | Ja | Ja | Nein |
| Ticket loeschen | Ja | Ja | Ja | Nein | Nein |
| Self-Hosting sehen/bearbeiten | Ja | Ja | Ja | Ja | Nein |
| Self-Hosting loeschen | Ja | Nein | Nein | Nein | Nein |
| Bot-Uebersicht im supsystem laden | Ja | Ja | Ja | Ja | Nein |
| User erstellen | Ja | Ja | Ja (Backend), UI meist versteckt | Nein | Nein |
| Accounts lesen/aendern | Ja | Ja | Ja (Backend), UI meist versteckt | Nein (UI+API) | Nein |
| Upload-Link sichtbar | Ja | Ja | Nein (UI) | Nein (UI) | Nein |
| PM2-Link sichtbar | Ja | Ja | Nein (UI) | Nein (UI) | Nein |

Hinweis zu `dev`: API-seitig ist `dev` bei `requireRole('admin')` erlaubt. UI-Checks in `admin.js`/`admin-accounts.js` richten sich aber haeufig explizit nach `owner/admin`.

## 6) API-Code (Frontend + Backend)

Die folgenden Bloecke sind die relevanten Endpunkte fuer `supsystem.html`.

### 6.1 `GET /api/auth/me`

Frontend (`admin.js`):

```js
async function apiMe() {
  const res = await fetch('/api/auth/me', { credentials: 'include' });
  if (!res.ok) return null;
  const data = await res.json();
  return data.success ? data.user : null;
}
```

Backend (`server.js`):

```js
app.get('/api/auth/me', async (req, res) => {
  try {
    const user = await getUserFromRequest(req);
    if (!user) return res.status(401).json({ success: false, error: 'unauthorized' });
    res.json({ success: true, user });
  } catch (err) {
    await logAction('ERROR', '/api/auth/me Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.2 `POST /api/auth/logout`

Frontend:

```js
async function apiLogout() {
  await fetch('/api/auth/logout', { method: 'POST', credentials: 'include' });
  location.href = '/login.html';
}
```

Backend:

```js
app.post('/api/auth/logout', (req, res) => {
  destroySessionFromCookie(req, res, STAFF_COOKIE_NAME);
  res.json({ success: true });
});
```

### 6.3 `GET /api/admin/requests`

Frontend:

```js
const res = await fetch('/api/admin/requests', { credentials: 'include' });
```

Backend:

```js
app.get('/api/admin/requests', requireAuth, requireRole('supporter'), async (req, res) => {
  try {
    const arr = supportRepo.all();
    await logAction('INFO', 'Supporter ruft Anfragen ab', { by: req.user.username });
    res.json({ success: true, requests: arr });
  } catch (err) {
    await logAction('ERROR', '/api/admin/requests Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.4 `PUT /api/admin/request/:id`

Frontend:

```js
const res = await fetch(`/api/admin/request/${id}`, {
  method: 'PUT',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ reply })
});
```

Backend:

```js
app.put('/api/admin/request/:id', requireAuth, requireRole('supporter'), async (req, res) => {
  try {
    const { reply } = req.body;
    if (!reply) return res.status(400).json({ success: false, error: 'reply missing' });

    const id = req.params.id;
    const request = supportRepo.getById(id);
    if (!request) return res.status(404).json({ success: false, error: 'not_found' });

    const now = new Date().toISOString();
    const assignedTo = request.assignedTo || req.user.username;
    const assignedAt = request.assignedAt || now;

    if (request.assignedTo && request.assignedTo !== req.user.username) {
      return res.status(403).json({ success: false, error: 'not_allowed' });
    }

    if (request.reply) {
      return res.status(409).json({ success: false, error: 'already_answered' });
    }

    supportRepo.updateReply({
      id,
      reply,
      repliedBy: req.user.username,
      repliedAt: now,
      assignedTo,
      assignedAt
    });
    await logAction('INFO', 'Antwort gespeichert', { id, by: req.user.username });
    res.json({ success: true, message: `Antwort fuer ${id} gespeichert` });
  } catch (err) {
    await logAction('ERROR', '/api/admin/request/:id PUT Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.5 `PUT /api/admin/request/:id/assign`

Frontend:

```js
const res = await fetch(`/api/admin/request/${id}/assign`, {
  method: 'PUT',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({})
});
```

Backend:

```js
app.put('/api/admin/request/:id/assign', requireAuth, requireRole('supporter'), async (req, res) => {
  try {
    const id = req.params.id;
    const item = supportRepo.getById(id);

    if (!item) {
      return res.status(404).json({ success: false, error: 'not_found' });
    }

    if (item.assignedTo && item.assignedTo !== req.user.username) {
      return res.status(409).json({
        success: false,
        error: 'already_assigned',
        assignedTo: item.assignedTo
      });
    }

    if (!item.assignedTo) {
      const assignedAt = new Date().toISOString();
      supportRepo.assign({ id, username: req.user.username, assignedAt });
      await logAction('INFO', 'Anfrage zugewiesen', { id, to: req.user.username });
      item.assignedTo = req.user.username;
      item.assignedAt = assignedAt;
    }

    res.json({ success: true, assignedTo: item.assignedTo });
  } catch (err) {
    await logAction('ERROR', '/api/admin/request/:id/assign Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.6 `DELETE /api/admin/request/:id`

Frontend:

```js
const res = await fetch(`/api/admin/request/${id}`, {
  method: 'DELETE',
  credentials: 'include'
});
```

Backend:

```js
app.delete('/api/admin/request/:id', requireAuth, requireRole('admin'), async (req, res) => {
  try {
    const id = req.params.id;
    const deleted = supportRepo.deleteById(id);
    if (!deleted) {
      return res.status(404).json({ success: false, error: 'not_found' });
    }

    await logAction('INFO', 'Anfrage geloescht', { id, by: req.user.username });
    res.json({ success: true, message: `Anfrage ${id} geloescht` });
  } catch (err) {
    await logAction('ERROR', '/api/admin/request/:id DELETE Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.7 `GET /api/admin/self-hosting`

Frontend:

```js
const res = await fetch('/api/admin/self-hosting', { credentials: 'include' });
```

Backend:

```js
app.get('/api/admin/self-hosting', requireAuth, requireRole('supporter'), async (req, res) => {
  try {
    const requests = selfHostRepo.listAll();
    res.json({ success: true, requests });
  } catch (err) {
    await logAction('ERROR', '/api/admin/self-hosting GET Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.8 `PATCH /api/admin/self-hosting/:id`

Frontend:

```js
const res = await fetch(`/api/admin/self-hosting/${id}`, {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({ status: select.value, adminNotes: notes ? notes.value : undefined })
});
```

Backend:

```js
app.patch('/api/admin/self-hosting/:id', requireAuth, requireRole('supporter'), async (req, res) => {
  try {
    const id = req.params.id;
    const status = req.body.status ? String(req.body.status).toLowerCase() : undefined;
    const notes = req.body.adminNotes ? String(req.body.adminNotes).trim() : undefined;

    if (!status && !notes) {
      return res.status(400).json({ success: false, error: 'missing_fields' });
    }

    if (status && !ALLOWED_SELFHOST_STATES.includes(status)) {
      return res.status(400).json({ success: false, error: 'invalid_status' });
    }

    const updated = selfHostRepo.updateStatus({ id, status, adminNotes: notes });
    if (!updated) {
      return res.status(404).json({ success: false, error: 'not_found' });
    }

    await logAction('INFO', 'Self-Hosting Status aktualisiert', {
      id,
      status,
      by: req.user.username
    });

    res.json({ success: true });
  } catch (err) {
    await logAction('ERROR', '/api/admin/self-hosting/:id PATCH Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.9 `DELETE /api/admin/self-hosting/:id`

Frontend:

```js
const res = await fetch(`/api/admin/self-hosting/${row.dataset.id}`, {
  method: 'DELETE',
  credentials: 'include'
});
```

Backend:

```js
app.delete('/api/admin/self-hosting/:id', requireAuth, requireRole('owner'), async (req, res) => {
  try {
    const id = req.params.id;
    const deleted = selfHostRepo.deleteById(id);
    if (!deleted) {
      return res.status(404).json({ success: false, error: 'not_found' });
    }
    await logAction('INFO', 'Self-Hosting Anfrage geloescht', { id, by: req.user.username });
    res.json({ success: true });
  } catch (err) {
    await logAction('ERROR', '/api/admin/self-hosting/:id DELETE Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.10 `GET /api/admin/bots`

Frontend:

```js
const res = await fetch('/api/admin/bots', { credentials: 'include' });
```

Backend:

```js
app.get('/api/admin/bots', requireAuth, requireRole('supporter'), async (req, res) => {
  try {
    const [snapshot, metaMap] = await Promise.all([
      fetchStatusSnapshot(),
      loadSelfhostersMeta()
    ]);
    const allBots = buildBotStatusList(snapshot, metaMap);
    const bots = allBots;
    res.json({
      success: true,
      bots,
      source: { host: snapshot?.host || null, ts: snapshot?.ts || null }
    });
  } catch (err) {
    await logAction('ERROR', '/api/admin/bots GET Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.11 `POST /api/admin/create-user`

Frontend:

```js
const res = await fetch('/api/admin/create-user', {
  method: 'POST',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ username, password, role })
});
```

Backend:

```js
app.post('/api/admin/create-user', requireAuth, requireRole('admin'), async (req, res) => {
  try {
    const username = sanitizeInput(req.body.username);
    const password = String(req.body.password || '');
    const role = req.body.role === 'admin' ? 'admin' : 'supporter';

    if (!username || !password)
      return res.status(400).json({ success: false, error: 'missing_fields' });

    const existing = staffRepo.findByUsername(username);
    if (existing)
      return res.status(409).json({ success: false, error: 'username_exists' });

    const user = {
      id: generateId(),
      username,
      passwordHash: hashPassword(password),
      role,
      createdAt: new Date().toISOString(),
      createdBy: req.user.username
    };
    staffRepo.insert(user);

    await logAction('INFO', 'Neuer User angelegt', { username, role, by: req.user.username });
    res.status(201).json({
      success: true,
      user: { id: user.id, username: user.username, role: user.role }
    });
  } catch (err) {
    await logAction('ERROR', '/api/admin/create-user Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.12 `GET /api/admin/accounts`

Frontend (`admin-accounts.js`):

```js
const res = await fetch(`/api/admin/accounts?type=${encodeURIComponent(type)}`, { credentials: 'include' });
```

Backend:

```js
app.get('/api/admin/accounts', requireAuth, requireRole('admin'), async (req, res) => {
  try {
    const type = String(req.query.type || 'users');
    if (type === 'team') {
      const rows = staffRepo.all();
      const safe = rows.map(r => ({ id: r.id, username: r.username, role: r.role, createdAt: r.createdAt, createdBy: r.createdBy }));
      return res.json({ success: true, accounts: safe });
    }

    const rows = publicUsersRepo.all();
    const safe = rows.map(r => ({ id: r.id, username: r.username, createdAt: r.createdAt, lastLoginAt: r.lastLoginAt }));
    return res.json({ success: true, accounts: safe });
  } catch (err) {
    await logAction('ERROR', '/api/admin/accounts Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.13 `PUT /api/admin/accounts/:id/reset-password`

Frontend:

```js
const res = await fetch(`/api/admin/accounts/${encodeURIComponent(item.id)}/reset-password?type=${encodeURIComponent(type)}`, {
  method: 'PUT',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ password: pw })
});
```

Backend:

```js
app.put('/api/admin/accounts/:id/reset-password', requireAuth, requireRole('admin'), async (req, res) => {
  try {
    const id = req.params.id;
    const type = String(req.query.type || req.body.type || 'users');
    const password = String(req.body.password || '');
    if (!password) return res.status(400).json({ success: false, error: 'missing_password' });

    if (type === 'team') {
      const target = staffRepo.findById(id);
      if (!target) return res.status(404).json({ success: false, error: 'not_found' });
      const newHash = hashPassword(password);
      const ok = staffRepo.updatePassword(id, newHash);
      if (!ok) return res.status(500).json({ success: false, error: 'update_failed' });
      await logAction('INFO', 'Staff Passwort zurueckgesetzt', { id, by: req.user.username });
      return res.json({ success: true });
    }

    const target = publicUsersRepo.findById(id);
    if (!target) return res.status(404).json({ success: false, error: 'not_found' });
    const newHash = hashPassword(password);
    const ok = publicUsersRepo.updatePassword(id, newHash);
    if (!ok) return res.status(500).json({ success: false, error: 'update_failed' });
    await logAction('INFO', 'Public Passwort zurueckgesetzt', { id, by: req.user.username });
    return res.json({ success: true });
  } catch (err) {
    await logAction('ERROR', '/api/admin/accounts/:id/reset-password Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.14 `PATCH /api/admin/accounts/:id`

Frontend:

```js
const res = await fetch(`/api/admin/accounts/${encodeURIComponent(item.id)}?type=${encodeURIComponent(type)}`, {
  method: 'PATCH',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ role: newRole })
});
```

Backend:

```js
app.patch('/api/admin/accounts/:id', requireAuth, requireRole('admin'), async (req, res) => {
  try {
    const id = req.params.id;
    const type = String(req.query.type || req.body.type || 'users');

    if (type === 'team') {
      const role = String(req.body.role || '').trim().toLowerCase();
      if (!role) return res.status(400).json({ success: false, error: 'missing_role' });
      const allowed = ['owner', 'admin', 'dev', 'supporter', 'hoster'];
      if (!allowed.includes(role)) return res.status(400).json({ success: false, error: 'invalid_role' });

      const target = staffRepo.findById(id);
      if (!target) return res.status(404).json({ success: false, error: 'not_found' });

      if (target.role === 'owner' && req.user.role !== 'owner') {
        return res.status(403).json({ success: false, error: 'forbidden' });
      }

      const ok = staffRepo.updateRole(id, role);
      if (!ok) return res.status(500).json({ success: false, error: 'update_failed' });
      await logAction('INFO', 'Staff Rolle geaendert', { id, role, by: req.user.username });
      return res.json({ success: true });
    }

    return res.status(400).json({ success: false, error: 'not_supported' });
  } catch (err) {
    await logAction('ERROR', '/api/admin/accounts/:id PATCH Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 6.15 `DELETE /api/admin/accounts/:id`

Frontend:

```js
const res = await fetch(`/api/admin/accounts/${encodeURIComponent(item.id)}?type=${encodeURIComponent(type)}`, {
  method: 'DELETE',
  credentials: 'include'
});
```

Backend:

```js
app.delete('/api/admin/accounts/:id', requireAuth, requireRole('admin'), async (req, res) => {
  try {
    const id = req.params.id;
    const type = String(req.query.type || req.body.type || 'users');

    if (type === 'team') {
      const target = staffRepo.findById(id);
      if (!target) return res.status(404).json({ success: false, error: 'not_found' });

      if (target.role === 'owner' && req.user.role !== 'owner') {
        return res.status(403).json({ success: false, error: 'forbidden' });
      }

      const ok = staffRepo.deleteById(id);
      if (!ok) return res.status(500).json({ success: false, error: 'delete_failed' });
      await logAction('INFO', 'Staff User geloescht', { id, by: req.user.username });
      return res.json({ success: true });
    }

    const target = publicUsersRepo.findById(id);
    if (!target) return res.status(404).json({ success: false, error: 'not_found' });
    const ok = publicUsersRepo.deleteById(id);
    if (!ok) return res.status(500).json({ success: false, error: 'delete_failed' });
    await logAction('INFO', 'Public User geloescht', { id, by: req.user.username });
    return res.json({ success: true });
  } catch (err) {
    await logAction('ERROR', '/api/admin/accounts/:id DELETE Fehler', { error: err.message });
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

## 7) Wichtiges Fazit fuer deine App-Integration

- Verlasse dich bei Berechtigung immer auf Backend-Middleware + Handlerlogik, nicht nur auf UI-Sichtbarkeit.
- In `supsystem` gibt es bewusst Frontend/Backend-Differenzen (z. B. `dev` oder Ticket-Delete-Button).
- Wenn du in der App Rollen sauber spiegeln willst, nutze zuerst `GET /api/auth/me`, danach Role-Gates pro Feature wie in dieser Datei.
