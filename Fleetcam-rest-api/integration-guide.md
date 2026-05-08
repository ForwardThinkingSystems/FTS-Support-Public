# FleetCam REST API — Integration Guide

For developers integrating with the FleetCam REST API. Covers authentication, the two account types (single-company customers and resellers), common workflows, and copy-paste code samples in **Python, C#, JavaScript (Node 18+ / browser), and React**.

> **Looking for the full reference?** The OpenAPI spec is at `https://<host>/fc-rest/v1/v3/api-docs` (JSON) and `/swagger-ui.html` (interactive). This guide explains the concepts and patterns; the spec is the authoritative endpoint catalog.

> **Code samples are collapsible — click a language heading below to expand it.** Default state is collapsed so you can pick the stack you care about without scrolling past the others.

---

## 1. Quick start

<details>
<summary><b>Python</b></summary>

```python
import requests

BASE = "https://<your-fleetcam-host>/fc-rest/v1"

# 1. Get a bearer token
r = requests.post(f"{BASE}/authentication/token",
                  json={"userName": "your-username", "password": "your-password"})
r.raise_for_status()
token = r.json()["accessToken"]

# 2. Call any endpoint with the token
headers = {"Authorization": f"Bearer {token}"}
r = requests.get(f"{BASE}/vehicles?pageSize=10&pageNumber=1", headers=headers)
r.raise_for_status()
print(r.json()["vehicles"])
```

</details>

<details>
<summary><b>JavaScript (Node 18+ / browser)</b></summary>

```javascript
const BASE = "https://<your-fleetcam-host>/fc-rest/v1";

// 1. Get a bearer token
const auth = await fetch(`${BASE}/authentication/token`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ userName: "your-username", password: "your-password" }),
});
if (!auth.ok) throw new Error(`auth failed: ${auth.status}`);
const { accessToken } = await auth.json();

// 2. Call any endpoint with the token
const r = await fetch(`${BASE}/vehicles?pageSize=10&pageNumber=1`, {
  headers: { Authorization: `Bearer ${accessToken}` },
});
if (!r.ok) throw new Error(`vehicles failed: ${r.status}`);
const { vehicles } = await r.json();
console.log(vehicles);
```

</details>

Tokens are JWTs with a finite lifetime (usually 1 hour). Re-authenticate before they expire — see [Token caching](#5-token-caching) for the recommended pattern.

---

## 2. Account types — what shapes your integration

Before you write any client code, identify which of these you are:

### Single-company customer
You manage one company. Every API call returns data scoped to that company. You don't need `?companyId=` on any request.

### Reseller
You manage a parent company that owns multiple sub-companies. The API offers two ways to fetch data:

| Pattern | Use when |
|---|---|
| **Per-sub-company** (recommended) | Daily operations, dashboards, alerts. Pass `?companyId=<sub-co>` and treat each sub-co as a separate query. |
| **Cross-company aggregation** | Reporting/exports across the hierarchy. Limited to **32 sub-companies per request** and requires a time bound on event search. |

If your hierarchy has more than 32 sub-companies, the cross-company aggregation path returns `400`. You must iterate per-sub-company. See [Reseller workflow](#7-reseller-workflow) for the concrete pattern.

---

## 3. Authentication

### Endpoint
```
POST /authentication/token
Content-Type: application/json

{
  "userName": "your-username",
  "password": "your-password"
}
```

### Successful response
```json
{
  "accessToken": "eyJ...",
  "type": "Bearer",
  "expiresInSeconds": 3600
}
```

### Using the token
Add the token to every request as a Bearer header:
```
Authorization: Bearer <accessToken>
```

### Failure modes
| Status | Cause |
|---|---|
| `401` | Wrong username or password, account inactive, or account revoked |
| `429` | Too many auth attempts (rate-limited) |

The auth response is intentionally vague between "user not found" and "wrong password" to avoid leaking account existence.

### Optional `warning` field
Some accounts (typically admin/super-roles) receive an additional `warning` field on the token response:
```json
{
  "accessToken": "eyJ...",
  "type": "Bearer",
  "expiresInSeconds": 3600,
  "warning": "Session has broad access. Use a scoped account where possible."
}
```
This is informational. The token still works as issued. Treat `warning` as optional and ignore if absent.

---

## 4. Common patterns

### Pagination
List endpoints (`/vehicles`, `/groups`, `/events`, etc.) take `pageNumber` (1-based) and `pageSize` query parameters and return:
```json
{
  "vehicles": [ ... ],
  "currentPageNumber": 1,
  "currentPageSize": 100,
  "requestedPageNumber": 1,
  "requestedPageSize": 100,
  "totalPages": 5,
  "hasMoreData": true
}
```
Iterate while `hasMoreData == true`.

### `companyId` field on every response row
Every top-level row in `events[]`, `vehicles[]`, and `groups[]` (including nested group children) carries a `companyId` field naming the sub-company that owns it. For single-company customers it's always your own id. For resellers fetching per-sub-co, it matches the sub you queried. For resellers using cross-company aggregation, it tags each row with its source sub-co — useful for grouping in reports.

(Sub-objects nested inside a vehicle row — e.g. `dvr.cameras[]`, the vehicle's `groups[]` membership list — don't carry their own `companyId`; they inherit it from the parent vehicle.)

### Error envelope
Every error response (4xx + 5xx) uses this shape:
```json
{
  "errorCode": 400,
  "errorMessage": "Multi-company event search requires a time bound: provide startDate+endDate, provide minutes, or narrow with ?companyId=",
  "errorDetails": null,
  "requestUrl": "/fc-rest/v1/events",
  "traceId": "9d691e9c-0a08-43d5-863d-0fcea7b5b513"
}
```
**Always log the `traceId`.** Send it to support if you need to debug a server-side issue — it ties directly to server logs.

### Time formats
All dates are ISO-8601. **Recommended: UTC with the `Z` suffix** — `2026-05-08T10:30:00Z`. The server accepts other offsets (`2026-05-08T06:30:00-04:00`) and will normalize internally, but UTC keeps your client logs and our server logs aligned to the same wall clock and avoids subtle bugs around DST transitions.

---

## 5. Token caching

Don't authenticate on every call. Cache the token until it's about to expire, then re-authenticate. A simple pattern:

<details>
<summary><b>Python</b></summary>

```python
import time, threading, requests

class FleetCamClient:
    def __init__(self, base_url, username, password):
        self.base_url = base_url.rstrip("/")
        self.username = username
        self.password = password
        self._token = None
        self._token_expires_at = 0
        self._lock = threading.Lock()

    def _token_or_renew(self):
        with self._lock:
            if self._token and time.time() < self._token_expires_at - 30:
                return self._token  # still valid (with 30s safety margin)
            r = requests.post(
                f"{self.base_url}/authentication/token",
                json={"userName": self.username, "password": self.password},
                timeout=10,
            )
            r.raise_for_status()
            body = r.json()
            self._token = body["accessToken"]
            self._token_expires_at = time.time() + body.get("expiresInSeconds", 3600)
            return self._token

    def request(self, method, path, **kwargs):
        headers = kwargs.pop("headers", {})
        headers["Authorization"] = f"Bearer {self._token_or_renew()}"
        return requests.request(method, f"{self.base_url}{path}",
                                headers=headers, timeout=30, **kwargs)
```

</details>

<details>
<summary><b>C#</b></summary>

```csharp
using System.Net.Http.Json;

public class FleetCamClient
{
    private readonly HttpClient _http;
    private readonly string _username;
    private readonly string _password;
    private string? _token;
    private DateTimeOffset _tokenExpiresAt = DateTimeOffset.MinValue;
    private readonly SemaphoreSlim _lock = new(1, 1);

    public FleetCamClient(string baseUrl, string username, string password)
    {
        _http = new HttpClient { BaseAddress = new Uri(baseUrl.TrimEnd('/') + "/") };
        _username = username;
        _password = password;
    }

    private async Task<string> GetTokenAsync(CancellationToken ct = default)
    {
        await _lock.WaitAsync(ct);
        try
        {
            if (_token is not null && DateTimeOffset.UtcNow < _tokenExpiresAt - TimeSpan.FromSeconds(30))
                return _token;

            var resp = await _http.PostAsJsonAsync("authentication/token",
                new { userName = _username, password = _password }, ct);
            resp.EnsureSuccessStatusCode();
            var body = await resp.Content.ReadFromJsonAsync<TokenResponse>(cancellationToken: ct)
                       ?? throw new InvalidOperationException("empty token body");
            _token = body.AccessToken;
            _tokenExpiresAt = DateTimeOffset.UtcNow + TimeSpan.FromSeconds(body.ExpiresInSeconds);
            return _token;
        }
        finally { _lock.Release(); }
    }

    public async Task<HttpResponseMessage> RequestAsync(
        HttpMethod method, string path, HttpContent? content = null, CancellationToken ct = default)
    {
        var req = new HttpRequestMessage(method, path) { Content = content };
        req.Headers.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", await GetTokenAsync(ct));
        return await _http.SendAsync(req, ct);
    }

    private record TokenResponse(string AccessToken, string Type, int ExpiresInSeconds, string? Warning);
}
```

</details>

<details>
<summary><b>JavaScript (Node 18+ / browser)</b></summary>

```javascript
class FleetCamClient {
  constructor(baseUrl, username, password) {
    this.baseUrl = baseUrl.replace(/\/$/, "");
    this.username = username;
    this.password = password;
    this._token = null;
    this._tokenExpiresAt = 0;
    this._refreshPromise = null;
  }

  async _getToken() {
    // 30s safety margin so we don't race expiry mid-request
    if (this._token && Date.now() < this._tokenExpiresAt - 30_000) {
      return this._token;
    }
    // De-dupe concurrent refresh attempts
    if (this._refreshPromise) return this._refreshPromise;

    this._refreshPromise = (async () => {
      const r = await fetch(`${this.baseUrl}/authentication/token`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ userName: this.username, password: this.password }),
      });
      if (!r.ok) throw new Error(`auth failed: HTTP ${r.status}`);
      const body = await r.json();
      this._token = body.accessToken;
      this._tokenExpiresAt = Date.now() + (body.expiresInSeconds ?? 3600) * 1000;
      return this._token;
    })();
    try { return await this._refreshPromise; }
    finally { this._refreshPromise = null; }
  }

  async request(method, path, { body, headers = {}, signal, ...rest } = {}) {
    const token = await this._getToken();
    const init = {
      method,
      headers: {
        ...headers,
        Authorization: `Bearer ${token}`,
        ...(body !== undefined ? { "Content-Type": "application/json" } : {}),
      },
      ...(body !== undefined ? { body: JSON.stringify(body) } : {}),
      ...(signal ? { signal } : {}),
      ...rest,
    };
    return fetch(`${this.baseUrl}${path}`, init);
  }
}

// Reusable error builder — used by every JS workflow sample below.
// Returns an Error with .status, .traceId, and .errorCode attached.
export async function toApiError(resp) {
  let envelope = null;
  try { envelope = await resp.json(); } catch { /* non-JSON body */ }
  const message = envelope?.errorMessage ?? `HTTP ${resp.status}`;
  const traceId = envelope?.traceId ?? "n/a";
  const err = new Error(`HTTP ${resp.status}: ${message} (traceId=${traceId})`);
  err.status = resp.status;
  err.traceId = traceId;
  err.errorCode = envelope?.errorCode;
  return err;
}
```

> **JavaScript samples below assume `FleetCamClient` and `toApiError` from the block above.** Expand §5 first if you're skipping around.

</details>

### 5.1 React integration

For React apps, wrap the client in a Context so every component can call the API without holding its own credentials. Pattern:

1. **`FleetCamProvider`** owns the singleton client and exposes it via Context.
2. **`useFleetCam()`** hook returns the client — components call it from event handlers / effects.
3. Custom data hooks like **`useVehicles()`** wrap a single endpoint call with loading + error state.

> **Production note.** Don't hard-code the password into a browser bundle. Front the API behind your own backend that performs the FleetCam auth and proxies requests, OR use a short-lived service token retrieved from your backend on app load. The samples below assume a Node-side client or a trusted environment.

<details>
<summary><b>React — Provider + base hook</b></summary>

```jsx
import { createContext, useContext, useMemo } from "react";

const FleetCamContext = createContext(null);

export function FleetCamProvider({ baseUrl, username, password, children }) {
  // Memoize so the client is created once per provider mount.
  const client = useMemo(
    () => new FleetCamClient(baseUrl, username, password),
    [baseUrl, username, password],
  );
  return <FleetCamContext.Provider value={client}>{children}</FleetCamContext.Provider>;
}

export function useFleetCam() {
  const client = useContext(FleetCamContext);
  if (!client) throw new Error("useFleetCam must be used inside <FleetCamProvider>");
  return client;
}
```

</details>

<details>
<summary><b>React — wiring at the app root</b></summary>

```jsx
// App.jsx — LOCAL DEV ONLY.
// In production, do NOT ship credentials in the browser bundle. Vite's
// VITE_* env vars are inlined into the client bundle and visible to anyone
// who opens DevTools. Use a backend proxy (your server performs the FleetCam
// auth and forwards the request) or a short-lived service token fetched from
// your backend on app load. See the Production note above.
function App() {
  return (
    <FleetCamProvider
      baseUrl={import.meta.env.VITE_FLEETCAM_BASE_URL}
      username={import.meta.env.VITE_FLEETCAM_USERNAME}
      password={import.meta.env.VITE_FLEETCAM_PASSWORD}  // ⚠ dev only — leaks to client bundle
    >
      <Dashboard />
    </FleetCamProvider>
  );
}
```

</details>

<details>
<summary><b>React — custom data hook with loading + error state</b></summary>

```jsx
import { useEffect, useState } from "react";
// toApiError comes from the FleetCamClient block in §5.

export function useVehicles({ pageNumber = 1, pageSize = 100 } = {}) {
  const client = useFleetCam();
  const [state, setState] = useState({ data: null, loading: true, error: null });

  useEffect(() => {
    const controller = new AbortController();
    setState({ data: null, loading: true, error: null });
    client
      .request(
        "GET",
        `/vehicles?pageSize=${pageSize}&pageNumber=${pageNumber}`,
        { signal: controller.signal },
      )
      .then(async (r) => {
        if (!r.ok) throw await toApiError(r);
        return r.json();
      })
      .then((body) => setState({ data: body, loading: false, error: null }))
      .catch((err) => {
        // Aborted on unmount or prop change — silently ignore; a fresh
        // request was kicked off (or the component is gone).
        if (err.name === "AbortError") return;
        setState({ data: null, loading: false, error: err });
      });
    // Cleanup: actually abort the in-flight fetch, not just gate state updates.
    return () => controller.abort();
  }, [client, pageNumber, pageSize]);

  return state;
}
```

</details>

<details>
<summary><b>React — using the hook in a component</b></summary>

```jsx
function VehicleList() {
  const { data, loading, error } = useVehicles({ pageSize: 50 });

  if (loading) return <div>Loading...</div>;
  if (error)   return <div>Error: {error.message} (traceId: {error.traceId})</div>;

  return (
    <ul>
      {data.vehicles.map((v) => (
        <li key={v.vehicleId}>{v.vehicleName} (company {v.companyId})</li>
      ))}
    </ul>
  );
}
```

</details>

---

## 6. Common workflows

### 6.1 List vehicles

<details>
<summary><b>Python</b></summary>

```python
client = FleetCamClient(BASE, "user", "pass")

r = client.request("GET", "/vehicles?pageSize=100&pageNumber=1")
r.raise_for_status()
body = r.json()
for vehicle in body["vehicles"]:
    print(vehicle["vehicleId"], vehicle["vehicleName"], "->", vehicle["companyId"])

# Page through if more data
while body["hasMoreData"]:
    page = body["currentPageNumber"] + 1
    r = client.request("GET", f"/vehicles?pageSize=100&pageNumber={page}")
    r.raise_for_status()
    body = r.json()
    for vehicle in body["vehicles"]:
        print(vehicle["vehicleId"], vehicle["vehicleName"])
```

</details>

<details>
<summary><b>C#</b></summary>

```csharp
var resp = await client.RequestAsync(HttpMethod.Get, "vehicles?pageSize=100&pageNumber=1");
resp.EnsureSuccessStatusCode();
var page = await resp.Content.ReadFromJsonAsync<VehiclesResponse>();
foreach (var v in page!.Vehicles)
    Console.WriteLine($"{v.VehicleId} {v.VehicleName} -> {v.CompanyId}");

while (page.HasMoreData)
{
    var next = page.CurrentPageNumber + 1;
    resp = await client.RequestAsync(HttpMethod.Get, $"vehicles?pageSize=100&pageNumber={next}");
    resp.EnsureSuccessStatusCode();
    page = await resp.Content.ReadFromJsonAsync<VehiclesResponse>();
    foreach (var v in page!.Vehicles)
        Console.WriteLine($"{v.VehicleId} {v.VehicleName}");
}

public record Vehicle(long VehicleId, string VehicleName, long CompanyId /* ... other fields ... */);
public record VehiclesResponse(
    List<Vehicle> Vehicles,
    int CurrentPageNumber, int CurrentPageSize,
    int RequestedPageNumber, int RequestedPageSize,
    int TotalPages, bool HasMoreData);
```

</details>

<details>
<summary><b>JavaScript</b></summary>

```javascript
const client = new FleetCamClient(BASE, "user", "pass");

let resp = await client.request("GET", "/vehicles?pageSize=100&pageNumber=1");
if (!resp.ok) throw await toApiError(resp);
let body = await resp.json();
for (const v of body.vehicles) {
  console.log(v.vehicleId, v.vehicleName, "->", v.companyId);
}

while (body.hasMoreData) {
  const next = body.currentPageNumber + 1;
  resp = await client.request("GET", `/vehicles?pageSize=100&pageNumber=${next}`);
  if (!resp.ok) throw await toApiError(resp);
  body = await resp.json();
  for (const v of body.vehicles) {
    console.log(v.vehicleId, v.vehicleName);
  }
}
```

</details>

### 6.2 Search events for a time window

<details>
<summary><b>Python</b></summary>

```python
from datetime import datetime, timedelta, timezone

end = datetime.now(timezone.utc).replace(microsecond=0)
start = end - timedelta(hours=24)

r = client.request("POST", "/events?pageSize=500&pageNumber=1",
                   json={
                       "startDate": start.isoformat().replace("+00:00", "Z"),
                       "endDate":   end.isoformat().replace("+00:00", "Z"),
                   })
r.raise_for_status()
events = r.json()["events"]
for e in events:
    print(e["eventId"], e["eventDate"], "company=", e["companyId"])
```

</details>

<details>
<summary><b>JavaScript</b></summary>

```javascript
const end = new Date();
end.setMilliseconds(0);
const start = new Date(end.getTime() - 24 * 60 * 60 * 1000);

const resp = await client.request("POST", "/events?pageSize=500&pageNumber=1", {
  body: {
    startDate: start.toISOString().replace(".000Z", "Z"),
    endDate:   end.toISOString().replace(".000Z", "Z"),
  },
});
if (!resp.ok) throw await toApiError(resp);
const { events } = await resp.json();
for (const e of events) {
  console.log(e.eventId, e.eventDate, "company=", e.companyId);
}
```

</details>

### 6.3 Quick "last N minutes" event lookup
For dashboard-style "what happened recently?" queries, the `minutes` shorthand avoids client-side date math:

<details>
<summary><b>Python</b></summary>

```python
r = client.request("POST", "/events?pageSize=500&pageNumber=1",
                   json={"minutes": 60})  # last 60 minutes, server-computed UTC
r.raise_for_status()
events = r.json()["events"]
```

</details>

<details>
<summary><b>JavaScript</b></summary>

```javascript
const resp = await client.request("POST", "/events?pageSize=500&pageNumber=1", {
  body: { minutes: 60 },  // last 60 minutes, server-computed UTC
});
if (!resp.ok) throw await toApiError(resp);
const { events } = await resp.json();
```

</details>

Constraints:
- `minutes` is **mutually exclusive** with `startDate`/`endDate`. Pass one or the other, not both.
- `minutes` produces a sliding window relative to "now," so it's only valid with `pageNumber=1`. For paged reads, use explicit dates.
- Range: `1` to `10080` (7 days).

### 6.4 Get a single event's details

<details>
<summary><b>Python</b></summary>

```python
event_id = 12345

# Vehicle telemetry around the event
r = client.request("GET", f"/events/logs/{event_id}")
r.raise_for_status()
logs = r.json()

# Pre-signed video URLs
r = client.request("GET", f"/events/{event_id}/urls")
r.raise_for_status()
video_urls = r.json()
```

</details>

<details>
<summary><b>JavaScript</b></summary>

```javascript
const eventId = 12345;

// Vehicle telemetry around the event
let resp = await client.request("GET", `/events/logs/${eventId}`);
if (!resp.ok) throw await toApiError(resp);
const logs = await resp.json();

// Pre-signed video URLs
resp = await client.request("GET", `/events/${eventId}/urls`);
if (!resp.ok) throw await toApiError(resp);
const videoUrls = await resp.json();
```

</details>

---

## 7. Reseller workflow

### 7.1 Discover your sub-companies

<details>
<summary><b>Python</b></summary>

```python
r = client.request("GET", "/companies?pageSize=500&pageNumber=1")
r.raise_for_status()
sub_companies = [c["companyId"] for c in r.json().get("content", [])]
print(f"You have {len(sub_companies)} sub-companies in your hierarchy.")
```

</details>

<details>
<summary><b>JavaScript</b></summary>

```javascript
const resp = await client.request("GET", "/companies?pageSize=500&pageNumber=1");
if (!resp.ok) throw await toApiError(resp);
const body = await resp.json();
const subCompanies = (body.content ?? []).map((c) => c.companyId);
console.log(`You have ${subCompanies.length} sub-companies in your hierarchy.`);
```

</details>

### 7.2 Per-sub-company iteration (recommended pattern)

This is how every reseller integration should look. It scales to any hierarchy size.

<details>
<summary><b>Python</b></summary>

```python
from datetime import datetime, timedelta, timezone

def get_recent_events_for_sub(client, sub_company_id, hours=24):
    end = datetime.now(timezone.utc).replace(microsecond=0)
    start = end - timedelta(hours=hours)
    all_events = []
    page = 1
    while True:
        r = client.request("POST", f"/events?pageSize=500&pageNumber={page}&companyId={sub_company_id}",
                           json={
                               "startDate": start.isoformat().replace("+00:00", "Z"),
                               "endDate":   end.isoformat().replace("+00:00", "Z"),
                           })
        r.raise_for_status()
        body = r.json()
        all_events.extend(body["events"])
        if not body["hasMoreData"]:
            break
        page += 1
    return all_events

# Iterate every sub-company
for sub_id in sub_companies:
    events = get_recent_events_for_sub(client, sub_id, hours=24)
    print(f"  sub {sub_id}: {len(events)} events")
```

</details>

<details>
<summary><b>C#</b></summary>

```csharp
async Task<List<Event>> GetRecentEventsForSubAsync(
    FleetCamClient client, long subCompanyId, int hours = 24, CancellationToken ct = default)
{
    var end = DateTimeOffset.UtcNow.ToUniversalTime();
    var start = end.AddHours(-hours);
    var all = new List<Event>();
    var page = 1;
    while (true)
    {
        var body = JsonContent.Create(new {
            startDate = start.ToString("yyyy-MM-ddTHH:mm:ssZ"),
            endDate   = end.ToString("yyyy-MM-ddTHH:mm:ssZ"),
        });
        var resp = await client.RequestAsync(
            HttpMethod.Post,
            $"events?pageSize=500&pageNumber={page}&companyId={subCompanyId}",
            body, ct);
        resp.EnsureSuccessStatusCode();
        var pageBody = await resp.Content.ReadFromJsonAsync<EventsResponse>(cancellationToken: ct);
        all.AddRange(pageBody!.Events);
        if (!pageBody.HasMoreData) break;
        page++;
    }
    return all;
}

public record Event(long EventId, long VehicleId, long CompanyId,
                    DateTimeOffset EventDate, int EventTypeId /* + lat/lon/hasVideo/etc. — see /v3/api-docs */);
public record EventsResponse(
    List<Event> Events,
    int CurrentPageNumber, int CurrentPageSize,
    int RequestedPageNumber, int RequestedPageSize,
    int TotalPages, bool HasMoreData);
```

</details>

<details>
<summary><b>JavaScript</b></summary>

```javascript
async function getRecentEventsForSub(client, subCompanyId, hours = 24) {
  const end = new Date();
  end.setMilliseconds(0);
  const start = new Date(end.getTime() - hours * 60 * 60 * 1000);
  const all = [];
  let page = 1;
  while (true) {
    const resp = await client.request(
      "POST",
      `/events?pageSize=500&pageNumber=${page}&companyId=${subCompanyId}`,
      {
        body: {
          startDate: start.toISOString().replace(".000Z", "Z"),
          endDate:   end.toISOString().replace(".000Z", "Z"),
        },
      },
    );
    if (!resp.ok) throw await toApiError(resp);
    const body = await resp.json();
    all.push(...body.events);
    if (!body.hasMoreData) break;
    page += 1;
  }
  return all;
}

// Iterate every sub-company
for (const subId of subCompanies) {
  const events = await getRecentEventsForSub(client, subId, 24);
  console.log(`  sub ${subId}: ${events.length} events`);
}
```

</details>

### 7.3 Cross-company aggregation (only for hierarchies ≤ 32 sub-cos)

If you have a small hierarchy and want a single response across all your sub-cos:

<details>
<summary><b>Python</b></summary>

```python
# Multi-scope query — server aggregates across your hierarchy
r = client.request("POST", "/events?pageSize=500&pageNumber=1",
                   json={"minutes": 60})  # required: a time bound
# Each event in the response carries a `companyId` so you can group them
```

Or with explicit dates:
```python
r = client.request("POST", "/events?pageSize=500&pageNumber=1",
                   json={
                       "startDate": "2026-05-07T00:00:00Z",
                       "endDate":   "2026-05-08T00:00:00Z"  # max 7 days
                   })
```

</details>

<details>
<summary><b>JavaScript</b></summary>

```javascript
// Multi-scope query — server aggregates across your hierarchy
const resp = await client.request("POST", "/events?pageSize=500&pageNumber=1", {
  body: { minutes: 60 },  // required: a time bound
});
if (!resp.ok) throw await toApiError(resp);
const { events } = await resp.json();
// Each event carries a `companyId` so you can group them:
const grouped = events.reduce((acc, e) => {
  (acc[e.companyId] ??= []).push(e);
  return acc;
}, {});
```

</details>

If your hierarchy exceeds 32 sub-companies, this path returns `400`:

```json
{
  "errorCode": 400,
  "errorMessage": "Multi-company query exceeds the synchronous limit of 32 sub-companies (resolved scope: 150). Narrow with ?companyId=<companyId>; use GET /companies to list accessible companies.",
  "errorDetails": null,
  "requestUrl": "/fc-rest/v1/events",
  "traceId": "9d691e9c-0a08-43d5-863d-0fcea7b5b513"
}
```

When you see this, switch to the per-sub-company iteration in 7.2.

---

## 8. Error handling

Every error response uses the same envelope. Implement a single error handler that pulls `errorMessage` for user display and `traceId` for support tickets.

<details>
<summary><b>Python</b></summary>

```python
import requests

def handle_response(r):
    if r.ok:
        return r.json()
    try:
        envelope = r.json()
        msg = envelope.get("errorMessage", "Unknown error")
        trace = envelope.get("traceId", "n/a")
    except ValueError:
        msg, trace = r.text or "Unknown error", "n/a"
    raise RuntimeError(f"HTTP {r.status_code}: {msg} (traceId={trace})")
```

</details>

<details>
<summary><b>C#</b></summary>

```csharp
public static async Task<T> HandleAsync<T>(HttpResponseMessage resp, CancellationToken ct = default)
{
    if (resp.IsSuccessStatusCode)
        return (await resp.Content.ReadFromJsonAsync<T>(cancellationToken: ct))!;

    ErrorResponse? env = null;
    try { env = await resp.Content.ReadFromJsonAsync<ErrorResponse>(cancellationToken: ct); } catch { }
    var msg   = env?.ErrorMessage ?? "Unknown error";
    var trace = env?.TraceId ?? "n/a";
    throw new HttpRequestException(
        $"HTTP {(int)resp.StatusCode}: {msg} (traceId={trace})");
}

public record ErrorResponse(int ErrorCode, string ErrorMessage, string? ErrorDetails,
                            string? RequestUrl, string TraceId);
```

</details>

<details>
<summary><b>JavaScript</b></summary>

```javascript
// toApiError — referenced from the React useVehicles hook in §5.1 and the
// JS workflow samples above. Returns an Error with status + traceId attached.
export async function toApiError(resp) {
  let envelope = null;
  try { envelope = await resp.json(); } catch { /* non-JSON body */ }
  const message = envelope?.errorMessage ?? `HTTP ${resp.status}`;
  const traceId = envelope?.traceId ?? "n/a";
  const err = new Error(`HTTP ${resp.status}: ${message} (traceId=${traceId})`);
  err.status = resp.status;
  err.traceId = traceId;
  err.errorCode = envelope?.errorCode;
  return err;
}

// Wrapper that throws on non-2xx — drop-in for fetch responses.
export async function handleResponse(resp) {
  if (resp.ok) return resp.json();
  throw await toApiError(resp);
}

// Usage:
//   const body = await handleResponse(await client.request("GET", "/vehicles"));
```

</details>

### Status code reference

| Status | Meaning | Likely cause | What to do |
|---|---|---|---|
| `400` | Bad request | Validation failure: malformed JSON, missing required field, invalid date, too-wide window, multi-company cap exceeded | Fix the request per `errorMessage` |
| `401` | Unauthenticated | Bad creds, expired token, no token | Re-authenticate |
| `403` | Forbidden | Caller lacks the role; or `?companyId=` is outside caller's hierarchy; or vehicle/event access denied | Verify role assignment / sub-co ownership |
| `404` | Not found | Resource doesn't exist. Some endpoints (e.g. `GET /groups/{id}`) also use `404` to hide existence of resources outside the caller's company. For most endpoints, an inaccessible resource returns `403` instead — see the row above | Verify the id; if you expected access, check role + sub-co ownership |
| `409` | Conflict | Resource in unexpected state (e.g. video paths missing) | Retry later or surface to user |
| `429` | Rate limited | Too many requests | Back off and retry |
| `500` | Internal error | Server-side bug | Capture `traceId` and contact support |
| `503` | Temporarily overloaded | Either (a) the server's fan-out queue is saturated under load, or (b) the rate-limit subsystem failed open (fail-closed safety) | Retry with exponential backoff + jitter |

### Retry strategy
- `503` and timeouts: **exponential backoff with jitter**, 3-5 attempts.
- `429`: respect any `Retry-After` header, otherwise back off.
- `401`: re-authenticate **once** and retry. Don't loop.
- `4xx` (other than 401/429): **don't retry** — the request shape is wrong.
- `500`: don't retry blindly; capture `traceId` and contact support.

---

## 9. Common pitfalls

| Pitfall | Fix |
|---|---|
| Reseller calling `POST /events` with `{}` and no `?companyId=` → `400` | Add `?companyId=<sub>` for daily ops, or supply `startDate`+`endDate` / `minutes` for cross-co aggregation |
| Reseller hierarchy > 32 sub-cos hits cap → `400` on every cross-co call | Use per-sub-co iteration ([§7.2](#72-per-sub-company-iteration-recommended-pattern)) — there is no cross-co synchronous path for big hierarchies |
| Sending a date range > 7 days for cross-co query → `400` | Use a smaller window, or iterate per-sub-co with the wider range |
| `minutes` + `pageNumber=2` → `400` | Use explicit `startDate`/`endDate` for paged queries; `minutes` is for one-shot reads |
| `minutes` + `startDate`/`endDate` → `400` | Pick one — they're mutually exclusive |
| Auth on every call → quota / latency burn | Cache the token until ~30s before expiry ([§5](#5-token-caching)) |
| Date strings without `Z` suffix | All dates must be ISO-8601 UTC: `2026-05-08T10:30:00Z` |
| Token expired mid-batch → `401`s pile up | Wrap your client to detect 401 → re-auth → retry once |

---

## 10. Support

When you contact support about a failed call, always include:
- The `traceId` from the response body
- The full request (URL, method, body)
- The HTTP status and `errorMessage`
- Approximate timestamp (UTC) of the call

The `traceId` ties to server-side logs that show the full request/response cycle, so support can pinpoint the issue without combing logs by hand.
