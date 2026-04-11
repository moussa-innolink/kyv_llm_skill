# KyvShield REST KYC API

Server-to-server KYC verification without SDK or WebSocket. Same pipeline as the SDK but in a single HTTP request.

## Step 1: Get Available Challenges

```bash
curl https://kyvshield-naruto.innolinkcloud.com/api/v1/challenges \
  -H "X-API-Key: YOUR_API_KEY"
```

Response tells you what images to provide per challenge mode:
```json
{
  "challenges": {
    "document": {
      "minimal": ["center_document"],
      "standard": ["center_document", "tilt_left", "tilt_right"],
      "strict": ["center_document", "tilt_left", "tilt_right", "tilt_forward", "tilt_back"]
    },
    "selfie": {
      "minimal": ["center_face", "close_eyes"],
      "standard": ["center_face", "close_eyes", "turn_left", "turn_right"],
      "strict": ["center_face", "close_eyes", "turn_left", "turn_right", "smile", "look_up", "look_down"]
    }
  }
}
```

## Step 2: Submit KYC Verification

**Image naming convention:** `{step}_{challenge}` — one JPEG per challenge.

### Full KYC: Selfie (standard) + Recto (minimal) + Verso (minimal)

```bash
curl -X POST https://kyvshield-naruto.innolinkcloud.com/api/v1/kyc/verify \
  -H "X-API-Key: YOUR_API_KEY" \
  -F 'steps=["selfie","recto","verso"]' \
  -F 'target=SN-CIN' \
  -F 'language=fr' \
  -F 'require_face_match=true' \
  -F 'require_aml=true' \
  -F 'kyc_identifier=user-12345' \
  -F 'selfie_challenge_mode=standard' \
  -F 'recto_challenge_mode=minimal' \
  -F 'verso_challenge_mode=minimal' \
  -F 'selfie_center_face=@selfie_centered.jpg' \
  -F 'selfie_close_eyes=@selfie_eyes_closed.jpg' \
  -F 'selfie_turn_left=@selfie_head_left.jpg' \
  -F 'selfie_turn_right=@selfie_head_right.jpg' \
  -F 'recto_center_document=@cin_recto.jpg' \
  -F 'verso_center_document=@cin_verso.jpg'
```

### Recto + Verso (standard document challenges, no selfie)

```bash
curl -X POST https://kyvshield-naruto.innolinkcloud.com/api/v1/kyc/verify \
  -H "X-API-Key: YOUR_API_KEY" \
  -F 'steps=["recto","verso"]' \
  -F 'target=SN-CIN' \
  -F 'language=fr' \
  -F 'recto_challenge_mode=standard' \
  -F 'verso_challenge_mode=minimal' \
  -F 'recto_center_document=@cin_recto_flat.jpg' \
  -F 'recto_tilt_left=@cin_recto_tilted_left.jpg' \
  -F 'recto_tilt_right=@cin_recto_tilted_right.jpg' \
  -F 'verso_center_document=@cin_verso.jpg'
```

### Recto Only (minimal)

```bash
curl -X POST https://kyvshield-naruto.innolinkcloud.com/api/v1/kyc/verify \
  -H "X-API-Key: YOUR_API_KEY" \
  -F 'steps=["recto"]' \
  -F 'target=SN-CIN' \
  -F 'language=fr' \
  -F 'recto_challenge_mode=minimal' \
  -F 'recto_center_document=@cin_recto.jpg'
```

### Selfie Only (minimal)

```bash
curl -X POST https://kyvshield-naruto.innolinkcloud.com/api/v1/kyc/verify \
  -H "X-API-Key: YOUR_API_KEY" \
  -F 'steps=["selfie"]' \
  -F 'language=fr' \
  -F 'selfie_challenge_mode=minimal' \
  -F 'selfie_center_face=@selfie_centered.jpg' \
  -F 'selfie_close_eyes=@selfie_eyes_closed.jpg'
```

## Response

```json
{
  "success": true,
  "session_id": "abc123...",
  "overall_status": "pass",
  "overall_confidence": 0.95,
  "face_verification": {
    "is_match": true,
    "similarity_score": 87.5
  },
  "steps": [
    {
      "step_index": 0,
      "step_type": "selfie",
      "success": true,
      "liveness": { "is_live": true, "score": 0.98, "confidence": "HIGH" },
      "verification": { "is_authentic": true, "confidence": 0.98, "checks_passed": ["center_face", "close_eyes", "turn_left", "turn_right"] },
      "captured_image": "base64...",
      "processing_time_ms": 2100
    },
    {
      "step_index": 1,
      "step_type": "recto",
      "success": true,
      "liveness": { "is_live": true, "score": 0.92, "confidence": "HIGH" },
      "aligned_document": "base64...",
      "extraction": [
        {"key": "document_id", "document_key": "numero_carte", "label": "N carte", "value": "1 06 19930515 00026 8", "display_priority": 1},
        {"key": "first_name", "document_key": "prenoms", "label": "Prenoms", "value": "MOUSSA", "display_priority": 2}
      ],
      "extracted_photos": [{"image": "base64...", "confidence": 0.85, "bbox": [94, 218, 210, 371], "area": 17748, "width": 116, "height": 153}],
      "processing_time_ms": 5200
    },
    {
      "step_index": 2,
      "step_type": "verso",
      "success": true,
      "liveness": { "is_live": true, "score": 0.95, "confidence": "HIGH" },
      "aligned_document": "base64...",
      "extraction": [
        {"key": "nin", "document_key": "nin", "label": "NIN", "value": "1548199301837", "display_priority": 1},
        {"key": "mrz", "document_key": "mrz", "label": "MRZ", "value": "I<SEN1061993...", "display_priority": 2},
        {"key": "code_pays", "document_key": "code_pays", "label": "Code Pays", "value": "SEN", "display_priority": 3}
      ],
      "verification": { "is_authentic": true, "confidence": 0.95, "checks_passed": ["center_document"] },
      "processing_time_ms": 4800
    }
  ],
  "aml_screening": {
    "status": "clear",
    "risk_level": "low",
    "is_sanctioned": false,
    "is_pep": false,
    "matches": [],
    "screened_against": ["ofac", "un", "eu", "uk", "fr"],
    "screened_at": "2026-04-10T12:00:00Z",
    "total_entries_checked": 75746
  },
  "processing_time_ms": 15000
}
```

## Parameters Reference

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `steps` | JSON array | Yes | `["selfie"]`, `["recto"]`, `["recto","verso"]`, `["selfie","recto","verso"]` |
| `target` | string | If document steps | `SN-CIN`, `SN-PASSPORT`, `SN-DRIVER-LICENCE` |
| `language` | string | Yes | `fr`, `en`, `wo` |
| `selfie_challenge_mode` | string | If selfie in steps | `minimal`, `standard`, `strict` |
| `recto_challenge_mode` | string | If recto in steps | `minimal`, `standard`, `strict` |
| `verso_challenge_mode` | string | If verso in steps | `minimal`, `standard`, `strict` |
| `require_face_match` | string | No | `true` to compare selfie vs document face |
| `require_aml` | string | No | `true` to enable AML sanctions screening |
| `kyc_identifier` | string | No | Your reference ID (returned in webhooks) |

## Image Files Reference

| File field name | When required | Description |
|----------------|---------------|-------------|
| `selfie_center_face` | selfie (all modes) | Face centered, looking at camera |
| `selfie_close_eyes` | selfie (all modes) | Both eyes closed |
| `selfie_turn_left` | selfie (standard, strict) | Head turned to the left |
| `selfie_turn_right` | selfie (standard, strict) | Head turned to the right |
| `selfie_smile` | selfie (strict) | Smiling |
| `selfie_look_up` | selfie (strict) | Looking upward |
| `selfie_look_down` | selfie (strict) | Looking downward |
| `recto_center_document` | recto (all modes) | Document flat, centered, all corners visible |
| `recto_tilt_left` | recto (standard, strict) | Document tilted to the left |
| `recto_tilt_right` | recto (standard, strict) | Document tilted to the right |
| `recto_tilt_forward` | recto (strict) | Document tilted forward |
| `recto_tilt_back` | recto (strict) | Document tilted backward |
| `verso_center_document` | verso (all modes) | Document back, flat, centered |
| `verso_tilt_left` | verso (standard, strict) | Document back tilted left |
| `verso_tilt_right` | verso (standard, strict) | Document back tilted right |

## Python Example

```python
import httpx

files = {
    "selfie_center_face": ("center.jpg", open("selfie_centered.jpg", "rb"), "image/jpeg"),
    "selfie_close_eyes": ("eyes.jpg", open("selfie_eyes_closed.jpg", "rb"), "image/jpeg"),
    "recto_center_document": ("recto.jpg", open("cin_recto.jpg", "rb"), "image/jpeg"),
    "verso_center_document": ("verso.jpg", open("cin_verso.jpg", "rb"), "image/jpeg"),
}

resp = httpx.post(
    "https://kyvshield-naruto.innolinkcloud.com/api/v1/kyc/verify",
    headers={"X-API-Key": "YOUR_API_KEY"},
    data={
        "steps": '["selfie","recto","verso"]',
        "target": "SN-CIN",
        "language": "fr",
        "selfie_challenge_mode": "minimal",
        "recto_challenge_mode": "minimal",
        "verso_challenge_mode": "minimal",
        "require_face_match": "true",
        "require_aml": "true",
        "kyc_identifier": "user-12345",
    },
    files=files,
    timeout=120,
)
result = resp.json()
print(f"Status: {result['overall_status']}")
print(f"Session: {result['session_id']}")

# Close files
for _, f in files.values():
    if hasattr(f, 'close'):
        f.close()
```

## Node.js Example

```javascript
const FormData = require('form-data');
const fs = require('fs');
const axios = require('axios');

const form = new FormData();
form.append('steps', '["selfie","recto","verso"]');
form.append('target', 'SN-CIN');
form.append('language', 'fr');
form.append('selfie_challenge_mode', 'minimal');
form.append('recto_challenge_mode', 'minimal');
form.append('verso_challenge_mode', 'minimal');
form.append('require_face_match', 'true');
form.append('require_aml', 'true');
form.append('kyc_identifier', 'user-12345');

// One image per challenge
form.append('selfie_center_face', fs.createReadStream('selfie_centered.jpg'));
form.append('selfie_close_eyes', fs.createReadStream('selfie_eyes_closed.jpg'));
form.append('recto_center_document', fs.createReadStream('cin_recto.jpg'));
form.append('verso_center_document', fs.createReadStream('cin_verso.jpg'));

const resp = await axios.post(
  'https://kyvshield-naruto.innolinkcloud.com/api/v1/kyc/verify',
  form,
  {
    headers: { 'X-API-Key': 'YOUR_API_KEY', ...form.getHeaders() },
    timeout: 120000,
  }
);
console.log('Status:', resp.data.overall_status);
```

## Server-Side Validation

Before LLM analysis, the server validates each image:

**Selfie challenges** — validated by Python MediaPipe:
- `center_face`: face detected, centered, head pose straight
- `close_eyes`: both eyes closed (Eye Aspect Ratio below threshold)
- `turn_left/right`: head yaw angle in expected range
- `smile`: mouth ratio above threshold
- `look_up/down`: head pitch in expected range

**Document challenges** — validated by Python corner detection:
- `center_document`: document detected, flat (perspective ratios ~1.0)
- `tilt_left/right`: horizontal perspective ratio in expected range
- `tilt_forward/back`: vertical perspective ratio in expected range

If validation fails, the API returns an error with details about which challenge failed and why.

## Notes

- **Processing time**: 10-60 seconds depending on steps and LLM provider
- **Image format**: JPEG recommended
- **Max body size**: 30MB total
- **Timeout**: 120 seconds recommended
- **Webhooks**: Same events as SDK (`selfie.completed`, `recto.completed`, etc.)
- **Identity**: Automatically created when recto+verso PASS
- **Billing**: Charged upfront at request time (same as SDK)
- **Document alignment**: Server aligns document automatically before LLM analysis
- **Heatmap**: Fraud detection heatmap generated automatically for document steps
- **Batch**: `verifyBatch()` supports up to 10 concurrent verifications

## Server SDKs

Typed, self-contained SDKs for server-to-server integration:

| Language | Install |
|----------|---------|
| Node.js/TypeScript | `npm install @kyvshield/rest-sdk` |
| PHP | `composer require kyvshield/rest-sdk` |
| Java | JitPack: `com.github.moussa-innolink.kyv_shield_restfull:java:1.2.1` |
| Kotlin | JitPack: `com.github.moussa-innolink.kyv_shield_restfull:kotlin:1.2.1` |
| Go | `go get github.com/moussa-innolink/kyv_shield_restfull/go` |

Source: [github.com/moussa-innolink/kyv_shield_restfull](https://github.com/moussa-innolink/kyv_shield_restfull)
