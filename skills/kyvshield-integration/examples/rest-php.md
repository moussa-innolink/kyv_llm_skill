# KyvShield REST SDK — PHP

Server-to-server KYC verification SDK. Wraps the REST API with typed classes, automatic multipart encoding, and webhook signature verification.

## Installation

```bash
composer require kyvshield/rest-sdk
```

Requires PHP 8.1+ with `curl` and `json` extensions.

## Quick Start

```php
use KyvShield\KyvShield;

$client = new KyvShield('YOUR_API_KEY', 'https://kyvshield-naruto.innolinkcloud.com');
```

The second argument (base URL) is optional and defaults to the production endpoint.

## Get Available Challenges

```php
$challenges = $client->getChallenges();
// $challenges->document->minimal  → ["center_document"]
// $challenges->document->standard → ["center_document", "tilt_left", "tilt_right"]
// $challenges->selfie->minimal    → ["center_face", "close_eyes"]
```

## Verify — Full KYC

Image keys follow the `{step}_{challenge}` convention. Each key maps to an image in any supported format.

```php
use KyvShield\VerifyOptions;

$result = $client->verify(new VerifyOptions(
    steps: ['selfie', 'recto', 'verso'],
    target: 'SN-CIN',
    language: 'fr',
    challengeMode: 'minimal',
    stepChallengeModes: [
        'selfie' => 'minimal',
        'recto'  => 'minimal',
        'verso'  => 'minimal',
    ],
    requireFaceMatch: true,
    kycIdentifier: 'user-12345',
    images: [
        'selfie_center_face'    => '/path/to/selfie_centered.jpg',
        'selfie_close_eyes'     => '/path/to/selfie_eyes_closed.jpg',
        'recto_center_document' => '/path/to/cin_recto.jpg',
        'verso_center_document' => '/path/to/cin_verso.jpg',
    ],
));

echo $result->overallStatus;     // 'pass' | 'reject'
echo $result->sessionId;
echo $result->overallConfidence;
```

### Recto Only (minimal)

```php
$result = $client->verify(new VerifyOptions(
    steps: ['recto'],
    target: 'SN-CIN',
    language: 'fr',
    stepChallengeModes: ['recto' => 'minimal'],
    images: [
        'recto_center_document' => '/path/to/cin_recto.jpg',
    ],
));
```

### Recto + Verso with Standard Document Challenges

```php
$result = $client->verify(new VerifyOptions(
    steps: ['recto', 'verso'],
    target: 'SN-CIN',
    language: 'fr',
    stepChallengeModes: [
        'recto' => 'standard',
        'verso' => 'minimal',
    ],
    images: [
        'recto_center_document' => '/path/to/cin_recto_flat.jpg',
        'recto_tilt_left'       => '/path/to/cin_recto_tilted_left.jpg',
        'recto_tilt_right'      => '/path/to/cin_recto_tilted_right.jpg',
        'verso_center_document' => '/path/to/cin_verso.jpg',
    ],
));
```

## Image Input Formats

The `images` array values accept multiple formats. The SDK detects the type automatically.

| Format | Example | Description |
|--------|---------|-------------|
| File path | `'/path/to/recto.jpg'` | Local file, read from disk |
| URL | `'https://example.com/recto.jpg'` | Downloaded by SDK before upload |
| Data URI | `'data:image/jpeg;base64,/9j/4AA...'` | Standard data URI |
| Base64 string | `'/9j/4AAQSkZJRgABA...'` | Raw base64 (no prefix) |

## Batch Verification

Submit up to 10 verifications concurrently. Returns an array of results in the same order.

```php
$results = $client->verifyBatch([
    new VerifyOptions(
        steps: ['recto'],
        target: 'SN-CIN',
        language: 'fr',
        stepChallengeModes: ['recto' => 'minimal'],
        images: ['recto_center_document' => '/path/to/user1_recto.jpg'],
    ),
    new VerifyOptions(
        steps: ['recto', 'verso'],
        target: 'SN-CIN',
        language: 'fr',
        stepChallengeModes: ['recto' => 'minimal', 'verso' => 'minimal'],
        images: [
            'recto_center_document' => '/path/to/user2_recto.jpg',
            'verso_center_document' => '/path/to/user2_verso.jpg',
        ],
    ),
]);

foreach ($results as $i => $r) {
    echo "Verification {$i}: {$r->overallStatus}\n";
}
```

## Webhook Signature Verification

Static method — does not require a client instance.

```php
// In your webhook controller (e.g., Laravel)
$rawBody = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_KYVSHIELD_SIGNATURE'] ?? '';

$valid = KyvShield::verifyWebhookSignature($rawBody, 'YOUR_API_KEY', $signature);
if (!$valid) {
    http_response_code(401);
    exit('Invalid signature');
}

$payload = json_decode($rawBody, true);
$event = $_SERVER['HTTP_X_KYVSHIELD_EVENT'] ?? '';
$sessionId = $_SERVER['HTTP_X_KYVSHIELD_SESSION'] ?? '';
// Process event...
http_response_code(200);
```

### Laravel Example

```php
use Illuminate\Http\Request;
use KyvShield\KyvShield;

Route::post('/webhook', function (Request $request) {
    $valid = KyvShield::verifyWebhookSignature(
        $request->getContent(),
        config('services.kyvshield.api_key'),
        $request->header('X-Kyvshield-Signature', '')
    );

    if (!$valid) {
        return response('Invalid signature', 401);
    }

    $event = $request->header('X-Kyvshield-Event');
    $payload = $request->json()->all();
    // Process event...
    return response('OK', 200);
});
```

## Response Structure

```php
class KycResponse {
    public bool $success;
    public string $sessionId;
    public string $overallStatus;       // 'pass' | 'reject'
    public float $overallConfidence;
    public ?FaceVerification $faceVerification;
    /** @var StepResult[] */
    public array $steps;
    public int $processingTimeMs;
}

class StepResult {
    public int $stepIndex;
    public string $stepType;            // 'selfie' | 'recto' | 'verso'
    public bool $success;
    public Liveness $liveness;
    public ?Verification $verification;
    /** @var ExtractionField[] */
    public ?array $extraction;
    /** @var ExtractedPhoto[] */
    public ?array $extractedPhotos;
    public ?string $alignedDocument;    // base64 JPEG (recto/verso)
    public ?string $capturedImage;      // base64 JPEG (selfie)
    public int $processingTimeMs;
}
```

## Error Handling

```php
use KyvShield\Exception\KyvShieldException;
use KyvShield\Exception\ValidationException;
use KyvShield\Exception\AuthenticationException;

try {
    $result = $client->verify($options);
} catch (AuthenticationException $e) {
    // 401 — Invalid API key
    echo "Auth error: " . $e->getMessage();
} catch (ValidationException $e) {
    // 422 — Missing or invalid fields
    echo "Validation: " . $e->getMessage();
    print_r($e->getDetails());
} catch (KyvShieldException $e) {
    // Other API errors (400, 500, timeout)
    echo "Error [{$e->getCode()}]: " . $e->getMessage();
}
```

## Notes

- **Timeout**: 120 seconds recommended (SDK default)
- **Max body size**: 30 MB total across all images
- **Image format**: JPEG recommended
- **Processing time**: 10-60 seconds depending on steps and challenge modes
- **Concurrency**: `verifyBatch` uses `curl_multi` for parallel execution

## API Documentation

Full documentation: **[https://kyvshield-naruto.innolinkcloud.com/developer](https://kyvshield-naruto.innolinkcloud.com/developer)**

Source: [github.com/moussa-innolink/kyv_shield_restfull](https://github.com/moussa-innolink/kyv_shield_restfull)
