# KyvShield REST SDK — Node.js / TypeScript

Server-to-server KYC verification SDK. Wraps the REST API with typed methods, automatic multipart encoding, and webhook signature verification.

## Installation

```bash
npm install @kyvshield/rest-sdk
```

## Quick Start

```typescript
import { KyvShield } from '@kyvshield/rest-sdk';

const client = new KyvShield('YOUR_API_KEY', 'https://kyvshield-naruto.innolinkcloud.com');
```

The second argument (base URL) is optional and defaults to the production endpoint.

## Get Available Challenges

```typescript
const challenges = await client.getChallenges();
// challenges.document.minimal  → ["center_document"]
// challenges.document.standard → ["center_document", "tilt_left", "tilt_right"]
// challenges.selfie.minimal    → ["center_face", "close_eyes"]
// challenges.selfie.standard   → ["center_face", "close_eyes", "turn_left", "turn_right"]
```

## Verify — Full KYC

Image keys follow the `{step}_{challenge}` convention. Each key maps to an image in any supported format.

```typescript
const result = await client.verify({
  steps: ['selfie', 'recto', 'verso'],
  target: 'SN-CIN',
  language: 'fr',
  challengeMode: 'minimal',
  rectoChallengeMode: 'minimal',
  versoChallengeMode: 'minimal',
  selfieChhallengeMode: 'minimal',
  requireFaceMatch: true,
  requireAml: true,
  kycIdentifier: 'user-12345',
  images: {
    selfie_center_face: './selfie_centered.jpg',
    selfie_close_eyes: './selfie_eyes_closed.jpg',
    recto_center_document: './cin_recto.jpg',
    verso_center_document: './cin_verso.jpg',
  },
});

console.log(result.overallStatus);    // 'pass' | 'reject'
console.log(result.sessionId);
console.log(result.overallConfidence);
```

### Recto Only (minimal)

```typescript
const result = await client.verify({
  steps: ['recto'],
  target: 'SN-CIN',
  language: 'fr',
  rectoChallengeMode: 'minimal',
  images: {
    recto_center_document: './cin_recto.jpg',
  },
});
```

### Recto + Verso with Standard Document Challenges

```typescript
const result = await client.verify({
  steps: ['recto', 'verso'],
  target: 'SN-CIN',
  language: 'fr',
  rectoChallengeMode: 'standard',
  versoChallengeMode: 'minimal',
  images: {
    recto_center_document: './cin_recto_flat.jpg',
    recto_tilt_left: './cin_recto_tilted_left.jpg',
    recto_tilt_right: './cin_recto_tilted_right.jpg',
    verso_center_document: './cin_verso.jpg',
  },
});
```

## Image Input Formats

The `images` map values accept multiple formats. The SDK detects the type automatically.

| Format | Example | Description |
|--------|---------|-------------|
| File path | `'./recto.jpg'` | Local file, read from disk |
| `Buffer` | `Buffer.from(rawBytes)` | In-memory bytes |
| URL | `'https://example.com/recto.jpg'` | Downloaded by SDK before upload |
| Data URI | `'data:image/jpeg;base64,/9j/4AA...'` | Standard data URI |
| Base64 string | `'/9j/4AAQSkZJRgABA...'` | Raw base64 (no prefix) |

## Batch Verification

Submit up to 10 verifications concurrently. Returns an array of results in the same order.

```typescript
const results = await client.verifyBatch([
  {
    steps: ['recto'],
    target: 'SN-CIN',
    language: 'fr',
    rectoChallengeMode: 'minimal',
    images: { recto_center_document: './user1_recto.jpg' },
  },
  {
    steps: ['recto', 'verso'],
    target: 'SN-CIN',
    language: 'fr',
    rectoChallengeMode: 'minimal',
    versoChallengeMode: 'minimal',
    images: {
      recto_center_document: './user2_recto.jpg',
      verso_center_document: './user2_verso.jpg',
    },
  },
]);

results.forEach((r, i) => {
  console.log(`Verification ${i}: ${r.overallStatus}`);
});
```

## Webhook Signature Verification

Static method — does not require a client instance.

```typescript
import express from 'express';
import { KyvShield } from '@kyvshield/rest-sdk';

const app = express();
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-kyvshield-signature'] as string;
  const valid = KyvShield.verifyWebhookSignature(req.body, 'YOUR_API_KEY', signature);

  if (!valid) return res.status(401).send('Invalid signature');

  const payload = JSON.parse(req.body.toString());
  const event = req.headers['x-kyvshield-event'];
  // Process event...
  res.status(200).send('OK');
});
```

## Response Structure

```typescript
interface KycResponse {
  success: boolean;
  sessionId: string;
  overallStatus: 'pass' | 'reject';
  overallConfidence: number;
  faceVerification?: {
    isMatch: boolean;
    similarityScore: number;
  };
  steps: Array<{
    stepIndex: number;
    stepType: 'selfie' | 'recto' | 'verso';
    success: boolean;
    liveness: { isLive: boolean; score: number; confidence: string };
    verification?: { isAuthentic: boolean; confidence: number; checksPassed: string[] };
    extraction?: Array<{
      key: string;
      documentKey: string;
      label: string;
      value: string;
      displayPriority: number;
    }>;
    extractedPhotos?: Array<{
      image: string;         // base64 JPEG
      confidence: number;
      bbox: number[];
      width: number;
      height: number;
    }>;
    alignedDocument?: string;  // base64 JPEG (recto/verso)
    capturedImage?: string;    // base64 JPEG (selfie)
    processingTimeMs: number;
  }>;
  processingTimeMs: number;
}
```

## Error Handling

```typescript
try {
  const result = await client.verify({ /* ... */ });
} catch (err) {
  if (err.status === 401) {
    console.error('Invalid API key');
  } else if (err.status === 400) {
    console.error('Bad request:', err.message);
  } else if (err.status === 422) {
    console.error('Validation failed:', err.details);
  } else {
    console.error('Server error:', err.message);
  }
}
```

## Notes

- **Timeout**: 120 seconds recommended (SDK default)
- **Max body size**: 30 MB total across all images
- **Image format**: JPEG recommended
- **Processing time**: 10-60 seconds depending on steps and challenge modes
- **Concurrency**: `verifyBatch` runs up to 10 verifications in parallel

## API Documentation

Full documentation: **[https://kyvshield-naruto.innolinkcloud.com/developer](https://kyvshield-naruto.innolinkcloud.com/developer)**

Source: [github.com/moussa-innolink/kyv_shield_restfull](https://github.com/moussa-innolink/kyv_shield_restfull)
