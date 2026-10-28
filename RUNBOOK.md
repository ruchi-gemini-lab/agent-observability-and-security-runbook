# Step-by-Step Execution Runbook — Observability and Security for Enterprise Agents

> **Environment Executed:** Google Cloud Platform (Qwiklabs Challenge Environment)  
> **Project ID:** `qwiklabs-gcp-04-e58085393db4`  
> **Project Number:** `403748473564`  
> **Organization ID:** `616463121992`  
> **Location / Region:** `us-central1`  
> **Student Account:** `student-04-108bfe631762@qwiklabs.net`

---

## Task 1. Prepare Your Environment

### 1.1 Set Project, Location, and Organization Variables

```bash
export PROJECT_ID=qwiklabs-gcp-04-e58085393db4
export LOCATION=us-central1
gcloud config set project ${PROJECT_ID}
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
export ORGANIZATION_ID=$(gcloud projects get-ancestors ${PROJECT_ID} --format=json | jq -r '.[] | select(.type=="organization") | .id')
echo "Project: ${PROJECT_ID} (${PROJECT_NUMBER}), Org: ${ORGANIZATION_ID}, Location: ${LOCATION}"
```

**Verified Output:**
```text
Updated property [core/project].
Project: qwiklabs-gcp-04-e58085393db4 (403748473564), Org: 616463121992, Location: us-central1
```

### 1.2 Copy Pre-Staged Lab Scripts & Deploy Baseline Agent Fleet

Copy the prerequisite scripts from the project Cloud Storage bucket:

```bash
gcloud storage cp -r gs://qwiklabs-gcp-04-e58085393db4-bucket/apex_labs .
```

Run the prerequisite script (`bash apex_labs/lab2_prereqs.sh`) to enable core deployment APIs (`aiplatform.googleapis.com`, `run.googleapis.com`, `cloudbuild.googleapis.com`, `artifactregistry.googleapis.com`, `iam.googleapis.com`, `iamcredentials.googleapis.com`, `logging.googleapis.com`, `cloudtrace.googleapis.com`), provision the Vertex AI Reasoning Engine service agent (`service-403748473564@gcp-sa-aiplatform-re.iam.gserviceaccount.com`), and deploy the two baseline agents:
- **`github-agent`** to **Vertex AI Agent Runtime** (`us-central1`)
- **`salesforce-agent`** to **Cloud Run** (`us-central1`)

Copy the agent source code and VS Code settings to your workspace:

```bash
gcloud storage cp -r gs://${PROJECT_ID}-bucket/agents .
mkdir -p ~/.vscode
gcloud storage cp -r gs://qwiklabs-gcp-04-e58085393db4-bucket/settings.json ~/.vscode/
```

### 1.3 Install and Upgrade `agents-cli`

```bash
export PATH="${HOME}/.local/bin:${PATH}"
uvx google-agents-cli setup
uv tool upgrade google-agents-cli
agents-cli info
```

**Verified Output:**
```text
CLI version:        1.12.0
CLI install path:   /home/student_04_108bfe631762/.local/share/uv/tools/google-agents-cli/lib/python3.12/site-packages/google/agents/cli
OS info:            Linux-6.18.55+-x86_64-with-glibc2.39
Telemetry:          enabled (opt out with AGENTS_CLI_TELEMETRY=0 or DO_NOT_TRACK=1)
Installed skills:   7 (project)
  - google-agents-cli-adk-code
  - google-agents-cli-deploy
  - google-agents-cli-eval
  - google-agents-cli-observability
  - google-agents-cli-publish
  - google-agents-cli-scaffold
  - google-agents-cli-workflow
```

### 1.4 Confirm the Fleet is Present

1. **List Cloud Run services (`salesforce-agent`):**

```bash
gcloud run services list --project=${PROJECT_ID} --region=${LOCATION} --format="table(metadata.name)"
```

**Verified Output:**
```text
NAME
salesforce-agent
```

2. **List agents running on Vertex AI Agent Runtime (`github-agent`):**

```bash
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://${LOCATION}-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/reasoningEngines" \
  | jq -r '(.reasoningEngines // [])[] | "\(.displayName)\t\(.name)"'
```

**Verified Output:**
```text
github-agent	projects/403748473564/locations/us-central1/reasoningEngines/146985188302258176
```

### 1.5 Enable the Observability and Discovery APIs

Enable Cloud Observability, Cloud Trace, Telemetry OTLP API, App Hub, App Topology, Agent Registry, Cloud Asset Inventory, Cloud Monitoring, and Cloud Logging:

```bash
gcloud services enable \
  observability.googleapis.com \
  cloudtrace.googleapis.com \
  telemetry.googleapis.com \
  apphub.googleapis.com \
  apptopology.googleapis.com \
  agentregistry.googleapis.com \
  cloudasset.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  --project=${PROJECT_ID}
```

**Verified Output:**
```text
Operation "operations/acat.p2-403748473564-89f44f0b-de60-46ed-a441-8c9d82ef3b86" finished successfully.
```

> ✅ **Objective 1 Verified:** *Enable the observability and discovery APIs*

---

## Task 2. Export Traces to Cloud Trace

### 2.1 Verify OpenTelemetry Instrumentation in Agent Code

1. **Check `~/agents/salesforce-agent/pyproject.toml`:**
Confirm that `google-adk` includes the `otel-gcp` extra alongside `opentelemetry-instrumentation-google-genai`:

```toml
dependencies = [
    "google-adk[gcp,otel-gcp]>=2.0.0,<3.0.0",
    "a2a-sdk[http-server]~=0.3.22",
    "aiohttp>=3.13.4",
    "opentelemetry-instrumentation-google-genai>=0.1.0,<1.0.0",
    ...
]
```

2. **Check `~/agents/salesforce-agent/app/app_utils/telemetry.py`:**
Confirm `setup_telemetry()` configures `get_gcp_exporters(enable_cloud_tracing=True)`:

```python
    # Set up OpenTelemetry exporters for Cloud Trace and Cloud Logging
    credentials, project_id = google.auth.default()
    otel_hooks = get_gcp_exporters(
        enable_cloud_tracing=True,
        enable_cloud_metrics=False,
        enable_cloud_logging=True,
        google_auth=(credentials, project_id),
    )
    otel_resource = get_gcp_resource(project_id)
    maybe_set_otel_providers(
        otel_hooks_to_setup=[otel_hooks],
        otel_resource=otel_resource,
    )

    # Set up GenAI SDK instrumentation
    _setup_instrumentation_lib_if_installed()
```

### 2.2 Grant the Trace-Writer Role (`roles/telemetry.tracesWriter`) to the Cloud Run Service Account

While Agent Runtime agents (`github-agent`) export traces via managed built-in telemetry, Cloud Run agents (`salesforce-agent`) export traces as their own service account (the Compute Engine default service account `403748473564-compute@developer.gserviceaccount.com`) to `telemetry.googleapis.com`.

Grant `roles/telemetry.tracesWriter` on the project to the `salesforce-agent` service account:

```bash
export SF_SA=$(gcloud run services describe salesforce-agent --region=${LOCATION} --project=${PROJECT_ID} \
  --format='value(spec.template.spec.serviceAccountName)')
# If empty, falls back to ${PROJECT_NUMBER}-compute@developer.gserviceaccount.com
export SF_SA=${SF_SA:-"${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"}

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SF_SA}" \
  --role="roles/telemetry.tracesWriter" \
  --condition=None
```

**Verified Output:**
```text
Updated IAM policy for project [qwiklabs-gcp-04-e58085393db4].
```

### 2.3 Generate Traffic so the System Emits Traces

Query `github_agent` on Agent Runtime (removing `:query` from the end of the Reasoning Engine query URL) using `agents-cli run` with `--url`, `--mode adk`, and `--app-name app`:

```bash
export GITHUB_AGENT_URL="https://${LOCATION}-aiplatform.googleapis.com/v1/projects/${PROJECT_NUMBER}/locations/${LOCATION}/reasoningEngines/146985188302258176"

agents-cli run \
  --url "${GITHUB_AGENT_URL}" \
  --mode adk \
  --app-name app \
  "Hello! What is your role?"
```

**Verified Output:**
```text
Querying remote agent: https://us-central1-aiplatform.googleapis.com/v1/projects/403748473564/locations/us-central1/reasoningEngines/146985188302258176 (mode: adk)
[user]: Hello! What is your role?
[github_agent]: Hello! I am a specialized GitHub assistant. 

My role is to help you interact with GitHub by searching for repositories, finding issues, and retrieving pull request information. Let me know how I can help you with your GitHub tasks today!

Session: 6694584800088424448
  Resume with: agents-cli run "<message>" --url "https://us-central1-aiplatform.googleapis.com/v1/projects/403748473564/locations/us-central1/reasoningEngines/146985188302258176" --mode adk --app-name "app" --session-id 6694584800088424448
```

Also query `salesforce-agent` on Cloud Run to emit spans from the Cloud Run service account:

```bash
export SF_URL=$(gcloud run services describe salesforce-agent --region=${LOCATION} --project=${PROJECT_ID} --format='value(status.url)')

agents-cli run \
  --url "${SF_URL}" \
  --mode adk \
  --app-name app \
  "Hello, who are you?"
```

### 2.4 Confirm Traces Landed in Trace Explorer

1. In the Google Cloud Console, navigate to **Monitoring > Trace explorer**.
2. Set the time range to **Last 1 hour**.
3. Filter by span name `call_llm` or `agent_run`.
4. Click a trace to inspect the waterfall hierarchy:
   - Top-level `invocation` / `agent_run` span for the root agent handling the request.
   - Nested `agent_run` spans for specialist A2A agent calls.
   - `call_llm` spans for Gemini model invocations and `execute_tool` spans for tool/MCP invocations.

> ✅ **Objective 2 Verified:** *Grant the trace-writer role to an agent's service account*

---

## Task 3. Confirm the Agent Runtime Agents Are Already Discovered

### 3.1 List Agents in Agent Registry

```bash
gcloud alpha agent-registry agents list --location=${LOCATION} --project=${PROJECT_ID} --format="table(displayName)"
```

**Verified Output:**
```text
DISPLAY_NAME
Workspace Agent
github-agent
```

### 3.2 Confirm App Hub Classifies the Agent Runtime Workload as `AGENT`

```bash
export TOKEN=$(gcloud auth print-access-token)
curl -s -H "Authorization: Bearer ${TOKEN}" -H "x-goog-user-project: ${PROJECT_ID}" \
  "https://apphub.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/discoveredWorkloads" \
  | jq -r '.discoveredWorkloads[] | "\(.workloadProperties.functionalType.type // "-")\t\(.workloadReference.uri)"'
```

**Verified Output:**
```text
AGENT	//aiplatform.googleapis.com/projects/403748473564/locations/us-central1/reasoningEngines/146985188302258176
```

---

## Task 4. Register the Cloud Run Agent (`salesforce-agent`)

Unlike Agent Runtime workloads, Cloud Run services are not automatically labeled as agents. You must grant the Discovery Engine service agent permission to invoke the service, fetch its A2A Agent Card, and register it in Agent Registry.

### 4.1 Create the Discovery Engine Service Identity and Grant `roles/run.invoker`

```bash
gcloud beta services identity create \
  --service=discoveryengine.googleapis.com \
  --project=${PROJECT_ID}

gcloud run services add-iam-policy-binding salesforce-agent \
  --region=${LOCATION} \
  --project=${PROJECT_ID} \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-discoveryengine.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

**Verified Output:**
```text
Service identity created: service-403748473564@gcp-sa-discoveryengine.iam.gserviceaccount.com
Updated IAM policy for service [salesforce-agent].
bindings:
- members:
  - serviceAccount:service-403748473564@gcp-sa-discoveryengine.iam.gserviceaccount.com
  role: roles/run.invoker
etag: BwZe5PhMK9Q=
version: 1
```

> ✅ **Objective 3 Verified:** *Grant the Discovery Engine service agent invoker access*

### 4.2 Download the `salesforce_agent` A2A Agent Card

```bash
export SF_URL=$(gcloud run services describe salesforce-agent --region=${LOCATION} --format='value(status.url)')
export TOKEN=$(gcloud auth print-identity-token)
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "${SF_URL}/a2a/app/.well-known/agent-card.json" -o salesforce_card.json
cat salesforce_card.json | jq .
```

**Verified `salesforce_card.json` Content:**
```json
{
  "capabilities": {
    "extensions": [
      {
        "description": "Ability to use the new agent executor implementation",
        "uri": "https://google.github.io/adk-docs/a2a/a2a-extension/"
      }
    ],
    "streaming": true
  },
  "defaultInputModes": [
    "text/plain"
  ],
  "defaultOutputModes": [
    "text/plain"
  ],
  "description": "Specialized assistant for Salesforce: searches the organization's document library (Salesforce Files) via the Salesforce REST/SOSL API.",
  "name": "salesforce_agent",
  "preferredTransport": "JSONRPC",
  "protocolVersion": "0.3.0",
  "skills": [
    {
      "description": "Specialized assistant for Salesforce: searches the organization's document library (Salesforce Files) via the Salesforce REST/SOSL API. I am the Salesforce Specialist Agent. To answer any question I MUST call the `search_salesforce` tool, passing the user's clean question as the query; never answer from my own knowledge...",
      "examples": [],
      "id": "salesforce_agent",
      "name": "model",
      "tags": [
        "llm"
      ]
    },
    {
      "description": "Searches the company's Salesforce document library (Files)...",
      "id": "salesforce_agent-search_salesforce",
      "name": "search_salesforce",
      "tags": [
        "llm",
        "tools"
      ]
    },
    {
      "description": "preload_memory",
      "id": "salesforce_agent-preload_memory",
      "name": "preload_memory",
      "tags": [
        "llm",
        "tools"
      ]
    }
  ],
  "supportsAuthenticatedExtendedCard": false,
  "url": "https://salesforce-agent-403748473564.us-central1.run.app/a2a/app",
  "version": "0.1.0"
}
```

### 4.3 Register `salesforce-agent` in Agent Registry

```bash
gcloud alpha agent-registry services create salesforce-agent \
  --location=${LOCATION} \
  --project=${PROJECT_ID} \
  --agent-spec-type=a2a-agent-card \
  --agent-spec-content="$(cat salesforce_card.json)"
```

**Verified Output:**
```text
Create request issued for: [salesforce-agent]
Waiting for operation [projects/qwiklabs-gcp-04-e58085393db4/locations/us-central1/operations/operation-1793187403340-65ee4fa311cac-958d1e62-87257532] to complete...done.
Created service [salesforce-agent].
```

Verify that both `github-agent` and `salesforce_agent` now appear in Agent Registry:

```bash
gcloud alpha agent-registry agents list --location=${LOCATION} --project=${PROJECT_ID} --format="table(displayName,location,protocols[0].type)"
```

**Verified Output:**
```text
DISPLAY_NAME      LOCATION     TYPE
Workspace Agent   global       A2A_AGENT
salesforce_agent  us-central1  A2A_AGENT
github-agent      us-central1  CUSTOM
```

> ✅ **Objective 4 Verified:** *Register the Cloud Run agent in Agent Registry*

---

## Task 5. Register the GitHub MCP Server

Register the external GitHub Copilot MCP server (`https://api.githubcopilot.com/mcp/`) as a service named `github-mcp` in Agent Registry using `--mcp-server-spec-type=no-spec` and `protocolBinding=JSONRPC`:

```bash
gcloud alpha agent-registry services create github-mcp \
  --location=${LOCATION} \
  --project=${PROJECT_ID} \
  --mcp-server-spec-type=no-spec \
  --interfaces="url=https://api.githubcopilot.com/mcp/,protocolBinding=JSONRPC"
```

**Verified Output:**
```text
Create request issued for: [github-mcp]
Waiting for operation [projects/qwiklabs-gcp-04-e58085393db4/locations/us-central1/operations/operation-1793186928786-65ee4dde7fa87-57958033-da598dc2] to complete...done.
Created service [github-mcp].
```

Confirm the MCP server is registered:

```bash
gcloud alpha agent-registry mcp-servers list --location=${LOCATION} --project=${PROJECT_ID}
```

**Verified Output:**
```yaml
---
attributes:
  agentregistry.googleapis.com/system/RuntimeReference:
    uri: //agentregistry.googleapis.com/projects/403748473564/locations/us-central1/services/github-mcp
createTime: '2026-10-28T11:28:49.544598Z'
interfaces:
- protocolBinding: JSONRPC
  url: https://api.githubcopilot.com/mcp/
mcpServerId: urn:mcp:projects-403748473564:projects:403748473564:locations:us-central1:agentregistry:services:github-mcp
name: projects/qwiklabs-gcp-04-e58085393db4/locations/us-central1/mcpServers/agentregistry-00000000-0000-0000-95dc-524dc7cadc85
updateTime: '2026-10-28T11:28:49.544598Z'
```

> ✅ **Objective 5 Verified:** *Register the GitHub MCP server in Agent Registry*

---

## Task 6. View the Agent Topology Graph

1. In the Google Cloud Console, navigate to **Agent Platform > Agent Registry** (`https://console.cloud.google.com/agent-platform/agent-registry`).
2. Select project `qwiklabs-gcp-04-e58085393db4`.
3. On the **Agents** tab, click `github-agent` (or `salesforce_agent`).
4. Click the **Topology** tab.
5. Once batch trace aggregation completes, the topology graph displays the nodes across both runtimes (**Agent Runtime** for `github-agent` and **Cloud Run** for `salesforce_agent`) along with the registered **GitHub MCP server** (`github-mcp`) and their request latency/call edges.

---

## Task 7. See the Fleet in SCC AI Protection, Then Introduce a Realistic Risk

1. In the Google Cloud Console, navigate to **Agent Platform > Agents > Security** (`https://console.cloud.google.com/agent-platform/security`).
2. Review the AI inventory widgets showing discovered agents and MCP servers.
3. Simulate a common real-world misconfiguration by granting the broad `roles/editor` role to the `salesforce-agent` Cloud Run service account:

```bash
export SF_SA=$(gcloud run services describe salesforce-agent --region=${LOCATION} \
  --format='value(spec.template.spec.serviceAccountName)')
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SF_SA}" \
  --role="roles/editor" --condition=None
```

**Verified Output:**
```text
SF_SA: 403748473564-compute@developer.gserviceaccount.com
Updated IAM policy for project [qwiklabs-gcp-04-e58085393db4].
```

---

## Task 8. Identify the Security Finding and Its Scope Using Policy Analyzer

1. In the **Agent Platform Security** tab (`https://console.cloud.google.com/agent-platform/security`), inspect the **Agents with excessive permissions** and **AI risks by severity** widgets to locate the over-privileged `salesforce_agent` Non-Human Identity finding.
2. Prove the exact resource blast radius using **Policy Analyzer**:
   - Navigate to **IAM & Admin > Policy Analyzer** (`https://console.cloud.google.com/iam-admin/analyzer`).
   - Click **Create custom query**.
   - Set **Scope** to project `qwiklabs-gcp-04-e58085393db4`.
   - Set **Parameter 1** to **Principal** and enter `serviceAccount:403748473564-compute@developer.gserviceaccount.com`.
   - Under **Advanced options**, ensure **List resources within resource(s) matching your query** is checked.
   - Click **Analyze > Run query**.
   - Observe that `roles/editor` grants modify access across nearly every resource in the project—far beyond the 4 narrow roles (`roles/aiplatform.user`, `roles/telemetry.tracesWriter`, `roles/logging.logWriter`, `roles/cloudtrace.agent`) the agent actually requires.

---

## Task 9. Fix It and Verify Least Privilege

### 9.1 Inspect Current Roles on the `salesforce-agent` Service Account

```bash
gcloud projects get-iam-policy ${PROJECT_ID} --flatten="bindings[].members" \
  --filter="bindings.members:${SF_SA}" \
  --format="value(bindings.role)"
```

**Verified Output (Before Remediation):**
```text
roles/aiplatform.user
roles/cloudtrace.agent
roles/editor
roles/logging.logWriter
roles/telemetry.tracesWriter
```

### 9.2 Remove `roles/editor` to Restore Least Privilege

```bash
gcloud projects remove-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SF_SA}" \
  --role="roles/editor" --condition=None
```

**Verified Output:**
```text
Updated IAM policy for project [qwiklabs-gcp-04-e58085393db4].
```

### 9.3 Verify Only the Narrow Roles Remain and `roles/editor` Is Gone

```bash
# Confirm remaining narrow roles
gcloud projects get-iam-policy ${PROJECT_ID} --flatten="bindings[].members" \
  --filter="bindings.members:${SF_SA}" \
  --format="value(bindings.role)"

# Confirm roles/editor returns nothing
gcloud projects get-iam-policy ${PROJECT_ID} \
  --flatten="bindings[].members" \
  --filter="bindings.members:${SF_SA} AND bindings.role:roles/editor" \
  --format="value(bindings.role)"
```

**Verified Output (After Remediation):**
```text
roles/aiplatform.user
roles/cloudtrace.agent
roles/logging.logWriter
roles/telemetry.tracesWriter
```
*(And empty output for the `roles/editor` filter, confirming complete removal).*

> ✅ **Objective 6 Verified:** *Remediate the over-privileged agent*

---

## Task 10. (Optional Exercise) Trigger and View an Agent's Runtime Attacks

**Agent Platform Threat Detection** monitors Vertex AI Agent Runtime containers for runtime attacks (such as malicious binary execution, container escapes, and reverse shells) and control-plane threats, surfacing findings in Security Command Center in near real-time:
1. Open **Agent Platform > Agents > Security** and locate the **AI threats** widget.
2. Run the documented **Test Agent Platform Threat Detection** script, which deploys a temporary Agent Runtime test agent whose tool executes a harmless simulated attack (such as the standard EICAR test file), triggers the detector, and deletes the test agent.
3. In **Security Command Center > Findings** (or the **AI threats** widget), inspect the resulting `Execution: Added Malicious Binary Executed` finding to review the affected Reasoning Engine resource, detector details, and remediation guidance.
