## Repositories
- **Example App**: https://github.com/moussa-innolink/kyb_example_web

# KyvShield Web SDK

**KyvShield** — Web SDK for identity verification (KYC). Selfie liveness, document OCR, face matching.

## Installation

Add the SDK via CDN script tag. Requires HTTPS or localhost for camera access.

```html
<script src="https://kyvshield-naruto.innolinkcloud.com/static/sdk/kyvshield.min.js"></script>
```

## Full Example

Complete HTML page with all options and result handling.

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>KyvShield KYC</title>
  <script src="https://kyvshield-naruto.innolinkcloud.com/static/sdk/kyvshield.min.js"></script>
</head>
<body>
  <button id="startKyc">Start KYC</button>

  <script>
    // ── Possible values ─────────────────────────────────────────────────
    // steps           : 'selfie', 'recto', 'verso'
    // challengeMode   : 'minimal', 'standard', 'strict'
    // selfieDisplayMode  : 'standard', 'compact', 'immersive', 'neonHud'
    // documentDisplayMode: 'standard', 'compact', 'immersive', 'neonHud'
    // themeMode       : 'light', 'dark'
    // language        : 'fr', 'en', 'wo'

    // ── Fetch available documents from API ──────────────────────────────

    async function fetchDocuments() {
      const res = await fetch(
        'https://kyvshield-naruto.innolinkcloud.com/api/v1/documents',
        { headers: { 'X-API-Key': 'YOUR_API_KEY' } }
      );
      const { documents } = await res.json();
      return documents;
    }

    // ── Start KYC ───────────────────────────────────────────────────────

    document.getElementById('startKyc').addEventListener('click', async () => {
      const documents = await fetchDocuments();
      const selectedDoc = documents[0]; // or let user pick

      const result = await KyvShield.initKyc(
        // Config
        {
          baseUrl: 'https://kyvshield-naruto.innolinkcloud.com',
          apiKey: 'YOUR_API_KEY',
          enableLog: true,
          theme: {
            primaryColor: '#EF8352',
            themeMode: 'light',
          },
        },
        // Flow
        {
          steps: ['selfie', 'recto', 'verso'],
          language: 'fr',
          showIntroPage: true,
          showInstructionPages: true,
          showResultPage: true,
          showSuccessPerStep: true,
          selfieDisplayMode: 'standard',
          documentDisplayMode: 'standard',
          challengeMode: 'minimal',
          requireFaceMatch: true,
          playChallengeAudio: true,
          maxChallengeAudioPlay: 'once',
          pauseBetweenAudioPlayMs: 1000,
          target: selectedDoc,
          kycIdentifier: 'user-12345',
        }
      );

      // ── Handle result ─────────────────────────────────────────────────

      console.log('Success:', result.success);
      console.log('Status:', result.overallStatus);   // 'pass', 'reject', 'error'
      console.log('Session:', result.sessionId);

      // Selfie
      if (result.selfieResult) {
        console.log('Live:', result.selfieResult.isLive);
        console.log('Confidence:', result.selfieResult.confidence);
        console.log('Selfie image (base64):', result.selfieResult.capturedImageBase64?.substring(0, 50) + '...');
      }

      // Recto (front of document)
      if (result.rectoResult) {
        console.log('Recto score:', result.rectoResult.score);
        console.log('Recto authentic:', result.rectoResult.isLive);
        console.log('Aligned document (base64):', result.rectoResult.alignedDocumentBase64?.substring(0, 50) + '...');

        // Extracted photos (face from document)
        if (result.rectoResult.extractedPhotos?.length) {
          result.rectoResult.extractedPhotos.forEach((photo, i) => {
            console.log(`Photo ${i}: ${photo.width}x${photo.height}, confidence=${photo.confidence}`);
          });
        }

        // OCR extracted fields
        if (result.rectoResult.extraction?.fields) {
          result.rectoResult.extraction.fields.forEach(field => {
            console.log(`${field.label}: ${field.value}`);
          });
        }

        // Face match (selfie vs document photo)
        if (result.rectoResult.faceVerification) {
          console.log('Face match:', result.rectoResult.faceVerification.isMatch);
          console.log('Similarity:', result.rectoResult.faceVerification.similarityScore);
          console.log('Confidence:', result.rectoResult.faceVerification.confidenceLevel);
        }
      }

      // Verso (back of document)
      if (result.versoResult) {
        console.log('Verso score:', result.versoResult.score);
        console.log('Aligned verso (base64):', result.versoResult.alignedDocumentBase64?.substring(0, 50) + '...');

        if (result.versoResult.extraction?.fields) {
          result.versoResult.extraction.fields.forEach(field => {
            console.log(`${field.label}: ${field.value}`);
          });
        }
      }

      // Convenience: extract specific field by key (searches recto + verso)
      console.log('Nom:', result.getExtractedValue?.('nom'));
      console.log('NIN:', result.getExtractedValue?.('nin'));
      console.log('Prenoms:', result.getExtractedValue?.('prenoms'));
      console.log('Date naissance:', result.getExtractedValue?.('date_naissance'));
    });
  </script>
</body>
</html>
```

## Configuration Reference

### Config object (first argument to `initKyc`)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `baseUrl` | string | required | API server URL |
| `apiKey` | string | required | Your API key |
| `enableLog` | boolean | `false` | Enable debug console logs |
| `theme.primaryColor` | string | `'#EF8352'` | Hex color for buttons and accents |
| `theme.themeMode` | string | `'light'` | `'light'` or `'dark'` |

### Flow object (second argument to `initKyc`)

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `steps` | string[] | `['selfie','recto','verso']` | Capture steps to perform |
| `language` | string | `'fr'` | UI language: `'fr'`, `'en'`, `'wo'` |
| `challengeMode` | string | `'standard'` | `'minimal'`, `'standard'`, `'strict'` |
| `selfieDisplayMode` | string | `'standard'` | `'standard'`, `'compact'`, `'immersive'`, `'neonHud'` |
| `documentDisplayMode` | string | `'standard'` | `'standard'`, `'compact'`, `'immersive'`, `'neonHud'` |
| `showIntroPage` | boolean | `true` | Show intro screen before KYC |
| `showInstructionPages` | boolean | `true` | Show step instructions |
| `showResultPage` | boolean | `true` | Show result summary at end |
| `showSuccessPerStep` | boolean | `true` | Show success after each step |
| `requireFaceMatch` | boolean | `true` | Compare selfie to document photo |
| `playChallengeAudio` | boolean | `false` | Play audio for liveness challenges |
| `maxChallengeAudioPlay` | string | `'once'` | `'once'`, `'twice'`, `'thrice'` |
| `pauseBetweenAudioPlayMs` | number | `1000` | Pause between audio replays (ms) |
| `target` | object | `null` | Document from `/api/v1/documents` |
| `kycIdentifier` | string | `null` | Your reference ID (returned in webhooks) |

## Display Modes

| Mode | Description |
|------|-------------|
| `standard` | Classic layout with header, camera, and instructions below |
| `compact` | Camera fills screen, instructions overlay at bottom |
| `immersive` | Full-screen camera with glass-effect overlays |
| `neonHud` | Futuristic dark theme with glow effects and monospace font |

## API Documentation

Full documentation: **[https://kyvshield-naruto.innolinkcloud.com/developer](https://kyvshield-naruto.innolinkcloud.com/developer)**
