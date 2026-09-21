# Workflows & Data Flow

This document walks through how the platform actually runs: the end-to-end workflow of a red-team pipeline, the decision logic of a single run, and how data moves between components and trust zones. Read [Architecture](architecture.md) first for the static structure; this document is the moving picture.

## Red Teaming as Code — end-to-end workflow

A run begins with a commit or a scheduled trigger and ends with a security posture report published back to GitHub. OpenClaw plans the attack through the LiteLLM gateway (which decides local vs. cloud inference), executes payloads inside a gVisor sandbox against a defined target, and hands the results to Purple Llama for scoring. Results persist so posture can be tracked across runs, and a regression can feed straight back into the next planning cycle.

```mermaid
flowchart LR
    A[Developer / Scheduler] -->|commit or cron| B[GitHub Repo]
    B -->|webhook / poll| C[OpenClaw Agent]
    C -->|request attack plan| D[LiteLLM Gateway]
    D -->|light logic| E[Local vLLM]
    D -->|heavy gen| F[Cloud Burst GPU]
    E --> C
    F --> C
    C -->|deploy payload| G[OpenBot Sandbox<br/>gVisor]
    G -->|attack| H[Target: Juice Shop]
    H -->|responses| G
    G -->|attack data| I[Purple Llama<br/>CyberSecEval]
    I -->|score + report| J[(Results Store)]
    J -->|posture report| K[GitHub: report artifact / PR comment]
    I -->|regression?| C
```

## Decision logic of a single run

Zooming into one pipeline execution: each payload is routed by task weight, checked by Llama Guard before it runs, executed only inside a healthy sandbox, and scored. A sandbox breakout or error kills the pod and alerts rather than continuing. The loop repeats per payload until the plan is exhausted, then the posture report is generated and published.

```mermaid
flowchart TD
    Start([Pipeline triggered]) --> Plan{Task type?}
    Plan -->|light| Local[Route to local vLLM]
    Plan -->|heavy| Cloud[Route to cloud GPU]
    Local --> Gen[Generate payload]
    Cloud --> Gen
    Gen --> Guard{Llama Guard<br/>output check}
    Guard -->|blocked| Log1[Log + skip] --> Next
    Guard -->|allowed| Sandbox[Spawn gVisor sandbox]
    Sandbox --> Exec[Execute against target]
    Exec --> Ok{Sandbox healthy?}
    Ok -->|breakout / error| Kill[Kill pod + alert] --> Next
    Ok -->|clean| Eval[CyberSecEval scores result]
    Eval --> Store[(Persist result)]
    Store --> More{More payloads?}
    More -->|yes| Next[Next payload] --> Plan
    More -->|no| Report[Generate posture report]
    Report --> End([Publish to GitHub])
```

## Data Flow Diagram

This view drops the sequencing and shows *what data goes where*, grouped by trust zone. Note the direction of the sensitive flows: secrets flow only into the control plane; attack traffic is confined between the execution plane's sandbox and its target; only scored results — not raw payloads or credentials — leave for GitHub.

```mermaid
flowchart LR
    subgraph External
        GH[GitHub Repo]
        HF[Hugging Face Hub]
        CGPU[Cloud GPU Provider]
    end
    subgraph Control[Control Plane - trusted]
        OC[OpenClaw]
        LL[LiteLLM]
        VL[vLLM]
        SEC[(Secrets:<br/>PAT / HF_TOKEN)]
    end
    subgraph Execution[Execution Plane - untrusted]
        OB[OpenBot Sandbox]
        TG[Target App]
    end
    subgraph Eval[Evaluation]
        PL[Purple Llama]
        DB[(Results Store)]
    end

    GH -->|trigger| OC
    SEC -->|PAT| OC
    SEC -->|token| VL
    HF -->|model weights| VL
    OC -->|prompts| LL
    LL <-->|inference| VL
    LL <-->|burst| CGPU
    OC -->|payload| OB
    OB -->|attack traffic| TG
    OB -->|attack data| PL
    PL -->|scores| DB
    DB -->|report| GH
```

## The second track — educational games

The tutoring track reuses the same platform with the opposite trust posture: a student interacts through a chat front end, Llama Guard filters input and output to prevent prompt injection and unsafe content, and any code the student runs executes in an ephemeral gVisor sandbox — the same isolation the red-team payloads use, for the same reason.

```mermaid
flowchart LR
    S[Student] -->|Discord / WhatsApp| OC2[OpenClaw Tutor]
    OC2 -->|prompt| GD[Llama Guard<br/>input filter]
    GD -->|clean| LL2[LiteLLM]
    LL2 <-->|inference| VL2[Local vLLM]
    LL2 -->|response| GD2[Llama Guard<br/>output filter]
    GD2 -->|safe| OC2
    OC2 -->|run code request| OB2[OpenBot<br/>ephemeral gVisor workspace]
    OB2 -->|result| OC2
    OC2 -->|reply| S
```

Keeping the two tracks as separate deployments on the shared platform ensures a student session and an offensive payload never share a namespace, even though they run the same underlying components.
