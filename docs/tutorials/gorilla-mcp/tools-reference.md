# Tools Reference

Complete reference for all MCP tools, resources, and prompts.

## Tools

### get_training_plan

Get the current training plan with logged progress.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file` | `string` | No | Specific training file to load. Defaults to the active file. |

**Returns:** Markdown with exercises grouped by day, including:
- Exercise name, weight, sets, reps, rest periods, notes
- Completion status per day: `[DONE]`, `[LOGGED]`, `[PARTIAL]`
- Logged sets with actual weight/reps and technique notes
- Weight suggestions based on progression history
- Schedule (day-to-date mapping) if available

**Example output:**
```markdown
# Training Plan: strength_cycle_3.csv
**Cycle:** Strength

## Day 1 — Squat [LOGGED]
- **Squat**: 125kg x 3 sets x 1 reps (rest: 3-5min) [suggested: 127.5kg via e1rm_progression]
  - Set 1: 100kg x 5 reps
  - Set 2: 115kg x 3 reps
  - Set 3: 125kg x 7 reps (AMRAP)
- **Hack Squat**: 90kg x 2 sets x 15-20 reps (rest: 90s)
```

**Source:** `GET /api/v2/training?file={file}`

---

### get_e1rm_history

Get estimated 1-rep max history for all primary lifts.

**Parameters:** None

**Returns:** Markdown table showing strength progression over time.

**Example output:**
```markdown
# Estimated 1RM History

| Date | Exercise | e1RM (kg) |
|------|----------|----------|
| 2026-04-01 | Squat | 155.0 |
| 2026-04-01 | Bench | 107.5 |
| 2026-03-28 | Deadlift | 180.0 |
```

**Source:** `GET /api/v2/training/e1rm-history`

---

### get_exercise_history

Get all logged sets for a specific exercise across all training plans.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `exercise` | `string` | Yes | Exercise name (case-sensitive, must match plan). |

**Returns:** Markdown table of all sets with weight, reps, date, and which plan file they came from.

**Example output:**
```markdown
# Exercise History: Squat

| Date | Weight | Reps | Plan |
|------|--------|------|------|
| 2026-04-01 | 125 | 7 | strength_cycle_3.csv |
| 2026-03-25 | 120 | 5 | strength_cycle_3.csv |
```

**Source:** `GET /api/v2/training/exercise-history?exercise={exercise}`

---

### get_garmin_daily

Get Garmin biometric data for a specific date or date range.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `date` | `string` | No | Single date (YYYY-MM-DD). Defaults to today. |
| `start` | `string` | No | Range start (YYYY-MM-DD). Use with `end`. |
| `end` | `string` | No | Range end (YYYY-MM-DD). Use with `start`. |

If `start` and `end` are both provided, returns a summary table for the range. Otherwise returns detailed single-day data.

**Single-day output includes:** Activity (steps, distance, calories, floors, intensity), Heart Rate (resting, avg, max), HRV (last night, weekly avg, status), Sleep (score, duration, deep/REM, overnight HR), Stress & Body Battery (avg stress, high/low battery), Training (readiness, load, VO2 max), Body (weight, body fat), Activities (name, type, duration, avg HR, load).

**Range output:** Summary table with date, steps, RHR, HRV, sleep score, stress, battery, readiness.

**Source:** `GET /api/v1/users/{id}/daily?date=` or `?start=&end=`

---

### get_garmin_vitals

Get today's vitals with 7-day baseline comparison. **Cached (5-min TTL).**

**Parameters:** None

**Returns:** Comparison table showing today's values vs 7-day rolling averages with delta.

**Example output:**
```markdown
# Vitals (Today vs 7-Day Baseline)

| Metric | Today | Baseline | Delta |
|--------|-------|----------|-------|
| HRV | 94.0 | 77.0 | +17.0 |
| RHR | 45.0 | 46.0 | +1.0 |
| Stress | 12.0 | 26.0 | -14.0 |
| Body Battery | 99.0 | 79.0 | +20.0 |
| Sleep Score | 90.0 | 78.0 | +12.0 |
| Training Readiness | 95.0 | - | - |

**HRV Status:** Balanced
```

**Source:** `GET /api/v1/users/{id}/vitals`

---

### get_garmin_baseline

Get N-day rolling biometric averages. **Cached (5-min TTL, per `days` value).**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `days` | `integer` | No | Number of days for rolling average. Default: 7. |

**Returns:** Averaged biometric stats over the specified period.

**Example output:**
```markdown
# Baseline (7-day averages)

- HRV: 77.0ms
- RHR: 46.0bpm
- Stress: 26.0
- Body battery: 79.0
- Sleep score: 78.0
- Sleep: 6.7hours
- Steps: 8500.0
```

**Source:** `GET /api/v1/users/{id}/baseline?days={days}`

---

### get_sitrep

Get today's SITREP (situation report). Daily snapshot of training status, biometrics, and readiness.

**Parameters:** None

**Returns:** Pre-formatted Markdown report from gorilla_coach including:
- Overall status (GREEN/AMBER/RED)
- Gatekeeper assessment (STRIKE/HOLD with reasons)
- Evidence (HRV, battery, sleep, vitals, SpO2, body mass, training readiness)
- Deployment orders and fuel status

**Source:** `GET /api/v2/reports/sitrep`

---

### get_aar

Get the after-action review (AAR). Retrospective analysis of the most recent training session.

**Parameters:** None

**Returns:** Pre-formatted Markdown report including:
- Mission summary (type, duration, work/rest ratio, intensity)
- Gatekeeper status at time of training
- Performance data (sets, reps, volume, time per set)
- Actual performance (user-logged lifts)
- Planned vs actual comparison
- Cardiac response (HR zones, training effect)
- Tactical assessment, sustain/improve, next mission

**Source:** `GET /api/v2/reports/aar`

---

### get_debrief

Get the 7-day debrief report. Weekly summary of training, biometrics, and recovery.

**Parameters:** None

**Returns:** Pre-formatted Markdown report including:
- Trajectory (climbing/stable/declining)
- Gatekeeper week summary (strike/hold days with reasons)
- System analysis (HRV, sleep, battery, training volume, RHR, stress, body mass)
- Pattern recognition (post-training dips, sleep-recovery links)
- Wins and flags
- Strategic orders

**Source:** `GET /api/v2/reports/debrief`

---

### get_dashboard

Get the training dashboard overview.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `date` | `string` | No | Date (YYYY-MM-DD). Defaults to today. |

**Returns:** Markdown dashboard with:
- Last Garmin sync timestamp
- Personal baseline (avg HRV, RHR, sleep score, stress, body battery, steps, sleep hours)
- Training log (exercises with sets, weights, reps by date)

**Source:** `GET /api/v2/dashboard?date={date}`

---

### get_auto_regulator

Get 5/3/1 auto-regulator status.

**Parameters:** None

**Returns:** Markdown with:
- Mission orders: `EXECUTE` (lift name), `STRUCTURAL HOLD` (recovery timer), `SYSTEMIC HOLD` (biometric flags), or `FIRST SESSION`
- Cycle week (Week 1 = 5s, Week 2 = 3s, Week 3 = 5/3/1)
- Working sets table (set number, weight, reps, percentage, AMRAP flag)
- Biometric telemetry (HRV status, sleep score, training readiness, body battery, stress)
- Accessories (name, sets, reps, notes)
- Training maxes (Squat, Bench, Deadlift, OHP)

**Source:** `GET /api/v2/auto-reg`

---

### log_training_sets

Log completed exercise sets to the training plan.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file` | `string` | Yes | Training plan filename (e.g., `strength_cycle_3.csv`) |
| `day` | `string` | Yes | Day label (e.g., `Day 1 — Squat`) |
| `exercise` | `string` | Yes | Exercise name (must match plan exactly) |
| `sets` | `array` | Yes | Array of set objects (see below) |
| `performed_at` | `string` | No | ISO 8601 timestamp. Defaults to now. |

**Set object:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `set` | `integer` | Yes | Set number (1-indexed) |
| `weight` | `string` | No | Weight used (e.g., `"125"`) |
| `reps` | `string` | No | Reps completed (e.g., `"7"`) |
| `technique` | `string` | No | Technique note (e.g., `"AMRAP"`, `"MR"`, `"DS"`) |

**Returns:** `"Training sets logged successfully."` on success.

**Side effects:** Invalidates training-related cache entries.

**Source:** `POST /api/v2/training/log`

---

### log_auto_reg

Log a completed 5/3/1 lift session.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `lift` | `string` | Yes | Lift name: `Squat`, `Bench`, `Deadlift`, or `Ohp` |
| `training_max` | `float` | Yes | Current training max in kg |
| `cycle_week` | `integer` | Yes | 1 (5s week), 2 (3s week), or 3 (5/3/1 week) |
| `amrap_weight` | `float` | No | Weight used on AMRAP set |
| `amrap_reps` | `integer` | No | Reps achieved on AMRAP set |
| `compound_sets` | `string` | No | JSON string of compound set data |
| `accessories` | `string` | No | JSON string of accessory data |

**Returns:** `"Auto-reg lift logged successfully."` on success.

**Side effects:** Invalidates auto-reg cache entries.

**Source:** `POST /api/v2/auto-reg/log`

---

### trigger_garmin_sync

Trigger an on-demand Garmin data sync.

**Parameters:** None

**Returns:** Sync status message (e.g., `"Sync triggered."`).

**Side effects:** Invalidates all cache entries.

**Source:** `POST /api/v2/settings/sync`

---

## Resources

### gorilla://settings

User settings including Garmin connection status, manual body fat percentage, and the currently active training file.

**Example output:**
```markdown
# Settings

- Garmin username: user@example.com
- Garmin connected: true
- Active training file: strength_cycle_3.csv
- Manual body fat: 15.0%
```

### gorilla://files

List of uploaded training plan files with sizes and which one is active.

**Example output:**
```markdown
# Training Files

- strength_cycle_3.csv (2048 bytes) **(active)**
- hypertrophy_block_1.csv (1536 bytes)
- deload_week.csv (512 bytes)
```

### gorilla://garmin/status

Garmin sync connection status.

**Example output:**
```markdown
# Garmin Connection

- Status: **connected**
- Username: user@example.com
```

---

## Prompts

### readiness_check

**Description:** Assess current training readiness based on biometrics and recovery status.

**Arguments:** None

**Uses tools:** `get_sitrep`, `get_garmin_vitals`, `get_auto_regulator`

**Claude is asked to provide:**
1. Overall readiness assessment (ready / caution / stand down)
2. Key biometric indicators and what they mean
3. Training recommendation for today
4. Any recovery actions needed

---

### weekly_review

**Description:** Comprehensive weekly training and recovery review.

**Arguments:** None

**Uses tools:** `get_debrief`, `get_e1rm_history`, `get_training_plan`

**Claude is asked to analyze:**
1. Training volume and consistency this week
2. Strength trends (e1RM changes)
3. Recovery quality trends (HRV, sleep, stress)
4. Program adherence and any missed sessions
5. Recommendations for next week

---

### program_evaluation

**Description:** Evaluate current training program effectiveness.

| Argument | Required | Description |
|----------|----------|-------------|
| `exercise` | No | Focus on a specific exercise (defaults to "all primary lifts") |

**Uses tools:** `get_training_plan`, `get_e1rm_history`, `get_exercise_history`

**Claude is asked to assess:**
1. Are the primary lifts progressing?
2. Is volume appropriate given recovery data?
3. Are accessories supporting the main lifts?
4. Recommended adjustments

---

### recovery_analysis

**Description:** Deep dive into recovery and biometric trends.

| Argument | Required | Description |
|----------|----------|-------------|
| `days` | No | Lookback period in days (default: 14) |

**Uses tools:** `get_garmin_daily` (date range), `get_garmin_baseline`, `get_sitrep`

**Claude is asked to focus on:**
1. HRV trend and what it indicates about CNS recovery
2. Sleep quality patterns
3. Stress/body battery patterns
4. Training load vs recovery capacity
5. Specific recovery recommendations
