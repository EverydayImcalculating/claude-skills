---
name: daily-update
description: >
  Summarize the current Claude Code session into a Thai-language team daily-update
  bullet block (casual Thai narrative + English technical terms), print it for
  copy-paste, and append it to a dated log file at ~/Opsta/logs/daily/YYYY-MM-DD.md.
  Use this whenever the user wants to write up what they did today for the team:
  triggers include "daily update", "สรุปงานวันนี้", "เขียน update", "team update",
  "daily standup", "what did I do today", "สรุปงาน", "อัปเดตงาน", or any end-of-day /
  end-of-session wrap-up of work done. Trigger even if the user doesn't name the skill —
  any request to turn this session's work into a team status report should use it.
compatibility: Requires write access to ~/Opsta/logs/daily/ and the `date` command.
---

# Daily Update (Thai team status)

Turn the work done in **this session** into a short Thai daily-update — the kind the
team posts in chat each day. The reader is a teammate skimming on their phone: they
want *what changed and what's stuck*, not the debugging play-by-play.

## Why this format

The team writes updates as a flat bullet list in casual Thai, keeping technical nouns in
English (ArgoCD, Vault, deploy, A2A). Each bullet is one unit of work stated as an
*outcome* — fixed, deployed, still broken, waiting, consulted. The value is in
collapsing a long session into a few honest status lines, not in completeness.

## Output format

Print this block to chat (copy-paste ready), and write the same bullets to the log file:

```
- Updates: (YYYY-MM-DD)
- <bullet>
- <bullet>
  - <nested detail — only when an item needs a why / trade-off>
- <bullet>
```

**Per-bullet shape:** `[งาน/ระบบ] [ทำอะไร, กริยาอดีต] [ผล/สถานะ][, อุปสรรค ถ้ามี]. [ก้าวถัดไป ถ้ามี]`

Write in first-person-implied casual Thai (the senior's register below). Keep tech nouns
in English. One bullet = one item of work.

## Status vocabulary

Pick the phrasing that matches the real outcome — don't dress up a stuck item as done:

| Outcome | Thai phrasing |
|---|---|
| Fixed / done | `…แก้ไขเรียบร้อยแล้ว` / `…แก้แล้ว` / `…เสร็จแล้ว` |
| Deployed | `deploy <x> แล้ว` |
| Now working / visible | `<x> ขึ้นแล้ว` / `<x> ใช้งานได้แล้ว` |
| Still broken / not showing | `<x> ยังไม่ขึ้น` / `<x> ยังพังอยู่` (+ `กำลังหาสาเหตุ`) |
| Blocked on others | `รอ <x>` / `ยังไม่มีการตอบกลับจาก <x>` |
| Investigated, tentative | `ตรวจสอบ <x> แล้ว ดูน่าจะ…` |
| Consult / coordinate | `consult <x> เรื่อง <y>` |
| Partial progress | `ได้ประมาณ X% ยังไม่… ` |

## What to keep, what to drop

The session transcript is far more detailed than a team update should be. Compress it:

- **Keep:** things done / fixed / deployed, blockers you hit, status changes, decisions
  made, and consults/coordination with people. These are what the team acts on.
- **Drop:** dead-end exploration that went nowhere, low-level tool mechanics, exact
  commands, file paths, and **secrets / tokens**. Collapse a long debug chain into the
  one team-facing outcome ("เจอบั๊ก X หลังย้ายมา Y, แก้แล้ว") — the team cares that it's
  fixed, not how.
- **Nest** (indented sub-bullets, ≤5) only when one item genuinely needs a *why* or a
  *trade-off* to make sense — e.g. explaining that A2A and ADK can't be used together.
  Don't nest routine items.
- **Order** naturally: roughly done → in-progress → blocked/waiting. Don't force category
  headers; the senior's updates mix freely and read fine.

## Procedure

1. Review what actually happened in this session (your own context). Identify the
   distinct units of work and their real outcomes.
2. Draft the bullets per the rules above. Be honest about status.
3. Print the block to chat.
4. Append to the log file, then tell the user the path:

```bash
mkdir -p ~/Opsta/logs/daily
DATE=$(date +%F); TIME=$(date +%H:%M)
FILE=~/Opsta/logs/daily/$DATE.md
[ -f "$FILE" ] || printf '# Daily Update — %s\n' "$DATE" > "$FILE"
{
  printf '\n## Session %s\n\n' "$TIME"
  cat <<'BULLETS'
- Updates:
- <paste the generated bullets here>
BULLETS
} >> "$FILE"
```

Replace the heredoc body with the real bullets before running. Current-session-only means
the user may run this several times a day — each run appends its own `## Session HH:MM`
section under the same daily header (header written once).

## Gold reference (match this register)

This is a real senior's update across several days. Match its casualness, English-term
placement, honesty about stuck items, and the occasional nested deep-dive — do **not**
copy its content.

```
- Updates:
- เจอบั๊ก extract email content ไม่ได้หลังจากย้ายมา GEAP แก้ไขเรียบร้อยแล้ว
- Topology ยังไม่ขึ้น ปรากฏว่าน่าจะต้องไป register app hub ก่อนมั้ง ลอง register ไปแล้ว ขึ้น Topology ใน app hub แล้ว เดี๋ยวลองรอดูฝั่ง GEAP ว่าจะขึ้นไหม
- gateway early access ยังไม่มีการตอบกลับจากที่ลงทะเบียน
- consult CCP เรื่อง load test

- Updates:
- deploy GEAP version เข้าไปแทน Cloud Run แล้ว ให้ support ทดสอบใช้งานตามปกติ
- Sessions view ขึ้นแล้ว
- memory bank น่าจะใช้ได้ แต่ต้องรอข้อมูลจริงวิ่งเข้าก่อน
- Topology ยังไม่ขึ้น กำลังหาสาเหตุ
- Evaluation ไม่สามารถทำ Experiments และ Online Metrics ได้ ยังพังทังคู่อยู่

- Updates:
- Deploy orchestrator + kb บน GEAP แล้ว
- ทั้งสองตัวใช้งาน memory bank และ session view ได้แล้ว
- เจอปัญหาหลายอย่างที่กูเกิลบอกไม่เคลียร์หรือไม่ได้บอก
  - A2A lib สร้างขึ้นมาจากพื้นฐาน ADK ก็จริง แต่ไม่เหมือนกัน และใช้แทนกันไม่ได้
  - ถ้าอยากใช้ A2A จะต้องยอมเสียบางส่วนของ ADK / ถ้าอยากใช้ส่วนนั้น ก็ต้องไม่ใช้ A2A
  - ตอนนี้เลยทำ orchestrator ด้วย ADK ทำ KB ด้วย A2A
- ตอนนี้ได้ประมาณ 60-70% ยังไม่สามารถเอามาแทนตัว Cloud Run ของเดิมได้
```

## Notes

- Casual Thai + English tech nouns. Don't translate established tool names.
- Never put secrets, tokens, or full file paths in the update — it goes to a team channel.
- Bullets are team-facing outcomes; if an item wouldn't matter to a teammate, drop it.
