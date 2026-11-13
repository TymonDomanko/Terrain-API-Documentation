# Terrain call map

Source of truth: **Terrain's own web app** (Nuxt, `terrain.scouts.com.au`), read on 2026-11-13. The app's config and all 267 of its JS bundles (4 entry files plus 263 lazy chunks) were read from a signed-in tab as static text, and every `this.$config.api.<host>` call site was extracted. That step saved nothing to disk and made no extra API calls.

The extraction is static, so calls built dynamically (a path assembled from variables, shown below as `{…}`) may have more variants than listed. `?` means the HTTP method could not be read from the minified code.

Legend for the last column:
- **Live**: ✔ = response shape inspected in a real session (see `terrain-api.md`). Blank = not inspected yet.

## Hosts

From `window.__NUXT__.config.api`. These are all of them:

| Key | Base URL |
|---|---|
| `members` | `https://members.terrain.scouts.com.au` |
| `achievements` | `https://achievements.terrain.scouts.com.au` |
| `events` | `https://events.terrain.scouts.com.au` |
| `templates` | `https://templates.terrain.scouts.com.au` |
| `metrics` | `https://metrics.terrain.scouts.com.au` |
| `agenda` | `https://agenda.terrain.scouts.com.au` |

Auth is Cognito (`cognito-idp.ap-southeast-2.amazonaws.com`): `InitiateAuth` (password and refresh), `GetUser`. The id token goes in `Authorization` (seen on the members and agenda hosts). Details in `terrain-api.md`.

## Members host

| Method | Path | Live |
|---|---|---|
| GET | `/profiles` | ✔ |
| GET | `/members/{id}` | ✔ |
| GET | `/members/?branch=&member_number=` | |
| GET | `/members/{id}/groups` | ✔ |
| GET | `/branches/{branchId}/groups?…` | |
| GET | `/groups/{groupId}/units` | ✔ |
| GET | `/groups/{groupId}/patrols` | ✔ |
| GET | `/groups/{groupId}/patrols/{patrolId}` | rejected |
| GET | `/groups/{groupId}/members` | ✔ (140 of 433 in Terrain's own client too, see notes) |
| POST | `/groups/{groupId}/units` | not called |
| PUT / DELETE | `/groups/{groupId}/units/{unitId}` | |
| PUT / DELETE | `/groups/{groupId}/units/{unitId}/members/{memberId}` (unit duty and council) | PUT ✔ tested and reverted (204) |
| PUT / DELETE | `/groups/{groupId}/patrols/{patrolId}/members/{memberId}` (patrol membership and patrol duty; `{formation}` is `units` or `patrols`) | both ✔ tested and reverted (204) |
| POST | `/groups/{groupId}/patrols` (create; 200 with the new patrol) | ✔ observed |
| PUT | `/groups/{groupId}/patrols/{patrolId}` (rename; 200) | ✔ observed |
| POST | `/groups/{groupId}/patrols/{patrolId}` (delete a patrol; 204, no body) | ✔ observed |
| GET | `/units/{unitId}` | ✔ |
| GET | `/units/{unitId}/members` | ✔ |
| GET | `/units/{unitId}/patrols` | ✔ |
| GET | `/units/{unitId}/patrols/{patrolId}` | |
| GET | `/units/{unitId}/metrics` (bare array, one entry per member, achievement summaries) | ✔ |
| GET | `/patrols/{patrolId}/members` (no `unit`, `patrol` or `date_of_birth`) | ✔ |
| POST | `/members/{id}/assume-profiles` → 204, no body | ✔ |
| POST | `/revoke-assumed-profiles` → 204, no body | ✔ |
| POST | `/members/{id}/reset` (password reset) | |
| POST | `/members…` (create) | |
| PATCH | `/users/{id}` | |
| GET | `/support-leader-groups` | 403 |
| POST | `/support-leader-access` | |
| GET | `/admin/support-leader-access-records?status=PENDING\|APPROVED` | |
| GET | `/admin/members/?member_number=` | |

## Achievements host

| Method | Path | Live |
|---|---|---|
| GET | `/members/{id}/achievements` (optional `?type=`) | `?type=milestone` ✔, `?type=special_interest_area` ✔ (empty), `?type=outdoor_adventure_skill` ✔; `intro_scouting` ✔, `intro_section` ✔ (mixed imported and recorded shapes), `adventurous_journey` ✔ (empty), `peak_award` ✔ (empty); others not seen |
| GET | `/members/{id}/achievements/{achievementId}` (+ `?type=`) | |
| GET | `/members/{id}/achievements/{achievementId}/submissions` | ✔ (`[]`) |
| GET | `/members/{id}/peak-award-progress?section=` (one object, same data as unit metrics) | ✔ |
| GET | `/members/{id}/logbook` | ✔ (own) |
| GET | `/members/{id}/logbook/{logbookId}` | ✔ (own) |
| GET | `/members/{id}/logbook-metrics` | ✔ (own) |
| GET | `/units/{unitId}/achievements?type={…}` (all members, no assume needed; `type` optional but the untyped call fails for big units) | ✔ |
| GET | `/units/{unitId}/achievements?type=outdoor_adventure_skill&stream=&branch=&stage=` (filters work) | ✔ |
| GET | `/units/{unitId}/submissions?status=pending\|finalised` | ✔ (200 for the Scout unit, 403 for the other three) |
| POST | `/members/{id}/achievements` | |
| PUT / DELETE | `/members/{id}/achievements/{achievementId}` | |
| POST | `/members/{id}/achievements/{achievementId}/archive?submission_id=` | |
| POST | `/members/{id}/create-achievement` | |
| POST | `/members/{id}/import-achievements` and `/import-achievements/submit` | |
| POST / PUT / DELETE | `/members/{id}/logbook` and `/logbook/{logbookId}` | |
| PUT | `/members/{id}/skip-achievement/{achievementId}` and `/unskip-achievement/{achievementId}` | |
| POST | `/submissions/{submissionId}/assessments` | |
| PUT | `/units/{unitId}/achievements/manage-oas-stage` | |
| PUT | `/units/{unitId}/achievements/milestone-import-add-credits` and `/milestone-import-award-stage` | |
| PUT | `/units/{unitId}/bulk-award-sia` | |
| PUT | `/units/{unitId}/members/{id}/import-achievements/introduction` | |
| POST | `/members/{id}/uploads` (file upload target) | not called |

## Events host

| Method | Path | Live |
|---|---|---|
| GET / PUT | `/members/{id}/calendars` | GET ✔ |
| GET | `/members/{id}/events?start_datetime=&end_datetime=` (ranges up to 20 years work; `&download_format=csv&download_tz=` returns CSV) | ✔ |
| GET | `/members/{id}/organised-events` | ✔ |
| GET / POST | `/members/{id}/calendar-subscription` (feed URLs are secrets) | GET ✔ |
| GET | `/events/{eventId}` (also DELETE) | ✔ |
| GET | `/events/{eventId}/invitees` (returns all 433 group members) | ✔ |
| POST | `/events` (create event, not a read) | not called |
| POST | `/units/{id}/events`, `/patrols/{id}/events`, `/groups/{id}/events` (create event, not reads) | not called |
| GET / POST | `/units/{unitId}/proposals` | GET ✔ |
| GET / DELETE | `/proposals/{id}` | GET ✔ |
| POST | `/units/{id}/uploads`, `/groups/{id}/uploads` (file upload targets) | not called |
| — | `/calendar-feeds/{member}/{feed}` is an iCal URL shown to the user, not an API call | |

## Templates host

| Method | Path | Live |
|---|---|---|
| GET | `/oas/{stream}/tree.json` (nine streams, all fetched together) | ✔ |
| GET | `/{stream}/latest.json` (built from `{…}`) | |
| GET | `/oas/{section}/overview/latest.json` | ✔ |
| GET | `/milestone/{section}/overview/latest.json`, `/milestone/{section}/review/{stage}/latest.json`, `/sia/{section}/overview\|plan/latest.json` | ✔ |
| GET | `/additional-awards/specifications.json` (bare JSON array) | ✔ |
| GET | `/additional-awards/historic-awards.json` | ✔ |
| GET | `/additional-awards/{name}.json` | |
| GET | `/sia/{area}/plan/latest.json` and `/sia/{area}/review/latest.json` | |
| GET | `{…}/{…}/overview/{…}` and other path-built template reads | |

## Agenda host

| Method | Path | Live |
|---|---|---|
| GET | `/units/{unitId}/member-agenda` → `{ count, generated_on, items[] }`; items are pending submissions | ✔ |
| ? | `/members/{id}/psl-agenda` | rejected (support leader only) |

## Metrics host

No call sites were found in the web client's bundles. A preflight to `/units/{unitId}/members` succeeded, so that route exists.

## Notes and open questions (updated after the full read sweep)

Full shapes are in `terrain-api.md`. The headline findings:

1. **Unit-level reads exist and need no assumed profile.** `GET achievements /units/{id}/achievements?type=…` returns every member's records in one call (filters `stream`, `branch`, `stage` work), and `GET members /units/{id}/metrics` gives per-member summaries. Per-member achievement reads also work for other members without assuming (`achievements`, `?type=`, `peak-award-progress`). Only the **logbook** calls (`/logbook`, `/logbook-metrics`) return 403 without an assumed profile, so assume and revoke is needed only for logbooks.
2. **No paging.** The achievements host ignores `limit`, `offset` and `page`; the members host rejects them. `groups/{id}/members` returns 140 of 433; `events/{id}/invitees` returns all 433. The unit members endpoint returns everything (`total` equals the result count in all 4 units).
3. **403 means no permission for that unit** (`units/{id}/submissions` is 200 only for the Scout unit; `support-leader-groups` 403). Treat as empty.
4. **Large and failing responses.** Unit reads reach several MB. `GET /units/{id}/achievements` with no `type` failed for the Scout unit; use `type=` per call.
5. **Allowed `type` values** (anything else is a 400 with `{ errors: { type: [...] } }`): `milestone, intro_scouting, intro_section, outdoor_adventure_skill, adventurous_journey, course_reflection, personal_reflection, peak_award, additional_award, special_interest_area`. `personal_development` is rejected.
6. **Writes.** Terrain's client has many write calls. Reading data needs none of them, except assume and revoke for another member's logbook.
7. **Still not seen live:** write responses (deliberately not called), `members/{id}/uploads`, unit/patrol/group `events` (these are create-event POSTs, not reads), `branches/{id}/groups`, `members/?branch=&member_number=` (not called, it looks up any member), `psl-agenda`, `calendar-feeds`.

## How this was produced

All in the signed-in Terrain tab, with in-page JavaScript:
1. Read `window.__NUXT__.config.api` for the hosts.
2. Read the webpack chunk-name and hash maps from the entry bundle and fetched all chunk URLs.
3. Regex over the text for `.api.<host>` followed by the URL template, resolving `*_PATH` constants from the exported constants module.
Endpoints not reachable by this route: anything built outside a template literal, and any call made from code loaded after the chunk list was read.
