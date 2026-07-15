# Schedule → ICS

Pull your **Inside PDM Student Schedule** straight out of the browser and into a proper `.ics` file - with reminders built in.

## What this actually does

This script operates the page itself — clicking "next week" over and over. It is reading each event off the grid, then exporting it into a single calendar file you can import anywhere.

| Included | Handling |
|---|---|
| Class times & locations | Read directly from each event block, including room number |
| Quizzes & exams | Auto-detected from event text and tagged `QUIZ:` / `EXAM:` in the title |
| Timezone | Tagged as `America/New_York` with a full `VTIMEZONE` block — displays correctly wherever you are, EDT or EST |
| Reminders | 10 minutes before every event, plus 1 week before exams/quizzes (both configurable) |

## Instructions

1. **Open your calendar page.** Log in and land on the work-week view you'd normally page through. **NOTE: The script starts scraping from whatever week is currently on screen, so start from the first week of classes**

2. **Open the browser console.** Right-click anywhere on the page → **Inspect** → click the **Console** tab. On Mac you can also use `Cmd+Option+J` (Chrome) or `Cmd+Option+K` (Firefox). On Windows, `Ctrl+Shift+J`.

3. **Adjust the config (optional).** Copy the script below and skim the config block at the top — set how many weeks forward to scrape and whether you want the reminder alerts. See the [Configuration](#configuration) section for what each option does.

4. **Paste and run.** Paste the whole script into the console and hit `Enter`. The calendar will visibly flip forward week by week — **don't click or navigate away while it's running.**

5. **Collect the file.** When it finishes, the console logs a count of events found and your browser downloads `schedule.ics` automatically. Import that file into Apple Calendar, Google Calendar, or Outlook.

> **Before you re-import:** if you've imported an earlier version of this file before, delete those old events from your calendar first — re-importing without removing the old set will create duplicates.

## The script

Paste this in full.

```javascript
(async function () {
  // ---- CONFIG ----
  const WEEKS_FORWARD = 20;   // how many weeks ahead to scrape from today
  const WEEKS_BACK = 0;       // how many weeks in the past to also grab
  const WAIT_MS = 1200;       // delay after clicking "next"/"previous" for the AJAX reload
  const SCHOOL_TIMEZONE = 'America/New_York'; // Philadelphia

  // Alerts
  const ENABLE_STANDARD_ALERT = true;   // alert before every event
  const STANDARD_ALERT_MINUTES_BEFORE = 10;

  const ENABLE_EXAM_QUIZ_ALERT = true;  // extra alert before exams/quizzes only
  const EXAM_QUIZ_ALERT_DAYS_BEFORE = 7;
  // -----------------

  const events = [];

  function parseWeek() {
    const dayHeaders = {};
    document.querySelectorAll('.wc-day-column-header').forEach(td => {
      const cls = [...td.classList].find(c => /^wc-day-\d$/.test(c));
      if (cls) {
        const dayNum = cls.split('-')[2];
        dayHeaders[dayNum] = td.textContent.trim(); // e.g. "Monday Aug 17, 2026"
      }
    });

    document.querySelectorAll('.wc-day-column-inner').forEach(col => {
      const dayClass = [...col.classList].find(c => /^day-\d$/.test(c));
      if (!dayClass) return;
      const dayNum = dayClass.split('-')[1];
      const dateStr = dayHeaders[dayNum];
      if (!dateStr) return;

      col.querySelectorAll('.wc-cal-event').forEach(evt => {
        const timeText = evt.querySelector('.wc-time')?.textContent.trim();
        const titleEl = evt.querySelector('.wc-title');
        if (!timeText || !titleEl) return;

        const parts = titleEl.innerHTML
          .split(/<br\s*\/?>/i)
          .map(p => p.replace(/<[^>]+>/g, '').trim())
          .filter(Boolean);

        const courseCode = parts[0] || '';
        const courseName = parts[1] || '';
        let room = '';
        let type = '';
        const extra = [];

        parts.slice(2).forEach(p => {
          if (/^Room:/i.test(p)) room = p.replace(/^Room:\s*/i, '').trim();
          else if (!type) type = p;
          else extra.push(p);
        });

        const isExam = /exam/i.test(type) || /exam/i.test(extra.join(' '));
        const isQuiz = /quiz/i.test(type) || /quiz/i.test(extra.join(' '));

        events.push({
          dateStr, timeText, courseCode, courseName, room, type,
          extra: extra.join(' | '), isExam, isQuiz
        });
      });
    });
  }

  function waitForReload() {
    return new Promise(resolve => setTimeout(resolve, WAIT_MS));
  }

  async function clickAndWait(selector) {
    const btn = document.querySelector(selector);
    if (!btn) return false;
    btn.click();
    await waitForReload();
    return true;
  }

  // Go back WEEKS_BACK weeks first
  for (let i = 0; i < WEEKS_BACK; i++) {
    await clickAndWait('.wc-prev');
  }

  parseWeek();
  for (let i = 0; i < WEEKS_FORWARD + WEEKS_BACK; i++) {
    const moved = await clickAndWait('.wc-next');
    if (!moved) break;
    parseWeek();
  }

  // Dedupe
  const seen = new Set();
  const unique = events.filter(e => {
    const key = JSON.stringify(e);
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });

  console.log(`Scraped ${unique.length} unique events.`);

  // ---- Build ICS ----
  function to24h(timePart, isPM) {
    let [h, m] = timePart.split(':').map(Number);
    if (isPM && h !== 12) h += 12;
    if (!isPM && h === 12) h = 0;
    return { h, m };
  }

  function parseTimeRange(timeText) {
    const [startStr, endStr] = timeText.split('-').map(s => s.trim());
    const startHour = parseInt(startStr.split(':')[0], 10);
    const endHour = parseInt(endStr.split(':')[0], 10);
    // Grid runs 7AM-7PM: hours 7-11 = AM, hour 12 and 1-6 = PM
    const startPM = !(startHour >= 7 && startHour <= 11);
    const endPM = !(endHour >= 7 && endHour <= 11);
    return { start: to24h(startStr, startPM), end: to24h(endStr, endPM) };
  }

  function pad(n) { return String(n).padStart(2, '0'); }

  // Format a plain local date/time (school-local, e.g. Philadelphia) for use
  // with DTSTART;TZID=America/New_York:... — no UTC conversion needed here,
  // since the VTIMEZONE block below tells calendar apps how to interpret it.
  function formatLocalDate(year, month, day, hour, minute) {
    return `${year}${pad(month)}${pad(day)}T${pad(hour)}${pad(minute)}00`;
  }

  // VTIMEZONE block defining EDT/EST switch rules for America/New_York
  const VTIMEZONE_BLOCK =
    'BEGIN:VTIMEZONE\r\n' +
    'TZID:America/New_York\r\n' +
    'X-LIC-LOCATION:America/New_York\r\n' +
    'BEGIN:DAYLIGHT\r\n' +
    'TZOFFSETFROM:-0500\r\n' +
    'TZOFFSETTO:-0400\r\n' +
    'TZNAME:EDT\r\n' +
    'DTSTART:19700308T020000\r\n' +
    'RRULE:FREQ=YEARLY;BYMONTH=3;BYDAY=2SU\r\n' +
    'END:DAYLIGHT\r\n' +
    'BEGIN:STANDARD\r\n' +
    'TZOFFSETFROM:-0400\r\n' +
    'TZOFFSETTO:-0500\r\n' +
    'TZNAME:EST\r\n' +
    'DTSTART:19701101T020000\r\n' +
    'RRULE:FREQ=YEARLY;BYMONTH=11;BYDAY=1SU\r\n' +
    'END:STANDARD\r\n' +
    'END:VTIMEZONE\r\n';

  function makeAlarm(triggerValue) {
    return `BEGIN:VALARM\r\nACTION:DISPLAY\r\nDESCRIPTION:Reminder\r\nTRIGGER:${triggerValue}\r\nEND:VALARM\r\n`;
  }

  // Builds the VALARM block(s) for one event, based on the CONFIG options above:
  // a standard alert before every event, plus an extra alert before exams/quizzes only
  function buildAlarms(isExam, isQuiz) {
    let alarms = '';

    if (ENABLE_STANDARD_ALERT) {
      alarms += makeAlarm(`-PT${STANDARD_ALERT_MINUTES_BEFORE}M`);
    }

    if (ENABLE_EXAM_QUIZ_ALERT && (isExam || isQuiz)) {
      alarms += makeAlarm(`-P${EXAM_QUIZ_ALERT_DAYS_BEFORE}D`);
    }

    return alarms;
  }

  function escapeICS(str) {
    return (str || '')
      .replace(/\\/g, '\\\\')
      .replace(/;/g, '\\;')
      .replace(/,/g, '\\,')
      .replace(/\n/g, '\\n');
  }

  let ics = 'BEGIN:VCALENDAR\r\nVERSION:2.0\r\nPRODID:-//Schedule Export//EN\r\nCALSCALE:GREGORIAN\r\n';
  ics += VTIMEZONE_BLOCK;

  unique.forEach((e, idx) => {
    const cleanDateStr = e.dateStr.replace(/^\w+\s/, ''); // strip "Monday " etc.
    const dateObj = new Date(cleanDateStr);
    if (isNaN(dateObj)) return;

    const { start, end } = parseTimeRange(e.timeText);
    const y = dateObj.getFullYear(), mo = dateObj.getMonth() + 1, d = dateObj.getDate();
    const dtStart = formatLocalDate(y, mo, d, start.h, start.m);
    const dtEnd = formatLocalDate(y, mo, d, end.h, end.m);

    const prefix = e.isExam ? 'EXAM: ' : e.isQuiz ? 'QUIZ: ' : '';
    const summary = `${prefix}${e.courseCode} ${e.type || ''}`.trim();
    const description = [e.courseName, e.type, e.extra].filter(Boolean).join(' — ');

    ics += 'BEGIN:VEVENT\r\n';
    ics += `UID:${dtStart}-${idx}@schedule-export\r\n`;
    ics += `DTSTART;TZID=${SCHOOL_TIMEZONE}:${dtStart}\r\n`;
    ics += `DTEND;TZID=${SCHOOL_TIMEZONE}:${dtEnd}\r\n`;
    ics += `SUMMARY:${escapeICS(summary)}\r\n`;
    ics += `LOCATION:${escapeICS(e.room)}\r\n`;
    ics += `DESCRIPTION:${escapeICS(description)}\r\n`;
    ics += buildAlarms(e.isExam, e.isQuiz);
    ics += 'END:VEVENT\r\n';
  });

  ics += 'END:VCALENDAR\r\n';

  const blob = new Blob([ics], { type: 'text/calendar' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'schedule.ics';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);

  console.log('Download triggered: schedule.ics');
})();
```

## Configuration

Every option lives in one block near the top of the script — change the value, then re-paste and re-run.

| Setting | Controls |
|---|---|
| `WEEKS_FORWARD` | How many weeks ahead of the starting page to scrape. Set this to cover your full term (e.g. `20` ≈ 5 months). |
| `WEEKS_BACK` | How many weeks *before* the current page to also capture. Leave at `0` unless you want past weeks included. |
| `WAIT_MS` | Delay in milliseconds after each "next week" click, to let the page finish reloading before scraping. Raise this if you notice missing events on a slow connection. |
| `SCHOOL_TIMEZONE` | IANA timezone of the school. Preset to `America/New_York` for Philadelphia. |
| `ENABLE_STANDARD_ALERT` | `true`/`false` — toggles the reminder that fires before every single event. |
| `STANDARD_ALERT_MINUTES_BEFORE` | How many minutes before each event that reminder fires. Default `10`. |
| `ENABLE_EXAM_QUIZ_ALERT` | `true`/`false` — toggles the extra early reminder for events tagged as exams or quizzes. |
| `EXAM_QUIZ_ALERT_DAYS_BEFORE` | How many days before an exam/quiz that extra reminder fires. Default `7`. |

## Troubleshooting

**Nothing downloaded / console shows an error immediately**
Make sure you're on the actual calendar page (not a login screen or a redirect) before pasting, and that you pasted the entire script — a partial paste breaks the closing brackets. If your browser strips a warning like *"Only paste code you understand,"* you may need to type `allow pasting` into the console first, then paste again.

**Console says "Scraped 0 unique events"**
This means the script couldn't find any event blocks on the page. Usually one of two things:
- You're on a week with no scheduled events — try starting from a week you know has classes.
- The calendar page's layout changed. This script reads specific CSS class names (`.wc-cal-event`, `.wc-day-column-inner`, etc.) — if the school's calendar system was updated, those may no longer match. Right-click a visible event → Inspect, and compare the class names to what's referenced near the top of `parseWeek()`. This may mean that the script needs an update. 

**Some weeks are missing events**
Usually a timing issue — the script clicked "next" before the page finished loading that week's data. Increase `WAIT_MS` (try `2000`) and run again.

**Event times look shifted by a few hours**
This means the file lost its timezone tag somewhere along the way — check that `DTSTART`/`DTEND` lines in the resulting `.ics` still read `DTSTART;TZID=America/New_York:...` and not a bare UTC value. If you hand-edited the file, re-run the script fresh instead of patching it manually.

**Duplicate events after importing again**
The script gives each event a fresh, non-deterministic ID on every run, so calendar apps treat a re-import as a brand new set rather than an update. Delete the previously imported events before importing a newer file.

**Reminders aren't showing up in my calendar app**
Confirm `ENABLE_STANDARD_ALERT` / `ENABLE_EXAM_QUIZ_ALERT` were `true` when you generated the file. Also check your calendar app's own notification settings — some apps (particularly on iOS) only show alerts if notifications are enabled for the specific calendar the events were imported into.

**The browser blocked the file download**
Some browsers block automatic downloads triggered from the console by default. Look for a blocked-download icon in the address bar and allow it, then re-run the script — it will re-generate and re-trigger the download.

**Exams or quizzes aren't getting the extra 1-week reminder**
The script only tags an event as an exam/quiz if the word "exam" or "quiz" appears in its event type text on the calendar page. If your school labels these differently (e.g. "Assessment" or "Practical"), the detection in `parseWeek()` needs the matching keyword added to the `isExam` / `isQuiz` checks. You may have to just update those manually in your calandar after your import.

---

Hope this helps
