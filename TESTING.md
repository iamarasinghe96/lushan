# LushNote – Testing Guide for Claude

Use this when testing with Claude in Chrome. Paste this file's contents into the chat, then share a screenshot and describe what you tried.

## App URL
https://iamarasinghe96.github.io/lushan/

## What the app does
LushNote is a clinical note builder for psychiatrists. It records or transcribes a consultation, then uses AI (Gemini or Groq) to generate a structured clinical progress note.

## Tab bar (top)
| Tab | What it does |
|---|---|
| Generate Report | Main tab. Choose how to input the consultation. |
| Edit & Preview | Edit the generated note fields. Save happens automatically via Firestore. |
| Export | Preview the finished note and copy/email it. |
| Patients | List of all patients with their session history. |

## Generate Report – 4 input buttons
| Button | Expected behaviour |
|---|---|
| Paste Transcript | Opens a text area. You paste a consultation transcript. Clicking Next shows a template picker, then generates the note and goes to Edit. |
| Dictate Note | Opens a microphone modal. Doctor speaks the note solo. On Stop, transcribes via Gemini (or Groq fallback) and generates. |
| Record Session | Opens a microphone modal for a full in-person or telehealth session. Same flow as Dictate after stopping. |
| Create Document | Opens a text/file area. Type or paste any content to structure into a note. |

## Green quota bar (below tabs on Generate screen)
Shows Gemini API usage today. Format: `Gemini today: X/20 generation · Y/20 chat left · Resets in Nh Mm`
- Counts are from localStorage (per device, per day, local midnight reset)
- Goes red when either hits 0
- Shows "Add Groq API to extend usage →" button at all times if no Groq key set
- Hidden entirely when Groq key is active

## Edit tab – top bar buttons
| Button | Expected behaviour |
|---|---|
| Change Template | Re-generates the note using a different template (uses existing transcript). |
| Transcript | Shows/hides the raw transcript section below the note fields. |
| Reassign | Lets you change which patient this note belongs to. |
| New Note | Clears all fields and starts fresh. |

## Chat FAB (green bubble, bottom right)
Opens a panel with two modes:
- **Ask AI**: LushNote AI assistant. Knows the app and your patient list. Uses Gemini Flash Lite (or Groq fallback on quota). Increments 'chat' counter.
- **Live Support**: Escalates to developer via Slack if AI can't answer.

## Patients tab
- Shows list of all patients grouped by name
- Click a patient → detail view with session list
- Each session has a **Delete** button (requires confirm dialog)
- Deleting the last session for a patient removes them from the list

## Settings (separate page: /settings/?tab=api)
- **Gemini API Key**: Required. Free tier at aistudio.google.com/api-keys. 20 req/day.
- **Groq API Key**: Optional. Free at console.groq.com. Used as fallback when Gemini quota hit. Increments are unlimited.

## Known quota behaviour
- Counter updates in localStorage each time an API call is made
- If quota is hit, counter is forced to 20 (0 remaining) immediately
- Counter does NOT reflect usage from other apps using the same key
- If counter shows remaining tries but Gemini returns quota error: counter was stale — try once more to force-sync it to 0

## How to test with Claude in Chrome
1. Paste this entire file into your Claude chat
2. Take a screenshot of the current screen
3. Share it and describe: "I clicked X, expected Y, got Z"
4. Claude can tell you if the behaviour is correct or a bug

## Common test scenarios
- [ ] Paste transcript → pick template → note appears in Edit tab
- [ ] Chat with AI → quota bar decrements chat count by 1
- [ ] Generate note → quota bar decrements generation count by 1
- [ ] Hit quota → bar turns red, shows 0 remaining
- [ ] Add Groq key → bar disappears (unlimited)
- [ ] Delete a session → confirm dialog appears → session removed from list
- [ ] Delete last session for patient → patient removed from list
