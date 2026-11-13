# Terrain API documentation

An unofficial reference to the Scouts Australia **Terrain** API: every call found, which level of the organisation it belongs to (group, unit, patrol, member, you), and what it returns.

**Read it here: https://tymondomanko.github.io/Terrain-API-Documentation/**

## What is in this repository

| File | What it is |
|---|---|
| `index.html` | The reference page (published with GitHub Pages). Calls sorted by level, a "I want to know…" finder, achievement fields, rules and gotchas. |
| `docs/terrain-api.md` | Response shapes, field by field, with counts and sizes from a real group. |
| `docs/terrain-api-map.md` | Every call the Terrain web client can make, by host, with which ones were checked against live responses. |

## How it was made

Observed on 2026-11-13 from a signed-in Terrain session for one NSW group with four units. Responses were recorded as **structure only** (field names, types and enum values): no member data, tokens or ids are stored here. Calls were reads only. The write calls listed come from the web client's code and were never called.

## Cautions

- Not affiliated with or endorsed by Scouts Australia. The API is undocumented and may change or be restricted at any time.
- The data behind it is personal information about members, many of them children. Use it only for what you are authorised to do, and respect Scouts Australia's terms.
- Do not call the write, password reset, admin or support-leader routes from tooling.
