# OpenScale Integration Project Documentation & Functional Specification

## 1. Project Overview
This project (**OpenScaleIntegrationApp**) provides IBM App Connect Enterprise (ACE) v13 integration flows to ingest Gen AI and Traditional Machine Learning scoring events from RESTful sources, perform data model transformations using ACE Mapping nodes (`.map`) and ESQL modules, and output standardized payloads compliant with the **IBM Watson OpenScale** monitoring schema.

## 2. Inbound Sources & Targets
1. **Gen AI Source (`Gen AI source.json`)**:
   - Ingests LLM chat completions and generation requests/responses (e.g., Gemini, Watsonx.ai, OpenAI formats).
   - Captures prompts, tool descriptions as context, user queries, LLM response choices, token metrics (prompt tokens, reasoning tokens, completion tokens), and transaction identifiers.
2. **Traditional ML Source (`traditional ML source.json`)**:
   - Ingests tabular / feature-based ML inference requests and predictions (`instances` feature maps and `predictions` arrays).
   - Flattens feature keys into dynamic `fields` and record values into `values` matrices.
3. **OpenScale Payload (`openscale schema.json`)**:
   - Standardized JSON monitoring structure containing `request`, `response`, `scoring_id`, `scoring_timestamp`, `response_time`, `asset_revision`, `multiple_records`, `user_id`, `input_tokens`, and `output_tokens`.

---

## 3. Mapping Specification (Gen AI & Traditional ML)

| OpenScale Target Field | Gen AI Source Mapping | Traditional ML Source Mapping | Description / Type |
|---|---|---|---|
| `scoring_id` | `data.transactionid` (or `data.request_headers.transId` / `transaction_id`) | `data.transactionid` | Unique scoring identifier (String) |
| `scoring_timestamp` | ISO-8601 UTC timestamp converted from `data.response_time_stamp` (e.g. `2026-06-15T17:30:11.483Z`) | ISO-8601 UTC timestamp converted from `data.response_time_stamp` | Timestamp of scoring event (DateTime) |
| `response_time` | `data.response_time` | `data.response_time` | Latency / execution duration in ms (Integer) |
| `asset_revision` | `"v1.0.0"` | `"v1.0.0"` | Model asset revision string |
| `multiple_records` | `false` | `false` | Batch indicator (Boolean) |
| `user_id` | `data.request_headers.x-request-id` | Imputed UUID (e.g. generated or mapped identifier) | User/caller tracking ID (String) |
| `input_tokens` | `data.response.usage.prompt_tokens` | `NULL` / omitted | Inbound prompt token count (Integer) |
| `output_tokens` | `data.response.usage.completion_tokens + COALESCE(data.response.usage.completion_tokens_details.reasoning_tokens, 0)` | `NULL` / omitted | Total generated / reasoning tokens (Integer) |
| `request.fields` | `["prompt", "context", "user_query"]` | Array of keys extracted from `data.request.instances[0]` | Column names array (Array[String]) |
| `request.values` | `[[ messages[0].content, concat(tools.description), messages[-1].content ]]` | Array of values extracted from `data.request.instances` in matching field order | Data matrix (Array[Array[Any]]) |
| `response.fields` | `["response"]` | `["prediction"]` (or key names from `response.predictions`) | Response column names array |
| `response.values` | `[[ data.response.choices[0].message.content ]]` | Matrix of prediction values from `data.response.predictions` | Prediction matrix (Array[Array[Any]]) |

---

## 4. Architecture & Message Flows
- **`OpenScale_GenAI_Ingest.msgflow`**:
  - `ComIbmWSInput` (`/openscale/genai`) $\rightarrow$ `ComIbmWSRequest` (Optional Gen AI REST fetch) $\rightarrow$ `ComIbmMSLMapping` (`OpenScale_GenAI_Mapping.map`) $\rightarrow$ `ComIbmCompute` (Timestamp / JSON normalization) $\rightarrow$ `ComIbmWSReply` / Outbound Dispatcher.
- **`OpenScale_TraditionalML_Ingest.msgflow`**:
  - `ComIbmWSInput` (`/openscale/traditionalml`) $\rightarrow$ `ComIbmMSLMapping` (`OpenScale_TraditionalML_Mapping.map`) $\rightarrow$ `ComIbmCompute` (Dynamic feature flattening & UUID imputation) $\rightarrow$ `ComIbmWSReply` / Outbound Dispatcher.
