# Mood API — Flutter Integration Guide

**For:** Flutter mobile app developer
**Backend version:** 2026-04-13 (post-`c5d6e7f8a9b0` migration)
**Base URL:** `https://test.globalvin.co/kayatest/api/v1`
**Auth:** Firebase ID token + device id (same as every other authenticated endpoint)

This guide covers the four mood endpoints the Home page mood card uses, plus
practical Dart code for the service layer, UI wiring, error handling, and
edge cases.

---

## 1. What to build

Per the Figma design (page 36), the mood widget appears on the Home page
**after the user completes their first intro session** — not before.
It contains:

1. Title: **"How are you feeling today?"**
2. A row of 10 numbered buttons, 1 through 10
3. The selected button is filled dark olive with white text (the latest rating)
4. Below the row, after a rating exists:
   - ↑ or ↓ arrow + **"+X from yesterday"** (delta vs yesterday's average)
   - **"Your average this week: 7.4"**
   - A small inline bar mini-chart of the 7 daily averages

There is no text input for a note in the Figma — the backend accepts it in
the POST body but you don't need to send it until the design asks for it.

---

## 2. Authentication (applies to every endpoint below)

Every request needs two headers:

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_unique_id>
```

This is the exact same pattern as the session and chat endpoints —
whatever your existing ApiService does for those, reuse it here.

**If the headers are missing or invalid** → `401 Unauthorized`
**If the user is logged in on another device** → `403 Session active on another device. Use /auth/force-login to switch.`

No special handling needed for mood specifically — whatever you already do
for `/sessions` and `/me` works here.

---

## 3. Endpoints

### 3.1. `POST /api/v1/mood` — Record a mood tap

Called when the user taps any number 1–10 on the Home card.

**Request body:**
```json
{
  "rating": 8
}
```

- `rating` — **required** integer, 1–10 inclusive. Pydantic will reject anything outside that range with `422 Unprocessable Entity`.
- `note` — **optional** string, max 500 chars. Send `null` or omit if you don't have a note input in the UI yet.

**Response: `200 OK`** — returns the full today snapshot, same shape as `GET /mood/today` (so you can update the UI without a second call):

```json
{
  "recorded": true,
  "latest": {
    "rating": 8,
    "recorded_at": "2026-04-13T15:42:00+03:00",
    "freshness": "just now"
  },
  "today": {
    "count": 3,
    "average": 6.67,
    "entries": [
      { "rating": 9, "at": "09:00" },
      { "rating": 4, "at": "14:00" },
      { "rating": 7, "at": "21:00" }
    ]
  },
  "yesterday": {
    "average": 7.0,
    "count": 2
  },
  "delta_vs_yesterday_avg": -0.33,
  "week_average": 7.14
}
```

**Important behavior — 30 second debounce:**
If the user taps the **same rating** within 30 seconds of the last entry (e.g., accidentally double-taps "8"), the backend returns the previous snapshot **without inserting a new row.** This is server-side protection against UI bounce — you don't need to add client-side debounce, but a small 300ms input lock on the buttons is still nice UX so the user can't spam-tap.

If the user taps a **different rating** within 30 seconds (e.g., changes from 8 to 5), a new row IS inserted — this is a real intentional change.

**Errors:**
| Status | When | What to show |
|---|---|---|
| `200` | Success (insert OR debounce no-op) | Update Home card |
| `401` | Token missing/invalid | Redirect to login |
| `403` | Session on another device | Show "Session active on another device" banner + force-login flow |
| `422` | `rating` outside 1–10 or wrong type | Shouldn't happen if buttons are 1–10 only — treat as a bug |
| `5xx` | Server down | Retry with exponential backoff, show "Couldn't save mood, try again" |

---

### 3.2. `GET /api/v1/mood/today` — Load the Home card

Called when:
1. The Home page first appears
2. User navigates back to Home from another tab
3. After any successful `POST /mood` (if you prefer to refetch instead of using the POST response)

**Request:** No body, no query params.

**Response: `200 OK`** — same shape as the POST response. Two variants:

**Case A — user has rated at least once today:**
```json
{
  "recorded": true,
  "latest": {
    "rating": 8,
    "recorded_at": "2026-04-13T15:42:00+03:00",
    "freshness": "3 hours ago"
  },
  "today": {
    "count": 2,
    "average": 7.0,
    "entries": [
      { "rating": 6, "at": "09:00" },
      { "rating": 8, "at": "12:30" }
    ]
  },
  "yesterday": { "average": 7.0, "count": 2 },
  "delta_vs_yesterday_avg": 0.0,
  "week_average": 7.14
}
```

**Case B — user has not rated yet today:**
```json
{
  "recorded": false,
  "latest": null,
  "today": {
    "count": 0,
    "average": null,
    "entries": []
  },
  "yesterday": { "average": 7.0, "count": 2 },
  "delta_vs_yesterday_avg": null,
  "week_average": 7.14
}
```

**UI rules:**
- `recorded == true` → highlight `latest.rating` button as selected; show the delta line and the week average line
- `recorded == false` → no button highlighted; hide the delta line; still show "Your average this week" if `week_average != null`; show nothing at all if the user is brand new

---

### 3.3. `GET /api/v1/mood/week` — 7-day mini-chart data

Called when:
1. The Home card wants to render the mini-chart below the average line
2. The future Progress tab wants a week view

**Request:** No body, no query params.

**Response: `200 OK`:**
```json
{
  "days": [
    { "date": "2026-04-07", "average": 6.5,  "count": 2 },
    { "date": "2026-04-08", "average": 7.0,  "count": 1 },
    { "date": "2026-04-09", "average": 7.5,  "count": 3 },
    { "date": "2026-04-10", "average": null, "count": 0 },
    { "date": "2026-04-11", "average": 6.0,  "count": 4 },
    { "date": "2026-04-12", "average": 7.0,  "count": 2 },
    { "date": "2026-04-13", "average": 8.0,  "count": 3 }
  ],
  "week_average": 7.0,
  "trend": "improving"
}
```

- **`days`** — always 7 items, oldest first (6 days ago → today). Days the user didn't rate have `average: null` and `count: 0` — render as blank/greyed bars.
- **`week_average`** — `null` if no entries in the last 7 days at all
- **`trend`** — one of `"improving"` / `"declining"` / `"stable"` / `"no_data"` (use this to decide arrow direction / color)

**UI rules:**
- If `week_average == null` → don't render the chart at all (or show an empty state)
- Otherwise → render a 7-bar horizontal chart, each bar height = `average / 10`, null days = empty slot
- The order is **oldest on the left, today on the right** (natural reading order)

---

### 3.4. `GET /api/v1/mood/history?days=30` — Flat history list

Not needed for the MVP Home card. Reserved for the future Progress tab chart.

**Query params:**
- `days` — optional int, default 30, max 90, min 1. Values >90 return `422`.

**Response: `200 OK`:**
```json
{
  "moods": [
    {
      "id": "8f3c-...",
      "rating": 8,
      "note": null,
      "date": "2026-04-13",
      "created_at": "2026-04-13T15:42:00+03:00"
    },
    ...
  ],
  "count": 5
}
```

Each entry is a raw row — useful for a timeline chart. Most recent first.

---

## 4. Dart models (paste into your codebase)

```dart
// lib/models/mood.dart

class MoodLatest {
  final int rating;
  final DateTime recordedAt;
  final String freshness;

  MoodLatest({
    required this.rating,
    required this.recordedAt,
    required this.freshness,
  });

  factory MoodLatest.fromJson(Map<String, dynamic> json) => MoodLatest(
    rating: json['rating'],
    recordedAt: DateTime.parse(json['recorded_at']),
    freshness: json['freshness'],
  );
}

class MoodEntry {
  final int rating;
  final String at; // "HH:MM"

  MoodEntry({required this.rating, required this.at});

  factory MoodEntry.fromJson(Map<String, dynamic> json) =>
      MoodEntry(rating: json['rating'], at: json['at']);
}

class MoodTodayBlock {
  final int count;
  final double? average;
  final List<MoodEntry> entries;

  MoodTodayBlock({
    required this.count,
    required this.average,
    required this.entries,
  });

  factory MoodTodayBlock.fromJson(Map<String, dynamic> json) => MoodTodayBlock(
    count: json['count'],
    average: json['average']?.toDouble(),
    entries: (json['entries'] as List)
        .map((e) => MoodEntry.fromJson(e))
        .toList(),
  );
}

class MoodYesterdayBlock {
  final double? average;
  final int count;

  MoodYesterdayBlock({required this.average, required this.count});

  factory MoodYesterdayBlock.fromJson(Map<String, dynamic> json) =>
      MoodYesterdayBlock(
        average: json['average']?.toDouble(),
        count: json['count'],
      );
}

class MoodTodayResponse {
  final bool recorded;
  final MoodLatest? latest;
  final MoodTodayBlock today;
  final MoodYesterdayBlock yesterday;
  final double? deltaVsYesterdayAvg;
  final double? weekAverage;

  MoodTodayResponse({
    required this.recorded,
    required this.latest,
    required this.today,
    required this.yesterday,
    required this.deltaVsYesterdayAvg,
    required this.weekAverage,
  });

  factory MoodTodayResponse.fromJson(Map<String, dynamic> json) =>
      MoodTodayResponse(
        recorded: json['recorded'],
        latest: json['latest'] != null
            ? MoodLatest.fromJson(json['latest'])
            : null,
        today: MoodTodayBlock.fromJson(json['today']),
        yesterday: MoodYesterdayBlock.fromJson(json['yesterday']),
        deltaVsYesterdayAvg: json['delta_vs_yesterday_avg']?.toDouble(),
        weekAverage: json['week_average']?.toDouble(),
      );
}

class MoodWeekDay {
  final DateTime date;
  final double? average;
  final int count;

  MoodWeekDay({required this.date, required this.average, required this.count});

  factory MoodWeekDay.fromJson(Map<String, dynamic> json) => MoodWeekDay(
    date: DateTime.parse(json['date']),
    average: json['average']?.toDouble(),
    count: json['count'],
  );
}

class MoodWeekResponse {
  final List<MoodWeekDay> days;
  final double? weekAverage;
  final String trend; // improving | declining | stable | no_data

  MoodWeekResponse({
    required this.days,
    required this.weekAverage,
    required this.trend,
  });

  factory MoodWeekResponse.fromJson(Map<String, dynamic> json) =>
      MoodWeekResponse(
        days: (json['days'] as List)
            .map((d) => MoodWeekDay.fromJson(d))
            .toList(),
        weekAverage: json['week_average']?.toDouble(),
        trend: json['trend'],
      );
}
```

---

## 5. Service layer (drop-in)

Assumes you already have an `ApiClient` that handles base URL, auth headers,
and JSON parsing. Adjust the method signatures to match yours.

```dart
// lib/services/mood_service.dart

class MoodService {
  final ApiClient _api;
  MoodService(this._api);

  /// Record a new mood tap. Returns the updated Home snapshot.
  Future<MoodTodayResponse> recordMood(int rating, {String? note}) async {
    final body = <String, dynamic>{'rating': rating};
    if (note != null) body['note'] = note;

    final resp = await _api.post('/mood', body: body);
    return MoodTodayResponse.fromJson(resp);
  }

  /// Load today's mood snapshot for the Home card.
  Future<MoodTodayResponse> getToday() async {
    final resp = await _api.get('/mood/today');
    return MoodTodayResponse.fromJson(resp);
  }

  /// Load per-day averages for the last 7 days (mini-chart).
  Future<MoodWeekResponse> getWeek() async {
    final resp = await _api.get('/mood/week');
    return MoodWeekResponse.fromJson(resp);
  }

  /// Load a flat history (used by Progress tab, not the Home card).
  Future<List<dynamic>> getHistory({int days = 30}) async {
    final resp = await _api.get('/mood/history', query: {'days': days});
    return resp['moods'] as List;
  }
}
```

---

## 6. Home card widget wiring (example)

This is a rough outline — adapt it to your state management pattern (Provider, Riverpod, Bloc, etc.).

```dart
// lib/widgets/mood_card.dart

class MoodCard extends StatefulWidget {
  @override
  State<MoodCard> createState() => _MoodCardState();
}

class _MoodCardState extends State<MoodCard> {
  MoodTodayResponse? _today;
  MoodWeekResponse? _week;
  bool _loading = true;
  bool _saving = false;
  DateTime? _lastTapAt; // client-side 300ms tap lock

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    try {
      final todayFuture = context.read<MoodService>().getToday();
      final weekFuture = context.read<MoodService>().getWeek();
      final results = await Future.wait([todayFuture, weekFuture]);
      setState(() {
        _today = results[0] as MoodTodayResponse;
        _week = results[1] as MoodWeekResponse;
        _loading = false;
      });
    } catch (e) {
      setState(() => _loading = false);
      // show error snackbar
    }
  }

  Future<void> _onRatingTapped(int rating) async {
    // 300ms client-side debounce to prevent accidental double-taps
    final now = DateTime.now();
    if (_lastTapAt != null && now.difference(_lastTapAt!).inMilliseconds < 300) {
      return;
    }
    _lastTapAt = now;

    if (_saving) return;
    setState(() => _saving = true);

    try {
      final updated = await context.read<MoodService>().recordMood(rating);
      // POST returns the full snapshot — use it directly, no need to refetch
      setState(() => _today = updated);
      // Week also changes (today's average in the week breakdown) — refresh it
      final week = await context.read<MoodService>().getWeek();
      setState(() => _week = week);
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text("Couldn't save mood. Try again.")),
      );
    } finally {
      setState(() => _saving = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_loading) return const Center(child: CircularProgressIndicator());
    if (_today == null) return const SizedBox.shrink();

    final selected = _today!.latest?.rating;

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text('How are you feeling today?',
                style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 12),

            // 1–10 button row
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: List.generate(10, (i) {
                final value = i + 1;
                final isSelected = selected == value;
                return GestureDetector(
                  onTap: _saving ? null : () => _onRatingTapped(value),
                  child: Container(
                    width: 32,
                    height: 32,
                    alignment: Alignment.center,
                    decoration: BoxDecoration(
                      color: isSelected ? kOliveDark : kCream,
                      borderRadius: BorderRadius.circular(6),
                    ),
                    child: Text(
                      '$value',
                      style: TextStyle(
                        color: isSelected ? Colors.white : kInk,
                      ),
                    ),
                  ),
                );
              }),
            ),

            const SizedBox(height: 12),

            // Delta line — only when today is rated AND we have a yesterday avg
            if (_today!.recorded &&
                _today!.deltaVsYesterdayAvg != null &&
                _today!.yesterday.average != null)
              _DeltaLine(
                delta: _today!.deltaVsYesterdayAvg!,
                yesterdayAvg: _today!.yesterday.average!,
              ),

            // Week average + mini-chart
            if (_week != null && _week!.weekAverage != null) ...[
              const SizedBox(height: 8),
              Row(
                children: [
                  Text(
                    'Your average this week: ${_week!.weekAverage!.toStringAsFixed(1)}',
                  ),
                  const SizedBox(width: 12),
                  Expanded(child: _WeekBarChart(days: _week!.days)),
                ],
              ),
            ],
          ],
        ),
      ),
    );
  }
}

class _DeltaLine extends StatelessWidget {
  final double delta;
  final double yesterdayAvg;
  const _DeltaLine({required this.delta, required this.yesterdayAvg});

  @override
  Widget build(BuildContext context) {
    final isUp = delta > 0;
    final isDown = delta < 0;
    final arrow = isUp ? '↑' : (isDown ? '↓' : '→');
    final color = isUp ? Colors.green : (isDown ? Colors.orange : Colors.grey);
    final sign = delta >= 0 ? '+' : '';

    return Text(
      '$arrow $sign${delta.toStringAsFixed(1)} from yesterday',
      style: TextStyle(color: color, fontSize: 12),
    );
  }
}

class _WeekBarChart extends StatelessWidget {
  final List<MoodWeekDay> days;
  const _WeekBarChart({required this.days});

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      height: 24,
      child: Row(
        children: days.map((d) {
          final ratio = d.average != null ? d.average! / 10.0 : 0.0;
          return Expanded(
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 2),
              child: FractionallySizedBox(
                heightFactor: ratio,
                alignment: Alignment.bottomCenter,
                child: Container(
                  decoration: BoxDecoration(
                    color: d.average == null ? Colors.grey.shade300 : kOliveDark,
                    borderRadius: BorderRadius.circular(2),
                  ),
                ),
              ),
            ),
          );
        }).toList(),
      ),
    );
  }
}
```

---

## 7. When to fetch and refetch

| Event | Action |
|---|---|
| Home page first appears | Call `/mood/today` + `/mood/week` in parallel |
| User taps a number | Call `POST /mood`, use the response to update `_today`, then refetch `/mood/week` (so the chart picks up today's new average) |
| User navigates Home → Coach → Home | Refetch `/mood/today` on re-entry (agent may have recorded a mood during the session) |
| Session ends or user returns from chat | Refetch `/mood/today` and `/mood/week` (agent might have logged a mood mid-conversation) |
| App resume from background after >5 min | Refetch `/mood/today` and `/mood/week` |
| Pull-to-refresh on Home | Refetch both |

**Why refetch after coming back from Coach:** the AI coach has its own
`record_mood` tool and can save mood entries during a session. Both paths
write to the same DB table, so refetching after any chat session picks up
whatever the agent logged.

---

## 8. Edge cases to handle

### User has never rated (brand new user after intro)
- `GET /mood/today` returns `recorded: false`, `latest: null`, `week_average` may still be `null` if no prior week data
- UI: show the 1–10 buttons, no selection, no delta line, no average line
- The whole card can be omitted if the user hasn't completed intro yet (per Figma — the mood widget only appears on Home from state 3 onwards)

### User had a row yesterday but nothing today
- `recorded: false`, but `yesterday.average != null` and `week_average != null`
- UI: show the buttons (no selection), don't show the delta line (`delta_vs_yesterday_avg` is `null`), **do** show "Your average this week: X" because week data exists

### User rates the same number twice in 30s (accidental double-tap)
- The second POST returns the same snapshot as the first, no new row created
- UI: no special handling — it looks like an instant re-save, which is correct

### User rapidly taps 8 → 3 → 9 (intentional changes)
- Each POST creates a new row — no debounce because ratings differ
- After the third tap, `latest.rating == 9`, `today.average == (8+3+9)/3 == 6.67`, `today.entries == [8, 3, 9]`
- UI: always reflects the latest — you only need to track `_today.latest.rating`

### Offline / network error on POST
- Catch the exception, show a snackbar "Couldn't save mood, try again"
- Do NOT optimistically update the UI — wait for success
- Retry only on explicit user action (don't auto-retry, mood data isn't critical enough to spam requests)

### `week_average` is null
- No entries in the last 7 days → hide the average line and chart entirely
- This only happens for brand-new users or users who haven't rated in a week

### Trend is `"no_data"`
- Hide any trend arrow indicator; just show the average line without color

---


