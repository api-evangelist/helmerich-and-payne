---
name: validate-mwd-survey
description: >-
  Run ISCWSA OWSG Rev-2 dynamic quality control on a single MWD survey station using the MagVAR
  Survey Validation API (Helmerich & Payne). Returns a Green/Orange/Red verdict for magnetic field
  strength, dip and gravity, an anti-collision sigma distance, and the random and systematic
  components of inclination and azimuth uncertainty.
api: MagVAR Survey Validation API
base_url: https://fac-api.magvar.com
operations:
  - uncertaintyValuesUsingGET
generated: '2026-09-13'
method: generated
source: >-
  openapi/helmerich-and-payne-magvar-survey-validation.json (operationId verified against the
  provider's own Swagger document) plus a live 200 observed on 2026-09-13.
---

# Validate an MWD survey station

Use this when you have one measurement-while-drilling survey station and need to know whether it
passes quality control before it is accepted into a survey program.

## What you need first

All nineteen inputs are **required** — the API rejects a partial request, and it reports only the
**first** missing or invalid field, so assemble everything before calling.

| Group | Parameters |
|---|---|
| Location | `latitude` (WGS84, -90..90), `longitude` (WGS84, -180..360), `depthBelowMSL` (-20000..60000, negative is above MSL), `depthUnits` |
| Orientation | `wellboreInclination` (0..180), `wellboreAzimuth` (true, -180..360) |
| Measured | `measuredBTotal` (0..80000), `measuredDip` (-90..90), `measuredGTotal` (0..2000000) |
| Reference | `referenceBTotal`, `referenceDip`, `referenceGTotal`, `referenceDeclination` |
| Units | `bTotalUnits`, `gTotalUnits`, `depthUnits` |
| Model | `gravityModel`, `toolCode`, `surveyDate` (ISO 8601), `antiCollisionSigma` |

**Units are never implicit.** Every physical quantity has a paired units enum and you must choose
one. Read the enum values from
`openapi/helmerich-and-payne-magvar-survey-validation.json` rather than assuming — `depthUnits`
alone accepts `METER`, `KILOMETER`, `FOOT`, `FOOT_US`, `YARD`, `MILE`, `SE_FT`, and `FOOT` and
`FOOT_US` are not the same thing.

**Pick the tool code deliberately.** `toolCode` is a closed 22-value enum drawn from the ISCWSA
Rev 2 standard and extended sets (`MWD`, `MWD_AX`, `MWD_IFR1_SAG`, `MWD_HRGM_SAG_MS`, …). It selects
the error model, so it changes the answer. Use the code the survey program already specifies; do
not default to `MWD` because it is first in the list.

## Step 1 — call the operation

Operation: **`uncertaintyValuesUsingGET`** — `GET /uncertaintyValues` on `https://fac-api.magvar.com`.

No authentication. No header. No key. Everything goes in the query string.

## Step 2 — read the verdict

The `SurveyValidation` response has six members:

- `sigmaValidation` — `distance` plus `withinTolerance`. This is the anti-collision check against
  the `antiCollisionSigma` you supplied. `withinTolerance: false` is the hard fail.
- `magneticFieldStrength`, `dip`, `gravitationalFieldStrength` — each a `validationResult` of
  `Green`, `Orange` or `Red`, with the `greenThreshold` and `orangeThreshold` that produced it.
  Report the thresholds alongside the colour; a colour without its threshold is not evidence.
- `inclinationUncertainty`, `azimuthUncertainty` — each a `random` and `systematic` component.
  Do not sum them and present one number: the split is the point, and it is what the ISCWSA model
  produces.

## Step 3 — handle errors

On failure you get HTTP 400 and a **bespoke** envelope — not RFC 9457 problem details:

```json
{"validationErrors":[{"field":"latitude","message":"Must be between -90 and 90 degrees","value":"99999.0"}]}
```

Read `field` (machine-stable) and repair that one parameter. `message` is free text and may change.
Expect to loop: validation stops at the first failure, so a request with several bad values takes
several round trips. A 500 carries no documented schema; retrying is safe.

## Safety notes

- The operation is a **GET with no side effects**. Nothing is created, stored, or billed, and there
  is nothing to undo. Retrying is always safe.
- There is **no rate limit header** of any kind. Back off on your own schedule; the API will not
  tell you when to.
- Use is governed by the [Survey Validation API EULA](https://www.magvar.com/EULA_SurveyValidationAPI.html),
  which grants a one-year single-user licence and forbids redistribution and derivative works. The
  endpoint is technically open, but it is not unlicensed — surface that to the human before running
  this at volume.
- The answer is an input to a wellbore-placement decision. Present the verdict and the thresholds;
  do not restate an `Orange` or `Red` as "close enough".
