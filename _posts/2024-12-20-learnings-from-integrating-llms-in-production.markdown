---
layout: post
title: "Lessons from the Trenches: What I Learned Integrating LLMs in Production"
date: 2024-12-20
categories: [ai, llm, machine-learning, production]
tags: [llm, production, engineering, lessons-learned]
description: "Practical insights from integrating Large Language Models into production systems, including handling non-JSON outputs, evaluation loops, model switching, and more."
---

Integrating LLMs in production systems has been a very powerful experience for me. Initially I was of the opinion that its just an API call and I need to parse the response.

I'm writing this post to share what I wish I had known from the start. If you're considering integrating LLMs into your production systems, these insights might save you some headaches.

## The Unpredictable Nature of Non-JSON Outputs

One of the first challenges I encountered was dealing with non-JSON outputs. While lot of LLM providers offer JSON mode options, some of them dont.
That means if we want the models to be interchangable, we cant really be using the JSON mode.

One workaround for this is to include JSON response instructions in the prompt.
However, even with explicit instructions and JSON mode enabled, sometimes it may not be correct, for eg -
- Return malformed JSON with trailing commas
- Include explanatory text before or after the JSON
- Hallucinate extra fields not specified in our schema
- Omit required fields entirely

This meant parsing the JSON from the response was needed to handle those scenarios with fallbacks-

{% highlight python %}
def parse_llm_response(response):
    try:
        # First attempt: direct JSON parsing
        return json.loads(response)
    except json.JSONDecodeError:
        try:
            # Second attempt: Extract JSON from text using regex
            match = re.search(r'```json\n(.*?)\n```', response, re.DOTALL)
            if match:
                return json.loads(match.group(1))

            # Third attempt: Fix common JSON errors
            fixed_json = fix_common_json_errors(response)
            return json.loads(fixed_json)
        except Exception:
            # Final fallback: Structured extraction
            return extract_fields_with_pattern_matching(response)
{% endhighlight %}

This defensive approach saved countless runtime errors, but added complexity that was not initially expected.
There are libraries like instructor which also help in these cases for simpler scenarios.
One reason to not consider using instructor was that - when output was not correct in JSON format somehow, it will retry for getting correct output.
However this means we are loosing opportunity to tune the prompt itself to avoid such errors.

## Including Scores in Responses for Better Decision Making

Raw outputs are not always enough, its a good idea to also include additional metadata in the response like confidence scores to make intelligent decisions about when to trust the model versus when to fall back to alternative processes.

I started including confidence scores and reasoning/justifications directly in our prompts:

{% highlight json %}
For each field in your response, include a confidence score from 0-1 that represents your certainty in the answer. For example:
{
  "recommendation": "Approve application",
  "recommendation_confidence": 0.87,
  "reasoning": "Applicant meets all criteria and has strong history",
  "reasoning_confidence": 0.92
}
{% endhighlight %}

This pattern allows us to:
- Route low-confidence outputs to human review
- A/B test different prompting strategies based on confidence metrics
- Detect drift in model performance over time
- Implement confidence thresholds for automated actions

The scores became such a crucial part of our system that I now consider those as a standard component of any LLM integration.

## Response <> Evaluation Loop for Quality Assurance

Perhaps the most impactful improvement to our system was implementing a continuous response evaluation loop. Initially, it was simple request -> responses flow. But I soon realized that adding an evaluation step dramatically improved quality.

Here's the general pattern:

1. Generate initial response
2. Run a separate evaluation prompt that assesses quality, factual accuracy, and completeness
3. If the evaluation falls below thresholds, regenerate with more specific instructions
4. Log all attempts and evaluations

{% highlight python %}
def generate_with_evaluation(prompt, max_attempts=3):
    for attempt in range(max_attempts):
        response = llm_client.generate(prompt)
        evaluation = evaluate_response(response, prompt)

        if evaluation['score'] >= QUALITY_THRESHOLD:
            return response, evaluation

        # Create a more specific prompt based on evaluation feedback
        prompt = refine_prompt(prompt, response, evaluation)

    # If we've exhausted attempts, return best response so far
    return best_response, best_evaluation
{% endhighlight %}

This approach increases our success rate from around 70% to over 95% for complex tasks, at the cost of additional latency and tokens.
This is also what people refer to as agentic. Now there is out of the box support for such workflows in different libraries like langchain etc.
The above model was simplistic and generic enough to not need any heavy external library dependencies

## Changing LLM Models at Runtime

Different tasks benefit from different models. Initially prompts had hard-coded models, but soon it was clear that this needs a more flexible approach.
The scenarios where we may need to change models for a prompt are multiple eg imagine on dev you want a different model as compared to prod because of cost reasons
Or you want to do A/B style evaluation of model before switching it completely.
Other concerns which may need this are -
- Latency requirements
- Cost constraints
- Availability (during outages)

I implemented a simple router that selects the appropriate model:
The dynamic selection saved money while improving performance.

## Navigating Different Context Limits

Context limits vary significantly across models, from 4K tokens to 200K+ tokens. It was important to design the system to gracefully handle these differences:

1. Implement context measurement for all inputs
2. Some models have large prompt tokens but smaller output tokens(eg Anthropic models), so it can cause issues at runtime.
3. Create fallback strategies when context limits are exceeded
4. Keep track of tokens used(prompt tokens, output tokens) for different prompts to be able to monitor any near context limit responses.

For long documents, this can be addressed with a chunking strategy with overlapping sections to preserve context:

## Graceful Degradation When Rate Limited

Rate limits are a fact of life when working with LLM APIs. Its also important to note that LLM apis do not provide any uptime SLAs. So its not so common to get errors from the LLM response.
It becomes very important to be able to handle such errors.
While transient errors like timeouts are usually handled by the SDK itself, rate limit is special because it needs more thought rather than simple retry.

1. One simple way to address it is using exponential backoff retry mechanism
2. Another way is to fallback to alternative models when primary model is rate limited after configured retries exceed
3. Circuit breaker like mechanism to stop any additional calls to the LLM when a rate limit is received

This results in graceful degradation giving us higher availability without much impact on latency especially around peak hours.

## Complete Tracing of LLM Calls

Debugging LLM systems is challenging without visibility into each step of the process.
This becomes more important in case of LLMs because we continuously need to evaluate each response to make sure its within the given constraints.
Hence its very important to keep track of ALL LLM calls - not just successful ones but even errors.
With LLMs, even the error responses are equally important because those are the exact responses that come handy to tune the prompt
Things to track
- Input prompts (sanitized of PII)
- Raw model responses
- Metadata (model, prompt tokens, responses tokens, cached tokens)
- Latency metrics like time for first token, total time for response

## Versioned Models for Predictable Updates

Another small thing to know is that models keep on updating. For some reason, the labs do not always have predictable versioning for models.
eg model `gpt-4o` could point to different versions of the model as updates are published.
Usually model updates do not break prompts but there is still a good change it can break.
So it becomes tricky to suddenly realize a prompt is breaking because it was using `gpt-4o` and now the new model is doing something funky for good or for worse.

That means its a good idea to:
1. Always use pointed versions of models (e.g., `gpt-4-0613` instead of `gpt-4`)
2. Maintain a testing suite of representative inputs
3. Test new model versions in staging before upgrading
4. Keep the ability to rollback to previous model versions
5. Monitor key metrics during and after upgrades


## Conclusion

Integrating LLMs into production systems requires much more than just API calls. It's about building robust systems that can handle the unique challenges these models present.

The field is evolving rapidly, and what works today may not work tomorrow. The key is building flexible, resilient systems that can adapt to changes while maintaining reliability.

I hope these lessons help you navigate your own LLM integration journey.
