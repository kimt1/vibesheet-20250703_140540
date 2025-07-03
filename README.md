```markdown
# vibesheet-20250703_140540  
_Universal JayZee Form-Filler ? ?OmniForm Phantom?_

A cross-platform **browser extension (MV3)** + **headless CLI** that reads data from Google Sheets, intelligently maps sheet columns to any web-form, fills the form while mimicking human behaviour, handles CAPTCHAs, and writes the result (incl. screenshots / diagnostics) back to the sheet ? all with zero plaintext secrets leaving your machine.

---

## Table of Contents
1. [What does it do?](#what-does-it-do)
2. [Features](#features)
3. [Architecture](#architecture)
4. [Getting Started](#getting-started)
   * [Prerequisites](#prerequisites)
   * [Install the browser extension](#install-the-browser-extension)
   * [Install / run the CLI](#install--run-the-cli)
5. [Quick Usage](#quick-usage)
   * [Interactive extension flow](#interactive-extension-flow)
   * [Headless CLI flow](#headless-cli-flow)
6. [Configuration](#configuration)
7. [Component Reference](#component-reference)
8. [Dependencies](#dependencies)
9. [Security Model](#security-model)
10. [Road-map](#road-map)
11. [Contributing](#contributing)
12. [License](#license)

---

## What does it do?
? Crawls the full DOM (incl. shadow DOM + iframes) of any page  
? Suggests a **column ? selector** mapping for a Google Sheet  
? Fills hundreds of rows while simulating typing cadence, mouse jitter and random delays  
? Supports pluggable CAPTCHA providers (2Captcha, Anti-Captcha, ?)  
? Updates the originating sheet with success/error codes, timings and a link to a screenshot  
? Offers a **Playwright-powered CLI** for CI/CD or server-side runs  
? Stores all selector maps & credentials in an **AES-GCM encrypted vault** on your device  

---

## Features
* DOM + shadowDOM + iframe crawling with heuristic ranking  
* Google Sheets OAuth-PKCE read/write with exponential back-off  
* Parallel multi-tab execution (extension) & Playwright pool (CLI)  
* WCAG-AA compliant React popup wizard (keyboard & screen-reader friendly)  
* Local encrypted audit log & artefact export (JSON/CSV)  
* Internationalisation template (`i18n/messages.pot`) ready for translation  
* Optional AI-powered auto-mapping hooks (Gemini / GPT-4 / DeepSeek)

---

## Architecture
Monorepo managed by **npm workspaces**

```
vibesheet/
??? extension/            # MV3 bundle (Vite)
?   ??? src/
?   ?   ??? background.ts         # service-worker orchestrator
?   ?   ??? contentScript.ts      # DOM scanner & filler
?   ?   ??? popup/                # React wizard
?   ?   ??? services/             # shared logic (DomScanner, ?)
?   ??? manifest.json
??? cli/                   # Headless automation
?   ??? runJob.ts
?   ??? browserPool.ts
?   ??? configLoader.ts
??? shared/                # isomorphic utils (crypto, types)
??? config/                # default.ini, sampleMapping.csv
??? i18n/                  # messages.pot
??? tests/                 # Jest / Playwright
??? .github/workflows/ci.yml
```

---

## Getting Started

### Prerequisites
* **Node ? 18** (LTS recommended)  
* **pnpm** or **npm**  
* Google account (to authorise Sheets)  
* Chrome / Chromium-based browser supporting MV3

### Install the browser extension
```bash
git clone https://github.com/your-org/vibesheet.git
cd vibesheet
pnpm install       # or npm i
pnpm run build:ext # outputs extension/dist
```
1. Open `chrome://extensions` ? ?Load unpacked? ? select `extension/dist`.  
2. Pin the icon and click it once to set a master password and grant Google OAuth.

### Install / run the CLI
```bash
# global install (npm registry)
npm i -g @your-org/vibesheet-cli
# or run from source
pnpm --filter cli build
```
Verify:
```bash
vibesheet --help
```

---

## Quick Usage

### Interactive extension flow
1. Browse to the target form.  
2. Click the extension icon ? **Scan**.  
3. Review / edit the suggested **Map** of sheet headers ? selectors.  
4. **Run** ? pick rows, watch real-time progress, abort or resume at will.

### Headless CLI flow
```bash
vibesheet \
  --config myJob.yaml      # YAML or JSON
  --mapping mapping.csv    # or let the CLI auto-map
  --sheet 1BxiMVs0XRA5nFM
  --range 'Sheet1!A2:E101'
```
On completion artefacts land under `./artifacts/YYYY-MM-DD/` (JSON report, screenshots, log).

---

## Configuration
* **config/default.ini** ? time-outs, captcha provider URL, log level, etc.  
* **sampleMapping.csv** ? header, css/xpath selector examples.  
* **myJob.yaml** ? CLI job descriptor:

```yaml
sheetId: 1BxiMVs0XRA5nFM
range: Sheet1!A2:E
mapping: ./mapping.csv
parallelTabs: 3
captcha:
  provider: 2captcha
  apiKey: $CAPTCHA_TOKEN
```

Environment variables override any file key (`VIBESHEET__GOOGLE_CLIENT_ID`, etc.).

---

## Component Reference (high-level)

| Component | Purpose |
|-----------|---------|
| `extension/src/background.ts` | Service-worker orchestrator, message bus, alarm scheduling |
| `extension/src/contentScript.ts` | Injected scanner & filler, talks to background |
| `services/DomScanner.ts` | Recursively traverses DOM/shadowRoots, ranks selectors |
| `services/SelectorVault.ts` | AES-GCM vault (IndexedDB / chrome.storage) |
| `services/MappingEngine.ts` | Suggests & validates column ? selector mapping |
| `services/GoogleSheetsService.ts` | OAuth-PKCE, read / write / retry |
| `services/FormFillerRunner.ts` | Executes fill flow per sheet row |
| `services/HumanSimulator.ts` | Types, clicks, moves cursor like a human |
| `services/CaptchaHandler.ts` | Detects & solves common CAPTCHAs |
| `services/AuditLogger.ts` | Encrypted log + screenshot rotation |
| `cli/runJob.ts` | CLI entry; loads config, spawns browser pool |
| `cli/browserPool.ts` | Playwright context pool |
| `cli/configLoader.ts` | YAML/JSON ? runtime config |
| `.github/workflows/ci.yml` | Lint / test / build / publish pipeline |

_(See [docs/COMPONENTS.md](docs/COMPONENTS.md) for the full generated list.)_

---

## Dependencies
Runtime
* `@playwright/test` (?1.45) ? headless automation
* `react`, `react-dom`
* `vite`, `typescript`, `ts-lib-config`
* `dexie` ? IndexedDB wrapper
* `@googleapis/sheets`, `google-auth-library`
* `axios` ? network util (CAPTCHA)
* `yaml`, `ini`, `csv-parse`
* `crypto` / WebCrypto polyfills

Dev / CI
* `jest` + `ts-jest`
* `eslint`, `prettier`
* `chrome-webstore-upload-cli`

---

## Security Model
* **AES-GCM (256 bit)** vault secured by a **user-supplied master password**.  
* OAuth tokens stored encrypted; refresh handled via chrome.identity.  
* No selectors, data or screenshots leave the local machine unless **you** connect a third-party CAPTCHA provider.  
* Optional `--air-gap` flag disables any outbound network except Google APIs & target site.

---

## Road-map
* Firefox (Web-Extension) build  
* Native reCAPTCHA v3 solver (no external service)  
* Sheet-driven conditional logic (if/else per row)  
* Electron wrapper for non-browser users  

Contributions & feature requests are very welcome!

---

## Contributing
1. Fork ? create branch (`feat/xyz`)  
2. `pnpm i && pnpm test` ? all green?  
3. Submit PR ? please follow the commit style in `CONTRIBUTING.md`.

---

## License
Licensed under the **MIT License** ? see [`LICENSE`](LICENSE) for details.

---

> Happy form-filling! If you hit a weird edge-case, open an issue with the debug log and we?ll squash it together ?
```