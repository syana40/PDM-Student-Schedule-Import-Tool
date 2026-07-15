# Schedule → ICS

A browser console script that scrapes Inside PDM Student Scedule's web calendar and exports it as an `.ics` file

**Live instructions page:** enable GitHub Pages on this repo (Settings → Pages) and open the published URL — it has full setup steps, the script itself, and troubleshooting.

## What it does

- Clicks through the calendar's "next week" view automatically and reads off each event
- Detects quizzes and exams and tags them (`QUIZ:` / `EXAM:`)
- Outputs correct `America/New_York` timezone info (works right even while traveling)
- Adds a 10-minute-before reminder on every event, plus a 1-week-before reminder on exams/quizzes
- Runs entirely in your browser — no server, no login, no data sent anywhere

## Quick start

1. Open your school's calendar page, log in.
2. Open the browser console (right-click → Inspect → Console tab).
3. Copy `scrape_schedule.js` from this repo (or the instructions page) and paste it into the console.
4. Press Enter and wait — the calendar will flip through weeks automatically. Don't navigate away.
5. `schedule.ics` downloads automatically when it's done. Import it into Apple Calendar, Google Calendar, or Outlook.

## Config

All settings are in one block near the top of `scrape_schedule.js`:

| Setting | Default | Controls |
|---|---|---|
| `WEEKS_FORWARD` | `20` | How many weeks ahead to scrape |
| `WEEKS_BACK` | `0` | How many weeks before the starting page to also grab |
| `WAIT_MS` | `1200` | Delay (ms) after each "next week" click |
| `SCHOOL_TIMEZONE` | `America/New_York` | IANA timezone used for event times |
| `ENABLE_STANDARD_ALERT` | `true` | Reminder before every event |
| `STANDARD_ALERT_MINUTES_BEFORE` | `10` | Minutes before, for the standard reminder |
| `ENABLE_EXAM_QUIZ_ALERT` | `true` | Extra reminder before exams/quizzes |
| `EXAM_QUIZ_ALERT_DAYS_BEFORE` | `7` | Days before, for the exam/quiz reminder |

## Re-importing

Delete previously imported events before importing a newer `.ics` file — each run generates fresh event IDs, so calendar apps treat a re-import as new events rather than updates, which causes duplicates.

## Troubleshooting

See the **Office Hours** section on the instructions page for common issues (zero events found, missing weeks, shifted times, blocked downloads, etc.).
