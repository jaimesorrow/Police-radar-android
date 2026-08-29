---
name: police-radar-review
description: Reviews changes to Police Radar Android, a not-yet-scaffolded Android app for crowdsourced pinning of police-vehicle sightings on a map (per README.md). Use this instead of a generic code review for the first commits that add a Gradle/Android project, any code handling location/GPS data or permissions, map pin creation/expiry, background or foreground location services, reporter identity/privacy, or Maps API keys. Until real source exists, apply it as a pre-flight checklist rather than a diff-vs-invariants review.
---

# Police Radar Android review

## Current repository state (verify this section still matches before relying on it)

As of this skill being written, the repository contains only `README.md`, `LICENSE`, and
`.gitignore` — there is no `build.gradle(.kts)`, no `CLAUDE.md`, no `app/` module, and no Kotlin/Java
source anywhere. The `.gitignore` is already Android-flavored (ignores `.gradle/`, `build/`,
`local.properties`, `*.apk`/`*.aab`, `*.jks`/`*.keystore`, and notably `google-services.json`),
which is the only signal so far that this will be a real Gradle/Android app, likely wired to a
Firebase-style backend for the crowdsourced pin data.

The entire product concept, from `README.md`, is: **"An app where people have the ability to
pinpoint on a map that a police vehicle has been spotted."** Every check below is derived directly
from what that concept implies once real code lands — there is no existing test suite or
architecture doc to treat as a spec yet, so this skill *is* the closest thing to project doctrine
until one exists. If a `CLAUDE.md` or test suite appears later, prefer those over this list where
they conflict, and update this file to match.

Because there's no source yet, most first diffs will be scaffolding (Gradle files, manifest,
first Activity/Composable, first map screen). Review those diffs against the checklist below as a
**pre-flight list of invariants the scaffolding must not foreclose**, not as a report of violations
already found in code that doesn't exist. Re-review this file's "current state" section first —
once `build.gradle`/`CLAUDE.md`/source show up, this section is stale and the checks below become
literal diff-vs-invariant checks against real files/packages.

## Checklist

1. **Reporter anonymity is the product's actual safety property, not a nice-to-have.** The whole
   pitch is people reporting live police locations. Anyone who submits a pin should not be
   identifiable from it — flag any change that:
   - introduces sign-in/account creation, a persistent per-user ID, or a device ID that gets stored
     server-side alongside pin data (e.g., in whatever replaces `google-services.json`'s Firestore/
     Realtime DB schema).
   - logs or transmits IP address, advertising ID, or any other identifier in the same
     request/record as a pin's coordinates or timestamp.
   - If accounts get introduced later, that's a real product decision — treat it the way the sibling
     Pocket-Lawbook repo treats its own no-accounts rule: something that needs an explicit, visible
     decision, not something that lands as a side effect of adding, say, a "my reports" feature.

2. **Pins must expire — a police vehicle is not still there an hour later.** Any pin/report data
   model needs a timestamp, and there must be actual code (client-side filter, server-side TTL, or
   both) that stops showing or actively removes pins older than some bounded window. Flag:
   - a pin entity/schema with no timestamp field.
   - a map layer that renders every pin ever returned by a query with no age filter.
   - an expiry check implemented only in the client's initial fetch (e.g., filtered once on load)
     but not re-applied as time passes while the map stays open.

3. **Location permission requests must match what the feature actually needs, and background
   location gets extra scrutiny.** `ACCESS_FINE_LOCATION`/`ACCESS_COARSE_LOCATION` for "show my
   position and let me drop a pin" is expected; `ACCESS_BACKGROUND_LOCATION` is a much bigger claim
   (Play Store policy requires prominent in-app disclosure of what it's for) and is only justified
   if there's a genuine background feature (e.g., proximity alerts while the app isn't open). Flag:
   - a background location permission request with no corresponding disclosure UI and no
     foreground-service-backed feature that needs it.
   - permission requests happening before/without a runtime rationale, or requesting more
     precision (`FINE` vs `COARSE`) than the feature at hand uses.

4. **Any "alert me when a pin is nearby" feature must be a proper foreground service, not a bare
   background `Service` or a busy-poll loop.** Since Android 8-10, background execution limits will
   kill a plain background service; a persistent-notification foreground service (or
   `WorkManager`/geofencing APIs) is the correct shape for continuous or periodic location checks.
   Flag a background location-polling loop that isn't backed by a foreground service, geofencing
   API, or WorkManager — it will look like it works in a debug session and silently stop in the
   field, which for this app means the safety feature quietly does nothing.

5. **Crowdsourced pins need at least a minimal abuse/spam guard given there are no accounts.**
   With no login, there's no per-user rate limit to fall back on — flag a "submit pin" code path
   that has no throttle at all (e.g., unlimited pins per device/session, no dedup of near-identical
   coordinates within a short window, no minimum distance/time between a device's own submissions).
   It doesn't need to be sophisticated, but it shouldn't be entirely absent.

6. **Maps/backend secrets stay out of git, and API keys stay restricted.** `.gitignore` already
   excludes `google-services.json` and `local.properties` — don't let a diff reintroduce a
   committed Maps/Firebase API key (in `AndroidManifest.xml`, a committed `google-services.json`,
   or a hardcoded string in source). If a Maps SDK key shows up in the manifest, it should be
   restricted (by package name + SHA-1, or by referenced API) rather than an unrestricted key
   pasted in for convenience.

7. **Gradle/project hygiene once the module exists:** the Gradle wrapper should be committed (so
   builds are reproducible without a pre-installed Gradle); `local.properties` (SDK path) should
   stay gitignored as it already is; package/namespace, minSdk/targetSdk/compileSdk should be
   consistent between `build.gradle(.kts)` and `AndroidManifest.xml`; and any new architectural
   decision worth remembering (data model for pins, how expiry is enforced, why a permission is
   needed) belongs in a `CLAUDE.md` at the repo root, since none exists yet for future reviews to
   lean on.
