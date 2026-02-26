# HealthGPT: LLM Tool-Use Implementation Guide

## The Problem

Today, HealthGPT fetches 6 fixed metrics for a fixed 14-day window, serializes everything into the system prompt, and hopes the LLM can answer whatever the user asks. This means:

- The LLM **cannot** access data older than 14 days
- The LLM **cannot** query metrics beyond the 6 hardcoded ones
- Every session fetches **all** data even if the user only asks about steps
- The system prompt is bloated with data the user may never ask about
- Data goes stale the moment the session starts

## The Solution

Replace the static data dump with **LLM function calling** (tool use). The LLM decides what data it needs based on the user's question, calls tools to fetch it from HealthKit on demand, and reasons over the results.

```
BEFORE:  App start → fetch ALL data → dump into prompt → chat (static)
AFTER:   User asks question → LLM calls tools → fetch RELEVANT data → LLM responds (dynamic)
```

## Compatibility Matrix

| LLM Backend | Function Calling Support | Strategy |
|-------------|------------------------|----------|
| **OpenAI** (`SpeziLLMOpenAI`) | Full native support via `LLMFunction` protocol | Use tool-based architecture |
| **Fog** (`SpeziLLMFog`) | Not supported in SpeziLLM v0.12.3 | Fall back to current system-prompt approach |
| **Local** (`SpeziLLMLocal`) | Not supported | Fall back to current system-prompt approach |
| **Mock** | Not supported | Continue using mock responses |

> Fog and Local backends keep working exactly as they do today. Tool use is an OpenAI-only enhancement.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   HealthGPTView                      │
│  Creates LLMOpenAISchema with @LLMFunctionBuilder   │
└──────────────────────┬──────────────────────────────┘
                       │ schema
                       ▼
┌─────────────────────────────────────────────────────┐
│              HealthDataInterpreter                   │
│  prepareLLM(with:) → LLMSession                     │
│  queryLLM() → generate loop (auto-handles tools)    │
└──────────────────────┬──────────────────────────────┘
                       │ LLM calls a tool
                       ▼
┌─────────────────────────────────────────────────────┐
│           LLMFunction implementations                │
│  GetHealthMetricFunction  │  GetSleepDataFunction    │
│  GetUserProfileFunction   │  ComparePeriodsFunction  │
│                                                      │
│  Each function holds a reference to                  │
│  HealthDataFetcher and calls it in execute()         │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              HealthDataFetcher (expanded)             │
│  New: flexible date ranges, more HK types            │
│  Existing: fetchLastTwoWeeks* methods preserved      │
└─────────────────────────────────────────────────────┘
```

The key insight: **SpeziLLMOpenAI already handles the entire tool-calling loop**. When the LLM returns a function call, the framework automatically:
1. Parses the function name and arguments
2. Injects `@Parameter` values into the function struct
3. Calls `execute()` (in parallel if multiple tools are called)
4. Appends results to the conversation context
5. Re-queries the LLM so it can reason over the results

You don't need to write any dispatch or retry logic.

---

## Step-by-Step Implementation

### Step 1: Expand `HealthDataFetcher` with Flexible Queries

**File:** `HealthGPT/HealthGPT/HealthDataFetcher.swift`

The current fetcher is hardcoded to 14-day windows. Add a generic method that accepts any date range.

```swift
/// Fetch daily aggregated data for any quantity type over any date range.
func fetchQuantityData(
    for identifier: HKQuantityTypeIdentifier,
    unit: HKUnit,
    options: HKStatisticsOptions,
    from startDate: Date,
    to endDate: Date
) async throws -> [(date: Date, value: Double)] {
    let quantityType = HKQuantityType(identifier)
    let predicate = HKQuery.predicateForSamples(
        withStart: startDate,
        end: endDate,
        options: .strictStartDate
    )
    let interval = DateComponents(day: 1)
    let query = HKStatisticsCollectionQueryDescriptor(
        predicate: .quantitySample(type: quantityType, predicate: predicate),
        options: options,
        anchorDate: startDate.startOfDay,
        intervalComponents: interval
    )
    let collection = try await query.result(for: healthStore)

    var results: [(date: Date, value: Double)] = []
    collection.enumerateStatistics(from: startDate.startOfDay, to: endDate) { statistics, _ in
        let value: Double
        switch options {
        case .cumulativeSum:
            value = statistics.sumQuantity()?.doubleValue(for: unit) ?? 0
        case .discreteAverage:
            value = statistics.averageQuantity()?.doubleValue(for: unit) ?? 0
        default:
            value = 0
        }
        results.append((date: statistics.startDate, value: value))
    }
    return results
}

/// Fetch sleep data over any date range (using 3PM-3PM windows).
func fetchSleepData(
    from startDate: Date,
    to endDate: Date
) async throws -> [(date: Date, hours: Double)] {
    // Same 3PM-3PM logic as fetchLastTwoWeeksSleep(),
    // but parameterized over the date range.
    // ...
}
```

**Keep the existing `fetchLastTwoWeeks*` methods.** They are still used by the Fog/Local fallback path. The new methods are additive.

Also expose a **mapping from string metric names to HK query parameters**, since the LLM will pass metric names as strings:

```swift
enum HealthMetric: String, CaseIterable {
    case steps, activeEnergy, exerciseMinutes, bodyWeight, restingHeartRate, sleep

    var identifier: HKQuantityTypeIdentifier? {
        switch self {
        case .steps: return .stepCount
        case .activeEnergy: return .activeEnergyBurned
        case .exerciseMinutes: return .appleExerciseTime
        case .bodyWeight: return .bodyMass
        case .restingHeartRate: return .restingHeartRate
        case .sleep: return nil // sleep uses category queries
        }
    }

    var unit: HKUnit? { /* ... */ }
    var options: HKStatisticsOptions? { /* ... */ }
    var displayName: String { /* ... */ }
}
```

---

### Step 2: Define LLM Functions

**New file:** `HealthGPT/HealthGPT/Functions/GetHealthMetricFunction.swift`

This is the primary tool. One function handles all 6 metrics, reducing token overhead vs. 6 separate function definitions.

```swift
import SpeziLLMOpenAI

struct GetHealthMetricFunction: LLMFunction {
    static let name = "get_health_metric"
    static let description = """
        Retrieve a specific health metric for the user over a given number of recent days. \
        Returns daily values. Available metrics: steps, activeEnergy, exerciseMinutes, \
        bodyWeight, restingHeartRate, sleep.
        """

    // Enum parameter — SpeziLLM auto-generates the JSON Schema enum constraint
    enum MetricType: String, LLMFunctionParameterEnum {
        case steps
        case activeEnergy
        case exerciseMinutes
        case bodyWeight
        case restingHeartRate
        case sleep
    }

    let healthDataFetcher: HealthDataFetcher

    @Parameter(description: "The health metric to retrieve")
    var metric: MetricType

    @Parameter(description: "Number of days to look back from today", minimum: 1, maximum: 90)
    var days: Int

    func execute() async throws -> String? {
        let endDate = Date()
        let startDate = Calendar.current.date(byAdding: .day, value: -days, to: endDate)!
        let healthMetric = HealthMetric(rawValue: metric.rawValue)!

        if healthMetric == .sleep {
            let data = try await healthDataFetcher.fetchSleepData(from: startDate, to: endDate)
            return formatSleepResults(data)
        } else {
            let data = try await healthDataFetcher.fetchQuantityData(
                for: healthMetric.identifier!,
                unit: healthMetric.unit!,
                options: healthMetric.options!,
                from: startDate,
                to: endDate
            )
            return formatQuantityResults(data, metric: healthMetric)
        }
    }

    private func formatQuantityResults(
        _ data: [(date: Date, value: Double)],
        metric: HealthMetric
    ) -> String {
        let formatter = DateFormatter()
        formatter.dateStyle = .short

        let lines = data.map { "\(formatter.string(from: $0.date)): \($0.value) \(metric.displayName)" }
        return lines.joined(separator: "\n")
    }

    private func formatSleepResults(_ data: [(date: Date, hours: Double)]) -> String {
        let formatter = DateFormatter()
        formatter.dateStyle = .short

        let lines = data.map { "\(formatter.string(from: $0.date)): \($0.hours) hours of sleep" }
        return lines.joined(separator: "\n")
    }
}
```

**New file:** `HealthGPT/HealthGPT/Functions/GetAvailableMetricsFunction.swift`

A lightweight function so the LLM knows what it can query.

```swift
struct GetAvailableMetricsFunction: LLMFunction {
    static let name = "get_available_metrics"
    static let description = "List all health metrics available to query and their descriptions."

    func execute() async throws -> String? {
        """
        Available metrics:
        - steps: Daily step count
        - activeEnergy: Active calories burned (kcal)
        - exerciseMinutes: Minutes of exercise
        - bodyWeight: Body weight (lbs)
        - restingHeartRate: Resting heart rate (bpm)
        - sleep: Total hours of sleep per night

        Use get_health_metric to retrieve any of these for up to 90 days.
        """
    }
}
```

**Optional future function:** `ComparePeriodsFunction.swift`

```swift
struct ComparePeriodsFunction: LLMFunction {
    static let name = "compare_periods"
    static let description = """
        Compare a health metric between two time periods. \
        Useful for questions like 'Am I sleeping more this week vs last week?'
        """

    let healthDataFetcher: HealthDataFetcher

    @Parameter(description: "The health metric to compare")
    var metric: GetHealthMetricFunction.MetricType

    @Parameter(description: "Start of first period as days ago from today")
    var period1Start: Int

    @Parameter(description: "End of first period as days ago from today")
    var period1End: Int

    @Parameter(description: "Start of second period as days ago from today")
    var period2Start: Int

    @Parameter(description: "End of second period as days ago from today")
    var period2End: Int

    func execute() async throws -> String? {
        // Fetch both periods, compute averages, return comparison
    }
}
```

---

### Step 3: Rewrite the System Prompt

**File:** `HealthGPT/HealthGPT/PromptGenerator.swift`

For OpenAI mode, the system prompt should instruct the LLM to use tools rather than containing the data itself.

```swift
class PromptGenerator {
    var healthData: [HealthData]

    init(with healthData: [HealthData] = []) {
        self.healthData = healthData
    }

    /// System prompt for tool-use mode (OpenAI).
    /// Contains NO health data — the LLM fetches what it needs via tools.
    static func buildToolUsePrompt() -> String {
        let today = Date.now.formatted(date: .abbreviated, time: .omitted)
        return """
            You are HealthGPT, an expert and friendly health assistant with access to the \
            user's Apple Health data.

            Today is \(today).

            INSTRUCTIONS:
            - When the user asks about their health, use the get_health_metric tool to fetch \
            the relevant data. Do NOT guess or assume values.
            - Fetch only the metrics and time range relevant to the question. For example, if \
            they ask about sleep this week, fetch sleep for 7 days.
            - You can call the tool multiple times to get different metrics if needed.
            - After receiving data, analyze it and provide clear, actionable insights.
            - If values are zero or missing for a day, it likely means the user didn't wear \
            their device or didn't log data — mention this possibility.
            - Be concise but thorough. Use specific numbers from the data.
            - When comparing periods, fetch both and highlight meaningful differences.
            - Do not provide medical diagnoses. Suggest consulting a doctor for concerns.
            """
    }

    /// System prompt for non-tool-use mode (Fog/Local).
    /// Contains the full 14-day data dump, same as today.
    func buildMainPrompt() -> String {
        // Keep the existing implementation exactly as-is
        // ...
    }
}
```

---

### Step 4: Modify `HealthDataInterpreter`

**File:** `HealthGPT/HealthGPT/HealthDataInterpreter.swift`

The interpreter needs two paths: one for tool-use schemas (OpenAI) and one for the legacy data-dump approach (Fog/Local).

```swift
@Observable
class HealthDataInterpreter: DefaultInitializable, Module, EnvironmentAccessible {
    @ObservationIgnored @Dependency(LLMRunner.self) private var llmRunner
    @ObservationIgnored @Dependency(HealthDataFetcher.self) private var healthDataFetcher

    var llm: (any LLMSession)?
    @ObservationIgnored private var systemPrompt = ""

    required init() { }

    /// Prepare LLM for tool-use mode (OpenAI).
    /// No health data is fetched upfront — the LLM will fetch on demand via tools.
    @MainActor
    func prepareLLMWithTools(with schema: LLMOpenAISchema) async throws {
        llm = llmRunner(with: schema)
        systemPrompt = PromptGenerator.buildToolUsePrompt()
        llm?.context.append(systemMessage: systemPrompt)
    }

    /// Prepare LLM for legacy mode (Fog/Local/Mock).
    /// Fetches all health data upfront and dumps it into the system prompt.
    @MainActor
    func prepareLLM(with schema: any LLMSchema) async throws {
        llm = llmRunner(with: schema)
        systemPrompt = await generateSystemPrompt()
        llm?.context.append(systemMessage: systemPrompt)
    }

    // queryLLM() and resetChat() remain the same — the SpeziLLM framework
    // handles tool call dispatch automatically within the generate() loop.

    @MainActor
    func queryLLM() async throws {
        // No changes needed here. When LLMOpenAISession.generate() encounters
        // a function call in the response, it automatically:
        // 1. Executes the function
        // 2. Appends results to context
        // 3. Re-queries the LLM
        // The streaming tokens you receive are the final text response.
        guard let llm, llm.context.last?.role == .user else { return }

        let stream = try await llm.generate()
        for try await token in stream {
            llm.context.append(assistantOutput: token)
        }
    }

    @MainActor
    func resetChat() async {
        if llm is LLMOpenAISession {
            // Tool-use mode: just reset context with the static prompt
            llm?.context = .init()
            llm?.context.append(systemMessage: systemPrompt)
        } else {
            // Legacy mode: re-fetch health data
            systemPrompt = await generateSystemPrompt()
            llm?.context = .init()
            llm?.context.append(systemMessage: systemPrompt)
        }
    }

    private func generateSystemPrompt() async -> String {
        let healthData = await healthDataFetcher.fetchAndProcessHealthData()
        return PromptGenerator(with: healthData).buildMainPrompt()
    }
}
```

> **Note:** You may need to check `LLMOpenAISession` visibility or use a flag rather than a type check. Adjust based on what SpeziLLM exports.

---

### Step 5: Wire Up Functions in `HealthGPTView`

**File:** `HealthGPT/HealthGPT/HealthGPTView.swift`

The schema construction in `.task(id:)` is where functions get registered.

```swift
.task(id: self.modelSettingRefreshId) {
    do {
        if FeatureFlags.mockMode {
            try await healthDataInterpreter.prepareLLM(with: LLMMockSchema())
        } else if FeatureFlags.localLLM || llmSource == .local {
            try await healthDataInterpreter.prepareLLM(
                with: LLMLocalSchema(model: .llama3_2_3B_4bit)
            )
        } else if llmSource == .fog {
            try await healthDataInterpreter.prepareLLM(
                with: LLMFogSchema(parameters: .init(modelType: self.fogModel))
            )
        } else {
            // OpenAI mode — with function calling
            let schema = LLMOpenAISchema(
                parameters: .init(modelType: openAIModel)
            ) {
                GetHealthMetricFunction(healthDataFetcher: healthDataFetcher)
                GetAvailableMetricsFunction()
                // Add more functions here as needed:
                // ComparePeriodsFunction(healthDataFetcher: healthDataFetcher)
            }
            try await healthDataInterpreter.prepareLLMWithTools(with: schema)
        }
    } catch {
        // handle error
    }
}
```

`healthDataFetcher` is available in the view via Spezi's `@Environment`:

```swift
@Environment(HealthDataFetcher.self) private var healthDataFetcher
```

---

### Step 6: Handle the `.callingTools` State in UI (Optional Enhancement)

`SpeziLLMOpenAI` sets `llm.state` to `.callingTools` while functions execute. You can show a loading indicator:

```swift
// In HealthGPTView body
if case .callingTools = healthDataInterpreter.llm?.state {
    HStack {
        ProgressView()
        Text("Fetching your health data...")
    }
}
```

---

### Step 7: Expand HealthKit Permissions (If Adding New Metrics Later)

**File:** `HealthGPT/HealthGPTAppDelegate.swift`

To support more metrics via tool use, add them to the HealthKit permission request:

```swift
private var healthKit: HealthKit {
    HealthKit {
        RequestReadAccess(
            quantity: [
                .activeEnergyBurned,
                .appleExerciseTime,
                .bodyMass,
                .heartRate,
                .restingHeartRate,  // Add this — currently missing but used by fetcher
                .stepCount,
                // Future expansions:
                // .heartRateVariabilitySDNN,
                // .vo2Max,
                // .bloodGlucose,
                // .bloodPressureSystolic,
                // .bloodPressureDiastolic,
                // .respiratoryRate,
                // .bodyMassIndex,
            ]
        )
        RequestReadAccess(category: [
            .sleepAnalysis,
            // Future: .mindfulSession
        ])
    }
}
```

> **Bug fix:** The current app requests `.heartRate` permission but queries `.restingHeartRate` in the fetcher. Add `.restingHeartRate` to the permission list.

---

### Step 8: Update Tests

**File:** `HealthGPTTests/PromptGeneratorTests.swift`

Add tests for the new tool-use system prompt:

```swift
@Test
func testToolUsePromptContainsInstructions() {
    let prompt = PromptGenerator.buildToolUsePrompt()
    #expect(prompt.contains("get_health_metric"))
    #expect(prompt.contains("HealthGPT"))
    #expect(!prompt.contains("steps")) // Should NOT contain data
}
```

Add tests for `GetHealthMetricFunction` with a mock `HealthDataFetcher` if feasible.

---

## File Change Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `HealthDataFetcher.swift` | **Modify** | Add `fetchQuantityData(for:unit:options:from:to:)` and `fetchSleepData(from:to:)` with flexible date ranges. Add `HealthMetric` enum. |
| `PromptGenerator.swift` | **Modify** | Add `static func buildToolUsePrompt() -> String`. Keep `buildMainPrompt()` unchanged. |
| `HealthDataInterpreter.swift` | **Modify** | Add `prepareLLMWithTools(with:)` method. Update `resetChat()` to handle both paths. |
| `HealthGPTView.swift` | **Modify** | Construct `LLMOpenAISchema` with `@LLMFunctionBuilder` closure for OpenAI mode. |
| `HealthGPTAppDelegate.swift` | **Modify** | Add `.restingHeartRate` to HealthKit permissions (bug fix). |
| `Functions/GetHealthMetricFunction.swift` | **New** | Primary tool: fetches any metric for any date range. |
| `Functions/GetAvailableMetricsFunction.swift` | **New** | Informational tool: lists available metrics. |
| `Functions/ComparePeriodsFunction.swift` | **New** (optional) | Comparison tool for "this week vs last week" questions. |
| `PromptGeneratorTests.swift` | **Modify** | Add test for tool-use prompt. |

---

## Implementation Order

```
1. HealthDataFetcher — flexible date range queries       (foundation, no breakage)
2. HealthMetric enum                                      (used by functions)
3. GetHealthMetricFunction + GetAvailableMetricsFunction  (core tool-use logic)
4. PromptGenerator.buildToolUsePrompt()                   (new static method)
5. HealthDataInterpreter.prepareLLMWithTools()             (new method)
6. HealthGPTView — wire up schema with functions           (connects everything)
7. HealthGPTAppDelegate — fix .restingHeartRate permission (bug fix)
8. UI: .callingTools state indicator                       (polish)
9. Tests                                                   (validation)
```

Each step is independently testable. Steps 1-2 don't change existing behavior at all. Steps 3-5 add new code paths. Step 6 is the single switch that activates tool use for OpenAI mode. Fog/Local/Mock paths are never touched.

---

## What This Unlocks

Once this foundation is in place, expanding becomes trivial:

- **New metric?** Add a case to `HealthMetric`, request the HK permission, done. The LLM can immediately query it.
- **Longer history?** Change the `maximum` on the `days` parameter from 90 to 365.
- **User profile context?** Add a `GetUserProfileFunction` that returns age, sex, height from HealthKit characteristics.
- **Trend analysis?** Add a `GetWeeklyAveragesFunction` that returns weekly aggregates for month-over-month comparisons.
- **Workout details?** Add a `GetWorkoutsFunction` that queries `HKWorkout` samples.
- **Medication tracking?** Add a function that queries `HKClinicalRecord` or medication logs.

The architecture becomes **open for extension without modifying existing code** — each new capability is just a new `LLMFunction` struct dropped into the builder closure.
