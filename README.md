# Elyt

AI browser automation across profiles.

[Website](https://elyt-ai.com/) | [Watch demo](https://elyt-ai.com/Elyt/videos/video.mp4) | [Product overview](docs/PRODUCT_OVERVIEW.md) | [Technical notes](docs/TECHNICAL_ARCHITECTURE.md)

Elyt is a private commercial platform for building AI-powered browser workflows, running them across many isolated browser profiles, scheduling recurring work, and monitoring execution from one place.

This repository is the public showcase for Elyt. It contains screenshots, demo media, architecture notes, and public-safe product documentation. The production source code remains private.

## Demo

<p align="center">
  <a href="https://elyt-ai.com/Elyt/videos/video.mp4">
    <img src="media/elyt-demo-poster.webp" alt="Elyt demo video poster" width="820">
  </a>
</p>

<p align="center">
  <a href="https://elyt-ai.com/Elyt/videos/video.mp4">Watch MP4</a>
  |
  <a href="https://elyt-ai.com/Elyt/videos/video.webm">Watch WebM</a>
  |
  <a href="media/elyt-demo.mp4">Repository copy</a>
</p>

## What Elyt Does

| Capability | What it means |
| --- | --- |
| Plain-English workflows | Turn natural-language instructions into reusable browser automation flows. |
| Profile orchestration | Run the same workflow across many isolated browser profiles instead of one local browser. |
| Visual workflow system | Chain nodes, branch logic, pass data between steps, and schedule repeatable work. |
| Execution monitoring | Track profile-level status, logs, screenshots, transcripts, generated files, and history. |
| Model flexibility | Route tasks through OpenAI, Anthropic, Google Gemini, Groq, Ollama, or local models. |
| Web and desktop modes | Use the web platform or run local workflows through a Tauri desktop shell. |

## Screenshots

| Workflow builder | Profile orchestration | Execution monitoring |
| --- | --- | --- |
| <img src="media/posters/workflow-builder.webp" alt="Elyt workflow builder preview" width="270"> | <img src="media/posters/profile-orchestration.webp" alt="Elyt profile orchestration preview" width="270"> | <img src="media/posters/execution-monitoring.webp" alt="Elyt execution monitoring preview" width="270"> |

<details>
<summary>Full-page marketing screenshot</summary>

<p align="center">
  <img src="media/elyt-full-page.png" alt="Elyt marketing site full-page screenshot" width="760">
</p>

</details>

## Architecture

Elyt is split into a control plane and an execution plane. The Node.js side owns product state, workflow definitions, scheduling, auth, and queue orchestration. The Python side owns browser execution, AI action planning, runtime observation, and execution artifacts.

```mermaid
flowchart TD
    subgraph Client["Client surfaces"]
        Web["Web app"]
        Desktop["Tauri desktop shell"]
    end

    subgraph Control["Control plane"]
        Api["Node.js Express API"]
        Auth["Auth and session services"]
        Scheduler["Schedule service"]
        Queue["Redis + BullMQ queue"]
        DB["LowDB-backed app state"]
    end

    subgraph Execution["Execution plane"]
        LocalApi["Python FastAPI engine"]
        Runner["Workflow runner"]
        BrowserRuntime["Browser automation runtime"]
        Artifacts["Screenshots, logs, transcripts, files"]
    end

    subgraph Integrations["Integration layer"]
        Profiles["Browser profile provider adapters"]
        Models["AI model provider adapters"]
        MCP["MCP tools and custom capabilities"]
    end

    Web --> Api
    Desktop --> Web
    Desktop --> LocalApi
    Api --> Auth
    Api --> Scheduler
    Api --> Queue
    Api --> DB
    Queue --> LocalApi
    LocalApi --> Runner
    Runner --> BrowserRuntime
    Runner --> Artifacts
    BrowserRuntime --> Profiles
    Runner --> Models
    Runner --> MCP
```

## Execution Flow

Each workflow run can fan out into profile-scoped jobs. That gives Elyt per-profile status, retries, artifact capture, and clean execution history instead of one opaque global run.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant UI as Web UI / Desktop
    participant API as Express API
    participant Queue as Redis Queue
    participant Engine as Python Automation Engine
    participant Model as AI Provider
    participant Browser as Browser Profile
    participant Store as Execution Artifacts

    User->>UI: Create or run workflow
    UI->>API: Save workflow and selected profiles
    API->>Queue: Enqueue profile-scoped jobs
    Queue->>Engine: Dispatch execution job
    Engine->>Model: Plan next browser action
    Engine->>Browser: Execute action
    Browser-->>Engine: DOM state, screenshot, result
    Engine->>Store: Save logs, screenshots, files
    Engine-->>API: Status update
    API-->>UI: Live progress and history
```

## Workflow Model

Workflows are built from reusable actions, AI nodes, profile selections, schedules, and artifact bundles. The useful part is not just running one prompt. It is composing repeatable automation that can be inspected, scheduled, and rerun.

```mermaid
flowchart LR
    Workflow["Workflow"]
    Prompt["Plain-English task"]
    NodeA["AI node"]
    NodeB["Action node"]
    Branch["Condition"]
    Retry["Retry / recovery"]
    Schedule["Schedule"]
    ProfileGroup["Profile group"]
    Profile["Profile execution"]
    Artifact["Artifact bundle"]

    Prompt --> Workflow
    Workflow --> NodeA
    Workflow --> NodeB
    NodeA --> Branch
    Branch --> NodeB
    Branch --> Retry
    Schedule --> Workflow
    Workflow --> ProfileGroup
    ProfileGroup --> Profile
    Profile --> Artifact
```

## Deployment Modes

Elyt supports both web and local execution paths. The desktop app wraps the same workflow concepts while adding a local Python sidecar for users who need work to run from their own machine.

```mermaid
flowchart TD
    subgraph WebMode["Web mode"]
        HostedUI["Hosted or local web UI"]
        HostedApi["Express API"]
        QueueA["Queue and scheduler"]
    end

    subgraph DesktopMode["Desktop mode"]
        Tauri["Tauri shell"]
        EmbeddedUI["Embedded web UI"]
        Sidecar["Local Python sidecar"]
    end

    subgraph Shared["Shared execution concepts"]
        Workflows["Workflow definitions"]
        Profiles["Profile selections"]
        History["Execution history"]
        Artifacts["Logs and screenshots"]
    end

    HostedUI --> HostedApi
    HostedApi --> QueueA
    QueueA --> Shared
    Tauri --> EmbeddedUI
    Tauri --> Sidecar
    Sidecar --> Shared
```

## What Is Technically Interesting

### Split control plane and execution plane

The product API and the automation runtime are separate. This keeps workflow state, scheduling, auth, and product APIs stable while the browser automation engine can evolve independently.

### Profile-scoped jobs

A workflow can be executed per profile. This makes status reporting, retry behavior, staggered runs, and artifact storage much easier to reason about.

### Queue-backed scheduling

Batch launches and recurring schedules cross a queue boundary. The UI stays responsive, long-running work is inspectable, and the system has a natural place to persist execution state.

### Adapter-based integrations

AI providers and browser profile providers sit behind adapter boundaries. Workflow definitions do not need to care whether a run uses OpenAI, Anthropic, Gemini, Groq, Ollama, or a local model.

### Artifact-first debugging

Runs create screenshots, logs, transcripts, generated files, and final statuses. Operators can inspect what happened at the node and profile level instead of guessing from a single success/failure flag.

### Desktop sidecar model

The desktop build uses Tauri as a shell and runs local services where needed. That gives the product a local execution path without forking the whole user experience.

### Contract-driven boundaries

Important request and response shapes are validated at service boundaries. That matters because workflow definitions and execution state move through web, Node.js, Python, queue, and desktop runtimes.

## Technology Map

| Layer | Main technologies |
| --- | --- |
| Web UI | Vanilla JavaScript, modular frontend services, CSS/SCSS |
| Product API | Node.js, Express, JWT/session services, AJV validation |
| Queueing | Redis, BullMQ |
| Automation engine | Python, FastAPI, Pydantic, async services |
| Browser runtime | Playwright and browser automation framework integrations |
| Desktop | Tauri, Rust shell, embedded local services |
| AI providers | OpenAI, Anthropic, Google Gemini, Groq, Ollama, local models |
| Artifacts | Screenshots, logs, transcripts, generated files |

## Public Repo Scope

Included:

- Product overview and README-first technical explanation.
- Screenshots, posters, and demo video links.
- High-level architecture diagrams.
- Responsible-use positioning.

Not included:

- Private application source code.
- API keys, form keys, customer data, logs, databases, cookies, or environment files.
- Provider internals, private deployment scripts, or operational details that should stay private.

## Use Cases

- Data collection from authorized sources.
- E-commerce monitoring and product research.
- QA and repeatable browser workflow testing.
- Internal operations that require repeatable browser tasks.
- Managed multi-account workflows where the operator owns or is authorized to control the accounts.

## Links

- Website: [elyt-ai.com](https://elyt-ai.com/)
- Founder: [ArshiaHemati.com](https://arshiahemati.com/)
- GitHub organization: [AHGesports](https://github.com/AHGesports)

## Status

Elyt is live as a private commercial product. Access is handled through demos and direct customer onboarding.
