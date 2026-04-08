# KyvShield LLM Skills Plugin

Utilisez Claude Code pour integrer KyvShield automatiquement dans votre projet. Le plugin donne a Claude toute la documentation, les exemples et les types SDK.

## Installation

Installez le plugin dans Claude Code :

```bash
/install-plugin https://github.com/moussa-innolink/kyv_llm_skill
```

Puis demandez a Claude :

```
"Integre KyvShield dans mon app Flutter"
"Add KYC verification to my Android app"
"Help me handle KyvShield webhooks in Node.js"
"Set up REST KYC verification server-to-server"
```

## Skills disponibles

| Skill | Description |
|-------|-------------|
| `/kyvshield flutter` | Integration Flutter/Dart |
| `/kyvshield android` | Integration Android/Kotlin |
| `/kyvshield swift` | Integration iOS/Swift |
| `/kyvshield web` | Integration Web (CDN) |
| `/kyvshield reactnative` | Integration React Native |
| `/kyvshield webhooks` | Configuration des webhooks |
| `/kyvshield rest-kyc` | API REST KYC (server-to-server) |

## Ce que fait KyvShield

KyvShield fournit une verification d'identite (KYC) complete :

- **Liveness selfie** — detection anti-spoofing (yeux fermes, tourner la tete, sourire)
- **Capture de document** — OCR automatique (CIN, passeport, permis de conduire)
- **Face matching** — comparaison selfie vs photo du document
- **Multi-langue** — Francais, Anglais, Wolof
- **REST API** — Verification server-to-server sans SDK

## Documentation

- [Documentation developpeur](https://kyvshield-naruto.innolinkcloud.com/developer)
- [Plugin sur GitHub](https://github.com/moussa-innolink/kyv_llm_skill)
