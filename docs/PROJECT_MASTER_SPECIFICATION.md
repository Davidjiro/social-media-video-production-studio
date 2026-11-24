# Local-First AI Episodic Video Production Studio

**Master Project Specification**  
**Version:** 0.1  
**Date:** 2026-11-23  
**Status:** Initial requirements baseline for subsequent Requirements, HLD, LLD, and Implementation Plan documents.

> This is the authoritative starting specification for the project. Future design documents should link back here, state any deviations explicitly, and preserve approved decisions in Git history.

## 1. Project Overview

Design and implement a local-first Python application that converts supplied PDF or TXT stories into realistic, cinematic, episodic videos suitable for YouTube Shorts and Instagram Reels. The application is initiated by a local CLI command, not a cloud scheduler. The user controls stories, characters, generation, approvals, and final files from their own computer.

The application must not depend on AWS infrastructure, cloud-hosted databases, hosted workflow engines, or cloud application servers. Orchestration, storage, character registry, story state, local asset management, video editing, and rendering run locally.

External AI generation providers may still be used through supported APIs, and **may incur per-request costs**. Support Google Veo as an initial programmatic video-generation candidate. Google Flow can be used as a creative interface and optional manually mapped source of character assets; do not assume it exposes supported external character-management APIs or stable, externally usable character IDs.

Users must be able to create characters independently, assign immutable local IDs, register optional Google Flow aliases/references, reuse the characters in different stories, and retain continuity within each story.

## 2. Core Business Objectives

1. Accept story and episode documents in PDF and TXT formats.
2. Support multiple independent stories, each with approximately 5–10 or more episodes.
3. Produce configurable-length episodes, initially targeting **120 seconds**.
4. Produce **photorealistic human characters**, not cartoons.
5. Keep identities, voices, and major visual characteristics consistent across shots, episodes, and different stories as far as providers permit; validate drift.
6. Create characters independently from any story.
7. Assign immutable internal character IDs and searchable names/aliases.
8. Reuse registered characters in new storylines without recreation.
9. Maintain a locally inspectable character registry and provider mappings.
10. Resolve character names/IDs/tags in the supplied story before scene generation.
11. Generate shots/scenes, dialogue, voiceover, and optional sound effects or music.
12. Assemble complete vertical 9:16 MP4 videos locally.
13. Preserve episodes, job history, assets, and story-specific continuity.
14. Support human review, scene-level regeneration, and rerendering.
15. Keep social-media publishing outside the initial MVP; add later if approved.

## 3. Technology Requirements

- **Language/runtime:** Python 3.12+.
- **Local database:** SQLite with schema migrations.
- **Data validation:** Pydantic.
- **Configuration:** YAML and JSON; sensitive values in environment variables or secure local secret storage.
- **Story import:** PyMuPDF for text-based PDFs; standard Python for TXT. Scanned PDFs should trigger an actionable OCR requirement/optional extension.
- **Video processing:** FFmpeg and FFprobe; MoviePy only when justified.
- **LLM:** provider abstraction; Google Gemini is an initial option.
- **Video generation:** Google Veo through officially documented programmatic APIs, where available, plus optional local models later.
- **Audio/voice and image generation:** provider abstractions with support for imported local assets.
- **Testing:** pytest with mocked providers by default; real paid API tests opt-in.
- **UI:** CLI first; optional local UI later.
- **No required AWS CDK, ECS, Step Functions, Lambda, S3, hosted SQL, or cloud orchestration.**

## 4. Character Management Requirements

### 4.1 Character Factory

Provide character creation from a structured YAML/JSON specification. Support:

- Name and aliases.
- Sex/gender presentation appropriate to the fictional role.
- Age or age range.
- Requested cultural background/ethnicity as fictional appearance context.
- Skin tone/color, eye color, hair color/style/length.
- Face shape, distinctive facial features.
- Body type, approximate height, posture, mannerisms.
- Clothing style, common wardrobe, accessories.
- Languages spoken, accent, voice characteristics.
- Personality and other distinguishing attributes.

Generate a structured appearance/voice profile and optionally create photorealistic reference imagery (front portrait, side/three-quarter portrait, full-body view). Users can import preexisting reference images instead. Avoid unconsented impersonation and unauthorized voice cloning.

### 4.2 Permanent Identity and Versioning

Each character has an immutable local identifier such as `CHAR-0001`. Identity is independent of story and provider. Separate **canonical identity** (physical appearance, voice, enduring traits) from **story/scene attributes** (role, relationships, clothing, facial expression, lighting, location). Changing wardrobe must not mutate identity. Canonical identity changes must be explicit and versioned.

### 4.3 Character Registry

SQLite is authoritative. Provide JSON export/import, readable profiles, and search/list/show/update/archive operations. Store at least:

- `character_id`, `character_name`, `aliases`.
- `physical_description`, structured appearance attributes.
- Voice characteristics, language, accent, optional voice asset.
- Local reference image paths and checksums.
- Provider mappings, project references, and returned provider asset IDs **only when verified**.
- Created/updated timestamps, identity version, and active/archive status.

### 4.4 Google Flow Mapping

Allow optional *manual* registration of a Flow character's display name, project association, alias, and locally retained reference assets, e.g.:

```json
{
  "character_id": "CHAR-0001",
  "character_name": "Amara",
  "provider": "google_flow",
  "flow_character_name": "Amara",
  "flow_alias": "@Amara",
  "flow_project_id": null,
  "provider_asset_ids": [],
  "reference_images": [
    "data/characters/CHAR-0001/front.png",
    "data/characters/CHAR-0001/side.png"
  ]
}
```

**Do not invent a Google Flow character ID, promise cross-project alias portability, or assume a public API for creating/listing Flow characters.** Confirm actual alias syntax and API features against official documentation before implementing. No undocumented private APIs, scraping, or fragile browser automation should be presented as supported integration. If direct Flow integration is unavailable, import character images/references locally and use supported Veo video APIs.

### 4.5 Cross-Story Reuse

Support many-to-many character-to-story casting:

- `STORY-0001`: Amara → `CHAR-0001`, David → `CHAR-0002`.
- `STORY-0002`: Amara → `CHAR-0001`, Mama Grace → `CHAR-0003`.

Roles, personal relationships, costumes, and plot histories are **story-scoped**, even when the underlying character appearance is reused.

## 5. Story Management Requirements

Assign every story a stable ID (e.g. `STORY-0001`). Persist title, genre, synopsis, main and supporting plots, character cast, episode order, locations, story chronology, relationships, important events, and unresolved conflicts.

Import and retain original PDF/TXT files plus extracted/normalized text. Parse explicit episode delimiters. Where boundaries are missing or unclear, propose a split for approval; do not silently rewrite the plot. Keep separate story bibles and histories so one storyline never overwrites another.

## 6. Episode Management Requirements

Track episode ID, story ID, episode number/title, original input script, characters, planned scenes, dialogue, generation prompts, provider job IDs, generated assets, processing state, review state, and final rendered video path.

Generate/retry a single episode without regenerating prior episodes. Retry a single scene without regenerating successful scenes. Target duration defaults to 120 seconds, with configurable tolerance documented in LLD.

## 7. Scene Generation Requirements

For each scene, define and persist:

- Scene ID, order, duration.
- Location, characters present, actions.
- Dialogue, narration, sound requirements.
- Camera angle/movement/framing.
- Lighting, atmosphere, emotional tone.
- Prompt text, linked character reference assets.
- Generation request/status/output paths.

Resolve character references to existing registry IDs before generation. Unrecognized or ambiguous aliases require explicit resolution; never silently generate a new identity for a known name. Prompt construction combines the canonical character profile with scene-specific instructions and references while respecting model limitations (e.g., number of reference images and maximum clip duration).

## 8. AI Provider Architecture

Define narrow, testable Python interfaces such as:

```python
from typing import Protocol

class VideoProvider(Protocol):
    async def generate_scene(self, request: "SceneGenerationRequest") -> "GenerationJob": ...
    async def get_job_status(self, job_id: str) -> "GenerationStatus": ...
    async def download_result(self, job_id: str, destination: str) -> str: ...
    async def cancel_job(self, job_id: str) -> None: ...
```

Implement a documented Google Veo adapter and future adapters for local video models. Google Flow integration remains optional and limited to supported capabilities or manual mapping. Similarly abstract LLM, image, and voice providers. Place API authentication, rate limits, provider errors, job polling, and download behavior inside the adapters.

Support asynchronous calls and persistent job identifiers. Do not assume every model supports reference images, synchronized dialogue, a fixed clip duration, or identical request schemas.

## 9. Local Audio and Video Rendering

Download and save scene assets locally. Use FFmpeg for scene concatenation, transitions, music and sound effects, dialogue/narration mixing, audio alignment, subtitle rendering, crop/pad, encoding, and output. Use FFprobe to validate outputs.

Default rendering profile:
- MP4 container.
- Vertical **9:16**.
- Target **1080 × 1920**.
- Configurable frame rate.
- AAC audio.
- **120-second** target, configurable tolerance.

The scene planner must accommodate provider-specific clip length limits; it may combine multiple shots per story scene and edit timing, subject to quality approval.

## 10. Story Continuity

Maintain a persistent story bible for each story. After an **approved** episode, save a summary, chronology, character state/role changes, relationships, new locations, discoveries, key dialogue, unresolved plots, and information required for later episodes.

Before planning a subsequent episode, load that story's prior context. Permanent character appearance is shared globally; narrative history and relationships are not shared across unrelated stories unless explicitly requested.

## 11. Local Configuration

Use centralized, editable configuration, such as:

- `config/settings.yaml`
- `config/video_providers.yaml`
- `config/rendering.yaml`
- `config/characters.json` — exported snapshot/mapping; SQLite remains authoritative.

Configure generation providers/models, story/episode duration, resolution, aspect ratio, maximum concurrent jobs, retry settings, paths, approval workflow, budget guardrails, and validation behavior. Keep secrets out of Git.

## 12. CLI Contract (Initial Proposal)

```bash
python main.py character create --name Amara --spec ./characters/amara.yaml
python main.py character list
python main.py character show --id CHAR-0001
python main.py character register-provider --id CHAR-0001 --provider google_flow --name Amara
python main.py character export --format json

python main.py story import --file ./stories/the_hidden_truth.pdf
python main.py story list

python main.py episode generate --story STORY-0001 --episode 01
python main.py episode preview --story STORY-0001 --episode 01
python main.py episode render --story STORY-0001 --episode 01
python main.py scene regenerate --story STORY-0001 --episode 01 --scene SCENE-004
```

Add a dry-run option to validate story structure, cast mappings, shot plan, estimated usage/cost, and provider readiness **without paid generation**.

## 13. Reliability and Error Handling

Implement structured local logs, persistent job states, database transactions/migrations, idempotency, retries with backoff, rate-limit handling, timeouts, partial episode recovery, scene-level restarts, checksums for local assets, and clear validation errors. Recover after machine reboot or terminal interruption. Avoid uncontrolled polling loops, duplicate submissions, and preventable duplicate API charges.

## 14. Testing

Provide tests for:

- Character creation, stable IDs, registry CRUD and export/import.
- Cross-story cast mapping and identity reuse.
- PDF/TXT parsing and episode segmentation.
- Scene generation and character alias resolution.
- Prompt composition and story continuity isolation.
- Provider adapters with mocked responses.
- Job persistence and interruption/recovery.
- Video downloads, FFmpeg assembly, audio/subtitles.
- FFprobe output validation and failure cases.

Paid integration tests must be opt-in and must never be required for the standard unit-test suite.

## 15. Documentation Deliverables

Create and version the following:

1. `docs/REQUIREMENTS.md` — functional/nonfunctional requirements, user stories, business rules, assumptions, constraints, acceptance criteria, traceable requirement IDs.
2. `docs/HLD.md` — architecture, components, major dataflows, boundaries, provider integrations, reliability, security, diagrams.
3. `docs/LLD.md` — package/class design, SQLite schema and migrations, interfaces, sequence diagrams, CLI/config contracts, exception handling.
4. `docs/IMPLEMENTATION_PLAN.md` — phases, milestones, implementation order, testing approach, acceptance gates.
5. `README.md` — installation, environment, local CLI, provider credentials, usage, troubleshooting.
6. `docs/decisions/` — architecture decision records (ADRs) for important design choices.

Preserve this master file as the initial baseline; amend it through explicit Git commits and link subsequent documents back to it.

## 16. Development Instructions for a Programming LLM

**Do not generate the entire codebase immediately.**

1. Analyze requirements, validate official provider/API capabilities, and identify constraints/costs.
2. Write `docs/REQUIREMENTS.md` with requirement IDs, acceptance criteria, assumptions, and unanswered questions; request review.
3. Write `docs/HLD.md` with architecture and data flows; request review.
4. Write `docs/LLD.md` with concrete implementation design; request review.
5. Write `docs/IMPLEMENTATION_PLAN.md` with milestones/test gates; request review.
6. Implement incrementally, starting with the local character factory, registry, and asset management.
7. Execute unit/integration tests and only explicitly authorized paid API tests.
8. Deliver a reproducible local end-to-end demonstration and installation/operations instructions.

Never invent provider APIs or character IDs. Never present undocumented Flow endpoints or browser automation as official integration. Provide documented manual/provider-neutral fallbacks where Flow lacks supported APIs.

## 17. Definition of Done

The MVP is successful when the user can:

1. Install and run the app locally.
2. Create a realistic fictional character with an immutable ID.
3. Persist, view, export, and retrieve character metadata and references.
4. Register an optional Google Flow character name/alias and any available assets.
5. Import a PDF/TXT story.
6. Associate already-created characters with that story.
7. Generate one selected episode with those identities.
8. Preserve character references and continuity across scenes/episodes.
9. Recover interrupted jobs and selectively regenerate failed scenes.
10. Render a complete approximately two-minute vertical video locally.
11. Save the final MP4 to a predictable episode directory.
12. Reuse those characters in a completely different story with separate narrative state.

The solution must be modular, maintainable, locally operated, testable, provider-independent where feasible, and extensible.

## 18. Constraints, Risks, and Open Decisions

- **Flow API constraint:** The availability of any officially supported external character CRUD/API workflow must be verified. Local mappings and manual Flow asset import are guaranteed application features; remote Flow control is conditional.
- **Appearance consistency:** No generator guarantees exact character identity across shots. Use references, checks, approvals, and retries.
- **Costs:** Local orchestration is inexpensive, but commercial AI APIs may bill. Set budget controls and dry-run defaults.
- **Privacy and rights:** Use authorized character likenesses, voices, and music. Label synthetic/altered content where platforms require it.
- **Local model hardware:** Completely offline video generation depends on local GPU/VRAM and supported model licenses; it is a future adapter, not a prerequisite for MVP.
- **Scheduling/publishing:** Daily automated social posting is explicitly deferred; the MVP is manually triggered per episode.
- **Future decisions:** Choose initial provider billing/auth mechanism, acceptable generation budget, character review workflow, clip duration strategy, voice/lip-sync approach, and whether to add a local UI.

## 19. Proposed Repository Layout

```text
social-media-video-production-studio/
├── docs/
│   ├── PROJECT_MASTER_SPECIFICATION.md
│   ├── REQUIREMENTS.md
│   ├── HLD.md
│   ├── LLD.md
│   ├── IMPLEMENTATION_PLAN.md
│   └── decisions/
├── config/
├── data/             # local story inputs and character assets; typically ignored from Git
├── src/
│   ├── character_manager/
│   ├── story_engine/
│   ├── scene_engine/
│   ├── providers/
│   ├── audio/
│   ├── rendering/
│   └── orchestration/
├── tests/
├── output/           # generated media; typically ignored from Git
├── main.py
├── pyproject.toml
├── .env.example
├── .gitignore
└── README.md
```

Keep generated videos, large assets, database files, and real credentials out of ordinary Git commits. Only store example profiles and non-sensitive, appropriately licensed fixtures as needed.
