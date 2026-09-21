# RTaC
Red Team as Code

   ## Documentation
   - [Architecture](docs/architecture.md) — high- and low-level design, trust boundaries
   - [Workflows & Data Flow](docs/workflows.md) — pipeline, decision logic, data flows

 ## Architecture
 ```mermaid
flowchart TB
    subgraph HOST[Windows 11 Host]
        direction TB
        subgraph WSL[WSL2 - Ubuntu 24.04]
            subgraph K3S[k3s + NetworkPolicy CNI]
                CP[Control Plane<br/>OpenClaw · LiteLLM · vLLM]
                EP[Execution Plane<br/>OpenBot · Target]
                EV[Evaluation<br/>Purple Llama · Results]
            end
        end
    end
    CLOUD[Cloud Burst GPU<br/>RunPod / Together AI]
    GHUB[GitHub<br/>repos + reports]

    GHUB <--> CP
    CP <--> CLOUD
    CP -->|controlled| EP
    EP --> EV
    EV --> GHUB

    style CP fill:#1a3a5c,color:#fff
    style EP fill:#5c1a1a,color:#fff
    style EV fill:#1a5c3a,color:#fff
```
