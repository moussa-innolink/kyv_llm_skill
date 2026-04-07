# KyvShield Webhooks

KyvShield sends HTTP POST webhooks to your configured endpoint after each verification step and at session completion.

## Events

| Event | Description |
|-------|-------------|
| `selfie.completed` | Selfie liveness check finished successfully |
| `selfie.failed` | Selfie liveness check failed |
| `recto.completed` | Front of document scanned and verified successfully |
| `recto.failed` | Front of document verification failed |
| `verso.completed` | Back of document scanned and verified successfully |
| `verso.failed` | Back of document verification failed |
| `session.completed` | Full KYC session finished (all steps done) |
| `session.failed` | KYC session failed or was abandoned |

## HTTP Headers

Every webhook request includes these headers:

| Header | Description | Example |
|--------|-------------|---------|
| `User-Agent` | Always `KyvShield-Webhook/1.0` | `KyvShield-Webhook/1.0` |
| `Content-Type` | Always `application/json` | `application/json` |
| `X-Kyvshield-Event` | The event type | `selfie.completed` |
| `X-Kyvshield-Session` | The KYC session ID | `7a4304a3e1c66c2405bcb72e661113bc` |
| `X-Kyvshield-Timestamp` | ISO 8601 timestamp | `2026-04-06T22:37:55Z` |
| `X-Kyvshield-Signature` | HMAC-SHA256 signature of the body | `sha256=88ffea867a01ba15...` |

## Signature Verification

Every webhook is signed with HMAC-SHA256 using your API key as the secret. The signature is in the `X-Kyvshield-Signature` header, prefixed with `sha256=`.

To verify: compute HMAC-SHA256 of the raw request body using your API key, then compare with the header value (after stripping the `sha256=` prefix).

### Python

```python
import hmac
import hashlib

def verify_webhook(body: bytes, signature_header: str, api_key: str) -> bool:
    expected = hmac.new(
        api_key.encode('utf-8'),
        body,
        hashlib.sha256
    ).hexdigest()
    received = signature_header.removeprefix('sha256=')
    return hmac.compare_digest(expected, received)

# Usage in Flask
@app.route('/webhook', methods=['POST'])
def webhook():
    body = request.get_data()
    signature = request.headers.get('X-Kyvshield-Signature', '')
    if not verify_webhook(body, signature, 'YOUR_API_KEY'):
        return 'Invalid signature', 401

    payload = request.get_json()
    event = request.headers.get('X-Kyvshield-Event')
    session_id = request.headers.get('X-Kyvshield-Session')
    # Process event...
    return 'OK', 200
```

### Node.js

```javascript
const crypto = require('crypto');

function verifyWebhook(body, signatureHeader, apiKey) {
  const expected = crypto
    .createHmac('sha256', apiKey)
    .update(body)
    .digest('hex');
  const received = signatureHeader.replace('sha256=', '');
  return crypto.timingSafeEqual(
    Buffer.from(expected, 'hex'),
    Buffer.from(received, 'hex')
  );
}

// Usage in Express
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-kyvshield-signature'] || '';
  if (!verifyWebhook(req.body, signature, 'YOUR_API_KEY')) {
    return res.status(401).send('Invalid signature');
  }

  const payload = JSON.parse(req.body);
  const event = req.headers['x-kyvshield-event'];
  const sessionId = req.headers['x-kyvshield-session'];
  // Process event...
  res.status(200).send('OK');
});
```

### Go

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"io"
	"net/http"
	"strings"
)

func verifyWebhook(body []byte, signatureHeader, apiKey string) bool {
	mac := hmac.New(sha256.New, []byte(apiKey))
	mac.Write(body)
	expected := hex.EncodeToString(mac.Sum(nil))
	received := strings.TrimPrefix(signatureHeader, "sha256=")
	return hmac.Equal([]byte(expected), []byte(received))
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
	body, _ := io.ReadAll(r.Body)
	signature := r.Header.Get("X-Kyvshield-Signature")
	if !verifyWebhook(body, signature, "YOUR_API_KEY") {
		http.Error(w, "Invalid signature", http.StatusUnauthorized)
		return
	}

	event := r.Header.Get("X-Kyvshield-Event")
	sessionID := r.Header.Get("X-Kyvshield-Session")
	// Process event...
	_ = event
	_ = sessionID
	w.WriteHeader(http.StatusOK)
}
```

## Webhook Body Examples

### `selfie.completed`

```json
{
  "event": "selfie.completed",
  "timestamp": "2026-04-06T22:37:55Z",
  "session_id": "7a4304a3e1c66c2405bcb72e661113bc",
  "app_id": "demo_app",
  "key_id": "demo_key_1",
  "document_type": "SN-CIN",
  "step_data": {
    "captured_image": "/9j/4AAQSkZJRgABA....k+owORXRhJQspPY+clG...",
    "liveness": {
      "confidence": "HIGH",
      "is_live": true,
      "score": 1
    },
    "processing_time_ms": 2146,
    "step_index": 0,
    "step_type": "selfie",
    "success": true,
    "user_messages": [
      "Selfie captur\u00e9 avec succ\u00e8s."
    ],
    "verification": {
      "checks_passed": [
        "center_face",
        "close_eyes"
      ],
      "confidence": 1,
      "fraud_indicators": [],
      "is_authentic": true,
      "issues": [],
      "warnings": []
    }
  }
}
```

### `recto.completed`

```json
{
  "event": "recto.completed",
  "timestamp": "2026-04-06T22:38:11Z",
  "session_id": "7a4304a3e1c66c2405bcb72e661113bc",
  "app_id": "demo_app",
  "key_id": "demo_key_1",
  "document_type": "SN-CIN",
  "step_data": {
    "aligned_document": "/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAIBA....QwEaP...",
    "extracted_photos": [
      {
        "image": "/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAMCAgMCA...",
        "confidence": 0.8443496227264404,
        "bbox": [94, 218, 210, 371],
        "area": 17748,
        "width": 116,
        "height": 153
      }
    ],
    "extraction": [
      {
        "key": "numero_carte",
        "document_key": "numero_carte",
        "label": "N\u00b0 de la carte d'identit\u00e9",
        "value": "1 06 19930515 00026 8",
        "display_priority": 1
      },
      {
        "key": "prenoms",
        "document_key": "prenoms",
        "label": "Pr\u00e9noms",
        "value": "MOUSSA",
        "display_priority": 2
      },
      {
        "key": "nom",
        "document_key": "nom",
        "label": "Nom",
        "value": "NDOUR",
        "display_priority": 3
      },
      {
        "key": "date_naissance",
        "document_key": "date_naissance",
        "label": "Date de naissance",
        "value": "1993-05-15",
        "display_priority": 4
      },
      {
        "key": "sexe",
        "document_key": "sexe",
        "label": "Sexe",
        "value": "M",
        "display_priority": 5
      },
      {
        "key": "taille",
        "document_key": "taille",
        "label": "Taille",
        "value": "175 cm",
        "display_priority": 6
      },
      {
        "key": "lieu_naissance",
        "document_key": "lieu_naissance",
        "label": "Lieu de naissance",
        "value": "KAOLACK",
        "display_priority": 7
      },
      {
        "key": "date_delivrance",
        "document_key": "date_delivrance",
        "label": "Date de d\u00e9livrance",
        "value": "2019-07-29",
        "display_priority": 8
      },
      {
        "key": "date_expiration",
        "document_key": "date_expiration",
        "label": "Date d'expiration",
        "value": "2029-07-28",
        "display_priority": 9
      },
      {
        "key": "centre_enregistrement",
        "document_key": "centre_enregistrement",
        "label": "Centre d'enregistrement",
        "value": "COMM. DE DIEUPPEUL",
        "display_priority": 10
      },
      {
        "key": "adresse_domicile",
        "document_key": "adresse_domicile",
        "label": "Adresse du domicile",
        "value": "ABATTOIRS",
        "display_priority": 11
      }
    ],
    "liveness": {
      "confidence": "HIGH",
      "is_live": true,
      "score": 0.95
    },
    "processing_time_ms": 7195,
    "step_index": 1,
    "step_type": "recto",
    "success": true,
    "user_messages": [
      "Document analys\u00e9 avec succ\u00e8s."
    ],
    "verification": {
      "checks_passed": [
        "center_document"
      ],
      "confidence": 0.95,
      "fraud_indicators": [],
      "is_authentic": true,
      "issues": [],
      "warnings": []
    }
  }
}
```

### `verso.completed`

```json
{
  "event": "verso.completed",
  "timestamp": "2026-04-06T22:38:24Z",
  "session_id": "7a4304a3e1c66c2405bcb72e661113bc",
  "app_id": "demo_app",
  "key_id": "demo_key_1",
  "document_type": "SN-CIN",
  "step_data": {
    "aligned_document": "/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAIBAQEBAQIBAQ...",
    "extraction": [
      {
        "key": "code_pays",
        "document_key": "code_pays",
        "label": "Code Pays",
        "value": "SEN",
        "display_priority": 1
      },
      {
        "key": "inscrit_liste_electorale",
        "document_key": "inscrit_liste_electorale",
        "label": "Inscrit sur la liste \u00e9lectorale",
        "value": "false",
        "display_priority": 2
      },
      {
        "key": "nin",
        "document_key": "nin",
        "label": "NIN",
        "value": "1548199301837",
        "display_priority": 3
      },
      {
        "key": "mrz",
        "document_key": "mrz",
        "label": "MRZ",
        "value": "I<SEN1061993051500026B2<<<<<<<\n9305155M2907284SEN<<<<<<<<<<<<<4\nNDOUR<<MOUSSA<<<<<<<<<<<<<<<<<<<",
        "display_priority": 4
      }
    ],
    "liveness": {
      "confidence": "HIGH",
      "is_live": true,
      "score": 0.95
    },
    "processing_time_ms": 8360,
    "step_index": 2,
    "step_type": "verso",
    "success": true,
    "user_messages": [
      "Document verso analys\u00e9 avec succ\u00e8s."
    ],
    "verification": {
      "checks_passed": [
        "center_document"
      ],
      "confidence": 0.95,
      "fraud_indicators": [],
      "is_authentic": true,
      "issues": [],
      "warnings": []
    }
  }
}
```

### `session.completed`

```json
{
  "event": "session.completed",
  "timestamp": "2026-04-06T22:38:24Z",
  "session_id": "7a4304a3e1c66c2405bcb72e661113bc",
  "app_id": "demo_app",
  "key_id": "demo_key_1",
  "document_type": "SN-CIN",
  "final_result": {
    "success": true,
    "status": "PASS",
    "document_type": "SN-CIN",
    "steps": {
      "face_match_passed": true,
      "face_match_score": 87.5,
      "overall_confidence": 0.9666666666666667,
      "processing_time_ms": 8360,
      "rejection_reason": "",
      "steps_completed": 3,
      "steps_failed": 0,
      "steps_passed": 3
    }
  }
}
```

### `session.failed`

```json
{
  "event": "session.failed",
  "timestamp": "2026-04-06T22:40:00Z",
  "session_id": "abc123def456",
  "app_id": "demo_app",
  "key_id": "demo_key_1",
  "document_type": "SN-CIN",
  "final_result": {
    "success": false,
    "status": "TIMEOUT",
    "document_type": "SN-CIN",
    "steps": {
      "reason": "session_expired"
    }
  }
}
```

## Body Structure Reference

### Step events (`selfie.*`, `recto.*`, `verso.*`)

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Event name |
| `timestamp` | string | ISO 8601 timestamp |
| `session_id` | string | KYC session identifier |
| `app_id` | string | Your application ID |
| `key_id` | string | API key identifier used |
| `document_type` | string | Target document type (`SN-CIN`, `SN-PASSPORT`, `SN-DRIVER-LICENCE`) |
| `step_data.step_type` | string | `selfie`, `recto`, or `verso` |
| `step_data.step_index` | int | 0-based step index |
| `step_data.success` | bool | Whether step passed |
| `step_data.processing_time_ms` | int | Server processing time |
| `step_data.captured_image` | string | Base64 JPEG (selfie only) |
| `step_data.aligned_document` | string | Base64 JPEG (recto/verso only) |
| `step_data.extracted_photos` | array | Detected face photos (recto only) |
| `step_data.extraction` | array | OCR extracted fields (recto/verso) |
| `step_data.liveness` | object | Liveness detection result |
| `step_data.liveness.is_live` | bool | Real person/document detected |
| `step_data.liveness.score` | float | Confidence score 0-1 |
| `step_data.liveness.confidence` | string | `HIGH`, `MEDIUM`, `LOW` |
| `step_data.verification` | object | Fraud/authenticity check |
| `step_data.verification.is_authentic` | bool | Document is genuine |
| `step_data.verification.confidence` | float | Authenticity score 0-1 |
| `step_data.verification.checks_passed` | array | Passed verification checks |
| `step_data.verification.fraud_indicators` | array | Detected fraud signals |
| `step_data.verification.issues` | array | Blocking issues |
| `step_data.verification.warnings` | array | Non-blocking warnings (currently always `[]`) |
| `step_data.user_messages` | array | Localized messages for end user |

### Session events (`session.*`)

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | `session.completed` or `session.failed` |
| `timestamp` | string | ISO 8601 timestamp |
| `session_id` | string | KYC session identifier |
| `app_id` | string | Your application ID |
| `key_id` | string | API key identifier used |
| `document_type` | string | Target document type (`SN-CIN`, `SN-PASSPORT`, `SN-DRIVER-LICENCE`) |
| `final_result.success` | bool | Overall KYC success |
| `final_result.status` | string | `PASS`, `REJECT`, or `TIMEOUT` |
| `final_result.document_type` | string | Target document type (same as top-level) |
| `final_result.steps.steps_completed` | int | Total steps completed |
| `final_result.steps.steps_passed` | int | Steps that passed |
| `final_result.steps.steps_failed` | int | Steps that failed |
| `final_result.steps.face_match_passed` | bool | Selfie matched document |
| `final_result.steps.face_match_score` | float | Face similarity score |
| `final_result.steps.overall_confidence` | float | Average confidence |
| `final_result.steps.processing_time_ms` | int | Total processing time |
| `final_result.steps.rejection_reason` | string | Reason if rejected (empty if passed) |

## Retry Policy

If your endpoint does not return a `2xx` status code, KyvShield retries delivery with exponential backoff:

| Attempt | Delay |
|---------|-------|
| 1st retry | ~1 second |
| 2nd retry | ~5 seconds |
| 3rd retry | ~30 seconds |

After 3 failed retries, the webhook failure is stored for 24 hours. You can query the session status via the REST API if you missed a webhook.

## Best Practices

- **Always verify the signature** before processing the payload to prevent spoofed requests.
- **Respond quickly** with a `200 OK` and process asynchronously. Webhook requests timeout after 10 seconds.
- **Use the session ID** (`X-Kyvshield-Session` header or `session_id` field) to correlate step events with the final session result.
- **Use `document_type`** to know which document was verified (e.g., `SN-CIN`, `SN-PASSPORT`). Present in all webhook events.
- **Handle idempotency** -- in rare cases a webhook may be delivered more than once. Use the session ID + event type as a deduplication key.
- **Store raw payloads** for debugging and audit trails.
- **`warnings` field** is currently always an empty array `[]` in webhook payloads.
