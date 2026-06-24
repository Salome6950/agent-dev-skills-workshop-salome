# Agent Development Skills Workshop - Salome

Submission for the **Agentic AI with the Google Agent Development Kit (ADK)** Skills Validation Workshop.

Each challenge is implemented as a standalone Jupyter notebook designed to run
on **Colab Enterprise** in the workshop's Skills Boost environment.

## Notebooks

| File | Challenge | Points | Description |
|---|---|---|---|
| `challenge_one.ipynb` | 1 - Real-Time Weather Alerts Agent | 15 | ADK agent with custom tools (NWS + Google Maps Geocoding), tested with both Gemini and Llama 3.3 via Groq (LiteLLM). |
| `challenge_two.ipynb` | 2 - Enhancing Agents with Callbacks | 15 | Adds `before_model_callback` chain (moderation + US-location validation + prompt logging) and `after_model_callback` for response logging. |
| `challenge_three.ipynb` | 3 - Developing Multi-Agent Systems | 15 | Root coordinator orchestrating a search sub-agent (as `AgentTool`) and the weather sub-agent (as `sub_agent`). |
| `challenge_four.ipynb` | 4 - Programming an Agent Workflow | 15 | Deterministic top-level `SequentialAgent` chaining Greeter -> [Search -> Critique -> Refine]. |
| `challenge_five.ipynb` | 5 - Bonus Deployment | 10 | Deploys the Challenge 4 agent to Vertex AI Agent Engine via `agent_engines.create()` and verifies it with a remote `stream_query`. |
| `challenge_six.ipynb` | 6 - FEMA `ReadyNow!` Case Study | 40 | Full multi-agent emergency-preparedness assistant: 4 specialists (weather / search / routes / Q&A) coordinated by an LlmAgent inside a `SequentialAgent` validate/refine pipeline, with callbacks for moderation, US-location validation, and audit logging. Deployed to Agent Platform. See [architecture.md](architecture.md) for the full diagram and component breakdown. |

## Environment

Tested in Colab Enterprise with:

- `google-cloud-aiplatform` (>= 1.158)
- `google-adk` (2.3.0) with the `extensions` extras
- `litellm` (>= 1.89)
- `requests`

The first two cells of every notebook install / upgrade these dependencies.

## Required environment variables

| Variable | Used by | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | All notebooks | Required for the native Gemini agents. |
| `GROQ_API_KEY` | Challenges 1-5 | Required for the LiteLLM-wrapped Groq / Llama 3.3 agent. Free tier: https://console.groq.com |
| `GOOGLE_MAPS_API_KEY` | All notebooks | Required for the `get_lat_lon` geocoding tool. |
| `GOOGLE_CLOUD_PROJECT` | Challenges 5-6 | GCP project id for Agent Engine deployment (falls back to a Qwiklabs value). |
| `GOOGLE_CLOUD_LOCATION` | Challenges 5-6 | Region for Agent Engine (defaults to `us-central1`). |
| `STAGING_BUCKET` | Challenges 5-6 | GCS bucket used by Agent Engine to upload the agent pickle + deps. |

Set them as Colab secrets (Tools -> Secrets) or export them in a shell cell
before running the notebooks.

## Running

1. Open the notebook in Colab Enterprise.
2. Run the install cells (cells 0 and 1).
3. Run all remaining cells top-to-bottom. Test cells at the end of each
   notebook print their own output trace.

## Architecture (Challenge 6)

The full system architecture for the FEMA `ReadyNow!` case study is documented
in [architecture.md](architecture.md). It contains:

- A Mermaid flowchart of the `readynow_pipeline` and its specialists
- A deployment-topology diagram
- A mapping from every FEMA stakeholder requirement (slide 72) to the
  notebook code that implements it

## Notes

- The weather data source is the U.S. National Weather Service (NWS) API,
  which only serves US territories. The validation callback in Challenges 2-6
  refuses non-US queries up front.
- `SequentialAgent` emits a `DeprecationWarning` in ADK 2.3.0 in favor of the
  newer `Workflow` API. The implementation still functions correctly.
