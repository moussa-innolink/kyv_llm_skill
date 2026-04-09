# KyvShield LLM Skills Plugin

Use Claude Code to automatically integrate KyvShield into your project. This plugin gives Claude all the documentation, examples, and SDK types it needs.

## Installation

Install the plugin in Claude Code:

```
/plugin marketplace add moussa-innolink/kyv_llm_skill
/plugin install kyvshield@kyvshield-plugins
/reload-plugins
```

Then ask Claude:

```
"Integrate KyvShield into my Flutter app"
"Add KYC verification to my Android app"
"Help me handle KyvShield webhooks in Node.js"
"Set up REST KYC verification server-to-server"
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `/kyvshield flutter` | Flutter/Dart integration |
| `/kyvshield android` | Android/Kotlin integration |
| `/kyvshield swift` | iOS/Swift integration |
| `/kyvshield web` | Web integration (CDN) |
| `/kyvshield reactnative` | React Native integration |
| `/kyvshield webhooks` | Webhook configuration |
| `/kyvshield rest-kyc` | REST KYC API (server-to-server) |

## What KyvShield Does

KyvShield provides complete identity verification (KYC):

- **Selfie liveness** — anti-spoofing detection (close eyes, turn head, smile)
- **Document capture** — automatic OCR (ID cards, passports, driver licenses)
- **Face matching** — selfie vs document photo comparison
- **Multi-language** — French, English, Wolof
- **REST API** — server-to-server verification without SDK
- **Server SDKs** — typed REST clients for Node.js, PHP, Java, Kotlin, Go with batch support

## Server SDKs (REST API)

For server-to-server KYC verification (no camera/UI needed):

| Language | Install |
|----------|---------|
| Node.js/TypeScript | `npm install @kyvshield/rest-sdk` |
| PHP | `composer require kyvshield/rest-sdk` |
| Java | JitPack: `com.github.moussa-innolink.kyv_shield_restfull:java:1.1.0` |
| Kotlin | JitPack: `com.github.moussa-innolink.kyv_shield_restfull:kotlin:1.1.0` |
| Go | `go get github.com/moussa-innolink/kyv_shield_restfull/go` |

## Documentation

- [Developer documentation](https://kyvshield.innolink.sn/developer)
- [Server SDKs](https://github.com/moussa-innolink/kyv_shield_restfull)
- [Plugin on GitHub](https://github.com/moussa-innolink/kyv_llm_skill)
