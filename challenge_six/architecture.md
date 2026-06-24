# ReadyNow! Architecture (Challenge 6)

ReadyNow! is the FEMA emergency-preparedness assistant required by Challenge 6
of the workshop. It is built as a deterministic `SequentialAgent` pipeline that
coordinates four specialist agents and applies a validate/refine workflow on
every response.

## High-level diagram

> The diagram below is an **agent hierarchy tree**: it shows the parent/child
> relationships rather than the data flow. The 4 children of `readynow_pipeline`
> are executed sequentially (1 -> 2 -> 3 -> 4), and the 4 children under
> `readynow_coordinator` are AgentTools that can be invoked by the LLM. Solid
> edges are containment, dashed edges are tool invocation.

```mermaid
flowchart TD
    Root["readynow_pipeline<br/>(SequentialAgent)<br/>top-level orchestrator"]

    G["1 - greeter_agent<br/>(LlmAgent)"]
    C["2 - readynow_coordinator<br/>(LlmAgent)"]
    K["3 - critique_agent<br/>(LlmAgent)"]
    R["4 - refine_agent<br/>(LlmAgent)"]

    Root --> G
    Root --> C
    Root --> K
    Root --> R

    W["weather_specialist<br/>(LlmAgent)"]
    S["search_specialist<br/>(LlmAgent)"]
    Rt["routes_specialist<br/>(LlmAgent)"]
    Q["qa_specialist<br/>(LlmAgent)"]

    C -. AgentTool .-> W
    C -. AgentTool .-> S
    C -. AgentTool .-> Rt
    C -. AgentTool .-> Q

    Wt["get_lat_lon<br/>get_extended_weather_forecast"]
    St["GoogleSearchTool()"]
    Rtt["get_directions"]

    W --> Wt
    S --> St
    Rt --> Rtt

    CB1["before_model_callback<br/>(chained: moderate<br/>-> validate_us_location<br/>-> log_user_prompt)"]
    CB2["after_model_callback<br/>(log_model_response)"]

    C -. callback .-> CB1
    C -. callback .-> CB2

    classDef root fill:#0d47a1,color:#fff,stroke:#0d47a1,font-weight:bold;
    classDef stage fill:#e3f2fd,stroke:#1565c0,color:#0d47a1;
    classDef specialist fill:#f3e5f5,stroke:#6a1b9a,color:#4a148c;
    classDef tool fill:#fff,stroke:#9e9e9e,color:#424242,stroke-dasharray: 3 3;
    classDef callback fill:#fff8e1,stroke:#ef6c00,color:#e65100;

    class Root root;
    class G,C,K,R stage;
    class W,S,Rt,Q specialist;
    class Wt,St,Rtt tool;
    class CB1,CB2 callback;
```

**Reading the tree**:

- **Solid arrows** = containment (parent -> child sub_agent or agent -> tool)
- **Dashed arrows** = the parent can call the child as an `AgentTool` or has
  it as a callback attached
- The 4 children of `readynow_pipeline` execute in numbered order (1 to 4)
  because the root is a `SequentialAgent`. The 4 children under
  `readynow_coordinator` are invoked on demand by the LLM as tools

## Components

### Top-level pipeline

`readynow_pipeline` is a `SequentialAgent` that runs four stages in order:

1. **greeter_agent** - introduces the assistant and its capabilities
2. **readynow_coordinator** - dispatches the query to the right specialist
3. **critique_agent** - reviews the coordinator's draft answer
4. **refine_agent** - rewrites the answer using the critique

State is propagated between stages through `output_key`:

- `greeter_agent.output_key = "greeting_message"`
- `readynow_coordinator.output_key = "initial_draft"`
- `critique_agent.output_key = "critique_suggestions"`
- `refine_agent.output_key = "final_response"`

The `critique_agent` and `refine_agent` instructions reference `{initial_draft}`
and `{critique_suggestions}` to inject the prior state.

### Specialists (exposed to the coordinator as `AgentTool`s)

| Specialist | Tools | Responsibility |
|---|---|---|
| `weather_specialist` | `get_lat_lon`, `get_extended_weather_forecast` | US weather forecasts (NWS API) |
| `search_specialist` | `GoogleSearchTool()` | Live web search for news / FEMA alerts |
| `routes_specialist` | `get_directions` | Evacuation / travel routes (Google Maps Directions) |
| `qa_specialist` | (none) | General preparedness Q&A |

Using `AgentTool` rather than `sub_agents` keeps the coordinator's output a
coherent textual draft (instead of transferring conversation control), which
is what the downstream critique / refine agents need.

### Cross-cutting callbacks

Attached to the `readynow_coordinator`:

- **before_model_callback** = `chained_before_callback` which runs, in order:
  1. `moderate_user_prompt` - blocks prompt injection / jailbreak attempts
  2. `validate_us_location` - rejects non-US queries (NWS API is US-only)
  3. `log_user_prompt` - records the incoming user message
- **after_model_callback** = `log_model_response` - records the outgoing
  response.

Returning a non-`None` `LlmResponse` from any of the validators short-circuits
the rest of the pipeline and returns the validation error directly to the
user.

### Deployment topology

```mermaid
flowchart LR
    Notebook[challenge_six.ipynb<br/>Colab Enterprise]
    AdkApp[reasoning_engines.AdkApp]
    Engine[(Vertex AI Agent Engine<br/>projects/.../reasoningEngines/...)]
    Bucket[(GCS Staging Bucket<br/>agent_engine.pkl + deps)]

    Notebook -- "agent_engines.create()" --> Engine
    Notebook -. "uploads via" .-> Bucket
    Notebook -- "remote_agent.stream_query()" --> Engine
    AdkApp -- "wraps readynow_pipeline" --> Engine
```

Local test path:

```
User -> AdkApp.async_stream_query() -> readynow_pipeline -> response
```

Cloud test path:

```
User -> remote_agent.stream_query() -> Vertex AI Agent Engine -> readynow_pipeline -> response
```

## Mapping to FEMA stakeholder requirements

| PDF requirement (slide 72) | Implementation |
|---|---|
| Real-time weather and news alerts | `weather_specialist` + `search_specialist` |
| In the event of a disaster, provide suggested routes to safety | `routes_specialist` (Google Maps Directions) |
| Log all interactions between the user and the agent | `log_user_prompt` + `log_model_response` |
| Validate that user input is appropriate and refuse off-mission requests | `moderate_user_prompt` + coordinator strict-scope instructions |
| Ensure agent responses are valid, well-written, and easy to understand | `critique_agent` -> `refine_agent` sequential workflow |
| Deploy the agent to Agent Platform | `vertexai.agent_engines.create(app_readynow, ...)` |
| Test code that demonstrates the agent's functionality | 6 local test scenarios + 2 remote (cloud) test scenarios |
