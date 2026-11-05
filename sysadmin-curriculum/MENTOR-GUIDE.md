# Mentor guide

For the person running this curriculum with a learner. This file and [MENTOR-NOTES.md](MENTOR-NOTES.md) contain answers and fault injections, so don't hand them to the learner.

## Your job

You are not the instructor and not the answer key. You set direction, ask questions, break things on purpose, and decide when something is understood. The learner does the typing and the reading. Your time per module is about 30 minutes plus a little preparation.

## Ground rules

1. **They drive, you watch.** They type; you ask. Demos don't stick; doing does.
2. **"Where would you look for that?"** is your most-used phrase. When they ask something documented, don't answer. Make them find it in `man`, `--help`, vendor docs, or the logs. This is the main skill gap between help desk and sysadmin.
3. **Break things on purpose.** Troubleshooting is learned from broken systems, not working ones. Use the fault pool.
4. **The homework is the real learning.** The 30-minute review only checks it. If they skip the lab, the review won't do much. Keep each module small enough to finish.
5. **Start each review with 3 minutes of "what did you do, and what got stuck?"**
6. **No shortcuts that hide the problem.** No `chmod 777`, no stopping firewalld to "fix" connectivity. Make them find the block and open the right service or port.
7. **AI is welcome, after they've tried.** See [AI-USE.md](AI-USE.md). They read the error and check the logs first, then ask with context, then verify the answer.
8. **Hold them to the standards.** Review every deliverable against [STANDARDS.md](STANDARDS.md): commit messages, naming, docs, change tickets. Review it the way a peer reviewer would, and ask "what if this fails?"
9. **Be honest and kind.** If a module isn't landing, say so and slow down. Don't sign off on something that isn't understood.

## Before you start with a new learner

- [ ] Prepare or hand over the lab VM ([LAB-SETUP.md](LAB-SETUP.md)), including a clean snapshot.
- [ ] Share the README, AI-USE, and a copy of the progress tracker. Don't share this file or the mentor notes.
- [ ] Have them fill in the starting-point checklist in the tracker. Use it to decide where to start and what to skip.
- [ ] Agree a pace and a regular time: for example, one module a week with one 30-minute review.
- [ ] Settle your organization's AI policy before they use AI on work material (see [AI-USE.md](AI-USE.md)).
- [ ] If your team uses different tools than the lab (backups, monitoring, config management), write down the mapping in a separate environment guide, and decide which modules you'll swap tools in for.

## Running a module

1. **Before the review:** the learner has read, done the lab, done the break-it, and produced the deliverable. You've skimmed the deliverable.
2. **The review (30 minutes):**

   | Time | Activity |
   |------|----------|
   | 0-3 | "What did you do, and what got stuck?" |
   | 3-10 | Walk through the deliverable. Ask them to explain a few lines or decisions. |
   | 10-22 | **Inject a fault** (from [MENTOR-NOTES.md](MENTOR-NOTES.md)) and watch them find it. Don't help until they've used the tools. |
   | 22-28 | Check-your-understanding questions; pick the ones that matter most for this learner. |
   | 28-30 | Sign off, or say what's left. Point at the next module. |

3. **Sign off** when they can explain the module's key ideas without looking anything up and have found a fault using method, not guessing. Update the tracker.

## What to watch for

Signs of real progress:

- Reads the error message before asking you.
- Says "I looked at X and found Y, but I don't understand Z."
- Does the lab without reminders.
- Remembers earlier modules.
- Asks why, not just what to type.
- Can explain a concept back in their own words.
- Diagnoses from logs and tools, not `chmod 777` or stopping the firewall.
- Uses AI to understand: asks "why," checks the answer, catches its mistakes.

Warning signs:

- The same questions come back after a week.
- Wants the exact commands to copy.
- Lab not done, or done by copying a tutorial without understanding.
- Gets stuck on documented things and goes straight to asking.
- Gets defensive when asked "where would you look?"
- Pastes AI output they can't explain, or asks AI before reading the error.

If you see warning signs, slow down, repeat a module's break-it with a different fault, and use the "AI off" check from [AI-USE.md](AI-USE.md).

## Adapting it

- **Fast learners:** skip what the starting-point checklist and the check questions show they know. Add a stretch goal from "Go deeper."
- **Slower pace:** split modules. Module 02 (storage) and 04 (networking) divide naturally in two.
- **A group of co-ops:** run the same modules as cohort sessions, have them pair for the lab, and let them take turns breaking each other's VMs.
- **Your own tools:** swap them in for modules 07, 08, and 11, keep the concepts, and keep the break-it and review structure.
- **Evolve it.** If a command is wrong on your version of Rocky, fix the module. If a break-it never lands, replace it. This is a living document.

## After the modules

If they're progressing, point them at a homelab and a certification such as RHCSA. If they're stalled, be kind and direct: a junior sysadmin or systems technician role, with more time on fundamentals, may fit better. Either way, they should leave with a repository and notes that show real work.
