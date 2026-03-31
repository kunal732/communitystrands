# Bringing AWS Strands Agents to Swift

<figure style="max-width:200px;margin:0 auto 2rem">
<img src="https://raw.githubusercontent.com/kunal732/communitystrands/master/blog/images/reinvent-talk.png" alt="Kunal Batra and Du'An Lightfoot at AWS re:Invent" style="width:100%;border-radius:10px">
<figcaption style="text-align:center;font-size:13px;color:#9399b2;margin-top:0.5rem">Myself and Du'An Lightfoot giving a talk at re:Invent on building agents with AWS Strands</figcaption>
</figure>

The past year has been an incredible time to be a developer. Agents are quickly moving from demos into production systems people actually rely on.

One framework I've really enjoyed building with during that shift is AWS's [Strands Agents SDK](https://github.com/strands-agents). What stood out to me was how quickly it fit into the systems I already had. I could go from an idea to a working agent and integrate it directly with services like SQS, Lambda, DynamoDB, and S3 without having to build additional orchestration or integration layers. It also emits OpenTelemetry traces out of the box using the GenAI semantic conventions, which makes it easier to verify behavior, monitor performance, and catch issues early.

While building with the Python SDK, it became clear that these agentic workflows shouldn't be limited to backend systems. They can be packaged and distributed directly within the applications users operate in every day, which is what led me to bring this experience into the Swift ecosystem.

## Introducing the Swift Strands Agents SDK

The [Strands Agents Swift SDK](https://github.com/kunal732/strands-agents-swift) is a community port of the framework for macOS, iOS, and tvOS. It keeps the same core model, agents, tools, and reasoning loop, but adapts it to a very different environment: the device.

It's a single Swift 6 package targeting macOS 14+, iOS 17+, and tvOS 17+. One import, no macros, no Xcode trust prompts.

You define tools as normal Swift functions, wrap them with `Tool()`, and the agent handles the rest.

```swift
import StrandsAgents

func wordCount(text: String) -> Int {
    text.split(whereSeparator: \.isWhitespace).count
}

let wordCountTool = Tool(wordCount, "Count the number of words in a block of text.")

let agent = Agent(
    model: MLXProvider(modelId: "mlx-community/Qwen3-8B-4bit"),
    tools: [wordCountTool]
)

for try await text in agent.streamText("How many words are in the Gettysburg Address?") {
    print(text, terminator: "")
}
```

## Running in the Cloud or On-Device

The SDK ships with two providers, both safe for production apps.

**AWS Bedrock** handles cloud inference using Amplify and Cognito to issue temporary, scoped credentials. No API keys ever live in your binary.

**MLX** handles on-device inference on Apple Silicon. No network calls, no credentials, and no data leaves the device. Models are pulled from HuggingFace and cached locally on first run.

Providers that require embedding permanent API keys are intentionally excluded.

## Hybrid Routing

A `HybridRouter` sits between your agent and its providers, choosing local or cloud per request based on real device signals like available memory, thermal state, power source, and observed latency.

```swift
let agent = Agent(
    router: HybridRouter(
        local: MLXProvider(modelId: "mlx-community/Qwen3-8B-4bit"),
        cloud: try BedrockProvider(config: BedrockConfig(
            modelId: "us.anthropic.claude-sonnet-4-20250514-v1:0"
        )),
        policy: LatencySensitivePolicy()
    )
)
```

## New Possibilities on Apple Devices

Running agents natively on Apple Silicon opens up use cases that don't really exist server-side.

You can run fully local agents that operate over private data without any network calls. You can mix local and cloud inference depending on context. And you can enable computer-use style workflows where the model interacts directly with the system.

In the demo below, I connected an agent to a local Mac MCP server and had it open Word and write a sentence from a menu bar app. At the end, you can see the full trace in Datadog, showing every tool invocation and span.

<div class="video-embed">
<iframe width="100%" height="400" src="https://www.youtube.com/embed/py8PAScf2EQ" frameborder="0" allowfullscreen></iframe>
</div>

## Observability Still Matters

Even on-device, the same principles apply. You still need to understand what your agent is doing.

Every agent invocation produces a full OpenTelemetry trace, with a hierarchy like:

```
invoke_agent
  └── event_loop_cycle
        ├── chat
        └── execute_tool
```

Spans follow the GenAI semantic conventions, capturing input/output messages, token counts, latency, and tool execution details. You can inspect traces locally or export them to any OTLP-compatible backend, including Datadog.

## Getting Started

Add the package to your `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/kunal732/strands-agents-swift.git", branch: "main")
]
```

Full documentation: [communitystrands.com/swift](https://communitystrands.com/swift/)

Apache 2.0 licensed. Contributions welcome.
