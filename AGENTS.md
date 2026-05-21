## Cursor Cloud specific instructions

This repository is an **artifact-only repository** — it contains three pre-built Android APK files for the MiFPS product suite (by Bestinet) and a markdown document (`BASE_URL_EXTRACTION.md`) with static analysis of those APKs.

### Repository contents

| File | Description |
|------|-------------|
| `MiFPS Employer_1.0.0.apk` | Employer-facing Flutter Android app (~41 MB) |
| `MiFPS RA_1.0.0.apk` | Registration Agent Flutter Android app (~38 MB) |
| `MiFPS Worker_1.0.0.apk` | Worker-facing Flutter Android app (~39 MB) |
| `BASE_URL_EXTRACTION.md` | Static analysis documenting backend URLs, Firebase config, OAuth endpoints |

### What is NOT in this repository

- No source code, build system, or dependency files
- No test framework or lint configuration
- No `package.json`, `requirements.txt`, `Makefile`, or `Dockerfile`
- No runnable services or development server

### Development notes

- There is nothing to build, lint, test, or run as a dev server. All work in this repo involves the APK binaries or documentation.
- APK files can be inspected with standard Android tools (`aapt`, `apktool`, `jadx`) if static analysis is needed.
- The markdown documentation can be edited directly.
