---
name: performance-test-generation
description: End-to-end workflow to analyze REST API specifications, design performance test scenarios (Load, Stress, Spike, Endurance), generate clean and runnable Apache JMeter test plans (*.jmx), parameterize test data (*.csv), and construct headless execution commands.
---

# Performance Test Generation Skill

## Overview
This skill guides an AI agent through designing and generating production-ready, runnable performance test artifacts from any standardized REST API specification. It ensures test plans follow industry-standard performance topologies, include proper data correlation, apply realistic pacing/think-times, and generate automated non-GUI CLI execution commands.

---

## Input Requirements
* **Primary (Mandatory):** OpenAPI / Swagger / Markdown / JSON REST API Specification.
* **Secondary (Optional):** Specific Performance Objectives, Target Service Level Objectives (SLOs), Traffic Profiles, Concurrency constraints.

---

## Standard 5-Step Generation Workflow

```
┌────────────────────────────────────────────────────────┐
│ Step 1: API Specification & Dependency Analysis        │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│ Step 2: Workload Topology & Scenario Selection         │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│ Step 3: JMeter Test Plan (*.jmx) XML Generation        │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│ Step 4: Data-Driven CSV Dataset Generation             │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│ Step 5: Headless CLI Execution Commands & Scripting    │
└────────────────────────────────────────────────────────┘
```

---

### Step 1: Analyze API Specification & Dynamic Flow
1. Read the input API specification and extract:
   - Target base URL, protocol, and default port.
   - Endpoint paths and HTTP Methods (`GET`, `POST`, `PUT`, `DELETE`, etc.).
   - Required headers (e.g., `Content-Type: application/json`, `Accept`).
   - Authentication scheme (e.g., Bearer JWT, API Key, Basic Auth, Session Cookie).
   - Request payloads (JSON schema, query parameters, path variables).
   - Response structures and dynamic values required for downstream requests.
2. Construct a realistic, chained **End-to-End (E2E) user journey**:
   - Establish dependency chains (e.g., Register/Login $\to$ Extract Token $\to$ Query Resource $\to$ Extract ID $\to$ Mutate Resource).
   - Ensure the workflow reflects authentic business behavior.
3. Consult `knowledge/api-analysis.md` for correlation and dependency strategies.

---

### Step 2: Design Performance Test Scenarios
Determine the appropriate test scenarios matching requirements:
* **Load Test (Baseline / Expected Load):** Validate performance under standard concurrent user traffic.
* **Stress Test (Saturation / Scalability):** Step-up staircase load to identify the degradation inflection point and maximum capacity.
* **Spike Test (Surge & Recovery):** Rapid surge to multiple times normal load to assess system elasticity and queue recovery.
* **Endurance Test (Soak / Stability):** Sustained load over an extended period (15m–24h) to detect memory leaks and resource exhaustion.
* Consult `knowledge/test-types.md` for sizing rules, ramp-up calculations, and pacing definitions.

---

### Step 3: Generate JMeter XML Test Plans (`*.jmx`)
1. Generate valid, well-structured Apache JMeter 5.6.3 compatible XML test plans.
2. Ensure every test plan contains:
   - `TestPlan` root element with user-defined variables (`host`, `port`, `threads`, `rampup`, `duration`).
   - Config elements: `ConfigTestElement` (HTTP Request Defaults), `HeaderManager` (global/custom headers).
   - Native Thread Groups matching scenario topology (Single for Load/Endurance; Multi-Thread Group with start delays for Staircase Stress; Baseline + Surge Thread Groups for Spike).
   - Chained `HTTPSamplerProxy` elements with explicit HTTP method, path, and body.
   - `JSONPostProcessor` or `RegexExtractor` for dynamic token/ID correlation.
   - `ResponseAssertion` for HTTP status code and response payload integrity.
   - Pacing Timers (`GaussianRandomTimer` or `UniformRandomTimer`) to model human think time.
   - Distinct Listener configuration per scenario to avoid duplication.
   - **Critical XML Rule:** Ensure `<assertionsResultsToSave>0</assertionsResultsToSave>` is an integer to prevent XStream `NumberFormatException`.
3. Consult `templates/jmx-plan-template.md` for structural XML definitions.

---

### Step 4: Generate Test Data (`*.csv`)
1. Create parameterized CSV datasets with realistic headers and values (e.g., `user_email`, `password`, `search_term`, `category`).
2. Implement dynamic runtime identity generation when unique entities are required per thread iteration (using `${__threadNum}` and `${__counter}`).
3. Save CSV files into a dedicated `data/` subfolder with relative paths configured in `CSVDataSet`.

---

### Step 5: Generate Headless Execution Commands
1. Construct non-GUI CLI execution commands for automated, reproducible test execution:
   ```bash
   jmeter -n \
     -t <test_plan_path>.jmx \
     -Jhost=<target_host> \
     -Jport=<target_port> \
     -Jthreads=<vus> \
     -Jrampup=<ramp_seconds> \
     -Jduration=<duration_seconds> \
     -l results/<scenario>/raw.jtl \
     -j results/<scenario>/jmeter.log \
     -e -o results/<scenario>/html
   ```
2. Include system resource monitoring guidelines (`htop`, Task Manager, Prometheus) to capture concurrent CPU, RAM, and I/O metrics.
3. Consult `templates/execution-commands-template.md` for full command and evidence capture specifications.

---

## Output Artifacts
* Complete, runnable JMeter Test Plans: `<scenario_name>.jmx`
* Parameterized Test Datasets: `data/<dataset_name>.csv`
* Headless CLI Execution & Monitoring Guide: `execution_commands.md`
