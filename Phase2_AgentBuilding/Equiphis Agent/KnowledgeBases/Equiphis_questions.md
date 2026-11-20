# NetVision Equipment History Copilot
## 25 Validation Questions and Expected Answers

### Purpose
Validate that the Copilot agent can accurately interrogate NetVision Equipment History, calculate equipment-performance measures, reconstruct incidents, identify recurring problems, and explain the evidence behind its answers.

The agent should answer what happened, when it happened, how long it lasted, whether it happened before, and what patterns exist. It should not provide step-by-step equipment repair, maintenance, calibration, adjustment, or troubleshooting instructions.

Replace `XXXXX`, `DATE`, and time placeholders with known examples from the validation dataset. Every answer should identify the records, timestamps, status transitions, or comments used to support the response.

---

### Validation Questions

| # | Topic | Question | Expected Answer |
| :--- | :--- | :--- | :--- |
| 1 | Current Equipment Status | What is the current status of equipment XXXXX? | Identify the latest valid equipment status, when that status began, and the latest NetVision Equipment History event supporting the answer. |
| 2 | Equipment History | Show me the equipment history for XXXXX for the last 24 hours. | Provide a chronological list of status changes and significant events, including timestamps, event/status type, and available comments. |
| 3 | Historical Status | What was the status of XXXXX at HH:MM on DATE? | Identify the status in effect at the requested time, including the event that established the status and the subsequent event that ended it. |
| 4 | Last Down Event | When did XXXXX last go down? | Provide the timestamp of the latest qualifying Down transition, previous status, resulting status, and associated comments or events. |
| 5 | Last Recovery | When did XXXXX last return to Production Up? | Provide the Production Up timestamp, preceding equipment status, duration of the preceding non-production interval, and supporting events. |
| 6 | Unscheduled Downtime | How much unscheduled downtime did XXXXX have during the last 30 days? | Provide total unscheduled downtime, number of qualifying downtime incidents, date range, and the downtime intervals used in the calculation. |
| 7 | Longest Downtime Event | What was the longest downtime event for XXXXX during the last 30 days? | Identify the start time, end time, duration, downtime classification, relevant comments/events, and recovery transition. |
| 8 | Equipment With the Most Downtime | Which 10 tools had the most unscheduled downtime last month? | Rank the equipment by total unscheduled downtime and show downtime duration and event count for each tool. |
| 9 | Equipment Availability | What was the availability of XXXXX last month? | Provide the availability percentage, Production Up duration, qualifying unavailable duration, date range, and calculation method. |
| 10 | Lowest-Availability Equipment | Which tools had the lowest availability last month? | Rank equipment by availability percentage and provide the production and unavailable durations supporting each calculation. |
| 11 | Number of Down Events | How many times did XXXXX go down during the last 30 days? | Provide the count and corresponding Down timestamps. Events belonging to the same continuous downtime incident should not be incorrectly counted as separate failures. |
| 12 | Mean Time to Repair | What was the MTTR for XXXXX during the last 90 days? | Provide MTTR, number of qualifying incidents, total repair/downtime duration, date range, and the definition used for determining the beginning and end of each interval. |
| 13 | Mean Time Between Failures | What was the MTBF for XXXXX during the last 90 days? | Provide MTBF, qualifying failure count, Production Up time between failures, date range, and calculation method. |
| 14 | Repeat Down Events | Did XXXXX go down again within 24 hours after returning to Production Up? | Identify each applicable recovery timestamp, subsequent Down timestamp, and elapsed time between recovery and the next failure. |
| 15 | Rapid Repeat Failures | Which tools experienced another Down event within four hours of returning to Production Up during the last 30 days? | Identify each tool, its recovery timestamp, subsequent Down timestamp, and elapsed time. |
| 16 | Scheduled Versus Unscheduled Downtime | How much of XXXXX's downtime last month was scheduled versus unscheduled? | Provide scheduled downtime, unscheduled downtime, total downtime, and the percentage attributable to each classification. |
| 17 | Preventive Maintenance History | When was the last PM performed on XXXXX? | Identify the start and end of the most recent qualifying PM interval, total PM duration, preceding equipment status, subsequent status, and relevant comments. |
| 18 | Post-PM Equipment Performance | Did XXXXX experience an unscheduled Down event within 24 hours after its last PM? | Answer yes or no and provide the PM completion time, subsequent Down time if applicable, elapsed time, and supporting events.<br><br>Do not claim that the PM caused the subsequent failure unless that relationship is explicitly documented. |
| 19 | What Happened Before the Failure? | What happened immediately before XXXXX went down? | Provide the chronological status changes, events, alarms, and comments occurring immediately before the Down transition.<br><br>Clearly distinguish sequence from causation. An event occurring before a failure does not necessarily mean that it caused the failure. |
| 20 | What Happened While the Tool Was Down? | What happened while XXXXX was down? | Construct a chronological timeline from the initial Down event through recovery, including recorded status changes, comments, alarms/events, maintenance-related events, qualification activity, FabGuard activity, and SECS/GEM events when present.<br><br>Do not provide instructions for repairing the equipment. |
| 21 | Has This Problem Happened Before? | Has XXXXX experienced this same problem before? | Identify similar prior occurrences based on equipment events, alarms, or comments.<br><br>For every matching occurrence, provide the date/time, relevant symptoms or comments, associated equipment status, and downtime duration when available.<br><br>Explain why the events were considered similar. |
| 22 | What Does the History Say Caused the Failure? | What does the equipment history say about the cause of XXXXX's last failure? | Report a cause only when the NetVision history explicitly documents one.<br><br>If no confirmed cause is recorded, respond accordingly and summarize the events, alarms, and comments that were recorded around the failure.<br><br>The agent must not convert a correlation, sequence of events, or technician comment into a confirmed root cause unless supported by the source data. |
| 23 | Similar Problems on Other Equipment | Have other tools of the same type experienced similar problems? | Identify comparable equipment with similar events, alarms, status transitions, or comments.<br><br>Provide equipment identifier, occurrence dates, downtime duration, relevant comments/events, and the similarities observed.<br><br>Do not assume that similar symptoms necessarily have the same root cause. |
| 24 | Recurring Equipment Problems | What recurring problems can you identify for XXXXX during the last 90 days? | Identify repeated alarms, events, status sequences, or comment patterns. For each pattern, provide: description of the pattern, number of occurrences, occurrence dates, associated downtime, relevant comments/events, and supporting source records.<br><br>Clearly distinguish a recurring pattern from a confirmed recurring root cause. |
| 25 | Incident and Problem-History Summary | Summarize the history of XXXXX's problem between DATE/TIME and DATE/TIME, including whether it has happened before. | Produce a concise chronological incident narrative showing: equipment status before the incident; initial failure/down event; significant events and alarms; equipment comments; FabGuard or SECS/GEM activity when present; maintenance-related events recorded in NetVision; qualification activity; recovery to Production Up; total downtime; repeat failures following recovery; similar historical occurrences; and documented cause, if available.<br><br>End with a short summary separating **Documented Fact**, **Calculated Result**, **Observed Pattern**, and **Unknown**.<br><br>Do not provide instructions describing how to repair, adjust, calibrate, tune, rebuild, or otherwise physically service the equipment. |

---

### Validation Scoring

| Score | Criteria |
| :--- | :--- |
| **2 - Pass** | The answer is correct, contains the appropriate calculations and supporting timestamps/events, and makes no unsupported claims. |
| **1 - Partial** | The answer is directionally correct but incomplete, lacks important supporting evidence, uses an ambiguous classification, or does not adequately explain a calculation. |
| **0 - Fail** | The answer is incorrect, uses an incorrect calculation, fabricates information, presents an unsupported causal conclusion, or cannot answer a question that is supported by the connected data. |

---

### Required Evidence Standard
The agent should distinguish among five different types of information:

* **Recorded Fact:** Information explicitly contained in NetVision Equipment History.
* **Calculated Result:** A result derived from source events using a defined and repeatable calculation, such as downtime, availability, MTTR, or MTBF.
* **Observed Pattern:** Multiple source records exhibiting similar events, states, alarms, comments, or sequences.
* **Interpretation:** A possible explanation suggested by the evidence but not explicitly established by the source data. This must be clearly labeled as an interpretation.
* **Unavailable Information:** Information that cannot be established using the data available to the agent.

---

### Functional Boundary

**The NetVision Equipment History Copilot should help someone answer:**
* What happened?
* When did it happen?
* What happened immediately before and after it?
* How long did it last?
* Has it happened before?
* Have similar tools experienced it?
* What did the equipment history record about the problem?
* What patterns exist across incidents?

**It should not answer:**
* How do I repair it?
* How do I rebuild it?
* How do I adjust it?
* How do I calibrate it?
* How do I tune it?

Those activities should remain governed by Polar's approved equipment procedures, maintenance documentation, and applicable engineering controls.