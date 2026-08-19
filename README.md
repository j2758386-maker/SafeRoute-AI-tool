# SafeRoute AI — Build Documentation

A working journey-safety app: real GPS tracking, deterministic route-deviation and
stop detection, a safety-check → escalation workflow, AI-assisted community hazard
reporting, a community safety map, and an explainable risk engine.

**Open `saferoute-ai.html` in a mobile-width browser window (or on a phone) to run it.**
It's a single self-contained page — no build step, no install.

---

## 1. What's real vs. what's substituted

This was built inside a sandboxed environment with **no outbound network access for
provisioning services** — no Firebase project, no Google Maps API key, no Gemini API
key could actually be created or called from here. Rather than fake those integrations
with hardcoded UI, every piece of *logic* is real and working; only the specific
**cloud vendor** for three integrations is substituted, and each is called out below,
per the brief's own allowance: *"If a requested external service cannot be implemented
because credentials or platform restrictions are unavailable, implement the closest
safe alternative and clearly document what is required to enable the real service."*

| Requirement | What's implemented | Why | To go to production |
|---|---|---|---|
| Backend (Firebase Auth + Firestore) | A persistent key-value store (`window.storage`), with a `personal` scope for the user's own profile/journeys and a `shared` scope for community hazard reports — genuinely persists across reloads | No Firebase project could be provisioned here | Swap `storeGet/storeSet` for Firestore reads/writes; swap the local `uid()`/profile object for Firebase Auth's user object. Schema (§5) maps 1:1 onto Firestore collections. |
| Maps (Google Maps Platform) | A canvas-drawn **schematic thread map**: real GPS coordinates plotted on a straight-line projection, with the planned route, travelled path, current position, and nearby hazards — genuinely computed, not decorative | No Google Maps API key available | Replace `drawThreadMap()` with the Google Maps JS SDK; replace the straight-line `expectedRoute` with a real Directions API polyline. |
| AI classification (Gemini) | A real, working AI call — **Claude (Sonnet)**, via the same in-artifact API mechanism this environment exposes — given the hazard photo + text and asked for the exact JSON contract specified in the brief | No Gemini API key available | Point `classifyHazardWithAI()` at the Gemini API endpoint instead of `api.anthropic.com/v1/messages`. The system prompt, output contract, and validation/fallback logic don't need to change — only the endpoint/request shape. |
| SMS/push to trusted contacts | Real backend alert *event* is created and stored (§6), plus a real browser `Notification` if the user has granted permission | No Twilio/SMS credentials or FCM project available | Add a Cloud Function (or any backend) that watches for new escalation events and sends via Twilio SMS / Firebase Cloud Messaging. The event payload already contains everything a real send needs (name, location, destination, time, reason, map link). |
| Destination search (Places/Geocoding) | Manual lat/lng entry, or "quick pick" sample destinations computed as a real bearing/distance offset from the user's **actual** current GPS position | No Places/Geocoding key available | Replace the destination form with the Places Autocomplete widget; feed the resolved coordinates into the same journey object. |

Everything else — account/profile creation, trusted-contact management, live geolocation
tracking (`navigator.geolocation`), deviation math, stop detection, the safety-check
countdown and escalation state machine, hazard report storage, duplicate detection,
route safety context, and the Phase 3 risk engine — is real, deterministic application
logic with no mocking.

---

## 2. Demo Mode

Toggle **Settings → Demo Mode** to reproduce the full workflow without a real multi-hour
walk. Everything Demo Mode touches is labeled **DEMO MODE** in the UI. It does not change
the architecture — it only swaps the *source* of location updates:

1. Onboard (real name, real contact, real location permission).
2. Turn on Demo Mode in Settings.
3. Start Journey → pick a quick-pick destination → Start journey. Demo Mode animates a
   synthetic path from origin toward the destination every 1.5s (still written through
   the same `onLocationUpdate` pipeline real GPS points use).
4. On the Active Journey screen, **Simulate deviation** injects one synthetic off-route
   point — after `deviationStreakNeeded` such readings, the real safety-check workflow
   fires.
5. Don't tap "I'm OK" / "I need help" — after the configured timeout, the real
   auto-escalation path runs and a real escalation event is stored (visible on the
   Active Journey screen's safety-check history).
6. Go to **Report** → **Use sample hazard photo (DEMO)** → Analyze with AI — this runs
   the real Claude-based classification call end-to-end.
7. Review/edit the AI's output → Confirm & submit → see it on the **Map** tab.
8. Start a second journey whose quick-pick destination is near the hazard you just
   filed → the route-preview card shows the real "This route has N recent community
   reports…" context line.
9. The **Live risk assessment** card on the Active Journey screen is the Phase 3 engine,
   updating in real time from deviation/stop/hazard indicators, with its factor list.

Lower `Settings → Safety-check response window` to a few seconds if you want the
no-response escalation to fire quickly during a demo.

---

## 3. Architecture

```
┌─────────────────────────────┐        ┌───────────────────────────┐
│         Client (SPA)        │        │   Persistent storage      │
│  onboarding / home / journey│◄──────►│  personal: profile,       │
│  / hazard report / map /    │        │            journeys       │
│  history / settings         │        │  shared:   hazardReports  │
└───────────┬──────────────────┘        └───────────────────────────┘
            │
   ┌────────┼─────────────────────────────────────────┐
   │        │                                          │
   ▼        ▼                                          ▼
navigator.geolocation                        AI classification call
(watchPosition / getCurrentPosition)         (image + text → structured
   │                                          JSON hazard record)
   ▼
Deterministic engine (no LLM):
  - distanceToRoute()      route deviation math
  - evaluateJourneyTick()  deviation streak + stop timer
  - triggerSafetyCheck()   safety-check state machine
  - escalateToContacts()   trusted-contact alert event + notification
  - computeRisk()          Phase 3 explainable risk score
```

Per the brief's AI-architecture guidance: the LLM is only ever used for hazard
image/text classification. GPS math, deviation detection, timers, escalation
triggering, authentication/authorization, and the risk engine are all ordinary,
auditable software logic — never LLM calls — for reliability and explainability.

---

## 4. Setup instructions (as shipped)

No build step. Open `saferoute-ai.html` in a browser (Chrome recommended for
`geolocation`/`Notification` support). Grant location permission when prompted during
onboarding. That's it — the app is fully self-contained and uses the platform's
built-in persistent storage.

### Environment variables (for the production swap described in §1)
```
# Firebase
FIREBASE_API_KEY=
FIREBASE_AUTH_DOMAIN=
FIREBASE_PROJECT_ID=
FIREBASE_APP_ID=

# Google Maps Platform
GOOGLE_MAPS_API_KEY=        # Maps JS API + Directions API enabled

# Gemini
GEMINI_API_KEY=
GEMINI_MODEL=gemini-2.5-flash   # or preferred vision-capable model

# SMS / push (pick one)
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_FROM_NUMBER=
# — or —
FCM_SERVER_KEY=
```

### Firebase configuration instructions
1. Create a Firebase project → enable **Authentication** (Email/Password or phone) and
   **Cloud Firestore**.
2. Create the collections in §5 (Firestore will create them lazily on first write —
   no manual schema step needed beyond security rules, §7).
3. Deploy the security rules in §7 with `firebase deploy --only firestore:rules`.
4. Add a Cloud Function triggered on new documents in `safetyEvents` where
   `status == "escalated"` to send the real SMS/push using the payload already present
   on the event (see §1 table).

### Google Maps configuration instructions
1. Enable **Maps JavaScript API** and **Directions API** in Google Cloud Console.
2. Restrict the API key to your app's domain.
3. Replace `drawThreadMap()`'s canvas rendering with a `google.maps.Map` instance;
   replace the straight-line `expectedRoute` with the polyline from a
   `DirectionsService` request between origin and destination.

### Gemini configuration instructions
1. Get a Gemini API key from Google AI Studio / Vertex AI.
2. In `classifyHazardWithAI()`, replace the `fetch('https://api.anthropic.com/v1/messages', ...)`
   call with a call to the Gemini `generateContent` endpoint, keeping the same
   system-prompt contract (category/description/severity/confidence JSON) and the same
   validation/fallback code that follows it.
3. Never call Gemini (or any LLM) from the deviation/stop/escalation path — those stay
   deterministic per §3.

---

## 5. Database schema

```
users
  id, name, authIdentifier, preferences { deviationThresholdM, deviationStreakNeeded,
    stopRadiusM, stopTimeoutMin, safetyCheckTimeoutSec, reEscalationCooldownMin,
    demoMode }, createdAt

trustedContacts
  id, userId, name, relationship, contact (phone/email), enabled

journeys
  id, userId, origin {lat,lng}, destination {lat,lng,label}, expectedRoute [{lat,lng}],
  startedAt, endedAt, status (active|completed|escalated)

locationEvents            -- embedded as journeys[].path in this build to minimize
  journeyId, timestamp, latitude, longitude    writes; split into its own collection
                                                 in Firestore for scale/retention control

safetyEvents
  id, journeyId, type (deviation|stop|manual), detail, severity, timestamp,
  status (pending|resolved|escalated), userResponse (ok|no_response|help)

escalations                -- the trusted-contact alert record
  id, journeyId, safetyEventId, userName, location, destination, time, reason,
  mapRef, notifiedContacts [{contactId, name, channel, sentAt}]

hazardReports
  id, reporterId, location {lat,lng}, category, description, severity, confidence,
  imageRef, createdAt, status (confirmed), relatedTo (id | null)
```

**Retention:** `journeys[].path` is capped (oldest points dropped past 2000 per
journey) and location is only ever stored for the user's own journeys or the
coordinates a user explicitly attaches to a hazard report — no background location
history is kept beyond an active journey.

---

## 6. Trusted-contact alert event (what actually gets created)

On escalation, a real record is written containing user name, current location,
journey destination, time, reason for escalation, and a map reference link — never
just an "Alert sent" toast with nothing behind it. In this build the *delivery*
channel is an in-app record plus a browser `Notification`; §1/§4 describe wiring a
real SMS/FCM send off the same event.

---

## 7. Security (Firestore rules sketch for production)

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, update: if request.auth.uid == userId;
      allow create: if request.auth.uid == userId;
    }
    match /journeys/{journeyId} {
      allow read, write: if request.auth.uid == resource.data.userId
        || (request.method == 'create' && request.auth.uid == request.resource.data.userId);
    }
    match /trustedContacts/{contactId} {
      allow read, write: if request.auth.uid == resource.data.userId;
    }
    match /hazardReports/{reportId} {
      allow read: if true;                 // community-visible, no personal data exposed
      allow create: if request.auth != null;
      allow update, delete: if request.auth.uid == resource.data.reporterId;
    }
  }
}
```
A user can never read or write another user's `journeys`/`trustedContacts`/location
data; `hazardReports` are readable by anyone (that's the point of the community map)
but never expose the reporter's identity in the UI, and only the original reporter can
edit/delete their own report.

In this build (no real auth backend available), the equivalent boundary is enforced by
storage scoping: `profile`/`journeys` use **personal** storage (isolated per user
session), and only `hazardReports` uses **shared** storage.

---

## 8. Error handling implemented

- Geolocation permission denied / unavailable / timeout → explicit message, no crash,
  retry button.
- No photo *and* no text on hazard report → Analyze button stays disabled.
- Gemini/Claude call fails, times out, or returns empty/invalid JSON → caught, surfaced
  with a "classify manually instead" fallback so the report isn't blocked.
- AI response validated field-by-field (category/severity whitelist, confidence
  clamped 0–1) before ever being shown or stored — "don't blindly trust AI output" per
  the brief.
- Storage write failures are caught and toasted rather than silently losing data.
- Duplicate/near-duplicate hazard reports (same category, ≤120m, ≤48h) are linked via
  `relatedTo` instead of listed as independent hazards.
- Repeated safety-check spam is prevented: only one pending check at a time, plus a
  cooldown after a resolved check.

---

## 9. Testing checklist

Manual test script (all reproducible via Demo Mode):

1. **Normal journey** — start, a few on-route location updates, end journey → status
   `completed`, no safety events.
2. **Small GPS deviation** — a single reading past the threshold does *not* trigger a
   check (debounced by `deviationStreakNeeded`).
3. **Significant route deviation** — sustained deviation triggers the safety-check
   sheet with the exact required copy.
4. **Unexpected stop** — stationary past `stopTimeoutMin` triggers a check.
5. **User confirms safety** — "I'm OK" resolves the event, no contact alert, journey
   continues.
6. **User requests help** — "I need help" (from the check, or "I need help now")
   immediately escalates.
7. **User doesn't respond** — countdown reaches zero → auto-escalates.
8. **Duplicate hazard report** — submit two similar reports near each other within 48h
   → second one links via `relatedTo` instead of appearing as a separate pin.
9. **Invalid/unrelated hazard image** — AI falls back to `category: other`, lower
   confidence; still reviewable/editable before submit.
10. **AI classification failure** — simulate by going offline before "Analyze with AI"
    → error surfaced, manual classification path offered.
11. **Network loss** — storage writes fail gracefully with a toast, app doesn't crash.
12. **Location permission denied** — onboarding and hazard capture both show a clear
    recoverable error instead of hanging.
13. **Unauthorized data access** — not exploitable in this single-user build; enforced
    server-side per §7 once Firebase Auth + rules are wired in.

---

## 10. Deployment instructions

This build is a static single HTML file — deploy it as-is to any static host (Firebase
Hosting, Netlify, Cloudflare Pages, S3+CloudFront). For the production version:
1. `firebase init hosting firestore functions`
2. Deploy Firestore rules (§7) and the escalation Cloud Function (§4).
3. Add the Maps/Gemini API keys as environment variables to your hosting/function
   config — never commit them to frontend source; proxy Gemini calls through a Cloud
   Function so the key isn't exposed client-side (this build's Claude call is exposed
   to prototyping constraints only — do not ship an LLM key directly in frontend JS in
   production).
4. `firebase deploy`

---

## 11. How AI is used (short version)

The only AI call in the system classifies a hazard report (photo + optional text) into
a structured `{category, description, severity, confidence}` record, using Claude as
the documented stand-in for Gemini (§1). The user always reviews/edits this before it's
stored. Everything safety-critical — GPS math, deviation/stop detection, the
safety-check timer, escalation, and the Phase 3 risk score — is deterministic software,
never an LLM, so it stays reliable, testable, and explainable. The risk engine explains
*which* indicators (deviation distance, stop duration, nearby hazard severity/recency)
contributed to its LOW/MODERATE/HIGH output, and never claims to predict crime or
guarantee safety.

## 12. Known limitations

- Map is schematic (straight-line), not road-following — no Maps/Directions key
  available in this environment (§1).
- Destination selection is manual coordinates or offset "quick picks," not real
  address geocoding — no Places/Geocoding key available.
- Hazard classification runs on Claude, not Gemini, as a documented stand-in with an
  identical output contract.
- Trusted-contact alerts create a real event and (if permitted) a real browser
  notification, but do not send a real SMS/push — no Twilio/FCM credentials available.
- Single-device/session persistence in this build (`window.storage`); multi-device
  sync requires the Firebase swap in §1/§4.
- "Community" hazard data is shared across sessions using this environment's shared
  storage scope, but there's no real multi-tenant auth boundary until Firebase Auth +
  the rules in §7 are wired in.
