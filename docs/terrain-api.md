# Terrain API notes

Observed from a signed-in Terrain web session (2026-11-13). Only response **structure and enum values** were recorded, no member data. "Observed" means seen in live traffic; "code only" means read from the Terrain web client's code.

## Auth

All observed on a real sign-in (only key names and types were recorded, never values).

| Call | Observed |
|---|---|
| `POST https://cognito-idp.ap-southeast-2.amazonaws.com/` `X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth` | Body `{ AuthFlow, ClientId, AuthParameters: { USERNAME, PASSWORD }, ClientMetadata: {} }`. Response `{ AuthenticationResult: { AccessToken, ExpiresIn, IdToken, RefreshToken, TokenType }, ChallengeParameters: {} }`. Username is `<branch>-<memberNumber>`, e.g. `nsw-<digits>`. Headers: `Content-Type`, `X-Amz-Target`, plus the web app's `X-Amz-User-Agent` and `Cache-Control`, which are not required. |
| `...GetUser` | Body `{ AccessToken }`. Response `{ UserAttributes: { Name, Value }[] (3 attributes), Username }`. The web app calls it several times. |
| `...InitiateAuth` refresh | Body `{ ClientId, AuthFlow, AuthParameters: { REFRESH_TOKEN, DEVICE_KEY } }`. Response has `AccessToken`, `ExpiresIn`, `IdToken`, `TokenType` and **no `RefreshToken`**, so keep the old one. |
| API calls | `Authorization` header (id token) plus `Accept`. |

Not observed: the login-failure response, and which Cognito app client ids are valid for which accounts.

## `GET members.terrain.scouts.com.au/profiles`

```
{ username: string,                       // "<branch>-<digits>"
  profiles: Profile[] }                   // 4 for this account: one per unit

Profile {
  member: { id, name, roles: string[] },
  unit:   { id, name, roles: string[], section },
  group:  { id, name, roles: string[] },
  // not present for this account: branch, is_support_group
}
```

Roles seen: member `achievement-write`, `member`; unit `unit-member`, `unit-write`, `unit-council` (on 1 of 4); group `group-leader` (on all 4). All 4 profiles share one member id and one group id.

Consequences: the leader test in `isYouthMember` (`group-leader`, `support-leader-group-*`, `unit-council`) reads these fields correctly. `support-leader-group-*` and the `branch` object are untested against live data (this account has neither).

## Hosts seen

| Host | Notes |
|---|---|
| `members.terrain.scouts.com.au` | profiles, units, patrols, members |
| `agenda.terrain.scouts.com.au` | `GET /units/{unitId}/member-agenda` → `{ count: number, generated_on: string, items: [] }`; `items` was empty for the Scout unit, so item shape is unknown |

## Group-level calls (`members.terrain.scouts.com.au`)

- `GET /groups/{groupId}/patrols` → `{ results: { id, unit_id, name }[] }` (17 patrols; no `duty`)
- `GET /groups/{groupId}/units` → `{ results: { id, name, section, group_id }[] }` (4 units: joey, cub, scout, venturer)
- `GET /groups/{groupId}/members` → `{ results, total, limit: null, offset: null }`. Each result has `id, first_name, last_name, status, member_number, date_of_birth, metadata`, `units[]` of `{ id, name, section, duty, unit_council, group: { id, name } }`, plus `role`, `unit` and `patrol`.
  - Differs from the unit endpoint: members can have several units (0 to 4 seen; 27 of 140 had none) and `unit.group` is `{ id, name }` instead of `group_id`.
  - **Returned 140 results but `total` was 433**, with no query string. Unexplained: a page cap or a capture error. The unit endpoint returned 49 of 49.

## `GET members.terrain.scouts.com.au/units/{unitId}/members`

Observed: 200, `Accept` + `Authorization` headers, 49 members returned in one response (`limit` and `offset` are `null`, no paging seen).

```
{ results: Member[], total: number, limit: null, offset: null }

Member {
  id, member_number, first_name, last_name, status,   // string
  date_of_birth,                                      // "yyyy-mm-dd" on all 49
  groups: { id, name }[],
  unit:   { id, section, duty, unit_council: boolean, group_id },
  patrol: { id, name, duty } | null,
  metadata: { "achievement-import": string }          // hyphenated key
}
```

Enum values seen:

- `unit.duty`: `member`, `adult_leader`, `unit_leader`
- `patrol.duty`: `member`, `patrol_leader`, `assistant_patrol_leader`
- `status`: `active`
- `unit.section`: `scout`

Duty combinations seen (unit duty | patrol duty): `member | -`, `member | patrol_leader`, `member | member`, `member | assistant_patrol_leader`, `unit_leader | -`, `adult_leader | member`, `adult_leader | -`.

Implications for clients:

- PL and APL come only from `patrol.duty`. `unit.duty` is never `patrol_leader`, so a patrol leader is identified by the patrol duty alone.
- Unit leaders have `patrol: null` and adult leaders may sit in a patrol with duty `member`. Adult test must be `unit.duty === 'adult_leader'` (`isAdultLeader`), never "has a patrol".
- The metadata key is hyphenated (`achievement-import`); an unhyphenated `achievementimport` does not match.

## `GET members.terrain.scouts.com.au/units/{unitId}/metrics`

Opens when you click a unit in Group Life. Observed: 200, one call for the whole unit (39 members). The body is a **bare array**, not `{ results }`.

```
MemberMetrics[] where MemberMetrics {
  member_name, member_id, date_of_birth,                      // strings
  achievement_metrics: {
    peak_award_progress: number,
    intro_scouting_completed_date, intro_section_completed_date: string,
    milestone_metrics: {
      has_met_milestone_3_requirement: boolean,
      current_milestone_stage: number,
      current_milestone_credits: { participate_community, participate_creative, participate_outdoor,
        participate_personal_growth, total_participates, total_assists, total_leads: number },
      section_milestone_statuses: { stage_1_status, stage_2_status, stage_3_status: string }
    },
    special_interest_area_metrics: { completed_projects, projects_in_progress, sia_percentage_progress: number,
      areas_completed: [] },
    outdoor_adventure_skill_metrics: { completed_progressions_in_section, oas_percentage_progress: number,
      highest_progressions: { stream, branch: string, stage: number }[] },
    adventurous_journey_completed_date, personal_development_completed_date,
    personal_reflection_completed_date: string,
    additional_awards: []
  },
  event_metrics: { recent_event_count, recent_unit_event_count: number }
}
```

Only the first element's shape was recorded, so a field absent on it (for example a member with no OAS progress) may be missing elsewhere. This one call gives the summary figures for every member at once, but it holds summaries only (no per-requirement detail).

## `GET templates.terrain.scouts.com.au/oas/{stream}/tree.json`

Observed 200 for nine streams: `alpine, aquatics, boating, bushcraft, bushwalking, camping, cycling, paddling, vertical`. The Terrain web app fetches all nine whenever Record Achievements or Group Life opens.

```
{ title, stream_id, description, achievement_type: string,
  tree: { branch_id, title: string, stage: number, template_link: string,
          children: Node[] } }   // Node has the same fields, nested 3+ levels
```

## `GET achievements.terrain.scouts.com.au/units/{unitId}/submissions?status=pending|finalised`

Observed on the Approvals page. Both calls returned **403 with an empty body** for the unit Terrain selected, and the page showed "No data available". So a 403 here means "no access", not an error to surface. Success shape not observed.

## Group members paging, again

Terrain's own web client calls `GET /groups/{groupId}/members` with no query and gets the same **140 of 433** results (twice, on Record Achievements). So this is how the server behaves for a plain call, not an artefact of the capture method. Whether `limit` or `offset` params change it is untested.

## Assume profile (observed 2026-11-13, one member)

Clicking "Manage achievement pathways" on a member in Group Life shows a confirmation dialog (30-minute timer, "switch back once you are done"). On Continue Terrain sent, in order:

1. `POST members…/revoke-assumed-profiles` → 204, no body, headers `Accept`, `Authorization`
2. `POST members…/members/{memberId}/assume-profiles` → 204, no body
3. `GET members…/profiles` → 200

Then it navigates to a youth basecamp. Terrain does not read the response of 1 or 2 (both 204).

`/profiles` while assumed returned **5 profiles instead of 4**: the account's own four plus one extra, which has an additional `datetime_expires` key and these roles:

- member roles: `member-on-behalf`, `achievement-write-on-behalf`
- unit roles: `unit-submission-write-on-behalf`

So the assumed profile is added, not swapped in. Do not use `profiles[0]` or "the profile with a unit" to decide who the user is while a profile is assumed. The side panel showed "Time left: 28 minutes".

To stop: avatar menu → "Stop managing achievement pathways" → confirm. Terrain sent `POST /revoke-assumed-profiles` → 204, then `GET /profiles` returned the original 4 profiles with no `datetime_expires`.

Implication for per-member loops (assume → read → revoke): each assume needs its own revoke, the token is reused, and the assumed profile expires on its own after 30 minutes.

## Per-member reads seen while assumed

Headers on all: `Accept`, `Authorization`.

### `GET achievements…/members/{memberId}/achievements?type=milestone`

200, `{ results: Achievement[] }` (3 entries, all `status: in_progress`, `section: scout`, `type: milestone`):

```
Achievement {
  id, member_id, section, type, status, status_updated, milestone_requirement_status: string,
  event_log: { credit_type, challenge_area, event_id, event_name, event_start_datetime: string }[],   // 25 entries on the first
  event_count: { participant|assistant|leader: { community, outdoors, creative, personal_growth: number } },
  achievement_meta: { stage: number }
}
```

No `latest_submission` on these in-progress entries, so the field can't be assumed present. Note `event_count` uses `outdoors` where the unit-metrics call uses `participate_outdoor`.

### `GET achievements…/members/{memberId}/achievements?type=special_interest_area`

200, `{ results: [] }` for this member (empty, so the shape is not known).

### Templates read alongside

- `GET templates…/milestone/scout/overview/latest.json` → `{ template, version, document: { title|description|about_box|i_participate_dialog|i_assist_dialog|i_lead_dialog|info_box: { type, label, links? } } }`
- `GET templates…/milestone/scout/review/1/latest.json` → `{ template, version, document: [{ input_groups: [{ inputs: { id, type, label }[] }] }] }`
- `GET templates…/sia/scout/overview/latest.json` and `/sia/scout/plan/latest.json` → `{ template, version, document }`

Pattern: `/{type}/{section}/{overview|plan|review/{stage}}/latest.json`.

## Still not observed

`/members/{id}/achievements` without `type`, `?type=outdoor_adventure_skill`, `adventurous_journey`, `peak_award`, `additional_award`, intro types, `/logbook*`, `/peak-award-progress`, `/units/{id}/achievements`.

## Logbook (own logbook, no assumed profile, observed 2026-11-13)

"My Logbook" in the side menu works on the signed-in user's own member id, so these reads need no assume. For another member the profile has to be assumed first (see "Permission model" below). Headers: `Accept`, `Authorization`.

### `GET achievements…/members/{memberId}/logbook`

`{ results: Entry[] }`, 14 entries. Summary only:

```
Entry { id, title, start_date: "yyyy-mm-dd", achievement_meta: { stream, branch: string } }
```

`stream` and `branch` are not always equal: stream `paddling` carries branch `canoeing`. Streams seen: cycling, camping, paddling, bushwalking.

### `GET achievements…/members/{memberId}/logbook-metrics`

`{ results: { name: string, value: number, last_updated: string }[] }`. Names seen: `total_nights_camped`, `total_distance_hiked`. The page shows these as "Camper Award" and "Walkabout Award" progress.

### `GET achievements…/members/{memberId}/logbook/{logbookId}`

Opened with "View" on an entry. 200, one object:

```
{ id, title, start_date, end_date: string,
  achievement_meta: { stream, branch },
  distance_travelled, distance_walkabout: number,
  categories: string[],
  details: { other_participants, activity_time_length, activity_grade, event_id, purpose, who_lead,
             verifier: { name, contact }, weather, location, event_title, your_role: string } }
```

Only one entry's detail was read, so fields may be optional on others.

## OAS (own pathways, no assumed profile)

Reached from Basecamp → Youth Basecamp → carousel → Outdoor Adventure Skills (`/oas`). That view shows the signed-in user's own pathways, in their own section.

### `GET achievements…/members/{memberId}/achievements?type=outdoor_adventure_skill`

200, `{ results: [] }`, 3 entries, 2,827 bytes, no `stream` param. Every entry had every listed key (no optional fields in this sample).

```
Achievement {
  id, member_id, template: string, version: number,
  section, type, status, status_updated: string,
  answers: { [key: string]: boolean | string },     // dynamic keys, see below
  achievement_meta: { stage: number, stream, branch: string }
}
```

`answers` keys are `<requirement-slug>_<6-char hash>` (boolean, ticked or not), with four string siblings per requirement: `…_verifierName`, `…_verifierContact`, `…_verifierType`, `…_verifiedDate`. Values seen: `type` `outdoor_adventure_skill`, `status` `draft_review`, `section` `scout`, `stream` `cycling`, `stage` 1, 2, 3. A `file_uploader` field did not appear in this response (0 occurrences), so it may only appear for entries with uploads.

### `GET templates…/oas/{section}/overview/latest.json`

Fetched with `section = joey` while viewing the Joey section: `{ template, version, document: { title, description, empty_state, info_box: { type, label, links: { path, text }[] } } }`. So `oas/{x}` is a section here, but `oas/{stream}/tree.json` takes a stream; both use the same prefix.

Note the section on the entries (`scout`) differed from the section of the page's overview template (`joey`), because the entries come from the member id, not from the section being viewed.

## Still not observed

`/members/{id}/achievements` with no `type`, `?type=adventurous_journey`, `peak_award`, `additional_award`, `intro_scouting`, `intro_section`, `/peak-award-progress`, `/units/{id}/achievements?type=…`, and `/members/{id}/achievements/{achievementId}`.

## More own-account reads (observed 2026-11-13, no assumed profile)

Your own member id has no data for these, so only the call patterns and empty bodies are known. Their populated shapes are not.

| Call | Result |
|---|---|
| `GET achievements…/members/{id}/achievements?type=intro_scouting` | 200, `{ results: [] }` |
| `GET achievements…/members/{id}/achievements?type=intro_section` | 200, `{ results: [] }` |
| `GET achievements…/members/{id}/achievements?type=adventurous_journey` | 200, `{ results: [] }` |

Templates fetched alongside: `intro-scouting/{section}/overview/latest.json`, `intro-section/{section}/overview/latest.json`, `adventurous-journey/{section}/overview/latest.json` and `…/plan/latest.json`. Same `/{type}/{section}/{overview|plan|review/{stage}}/latest.json` pattern as milestone and SIA.

### Additional awards page (own)

The page made **no `achievements` call**. It only read two template files:

- `GET templates…/additional-awards/specifications.json` → bare array, 31 entries: `{ title, id, sections: string[5], description }`. Ids look like `camper_award_10`, `camper_award_25`. Sections: `joey, cub, scout, venturer, rover`.
- `GET templates…/additional-awards/historic-awards.json` → same shape, 1 entry.

### Youth Basecamp carousel (own), what each page opens

Program Essentials (Introduction to Scouting, Introduction to Section, Milestones), Special Interest Areas, Outdoor Adventure Skills, Adventurous Journey, My Progress. "My Progress" ("progress towards the Australian Scout Award") had no link or call when clicked, so `peak-award-progress` is still not seen.

## What the unit metrics say about where data exists (counts only)

In the Scout unit's metrics (39 members): 36 have an Introduction to Scouting date, 21 an Introduction to Section date, 0 an Adventurous Journey date, and 36 have non-zero peak award progress. So populated `intro_scouting`, `intro_section` and peak-related shapes are available by assuming one such member. Adventurous Journey has no completed example in this unit.

## Populated per-member shapes (second assumed member, observed 2026-11-13)

One member with intro, milestone, SIA, OAS and logbook data was assumed via the same sequence as above (revoke, assume, profiles; then revoke on stop; profiles back to 4 with no `datetime_expires`). Only structure was recorded.

### Records come in different shapes within one `type`

An achievement can be **imported** (no template, no answers) or **recorded in Terrain** (template, answers, submission). The same type returns both, so every field except `id, member_id, section, type, status, status_updated` must be treated as optional.

`?type=intro_scouting` → `{ results: [ { id, member_id, section, type, status, status_updated, imported: { date_awarded, awarded_by: { id, name } } } ] }` (1 entry, `status: awarded`).

`?type=intro_section` → 2 entries, both `awarded`:
- recorded: `{ …base, template, version, answers: { <23 requirement keys>: boolean }, last_updated, latest_submission: { submission_id, unit_id, submission_time } }`
- imported: `{ …base, imported: { … } }`

### `?type=outdoor_adventure_skill` for a populated member

42 entries, about 270 KB, 18 distinct branches across 8 streams. Key presence across the 42: `id, member_id, template, version, section, type, answers, status, status_updated, achievement_meta` on all; `last_updated` 34; `latest_submission` 28; `uploads` 8; `date_awarded` 7. Statuses: `awarded` 28, `draft_review` 14. `latest_submission` and `date_awarded` appeared only on `awarded` entries.

- `answers` values are boolean, string, or arrays. Keys ending `_uploader` hold arrays of strings (35 of them, mostly empty, max 196 bytes), which are the file-upload answers. Other key suffixes seen: `_verifierName`, `_verifierContact`, `_verifierType`, `_verifiedDate`, `_eventId`, `_date`.
- `uploads: { id, filename, bucket, key, url, uploaded_on }[]`.
- `date_awarded`: `yyyy-mm-dd`. `status_updated`: ISO with microseconds and offset (`yyyy-mm-ddThh:mm:ss.ffffff+hh:mm`).
- Stream and branch differ often. Branches seen: kayaking, canoeing, paddling, sea-kayaking, boating, sailing, cycling, mountain-biking, bushwalking, camping, bushcraft, survival-skills, pioneering, vertical, abseiling, aquatics, snorkelling, life-saving.

No large payloads: the biggest single `_uploader` value was 196 bytes. File-upload answers are small, so they need no special handling.

### `GET achievements…/members/{id}/peak-award-progress?section=scout`

200, one object (not wrapped):

```
{ intro_scouting, intro_section: boolean,
  milestones: { has_met_milestone_3_requirement: boolean, current_milestone_stage: number,
                current_milestone_credits: { participate_community, participate_creative, participate_outdoor,
                  participate_personal_growth, total_participates, total_assists, total_leads: number },
                section_milestone_statuses: { stage_1_status, stage_2_status, stage_3_status: string } },
  oas: { completed_progressions_in_section, oas_percentage_progress: number,
         highest_progressions: { stream, branch: string, stage: number }[] },     // 8 entries here
  sia: { completed_projects, projects_in_progress, sia_percentage_progress: number,
         areas_completed: string[] },                                            // ids like sia_adventure_sport
  adventurous_journey_progress, personal_development_progress, personal_reflection_progress,
  essentials_progress, peak_award_progress: number }
```

This is the same data as the unit metrics call, for one member, with renamed keys: `milestones` vs `milestone_metrics`, `oas` vs `outdoor_adventure_skill_metrics`, `sia` vs `special_interest_area_metrics`, and the progress values are numbers here instead of completion dates. The Peak Award page also calls `?type=peak_award`, which returned `{ results: [] }` for this member (not awarded yet), plus the logbook list and metrics.

### Logbook for an assumed member

Same shapes as the own logbook: `/logbook` returned 44 entries, `/logbook-metrics` 2.

### Still not seen

`?type=additional_award`, `personal_reflection`, `course_reflection` with data, `achievements` with no `type`, `units/{id}/achievements`, `submissions` success, and a populated `peak_award`.

# Full read sweep (2026-11-13)

Every read endpoint from the bundle list was called with `GET` from the signed-in page, using the page's own auth header (kept in page memory, never printed), against your own member, your 4 units, your group and 17 patrols. Writes, admin calls and "look up any member by number" were not called. Only status, size and structure were recorded.

A call that fails with an opaque **"Failed to fetch"** is the API gateway rejecting the route or parameters without CORS headers, so the status code is not visible. Those are listed as "rejected".

## Result matrix

### members host

| Call | Result |
|---|---|
| `GET /members/{memberId}` | 200. `{ id, first_name, last_name, status, role, member_number, date_of_birth, units: { id, section, duty, unit_council }[], patrols: { id, name, duty }[], groups: { id, name }[] }`. Own member: 4 units, 1 patrol, 1 group. |
| `GET /members/{memberId}/groups` | 200, bare array `{ id, name }[]` |
| `GET /units/{unitId}` | 200 `{ id, name, section, group_id }` (all 4 units) |
| `GET /units/{unitId}/patrols` | 200 `{ results: { id, unit_id, name }[] }` (2, 6, 7 and 2 patrols) |
| `GET /units/{unitId}/members` | 200, `total` equals `results.length` for all 4 units (19, 36, 49, 26). No paging. |
| `GET /units/{unitId}/metrics` | 200 bare array, one entry per member (11, 26, 39, 21). Up to 65 KB. |
| `GET /patrols/{patrolId}/members` | 200 `{ results: { id, member_number, first_name, last_name, status }[] }`. **No `unit`, `patrol` or `date_of_birth`**, unlike the unit endpoint. |
| `GET /groups/{groupId}/members` | 200, 140 of `total` 433, 107 KB. |
| `GET /groups/{groupId}/members?limit=` / `?limit=&offset=` | rejected (opaque). The members host does not accept those params. |
| `GET /groups/{groupId}/patrols/{patrolId}` | rejected (not a GET route, or a different path) |
| `GET /support-leader-groups` | 403, empty body (not a support leader) |

### achievements host

Your own member:

| Call | Result |
|---|---|
| `GET /members/{id}/achievements` (no type) | 200, 4 results, **all types in one list**: 1 milestone plus 3 OAS here. Same entry shape as the typed calls. |
| `?type=` `milestone`, `intro_scouting`, `intro_section`, `outdoor_adventure_skill`, `adventurous_journey`, `course_reflection`, `personal_reflection`, `peak_award`, `additional_award`, `special_interest_area` | all 200. Entries only for milestone (1) and OAS (3); others `{ results: [] }` (15 bytes). |
| `?type=personal_development` | **400** `{ errors: { type: ["unallowed value personal_development"] } }` |
| `?type=bogus_type` | **400**, same error shape. So the type list above is the full allowed set, minus any not tried. |
| `?type=outdoor_adventure_skill&stream=cycling` | 200, same 3 entries. The `stream` param was ignored for a member call. |
| `GET /members/{id}/achievements/{achievementId}?type=…` | 200, one achievement object (same fields as a list entry) |
| `GET /members/{id}/achievements/{achievementId}/submissions` | 200, bare empty array |
| `GET /members/{id}/uploads` | rejected |

**Unit-level reads work for every unit you have, with no assumed profile.** `GET /units/{unitId}/achievements?type=…` returned all members' records in one call:

| Type | Joey | Cub | Scout | Venturer |
|---|---|---|---|---|
| milestone | 40 | 102 | **200** (47 members) | 113 |
| additional_award | 4 | 8 | 85 | 41 |
| special_interest_area | 23 | 21 | 126 | 69 |
| intro_scouting | 5 | 21 | 43 | 18 |
| intro_section | 5 | 26 | 64 | 26 |
| peak_award | 0 | 0 | 2 | 1 |
| outdoor_adventure_skill (`stream=cycling&branch=cycling&stage=1`) | 7 | 26 | 35 | 18 |
| no `type` | 170 (303 KB) | 574 (2.2 MB) | **rejected** | 973 (4.4 MB) |

Notes:

- Results are grouped by member, and every member in the unit metrics appears in the Scout milestone result. The 200 for Scout milestone is exactly round, but it is not a cap: the Cub unit's unfiltered OAS call returned 396 entries, and no member's records are cut off. The params `limit`, `offset`, `page` and `next_token` made no difference (always the same 200, params ignored), so there is no paging.
- The Scout unit with no `type` was rejected while the 4.4 MB Venturer one succeeded, which is consistent with a response size limit. Use per-type calls.
- On the OAS unit call the `stream`, `branch` and `stage` params **do filter** (Cub unit: 396 entries with no filter, 50 for `stream=cycling`, 26 for stream, branch and `stage=1`, 15 for `stage=2`). The unfiltered call returned 1.9 MB.
- Entry fields vary by type (counts are for the Scout unit): milestone has `event_log`, `event_count`, `milestone_requirement_status` on all and `imported` on 31 of 200; additional awards are all `imported`, with `expiry_date` on 6 of 85; SIA has `can_archive` on 9 of 126 and statuses `awarded`, `in_progress`, `draft_review`, `draft_approval`, `approved`, `pending_review`; milestone statuses add `not_required`.
- Always treat everything beyond `id, member_id, section, type, status, status_updated` as optional.

`GET /units/{unitId}/submissions?status=pending|finalised`: **200 only for the Scout unit** (10 pending, 976 finalised = 906 KB); **403 for Joey, Cub and Venturer** (the account's `unit-council` role is on one of its four profiles, probably the Scout unit). Pending entry shape: `{ achievement: { id, type, achievement_meta }, submission: { id, type, date, status, actioned_by: [] }, member: { id, first_name, last_name }, unit_permissions: { write: boolean } }`.

### events host

| Call | Result |
|---|---|
| `GET /members/{id}/calendars` | 200 `{ member_id, own_calendars: { id, type, title, selected, section }[7], other_calendars: same[16] }` |
| `GET /members/{id}/events?start_datetime=&end_datetime=` | 200 `{ results: { id, start_datetime, end_datetime, title, invitee_type, status, challenge_area, section, invitee_id, invitee_name, group_id }[168] }` for 2026. Status `concluded` 133, `planned` 35. |
| `GET /members/{id}/organised-events` | 200 `{ results: { id, start_datetime, end_datetime, title, status, group_id }[] }` |
| `GET /members/{id}/calendar-subscription` | 200 `{ member_id, feeds: { own_feed_id, others_feed_id, own_feed_url, others_feed_url } }`. **The feed URLs are secrets.** Shape only recorded. |
| `GET /events/{eventId}` | 200, a large object: `id, status, title, location, organisers[], challenge_area, start/end_datetime, attendance: { leader_members, assistant_members, participant_members, attendee_members }, invitees[], review: { general_rating, general_tags, scout_method_elements, scout_spices_elements }, owner_type, owner_id, achievement_pathway_oas_data: { award_rule, verifier, groups[{ title, template, version, stage, stream, branch, statements[] }] }, achievement_pathway_logbook_data` |
| `GET /events/{eventId}/invitees` | 200 `{ results: { id, first_name, last_name, member_number }[433] }`. It lists **all 433 group members**, not just those invited. Likely the whole-group member list that `groups/{id}/members` truncates to 140. |
| `GET /units/{id}/proposals` | 200 `{ results: { id, title, description, justification, leader, challenge_area, created_at, proposing_member: { id, first_name, last_name }, uploads: [] }[] }` (1, 7, 10, 0) |
| `GET /proposals/{id}` | 200, same fields (`proposing_member` also has `patrol_name`) |
| `GET /units/{id}/events…`, `/patrols/{id}/events…`, `/groups/{id}/events…` | rejected with the date parameters tried |

### templates host

Path pattern `/{type}/{section}/{doc}/latest.json`; sections `joey, cub, scout, venturer, rover`. 450 paths tried. Present:

| Type | Docs that exist (every section) |
|---|---|
| `milestone` | `overview`, `review/1`, `review/2`, `review/3` (no `plan`) |
| `sia` | `overview`, `plan`, `review` |
| `intro-scouting`, `intro-section`, `oas` | `overview` only |
| `adventurous-journey` | `overview`, `plan`, `review` |
| `personal-reflection` | `overview`, `plan`, `review`, `do` |
| `personal-development-course` | `overview` (checked for `scout`) |
| `peak-award`, `course-reflection`, `personal-development` | none found under those names |

- `GET /oas/{stream}/tree.json` lists the stage template paths in `template_link` (27 per stream for alpine), e.g. `oas/alpine/1/latest.json` and `oas/alpine/cross-country-skiing/4/latest.json`. Fetching one: 200, `{ template, meta: { stage, stream }, version, document: [{ title, input_groups: [{ title, inputs: [] }] }] }`.
- `GET /additional-awards/{id}.json` (tried `camper_award_10`) → **404**, non-JSON body.
- A missing template returns 404 with a small non-JSON body (about 300 bytes).

### agenda host

| Call | Result |
|---|---|
| `GET /units/{unitId}/member-agenda` | 200 `{ count, generated_on, items[] }`. Joey, Cub and Venturer: 0 items (77 bytes). Scout: 7 items. |
| item | `{ date, title, scope, type: "submission", data: { achievement, submission, member, unit_permissions } }`, the same data as a pending submission. So the agenda is the pending-approval list. |
| `GET /members/{id}/psl-agenda` | rejected (support-leader call) |

## What this changes

1. **Per-member loops are not required.** Unit-level `achievements?type=…` returns every member's records for a unit, and the unit metrics call gives per-member summaries. Neither needs an assumed profile. An assume, read, revoke loop per member is unnecessary for these reads. (Assuming was still needed for logbooks, which are per member.)
2. **Paging does not exist on the achievements host and is rejected by the members host.** For a group-wide member list, `/events/{id}/invitees` returns all 433; check whether the unit endpoints are sufficient instead.
3. **A 403 means "no permission for that unit"** (submissions, support-leader calls). Treat as empty.
4. **Do not rely on the `?type=` list** being complete beyond the ten verified values; unknown values return 400 with a readable error.
5. **Large unit responses.** Plan for MB-sized responses, and for the no-`type` call to fail on large units.

## Not callable or not tried

Writes (create, update, delete, submit, import, calendar `PUT`, assume and revoke `POST`s except the ones observed), admin endpoints, `GET /members/?branch=&member_number=` (arbitrary member lookup), `GET /calendar-feeds/…` (secret feeds), `/uploads` routes (rejected), `GET /branches/{id}/groups` (no branch id in this account), `GET /support-leader-groups` (403).

# Follow-up: code-level findings (2026-11-13)

Read from the web app's bundles. **None of these write calls were made.** Success codes and request bodies are what the web app sends and expects; response bodies for writes are unknown.

## Corrections to the read sweep

- `/groups/{id}/events`, `/units/{id}/events`, `/patrols/{id}/events` and `/events` are **create-event `POST`s** in the web app, not reads. That is why plain `GET` attempts at them are rejected. The read for events is `GET /members/{id}/events?start_datetime=&end_datetime=` (ISO strings, optional extra query appended by the app).
- `/groups/{id}/uploads`, `/units/{id}/uploads` (events host) and `/members/{id}/uploads` (achievements host) are file-upload targets, not `GET` reads.
- `/calendar-feeds/{memberId}/{feedId}` is a URL the app builds to show the user (an iCal feed), not an API call it makes. Creating a subscription is `POST /members/{id}/calendar-subscription` with body `{}`.
- Terrain's own client never passes paging params, with one exception: `GET members /branches/{branchId}/groups?…&limit={n}` (a group search in the support-leader access flow) does take `limit`. `{branchId}` comes from the app's user store; the profiles response for this account has no branch.
- The templates host also serves static files under `/resources/…`, linked from template `info_box.links[].path`: SIA prompter PDFs per section (`/resources/sia/{joey-scout|cub-scout|scout|venturer-scout|rover-scout}/sia_prompters.pdf`), OAS detail PDFs (`/resources/oas/{section}/oas_detail.pdf`) and Adventurous Journey PDFs. All returned 200, 0.4 to 1.6 MB.

## Write calls (web app code, not called)

| Call | Body | Success |
|---|---|---|
| `PATCH events /events/{id}` (update, conclude) | event body (built by the app) | 204 |
| `DELETE events /events/{id}`, `/proposals/{id}` | none | |
| `POST events /units/{id}/proposals` | proposal object | 201 |
| `POST events /{groups\|units\|patrols}/{id}/events`, `/events` | event body | |
| `PUT events /members/{id}/calendars` | the calendars object (`own_calendars`, `other_calendars` with `selected` flags) | 200 |
| `POST events /members/{id}/calendar-subscription` | `{}` | |
| `PUT achievements /units/{u}/achievements/manage-oas-stage` | `{ items }` | 200 |
| `PUT achievements /units/{u}/achievements/milestone-import-add-credits` | `{ items }` | 200 |
| `PUT achievements /units/{u}/achievements/milestone-import-award-stage` | `{ section, stage, member_id, date_awarded }` | 200 |
| `PUT achievements /units/{u}/bulk-award-sia` | `{ plan_template, plan_version, member_ids, answers, uploads }` | 200 |
| `PUT achievements /units/{u}/members/{m}/import-achievements/introduction` | `{ achievement_type, date_awarded, section, action }` | 200 |
| `POST achievements /members/{id}/create-achievement` | `{ section, stage }` | 200 |
| `PUT achievements /members/{id}/skip-achievement/{aid}` | `{ skipped_by, skip_notes }` | 200 |
| `PUT achievements /members/{id}/unskip-achievement/{aid}` | | |
| `POST achievements /submissions/{id}/assessments` | `{ outcome: "approved" \| …, comment }` | 204 |
| `POST achievements /members/{id}/achievements/{aid}/archive?submission_id=` | none | 204 |
| `POST/PUT/DELETE achievements /members/{id}/achievements[/{aid}]`, `/logbook[/{id}]`, `/import-achievements[/submit]` | achievement or logbook record | |
| `POST members /members/{id}/assume-profiles`, `/revoke-assumed-profiles` | none | 204 (observed) |
| `PUT members /groups/{g}/units/{u}/members/{m}` | `{ duty, unit_council }` | 204 |
| `DELETE members /groups/{g}/units/{u}/members/{m}`, `/groups/{g}/units/{u}` | | |
| `POST members /groups/{g}/units`, `/groups/{g}/patrols`, `/units/{u}/patrols` | `{ name, unit_id }` for patrols | |
| `PATCH members /users/{username}` | `{ email }` | |
| `POST members /support-leader-access` | `{ member_id, group_id, access_level, action: "REQUEST", request_note }` | |
| `POST members /members/{id}/reset` | none | 200, **body contains the new password** |
| `POST members /members/expired_password_reset` | `{ member_number, branch }` | |

Do not call `reset` or the support-leader and admin routes from a read-only client.

## Still not seen

- The login-failure response from Cognito (Cognito's standard `{ __type, message }` body is expected). Producing one means a failed login on a real account, so it was not attempted.
- `GetUser` attribute names (needs the access token, which the page script did not capture).
- Write responses, `branches/{id}/groups`, `members/?branch=&member_number=`, `psl-agenda`.

# Permission model and roster findings (2026-11-13)

Checked with no profile assumed (`/profiles` returned the account's 4 profiles, none with `datetime_expires`).

## Which per-member reads need an assumed profile

For another member in the Scout unit (not the signed-in user):

| Call | Without assuming |
|---|---|
| `GET achievements /members/{id}/achievements?type=outdoor_adventure_skill` | **200** (36 entries, 234 KB) |
| `GET achievements /members/{id}/achievements?type=milestone` | **200** |
| `GET achievements /members/{id}/achievements` (no type) | **200** (66 entries, 419 KB) |
| `GET achievements /members/{id}/peak-award-progress?section=scout` | **200** |
| `GET achievements /members/{id}/logbook` | **403** |
| `GET achievements /members/{id}/logbook-metrics` | **403** |

So **assuming a profile is only needed for logbooks** (and presumably logbook detail). Every achievement read works for a leader without it, per member or per unit.

## Roster endpoints are not interchangeable

- `GET /groups/{id}/members` (140): all `status: active`; `role` is `leader` 42, `member` 98; 27 have no unit, 93 have a patrol, 47 do not.
- `GET /events/{id}/invitees` (433) is a superset: every one of the 140 appears in it.
- Of the 293 members only in the invitees list, a sample of 40 returned **403** on `GET /members/{id}`, so they are probably not current members of this group (inactive, archived or from other groups). Unproven.
- The 4 units hold 115 distinct members (130 unit-member rows, so 15 are in more than one unit). **3 of those 115 are missing from `groups/{id}/members`.** Do not use the group list as the youth roster; build it from the unit endpoints.

## Cognito token and user attributes

- The id token's claims are only standard ones: `sub, email_verified, amr, iss, cognito:username, aud, acr, event_id, token_use, auth_time, exp, iat, email`. **No custom attributes** (no branch, group or role).
- `GetUser` attributes: `email`, `email_verified`, `sub`.
- Roles, branch and group therefore come only from `GET /profiles` (and `/members/{id}`), never from the token.
- `exp` and `iat` are numeric epoch seconds, so token expiry can be read without a clock offset guess. Refreshing a minute before expiry is a safe margin.

## Login errors seen in the web app's code (not triggered)

`NotAuthorizedException` with message "Temporary password has expired and must be reset by an administrator." is handled by `POST /members/expired_password_reset` `{ member_number, branch }`. Cognito error bodies use `{ __type, message }`.

# CORS, filters and limits (2026-11-13)

## CORS: a static SPA on any origin can call Terrain

Unauthenticated `OPTIONS` preflights, sent from this machine with `Origin` set to `http://localhost:5173`, `https://nomisnostab.github.io` and `https://example.com`, all got `200` with `access-control-allow-origin: *` on every host, once the path was a route that host serves (a path a host doesn't serve returns 403 on the preflight).

| Host | Allowed request headers | Allowed methods |
|---|---|---|
| members, achievements, events, agenda, metrics | `authorization, content-type` | `DELETE, GET, PATCH, POST, PUT` (the members host also lists `OPTIONS`) |
| templates | `authorization, accept` | `GET` only |
| Cognito (`cognito-idp`) | `content-type, x-amz-target, x-amz-user-agent` | `POST` |

Implications for browser clients:

- No proxy is needed; a browser-only client works.
- **Do not add other custom headers** to API calls (for example `Cache-Control` or `X-Amz-User-Agent` on non-Cognito calls). Only the listed headers pass preflight. `Accept` is safelisted and fine.
- Cognito calls may send `X-Amz-User-Agent` but not `Cache-Control`, even though the web app's own calls list it.
- Send the **id token** in `authorization`, no `Bearer` prefix.

## Unit achievement filters

`GET achievements /units/{id}/achievements?type=…` (Scout unit, `type=milestone`, 200 results):

| Extra param | Effect |
|---|---|
| `stage=2` | **filters** (62 results) |
| `section=scout` | **filters** (86 results). The unfiltered 200 contained milestones from all sections: cub 92, scout 86, joey 16, venturer 6. |
| `stream`, `branch`, `stage` (OAS) | filter (see earlier) |
| `status=awarded` | ignored (same 200) |
| `member_id=…` | ignored (same 200) |
| `limit`, `offset`, `page`, `next_token` | ignored |

So a member's milestone history spans their earlier sections. Filter on `section`, never assume a record belongs to the unit's section.

Unit-level `adventurous_journey`, `course_reflection`, `personal_reflection` (empty for Joey and Cub):

| | Scout | Venturer |
|---|---|---|
| adventurous_journey | 12 entries (11 members), 341 KB, `awarded` 11 and `draft_approval` 1; `uploads` on 11, `can_archive` on 1 | 4 entries, 234 KB, all `awarded`, `uploads` on all 4 |
| course_reflection | 27 (22 members), 38 KB; `awarded` 25, `draft_review` 2 | 12 (10 members), 24 KB; `imported` on 4 |
| personal_reflection | 11, 75 KB; `awarded` 10, `pending_review` 1 | 5, 17 KB |

All carry `template, version, answers, last_updated, latest_submission` on most entries and `date_awarded` on awarded ones. `imported` appears only on Venturer course reflections.

## Events: ranges, CSV and errors

`GET events /members/{id}/events?start_datetime=&end_datetime=`:

| Case | Result |
|---|---|
| ISO with `Z` (`2026-01-01T00:00:00.000Z`) | 200 (as the web app sends) |
| One year | 168 events, 68 KB |
| 2020 to 2030 | 1,333 events, 542 KB |
| 2015 to 2035 | 1,334 events, 542 KB. **No range limit seen.** |
| One month | 18 events |
| `end_datetime` missing | **400** `{ "errors": ["end_datetime required"] }` |
| no params | rejected (opaque) |
| `&download_format=csv&download_tz=Australia/Melbourne` | 200, **CSV text** (18 KB for 2026) |

CSV layout: line 1 is a title `Programming [dd-mm-yyyy to dd-mm-yyyy]` padded with commas; line 2 is the header `Start date,Start time,End date,End time,Location,Activity name,Who is this activity for?,Who is leading?,Who is assisting?`; following lines hold the data with `dd/mm/yyyy` dates and 12-hour `hh:mm AM` times in the requested timezone, quoted where the value contains commas. The server does the export, so the Programming report's data could come straight from this.

## Other points

- `GET /members/{id}/events` returns events the member is invited to, including unit and group activities (`invitee_type`, `invitee_name`).
- The `metrics` host serves no routes the web client calls. A preflight to `/units/{id}/members` succeeded, so that route exists.

# Achievement entries: field-by-field (Scout unit, 2026-11-13)

Built by merging every entry of each unit-level response (counts are `present/total`). Use this, not the earlier single-entry samples, to type the responses.

## Fields common to all types

Always present: `id, member_id, section, type, status, status_updated`. Everything below is optional.

| Field | Where | Notes |
|---|---|---|
| `template`, `version` | recorded in Terrain | absent on imported records (milestone 55/200, SIA 123/126, intro_section 48/64) |
| `answers` | recorded | object; value types vary per type (below) |
| `last_updated` | most | milestone 60/200, SIA 120/126, intro_section 45/64 |
| `latest_submission` `{ submission_id, unit_id, submission_time }` | submitted records | milestone 53/200, SIA 114/126, intro_section 47/64, AJ 11/12, personal reflection 11/11, peak 2/2 |
| `imported` `{ date_awarded?, awarded_by: { id, name }, event_count? }` | imported records | milestone 31/200; `imported.event_count` (participant, assistant, leader × community, outdoors, creative, personal_growth) on 19 of those |
| `date_awarded` | awarded, recorded in Terrain | **top level**, `yyyy-mm-dd`: milestone 27 of 55 awarded, SIA 67/97, intro_scouting 13/43, intro_section 21/62, OAS (cycling) 14/24, AJ 8/11, personal reflection 7/10 |
| `uploads` `{ id, filename, bucket, key, url, uploaded_on }[]` | with attachments | SIA 33/126, AJ 11/12, milestone 1/200 |
| `can_archive` | boolean | SIA 9/126, AJ 1/12 |
| `expiry_date` | additional award | 6/85 |

`achievement_meta` by type:

| Type | Keys |
|---|---|
| milestone | `{ stage }` |
| outdoor_adventure_skill | `{ stage, stream, branch }` |
| special_interest_area | `{ sia_area, plan_type, review_type }` on 92/126; **absent on 34** |
| additional_award | `{ additional_award_id }` |
| others | none |

Milestone-only: `milestone_requirement_status`, `event_count` (participant, assistant, leader), `event_log[]` (184/200). A **skipped** milestone has `status: not_required` and `answers: { skipped_on, skipped_by, skip_notes, previous_status }`.

`answers` value types: milestone strings and arrays; SIA strings and arrays (answer keys such as `project_name`, `special_interest_area_selection`, `main_activity`, `who_is_involved`, `step_1`..`step_3`, `attach_documentation[]`); AJ objects, strings, booleans and arrays; intro_section booleans (930) and strings (140); personal reflection strings and arrays; course reflection strings, arrays and objects. So `answers` is not a string-to-string map.

## Award date: three sources

`date_awarded` (top level), `imported.date_awarded`, and `status_updated`. In the Scout unit, an awarded record never had both of the first two. Counts of awarded records with each: milestone 27 / 2 / neither 26; SIA 67 / 3 / 27; intro_scouting 13 / 17 / 13; intro_section 21 / 15 / 26; OAS 14 / 2 / 8.

Where a top-level `date_awarded` exists it differs from the `status_updated` day in about half the cases (milestone 14 of 27, up to 22 days; SIA 34 of 67, up to 168 days; intro_scouting 11 of 13 and intro_section 15 of 21, up to 427 days).

## Submissions (`/units/{id}/submissions`)

Scout unit, one call each:

- **pending**: 10 entries (`type` award 7, review 3).
- **finalised**: **976 entries, 906 KB, covering 2020-11-04 to 2026-11-12, 78 members**. `submission.type`: review 605, approval 51, award 320. `submission.status` is `finalised`. Achievement types in it: OAS 639, SIA 110, intro_section 88, milestone 60, course_reflection 36, intro_scouting 26, adventurous_journey 10, personal_reflection 6, peak_award 1.

Entry:

```
{ achievement: { id, type, achievement_meta: { stage?, stream?, branch?, sia_area?, plan_type?, review_type? } },
  submission: { id, type, date, status, actioned_by: ActionedBy[], outcome?: string },   // outcome on finalised only
  unit_permissions: { write: boolean },
  member: { id, first_name, last_name },
  submitted_by?: { id, first_name, last_name } }                                         // 257/976
ActionedBy { member_id, member_first_name, member_last_name, outcome, time, comment: string|null, date_awarded: string|null }
```

This is a six-year activity log in one request.

## Typing notes: places a naive model breaks

| Area | Note |
|---|---|
| `answers` | Not a string-to-string map. Values are booleans, arrays and objects as well as strings, so type it as `Record<string, unknown>`. A parser that expects strings fails on the array-valued upload answers. |
| Award dates | Many records carry a top-level `date_awarded`. A model built only on `imported.date_awarded` and `status_updated` misses it. |
| `imported.date_awarded` | Present on few awarded milestones (2 of 55 in the Scout unit). |
| `metadata` | The key is `achievement-import` (hyphenated). |
| Approvals | `submission.outcome`, `submitted_by` and `actioned_by[].comment` all exist (`comment` was `null` in the data seen). |
| Optional fields | Everything except `id, member_id, section, type, status, status_updated` can be absent on an achievement. |

**Award date rule (decided 2026-11-13):** prefer `date_awarded`, then `imported.date_awarded`, then `status_updated`. `date_awarded` agrees with the `status_updated` day only about half the time on records that have both; the 25 to 50% of awarded records with neither have only `status_updated`.

# Write calls tested: patrols, duties and unit leader (2026-11-13)

Tested on the signed-in leader's **own member record only** (an adult leader, never a child's record), one change at a time. Each change was followed by a read to confirm it, and every change was reverted. After the last revert the member record was identical to the pre-test snapshot, and all four units' member lists matched it exactly (0 differing rows across 130 unit member rows; the group's 17 patrols unchanged). The tests used an empty patrol and a unit with no unit leader, so no other member's role could have been affected.

## Results

| Action | Call | Body | Result |
|---|---|---|---|
| Add to a patrol | `PUT members /groups/{g}/patrols/{p}/members/{m}` | `{ "duty": "member" }` | 204, empty body |
| Make patrol leader | same | `{ "duty": "patrol_leader" }` | 204; the member's patrol duty read back as `patrol_leader` |
| Make assistant patrol leader | same | `{ "duty": "assistant_patrol_leader" }` | 204; read back as `assistant_patrol_leader` |
| Back to plain member | same | `{ "duty": "member" }` | 204 |
| Unknown duty | same | `{ "duty": "bogus_duty" }` | **400** `{ "errors": { "duty": ["unallowed value bogus_duty"] } }`, nothing changed |
| Move between patrols | `PUT` the new patrol, then `DELETE` the old one (how the web client does it) | duty body on the `PUT` | 204 and 204 |
| Remove from a patrol | `DELETE members /groups/{g}/patrols/{p}/members/{m}` | none | 204 |
| Remove when not a member | same `DELETE` | none | **400**, empty body |
| Make unit leader | `PUT members /groups/{g}/units/{u}/members/{m}` | `{ "duty": "unit_leader", "unit_council": false }` | 204; read back as `unit_leader` on both `/members/{id}` and `/units/{id}/members` |
| `patrol_leader` as a unit duty | same | `{ "duty": "patrol_leader", "unit_council": false }` | **400** `unallowed value patrol_leader`, nothing changed |
| Restore unit duty | same | `{ "duty": "adult_leader", "unit_council": false }` | 204 |

All 204 responses have an empty body. Reads show the change straight away.

## What this shows

- **Two separate levels.** Patrol leader and assistant patrol leader are the member's **patrol duty** (`patrol.duty`). Unit leader and adult leader are the **unit duty** (`unit.duty`). Setting one does not change the other.
- **Allowed values.** Unit duty: `member`, `adult_leader`, `unit_leader`. Patrol duty: `member`, `patrol_leader`, `assistant_patrol_leader`. Each endpoint rejects the other level's values with a 400.
- **Adding to a patrol is a `PUT`; a move is a `PUT` then a `DELETE`.** The server does not remove the old patrol for you. A member can be in **two patrols of the same unit at once** (a member record showed both until the `DELETE`).
- **Patrol duty works for an adult leader.** The patrol `PUT` was accepted while the unit duty stayed `adult_leader`. The web client's duty menu offers patrol leader and assistant patrol leader only to a member who is in a patrol; a member with no patrol is offered only member, adult leader and unit leader.
- **Unit membership** goes through the same URL shape with `units` in place of `patrols`, and includes `unit_council`.
- **Group id in the path.** All of these calls put the group id in the URL (`/groups/{groupId}/…`).

## Not tested, and why

- **Creating a patrol.** The web client creates one with `POST /groups/{g}/patrols` or `POST /units/{u}/patrols` and a body of `{ name, unit_id }`. It has **no call to delete or rename a patrol** (none in any of its 267 bundles), so a created patrol could not be removed again. It was not attempted.
- **Replacing an existing leader.** Whether the server rejects or demotes a second patrol leader, assistant patrol leader or unit leader in the same patrol or unit was not tested, because the only way to find out is to risk changing another member's role.
- **A youth member's record.** Everything above used an adult leader's record. Results for a youth may differ (for example the web client also re-sends the unit membership when a patrol duty is set for a youth).
- **Unit council.** `unit_council` was only sent unchanged. Changing it can change a person's permissions, so it was not exercised.
