# Goals + Progress API — Flutter Integration Guide

**Status:** Live on `https://test.globalvin.co/kayatest` (as of 2026-04-16)
**Base URL:** `{BASE_URL}/api/v1/goals` and `{BASE_URL}/api/v1/progress`
**Auth:** Firebase ID token + single-device header (same pattern as habits + mood + sessions)

This guide covers everything the Flutter app needs to implement the Goals
CRUD flow (Progress → Goals tab) and the Progress-tab hero card (streak +
weekly progress + 3 stat tiles).

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

## 2. Goals data model

### Goal fields

| Field | Type | Editable? | Notes |
|---|---|---|---|
| `id` | UUID string | ❌ system | |
| `title` | string | ✅ | **Specific Goal** — required at create, max 255 chars |
| `measurable` | string? | ✅ | **Measurable / Target** — free text, e.g. "Finish under 2 hours" |
| `steps` | string[]? | ✅ | **Attainable & Realistic** — ordered list, 1+ items when present |
| `timeframe` | enum? | ✅ | `tomorrow` / `within_week` / `within_2_weeks` / `within_1_month` / `custom` |
| `target_date` | ISO date? | ✅ | Required when `timeframe='custom'`, optional otherwise |
| `status` | enum | ✅ | `active` / `completed` / `paused` / `abandoned` |
| `is_pinned` | bool | ✅ (via pin/unpin) | |
| `pillar` | enum? | ⚠️ agent-only | Set by the AI coach; UI does not expose a pillar picker |
| `description` | string? | ⚠️ agent-only | Longer context written by the agent |
| `created_at` | ISO datetime | ❌ system | |
| `updated_at` | ISO datetime | ❌ system | |

### Timeline enum → UI label

| API value | UI label | How it renders |
|---|---|---|
| `tomorrow` | Tomorrow | pill chip |
| `within_week` | Within a week | pill chip |
| `within_2_weeks` | Within 2 weeks | pill chip |
| `within_1_month` | Within 1 month | pill chip |
| `custom` | By {date} | show `target_date` formatted, e.g. "By 28 June" |

`target_date` is only set when `timeframe='custom'`. For preset timeframes, leave `target_date` null — the UI computes the resolved date on the fly if it needs one.

### Agent vs. user-created goals

Goals can be created by the AI coach during a Foundation session (via the `set_goal` tool) or by the user directly from the Progress tab. Both flow to the same table. Agent-created goals may have `pillar` and `description` filled in where user-created ones don't — render those fields conditionally when present, and hide them when `null`.

---

## 3. Goals endpoint reference

### `POST /api/v1/goals`

Create a new goal.

**Request body:**
```json
{
  "title": "Run a half-marathon",
  "measurable": "Finish under 2 hours",
  "steps": ["Buy tracker", "Run 5k baseline", "Weekly long runs"],
  "timeframe": "within_1_month"
}
```

or with a custom date:
```json
{
  "title": "Meditate daily",
  "steps": ["Download app", "Morning routine"],
  "timeframe": "custom",
  "target_date": "2026-08-01"
}
```

Only `title` is required. `measurable`, `steps`, `timeframe`, `target_date` are all optional, but if `timeframe='custom'` then `target_date` is mandatory.

**Response 201:** full Goal object.

**Validation errors (422):**
- Missing `title` or empty string
- `timeframe` not in the 5 allowed values
- `timeframe='custom'` without `target_date`
- `steps` contains empty or whitespace-only strings
- `status` not in `active/completed/paused/abandoned` (PATCH only)

---

### `GET /api/v1/goals`

List the user's goals, ordered pinned-first then oldest-first.

**Query params:**
- `status` (default `active`) — one of `active` / `completed` / `paused` / `abandoned` / `all`
- `pinned` (optional, `true`/`false`) — filter to only pinned or only unpinned

**Response 200:**
```json
{
  "goals": [
    { /* Goal object */ },
    { /* Goal object */ }
  ],
  "count": 2
}
```

**Ordering:** pinned goals always appear at the top — no separate fetch needed for a "pinned goals" widget.

**Typical UI calls:**
- Active tab → omit `status` (defaults to active)
- Completed archive → `?status=completed`
- Everything → `?status=all`

---

### `GET /api/v1/goals/{goal_id}`

Return a single goal.

**Response 200:** full Goal object.
**Response 404:** goal doesn't exist or isn't owned by the current user.

---

### `PATCH /api/v1/goals/{goal_id}`

Partial update — include only the fields you want to change.

**Request body (any combination of editable fields):**
```json
{ "title": "Half-marathon 2026" }
```
```json
{ "steps": ["Only one step now"] }
```
```json
{ "timeframe": "custom", "target_date": "2026-09-01" }
```
```json
{ "status": "paused" }
```

**Response 200:** full updated Goal.
**Response 400:** empty body (`"No editable fields provided"`).
**Response 404:** not owned.

**Non-destructive guarantees:**
- Historical session references to this goal id stay valid forever.
- `created_at`, `session_id`, agent-managed fields (`pillar`, `description`, `smart_*`) are not touched by user PATCH.
- `steps` is **fully replaced** when provided — send the whole array, not a diff.
- Editing `timeframe` does not reset or recompute anything; it's just a label/date update.

---

### `DELETE /api/v1/goals/{goal_id}`

Soft delete — flips `status='abandoned'`. The row is preserved so the agent's past references (via `update_goal` or stored session transcripts) remain coherent.

**Response 204** on success.
**Response 404** if not owned.

Abandoned goals disappear from the default list but are still visible via `?status=abandoned` or `?status=all`.

---

### `POST /api/v1/goals/{goal_id}/complete`

Mark a goal as completed. Dedicated endpoint for the Flutter "done" action — cleaner than `PATCH { status: "completed" }` but does the same thing server-side.

**Response 200:** full updated Goal (with `status='completed'`).
**Response 404:** not owned.

Completed goals disappear from the default list and surface under `?status=completed`.

---

### `POST /api/v1/goals/{goal_id}/pin`

Pin a goal. Idempotent.

**Response 200:** full updated Goal (with `is_pinned=true`).
**Response 404:** not owned.

### `DELETE /api/v1/goals/{goal_id}/pin`

Unpin a goal. Idempotent.

**Response 200:** full updated Goal (with `is_pinned=false`).
**Response 404:** not owned.

**UX recommendation:**
- Pinned goals sort to the top of the same `GET /goals` list. The Progress → Goals tab just renders the list top-to-bottom and checks `is_pinned` per row for the pin-icon visual.
- For a dedicated "pinned only" section, use `GET /goals?pinned=true`.
- No hard cap on pin count.

---

## 4. Progress endpoint reference

### `GET /api/v1/progress/summary`

Single call that populates the entire Progress tab hero + stats row.

**Response 200:**
```json
{
  "current_streak": 12,
  "week_progress": {
    "days_completed": 5,
    "days_total": 7
  },
  "stats": {
    "sessions_count": 8,
    "avg_mood": 7.2,
    "habits_done_count": 34
  }
}
```

Empty users get all zeros cleanly — no 404, no null fields.

### What each number means

| Field | Meaning | Window |
|---|---|---|
| `current_streak` | Consecutive days with **any** habit completion (user-level, not per-habit) | walking back from today; soft-today (see below) |
| `week_progress.days_completed` | Distinct dates in the current ISO week (Mon–today) with ≥1 habit completion | current Mon–Sun |
| `week_progress.days_total` | Always `7` | — |
| `stats.sessions_count` | Coaching sessions (intro + foundation + followup), all statuses | all-time |
| `stats.avg_mood` | Mean of `moods.rating` across all entries | all-time |
| `stats.habits_done_count` | Count of completed `habit_logs` rows | last 30 days |

### Soft-today streak

The streak counts days where ≥1 habit was completed, walking backward from today. **If today has no log yet, we start walking from yesterday** — so a user who was perfect Mon–Thu and opens the app Friday morning sees `current_streak=4`, not `0`. When they log their first habit on Friday, it bumps to `5`. This is the Duolingo pattern.

You don't need to implement anything for this on the client — just display the number.

### Sessions count excludes general-chat

The "Sessions" tile represents coaching journey progress. General-chat threads (the "Chat with Kaya" shortcut from the Home page) are excluded. Only `intro`, `foundation`, and `followup` sessions count, regardless of whether they finished via completion or 24-hour expiry.

### Avg mood source

`avg_mood` is the average of **all** rows in the `moods` table, not just today's. It uses the multi-entry-per-day mood feature shipped earlier — if a user logs their mood 3 times in one day, all 3 contribute to the average. Rounded to 1 decimal, displayed as-is.

If the user has never logged a mood, `avg_mood` is `0.0`. The UI should treat this as "no data yet" and show a placeholder (e.g. "—") instead of the number 0.

---

## 5. Dart models

```dart
enum GoalTimeframe {
  tomorrow,
  withinWeek,
  within2Weeks,
  within1Month,
  custom;

  String get apiValue => switch (this) {
        GoalTimeframe.tomorrow => 'tomorrow',
        GoalTimeframe.withinWeek => 'within_week',
        GoalTimeframe.within2Weeks => 'within_2_weeks',
        GoalTimeframe.within1Month => 'within_1_month',
        GoalTimeframe.custom => 'custom',
      };

  String get label => switch (this) {
        GoalTimeframe.tomorrow => 'Tomorrow',
        GoalTimeframe.withinWeek => 'Within a week',
        GoalTimeframe.within2Weeks => 'Within 2 weeks',
        GoalTimeframe.within1Month => 'Within 1 month',
        GoalTimeframe.custom => 'Choose date',
      };

  static GoalTimeframe? fromApi(String? s) => switch (s) {
        'tomorrow' => GoalTimeframe.tomorrow,
        'within_week' => GoalTimeframe.withinWeek,
        'within_2_weeks' => GoalTimeframe.within2Weeks,
        'within_1_month' => GoalTimeframe.within1Month,
        'custom' => GoalTimeframe.custom,
        _ => null,
      };
}

enum GoalStatus { active, completed, paused, abandoned;
  String get apiValue => name;
  static GoalStatus fromApi(String s) => GoalStatus.values.byName(s);
}

class Goal {
  final String id;
  final String title;
  final String? measurable;
  final List<String>? steps;
  final GoalTimeframe? timeframe;
  final DateTime? targetDate;
  final GoalStatus status;
  final bool isPinned;
  final String? pillar;        // agent-set, optional display
  final String? description;   // agent-set, optional display
  final DateTime? createdAt;
  final DateTime? updatedAt;

  Goal({
    required this.id,
    required this.title,
    this.measurable,
    this.steps,
    this.timeframe,
    this.targetDate,
    required this.status,
    required this.isPinned,
    this.pillar,
    this.description,
    this.createdAt,
    this.updatedAt,
  });

  factory Goal.fromJson(Map<String, dynamic> j) => Goal(
        id: j['id'] as String,
        title: j['title'] as String,
        measurable: j['measurable'] as String?,
        steps: (j['steps'] as List?)?.cast<String>(),
        timeframe: GoalTimeframe.fromApi(j['timeframe'] as String?),
        targetDate: j['target_date'] != null
            ? DateTime.parse(j['target_date'] as String)
            : null,
        status: GoalStatus.fromApi(j['status'] as String),
        isPinned: j['is_pinned'] as bool,
        pillar: j['pillar'] as String?,
        description: j['description'] as String?,
        createdAt: j['created_at'] != null
            ? DateTime.parse(j['created_at'] as String)
            : null,
        updatedAt: j['updated_at'] != null
            ? DateTime.parse(j['updated_at'] as String)
            : null,
      );
}

class WeekProgress {
  final int daysCompleted;
  final int daysTotal;
  WeekProgress({required this.daysCompleted, required this.daysTotal});

  factory WeekProgress.fromJson(Map<String, dynamic> j) => WeekProgress(
        daysCompleted: j['days_completed'] as int,
        daysTotal: j['days_total'] as int,
      );
}

class ProgressStats {
  final int sessionsCount;
  final double avgMood;
  final int habitsDoneCount;

  ProgressStats({
    required this.sessionsCount,
    required this.avgMood,
    required this.habitsDoneCount,
  });

  factory ProgressStats.fromJson(Map<String, dynamic> j) => ProgressStats(
        sessionsCount: j['sessions_count'] as int,
        avgMood: (j['avg_mood'] as num).toDouble(),
        habitsDoneCount: j['habits_done_count'] as int,
      );
}

class ProgressSummary {
  final int currentStreak;
  final WeekProgress weekProgress;
  final ProgressStats stats;

  ProgressSummary({
    required this.currentStreak,
    required this.weekProgress,
    required this.stats,
  });

  factory ProgressSummary.fromJson(Map<String, dynamic> j) => ProgressSummary(
        currentStreak: j['current_streak'] as int,
        weekProgress: WeekProgress.fromJson(j['week_progress']),
        stats: ProgressStats.fromJson(j['stats']),
      );
}
```

---

## 6. Service class (drop-in)

```dart
class GoalsService {
  final Dio _dio;
  final String baseUrl;

  GoalsService(this._dio, this.baseUrl);

  String get _goalsUrl => '$baseUrl/api/v1/goals';

  Future<Goal> create({
    required String title,
    String? measurable,
    List<String>? steps,
    GoalTimeframe? timeframe,
    DateTime? targetDate,
  }) async {
    final r = await _dio.post(_goalsUrl, data: {
      'title': title,
      if (measurable != null) 'measurable': measurable,
      if (steps != null) 'steps': steps,
      if (timeframe != null) 'timeframe': timeframe.apiValue,
      if (targetDate != null)
        'target_date': targetDate.toIso8601String().split('T').first,
    });
    return Goal.fromJson(r.data);
  }

  Future<List<Goal>> list({GoalStatus? status, bool? pinned}) async {
    final r = await _dio.get(_goalsUrl, queryParameters: {
      if (status != null) 'status': status.apiValue,
      if (pinned != null) 'pinned': pinned.toString(),
    });
    return (r.data['goals'] as List)
        .map((j) => Goal.fromJson(j as Map<String, dynamic>))
        .toList();
  }

  Future<Goal> get(String id) async {
    final r = await _dio.get('$_goalsUrl/$id');
    return Goal.fromJson(r.data);
  }

  Future<Goal> update(String id, Map<String, dynamic> changes) async {
    final r = await _dio.patch('$_goalsUrl/$id', data: changes);
    return Goal.fromJson(r.data);
  }

  Future<void> delete(String id) async {
    await _dio.delete('$_goalsUrl/$id');
  }

  Future<Goal> complete(String id) async {
    final r = await _dio.post('$_goalsUrl/$id/complete');
    return Goal.fromJson(r.data);
  }

  Future<Goal> pin(String id) async {
    final r = await _dio.post('$_goalsUrl/$id/pin');
    return Goal.fromJson(r.data);
  }

  Future<Goal> unpin(String id) async {
    final r = await _dio.delete('$_goalsUrl/$id/pin');
    return Goal.fromJson(r.data);
  }
}

class ProgressService {
  final Dio _dio;
  final String baseUrl;

  ProgressService(this._dio, this.baseUrl);

  Future<ProgressSummary> summary() async {
    final r = await _dio.get('$baseUrl/api/v1/progress/summary');
    return ProgressSummary.fromJson(r.data);
  }
}
```

---

## 7. UX patterns

### Add Goal modal

Fields in order (matching the Figma):
1. **Specific Goal** → `title` (text, required)
2. **Measurable → Target** → `measurable` (text, optional)
3. **Attainable & Realistic** → `steps` (repeating text list with + Add Step button)
4. **Timeline** → `timeframe` (dropdown showing the 5 labels; if user picks "Choose date", reveal a date picker that feeds `target_date` and sets `timeframe='custom'`)

```dart
Future<void> onSave(Map<String, dynamic> formValues) async {
  await goalsService.create(
    title: formValues['title'] as String,
    measurable: formValues['measurable'] as String?,
    steps: (formValues['steps'] as List<String>?)?.where((s) => s.trim().isNotEmpty).toList(),
    timeframe: formValues['timeframe'] as GoalTimeframe?,
    targetDate: formValues['target_date'] as DateTime?,
  );
  // refetch list
}
```

### Edit Goal modal

Same fields, pre-filled. PATCH only the keys that actually changed:

```dart
Future<void> onSaveEdit(Goal original, Map<String, dynamic> form) async {
  final changes = <String, dynamic>{};
  if (form['title'] != original.title) changes['title'] = form['title'];
  if (form['measurable'] != original.measurable) changes['measurable'] = form['measurable'];
  if (!listEquals(form['steps'], original.steps)) changes['steps'] = form['steps'];
  if (form['timeframe'] != original.timeframe) {
    changes['timeframe'] = (form['timeframe'] as GoalTimeframe?)?.apiValue;
  }
  if (form['target_date'] != original.targetDate) {
    changes['target_date'] = (form['target_date'] as DateTime?)
        ?.toIso8601String()
        .split('T')
        .first;
  }

  if (changes.isEmpty) return;
  await goalsService.update(original.id, changes);
}
```

### Goal card rendering

```dart
Widget goalCard(Goal g) {
  return Card(
    child: Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(children: [
            const Icon(Icons.flag_outlined),
            const SizedBox(width: 8),
            Expanded(child: Text(g.title, style: Theme.of(context).textTheme.titleMedium)),
            IconButton(
              icon: Icon(g.isPinned ? Icons.push_pin : Icons.push_pin_outlined),
              onPressed: () => g.isPinned
                  ? service.unpin(g.id)
                  : service.pin(g.id),
            ),
            IconButton(
              icon: const Icon(Icons.edit_outlined),
              onPressed: () => showEditModal(g),
            ),
          ]),
          if (g.timeframe != null) ...[
            const SizedBox(height: 4),
            Text(
              g.timeframe == GoalTimeframe.custom && g.targetDate != null
                  ? 'By ${DateFormat.yMMMd().format(g.targetDate!)}'
                  : g.timeframe!.label,
              style: Theme.of(context).textTheme.bodySmall,
            ),
          ],
          if (g.steps != null && g.steps!.isNotEmpty) ...[
            const SizedBox(height: 12),
            for (var i = 0; i < g.steps!.length; i++)
              Padding(
                padding: const EdgeInsets.only(bottom: 4),
                child: Row(children: [
                  Text('Step ${i + 1}  ', style: const TextStyle(fontWeight: FontWeight.w600)),
                  Expanded(child: Text(g.steps![i])),
                ]),
              ),
          ],
          if (g.measurable != null) ...[
            const SizedBox(height: 8),
            Row(children: [
              const Text('Measure by: ', style: TextStyle(fontWeight: FontWeight.w600)),
              Expanded(child: Text(g.measurable!)),
            ]),
          ],
        ],
      ),
    ),
  );
}
```

### Mark a goal complete

```dart
await goalsService.complete(g.id);
// Refetch — the goal disappears from the active list on its own.
```

### Progress tab hero + stats

```dart
final summary = await progressService.summary();

// Hero
Text('${summary.currentStreak} Days', style: heroLarge);
const Text('Current Streak', style: heroSmall);
LinearProgressIndicator(
  value: summary.weekProgress.daysCompleted / summary.weekProgress.daysTotal,
);
Text('${summary.weekProgress.daysCompleted} of ${summary.weekProgress.daysTotal} days completed this week');

// 3 stat tiles
statTile('Sessions',    '${summary.stats.sessionsCount}');
statTile('Avg. Mood',   summary.stats.avgMood > 0 ? summary.stats.avgMood.toStringAsFixed(1) : '—');
statTile('Habits Done', '${summary.stats.habitsDoneCount}');
```

---

## 8. When to refetch

Refetch `GET /goals` after:
- Create, update, delete
- Complete, pin / unpin
- Session end (agent may have created or updated goals during the chat)
- App resume

Refetch `GET /progress/summary` after:
- Any habit log / unlog
- Any new mood entry
- Any session ending
- App resume
- Crossing midnight (streak + week progress shift at day boundaries)

The backend does **not** push updates; Flutter polls or refetches on the triggers above.

---

## 9. Error handling

| Status | Meaning | Flutter should |
|---|---|---|
| 200 / 201 | success | parse response |
| 204 | delete success | no body, update UI |
| 400 | empty PATCH body | show "Nothing to save" |
| 403 | device conflict | prompt user to force-login or switch device |
| 404 | not owned / wrong id | refresh list, the goal may have been deleted elsewhere |
| 422 | validation error | show the error from `detail` field next to the offending input |
| 500 | server error | retry once, then show generic error toast |

Validation errors come back as:
```json
{
  "detail": [
    { "loc": ["body", "timeframe"], "msg": "timeframe must be one of [...]", "type": "value_error" }
  ]
}
```

Map `loc` to the field name in the form and highlight that field.

---
