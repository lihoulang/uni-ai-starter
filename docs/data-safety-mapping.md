# Data Safety Mapping

Updated: 2026-05-02

This file is a working draft for Google Play `Data safety`. Final answers must match the production backend and legal pages exactly.

## Data Types Currently In Scope

### Personal info

- Username
- Optional nickname

Purpose:

- account creation and authentication
- account display inside the app

### App activity

- Chat prompts
- Conversation titles
- AI report submissions

Purpose:

- provide core AI features
- preserve history
- abuse prevention and moderation handling

### Photos and videos

- User-uploaded images for image understanding

Purpose:

- multimodal AI request processing

### App info and performance

- Basic request metadata and failure context if you later add logging/monitoring

Purpose:

- diagnostics
- abuse prevention

## Third-Party Processing

Current code paths may send user prompts or images to model providers when the user explicitly triggers AI features.

Review before release:

- DeepSeek
- DashScope / Qwen / Wanx related endpoints
- Volcano Engine / Doubao related endpoints
- Google Gemini related endpoints

## Questions To Confirm Before Filling Play Console

- Is chat history stored server-side in production?
- How long are logs retained?
- Are backups enabled?
- Is data encrypted at rest in production?
- Is data shared with any analytics SDKs?
- Are crash reporting tools enabled?
- Will you ship AI video in v1.0?

## Current In-App Controls

- logout
- delete account
- privacy policy page
- account deletion explanation page
- AI output report entry

## Release Reminder

Do not copy this file into Play Console blindly. Treat it as an engineering source checklist, then align the final declaration with legal, backend, and actual production behavior.
