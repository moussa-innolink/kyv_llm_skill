# KyvShield REST SDK — Go

Server-to-server KYC verification SDK. Wraps the REST API with functional options, context support, and webhook signature verification.

## Installation

```bash
go get github.com/moussa-innolink/kyv_shield_restfull/go
```

Requires Go 1.21+.

## Quick Start

```go
import kyvshield "github.com/moussa-innolink/kyv_shield_restfull/go"

client := kyvshield.NewClient("YOUR_API_KEY",
    kyvshield.WithBaseURL("https://kyvshield-naruto.innolinkcloud.com"),
)
```

`WithBaseURL` is optional and defaults to the production endpoint.

## Get Available Challenges

```go
challenges, err := client.GetChallenges(ctx)
if err != nil {
    log.Fatal(err)
}
// challenges.Document.Minimal  → ["center_document"]
// challenges.Document.Standard → ["center_document", "tilt_left", "tilt_right"]
// challenges.Selfie.Minimal    → ["center_face", "close_eyes"]
```

## Verify — Full KYC

Image keys follow the `{step}_{challenge}` convention. All methods accept a `context.Context` for cancellation and timeouts.

```go
resp, err := client.Verify(ctx, &kyvshield.VerifyOptions{
    Steps:                []string{"selfie", "recto", "verso"},
    Target:               "SN-CIN",
    Language:             "fr",
    ChallengeMode:        "minimal",
    SelfieChhallengeMode: "minimal",
    RectoChallengeMode:   "minimal",
    VersoChallengeMode:   "minimal",
    RequireFaceMatch:     true,
    KycIdentifier:        "user-12345",
    Images: map[string]string{
        "selfie_center_face":    "./selfie_centered.jpg",
        "selfie_close_eyes":     "./selfie_eyes_closed.jpg",
        "recto_center_document": "./cin_recto.jpg",
        "verso_center_document": "./cin_verso.jpg",
    },
})
if err != nil {
    log.Fatal(err)
}

fmt.Println(resp.OverallStatus)      // "pass" | "reject"
fmt.Println(resp.SessionID)
fmt.Println(resp.OverallConfidence)
```

### Recto Only (minimal)

```go
resp, err := client.Verify(ctx, &kyvshield.VerifyOptions{
    Steps:              []string{"recto"},
    Target:             "SN-CIN",
    Language:           "fr",
    RectoChallengeMode: "minimal",
    Images: map[string]string{
        "recto_center_document": "./cin_recto.jpg",
    },
})
```

### Using []byte Instead of File Paths

```go
rectoBytes, _ := os.ReadFile("cin_recto.jpg")

resp, err := client.Verify(ctx, &kyvshield.VerifyOptions{
    Steps:              []string{"recto"},
    Target:             "SN-CIN",
    Language:           "fr",
    RectoChallengeMode: "minimal",
    ImageBytes: map[string][]byte{
        "recto_center_document": rectoBytes,
    },
})
```

## Image Input Formats

The `Images` map values accept multiple formats. The SDK detects the type automatically.

| Format | Example | Description |
|--------|---------|-------------|
| File path | `"./recto.jpg"` | Local file, read from disk |
| URL | `"https://example.com/recto.jpg"` | Downloaded by SDK before upload |
| Data URI | `"data:image/jpeg;base64,/9j/..."` | Standard data URI |
| Base64 string | `"/9j/4AAQSkZJRgABA..."` | Raw base64 (no prefix) |

The `ImageBytes` map accepts `[]byte` values for in-memory image data. If the same key appears in both `Images` and `ImageBytes`, `ImageBytes` takes precedence.

## Batch Verification

Submit up to 10 verifications concurrently. Returns a slice of results in the same order.

```go
results, err := client.VerifyBatch(ctx, []*kyvshield.VerifyOptions{
    {
        Steps:              []string{"recto"},
        Target:             "SN-CIN",
        Language:           "fr",
        RectoChallengeMode: "minimal",
        Images: map[string]string{
            "recto_center_document": "./user1_recto.jpg",
        },
    },
    {
        Steps:              []string{"recto", "verso"},
        Target:             "SN-CIN",
        Language:           "fr",
        RectoChallengeMode: "minimal",
        VersoChallengeMode: "minimal",
        Images: map[string]string{
            "recto_center_document": "./user2_recto.jpg",
            "verso_center_document": "./user2_verso.jpg",
        },
    },
})
if err != nil {
    log.Fatal(err)
}

for i, r := range results {
    fmt.Printf("Verification %d: %s\n", i, r.OverallStatus)
}
```

## Webhook Signature Verification

Package-level function — does not require a client instance.

```go
import kyvshield "github.com/moussa-innolink/kyv_shield_restfull/go"

func webhookHandler(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }

    signature := r.Header.Get("X-Kyvshield-Signature")
    valid := kyvshield.VerifyWebhookSignature(body, "YOUR_API_KEY", signature)
    if !valid {
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

### Gin Example

```go
import kyvshield "github.com/moussa-innolink/kyv_shield_restfull/go"

r.POST("/webhook", func(c *gin.Context) {
    body, _ := c.GetRawData()
    signature := c.GetHeader("X-Kyvshield-Signature")

    if !kyvshield.VerifyWebhookSignature(body, "YOUR_API_KEY", signature) {
        c.String(http.StatusUnauthorized, "Invalid signature")
        return
    }

    event := c.GetHeader("X-Kyvshield-Event")
    sessionID := c.GetHeader("X-Kyvshield-Session")
    // Process event...
    _ = event
    _ = sessionID
    c.String(http.StatusOK, "OK")
})
```

## Response Structure

```go
type KycResponse struct {
    Success            bool              `json:"success"`
    SessionID          string            `json:"session_id"`
    OverallStatus      string            `json:"overall_status"`       // "pass" | "reject"
    OverallConfidence  float64           `json:"overall_confidence"`
    FaceVerification   *FaceVerification `json:"face_verification"`
    Steps              []StepResult      `json:"steps"`
    ProcessingTimeMs   int               `json:"processing_time_ms"`
}

type StepResult struct {
    StepIndex        int               `json:"step_index"`
    StepType         string            `json:"step_type"`           // "selfie" | "recto" | "verso"
    Success          bool              `json:"success"`
    Liveness         Liveness          `json:"liveness"`
    Verification     *Verification     `json:"verification"`
    Extraction       []ExtractionField `json:"extraction"`
    ExtractedPhotos  []ExtractedPhoto  `json:"extracted_photos"`
    AlignedDocument  string            `json:"aligned_document"`    // base64 JPEG (recto/verso)
    CapturedImage    string            `json:"captured_image"`      // base64 JPEG (selfie)
    ProcessingTimeMs int               `json:"processing_time_ms"`
}

type ExtractionField struct {
    Key             string `json:"key"`
    DocumentKey     string `json:"document_key"`
    Label           string `json:"label"`
    Value           string `json:"value"`
    DisplayPriority int    `json:"display_priority"`
}
```

## Error Handling

```go
resp, err := client.Verify(ctx, opts)
if err != nil {
    var apiErr *kyvshield.APIError
    if errors.As(err, &apiErr) {
        switch apiErr.StatusCode {
        case 401:
            fmt.Println("Invalid API key")
        case 422:
            fmt.Printf("Validation failed: %s\n", apiErr.Message)
            fmt.Printf("Details: %v\n", apiErr.Details)
        default:
            fmt.Printf("API error [%d]: %s\n", apiErr.StatusCode, apiErr.Message)
        }
    } else {
        // Network error, timeout, etc.
        fmt.Printf("Request failed: %v\n", err)
    }
    return
}
```

## Client Options

```go
client := kyvshield.NewClient("YOUR_API_KEY",
    kyvshield.WithBaseURL("https://kyvshield-naruto.innolinkcloud.com"),
    kyvshield.WithTimeout(120 * time.Second),
    kyvshield.WithHTTPClient(customHTTPClient),
)
```

| Option | Description | Default |
|--------|-------------|---------|
| `WithBaseURL(url)` | API base URL | Production endpoint |
| `WithTimeout(d)` | Request timeout | 120 seconds |
| `WithHTTPClient(c)` | Custom `*http.Client` | Default client |

## Notes

- **Context**: All methods accept `context.Context` for cancellation and deadline propagation
- **Max body size**: 30 MB total across all images
- **Image format**: JPEG recommended
- **Processing time**: 10-60 seconds depending on steps and challenge modes
- **Concurrency**: `VerifyBatch` uses goroutines internally; client is safe for concurrent use

## API Documentation

Full documentation: **[https://kyvshield-naruto.innolinkcloud.com/developer](https://kyvshield-naruto.innolinkcloud.com/developer)**

Source: [github.com/moussa-innolink/kyv_shield_restfull](https://github.com/moussa-innolink/kyv_shield_restfull)
