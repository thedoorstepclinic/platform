# Data Model: TDC PHR Prototype

Prototype schema. Track B will re-model for production (encryption-at-rest
detail, DPDP retention, audit tables). Types are indicative.

## Entities

### users
| Field | Type | Notes |
|-------|------|-------|
| id | uuid PK | |
| phone | string | E.164; unique |
| display_name | string | |
| created_at | timestamptz | |

### profiles
| Field | Type | Notes |
|-------|------|-------|
| id | uuid PK | |
| owner_id | uuid FK → users | account that owns this profile |
| name | string | e.g. "Asha K." |
| relation | enum | self / mother / father / other |
| dob | date | for age display |
| created_at | timestamptz | |

### caregiver_grants  *(the consent log — Principle II)*
| Field | Type | Notes |
|-------|------|-------|
| id | uuid PK | |
| grantor_id | uuid FK → users | who granted |
| grantee_id | uuid FK → users | who received caregiver access |
| profile_id | uuid FK → profiles | profile the grant is over |
| ts | timestamptz | **the recorded consent moment** |
| revoked_ts | timestamptz null | set on revoke (one toggle) |

### records
| Field | Type | Notes |
|-------|------|-------|
| id | uuid PK | |
| profile_id | uuid FK → profiles | |
| type | enum | rx / lab / discharge / other |
| title | string | manual (no OCR) |
| file | file/url | image or PDF |
| record_date | date | user-set |
| created_at | timestamptz | timeline ordering (newest-first) |
| source | enum | user / hms *(P1)* |

### medications
| Field | Type | Notes |
|-------|------|-------|
| id | uuid PK | |
| profile_id | uuid FK → profiles | |
| name | string | e.g. "Metformin" |
| dose | string | e.g. "500mg" |
| times | string[] | e.g. ["08:00","20:00"] — drives local reminders |
| stock_count | int | units remaining |
| threshold | int | refill threshold; low-stock badge when days-left ≤ 5 |

> **Days-left** = `stock_count / len(times)` (doses per day). Badge at ≤ 5.

### emergency_profiles  *(1:1 with profile)*
| Field | Type | Notes |
|-------|------|-------|
| profile_id | uuid PK/FK → profiles | one-to-one |
| blood_group | string | rendered largest on responder page |
| allergies | string | rendered red |
| conditions | string | |
| abha_no | string | free text (manual); P1 may populate via ABHA M1 |

### emergency_contacts
| Field | Type | Notes |
|-------|------|-------|
| id | uuid PK | |
| profile_id | uuid FK → profiles | |
| name | string | |
| phone | string | tap-to-call on responder page |
| relation | string | |
| priority | int | ordering; 2+ required |

### cards
| Field | Type | Notes |
|-------|------|-------|
| uid | string PK | NTAG 424 UID |
| profile_id | uuid FK → profiles | bound profile |
| sdm_key_ref | string | reference to per-card CMAC key (not the key itself) |
| last_ctr | int | last accepted counter; reject `ctr ≤ last_ctr` |
| active | bool | kill-switch; false ⇒ neutral responder page |

### scan_events
| Field | Type | Notes |
|-------|------|-------|
| id | uuid PK | |
| card_uid | string FK → cards | |
| ctr | int | counter presented |
| ts | timestamptz | |
| ip | inet | |
| geo | string null | coarse location if scanner shared it |

### emergency_payload  *(denormalized snapshot — read by Fastlane)*
Derived, not a source of truth. Refreshed on save of the profile's emergency
profile, meds, or contacts.
| Field | Type | Notes |
|-------|------|-------|
| card_uid | string PK/FK → cards | one per active card |
| profile_name | string | |
| blood_group | string | |
| allergies | string | |
| conditions | string | |
| current_meds | json | [{name, dose}] |
| contacts | json | [{name, phone, relation, priority}] |
| abha_no | string | |
| updated_at | timestamptz | |

## Relationships (summary)
```
users 1───* profiles 1───1 emergency_profiles
users 1───* caregiver_grants *───1 profiles
profiles 1───* records
profiles 1───* medications
profiles 1───* emergency_contacts
profiles 1───* cards 1───* scan_events
cards 1───1 emergency_payload   (denormalized snapshot)
```

## Snapshot refresh rule (FR-010)
On create/update/delete of `emergency_profiles`, `medications`, or
`emergency_contacts` for a profile, rebuild `emergency_payload` for each active
card bound to that profile. This is the ONLY write path Fastlane depends on;
Fastlane never joins Core tables at read time (Principle I).

## State transitions
- **Card:** `linked(active=false)` → `active=true` (link) → `active=false`
  (revoke toggle). Revoked ⇒ responder returns neutral page.
- **Caregiver grant:** `active(revoked_ts=null)` → `revoked(revoked_ts set)`.
- **Counter:** monotonic; a scan with `ctr ≤ last_ctr` is rejected (replay).
