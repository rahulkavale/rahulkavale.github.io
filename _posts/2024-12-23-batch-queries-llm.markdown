---
layout: post
title: "Implementing Batched Queries with LLMs: Handling Large Datasets Efficiently"
date: 2024-12-23
categories: [llm, data-processing]
tags: [llm, batching, large-datasets, python]
---

Large Language Models (LLMs) have become essential tools for various data processing tasks, including data labeling, classification, and enrichment. However, when working with substantial datasets, context windows size starts getting in the way/
Also, as the dataset itself grows, the hallucinations increase too.
Effective way I have found to mitigate this problem is using batched queries with LLM. Lets explore it in details.

## The Challenge of Large Datasets

When using LLMs for tasks like data labeling, several challenges emerge:

- **Context Window Limitations**: Most LLMs have a maximum token limit (context window) that restricts how much data can be processed in a single query.
- **Hallucinations**: LLMs may inadvertently drop datapoints, add unexpected ones, or modify existing data during processing.

Let's explore the approach I followed to address these challenges.

## Implementing a Batched Query System

### 1. Data Preparation and ID Assignment

Before sending data to an LLM, each datapoint should be assigned a numeric identifier. This is a critical first step that allows us to track data throughout the processing pipeline.

These IDs serve multiple critical purposes:
- Tracking which datapoints have been processed
- Identifying dropped or added datapoints
- Enabling proper aggregation of results
- Supporting incremental processing approaches

The implementation is straightforward - simply iterate through your dataset and add an ID field to each datapoint if one doesn't already exist.

### 2. Batching Strategy

Break your dataset into manageable batches.
This helps in addressing the context size limitation. However, it also needs to consider the appropriate batch size.
Too small of a batch and then we end up doing lot of LLM calls.
Too large batch and then we start seeing hallucinations or consistency issues.

Consider these factors when determining optimal batch size:
- LLM context window limits
- Complexity of your datapoints
- Type of processing required
- Cost per token considerations

In practice, you might start with a conservative batch size and adjust based on performance.
Ideally the batch size can be made configurable too.

### 3. Processing and Validation Loop

Implement a robust processing loop that handles data integrity issues. The core logic involves:

1. Processing each batch with the LLM
2. Validating that all datapoints were processed correctly
3. Identifying missing or added datapoints
4. Collecting missed datapoints for later retry to be processed separately

This validation step is crucial as LLMs may occasionally drop datapoints, especially when context windows are nearly full.

### 4. Handling Missed Datapoints

Implement a retry mechanism for missed datapoints. When datapoints are missed in the initial processing:

1. Collect all missed datapoints
2. Process all missed items in same size batches
3. Since this itself can cause items to be dropped, we need to continue doing this till no item is left

With a well-designed retry mechanism, you can often recover nearly 100% of your datapoints, even when dealing with challenging data.

### 5. Incremental Processing with Existing Labels

Because we are processing in iterative way in batches, we may need to keep outputs of previous steps as input for next steps.
Think of it like a fold operation in FP where initially the accumulator is empty, with each iteration, the accumulator is updated with new operation.

1. Maintain a database of previously processed results
2. Check each new datapoint against existing results
3. Only process new or changed datapoints - this allows us to deduplicate results across batches
4. Combine new results with existing ones for a complete dataset

## Conclusion
With these patterns, you can build robust data processing pipelines that leverage the power of LLMs while managing their limitations.
