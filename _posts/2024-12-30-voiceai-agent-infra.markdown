---
layout: post
title: "Modeling Voice AI Agents For Scale: A Kubernetes Approach"
date: 2024-12-30
categories: [ai, voice-ai, kubernetes]
tags: [infrastructure, kubernetes, scalability, microservices, voice-agents]
---

# Modeling Voice Pipelines as Scalable Agents: A Kubernetes Approach

In the world of Voice AI agents, one of the key challenges is creating agent architectures such that it can scale to support concurrent calls while maintaining low latency. Voice interactions need very low latencies because it becomes a bad experience for the users if there are lags.
In this post, I will share how I approached this challenge.

Key goals for a voice agent are -
1. Low startup times - when call is starting, the user should not be kept waiting for long as it can cause drop off
2. Low latencies - ideally the agent responses should be quick without any hiccups in between responses
3. Clean closing of the call - Once the call ends, the call details should be available reliably for post processing.

## Agent Startup Performance

When building voice-ai agent systems, startup time is critical.
Any initialization of STT, TTS services, the voice pipeline, connection establishment itself takes time.
Ideally the agent startup should be less than 200ms otherwise the user can get distracted and drop off.

## Low latency Agent Responses
When a call is in progress, there is significant processing going on at the agent side.
Usually a voice ai agent pipeline looks like - STT -> LLM ->application logic -> TTS
This means there are already multiple network calls involved.
So any delay in either of the above steps due to resource constraints can hamper user experience.
A nice thing about these agents is that each call is different from other, so we can leverage horizontal scaling to address this problem

## Clean closing of the call
Once the call ends, the agent needs to do a graceful hand over to the main application to do the post processing.
The post processing itself can be a heavy task needing multiple LLM, external service calls
This means for a call to be successful end to end, graceful shutdown of the agent is critical, otherwise it can endup loosing the call.

## Evaluating Abstraction Options

When designing our architecture, I evaluated several options for agent abstraction:

| Abstraction | Pros | Cons                                             |
|-------------|------|--------------------------------------------------|
| **Threads** | Low overhead, shared memory | Limited isolation, potential resource contention |
| **Processes** | Better isolation | Needs heavyweight machines, difficult recovery   |
| **Containers** | Isolation with controlled resource usage | Slightly higher startup overhead                 |
| **Jobs** | Complete isolation, scalable | Potentially higher startup latency               |

Traditional process-based approaches other problems though, for eg -
1. They required heavy machines to handle multiple concurrent processes
2. Recovery is impossible when worker/process crashes
3. Another major downside is deployments. We do not want an ongoing agent to crash/stop if a redeployment happens.

## The Shared-Nothing Architecture

A key insight that guided the design was that voice calls do not need to share state among them. Each call exists in isolation from others, which means we can leverage a **shared-nothing architecture** for the agents.

This nature of the problem also allows to scale the agents horizontally. That way we can:
- Distribute calls across multiple workers
- Scale out to handle concurrent call and call bursts
- Isolate failures to individual calls rather than affecting the entire system

## Final Solution: Kubernetes Jobs

After evaluating the options, I ended up implementing the voice ai agents as K8s jobs. This approach offers several key advantages:
1. Fast Startup Times -
Container images are cached on the nodes, which helps us manage the startup times for agents.
When a new call comes in, Kubernetes can schedule a job almost instantly using pre-pulled images.
2. Deployment independent - Deployments can continue to happen in the main service while the ai agents can keep on serving the calls as the corresponding container images dont need to be refreshed.
3. Horizontal Scalability - when burst of calls come in, K8s automatically handles scaling the pods supporting huge number of concurrent calls.
The limitation is the underlying resource limits for the k8s.
4. Job auditing - K8s provides monitoring for the individual jobs. Also, the job logs are exported to our logging system at the end of the call, making the entire
voice pipeline debuggable.
5. Out of the box monitoring - With this approach, monitoring for the jobs is also available with the out of the box monitoring from the k8s cluster.
6. Isolation - Each job can be isolated and controlled with resource limits.
This does not allow any single agent to consume excessive resources and affect other calls.


## Conclusion

By using Kubernetes jobs as a way to implement voice ai agent jobs, I could achieve the non functional requirements from the agents.
If you're building voice ai agents systems that need to scale, this could be good approach.
The combination of K8s, job-based processing, and horizontal scaling provides an excellent architecture choice for voice agents.
What approaches have you used for scaling voice processing systems? Share your experiences in the comments below!
