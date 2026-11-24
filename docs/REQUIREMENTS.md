# Software Requirements Specification (SRS)
## Local-First AI Episodic Video Production Studio

| Metadata | Value |
|---|---|
| Document ID | SRS-001 |
| Version | 0.1.0 — Draft for review |
| Date | 2026-11-23 |
| Project | Social Media Video Production Studio |
| Status | Draft; implementation decisions not yet approved |
| Source of truth | [PROJECT_MASTER_SPECIFICATION.md](./PROJECT_MASTER_SPECIFICATION.md) |
| Next artifact | `docs/HLD.md` after requirements review |

> **Scope control:** This SRS derives from the master specification. It refines that baseline into testable requirements but does not assert that undocumented Google Flow APIs exist, guarantee perfect character likeness, or require an AWS deployment. Requirements marked **MVP** define the initial end-to-end application; **Later** requirements are deferred. Acceptance criteria and proposed thresholds are subject to user approval.

## 1. Purpose and Scope

### 1.1 Purpose

Specify the functional and non-functional requirements for a **locally run, manually triggered** application that ingests user-supplied story scripts in PDF/TXT, resolves reusable fictional characters by stable local IDs, plans short film scenes, invokes selected AI image/video/voice services, and renders approximately two-minute photorealistic vertical video episodes to local MP4 files.

The user creates and retains a character library independently of stories. A character can be assigned to multiple unrelated stories and retain the same visual identity while playing different roles. A story typically contains **5–10 episodes**, with more supported.

### 1.2 Scope

**In scope for MVP**

- Local Python CLI, local configuration, local state, local storage, and recoverable generation jobs.
- Character creation/import, character profiles, local image/voice reference assets, unique IDs, search and reuse.
- Optional manual registry of a Google Flow character name/alias and reference artifacts, without relying on unsupported APIs.
- PDF and TXT story/episode import, explicit episode parsing, reviewable proposed segmentation.
- Story-specific cast assignments, episode and scene planning, character resolution, continuity.
- Documented programmatic external provider adapter(s), initially targeting Google Veo where API access and features are confirmed.
- Image/voice/LLM provider abstractions and local asset imports.
- Shot generation, asynchronous status tracking, scene-level retry, asset download.
- Local FFmpeg rendering, audio/subtitles, MP4 validation, manual approval, output archiving.
- Paid-generation safeguards and no-charge dry run; automated tests without live API spending.

**Not required in MVP**

- Automatic daily creation or timed scheduling.
- YouTube/Instagram account connection, upload, scheduling or analytics.
- Cloud infrastructure (AWS CDK, Lambda, Step Functions, ECS, S3, managed databases).
- A hosted web app or mandatory local desktop UI.
- Direct Flow character creation/list/sync via an undocumented endpoint.
- Fully offline local video inference, which depends on hardware and supported models.
- Guaranteed photorealism, perfect face identity, lip sync or native AI video audio from all providers.
- Automatic character creation triggered by an unknown name in a story.

### 1.3 Intended Users

- **Creator/Operator:** imports and authors stories, creates characters, selects casts, runs generation, inspects and approves outputs.
- **Maintainer/Developer:** installs dependencies, implements providers, maintains configurations and tests.
- Initial deployment is single-user on a local computer; multiuser roles/permissions are not required.

### 1.4 Glossary

| Term | Meaning |
|---|---|
| Character | Reusable fictional visual/voice identity, globally registered |
| Character ID | Application-owned immutable identifier, e.g. `CHAR-0001` |
| Provider mapping | Optional association between a local character and a remote provider's name, project or verified asset ID |
| Story | Independent narrative with its own cast assignments, timeline and story bible |
| Episode | Ordered installment belonging to one story |
| Scene | Narrative unit containing one or more generated video shots |
| Shot | Video clip generated or imported as a renderable asset |
| Story bible | Persistent story-scoped facts and continuity state |
| Dry run | Plan/validate/estimate without paid generation requests |
| Flow-assisted | Manual Google Flow creative activity plus locally imported/mapped artifacts, absent a supported automation API |
| Provider adapter | Code translating common application contracts into a specific AI service's supported API |

## 2. Stakeholder Goals and User Stories

| User story ID | User story | Success condition |
|---|---|---|
| US-001 | As a creator, I want to define a character once so I can reuse them in many episodes. | Saved character receives a permanent ID and is retrievable. |
| US-002 | As a creator, I want to associate a character with a Google Flow name or reference so I can recognize my existing Flow assets. | Local mapping can be saved, displayed and used in a Flow-assisted workflow. |
| US-003 | As a creator, I want to see my registered characters before writing/casting a story. | Search/list and JSON export show all relevant characters. |
| US-004 | As a creator, I want to import a story from PDF or TXT so I don't manually retype it. | Original and extracted content are retained and episode structure presented. |
| US-005 | As a creator, I want to specify character IDs in my story so the same identity is reused reliably. | All character mentions resolve or fail with an actionable error. |
| US-006 | As a creator, I want to generate Episode 3 without recreating Episodes 1–2. | Only Episode 3 is submitted; prior assets remain unchanged. |
| US-007 | As a creator, I want a draft scene plan before paying for video generation. | Dry run produces scene plan and provider/cost readiness report with zero generation calls. |
| US-008 | As a creator, I want to regenerate a faulty scene so I don't pay to regenerate the whole episode. | Only selected scene is resubmitted; other successful clips are reused. |
| US-009 | As a creator, I want to resume after an interruption so I don't lose work or pay twice unnecessarily. | Persisted jobs and local artifacts are recovered and safely reconciled. |
| US-010 | As a creator, I want to approve a video before it is considered final. | Review state is explicit and final approval is recorded. |
| US-011 | As a creator, I want one character to feature in an unrelated new story. | Identity remains consistent; narrative facts stay separate. |
| US-012 | As a developer, I want a replaceable video provider so I can use other services later. | Core story/scene modules depend only on provider contracts. |

## 3. Functional Requirements

**Priority key:** `MVP` = required for first usable application; `Later` = future extension. Each requirement uses a stable identifier for traceability. Requirements describe *what* the system must do; implementation details are deferred to HLD/LLD except when explicitly constrained by the master specification.

### 3.1 Local Execution and Configuration

| ID | Priority | Requirement |
|---|---|---|
| FR-LOC-001 | MVP | The application **shall** run as a Python 3.12+ process on the operator's local computer. |
| FR-LOC-002 | MVP | The application **shall** support CLI commands without requiring a hosted service. |
| FR-LOC-003 | MVP | The application **shall** persist its authoritative records in local SQLite with a migration path. |
| FR-LOC-004 | MVP | The application **shall** load non-secret provider, rendering, concurrency and path settings from local YAML/JSON files. |
| FR-LOC-005 | MVP | The application **shall** acquire API credentials from environment variables or supported secure local credential storage, not source-controlled config. |
| FR-LOC-006 | MVP | The application **shall** let the operator configure media and database directories and shall avoid overwriting existing outputs without explicit intent. |
| FR-LOC-007 | MVP | The application **shall** provide a dry-run mode which does not invoke any paid generation operation. |
| FR-LOC-008 | Later | The application **may** offer a local browser/desktop UI without changing core services. |

### 3.2 Character Creation and Canonical Identity

| ID | Priority | Requirement |
|---|---|---|
| FR-CHR-001 | MVP | The application **shall** create characters from YAML/JSON specifications independently of any story. |
| FR-CHR-002 | MVP | Character specifications **shall** support name, aliases, sex/gender presentation, age range, requested fictional cultural background, skin tone, eye color, hair attributes, facial features, body type, height range, wardrobe style, personality, language/accent and voice characteristics, where provided. |
| FR-CHR-003 | MVP | Each character **shall** receive a globally unique, stable, immutable local ID. |
| FR-CHR-004 | MVP | The application **shall** store canonical identity traits separately from per-story role/relationships and per-scene appearance. |
| FR-CHR-005 | MVP | The application **shall** permit importing local front, side/three-quarter, full-body and optional voice reference assets. |
| FR-CHR-006 | MVP | The application **shall** support generating reference imagery through a replaceable image provider when configured and approved. |
| FR-CHR-007 | MVP | The application **shall** preserve local reference file paths and integrity metadata for references it manages. |
| FR-CHR-008 | MVP | Canonical identity edits **shall** be explicit and versioned; scene clothing or expressions must not mutate canonical identity. |
| FR-CHR-009 | MVP | The application **shall** support character create, show, list/search, update, archive, JSON export and import operations. |
| FR-CHR-010 | MVP | The application **shall** detect or reject duplicate/ambiguous aliases during character resolution, without silently selecting the wrong identity. |
| FR-CHR-011 | MVP | Generated likeness/voice references **shall** require appropriate rights/consent as applicable; the application shall not silently clone a real person's voice/likeness. |

### 3.3 Google Flow and Provider Identity Mapping

| ID | Priority | Requirement |
|---|---|---|
| FR-FLOW-001 | MVP | The application **shall** allow manual association of a local character ID with a Flow display name and an optional alias such as `@Amara`. |
| FR-FLOW-002 | MVP | The application **shall** permit optional Flow project information, manual notes, and imported reference assets to be associated with the character. |
| FR-FLOW-003 | MVP | The application **shall** store a provider asset/project ID only when it was returned by, or independently verified against, a supported provider interface; otherwise the field shall remain unset. |
| FR-FLOW-004 | MVP | The application **shall** work when Flow mappings are absent or unusable; local canonical reference assets remain authoritative. |
| FR-FLOW-005 | MVP | The application **shall** allow retrieving and exporting a character's provider mappings without exposing secret credentials. |
| FR-FLOW-006 | Later / conditional | Automated create/list/sync in Flow **may** be implemented only if supported, stable, and documented integration capabilities are confirmed. |
| FR-FLOW-007 | MVP | Flow-assisted generation instructions **shall** identify which steps are manual rather than presenting them as automatic API actions. |

### 3.4 Story Import, Episode Parsing and Cast

| ID | Priority | Requirement |
|---|---|---|
| FR-STORY-001 | MVP | The application **shall** import UTF-8 TXT and readable text-based PDF story/episode content from local paths. |
| FR-STORY-002 | MVP | The application **shall** store the original input file and normalized/extracted text with source metadata. |
| FR-STORY-003 | MVP | The application **shall** assign stable story IDs and maintain title, genre, synopsis, plots, episode ordering and cast. |
| FR-STORY-004 | MVP | The application **shall** recognize explicit episode delimiters and provide a proposed segmentation for ambiguous or unsplit content. |
| FR-STORY-005 | MVP | Ambiguous segmentation and material storyline alterations **shall** require explicit operator approval before generation. |
| FR-STORY-006 | MVP | A story **shall** associate local character IDs with story-specific role names/aliases, relationships and wardrobe overrides. |
| FR-STORY-007 | MVP | The application **shall** validate every referenced character name/tag/ID and block generation if any reference is missing or ambiguous. |
| FR-STORY-008 | MVP | The application **shall** support at least 10 episodes in one story without fixed-code episode limits. |
| FR-STORY-009 | MVP | The application **shall** allow a character to be assigned to multiple stories while isolating each story's plot state. |
| FR-STORY-010 | Later | Image-only/scanned PDF input **may** be supported through an optional OCR step with explicit accuracy warnings. |

### 3.5 Episode, Scene and Prompt Planning

| ID | Priority | Requirement |
|---|---|---|
| FR-PLAN-001 | MVP | The application **shall** generate a structured episode plan from selected episode content and the story bible. |
| FR-PLAN-002 | MVP | The episode plan **shall** include ordered scenes with scene IDs, duration targets, locations, characters, actions, dialogue/narration, camera and lighting instructions, emotional tone and referenced assets. |
| FR-PLAN-003 | MVP | The application **shall** transform narrative scenes into one or more shots within the chosen provider's supported clip length and input constraints. |
| FR-PLAN-004 | MVP | Each shot prompt **shall** include resolved canonical character details and available reference images rather than relying on a bare name. |
| FR-PLAN-005 | MVP | The application **shall** persist the planned scene/shot structure and corresponding prompts for inspection and reproducibility. |
| FR-PLAN-006 | MVP | Episode duration **shall** be configurable and default to a target of 120 seconds, not to an assumption of single-request generation. |
| FR-PLAN-007 | MVP | The operator **shall** be able to inspect and approve the proposed plan before incurring paid generation costs. |
| FR-PLAN-008 | MVP | The application **shall** support generation, retry, or rerender of an individual scene without replacing all other approved scenes. |
| FR-PLAN-009 | MVP | The application **shall** preserve user-authored plot points and shall flag proposed major additions/changes for approval. |

### 3.6 AI Provider Integration and Job Management

| ID | Priority | Requirement |
|---|---|---|
| FR-AI-001 | MVP | The application **shall** define provider-neutral interfaces for LLM, image, video and voice generation. |
| FR-AI-002 | MVP | At least one officially documented programmatic video provider **shall** be implemented and tested for the MVP; Google Veo is preferred subject to account access and applicable capabilities. |
| FR-AI-003 | MVP | Provider adapters **shall** handle authentication, request translation, response parsing, transient errors, rate limits and unsupported feature detection. |
| FR-AI-004 | MVP | The application **shall** persist provider job identifiers, provider/model names, request metadata, statuses and output paths. |
| FR-AI-005 | MVP | Asynchronous generation jobs **shall** be monitored with bounded polling/backoff and timeouts, without uncontrolled busy loops. |
| FR-AI-006 | MVP | The application **shall** support configurable maximum concurrent jobs to limit cost and provider overload. |
| FR-AI-007 | MVP | The application **shall** preserve successful generation results during scene-level failures. |
| FR-AI-008 | MVP | The application **shall** surface actionable failures for unsupported references, quota exhaustion, expiration, inaccessible results and provider unavailability. |
| FR-AI-009 | MVP | The application **shall** provide preflight provider readiness and estimated cost/usage information where pricing/usage is known, indicating uncertainty otherwise. |
| FR-AI-010 | MVP | The application **shall** support explicit configured budget/approval gates before new paid generation requests. |
| FR-AI-011 | Later | The application **may** integrate local video inference via an alternate adapter and explicit hardware validation. |
| FR-AI-012 | MVP | The application **shall not** assume equal clip durations, reference image counts, audio capabilities or character IDs across providers. |

### 3.7 Audio, Subtitles, Rendering and Approval

| ID | Priority | Requirement |
|---|---|---|
| FR-VID-001 | MVP | The application **shall** save imported/generated shot clips and associated audio into local episode-scoped directories. |
| FR-VID-002 | MVP | The application **shall** support dialogue and narration audio through selected provider assets or user-imported recordings. |
| FR-VID-003 | MVP | The application **shall** support optional licensed music and sound effects and mix them with dialogue/narration. |
| FR-VID-004 | MVP | The application **shall** support readable subtitles/captions, including a way to correct text/timing before final render. |
| FR-VID-005 | MVP | The application **shall** use local FFmpeg processing for assembly, timing, transitions, scaling/cropping/padding, mixing and export. |
| FR-VID-006 | MVP | The default final export **shall** be a vertical 9:16 MP4, target 1080×1920, with configurable frame rate and AAC audio. |
| FR-VID-007 | MVP | The application **shall** use FFprobe or equivalent inspection to validate resolution, aspect ratio, container, streams and duration. |
| FR-VID-008 | MVP | The application **shall** save a local preview, explicit approval decision, and final approved render; it shall not treat failed validation as approval. |
| FR-VID-009 | MVP | The application **shall** retain intermediate assets needed for scene-level replacement and rerendering, subject to user-configured retention rules. |
| FR-VID-010 | Later | The application **may** provide automatic lip-sync correction and advanced character-consistency scoring when supported technology is chosen. |

### 3.8 Continuity, Persistence and Recovery

| ID | Priority | Requirement |
|---|---|---|
| FR-STATE-001 | MVP | The application **shall** store a separate story bible and chronological event log for each story. |
| FR-STATE-002 | MVP | Planning Episode N **shall** load approved earlier episodes' relevant continuity before preparing prompts. |
| FR-STATE-003 | MVP | Story state updates **shall** occur after episode approval; failed or rejected generations shall not silently alter canonical story facts. |
| FR-STATE-004 | MVP | A reused character's canonical identity **shall** remain shared but story-specific history/relationships **shall** remain isolated. |
| FR-STATE-005 | MVP | The application **shall** persist generation state transitions and enough metadata to recover or safely reconcile jobs after application restart. |
| FR-STATE-006 | MVP | Recovery **shall** avoid duplicate billable submissions whenever job state can be verified; ambiguous submissions must be flagged for operator review. |
| FR-STATE-007 | MVP | The application **shall** maintain an auditable association between source script, cast mapping, scene plan, provider jobs, resulting assets and final render. |

### 3.9 CLI Operations

| ID | Priority | Requirement |
|---|---|---|
| FR-CLI-001 | MVP | Provide character commands: `create`, `list`, `show`, `update`, `archive`, `export`, `import`, `register-provider`. |
| FR-CLI-002 | MVP | Provide story commands: `import`, `list`, `show`, and cast/episode inspection commands. |
| FR-CLI-003 | MVP | Provide episode commands: `plan`, `generate`, `status`, `preview`, `approve`, `render`. |
| FR-CLI-004 | MVP | Provide a `scene regenerate` operation targeting an explicit story, episode and scene ID. |
| FR-CLI-005 | MVP | Commands **shall** output a clear outcome, output path/ID and actionable errors and support `--help`. |
| FR-CLI-006 | MVP | Paid actions **shall** distinguish planning/preflight from submission; dry-run must not send billable work. |

## 4. Business Rules

| ID | Rule |
|---|---|
| BR-001 | The **local character ID**, not a remote Flow name or provider ID, is the permanent identity key. |
| BR-002 | Remote names/aliases are descriptive mappings; they are not guaranteed portable between Flow projects or API products. |
| BR-003 | A character can be cast in multiple unrelated stories; story-specific facts may not leak between them. |
| BR-004 | Character traits require explicit versioned edits; scene changes do not change permanent traits. |
| BR-005 | Unknown or ambiguous names/IDs block generation until resolved or approved. |
| BR-006 | Approval is required before paid generation in the initial workflow and before a video becomes final. |
| BR-007 | User-provided storyline remains authoritative; LLM additions/segmentation require review if material. |
| BR-008 | Failed or unapproved episodes do not update the official story bible. |
| BR-009 | Successful scenes should be reused on retries unless explicitly replaced. |
| BR-010 | No credential, private token, database file, paid source media or generated full video is committed to Git by default. |
| BR-011 | Use only officially supported programmatic provider APIs; unsupported Flow functionality uses explicit manual steps. |
| BR-012 | The MVP produces local files and never autonomously posts to public social networks. |
| BR-013 | User must have rights/consent to use relevant images, voices, music and any reference materials. |

## 5. Non-Functional Requirements

**Note:** Some numerical defaults are *proposed* and must be confirmed in HLD/LLD; they are not claims about AI provider performance.

| ID | Area | Requirement / measurable verification |
|---|---|---|
| NFR-001 | Deployment | Clean documented local setup works on one supported desktop OS, with further OS support documented rather than assumed. |
| NFR-002 | Local-first | No AWS account, remote database or cloud workload is required to run, manage characters/stories or render from local clips. |
| NFR-003 | Data durability | SQLite commits and media writes are recoverable; interrupted operations do not leave silent successful-but-corrupt records. |
| NFR-004 | Resume | After simulated restart, a persisted generation job can be inspected and reconciled without restarting every completed scene. |
| NFR-005 | Security | Credentials are read from approved non-repository sources; logs redact secret values; example config contains placeholders only. |
| NFR-006 | Provider isolation | Core modules can be tested with fake LLM/video/image/voice providers; adapters do not leak vendor-specific contracts into story logic. |
| NFR-007 | Cost safety | Dry-run test demonstrates **zero paid-generation API submissions**; any estimate is labeled approximate and incomplete when pricing unknown. |
| NFR-008 | Testability | Unit/integration tests run without network access or paid APIs; paid end-to-end calls are explicitly opt-in. |
| NFR-009 | Observability | Logs include operation, story/episode/scene/job IDs, lifecycle states and errors without credential leakage. |
| NFR-010 | Validation | Invalid input files, unresolved casts, unsupported model capability and missing references return actionable errors. |
| NFR-011 | Output | Final renderer verifies 9:16, MP4, configured resolution, audio/video stream presence and expected duration tolerance. |
| NFR-012 | Accessibility | CLI includes usage/help and readable status/error messages; subtitles are editable before final render. |
| NFR-013 | Maintainability | Type-annotated modules with explicit boundaries and documented tests, schemas and configuration. |
| NFR-014 | Extensibility | A second compatible provider can be introduced without changing the character/story persistence schemas solely to accommodate it. |
| NFR-015 | Portability | Local asset paths in saved records are managed portably (relative paths where practical); archive/export strategy is documented. |
| NFR-016 | Media integrity | Managed files can be checked with size/checksum and validated on access; incomplete downloads are never treated as final assets. |
| NFR-017 | Recovery safety | Uncertain remote generation jobs are surfaced for review instead of blindly resubmitted and potentially billed again. |
| NFR-018 | Privacy | User-controlled local storage is default; only explicitly selected provider requests may send script or character assets externally. |
| NFR-019 | Quality | Human approval remains an explicit gate because photorealism, character identity fidelity and lip-sync accuracy cannot be guaranteed by specification alone. |
| NFR-020 | Performance | No fixed total video-generation SLA is promised because providers and hardware vary; local validation/assembly times should be measured during benchmark testing. |

## 6. External Interfaces and Input/Output Contracts

### 6.1 CLI (illustrative, exact syntax finalized in LLD)

```bash
python main.py character create --name Amara --spec ./characters/amara.yaml
python main.py character list
python main.py character show --id CHAR-0001
python main.py character register-provider --id CHAR-0001 --provider google_flow --name Amara
python main.py character export --format json

python main.py story import --file ./stories/the_hidden_truth.pdf
python main.py story list
python main.py episode plan --story STORY-0001 --episode 01 --dry-run
python main.py episode generate --story STORY-0001 --episode 01
python main.py episode status --story STORY-0001 --episode 01
python main.py episode preview --story STORY-0001 --episode 01
python main.py episode approve --story STORY-0001 --episode 01
python main.py episode render --story STORY-0001 --episode 01
python main.py scene regenerate --story STORY-0001 --episode 01 --scene SCENE-004
```

### 6.2 Character Specification (example, not yet final schema)

```yaml
character_name: Amara
aliases: [Ama]
age_range: [30, 35]
sex: female
skin_tone: dark brown
hair_style: shoulder-length braids
eye_color: brown
clothing_style: contemporary smart casual
languages: [English, Yoruba]
accent: Nigerian English
voice: calm and expressive
reference_images:
  front: data/characters/CHAR-0001/front.png
  three_quarter: data/characters/CHAR-0001/three_quarter.png
  full_body: data/characters/CHAR-0001/full_body.png
```

### 6.3 Story Cast Mapping (example)

```yaml
story_id: STORY-0002
title: The Last Promise
cast:
  - story_role_name: Amara
    character_id: CHAR-0001
  - story_role_name: Mama Grace
    character_id: CHAR-0003
episodes:
  expected_count: 8
  target_duration_seconds: 120
```

### 6.4 Provider Contracts

Video provider must support well-defined operations for **submit, inspect status, download output and optional cancellation**. Capabilities (reference images, duration, format, native sound, supported regions/models, authentication and quota) must be probed or configured based on official docs. An unsupported feature must generate explicit preflight feedback; do not silently discard important character references.

### 6.5 Local Output Contract

Proposed output layout:

```text
data/
  characters/CHAR-0001/...
  stories/STORY-0001/
    originals/
    story_bible.json
    episodes/EP-0001/
      source/
      plan/
      scenes/SCENE-001/
        requests/
        clips/
        audio/
      preview/
      final/
        STORY-0001_EP-0001.mp4
output/
  final/  # optional convenience exports
```

All paths are illustrative and should be finalized in LLD. Generated media and personal/licensed story assets are excluded from Git by default.

## 7. Acceptance Test Scenarios

Each scenario has observable **Given / When / Then** behavior. Exact duration tolerance and quality thresholds remain open decisions rather than invented guarantees.

### AT-01 — Create and persist a reusable character
**Given** a valid YAML character specification and reference assets, **when** the operator creates the character, **then** the registry returns a unique `CHAR-...` ID, stores the assets/metadata, and the character remains available after restart.  
**Covers:** FR-CHR-001–009, FR-LOC-003.

### AT-02 — Register a Flow alias without assuming unsupported API
**Given** `CHAR-0001` exists, **when** the operator registers a Flow display name and optional alias, **then** the local mapping is saved/displayed and no undocumented Flow API request is made.  
**Covers:** FR-FLOW-001–007.

### AT-03 — Import story and resolve cast
**Given** a readable PDF/TXT story with 8 episodes and references to known character IDs, **when** it is imported and cast assignments validated, **then** the original is retained and all referenced characters resolve; an unknown or ambiguous name is reported.  
**Covers:** FR-STORY-001–009.

### AT-04 — Reject unsafe or ambiguous automatic segmentation
**Given** a story without episode boundaries, **when** parsing proposes episode divisions, **then** the operator sees and approves the proposed divisions before generation.  
**Covers:** FR-STORY-004, FR-STORY-005.

### AT-05 — No-charge preflight
**Given** an approved episode script and cast, **when** the operator runs dry-run planning, **then** the system outputs scenes, prompt/reference linkage, estimated provider usage and blocking issues and submits **zero** billable generation calls.  
**Covers:** FR-LOC-007, FR-PLAN-001–007, FR-AI-009–010.

### AT-06 — Generate one episode using registered identities
**Given** an approved story episode, characters and a working selected provider, **when** the operator explicitly generates the episode, **then** outputs/IDs for generated shots are persisted and each relevant request uses the resolved character's profile/references within model capability limits.  
**Covers:** FR-PLAN-004–008, FR-AI-001–008.

### AT-07 — Regenerate one scene
**Given** an episode with several successful shots and one rejected scene, **when** the operator regenerates that scene, **then** unaffected shots retain their saved outputs, while only the requested scene gets a new generation version.  
**Covers:** FR-PLAN-008, FR-AI-007, FR-VID-009.

### AT-08 — Restart and recover
**Given** persistent jobs including completed, in-progress and uncertain states, **when** the process is stopped/restarted, **then** completed work stays intact, known remote jobs are reconciled, and uncertain submissions are flagged rather than automatically charged again.  
**Covers:** FR-STATE-005–007, NFR-004, NFR-017.

### AT-09 — Render and validate local 120-second target
**Given** valid approved scene/media inputs, **when** the episode is rendered, **then** a local MP4 file is generated with configured 9:16 dimensions and AAC audio and validated using FFprobe; actual duration is compared with the configured target/tolerance.  
**Covers:** FR-VID-001–009.

### AT-10 — Reuse character across unrelated stories
**Given** `CHAR-0001` appears in stories A and B, **when** an episode of story B is prepared, **then** the same canonical character identity is used but story A's plot/relationships are not imported into B.  
**Covers:** FR-CHR-004, FR-STORY-009, FR-STATE-001–004.

### AT-11 — Offline automated test suite
**Given** mock providers, **when** pytest runs in a network-restricted environment, **then** the suite tests the registry, story processing, prompt planning, job recovery and renderer without calling live paid APIs.  
**Covers:** NFR-006–008.

### AT-12 — Security and nonpublishing boundary
**Given** valid project credentials and locally generated MP4s, **when** normal tests/workflows run, **then** credentials are excluded from logs/source control and no YouTube/Instagram publish call is made.  
**Covers:** NFR-005, BR-010, BR-012.

## 8. Requirement Traceability and Release Gates

| Release / milestone | Core requirement families | Exit evidence |
|---|---|---|
| M1 — Local foundation and registry | FR-LOC, FR-CHR, FR-FLOW | AT-01, AT-02, unit tests, character JSON export |
| M2 — Story and continuity | FR-STORY, FR-STATE | AT-03, AT-04, AT-10, imported story fixture |
| M3 — Scene planning and AI provider | FR-PLAN, FR-AI | AT-05, AT-06, mock + opt-in live provider test |
| M4 — Rendering and recovery | FR-VID, FR-STATE | AT-07, AT-08, AT-09, validated MP4 |
| M5 — Integrated CLI and hardening | FR-CLI, NFR | AT-11, AT-12, end-to-end operator guide |

All requirement IDs must remain stable across HLD, LLD, tests and implementation tickets. When a requirement is revised or dropped, document the decision and commit rather than silently renumbering or deleting it.

## 9. Assumptions and Dependencies

| ID | Assumption / dependency | Implication |
|---|---|---|
| AS-001 | The user operates a computer capable of running Python and FFmpeg. | Exact supported OS and minimum specs must be validated. |
| AS-002 | Initial scripts are accessible PDF/TXT. | Scanned PDFs may need optional OCR. |
| AS-003 | At least one video API account is available and permitted to generate the requested imagery. | The app must handle missing access/quotas cleanly. |
| AS-004 | Video inference may remain cloud-hosted even though the application runs locally. | API charges and data transmission are disclosed. |
| AS-005 | Google Flow's internal characters/project assets are not guaranteed accessible from an external supported API. | Manual mapping/import is the MVP baseline. |
| AS-006 | Character consistency is probabilistic, not guaranteed. | Reference-image use and human quality approval remain necessary. |
| AS-007 | The creator has the rights to requested reference imagery, voices and music. | Permissions/disclosure policy is enforced operationally. |
| AS-008 | Google Veo is preferred but may vary by model, region, quotas, billing and API features. | The supported API, request schema and costs must be checked before implementation. |
| AS-009 | The project is single-user and locally initiated. | Authentication/RBAC and cloud schedulers are not in MVP. |

## 10. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Unsupported Google Flow automation | Direct character synchronization may be infeasible | Keep local registry authoritative; use manual mappings and supported Veo APIs |
| Character appearance drift | Episodes may look inconsistent | Canonical image references, preview approval, scene regeneration, future automated checks |
| Provider output duration limits | One request cannot generate entire episode | Scene/shot planning and FFmpeg assembly |
| Paid API overuse | Unexpected cost | Dry-run, estimate/approval gates, concurrency limits, scene-level retries |
| Inconsistent audio/lips | Reduced realism | Configurable voice pipeline, preview/review, optional later lip-sync |
| Lost or duplicated remote jobs | Lost progress or duplicated charges | Persistent IDs, reconciliation and explicit review of uncertain states |
| Story continuity leakage | Incorrect plots between unrelated series | Separate story bibles and story-specific casting |
| Missing assets or corrupted files | Broken render | Paths/checksums, FFprobe, repairable job state |
| Unauthorized likeness/audio use | Privacy and IP harm | Use only owned/licensed or consented assets, disclosure as required |
| Provider API changes | Generation failures | Adapter boundaries, capability checks and pinned integration tests |

## 11. Open Decisions for Approval

The following must not be silently guessed by an implementing LLM.

| ID | Decision | Proposed default / status |
|---|---|---|
| OD-001 | Primary desktop platform | **Open:** macOS, Windows or Linux; test one first |
| OD-002 | Initial video API authentication and billing account | **Open:** must verify supported Google Veo access |
| OD-003 | Budget per 120-second episode | **Open:** enforce approval even if no numerical cap is provided |
| OD-004 | Character reference image-generation provider | **Open:** accept manual imports first |
| OD-005 | Voice and dialogue model | **Open:** provider-neutral with imported audio fallback |
| OD-006 | Episode duration tolerance | **Open:** set explicit seconds in LLD |
| OD-007 | Initial shot length strategy | **Open:** derive from supported provider model at implementation |
| OD-008 | Approval gates | **Proposed:** approve cast/plan, paid generation, and final video |
| OD-009 | Character visual quality threshold | **Open:** human approval in MVP; measured scoring later |
| OD-010 | Cross-story canonical character voice strategy | **Proposed:** share canonical voice profile; allow story-scoped interpretation |
| OD-011 | How much source narrative rewriting is permitted | **Proposed:** only minimally and with approval for material changes |
| OD-012 | Local UI | **Deferred:** CLI first |
| OD-013 | Scheduled posting and social APIs | **Deferred:** after reliable local generation |
| OD-014 | Local offline video-generation hardware/model | **Deferred:** assess GPUs, VRAM and licenses separately |
| OD-015 | Data retention/archive policy for large media | **Open:** protect approved final videos and canonical character references by default |

## 12. Documentation and Change Control

- **Baseline:** [PROJECT_MASTER_SPECIFICATION.md](./PROJECT_MASTER_SPECIFICATION.md) (v0.1).
- **This SRS:** requirement identifiers and acceptance scenarios; status is **draft for review**.
- **Next:** `HLD.md` shall map major components and flows to requirement IDs.
- **Then:** `LLD.md` shall define concrete schemas, modules, types, state machines, data integrity and CLI semantics.
- **Then:** `IMPLEMENTATION_PLAN.md` shall break implementation into reviewed milestones and tests.
- Store architecture choices in `docs/decisions/` with an explicit decision/rationale/consequences format.
- Record material requirement changes as Git commits; preserve baseline and avoid silently changing approved scope.

## 13. Formal MVP Acceptance Checklist

- [ ] Local installation documented and verified on the selected OS.
- [ ] Character specifications create stable IDs and persistent records.
- [ ] Reference images and optional voice assets are stored and retrievable.
- [ ] Flow display names/aliases can be manually mapped without fake API dependencies.
- [ ] Character list/show/search/export/import work.
- [ ] PDF/TXT import preserves source and extracts usable text.
- [ ] Episode segmentation and ambiguous cases are reviewable.
- [ ] Character names/IDs/tags resolve or fail safely.
- [ ] Two unrelated stories can reuse the same identity with isolated narrative states.
- [ ] Plan/dry-run produces a scene/shot breakdown with zero billable API requests.
- [ ] At least one documented video provider can produce a usable clip.
- [ ] Scene-level retry avoids regeneration of unaffected work.
- [ ] Interrupted jobs can be recovered or explicitly reconciled.
- [ ] Audio/subtitles can be included and edited for final rendering.
- [ ] Local FFmpeg produces a validated vertical MP4 near the 120-second configured target.
- [ ] The episode requires explicit final review/approval.
- [ ] Normal tests run without live paid API calls.
- [ ] No application cloud infrastructure is required.
- [ ] No credentials, generated large media, or private project data are committed by default.

**Approval status:** Pending creator review. Do not interpret this Git commit as approval of open decisions or readiness to implement.
