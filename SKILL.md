---
name: ThePinPointer
description: Use when the user wants to investigate, analyze, or find anomalies in Snowflake data. Profiles databases, runs data investigations, and builds a Streamlit dashboard to visualize findings. Supports broad exploration or targeted analysis based on user goals.
---

# ThePinPointer

An interactive, multi-turn investigation skill. Give it a Snowflake database and an optional goal — it will explore the data, build a semantic understanding, uncover anomalies, and produce a polished Streamlit dashboard to visualize the findings. Every major step is confirmed with the user before proceeding.

## Prerequisites

- The agent already has access to Snowflake (snowhouse). No additional connection setup is required.

## Workflow

### Step 1: Identify Target Databases

- The user provides one or more Snowflake database names to analyze.
- If the user does not specify any databases, the agent **must** ask the user which database(s) to work with before proceeding.

### Step 2: Understand the Investigation Goal

- Ask the user what they want to investigate or look for in the database.
- The user may provide detailed context (e.g., specific tables, columns, metrics, time ranges, or known issues) or they may provide very little.
- Gently let the user know that **the more details they provide, the more targeted and useful the analysis will be** — but do not block progress if they have nothing specific in mind.
- If the user has no specific goal, proceed with a broad exploratory analysis.

### Step 3: Verify Database Access

- Before doing any analysis, attempt to access the specified database(s).
- If the account does not have access to one or more of the databases, **immediately inform the user** which database(s) are inaccessible and ask them to either:
  - Grant the necessary permissions, or
  - Provide an alternative database to use.
- Do not proceed with analysis on any database the agent cannot access.

### Step 4: Create an Investigation Workspace

- Create a local workspace for the investigation under the skill folder at `investigations/<investigation-name>/`.
- The `<investigation-name>` should be descriptive and include the date, e.g., `2026-03-09_sales_anomalies`.
- The workspace structure should look like:
  ```
  ThePinPointer/
  ├── SKILL.md
  └── investigations/
      └── <investigation-name>/
          ├── streamlit_app.py (Streamlit dashboard)
          ├── data/           (cached query results, CSVs, etc.)
          └── ...
  ```
- All investigation artifacts (scripts, data exports, dashboard files) go inside this workspace.
- The workspace is a general-purpose working directory — the agent is free to create any additional files or subdirectories as needed beyond the ones explicitly named in this skill (e.g., helper scripts, intermediate CSVs, additional markdown notes, config files, etc.).
- Each new investigation gets its own subfolder — never overwrite a previous one.
- Once the workspace is created, initialize a `progress.md` file inside it to track investigation state.
  - Keep it **concise** — just the current step, what's done, what's next, and any blockers. No lengthy summaries.
  - Update it at the end of every step.
  - Read it first at the start of every new conversation turn to resume where you left off.

### Step 5: Explore Data and Build a Semantic Understanding

- Investigate the database(s) thoroughly: schemas, tables, columns, data types, row counts, primary/foreign keys, and relationships between tables.
- Sample data from each relevant table to understand value distributions, ranges, null rates, and common patterns.
- Document everything in a `data_profile.md` file inside the investigation workspace. This document should include:
  - An overview of the database structure (schemas and tables).
  - A description of each table's purpose and how tables relate to each other.
  - Column-level details: data type, example values, nullability, cardinality, and any notable patterns.
  - Key metrics and aggregates (row counts, date ranges, value distributions).
  - Any initial observations or potential data quality issues noticed during exploration.
- This file serves as the foundation for all subsequent analysis — be as detailed as possible.
- If anything is unclear or ambiguous, call it out in the document and ask the user for clarification before proceeding.

### Step 6: Create an Investigation Plan

- Based on the `data_profile.md` and the user's investigation goal (from Step 2), create an `investigation_plan.md` in the workspace.
- If the user provided a specific goal, tailor the plan to that goal. If not, the agent should devise its own investigation plan based on what it learned about the data.
- The plan should describe:
  - What aspects of the data will be analyzed.
  - What types of anomalies or patterns will be looked for.
  - Which tables, columns, and metrics are in scope.
  - The approach and queries the agent intends to run.
- Be flexible — any type of anomaly or analysis is fair game depending on the data and the user's needs.
- **Present the plan to the user and ask for confirmation before proceeding.** The user may refine, add, or remove items from the plan.

### Step 7: Execute the Investigation

- Once the user confirms the plan, execute the analysis using SQL queries against Snowflake.
- Leverage all previously generated artifacts (`data_profile.md`, `investigation_plan.md`) to guide the analysis.
- Follow the user's guidance where provided; otherwise, follow the confirmed investigation plan.
- Record key findings, anomalies, and insights as they are discovered in a `scratch_pad.md` file in the workspace. This is a living document — update it continuously as new findings emerge. Include raw observations, query results, and any hypotheses worth exploring further.

### Step 8: Create a Dashboard PRD

- After the investigation is complete, create a `dashboard_PRD.md` in the workspace.
- This document defines what the Streamlit dashboard should contain, based on the investigation findings. It should include:
  - The key metrics and anomalies to highlight.
  - Proposed charts, visualizations, and their layout.
  - Filters or interactive elements the user should be able to control.
  - A description of each dashboard section and what insight it delivers.
- The goal is a **beautiful, relevant dashboard** that tells the story of the investigation.
- **Present the PRD to the user and ask for confirmation before building.** The user may request changes to the layout, add/remove charts, or adjust focus areas.

### Step 9: Build the Streamlit Dashboard

- For all Streamlit-related development, always use the `developing-with-streamlit` skill as a reference for best practices, patterns, and guidelines.
- Once the user confirms the PRD, build the Streamlit dashboard as `streamlit_app.py` inside the investigation workspace.
- The dashboard should:
  - Be visually polished and modern.
  - Faithfully implement everything described in the confirmed `dashboard_PRD.md`.
  - Use clear titles, labels, and annotations so findings are easy to understand.
  - Include interactivity where appropriate (filters, date selectors, drill-downs).
- Launch the dashboard locally using `streamlit run streamlit_app.py`.
- Use the Playwright MCP to:
  - Navigate to the running Streamlit app.
  - Take screenshots of every page/section of the dashboard.
  - Verify that all charts render correctly, data loads without errors, and interactive elements function as expected.
  - Check visual quality — ensure the layout is clean, text is readable, colors are consistent, and nothing looks broken or misaligned.
- If any issues are found (rendering errors, broken charts, poor layout, slow load times), fix them and re-verify with Playwright until the dashboard meets a high quality bar.
- Only present the dashboard to the user once it has been visually verified and confirmed working. Ask the user if they want to deploy it to Snowflake (Streamlit-in-Snowflake).

### Step 10: Deploy the Dashboard to Snowflake

- After the dashboard is verified locally, ask the user if they want to deploy it to Snowflake (Streamlit-in-Snowflake).
- **Ask the user which database and schema to deploy to.** If they already specified one earlier in the investigation, suggest that as the default.
- **Ask which Snowflake role to use for deployment.** The role must have write access to the target schema and access to the compute pool.

#### Create deployment files

**`pyproject.toml`**: If one already exists from local development, edit it in place. Otherwise create one. Ensure:
  - `requires-python = ">=3.11"` (Only 3.11 is supported in container runtime).
  - `snowflake-connector-python>=3.3.0` is listed as an explicit dependency (on Python 3.12+ the `streamlit[snowflake]` extra silently skips it).
  - Example:
    ```toml
    [project]
    name = "investigation-dashboard"
    version = "0.1.0"
    requires-python = ">=3.11"
    dependencies = [
        "snowflake-connector-python>=3.3.0",
        "streamlit[snowflake]>=1.54.0",
    ]
    ```

**`snowflake.yml`**: Create the deployment manifest in the investigation workspace.
  - Use these defaults:
    - `runtime_name: SYSTEM$ST_CONTAINER_RUNTIME_PY3_11`
    - `compute_pool: STREAMLIT_DEDICATED_POOL_L`
    - `external_access_integrations: [PYPI_ACCESS_INTEGRATION]`
  - Set `database`, `schema`, and `query_warehouse` based on user input.
  - The `artifacts` list must include **every file the app needs**: `streamlit_app.py`, `pyproject.toml`, `snowflake.yml`, and any additional modules or data files.
  - Example:
    ```yaml
    definition_version: 2
    entities:
      dashboard:
        type: streamlit
        identifier:
          name: <APP_NAME>
          database: <USER_DATABASE>
          schema: <USER_SCHEMA>
        query_warehouse: SNOWADHOC
        runtime_name: SYSTEM$ST_CONTAINER_RUNTIME_PY3_11
        compute_pool: STREAMLIT_DEDICATED_POOL_L
        external_access_integrations:
          - PYPI_ACCESS_INTEGRATION
        main_file: streamlit_app.py
        artifacts:
          - streamlit_app.py
          - pyproject.toml
          - snowflake.yml
    ```

#### Requirements for deployment setup
- Do **not** include `.streamlit/secrets.toml` in the artifacts list (it contains local credentials).
- The main entry point **must** be named `streamlit_app.py`.

#### Run Deploy
- Run `snow streamlit deploy --replace --connection default --role <ROLE>` from the investigation workspace.
- Once deployed, share the Snowsight URL with the user.
