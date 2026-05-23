# LushNote – Project Bible for Claude Code

## What This Is
Single-file HTML/JS/CSS SPA — a clinical note builder for psychiatrists.
Deployed on GitHub Pages. No build step. No frameworks. No bundler.

## Files
```
index.html               — entire app (auth, generate, edit, export, patients tabs)
settings/index.html      — settings page (separate HTML file, not a SPA route)
email-verified/          — branded email verification landing page
templates-prompts.json   — template prompt definitions fetched at runtime
LushNotes_templates.md   — template source (markdown, not served directly)
```

## Deployment
- **GitHub Pages serves from branch:** `claude/combine-transcription-prompt-6BmeG` (this is the DEFAULT branch — confirmed via GitHub UI)
- **All code changes must be pushed to:** `claude/combine-transcription-prompt-6BmeG`
- The other branches (`claude/continue-previous-work-7o43J`, `claude/fix-empty-text-blocks-xYhZD`) are NOT served by GitHub Pages — pushing to them has no effect on the live site.
- There is no `main` or `master` branch.
- After every push, run `node --check` on the extracted script block to verify no syntax errors before pushing.
- **IMPORTANT:** At the start of every session, confirm the working branch with `git branch --show-current`. If not on `claude/combine-transcription-prompt-6BmeG`, switch to it before making any changes.

## Firebase Project
- **Project ID:** `lush-note`
- **Auth domain:** `lush-note.firebaseapp.com`
- **Config is hardcoded** in `index.html` at `const FB_CONFIG` (line ~1701). Do NOT move it to localStorage.
- Firebase SDK loaded via `<script defer>` tags for all three libs (app, auth, firestore).
- `DOMContentLoaded` is used for init (not `window.onload`) because scripts are deferred.

## Routing
- Hash-based: `#generate`, `#edit`, `#export`, `#patients`, `#history`
- `switchTab(name)` handles tab changes and updates the hash.
- `showView(name)` shows one of: `landing`, `auth`, `pending`, `onboarding`, `app`.
- When showing `landing` or `auth`, the hash is cleared.

## Auth Flow
```
Page load → auth-loading-overlay visible by default (display:flex)
→ DOMContentLoaded → init() → initFirebase() → startAuthListener()
→ onAuthStateChanged(user)
  → no user:  showView('landing')
  → has user: load Firestore profile (up to 3 retries, 8s timeout each)
              → profile missing/new: showView('onboarding')
              → onboardingComplete false: showView('onboarding')
              → success: showView('app') → initApp()
```
The auth-loading-overlay is hidden by both `showView()` and explicitly in the auth listener.
After 12 seconds of loading, a "Reload page" button appears inside the overlay.

## Storage Rules — STRICT, DO NOT CHANGE
| Data | Storage | Reason |
|---|---|---|
| `firebase_config` | `localStorage` | Needed across sessions, not sensitive |
| `groq_api_key` | `sessionStorage` (runtime) + Firestore `users/{uid}.groqApiKey` (persistent) | Security audit decision |
| `lnTemplateUsage` | `localStorage` | Non-sensitive UI preference |
| Patient notes | Firestore `progress_notes` ONLY | Never client storage |
| Patient profiles | Firestore `users/{uid}/patientProfiles/` ONLY | Never client storage |
| User profile | Firestore `users/{uid}` ONLY | Never client storage |

**On sign-out:** All in-memory state cleared: `state.allNotes`, `state.patientProfiles`, `state.lastTranscript`, `_patientIndex`, `currentUser`, etc. `sessionStorage` wipes on tab close automatically.

**`getGroqKey()` function** (line ~1693): Returns `sessionStorage.getItem('groq_api_key')` only. No localStorage fallback — Firestore is the persistent source. On sign-in, profile.groqApiKey is copied to sessionStorage if not already set.

## Firestore Collections
### `progress_notes/{noteId}`
Fields allowed (enforced by `noteValid()` with `hasOnly()`):
`userId, patient, reg_number, date, time, clinician, session_number, attendance, diagnosis, presentation, history, medications, mse, content, scales, risk, referrals, summary, nextsteps, createdAt, updatedAt`
**Adding new fields here requires updating Firestore rules AND noteValid() enforcement.**

### `users/{userId}`
Fields validated by `profileValid()` (NO `hasOnly()` — extra fields are permitted):
`displayName, credentials, email, status, tier, emailPretext, activeWorkplaceId, onboardingComplete, notesMigrated, workplaces, favoriteTemplateIds, customTemplates`
Extra fields written by app (allowed, not in profileValid validation):
`groqApiKey, transcriptPrivacy, recordingDefaults, personalisation`

### `users/{userId}/patientProfiles/{profileId}`
Fields: `displayName, dob, gender` (open write for owner)

### `deletion_feedback/{userId}`
Written once on account deletion. Fields: `userId, email, reasons` (array of strings), `message` (string), `deletedAt` (timestamp).
Read is blocked by rules — view in Firebase console only.

## Firestore Security Rules (verbatim — never write code that violates these)
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function verified() { return request.auth != null; }
    function owns(uid) { return verified() && request.auth.uid == uid; }
    function ownsNote() { return verified() && request.auth.uid == resource.data.userId; }
    function writingOwnNote() { return verified() && request.auth.uid == request.resource.data.userId; }

    function noteValid() {
      let d = request.resource.data;
      return d.userId is string && d.userId.size() <= 128
          && (!('patient'        in d) || (d.patient        is string && d.patient.size()        <= 300))
          && (!('reg_number'     in d) || (d.reg_number     is string && d.reg_number.size()     <= 100))
          && (!('clinician'      in d) || (d.clinician      is string && d.clinician.size()      <= 300))
          && (!('date'           in d) || (d.date           is string && d.date.size()           <= 50))
          && (!('time'           in d) || (d.time           is string && d.time.size()           <= 50))
          && (!('session_number' in d) || (d.session_number is string && d.session_number.size() <= 100))
          && (!('attendance'     in d) || (d.attendance     is string && d.attendance.size()     <= 500))
          && (!('diagnosis'      in d) || (d.diagnosis      is string && d.diagnosis.size()      <= 3000))
          && (!('presentation'   in d) || (d.presentation   is string && d.presentation.size()   <= 8000))
          && (!('history'        in d) || (d.history        is string && d.history.size()        <= 8000))
          && (!('medications'    in d) || (d.medications    is string && d.medications.size()    <= 3000))
          && (!('mse'            in d) || (d.mse            is string && d.mse.size()            <= 5000))
          && (!('content'        in d) || (d.content        is string && d.content.size()        <= 15000))
          && (!('scales'         in d) || (d.scales         is string && d.scales.size()         <= 2000))
          && (!('risk'           in d) || (d.risk           is string && d.risk.size()           <= 5000))
          && (!('referrals'      in d) || (d.referrals      is string && d.referrals.size()      <= 3000))
          && (!('summary'        in d) || (d.summary        is string && d.summary.size()        <= 8000))
          && (!('nextsteps'      in d) || (d.nextsteps      is string && d.nextsteps.size()      <= 5000))
          && request.resource.data.keys().hasOnly([
               'userId','patient','reg_number','date','time','clinician',
               'session_number','attendance','diagnosis','presentation',
               'history','medications','mse','content','scales','risk',
               'referrals','summary','nextsteps','createdAt','updatedAt'
             ]);
    }

    function profileValid() {
      let d = request.resource.data;
      return (!('displayName'        in d) || (d.displayName        is string && d.displayName.size()        <= 200))
          && (!('credentials'        in d) || (d.credentials        is string && d.credentials.size()        <= 200))
          && (!('email'              in d) || (d.email              is string && d.email.size()              <= 300))
          && (!('status'             in d) || (d.status             is string && d.status.size()             <= 50))
          && (!('tier'               in d) || (d.tier               is string && d.tier.size()              <= 50))
          && (!('emailPretext'       in d) || (d.emailPretext       is string && d.emailPretext.size()       <= 1000))
          && (!('activeWorkplaceId'  in d) || (d.activeWorkplaceId  is string && d.activeWorkplaceId.size()  <= 100))
          && (!('onboardingComplete' in d) || (d.onboardingComplete is bool))
          && (!('notesMigrated'      in d) || (d.notesMigrated      is bool))
          && (!('workplaces'         in d) || (d.workplaces         is list   && d.workplaces.size()         <= 30))
          && (!('favoriteTemplateIds'in d) || (d.favoriteTemplateIds is list  && d.favoriteTemplateIds.size() <= 200))
          && (!('customTemplates'    in d) || (d.customTemplates    is list   && d.customTemplates.size()    <= 50));
      // NOTE: groqApiKey, transcriptPrivacy, recordingDefaults, personalisation
      // are NOT in profileValid() but are allowed because there is no hasOnly() restriction.
    }

    match /progress_notes/{noteId} {
      allow get:    if ownsNote();
      allow list:   if verified() && request.query.limit <= 500;
      allow create: if writingOwnNote() && noteValid();
      allow update: if ownsNote() && writingOwnNote() && noteValid()
                    && request.resource.data.userId == resource.data.userId;
      allow delete: if ownsNote();
    }

    match /users/{userId} {
      allow get:    if owns(userId);
      allow create: if owns(userId) && profileValid();
      allow update: if owns(userId) && profileValid();
      allow delete: if owns(userId);

      match /patientProfiles/{profileId} {
        allow read:   if owns(userId);
        allow write:  if owns(userId);
        allow delete: if owns(userId);
      }
    }

    match /deletion_feedback/{docId} {
      allow create: if verified() && request.resource.data.userId == request.auth.uid;
    }

    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

**IMPORTANT — deploy these updated rules to Firebase console.** Two changes from the previous version:
1. `users/{userId}` — `allow delete` changed from `if false` to `if owns(userId)` (required for account deletion)
2. New `deletion_feedback/{docId}` collection — allows users to write their own deletion reason on the way out

Until deployed: the user document deletion will fail (security error), and deletion feedback will silently not save.

## Key State Object
```javascript
const state = {
  currentNoteId: null,         // Firestore doc ID of note being edited
  allNotes: [],                 // loaded from Firestore on history tab
  lastTranscript: null,         // raw transcript text from last recording/paste
  lastTranscriptMode: 'paste',  // 'paste' | 'conversation' | 'dictation' | 'document'
  lastChosenTemplate: null,     // template object chosen for last generation
  patientProfiles: {},          // loaded from Firestore patientProfiles subcollection
  lastRecordingDuration: 0,     // wall-clock seconds of last audio recording
};
```

## Recording / Audio
- `_recStartTime = Date.now()` set when recording starts (wall-clock, accurate during screen lock)
- Timer uses `Math.floor((Date.now() - _recStartTime) / 1000)` — NOT an incrementing counter
- `visibilitychange` listener resyncs timer display when phone unlocks
- `state.lastRecordingDuration` is set in `_stopAndProcess()` before stopping the recorder
- Supports: Record Session (conversation), Dictate Note, Upload Recording (currently hidden)
- Upload Recording is hidden (`display:none`) — code preserved, do NOT delete
- Recording modals (`record-modal`, `dictate-modal`, `upload-modal`) are created dynamically
  via `document.createElement` and `appendChild` — they are NOT in static HTML, this is intentional.

## Two Settings Contexts — Do NOT Confuse
There are two separate settings UIs. They are different elements, different functions, different scope.

### 1. Quick Settings Modal (in-app overlay, line ~1220)
- Element: `#settings-overlay` (`.modal-overlay`, `display:none`)
- Opens via: user menu → settings icon
- Functions: `showSettings()`, `closeSettings()` (~line 1875), `saveSettings()` (~line 1881)
- Contains: Groq API key field (`#s-groq-key`), Firebase config field (`#s-firebase-config`)
- Purpose: quick key management without leaving the app

### 2. Full Settings Page (embedded view, line ~5662)
- Element: `#view-settings` (`display:none;position:fixed;inset:0`)
- Opens via: user menu → "Settings" link
- Functions: `openSettings(tab)` (~line 5217), `closeSettings()` (~line 5264), `settingsNav(tab)`
- Contains: Profile, Workplaces, Templates, Transcript Privacy, Personalisation tabs
- Purpose: full account management

**IMPORTANT**: Both contexts define `closeSettings()` — the one at line ~1875 is for the modal,
the one at ~5264 is for the full view. They do not conflict because only one is active at a time,
but do NOT merge or rename them.

## Workplace Management — KNOWN BROKEN
The full settings page renders workplaces via `renderAccountWorkplaces()`. This function
generates onclick handlers that call:
- `asAddWorkplace()`, `asSetActive(wpId)`, `asToggleEdit(wpId)`, `asDeleteWorkplace(wpId)`, `asSaveWpEdit(wpId)`

**These functions are NOT defined in index.html.** Clicking workplace buttons throws
`ReferenceError`. The equivalent functions (`wpAdd`, `wpToggleEdit`, `wpSave`, `wpDelete`)
exist in `settings/index.html` but in a different execution context.

**Do NOT silently add stubs.** When implementing workplace management in index.html,
implement the full CRUD logic matching the settings page behaviour.

## API Status Bar — KNOWN MISSING ELEMENTS
`updateApiStatusBar()` (~line 1920) references these DOM element IDs that do NOT exist in HTML:
- `#groq-dot`, `#groq-status-text` — Groq API key status indicator
- `#fb-dot`, `#fb-status-text` — Firebase connection status indicator

The function has null-checks so it doesn't crash, but the status indicators are invisible.
If adding a status bar to the UI, use exactly these IDs and the existing function will populate them.

## Dead Code (do not add more of the same)
- `closeAccountSettings()` (~line 5502) — alias for `closeSettings()`, never called, do NOT use
- `acctTab()` (~line 5505) — empty stub, never called, do NOT use
Both exist as scaffolding remnants. Remove when refactoring that section.

## Safari / iOS Compatibility Rules
- NO lookbehind regex (`(?<=...)`) — crashes Safari < iOS 16.4. Use `/[.!?]+\s+/` instead.
- NO `??` nullish coalescing on older Safari — use ternary `(x !== null && x !== undefined ? x : y)`
- Optional chaining `?.` is acceptable — supported Safari 13.1+ / iOS 13.4+ (2020)
- Firebase `defer` scripts + `DOMContentLoaded` required to avoid blank page on iPad

## API Keys
- **Groq API key**: User provides their own. No shared/hardcoded key exists.
  - Saved to: `sessionStorage` (runtime) + `users/{uid}.groqApiKey` in Firestore (persistent)
  - Loaded from: Firestore profile on sign-in → copied to sessionStorage
  - `getGroqKey()` reads sessionStorage only
- **Firebase config**: Hardcoded in `FB_CONFIG` constant in index.html. Do NOT accept from user input or move to localStorage.

## Tab Bar
- Tabs: Generate, Edit, Export, History, Patients
- `.tabs-wrap` has CSS edge fade (::before / ::after gradients) for scroll indicators
- `switchTab(name)` auto-scrolls active tab into view with `scrollIntoView({inline:'center'})`
- Hash mapping: `#generate`, `#edit`, `#export`, `#history`, `#patients`

## Critical Flows (verified working by audit)
- **Paste & Generate**: `pasteAndGenerate()` → `transcriptConfirmModal()` → `showTemplatePickerModal()` → `callGroq()` → `populateFields()` → `switchTab('edit')`
- **Record Session**: `openRecordModal()` → `startInPersonRecord()` → `_startRecorderForSession()` → `_stopAndProcess()` → `processAudioAndGenerate()` → `switchTab('edit')`
- **Dictate**: `openDictateModal()` → `startDictateRecording()` → `_stopAndProcess('dictate')` → `processAudioAndGenerate()` → `switchTab('edit')`
- **History**: `switchTab('history')` → `loadHistoryData()` → `fbLoadHistory()` → `renderPatients()`
- **Patient CRUD**: `openAddPatientModal()` → `pmSave()` → `savePatientProfile()`
- After audio generation: stats card (`#session-stats-card`) and raw transcript (`#raw-transcript-section`) are shown in Edit tab

## Consistency Rule — Fix All Similar Instances
When the user reports a bug or requests a change, always scan the entire codebase for other places where the same pattern exists and apply the fix consistently across all of them — **with the user's explicit consent before making the sweeping change**.

Workflow:
1. Fix the reported instance.
2. Identify all other similar instances (e.g. same missing guard, same missing save call, same missing field clear).
3. Tell the user what other instances were found and what the consistent fix would be.
4. Wait for the user to confirm before applying the broader fix.

Example from this project: when auto-save after generation was missing from the recording flow, the same omission existed in the Create from Document flow. The fix was applied to both only after the user confirmed.

## DO NOT rules
- Do NOT store patient data in localStorage or sessionStorage
- Do NOT add hardcoded Groq API keys
- Do NOT use lookbehind regex
- Do NOT move `firebase_config` out of `FB_CONFIG` constant
- Do NOT add fields to `progress_notes` without updating Firestore `noteValid()` rules
- Do NOT push to any branch other than `claude/continue-previous-work-7o43J`
- Do NOT create a `main` or `master` branch
- Do NOT add emoji to UI unless user explicitly asks
- Do NOT add comments to code unless the WHY is non-obvious
- Do NOT create new files unless explicitly required
- Do NOT add `console.log` debug statements
- Do NOT implement defensive code (timeouts, fallbacks, retries) without understanding the real failure mode
- Do NOT add functions without verifying the element IDs they reference exist in the HTML
- Do NOT add HTML element IDs without verifying any JS that references them is correct
- Do NOT make changes to multiple systems at once — one concern per commit
- ALWAYS run `node --check` on the extracted script after editing JS to catch syntax errors before pushing

## TEMPORARY DEBUG CODE — REMOVE BEFORE PRODUCTION

`debugLog(event, data)` writes to `users/{uid}._debugLog` via `firebase.firestore.FieldValue.arrayUnion`.
Allowed by Firestore rules because `profileValid()` has no `hasOnly()`.

**Remove ALL of the following before production:**
1. `function debugLog(...)` definition (~line 4452) — marked `// TEMP DEBUG`
2. `debugLog('rec_start', ...)` call in `_startRecorderForSession` (~line 4946)
3. `debugLog('rec_heartbeat', ...)` call inside `_recTimer` setInterval in `_startRecorderForSession`
4. `debugLog('rec_start', ...)` call in `startDictateRecording` (~line 4773)
5. `debugLog('rec_heartbeat', ...)` call inside `_recTimer` setInterval in `startDictateRecording`
6. `debugLog('rec_stop', ...)` call in `_stopAndProcess` (~line 4957)
7. `debugLog('transcribe_start', ...)` call in `transcribeAudio` (~line 4477)
8. `debugLog('transcribe_error', ...)` call in `transcribeAudio` error branch
9. `debugLog('transcribe_done', ...)` calls (×2) after `transcribeAudio` response parse
10. `debugLog('generate_done', ...)` call in `processAudioAndGenerate` success path
11. `debugLog('generate_error', ...)` call in `processAudioAndGenerate` catch block

After removing: delete the `_debugLog` field from the Firestore user document in Firebase console.

**Events logged:** `rec_start` · `rec_heartbeat` (every 5 min) · `rec_stop` · `transcribe_start` · `transcribe_done` · `transcribe_error` · `generate_done` · `generate_error`
