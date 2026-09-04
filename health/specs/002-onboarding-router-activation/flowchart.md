# Flowcharts: Onboarding → ABHA Link → Sync Consent

Companion diagrams for [`spec.md`](./spec.md) (onboarding/router/ABHA linking)
and [`../003-abdm-sync-subscription/spec.md`](../003-abdm-sync-subscription/spec.md)
(sync consent + revoke). A rendered, styled version of these three diagrams
was also published as a standalone visual for review — see PR discussion.

## Diagram 1 — Account to active sync (the full path)

Node color key: **router** (the one branching decision) · **decision**
(system/CM checks) · **async** (background job, dashed) · **deferred**
(skip/not-now, re-prompted at most once) · **terminal** (Home or a completed
outcome).

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'primaryColor':'#FFFFFF','primaryBorderColor':'#3194ff','primaryTextColor':'#52a6fb','lineColor':'#1c85f5','fontFamily':'ui-monospace, SFMono-Regular, Consolas, monospace','fontSize':'13px','edgeLabelBackground':'#FFFFFF'}}}%%
flowchart TD
    A["Open app"] --> B["Phone number + OTP"]
    B --> C{"Account created?"}
    C -->|"yes"| D[["Router: who is this for?"]]

    D -->|"Just me"| E1["Self profile auto-created"]
    D -->|"Me and family"| E2["Prompt: add first family member"]
    D -->|"Setting up for someone else"| E3["Guided assisted setup"]

    E2 --> F{"How will they join?"}
    F -->|"No device of their own"| G1["Assisted add: name / relation / DOB"]
    F -->|"Has their own phone"| G2["Generate family QR or code"]
    G2 --> G3["Invitee: phone + OTP, own device"]
    G3 --> G4["Enter family code"]
    G4 --> G5["Attached to family, own account"]

    E3 --> G1
    E1 --> H(("Home — persona-weighted actions"))
    G1 --> H
    G5 --> H

    H --> I1["Upload a record"]
    H --> I2["Add another family member"]
    H --> I3["Create or link ABHA"]
    I1 --> H
    I2 --> E2

    I3 --> K["Enter Aadhaar number"]
    K --> L["OTP to Aadhaar-linked mobile"]
    L --> M{"ABDM lookup: ABHA exists?"}
    M -->|"Yes"| N1["Link existing ABHA"]
    M -->|"No"| N2["Create new ABHA"]
    N1 --> O["ABHA verified + linked to profile"]
    N2 --> O

    O --> P{"Sync permission — single Allow, editable scope"}
    P -->|"Allow"| Q["Subscription + first-fetch consent, one action"]
    P -->|"Not now"| R["Nothing breaks — re-prompt once, contextually"]
    Q --> S(("Return to app immediately"))
    S -.-> T[("Backfill job — runs in background")]
    T -->|"records found"| U(("Timeline updates + push notification"))
    T -->|"no data yet"| V["Empty state — add manually anytime"]

    R --> H
    U --> H
    V --> H

    classDef router fill:#2F5FC4,stroke:#1E3F87,color:#FFFFFF,stroke-width:2px
    classDef decision fill:#E8F0FC,stroke:#2F5FC4,color:#3194ff,stroke-width:1.5px
    classDef terminal fill:#E3F3E9,stroke:#1F7A4C,color:#3194ff,stroke-width:1.5px
    classDef deferred fill:#FBF0D9,stroke:#9A6B08,color:#3194ff,stroke-width:1.5px
    classDef async fill:#F1F2F6,stroke:#3194ff,color:#3194ff,stroke-width:1px,stroke-dasharray:4 3

    class D router
    class C,F,M,P decision
    class H,U terminal
    class R,V deferred
    class T async
```

## Diagram 2 — ABHA & sync state

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'primaryColor':'#FFFFFF','primaryBorderColor':'#3194ff','primaryTextColor':'#3194ff','lineColor':'#3194ff','fontFamily':'ui-monospace, SFMono-Regular, Consolas, monospace','fontSize':'13px'}}}%%
stateDiagram-v2
    [*] --> NoABHA
    NoABHA --> AadhaarOTPPending: enters Aadhaar number
    AadhaarOTPPending --> Verified: OTP confirmed
    AadhaarOTPPending --> NoABHA: OTP failed / abandoned (resumable)
    Verified --> Linked: exists = true
    Verified --> Created: exists = false
    Linked --> SyncPending
    Created --> SyncPending
    SyncPending --> SyncActive: Allow tapped
    SyncPending --> SyncDeferred: Not now
    SyncDeferred --> SyncPending: contextual re-prompt (max once)
    SyncActive --> SyncRevoked: user toggles off
    SyncRevoked --> [*]

    classDef good fill:#E3F3E9,stroke:#1F7A4C,color:#121A22
    classDef warn fill:#FBF0D9,stroke:#9A6B08,color:#121A22

    class SyncActive good
    class SyncDeferred,SyncRevoked warn
```

## Diagram 3 — Revoke

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'primaryColor':'#FFFFFF','primaryBorderColor':'#3194ff','primaryTextColor':'#3194ff','lineColor':'#3194ff','fontFamily':'ui-monospace, SFMono-Regular, Consolas, monospace','fontSize':'13px'}}}%%
flowchart LR
    A2["Family & Consent screen"] --> B2["Select profile"]
    B2 --> C2["Toggle: stop automatic sync"]
    C2 --> D2["Cancel subscription at CM"]
    D2 --> E2["Revoke standing consent artifacts"]
    E2 --> F2(("Local records retained — stated on screen"))

    classDef danger fill:#FBE8E6,stroke:#B2382F,color:#3194ff,stroke-width:1.5px
    classDef terminal fill:#E3F3E9,stroke:#1F7A4C,color:#3194ff,stroke-width:1.5px

    class C2,D2,E2 danger
    class F2 terminal
```
