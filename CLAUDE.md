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
- **GitHub Pages serves from branch:** `claude/continue-previous-work-7o43J`
- **Working branch (push here):** `claude/continue-previous-work-7o43J`
- All commits must be pushed to `claude/continue-previous-work-7o43J` to go live.
- There is no `main` or `master` branch.

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
      allow delete: if false;

      match /patientProfiles/{profileId} {
        allow read:   if owns(userId);
        allow write:  if owns(userId);
        allow delete: if owns(userId);
      }
    }

    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

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

## Safari / iOS Compatibility Rules
- NO lookbehind regex (`(?<=...)`) — crashes Safari < iOS 16.4. Use `/[.!?]+\s+/` instead.
- NO `??` nullish coalescing on older Safari — use ternary `(x !== null && x !== undefined ? x : y)`
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
