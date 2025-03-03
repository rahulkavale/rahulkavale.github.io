---
layout: post
title: "Balancing Act: Using Structured Output with Streaming in LLMs"
date: 2024-12-26
categories: [llm, ai]
tags: [llm, structured output, streaming, latency]
---

# Balancing Act: Using Structured Output with Streaming in LLMs

When building applications with LLMs, one of a very important trade off comes is - how to use structured responses while using streaming responses.
This choice significantly impacts user experience, system architecture, and overall application performance.

## The Core Challenge

Streaming and structured outputs represent two powerful but somewhat contradictory capabilities of modern LLMs
The tradeoff here is that
1. If you choose streaming responses, you can change the latency to be the `time take for first token` ie lower apparent latency from user perspective. However, you cant get structured output as you need complete response to parse the structure in response. This means streaming needs buffering which means we loose the streaming property
2. If we choose for structured response, we can get parsable output but then this adds latency as you can't parse half a JSON object! This results in bad experience for the end user.

I have tried different approaches for addressing this, each with its own tradeoffs.
Let's explore those -

## Approach 1: Parallel Calls for Streaming and Structure

### Implementation
In this approach, we make two separate, simultaneous calls to the LLM:
- One call requests a streaming, human-readable response
- Another call requests a structured output (like JSON)

```python
# This is a Pseudocode
import asyncio

async def handle_query(user_query):
    # Create two tasks to run in parallel
    streaming_task = asyncio.create_task(llm.generate_streaming(user_query))
    structured_task = asyncio.create_task(llm.generate_structured(user_query, format="json"))

    # Start streaming immediately
    streaming_response = await streaming_task
    stream_to_user(streaming_response)

    # Process structured data when available
    structured_data = await structured_task
    process_structured_output(structured_data)
```

### Advantages
- **Low Perceived Latency**: Users see responses immediately via streaming
- **No Blocking**: Structured output processing doesn't block the streaming experience

### Disadvantages
- **Divergent Outputs**: The streaming and structured outputs might contain different information since they're generated independently
- **Resource Intensive**: Requires twice the LLM calls increasing usage, cost
- **Consistency Challenges**: Maintaining consistency between the two outputs can be difficult

## Approach 2: Single Call with Streamed Separation

### Implementation
In this pattern, we make a single call but ask the LLM to include both a conversational response and a structured output in the same generation, typically with the structured data at the end.
Since the structured output is at the end of the response, we can keep on streaming the initial part of the response.
Once we get a json marker in response, we can accumulate the remaining response and parse it.
This means we cant use standard JSON modes for response(or instructor like libraries).

```python
# This is a Pseudocode
async def handle_query(user_query):
    response = await llm.generate_streaming(
        user_query,
        instruction="Provide a conversational response followed by JSON data at the end"
    )

    text_content = ""
    json_content = ""
    json_started = False

    async for chunk in response:
        if "```json" in chunk or json_started:
            json_started = True
            json_content += chunk
        else:
            text_content += chunk
            stream_to_user(chunk)

    # Process complete JSON at the end
    structured_data = json.loads(json_content.replace("```json", "").replace("```", "").strip())
    process_structured_output(structured_data)
```
One problem here is that, even though we ask LLM to include structured content at the end, there is no guarantee that it will in fact be at the end.
It can very well appear at the beginning. In that case the parsing logic fails.

### Advantages
- **Alignment Guaranteed**: Text and structured outputs come from the same generation, so the two outputs will be aligned
- **Resource Efficient**: Only requires one LLM call

### Disadvantages
- **Latency Issues**: Users must wait for the entire response before structured data processing can begin
- **Parsing Complexity**: Requires robust parsing logic to separate the structured portion.
- **Response Format Sensitivity**: This assumes that LLM will include the structured part and streaming part correctly in the response

## Approach 3: Sequential Calls

### Implementation
This approach makes two sequential calls:
1. First, generate and stream the conversational response
2. After completion, make a second call to generate the structured data

```python
# This is a Pseudocode
async def handle_query(user_query):
    # First call for streaming text
    text_response = await llm.generate_streaming(user_query)
    stream_to_user(text_response)

    # Second call for structured data
    structured_prompt = f'Based on the query "{user_query}" and your response "{text_response}", generate a structured JSON representation.'
    structured_data = await llm.generate_structured(structured_prompt, format="json")
    process_structured_output(structured_data)
```

### Advantages
- **Better Aligned Outputs**: Second call has access to the first response, improving consistency
- **Clean Separation**: Streaming and structured processing are fully separated, no fragile parsing of outputs

### Disadvantages
- **Higher Latency**: Sequential calls mean waiting for the first to complete before starting the second
- **Longer Context**: Second call requires including the first response, which uses more tokens
- **Cost**: Overall cost is doubled as now we have two separate LLM calls

## Approach 4: Sequential Calls with Model Combination
We can simplify the above approach a bit more to address the latency aspect, lets explore how.

### Implementation
This refined approach uses different models for different parts of the process:
1. Use a large, powerful model for the streaming conversational response
2. Use a smaller, faster model for generating the structured output. This step works because we include the already streamed response to the smaller model.

```python
# This is a Pseudocode
async def handle_query(user_query):
    # Large model for streaming text
    text_response = await large_model.generate_streaming(user_query)
    stream_to_user(text_response)

    # Small model for structured data
    structured_prompt = f'Based on the query "{user_query}" and the response "{text_response}", generate a structured JSON representation.'
    structured_data = await small_model.generate_structured(structured_prompt, format="json")
    process_structured_output(structured_data)
```

### Advantages
- **Better Aligned Outputs**: Second call has access to the first response, hence the outputs from both calls are aligned
- **Clean Separation**: Streaming and structured processing are fully separated, no fragile parsing of outputs
- **Better Performance**: Smaller models can generate structured outputs faster

### Disadvantages
- **Potential Capability Gap**: Smaller model might not understand complex aspects of the main response
- **Cost**: Overall cost is still higher than single call but its not double, so a good tradeoff.

## Conclusion

Balancing streaming and structured outputs in LLM integration presents a very interesting design tradeoff between apparent latency, consistency, and implementation complexity.
However, we can make some integration changes to keep lower latency while using structured responses.

The area is evolving very fast, who knows if future LLM innovations may better address this fundamental tension and allow separate handles for stream and structured data in output.

What approach are you using in your LLM applications? Have you discovered other patterns that work well? Share your experiences in the comments!
