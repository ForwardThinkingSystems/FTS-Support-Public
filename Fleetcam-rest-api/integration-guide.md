# FleetCam REST API — Integration Guide

For developers integrating with the FleetCam REST API. Covers authentication, the two account types (single-company customers and resellers), common workflows, and complete copy-paste code samples in Python and C#.

> **Looking for the full reference?** The OpenAPI spec is at `https://<host>/fc-rest/v1/v3/api-docs` (JSON) and `/swagger-ui.html` (interactive). This guide explains the concepts and patterns; the spec is the authoritative endpoint catalog.

---

## 1. Quick start

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
Every event/vehicle/group row carries a `companyId` field naming the sub-company that owns it. For single-company customers it's always your own id. For resellers fetching per-sub-co, it matches the sub you queried. For resellers using cross-company aggregation, it tags each row with its source sub-co — useful for grouping in reports.

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
All dates are ISO-8601 in UTC: `2026-05-08T10:30:00Z`. Don't send timezones other than `Z`.

---

## 5. Token caching

Don't authenticate on every call. Cache the token until it's about to expire, then re-authenticate. A simple pattern:

**Python:**
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

**C#:**
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

---

## 6. Common workflows

### 6.1 List vehicles

**Python:**
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

**C#:**
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

### 6.2 Search events for a time window

**Python:**
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

### 6.3 Quick "last N minutes" event lookup
For dashboard-style "what happened recently?" queries, the `minutes` shorthand avoids client-side date math:

```python
r = client.request("POST", "/events?pageSize=500&pageNumber=1",
                   json={"minutes": 60})  # last 60 minutes, server-computed UTC
r.raise_for_status()
events = r.json()["events"]
```

Constraints:
- `minutes` is **mutually exclusive** with `startDate`/`endDate`. Pass one or the other, not both.
- `minutes` produces a sliding window relative to "now," so it's only valid with `pageNumber=1`. For paged reads, use explicit dates.
- Range: `1` to `10080` (7 days).

### 6.4 Get a single event's details

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

---

## 7. Reseller workflow

### 7.1 Discover your sub-companies

```python
r = client.request("GET", "/companies?pageSize=500&pageNumber=1")
r.raise_for_status()
sub_companies = [c["companyId"] for c in r.json().get("content", [])]
print(f"You have {len(sub_companies)} sub-companies in your hierarchy.")
```

### 7.2 Per-sub-company iteration (recommended pattern)

This is how every reseller integration should look. It scales to any hierarchy size.

**Python:**
```python
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

**C#:**
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
                    DateTimeOffset EventDate, int EventTypeId /* ... */);
public record EventsResponse(List<Event> Events, bool HasMoreData /* ... pagination fields ... */);
```

### 7.3 Cross-company aggregation (only for hierarchies ≤ 32 sub-cos)

If you have a small hierarchy and want a single response across all your sub-cos:

```python
# Multi-scope query — server fans out across your hierarchy
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

If your hierarchy exceeds 32 sub-companies, this path returns `400`:

```json
{
  "errorCode": 400,
  "errorMessage": "Multi-company query exceeds the synchronous limit of 32 sub-companies (resolved scope: 150). Narrow with ?companyId=<companyId>; use GET /companies to list accessible companies.",
  "traceId": "..."
}
```

When you see this, switch to the per-sub-company iteration in 7.2.

---

## 8. Error handling

Every error response uses the same envelope. Implement a single error handler that pulls `errorMessage` for user display and `traceId` for support tickets.

**Python:**
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

**C#:**
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

### Status code reference

| Status | Meaning | Likely cause | What to do |
|---|---|---|---|
| `400` | Bad request | Validation failure: malformed JSON, missing required field, invalid date, too-wide window, multi-company cap exceeded | Fix the request per `errorMessage` |
| `401` | Unauthenticated | Bad creds, expired token, no token | Re-authenticate |
| `403` | Forbidden | Caller lacks the role; or `?companyId=` is outside caller's hierarchy | Verify role assignment / sub-co ownership |
| `404` | Not found | Resource doesn't exist or isn't accessible to caller | Verify the id |
| `409` | Conflict | Resource in unexpected state (e.g. video paths missing) | Retry later or surface to user |
| `429` | Rate limited | Too many requests | Back off and retry |
| `500` | Internal error | Server-side bug | Capture `traceId` and contact support |
| `503` | Temporarily overloaded | Server queue saturated under load (rare) | Retry with exponential backoff |

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
