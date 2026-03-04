# Generic Flink MCP Testing Context

## Goal
This repository may contain one or more **Apache Flink applications**. The purpose of this context file is to guide an AI agent (such as Claude Code) to build and operate an **MCP (Model Context Protocol) testing server** that can automatically test and fine‑tune **event‑time behavior and out‑of‑order event handling** in Flink streaming jobs.

The MCP server should orchestrate a **deterministic local test environment** capable of:

1. Spinning up **Kafka locally** (preferably using Testcontainers).
2. Spinning up a **Flink MiniCluster**.
3. Submitting one of the Flink jobs from this repository.
4. Producing controlled test events into the job's **Kafka source topics**.
5. Generating **ordered and out‑of‑order event patterns**.
6. Consuming the **output Kafka topic(s)**.
7. Producing structured reports describing correctness, lateness behavior, and watermark progression.

The primary purpose is to help engineers **debug, test, and tune watermarking and event‑time behavior** in Flink applications.

---

# Repository Structure

The MCP server and testing harness should live inside the repository but must be isolated from production code.

Recommended structure:

```
tools/
  mcp-flink-test/
      server/
      scenarios/
      runner/
      DISCOVERY.md
```

The MCP implementation must:

• not interfere with production builds
• not introduce runtime dependencies for production code
• be runnable independently

---

# Primary Testing Objectives

This MCP framework should support testing scenarios such as:

• events arriving in correct order
• events arriving slightly out of order
• events arriving heavily out of order
• events arriving beyond allowed lateness
• watermark propagation behavior
• stateful operator correctness
• joins across multiple streams

The framework must **adapt dynamically to each Flink job** by discovering its sources and sinks.

A job may have:

• multiple Kafka source streams
• multiple sinks
• joins or co‑process operators

The MCP harness must automatically determine these through **code discovery**.

---

# Required MCP Tools

The MCP server should expose the following tools.

## 1. `flink_test.run_scenario`

Runs a complete end‑to‑end test scenario.

Inputs:

```
scenario_id
parallelism
watermark_profile
out_of_order_profile
allowed_lateness_ms
max_out_of_orderness_ms
records_per_stream
seed
```

Responsibilities:

• Start Kafka
• Start Flink MiniCluster
• Submit the selected Flink job
• Generate event scenarios
• Produce events into discovered source topics
• Consume output topics
• Evaluate correctness

Output:

Structured JSON report including:

```
run_id
pass
output_counts
late_event_counts
watermark_progression
logs
artifacts
```

---

## 2. `flink_test.generate_plan`

Generates deterministic event scenarios used for testing.

Example output:

```
{
  "streams": [
    {
      "topic": "topicA",
      "events": [
        {
          "key": "123",
          "event_time_ms": 1700000000000,
          "produce_delay_ms": 0,
          "payload": {}
        }
      ]
    }
  ]
}
```

The event plan should control:

• event timestamps
• production delays
• ordering patterns
• key distribution
• correlations between streams

---

## 3. `flink_test.get_last_run_artifacts`

Returns artifacts from the most recent scenario execution.

Artifacts may include:

• event plan
• logs
• consumed output samples
• summary report

---

## 4. `flink_test.discover_stream_contracts`

This tool performs **automated code discovery** to understand how the Flink job interacts with its streams.

It must inspect the repository and determine:

For each **Kafka source stream**:

• topic name
• serialization format (JSON / Avro / Protobuf)
• event‑time field
• watermark strategy
• keyBy logic
• partitioning assumptions
• join relationships
• late event handling

For each **sink**:

• topic or storage destination
• expected output schema

The results must be written to:

```
tools/mcp-flink-test/DISCOVERY.md
```

and returned as structured JSON.

---

# Discovery Requirements

Before generating scenarios, the system must inspect the repository to determine the **true stream contracts**.

The discovery step must:

1. Locate all Kafka source definitions.
2. Identify deserializers.
3. Identify event‑time extraction logic.
4. Identify watermark strategies.
5. Identify `keyBy` operations.
6. Identify joins between streams.
7. Identify side outputs for late events.

The system must **never invent schemas or topic structures**.

Only information that exists in code or configuration should be used.

---

# Scenario Generation Principles

Event scenarios should reflect realistic streaming patterns.

Examples:

### In‑order events

All events arrive with increasing timestamps.

### Slight out‑of‑order

Events arrive within the allowed watermark bounds.

### Heavy out‑of‑order

Events arrive in a highly shuffled order.

### Late beyond allowed

Events arrive after the allowed lateness window.

---

# Multi‑Stream Scenario Design

Many Flink jobs involve joins or keyed state.

Scenario generation should therefore:

• reuse keys across multiple streams
• simulate realistic partition distributions
• stress stateful operators
• test join alignment under out‑of‑order arrival

---

# Runtime Environment

Preferred local runtime:

• **Testcontainers Kafka**
• **Flink MiniCluster**

The system should avoid external dependencies and ensure that tests are **deterministic and reproducible**.

---

# Deliverables

The MCP framework should provide:

1. MCP server implementation
2. scenario runner
3. event plan generator
4. automated stream discovery
5. structured test reports
6. documentation for adding new scenarios

---

# Long‑Term Vision

This MCP server should evolve into a **general debugging interface for Flink applications**.

It should allow AI agents to:

• run controlled streaming experiments
• inspect state behavior
• debug watermarking issues
• simulate replay scenarios
• validate changes before production deployment

The initial focus is **out‑of‑order event testing**, but the framework should remain flexible enough to support broader streaming diagnostics in the future.