# Habits API — Flutter Integration Guide

**Status:** Live on `https://test.globalvin.co/kayatest` (as of 2026-04-14)
**Base URL:** `{BASE_URL}/api/v1/habits`
**Auth:** Firebase ID token + single-device header (same pattern as mood + sessions)

This guide covers everything the Flutter app needs to implement Habit
creation, editing, tracking, pinning, and the Progress-tab streak display.

---

## 1. Authentication headers

Every request needs both headers:

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <unique_device_identifier>
Content-Type: application/json
```

If another device has the active session, requests return `403`. Use
`POST /api/v1/auth/force-login` with the same headers to claim the session.

---

## 2. Data model

### Habit fields

| Field | Type | Editable? | Notes |
|---|---|---|---|
| `id` | UUID string | ❌ system | |
| `title` | string | ✅ | Required at create, max 255 chars |
| `frequency` | enum | ✅ | `daily` / `every_other_day` / `weekly` / `bi_weekly` / `monthly` |
| `anchor` | string? | ✅ | Optional "attach to action" cue, e.g. "After coffee" |
| `pillar` | enum? | ✅ | `nutrition` / `movement` / `sleep` / `stress` / `relationships` |
| `target_type` | enum? | ✅ | `count` or `duration` (measurable habits only) |
| `target_value` | number? | ✅ | Required when `target_type` set, must be > 0 |
| `target_unit` | string? | ✅ | Required when `target_type` set, e.g. "minutes" |
| `reminder_time` | `HH:MM`? | ✅ | 24-hour clock |
| `current_streak` | int | ❌ system | Updated on log/unlog |
| `best_streak` | int | ❌ system | Lifetime high — never reduced |
| `is_active` | bool | ❌ system | Flipped by DELETE (soft delete) |
| `is_pinned` | bool | ✅ (via pin/unpin) | |
| `done_today` | bool | ❌ computed | Today's log row exists + completed |
| `today_value` | number? | ❌ computed | Today's logged value (measurable habits) |
| `done_this_period` | bool | ❌ computed | Any log in the current period — same as `done_today` for `daily`, wider window for others |
| `period_ends_at` | ISO date | ❌ computed | End of the current period (see table below) |
| `created_at` | ISO datetime | ❌ system | |

### `period_ends_at` per frequency

| Frequency | Period definition | `period_ends_at` example (today = Thu 2026-04-16) |
|---|---|---|
| `daily` | today only | `2026-04-16` |
| `every_other_day` | 2-day rolling window ending today | `2026-04-16` |
| `weekly` | Mon–Sun of current ISO week | `2026-04-19` (Sunday) |
| `bi_weekly` | 14-day rolling window ending today | `2026-04-16` |
| `monthly` | 1st–last of current calendar month | `2026-04-30` |

Use this field to show "due by Sunday" / "due by April 30" hints on non-daily habits.

---

## 3. Endpoint reference

### `POST /api/v1/habits`

Create a new habit.

**Request body:**
```json
{
  "title": "Morning walk",
  "frequency": "daily",
  "anchor": "After coffee",
  "pillar": "movement",
  "target_type": "duration",
  "target_value": 30,
  "target_unit": "minutes",
  "reminder_time": "07:30"
}
```

Only `title` and `frequency` are required. `target_type` + `target_value` + `target_unit` must all be set together, or none.

**Response 201:** full Habit object (see §4 for Dart model).

**Validation errors (422):**
- Missing `title` or empty string
- `frequency` not in the 5 allowed values
- `pillar` not in the 5 allowed values
- `target_type` set without `target_value` + `target_unit`
- `target_value <= 0`
- `reminder_time` not `HH:MM` 24-hour format

---

### `GET /api/v1/habits`

List the user's habits, ordered pinned-first then oldest-first.

**Query params:**
- `frequency` (optional) — filter to one of the 5 values
- `include_inactive` (default `false`) — include soft-deleted habits
- `pinned` (optional, `true`/`false`) — only pinned or only unpinned

**Response 200:**
```json
{
  "habits": [
    { /* Habit object */ },
    { /* Habit object */ }
  ],
  "count": 2
}
```

**Ordering:** pinned habits always appear at the top of the list — no separate widget fetch needed.

**Filter options for the UI's dropdown:**
- `All` → omit `frequency` param
- `Daily` → `?frequency=daily`
- `Every other day` → `?frequency=every_other_day`
- `Weekly` → `?frequency=weekly`
- `Bi-weekly` → `?frequency=bi_weekly`
- `Monthly` → `?frequency=monthly`

---

### `GET /api/v1/habits/{habit_id}`

Return one habit with today's completion status attached.

**Response 200:** full Habit object.
**Response 404:** habit doesn't exist or isn't owned by the current user.

---

### `PATCH /api/v1/habits/{habit_id}`

Partial update — include only the fields you want to change.

**Request body (any combination of editable fields):**
```json
{ "target_value": 45 }
```
```json
{ "title": "New name", "anchor": "new anchor", "pillar": "sleep" }
```
```json
{ "frequency": "weekly" }
```

**Response 200:** full updated Habit.
**Response 400:** empty body (`"No editable fields provided"`).
**Response 404:** not owned.

**Non-destructive guarantee:**
- `current_streak` and `best_streak` are **never** changed by PATCH.
- Log history is preserved — past completions stay visible in stats.
- Changing `frequency` preserves the current streak; the next log is evaluated against the new frequency's period.
- Changing `target_value` does not rewrite past values. Old completions keep whatever value they had.

---

### `DELETE /api/v1/habits/{habit_id}`

Soft delete — the habit is hidden from the default list but log history is preserved.

**Response 204** on success.
**Response 404** if not owned. Already-deleted habits return 204 (idempotent).

Deleted habits reappear when you pass `?include_inactive=true`. Progress tab stats continue to include their past completions.

---

### `POST /api/v1/habits/{habit_id}/log`

Mark today as completed for this habit.

**Request body (optional):**
```json
{ "value": 25 }
```

Use `value` for measurable habits — e.g. "I walked for 25 min even though target was 30". For non-measurable habits, send `{}`.

**Response 200:**
```json
{
  "habit_id": "701b5036-...",
  "date": "2026-04-14",
  "completed": true,
  "value": 25,
  "current_streak": 1,
  "best_streak": 1
}
```

**Response 404:** habit missing or soft-deleted.

**⚠️ Period-level idempotency (important):**

One completion counts for the whole period. Tapping "done" again in the same period is a no-op — streak stays the same, no new log created:

| Frequency | "Same period" = |
|---|---|
| daily | same day |
| every_other_day | 2-day window |
| weekly | same Mon–Sun week |
| bi_weekly | 14-day window |
| monthly | same calendar month |

A weekly habit tapped Monday and again Wednesday of the same week counts as **one** weekly completion, streak = 1. Tapping the next Monday bumps it to 2.

---

### `DELETE /api/v1/habits/{habit_id}/log`

Undo today's completion.

**Query params:**
- `date` (optional ISO date, default = today) — which day's log to remove

**Response 200:**
```json
{
  "habit_id": "701b5036-...",
  "date": "2026-04-14",
  "completed": false,
  "value": null,
  "current_streak": 0,
  "best_streak": 1
}
```

**Response 400:** `date` param not in `YYYY-MM-DD` format.
**Response 404:** habit missing.

**Guarantees:**
- `best_streak` is **never** reduced (lifetime high).
- `current_streak` always reflects real log history — no drift.
- Unlogging a past date (`?date=2026-04-10`) is supported and triggers a streak recount.

---

### `POST /api/v1/habits/{habit_id}/pin`

Pin a habit. Idempotent.

**Response 200:** full updated Habit (with `is_pinned=true`).
**Response 404:** habit missing or soft-deleted.

### `DELETE /api/v1/habits/{habit_id}/pin`

Unpin a habit. Idempotent.

**Response 200:** full updated Habit (with `is_pinned=false`).
**Response 404:** habit missing.

**UX recommendation:**
- Pinned habits sort to the top of the same `GET /habits` list — **no separate endpoint needed** for the Home pinned widget. Just render the list top-to-bottom and check each item's `is_pinned` flag.
- For a dedicated "pinned only" section, use `GET /habits?pinned=true`.
- No hard cap on pin count — let the user pin as many as they want; the list scrolls.

---

## 4. Dart models

```dart
enum HabitFrequency {
  daily,
  everyOtherDay,
  weekly,
  biWeekly,
  monthly;

  String get apiValue => switch (this) {
        HabitFrequency.daily => 'daily',
        HabitFrequency.everyOtherDay => 'every_other_day',
        HabitFrequency.weekly => 'weekly',
        HabitFrequency.biWeekly => 'bi_weekly',
        HabitFrequency.monthly => 'monthly',
      };

  String get label => switch (this) {
        HabitFrequency.daily => 'Daily',
        HabitFrequency.everyOtherDay => 'Every other day',
        HabitFrequency.weekly => 'Weekly',
        HabitFrequency.biWeekly => 'Bi-weekly',
        HabitFrequency.monthly => 'Monthly',
      };

  static HabitFrequency fromApi(String s) => switch (s) {
        'daily' => HabitFrequency.daily,
        'every_other_day' => HabitFrequency.everyOtherDay,
        'weekly' => HabitFrequency.weekly,
        'bi_weekly' => HabitFrequency.biWeekly,
        'monthly' => HabitFrequency.monthly,
        _ => throw ArgumentError('Unknown frequency: $s'),
      };
}

enum HabitPillar {
  nutrition, movement, sleep, stress, relationships;

  String get apiValue => name;
  String get label => switch (this) {
        HabitPillar.nutrition => 'Nutrition',
        HabitPillar.movement => 'Movement',
        HabitPillar.sleep => 'Sleep',
        HabitPillar.stress => 'Stress Management',
        HabitPillar.relationships => 'Relationships & Connection',
      };
}

enum HabitTargetType { count, duration }

class Habit {
  final String id;
  final String title;
  final HabitFrequency frequency;
  final String? anchor;
  final HabitPillar? pillar;
  final HabitTargetType? targetType;
  final double? targetValue;
  final String? targetUnit;
  final String? reminderTime; // 'HH:MM'
  final int currentStreak;
  final int bestStreak;
  final bool isActive;
  final bool isPinned;
  final bool doneToday;
  final double? todayValue;
  final bool doneThisPeriod;
  final DateTime periodEndsAt;
  final DateTime? createdAt;

  Habit({
    required this.id,
    required this.title,
    required this.frequency,
    this.anchor,
    this.pillar,
    this.targetType,
    this.targetValue,
    this.targetUnit,
    this.reminderTime,
    required this.currentStreak,
    required this.bestStreak,
    required this.isActive,
    required this.isPinned,
    required this.doneToday,
    this.todayValue,
    required this.doneThisPeriod,
    required this.periodEndsAt,
    this.createdAt,
  });

  factory Habit.fromJson(Map<String, dynamic> j) => Habit(
        id: j['id'] as String,
        title: j['title'] as String,
        frequency: HabitFrequency.fromApi(j['frequency'] as String),
        anchor: j['anchor'] as String?,
        pillar: j['pillar'] != null
            ? HabitPillar.values.byName(j['pillar'] as String)
            : null,
        targetType: j['target_type'] != null
            ? HabitTargetType.values.byName(j['target_type'] as String)
            : null,
        targetValue: (j['target_value'] as num?)?.toDouble(),
        targetUnit: j['target_unit'] as String?,
        reminderTime: j['reminder_time'] as String?,
        currentStreak: j['current_streak'] as int,
        bestStreak: j['best_streak'] as int,
        isActive: j['is_active'] as bool,
        isPinned: j['is_pinned'] as bool,
        doneToday: j['done_today'] as bool,
        todayValue: (j['today_value'] as num?)?.toDouble(),
        doneThisPeriod: j['done_this_period'] as bool,
        periodEndsAt: DateTime.parse(j['period_ends_at'] as String),
        createdAt: j['created_at'] != null
            ? DateTime.parse(j['created_at'] as String)
            : null,
      );

  bool get isMeasurable => targetType != null;
}

class HabitLogResult {
  final String habitId;
  final DateTime date;
  final bool completed;
  final double? value;
  final int currentStreak;
  final int bestStreak;

  HabitLogResult({
    required this.habitId,
    required this.date,
    required this.completed,
    this.value,
    required this.currentStreak,
    required this.bestStreak,
  });

  factory HabitLogResult.fromJson(Map<String, dynamic> j) => HabitLogResult(
        habitId: j['habit_id'] as String,
        date: DateTime.parse(j['date'] as String),
        completed: j['completed'] as bool,
        value: (j['value'] as num?)?.toDouble(),
        currentStreak: j['current_streak'] as int,
        bestStreak: j['best_streak'] as int,
      );
}
```

---

## 5. Service class (drop-in)

```dart
class HabitsService {
  final Dio _dio; // or http.Client — adapt as needed
  final String baseUrl;

  HabitsService(this._dio, this.baseUrl);

  String get _habitsUrl => '$baseUrl/api/v1/habits';

  Future<Habit> create({
    required String title,
    required HabitFrequency frequency,
    String? anchor,
    HabitPillar? pillar,
    HabitTargetType? targetType,
    double? targetValue,
    String? targetUnit,
    String? reminderTime,
  }) async {
    final r = await _dio.post(_habitsUrl, data: {
      'title': title,
      'frequency': frequency.apiValue,
      if (anchor != null) 'anchor': anchor,
      if (pillar != null) 'pillar': pillar.apiValue,
      if (targetType != null) 'target_type': targetType.name,
      if (targetValue != null) 'target_value': targetValue,
      if (targetUnit != null) 'target_unit': targetUnit,
      if (reminderTime != null) 'reminder_time': reminderTime,
    });
    return Habit.fromJson(r.data);
  }

  Future<List<Habit>> list({
    HabitFrequency? frequency,
    bool includeInactive = false,
    bool? pinned,
  }) async {
    final r = await _dio.get(_habitsUrl, queryParameters: {
      if (frequency != null) 'frequency': frequency.apiValue,
      if (includeInactive) 'include_inactive': 'true',
      if (pinned != null) 'pinned': pinned.toString(),
    });
    return (r.data['habits'] as List)
        .map((j) => Habit.fromJson(j as Map<String, dynamic>))
        .toList();
  }

  Future<Habit> get(String id) async {
    final r = await _dio.get('$_habitsUrl/$id');
    return Habit.fromJson(r.data);
  }

  Future<Habit> update(String id, Map<String, dynamic> changes) async {
    final r = await _dio.patch('$_habitsUrl/$id', data: changes);
    return Habit.fromJson(r.data);
  }

  Future<void> delete(String id) async {
    await _dio.delete('$_habitsUrl/$id');
  }

  Future<HabitLogResult> logToday(String id, {double? value}) async {
    final r = await _dio.post(
      '$_habitsUrl/$id/log',
      data: {if (value != null) 'value': value},
    );
    return HabitLogResult.fromJson(r.data);
  }

  Future<HabitLogResult> unlogToday(String id, {DateTime? date}) async {
    final r = await _dio.delete(
      '$_habitsUrl/$id/log',
      queryParameters: {
        if (date != null) 'date': date.toIso8601String().split('T').first,
      },
    );
    return HabitLogResult.fromJson(r.data);
  }

  Future<Habit> pin(String id) async {
    final r = await _dio.post('$_habitsUrl/$id/pin');
    return Habit.fromJson(r.data);
  }

  Future<Habit> unpin(String id) async {
    final r = await _dio.delete('$_habitsUrl/$id/pin');
    return Habit.fromJson(r.data);
  }
}
```

---

## 6. UX patterns

### Home screen "Pinned habits" widget
```dart
// Single call — the list is already sorted pinned-first.
final habits = await service.list();
final pinned = habits.where((h) => h.isPinned).toList();
// Render vertical list of pinned habits with their current_streak pills.
```

### Progress tab "Habits" list
```dart
// Same endpoint, render all. Pinned ones naturally group at top.
final habits = await service.list(frequency: selectedFilter);
ListView.builder(
  itemCount: habits.length,
  itemBuilder: (_, i) {
    final h = habits[i];
    return HabitTile(habit: h, onToggle: () async {
      if (h.doneThisPeriod) {
        await service.unlogToday(h.id);
      } else {
        await service.logToday(h.id);
      }
      // refresh list
    });
  },
);
```

### Rendering daily vs non-daily habits
For `daily` habits, `doneToday == doneThisPeriod` — a simple checkbox works.

For `weekly` / `bi_weekly` / `monthly`, use `doneThisPeriod` instead of `doneToday`:
- If `doneThisPeriod == true`: show the habit as "✓ done" with a subtle "logged 2 days ago" label — **don't** offer a second checkbox for this period.
- If `doneThisPeriod == false`: show the checkbox with a "due by {periodEndsAt}" hint.

```dart
Widget statusWidget(Habit h) {
  if (h.frequency == HabitFrequency.daily) {
    return Checkbox(value: h.doneToday, onChanged: (_) => ...);
  }
  if (h.doneThisPeriod) {
    return Row(children: [
      Icon(Icons.check_circle, color: Colors.green),
      Text('Done this ${_periodLabel(h.frequency)}'),
    ]);
  }
  return Checkbox(
    value: false,
    onChanged: (_) => service.logToday(h.id),
  );
}
```

### Measurable habits
When `h.isMeasurable`, the log dialog should let the user type the actual value:
```dart
final actualMinutes = await showValueDialog(
  title: 'How many minutes?',
  hint: '${h.targetValue} ${h.targetUnit}',
);
await service.logToday(h.id, value: actualMinutes);
```
Subsequent fetches will return `today_value: actualMinutes` so you can show "25 / 30 min" on the tile.

### Streak pill
```dart
Widget streakPill(Habit h) {
  if (h.currentStreak == 0) return const SizedBox.shrink();
  return Chip(
    avatar: const Text('🔥'),
    label: Text('${h.currentStreak}'),
  );
}
```

---

## 7. When to refetch

Refetch `GET /habits` after:
- Create, update, delete
- Log / unlog
- Pin / unpin
- App resume (to pick up changes from the AI coach during a session)
- Crossing midnight (`doneToday` and `period_ends_at` change at day boundaries)

The backend does **not** push updates; Flutter polls or refetches.
