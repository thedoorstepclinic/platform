# Quickstart: TDC PHR Prototype

Stand up the demo world and walk the golden path. Prototype-grade; commands are
indicative until the scaffolds land. **Replanned 2026-08-15:** paths and app
commands updated for the Flutter stack and the `health/apps/`+`health/services/`
monorepo layout (`docs/repo-structure.md`), replacing the earlier Expo RN /
flat-repo-root version. **Amended 2026-08-20:** the two services moved out of
`health/` to `platform/services/` — core-api now serves every TDC client, not
just the patient app (see `docs/repo-structure.md`). Commands below reflect the
new paths.

## Prerequisites
- Flutter SDK, Python 3.11, Postgres.
- Android device/emulator (primary) or web target.
- 2 physical NTAG 424 cards + 1 spare (for the full card beat).

## 1. Core API (Django/DRF)
```bash
cd services/core-api
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python seed_demo.py            # rebuilds Rohan / Asha / Prakash demo world
python manage.py runserver 0.0.0.0:8000
```
`DEMO_MODE=1` enables mock OTP `000000`.

## 2. Fastlane (FastAPI)
```bash
cd services/fastlane
uvicorn main:app --host 0.0.0.0 --port 8100
```
Serves `GET /e/{uid}` from the `emergency_payload` snapshot. In production this
is `https://e.thedoorstepclinic.com`.

## 3. App (Flutter)
```bash
cd health/apps/health
flutter pub get
flutter run -d android          # or: flutter run -d chrome (web target)
```
Point the API client at the Core API host; ensure the device is on the same
network (or use the hotspot fallback).

## 4. Reset between demos (Principle V)
```bash
cd services/core-api && python seed_demo.py     # one command, instant clean state
```

## Golden path (5-minute script)
1. Launch → login as Rohan (OTP `000000`).
2. Home shows Rohan / Asha / Prakash.
3. Tap **Asha** → seeded timeline. Camera-upload a paper Rx → appears live.
4. **Health Summary** → renders → share PDF via WhatsApp.
5. **Meds** → Metformin schedule; 8pm reminder fires; stock "4 days left".
6. **Money shot:** second logged-out phone taps Asha's card → responder page
   in <2s (blood group, allergies, conditions, meds, contacts) → Rohan's and
   Soham's phones buzz: *"Asha's emergency card was just scanned near Kothrud."*
7. **Trust screen** → encrypted · stored in India · every access logged · you
   control sharing.

## Pre-demo checklist
- ☐ `seed_demo.py` re-run · ☐ 2 cards written + spare · ☐ demo phone +
  responder phone (NFC on, logged out) + Soham's phone · ☐ hotspot fallback ·
  ☐ backup screen-recording · ☐ airplane-mode test.

## Card personalization (D10)
Write the SDM URL to each NTAG 424:
`https://e.thedoorstepclinic.com/e/{uid}?ctr={counter}&cmac={cmac}` with SDM
mirroring the counter + CMAC. Test-scan on 3+ Android phones before demo day
(NFC read variability is a top risk).
