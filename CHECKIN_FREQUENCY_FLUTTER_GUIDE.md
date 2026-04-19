# Check-in Frequency — Flutter Integration Guide

**For:** Flutter mobile app developer
**Backend version:** 2026-04-20
**Base URL:** `https://test.globalvin.co/kayatest/api/v1`
**Auth:** Firebase ID token + device id (same as every other authenticated endpoint)

This guide covers the two endpoints the Settings → Profile screen uses to
display and update the user's check-in frequency, plus practical Dart code
for the service layer, UI wiring, and edge cases.

---

## 1. What to build

Per the Figma design (page 52 — Profile screen), the Profile row shows:

```
Check-in Frequency     Daily  →
```

Tapping the row opens a picker with 5 options. The backend stores an integer
`frequency_days`; the UI labels map to:

| Label | `frequency_days` |
|---|---|
| Daily | `1` |
| Every 2 days | `2` |
| Weekly | `7` |
| Bi-weekly | `14` |
| Monthly | `30` |

When the user picks a new option, call `PUT /me/checkins` and update the
Profile row label with the new value.

---

## 2. Authentication (applies to every endpoint below)

Every request needs two headers:

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_unique_id>
Content-Type: application/json
```

---

## 3. Endpoints

### 3.1 Read current schedule

```
GET /api/v1/me/checkins
```

**Response 200 — schedule exists:**
```json
{
  "schedule": {
    "frequency_days": 7,
    "is_active": true,
    "last_sent_at": "2026-04-15T09:00:00+00:00",
    "next_due_at":  "2026-04-22T09:00:00+00:00"
  }
}
```

**Response 200 — no schedule yet (user hasn't finished foundation, or AI
never scheduled one):**
```json
{ "schedule": null }
```

### 3.2 Update frequency

```
PUT /api/v1/me/checkins
```

**Request:**
```json
{ "frequency_days": 7 }
```

**Response 200:**
```json
{
  "schedule": {
    "frequency_days": 7,
    "is_active": true,
    "last_sent_at": null,
    "next_due_at":  "2026-04-27T12:00:00+00:00"
  }
}
```

**Validation:**
- `frequency_days` is required, integer, `1 ≤ n ≤ 30`.
- Out-of-range values return `422` with the standard FastAPI validation body.

**Behaviour:**
- Upserts — creates the schedule row if missing, updates it otherwise.
- `next_due_at` is set to `now() + frequency_days` on every call, so the
  next check-in is measured from when the user changes the setting, not
  from the previous schedule.
- `is_active` is always set to `true` (paused state is not exposed in MVP).

---

## 4. Dart models

```dart
class CheckinSchedule {
  final int frequencyDays;
  final bool isActive;
  final DateTime? lastSentAt;
  final DateTime? nextDueAt;

  CheckinSchedule({
    required this.frequencyDays,
    required this.isActive,
    this.lastSentAt,
    this.nextDueAt,
  });

  factory CheckinSchedule.fromJson(Map<String, dynamic> json) {
    return CheckinSchedule(
      frequencyDays: json['frequency_days'],
      isActive: json['is_active'],
      lastSentAt: json['last_sent_at'] != null
          ? DateTime.parse(json['last_sent_at'])
          : null,
      nextDueAt: json['next_due_at'] != null
          ? DateTime.parse(json['next_due_at'])
          : null,
    );
  }

  String get label => _labelFor(frequencyDays);

  static String _labelFor(int days) {
    switch (days) {
      case 1:  return 'Daily';
      case 2:  return 'Every 2 days';
      case 7:  return 'Weekly';
      case 14: return 'Bi-weekly';
      case 30: return 'Monthly';
      default: return 'Every $days days';
    }
  }
}
```

---

## 5. Service class (drop-in)

```dart
class CheckinService {
  final ApiClient api;

  CheckinService(this.api);

  /// Read the current schedule. Returns null if not set yet.
  Future<CheckinSchedule?> getSchedule() async {
    final r = await api.get('/api/v1/me/checkins');
    final s = r.data['schedule'];
    return s == null ? null : CheckinSchedule.fromJson(s);
  }

  /// Update or create the schedule.
  Future<CheckinSchedule> setFrequency(int frequencyDays) async {
    final r = await api.put('/api/v1/me/checkins', body: {
      'frequency_days': frequencyDays,
    });
    return CheckinSchedule.fromJson(r.data['schedule']);
  }
}
```

---

## 6. UI — Profile row + picker

```dart
class CheckinFrequencyRow extends StatefulWidget {
  final CheckinService service;

  const CheckinFrequencyRow({required this.service});

  @override
  State<CheckinFrequencyRow> createState() => _CheckinFrequencyRowState();
}

class _CheckinFrequencyRowState extends State<CheckinFrequencyRow> {
  CheckinSchedule? schedule;
  bool loading = true;

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    final s = await widget.service.getSchedule();
    setState(() { schedule = s; loading = false; });
  }

  Future<void> _pickNew() async {
    final current = schedule?.frequencyDays ?? 1;
    final picked = await showModalBottomSheet<int>(
      context: context,
      builder: (_) => _FrequencyPicker(current: current),
    );
    if (picked == null || picked == current) return;

    try {
      final updated = await widget.service.setFrequency(picked);
      setState(() { schedule = updated; });
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Could not save — please try again')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    final label = loading
        ? '…'
        : (schedule?.label ?? 'Not set');
    return ListTile(
      title: const Text('Check-in Frequency'),
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(label, style: Theme.of(context).textTheme.bodyMedium),
          const Icon(Icons.chevron_right),
        ],
      ),
      onTap: loading ? null : _pickNew,
    );
  }
}

class _FrequencyPicker extends StatelessWidget {
  final int current;
  const _FrequencyPicker({required this.current});

  static const options = [
    (1,  'Daily'),
    (2,  'Every 2 days'),
    (7,  'Weekly'),
    (14, 'Bi-weekly'),
    (30, 'Monthly'),
  ];

  @override
  Widget build(BuildContext context) {
    return SafeArea(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const Padding(
            padding: EdgeInsets.all(16),
            child: Text('Check-in Frequency',
                style: TextStyle(fontWeight: FontWeight.bold, fontSize: 18)),
          ),
          ...options.map((o) => ListTile(
                title: Text(o.$2),
                trailing: o.$1 == current
                    ? const Icon(Icons.check, color: Colors.green)
                    : null,
                onTap: () => Navigator.pop(context, o.$1),
              )),
          const SizedBox(height: 8),
        ],
      ),
    );
  }
}
```

---

## 7. When to call which endpoint

| Event | Call |
|---|---|
| Settings/Profile screen opens | `getSchedule()` to populate the row |
| User picks a new option from the sheet | `setFrequency(days)` then refresh the row |
| User completes a coaching session | Nothing — the agent may have updated it via `schedule_checkin`. Re-read on next Profile visit. |

There is no need to poll or subscribe. Read once on screen open, write when
the user changes it.

---

## 8. Interaction with the AI agent

The AI agent can also set the frequency during a foundation or follow-up
session via its internal `schedule_checkin` tool. The backend handles both
sources cleanly:

- If the agent sets frequency in session → next Profile open shows the new
  value.
- If the user changes frequency from Settings → the agent is informed at the
  start of the next session and will reference it naturally ("We agreed I'd
  check in with you weekly — does that still work?"), not re-ask from
  scratch.

Your Flutter app does not need to do anything special to coordinate these
— just re-read the schedule when the Profile screen becomes visible.

---

## 9. Empty state

If `GET /me/checkins` returns `{"schedule": null}`, the user hasn't finished
their foundation session yet (the AI only sets the schedule after step 15
of the foundation outline, or the user can set it themselves from Settings
after that — same endpoint).

**Recommended UI:** still show the row with label `"Not set"` and let the
user tap to open the picker. Any selection calls `PUT /me/checkins` which
creates the row.

---

## 10. Error handling

| HTTP Code | Meaning | Action |
|---|---|---|
| `200` | Success | Process response |
| `401` | Token expired | Refresh Firebase token, retry |
| `403` | Device conflict | Show force-login prompt |
| `422` | `frequency_days` out of range (not 1–30) | Should not happen if UI only offers valid options — log as bug |
| `500` | Server error | Show generic "try again" snackbar |

---


*End of guide — updated 2026-04-20*
