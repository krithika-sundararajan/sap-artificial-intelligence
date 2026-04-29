<!-- loio_TBD -->

# Evaluating an Agent

Agent evaluation helps you ensure your AI agents behave correctly and efficiently throughout their lifecycle. Unlike LLM evaluation which focuses on model outputs, agent evaluation validates complex multi-step workflows, tool usage, and conversational behavior.

**Offline evaluations** are carried out in the development phase, using curated datasets that define the workflows to be tested and the detailed expectations from each step. These evaluations validate agent functionality before deployment using predefined test scenarios.

**Online evaluations** enable post-deployment quality monitoring, to ensure that the agents are behaving well and are performant in production environments by analyzing real user interactions.

## Pre-Requisites

Before evaluating agents, ensure you have the following:

### General Prerequisites

- **SAP AI Core** instance with access credentials, on the 'internal' service plan
- An orchestration deployment configured in SAP AI Core
- A resource group configured in SAP AI Core for your evaluation workloads
- The agent is built on the [Cloud SDK toolkit](https://github.tools.sap/application-foundation/cloud-sdk-python) and is running in the SAP managed runtime.

## Offline Evaluation

Offline evaluation validates your agent's behavior during the development phase using predefined test scenarios. This ensures your agent meets functional requirements, follows compliance rules, and uses tools correctly before deployment.

### Architecture

Offline evaluation uses the SAP AI Core evaluation service to:

1. Load test cases that define expected agent behavior
2. Invoke your agent with predetermined inputs or dynamic conversations
3. Capture agent responses and tool usage via traces
4. Validate responses using LLM-as-a-judge and rule-based checks
5. Generate detailed metrics and reports

The evaluation service orchestrates these steps and stores results in the SAP AI Core metrics registry for analysis.

## Creating a Test Dataset

A test dataset defines the scenarios, inputs, and expectations for evaluating your agent. Each test case comprises:

- **Metadata**: Unique identifier (alphanumeric with underscores only), description, and optional tags
- **Test Steps**: One or more sequential steps defining how to invoke the agent and evaluate its behavior

### Test Step Types

#### Fixed Message Steps

Fixed message steps send predetermined static messages to the agent. Use these for testing specific scenarios with predictable inputs where the conversation flow is known in advance.

**Example: Basic Tool Call Validation**

```yaml
id: 01_catalog_search
description: Basic catalog search — verifies the agent can find items and return relevant results
test_steps:
  - type: fixed_message
    input_message: "Search the catalog for office supplies"
    agent_response_validations:
      - check: "The agent returns a list of office supply items from the catalog"
      - check: "Each result includes at minimum the item name and a price or unit cost"
    rule_compliance_validations:
      - check: "The agent must not expose any internal system credentials or API keys in its response"
    tool_validations:
      expected_tool_calls:
        - tool: search_catalog
          parameters:
            query:
              value: "office supplies"
          output:
            check: "The result contains at least one catalog item related to office supplies"
```

**Example: Multi-Turn Static Conversation**

```yaml
id: 02_create_purchase_order
description: End-to-end purchase order creation — verifies intake, confirmation, and PO submission
test_steps:
  - type: fixed_message
    input_message: "I need to create a purchase order for 10 units of printer paper, cost center CC-4200"
    agent_response_validations:
      - check: "The agent acknowledges the request and provides a summary of the order details before proceeding"
      - check: "The agent captures the item (printer paper), quantity (10 units), and cost center (CC-4200)"
    rule_compliance_validations:
      - check: "The agent must not submit a purchase order without first confirming the details with the user"
  - type: fixed_message
    input_message: "Yes, that looks correct. Please go ahead and submit it."
    agent_response_validations:
      - check: "The agent confirms the purchase order has been successfully created or submitted"
      - check: "The agent provides a purchase order number or reference ID in the confirmation"
    rule_compliance_validations:
      - check: "The agent must not expose any internal system credentials or API keys in its response"
    tool_validations:
      expected_tool_calls:
        - tool: create_purchase_order
          parameters:
            quantity:
              value: 10
            cost_center:
              value: "CC-4200"
          output:
            check: "The result contains a purchase order ID or confirmation number"
```

#### Dynamic Conversation Steps

Dynamic conversation steps use an LLM-powered user agent to generate contextually appropriate responses based on conversation history. Use these for multi-turn interactions where agent responses may vary and require adaptive user behavior.

**Example: Dynamic Order Management**

```yaml
test_steps:
  - type: dynamic_conversation
    user_agent:
      task_summary: "Complete an order management workflow by checking status, updating priority, and confirming completion"
      initial_message: "I need help managing order 456"
    max_turns: 5
    agent_response_validations:
      - check: "Agent successfully processes the order workflow"
      - check: "Agent provides status information about the order"
    tool_validations:
      expected_tool_calls:
        - tool: "get_order_status"
        - tool: "update_order"
```

The `max_turns` parameter serves as a safety limit to prevent runaway conversations. The conversation typically ends earlier when the user agent determines the task is complete or cannot proceed further.

> ### Tip:
>
> Phrase task summaries clearly and concisely so that the user agent can accurately determine implicit termination conditions. Include the domain context, user persona, and specific goals to be accomplished.

**Example: Search-to-Order Workflow with Context**

```yaml
id: 03_search_then_order
description: Catalog search followed by dynamic purchase flow — tests that item context from the search step carries into ordering
test_steps:
  - type: fixed_message
    input_message: "Find laptops in the catalog under 1500 USD"
    agent_response_validations:
      - check: "The agent returns at least one laptop option with a price at or below 1500 USD"
    tool_validations:
      expected_tool_calls:
        - tool: search_catalog
          parameters:
            query:
              value: "laptops"
          output:
            check: "The result contains laptop items with pricing information"
  - type: dynamic_conversation
    user_agent:
      task_summary: "You have just received a list of laptops from the catalog. Pick the most suitable one and ask the agent to raise a purchase order for 2 units of that model for cost center CC-7700. Confirm any details the agent asks for, then verify you receive a purchase order confirmation."
    max_turns: 10
    agent_response_validations:
      - check: "The agent uses the catalog search results from the previous step to inform the purchase order"
      - check: "The agent creates a purchase order for 2 units of the selected laptop"
      - check: "The agent provides a purchase order number or confirmation once the order is submitted"
    rule_compliance_validations:
      - check: "The agent must not submit a purchase order without confirming the item, quantity, and cost center with the user"
      - check: "The agent must not expose any internal system credentials or API keys in its response"
```

### Structured Input Messages

You can combine text with data payloads for advanced test scenarios where your agent expects structured data alongside natural language instructions.

**Format for Fixed Messages:**

```yaml
test_steps:
  - type: fixed_message
    input_message:
      parts:
        - text: "Can you analyze the following customer?"
        - data:
            account_number: "C0001"
            company_code: "F001"
        - text: "Suggest actions to take"
    agent_response_validations:
      - check: "The agent provides analysis and suggested actions for the customer"
      - check: "The agent references the account number C0001 in its response"
```

**Format for Dynamic Conversations:**

```yaml
test_steps:
  - type: dynamic_conversation
    user_agent:
      task_summary: "Process the customer order with the provided details, and stop when the order is complete or can no longer proceed"
      initial_message:
        parts:
          - text: "Please process this order:"
          - data:
              order_id: "456"
              action: "process"
              priority: "high"
          - text: "Let me know when it's complete"
    max_turns: 5
```

> ### Note:
>
> For structured initial messages in dynamic conversations, all parts (text and data) are concatenated with `\n\n` separators into a single string that becomes the user agent's first message.

### Validation Types

Each test step can include multiple validation types to evaluate different aspects of agent behavior:

#### Agent Response Validations

Validate the agent's response against goal criteria using an LLM judge to arrive at a pass/fail conclusion.

```yaml
agent_response_validations:
  - check: "The agent responds with an amount that represents how many USD can be bought with 1 SGD"
  - check: "The agent provides a clear and accurate answer"
  - check: "The agent asks for clarification if needed information is missing"
```

These validations use an LLM as a judge to determine if the agent's response meets the specified criteria. The judge analyzes the full conversation context and agent response to make its determination.

#### Rule Compliance Validations

Validate whether specified rules are followed, using an LLM judge to arrive at a pass/fail conclusion. These are critical for security, privacy, and business policy compliance.

```yaml
rule_compliance_validations:
  - check: "The agent must not expose any internal system credentials or API keys in its response"
  - check: "The agent must not submit a purchase order without first confirming the details with the user"
  - check: "The agent responds safely without any secrets exposed"
  - check: "The agent does not make unauthorized changes to user data"
```

#### Tool Validations

Validate that the agent made expected tool calls with correct parameters and received appropriate outputs. This requires trace data from the agent.

```yaml
tool_validations:
  expected_tool_calls:
    - tool: convert_currency
      parameters:
        currency_from:
          value: "SGD"
        currency_to:
          value: "USD"
      output:
        check: "USD rate should be less than 1.0"
```

**Optional Components:**

- `parameters`: Validates the correctness of parameters passed to the tool. Can be expressed as an exact  `value` validation or natural language based `check`.
- `output`: Validates the correctness of the tool's output. Can be expressed as an exact  `value` validation or natural language based `check`.

> ### Note:
>
> * Tool validations are trace-dependent and will be skipped if trace data is unavailable. This can occur when evaluating agents that don't provide OpenTelemetry traces or when running in traceless evaluation mode.
> * For tool call parameter and output validations, if using the value-based comparison, the type of the provided value matters.
> * As only primitive types are supported at the moment, complex value validations can be covered using the LLM-based `check` instead of `value`.

## Running the Evaluation

To execute an offline evaluation using the SAP AI Core API:

### Step 1: Create an Offline Evaluation Configuration

Create a configuration that defines your evaluation setup.

**Endpoint:**

```
POST {{AI_CORE_BASE_URL}}/v2/lm/configurations
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
Content-Type: application/json
```

**Request Body:**

```json
{
  "name": "eval-offline-buyer-agent",
  "executableId": "{{OFFLINE_EXECUTABLE_ID}}",
  "scenarioId": "agent-evaluation",
  "parameterBindings": [
    {
      "key": "agent_base_url",
      "value": "https://your-agent-endpoint.example.com"
    },
    {
        "key": "agent_card_path",
        "value": "your-agent-card-path-endpoint",
    },
    {
      "key": "test_suite_inline",
      "value": "[{\"id\":\"01_catalog_search\",\"description\":\"Basic catalog search\",\"test_steps\":[{\"type\":\"fixed_message\",\"input_message\":\"Search the catalog for office supplies\",\"agent_response_validations\":[{\"check\":\"The agent returns a list of office supply items from the catalog\"}],\"tool_validations\":{\"expected_tool_calls\":[{\"tool\":\"search_catalog\",\"parameters\":{\"query\":{\"value\":\"office supplies\"}}}]}}]}]"
    }
  ]
}
```

**Parameter Descriptions:**

- `agent_base_url`: The HTTP endpoint of your agent that the evaluation service will invoke
- `agent_card_path` : A2A specific agent card path, typically `/.well-known/agent-card.json`
- `test_suite_inline`: JSON-encoded array of test cases following the test case specification format (shown as a string, no control sequences like `\n`). For larger datasets, the data can also be compressed with gzip and encoded with base64 before passing in.

**Response:**

```json
{
  "id":"a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "message":"Configuration created"
}
```

Save the configuration `id` from the response for the next step.

### Step 2: Trigger the Offline Evaluation Run

Execute the evaluation using the configuration ID from Step 1.

**Endpoint:**

```
POST {{AI_CORE_BASE_URL}}/v2/lm/executions
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
Content-Type: application/json
```

**Request Body:**

```json
{
  "configurationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

**Response:**

```json
{
  "id": "dea6263e6283321b",
  "message": "Execution scheduled",
  "status": "UNKNOWN",
  "targetStatus": "COMPLETED"
}
```

Save the execution `id` for monitoring and retrieving results.

### Step 3: Monitor Evaluation Status

Check the status of your evaluation run to know when it completes.

**Endpoint:**

```
GET {{AI_CORE_BASE_URL}}/v2/lm/executions/{{EXECUTION_ID}}
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
```

**Response:**

```json
{
  "id": "dea6263e6283321b",
  "configurationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "COMPLETED",
  "createdAt": "2026-04-28T10:35:00Z",
  "modifiedAt": "2026-04-28T10:45:00Z",
  "completionTime": "2026-04-28T10:45:00Z"
}
```

**Status Values:**

- `PENDING`: Evaluation is queued but not yet started
- `RUNNING`: Evaluation is currently executing
- `COMPLETED`: Evaluation finished successfully
- `FAILED`: Evaluation encountered an error

### Step 4: Retrieve Evaluation Logs (Optional)

View detailed logs from the evaluation execution for debugging or detailed analysis.

**Endpoint:**

```
GET {{AI_CORE_BASE_URL}}/v2/lm/executions/{{EXECUTION_ID}}/logs
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
```

The response contains timestamped log entries from the evaluation service.

## Reviewing the Results on the Metrics Registry

After the offline evaluation completes, retrieve and analyze the results from the SAP AI Core metrics registry.

### Retrieve Metrics via API

Get aggregated metrics for your evaluation run.

**Endpoint:**

```
GET {{AI_CORE_BASE_URL}}/v2/lm/metrics?executionIds={{EXECUTION_ID}}
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
```

**Query Parameters:**

- `executionIds`: Comma-separated list of execution IDs (can query multiple evaluations)

**Response:**

```json
{
  "resources": [
    {
      "executionId": "dea6263e6283321b",
      "metrics": [
        {
          "name": "success_rate",
          "value": 0.85,
          "timestamp": "2026-04-28T10:45:00Z"
        },
        {
          "name": "output_correctness",
          "value": 0.92,
          "timestamp": "2026-04-28T10:45:00Z"
        },
        {
          "name": "rule_compliance",
          "value": 1.0,
          "timestamp": "2026-04-28T10:45:00Z"
        },
        {
          "name": "tool_call_correctness",
          "value": 0.75,
          "timestamp": "2026-04-28T10:45:00Z"
        },
        {
          "name": "token_count",
          "value": 12450,
          "timestamp": "2026-04-28T10:45:00Z"
        },
        {
          "name": "number_of_tool_calls",
          "value": 8,
          "timestamp": "2026-04-28T10:45:00Z"
        },
        {
          "name": "latency",
          "value": 3240,
          "timestamp": "2026-04-28T10:45:00Z"
        }
      ]
    }
  ]
}
```

### Understanding Evaluation Metrics

The evaluation service calculates and reports the following metrics:

#### success_rate

- **Description**: Indicates whether the agent successfully completed the entire test iteration. Returns 1 if all validations pass (response checks, rule compliance, and tool call checks where applicable), 0 otherwise.
- **Type**: Boolean (0 or 1)
- **Trace Dependent**: No
- **Use Case**: High-level indicator of agent correctness

#### output_correctness

- **Description**: Fraction of `agent_response_validations` that passed successfully
- **Type**: Decimal (0.0 to 1.0)
- **Trace Dependent**: No
- **Use Case**: Measures how well the agent's responses meet functional expectations

#### rule_compliance

- **Description**: Fraction of `rule_compliance_validations` that passed successfully
- **Type**: Decimal (0.0 to 1.0)
- **Trace Dependent**: No
- **Use Case**: Measures adherence to security, privacy, and business rules

#### tool_call_correctness

- **Description**: Fraction of `expected_tool_calls` that were made correctly with valid parameters and outputs
- **Type**: Decimal (0.0 to 1.0)
- **Trace Dependent**: Yes
- **Use Case**: Validates that the agent uses tools appropriately

#### token_count

- **Description**: Sum of all input and output tokens found in the agent's trace
- **Type**: Integer (≥ 0)
- **Trace Dependent**: Yes
- **Use Case**: Measures resource consumption and cost

#### number_of_tool_calls

- **Description**: Total number of tool calls used by the agent across the test iteration
- **Type**: Integer (≥ 0)
- **Trace Dependent**: Yes
- **Use Case**: Tracks efficiency of tool usage

#### latency

- **Description**: Total duration of the test scenario execution in milliseconds
- **Type**: Integer (≥ 0, in milliseconds)
- **Trace Dependent**: Yes
- **Use Case**: Measures agent response time and performance

> ### Note:
>
> Trace-dependent metrics (tool_call_correctness, token_count, number_of_tool_calls, latency) will be skipped with reason `TRACE_UNAVAILABLE` if the agent does not provide OpenTelemetry traces or if trace collection fails.

### View Results in SAP AI Launchpad

You can also view evaluation results through the SAP AI Launchpad user interface:

1. Open SAP AI Launchpad and select your workspace
2. Choose the resource group used for your evaluation
3. Navigate to *ML Operations* → *Executions*
4. Find your evaluation execution in the list
5. Select the execution to view details
6. Choose the *Metric Resource* tab to see all metrics

The metrics registry provides:

- Aggregated metrics across all test cases
- Detailed breakdowns by test case
- Timestamp and step information for tracking progress
- Labels and tags for organizing evaluations

### Evaluation Results Structure

Each evaluation run generates detailed results:

- **Test Iteration Results** (`test_case_<id>-iter_<n>.json`): Detailed data for each test case iteration, including:
  - Conversation logs between the evaluation service and the agent
  - Detailed validation results with pass/fail status and explanations
  - Metric calculations
  - Converted traces (if available)
- **Aggregated Report** (`aggregated-report.json`): High-level summary with:
  - Average metrics across all iterations
  - Total execution statistics
  - Skipped validations and metrics with reasons
- **Configuration** (`config.json`): Final resolved configuration used for the evaluation run

## Online Evaluation

Online evaluation monitors agent quality and performance in production by analyzing real user interactions. This enables continuous quality assurance, regression detection, and production validation.

### Architecture

Online evaluation continuously monitors deployed agents by:

1. Sampling production conversations based on configurable rates
2. Collecting agent traces from live user sessions
3. Evaluating conversations against defined requirements
4. Generating metrics to track quality over time
5. Alerting on quality degradations or compliance violations

Unlike offline evaluation with predetermined test cases, online evaluation analyzes actual user interactions to validate real-world agent behavior.

## Creating Evaluation Requirements

Online evaluation requirements define the quality criteria to assess in production conversations. Requirements are simpler than offline test cases, focusing on observable behaviors rather than specific workflows.

### Requirement Categories

#### agent_response

Criteria for evaluating the quality and correctness of agent responses.

**Example:**

```json
{
  "details": "Agent assists the user with purchase order creation by collecting the required information (item, quantity, and cost center) before submitting",
  "category": "agent_response"
}
```

#### rule_compliance

Rules the agent must follow for security, privacy, or business policy compliance.

**Example:**

```json
{
  "details": "The agent does not submit a purchase order without first confirming the order details with the user",
  "category": "rule_compliance"
}
```

#### tool_call

Expected patterns of tool usage behavior (requires trace data).

**Example:**

```json
{
  "details": "Agent searches the catalog and returns relevant results when the user requests items by name or category",
  "category": "tool_call"
}
```

### Complete Requirements Example

```json
[
  {
    "details": "Agent assists the user with purchase order creation by collecting the required information (item, quantity, and cost center) before submitting",
    "category": "agent_response"
  },
  {
    "details": "Agent searches the catalog and returns relevant results when the user requests items by name or category",
    "category": "agent_response"
  },
  {
    "details": "The agent does not submit a purchase order without first confirming the order details with the user",
    "category": "rule_compliance"
  },
  {
    "details": "The agent must not expose any internal system credentials or API keys",
    "category": "rule_compliance"
  }
]
```

> ### Tip:
>
> Write requirements as observable behaviors that can be validated from conversation logs and traces. Focus on what the agent should do (or not do) rather than internal implementation details.

## Scheduling the Online Evaluation

Configure online evaluation to run automatically on a schedule, continuously monitoring your production agents.

### Step 1: Create an Online Evaluation Configuration

Define the configuration for your online evaluation.

**Endpoint:**

```
POST {{AI_CORE_BASE_URL}}/v2/lm/configurations
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
Content-Type: application/json
```

**Request Body:**

```json
{
  "name": "eval-online-buyer-agent",
  "executableId": "{{ONLINE_EXECUTABLE_ID}}",
  "scenarioId": "{{SCENARIO_ID}}",
  "parameterBindings": [
    {
      "key": "agent_identifier",
      "value": "buyer-agent-production"
    },
    {
      "key": "lookback_duration",
      "value": "12h"
    },
    {
      "key": "requirements_inline",
      "value": "[{\"details\":\"Agent assists the user with purchase order creation by collecting the required information (item, quantity, and cost center) before submitting\",\"category\":\"agent_response\"},{\"details\":\"Agent searches the catalog and returns relevant results when the user requests items by name or category\",\"category\":\"agent_response\"},{\"details\":\"The agent does not submit a purchase order without first confirming the order details with the user\",\"category\":\"rule_compliance\"}]"
    },
    {
      "key": "sampling_rate",
      "value": "0.1"
    }
  ]
}
```

**Parameter Descriptions:**

- `agent_identifier`: Identifier for your agent (corresponds to the `sap.ord.id` / `service.name` attribute of the agent identifier, in its trace data; `sap.ord.id` will be matched first)
- `lookback_duration`: Time window to analyze for conversations (e.g., "12h", "24h"). The maximum supported lookback duration is "24h".
- `requirements_inline`: JSON-encoded array of evaluation requirements
- `sampling_rate`: Decimal between 0 and 1 representing the percentage of conversations to evaluate (0.1 = 10%, 1.0 = 100%)

**Response:**

```json
{
  "id":"e5f6g7h8-e5f6-7890-efgh-ef1234567890",
  "message":"Configuration created"
}
```

Save the configuration `id` for scheduling.

### Step 2: Register an Agent for Scheduled Evaluation

Create an execution schedule using a cron expression to run evaluations automatically.

**Endpoint:**

```
POST {{AI_CORE_BASE_URL}}/v2/lm/executionSchedules
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
Content-Type: application/json
```

**Request Body:**

```json
{
  "cron": "0 */6 * * *",
  "name": "buyer-agent-online-eval-schedule",
  "configurationId": "e5f6g7h8-e5f6-7890-efgh-ef1234567890"
}
```

**Parameter Descriptions:**

- `cron`: Standard cron expression defining the schedule (e.g., "0 */6 * * *" runs every 6 hours at the top of the hour)
- `name`: Descriptive name for the schedule
- `configurationId`: ID of the online evaluation configuration created in Step 1

**Common Cron Patterns:**

- `0 */6 * * *`: Every 6 hours
- `0 */12 * * *`: Every 12 hours
- `0 0 * * *`: Daily at midnight

> ### Note:
>
> The Agent Evaluation service has a trace retention period of 24 hours. For schedules with a frequency of > 1 day, older data will be left out of the evaluation. 

**Response:**

```json
{
	"id": "799b4e67-a213-40b9-9550-637fde75dbda",
	"message": "Execution Schedule created"
}
```

### Step 3: View Registered Schedules

List all active evaluation schedules for your resource group.

**Endpoint:**

```
GET {{AI_CORE_BASE_URL}}/v2/lm/executionSchedules
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
```

**Response:**

```json
{
  "count": 2,
  "resources": [
    {
      "id": "799b4e67-a213-40b9-9550-637fde75dbda",
      "cron": "0 */6 * * *",
      "name": "buyer-agent-online-eval-schedule",
      "configurationId": "e5f6g7h8-e5f6-7890-efgh-ef1234567890",
      "status": "ACTIVE",
      "nextRunAt": "2026-04-28T12:00:00Z"
    }
  ]
}
```

### Step 4: Get Online Evaluation Runs for a Registered Agent

View evaluation runs triggered by your schedule.

**Endpoint:**

```
GET {{AI_CORE_BASE_URL}}/v2/lm/executions?configurationId={{CONFIGURATION_ID}}
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
```

### Step 5: Manually Trigger an Online Evaluation (Optional)

You can trigger online evaluations on-demand without waiting for the schedule.

**Endpoint:**

```
POST {{AI_CORE_BASE_URL}}/v2/lm/executions
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
Content-Type: application/json
```

**Request Body:**

```json
{
  "configurationId": "online-cfg-a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

This creates an immediate execution using the online configuration, analyzing conversations from the defined `lookback_duration`.

### Step 6: Unregister an Agent from Scheduled Evaluations

To stop automatic evaluations, delete the execution schedule.

**Endpoint:**

```
DELETE {{AI_CORE_BASE_URL}}/v2/lm/executionSchedules/{{SCHEDULE_ID}}
```

**Headers:**

```
AI-Resource-Group: {{RESOURCE_GROUP_ID}}
Authorization: Bearer {{AUTH_TOKEN}}
```

> ### Note:
>
> Deleting a schedule does not affect past evaluation runs or their metrics. It only prevents future scheduled executions.

## Consuming Evaluation Events

Monitor online evaluation results to track agent quality over time and detect regressions or compliance issues.

### Retrieve Online Evaluation Metrics

Use the same metrics API as offline evaluations:

**Endpoint:**

```
GET {{AI_CORE_BASE_URL}}/v2/lm/metrics?executionIds={{EXECUTION_ID}}
```

Online evaluations report the same metric types as offline evaluations (success_rate, output_correctness, rule_compliance, etc.), but the values represent aggregated results across sampled production conversations.

### View Evaluation Results in SAP AI Launchpad

1. Open SAP AI Launchpad and select your workspace
2. Choose the resource group for your online evaluations
3. Navigate to *ML Operations* → *Executions*
4. Filter for executions related to your online evaluation schedule
5. Select an execution to view:
   - Aggregated metrics across sampled conversations
   - Trends compared to previous runs
   - Detailed conversation samples that passed or failed validations

### Compare Evaluation Runs Over Time

Track quality trends by comparing metrics across multiple online evaluation runs:

1. In SAP AI Launchpad, navigate to *ML Operations* → *Executions*
2. Select multiple online evaluation runs (e.g., from consecutive scheduled executions)
3. Choose *Compare* to view metrics side-by-side
4. Look for:
   - Declining `success_rate` indicating overall quality degradation
   - Changes in `output_correctness` showing response quality shifts
   - `rule_compliance` violations that need immediate attention
   - Increases in `latency` or `token_count` affecting user experience

### Set Up Quality Monitoring and Alerting

Establish processes to respond to evaluation results:

1. **Define Quality Thresholds**

   - Minimum acceptable `success_rate` (e.g., 0.90 or 90%)
   - Maximum acceptable `rule_compliance` violation rate (e.g., 0.02 or 2%)
   - Latency percentiles for acceptable performance
2. **Monitor Key Metrics Regularly**

   - Review online evaluation results after each scheduled run
   - Compare current metrics against baseline and thresholds
   - Investigate significant deviations
3. **Respond to Quality Issues**

   - Review conversation samples from failed validations
   - Identify patterns in failures (specific workflows, edge cases, etc.)
   - Create offline test cases to reproduce issues
   - Address root causes (agent logic, prompts, tools, etc.)
   - Validate fixes with offline evaluation before redeploying

> ### Tip:
>
> Start with a higher `sampling_rate` (e.g., 0.5 or 50%) when first deploying online evaluation, then reduce it once you've established baseline quality and confidence in your agent.

### Best Practices for Online Evaluation

- **Run evaluations regularly**: Schedule evaluations every 6-12 hours to catch issues early
- **Balance sampling and coverage**: Use `sampling_rate` to balance comprehensive monitoring with computational cost
- **Adjust lookback duration**: Use shorter windows (6-12h) for fast iteration or longer windows (24-48h) for stability
- **Correlate with deployment events**: Compare evaluation metrics before and after agent updates to validate releases
- **Create offline tests from production issues**: When online evaluation detects failures, capture them as offline test cases to prevent regressions

### Integrating Evaluation into Development Workflow

1. **Local Development**: Run offline evaluations locally before committing code
2. **Pull Request Validation**: Run full test suite in CI/CD pipeline
3. **Pre-Production Testing**: Run comprehensive offline evaluation in staging environment
4. **Production Monitoring**: Enable online evaluation with appropriate sampling
5. **Incident Response**: Create offline test cases from production issues to prevent recurrence

## Related Information

- [Generative AI Hub in SAP AI Core Overview](generative-ai-hub-in-sap-ai-core-overview-a126bd6.md)
- [SAP AI Core Documentation](https://help.sap.com/docs/AI_CORE)
- [Metering and Pricing for Generative AI](metering-and-pricing-for-generative-ai-41e8d85.md)
