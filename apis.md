# Supsystem API + Rollen + Code (Frontend/Backend)

Diese Datei zeigt fuer `supsystem.html` nicht nur die API-Namen, sondern auch den jeweiligen Code.

## Rollenmodell

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

## API-Index (Supsystem)

| Methode | Endpoint | Effektiv erlaubt |
|---|---|---|
| `GET` | `/api/auth/me` | alle Staff-Rollen mit gueltiger Session |
| `POST` | `/api/auth/logout` | alle |
| `GET` | `/api/admin/requests` | `supporter+` |
| `PUT` | `/api/admin/request/:id` | `supporter+` (mit Ticket-Regeln) |
| `PUT` | `/api/admin/request/:id/assign` | `supporter+` (mit Ticket-Regeln) |
| `DELETE` | `/api/admin/request/:id` | `admin+` |
| `GET` | `/api/admin/self-hosting` | `supporter+` |
| `PATCH` | `/api/admin/self-hosting/:id` | `supporter+` |
| `DELETE` | `/api/admin/self-hosting/:id` | `owner` |
| `GET` | `/api/admin/bots` | `supporter+` |
| `POST` | `/api/admin/create-user` | `admin+` |
| `GET` | `/api/admin/accounts?type=...` | `admin+` |
| `PUT` | `/api/admin/accounts/:id/reset-password?type=...` | `admin+` |
| `PATCH` | `/api/admin/accounts/:id?type=...` | `admin+` (Owner-Ziel nur Owner) |
| `DELETE` | `/api/admin/accounts/:id?type=...` | `admin+` (Owner-Ziel nur Owner) |

## API-Code je Endpoint

### 1) `GET /api/auth/me`

Frontend (`public/js/admin.js`):

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
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 2) `POST /api/auth/logout`

Frontend (`public/js/admin.js`):

```js
async function apiLogout() {
  await fetch('/api/auth/logout', { method: 'POST', credentials: 'include' });
  location.href = '/login.html';
}
```

Backend (`server.js`):

```js
app.post('/api/auth/logout', (req, res) => {
  destroySessionFromCookie(req, res, STAFF_COOKIE_NAME);
  res.json({ success: true });
});
```

### 3) `GET /api/admin/requests`

Frontend (`public/js/admin.js`):

```js
const res = await fetch('/api/admin/requests', { credentials: 'include' });
```

Backend (`server.js`):

```js
app.get('/api/admin/requests', requireAuth, requireRole('supporter'), async (req, res) => {
  try {
    const arr = supportRepo.all();
    res.json({ success: true, requests: arr });
  } catch (err) {
    res.status(500).json({ success: false, error: 'internal_error' });
  }
});
```

### 4) `PUT /api/admin/request/:id` (Antwort speichern)

Frontend (`public/js/admin.js`):

```js
const res = await fetch(`/api/admin/request/${id}`, {
  method: 'PUT',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ reply })
});
```

Backend (`server.js`):

```js
app.put('/api/admin/request/:id', requireAuth, requireRole('supporter'), async (req, res) => {
  const { reply } = req.body;
  const request = supportRepo.getById(req.params.id);

  if (!reply) return res.status(400).json({ success: false, error: 'reply missing' });
  if (!request) return res.status(404).json({ success: false, error: 'not_found' });
  if (request.assignedTo && request.assignedTo !== req.user.username) {
    return res.status(403).json({ success: false, error: 'not_allowed' });
  }
  if (request.reply) {
    return res.status(409).json({ success: false, error: 'already_answered' });
  }

  supportRepo.updateReply({
    id: req.params.id,
    reply,
    repliedBy: req.user.username,
    repliedAt: new Date().toISOString(),
    assignedTo: request.assignedTo || req.user.username,
    assignedAt: request.assignedAt || new Date().toISOString()
  });

  res.json({ success: true });
});
```

### 5) `PUT /api/admin/request/:id/assign` (Claim)

Frontend (`public/js/admin.js`):

```js
const res = await fetch(`/api/admin/request/${id}/assign`, {
  method: 'PUT',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({})
});
```

Backend (`server.js`):

```js
app.put('/api/admin/request/:id/assign', requireAuth, requireRole('supporter'), async (req, res) => {
  const item = supportRepo.getById(req.params.id);
  if (!item) return res.status(404).json({ success: false, error: 'not_found' });

  if (item.assignedTo && item.assignedTo !== req.user.username) {
    return res.status(409).json({ success: false, error: 'already_assigned', assignedTo: item.assignedTo });
  }

  if (!item.assignedTo) {
    supportRepo.assign({ id: req.params.id, username: req.user.username, assignedAt: new Date().toISOString() });
  }

  res.json({ success: true, assignedTo: item.assignedTo || req.user.username });
});
```

### 6) `DELETE /api/admin/request/:id`

Frontend (`public/js/admin.js`):

```js
const res = await fetch(`/api/admin/request/${id}`, {
  method: 'DELETE',
  credentials: 'include'
});
```

Backend (`server.js`):

```js
app.delete('/api/admin/request/:id', requireAuth, requireRole('admin'), async (req, res) => {
  const deleted = supportRepo.deleteById(req.params.id);
  if (!deleted) return res.status(404).json({ success: false, error: 'not_found' });
  res.json({ success: true });
});
```

### 7) `GET /api/admin/self-hosting`

Frontend (`public/js/admin.js`):

```js
const res = await fetch('/api/admin/self-hosting', { credentials: 'include' });
```

Backend (`server.js`):

```js
app.get('/api/admin/self-hosting', requireAuth, requireRole('supporter'), async (req, res) => {
  const requests = selfHostRepo.listAll();
  res.json({ success: true, requests });
});
```

### 8) `PATCH /api/admin/self-hosting/:id`

Frontend (`public/js/admin.js`):

```js
const res = await fetch(`/api/admin/self-hosting/${id}`, {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({ status: select.value, adminNotes: notes.value })
});
```

Backend (`server.js`):

```js
app.patch('/api/admin/self-hosting/:id', requireAuth, requireRole('supporter'), async (req, res) => {
  const status = req.body.status ? String(req.body.status).toLowerCase() : undefined;
  const notes = req.body.adminNotes ? String(req.body.adminNotes).trim() : undefined;

  if (!status && !notes) {
    return res.status(400).json({ success: false, error: 'missing_fields' });
  }

  const updated = selfHostRepo.updateStatus({ id: req.params.id, status, adminNotes: notes });
  if (!updated) return res.status(404).json({ success: false, error: 'not_found' });

  res.json({ success: true });
});
```

### 9) `DELETE /api/admin/self-hosting/:id`

Frontend (`public/js/admin.js`):

```js
const res = await fetch(`/api/admin/self-hosting/${row.dataset.id}`, {
  method: 'DELETE',
  credentials: 'include'
});
```

Backend (`server.js`):

```js
app.delete('/api/admin/self-hosting/:id', requireAuth, requireRole('owner'), async (req, res) => {
  const deleted = selfHostRepo.deleteById(req.params.id);
  if (!deleted) return res.status(404).json({ success: false, error: 'not_found' });
  res.json({ success: true });
});
```

### 10) `GET /api/admin/bots`

Frontend (`public/js/admin.js`):

```js
const res = await fetch('/api/admin/bots', { credentials: 'include' });
```

Backend (`server.js`):

```js
app.get('/api/admin/bots', requireAuth, requireRole('supporter'), async (req, res) => {
  const [snapshot, metaMap] = await Promise.all([fetchStatusSnapshot(), loadSelfhostersMeta()]);
  const bots = buildBotStatusList(snapshot, metaMap);
  res.json({
    success: true,
    bots,
    source: { host: snapshot?.host || null, ts: snapshot?.ts || null }
  });
});
```

### 11) `POST /api/admin/create-user`

Frontend (`public/js/admin.js`):

```js
const res = await fetch('/api/admin/create-user', {
  method: 'POST',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ username, password, role })
});
```

Backend (`server.js`):

```js
app.post('/api/admin/create-user', requireAuth, requireRole('admin'), async (req, res) => {
  const username = sanitizeInput(req.body.username);
  const password = String(req.body.password || '');
  const role = req.body.role === 'admin' ? 'admin' : 'supporter';

  if (!username || !password)
    return res.status(400).json({ success: false, error: 'missing_fields' });

  const existing = staffRepo.findByUsername(username);
  if (existing)
    return res.status(409).json({ success: false, error: 'username_exists' });

  staffRepo.insert({
    id: generateId(),
    username,
    passwordHash: hashPassword(password),
    role,
    createdAt: new Date().toISOString(),
    createdBy: req.user.username
  });

  res.status(201).json({ success: true });
});
```

### 12) `GET /api/admin/accounts?type=...`

Frontend (`public/js/admin-accounts.js`):

```js
const res = await fetch(`/api/admin/accounts?type=${encodeURIComponent(type)}`, {
  credentials: 'include'
});
```

Backend (`server.js`):

```js
app.get('/api/admin/accounts', requireAuth, requireRole('admin'), async (req, res) => {
  const type = String(req.query.type || 'users');
  if (type === 'team') {
    const rows = staffRepo.all();
    return res.json({ success: true, accounts: rows.map(r => ({ id: r.id, username: r.username, role: r.role, createdAt: r.createdAt, createdBy: r.createdBy })) });
  }

  const rows = publicUsersRepo.all();
  return res.json({ success: true, accounts: rows.map(r => ({ id: r.id, username: r.username, createdAt: r.createdAt, lastLoginAt: r.lastLoginAt })) });
});
```

### 13) `PUT /api/admin/accounts/:id/reset-password?type=...`

Frontend (`public/js/admin-accounts.js`):

```js
const res = await fetch(`/api/admin/accounts/${encodeURIComponent(item.id)}/reset-password?type=${encodeURIComponent(type)}`, {
  method: 'PUT',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ password: pw })
});
```

Backend (`server.js`):

```js
app.put('/api/admin/accounts/:id/reset-password', requireAuth, requireRole('admin'), async (req, res) => {
  const id = req.params.id;
  const type = String(req.query.type || req.body.type || 'users');
  const password = String(req.body.password || '');
  if (!password) return res.status(400).json({ success: false, error: 'missing_password' });

  // team oder users, jeweils Hash-Update
  res.json({ success: true });
});
```

### 14) `PATCH /api/admin/accounts/:id?type=...`

Frontend (`public/js/admin-accounts.js`):

```js
const res = await fetch(`/api/admin/accounts/${encodeURIComponent(item.id)}?type=${encodeURIComponent(type)}`, {
  method: 'PATCH',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ role: newRole })
});
```

Backend (`server.js`):

```js
app.patch('/api/admin/accounts/:id', requireAuth, requireRole('admin'), async (req, res) => {
  const type = String(req.query.type || req.body.type || 'users');
  if (type === 'team') {
    const role = String(req.body.role || '').trim().toLowerCase();
    const allowed = ['owner', 'admin', 'dev', 'supporter', 'hoster'];
    if (!allowed.includes(role)) return res.status(400).json({ success: false, error: 'invalid_role' });

    const target = staffRepo.findById(req.params.id);
    if (target.role === 'owner' && req.user.role !== 'owner') {
      return res.status(403).json({ success: false, error: 'forbidden' });
    }

    return res.json({ success: true });
  }

  return res.status(400).json({ success: false, error: 'not_supported' });
});
```

### 15) `DELETE /api/admin/accounts/:id?type=...`

Frontend (`public/js/admin-accounts.js`):

```js
const res = await fetch(`/api/admin/accounts/${encodeURIComponent(item.id)}?type=${encodeURIComponent(type)}`, {
  method: 'DELETE',
  credentials: 'include'
});
```

Backend (`server.js`):

```js
app.delete('/api/admin/accounts/:id', requireAuth, requireRole('admin'), async (req, res) => {
  const type = String(req.query.type || req.body.type || 'users');

  if (type === 'team') {
    const target = staffRepo.findById(req.params.id);
    if (target.role === 'owner' && req.user.role !== 'owner') {
      return res.status(403).json({ success: false, error: 'forbidden' });
    }
    return res.json({ success: true });
  }

  return res.json({ success: true });
});
```

## Kurzfazit je Rolle fuer supsystem

- `owner`: volle Rechte im supsystem.
- `admin`: fast alles, aber kein Self-Hosting-Delete.
- `dev`: backendseitig bei `requireRole('admin')` erlaubt, UI blendet einige Elemente aus.
- `supporter`: Tickets/Selhost/Bots lesen und bearbeiten, aber kein Ticket-Delete, kein Accounts/User-Management.
- `hoster` und `user`: bei supsystem-APIs meist `403`.
