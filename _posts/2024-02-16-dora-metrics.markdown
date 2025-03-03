---
layout: post
title: "DORA Metrics: Simple Definitions and What They Measure"
date: 2024-02-16
categories: [infrastructure, metrics, engineering]
tags: [devops]
---

# DORA Metrics: Simple Definitions and What They Measure

Introduced by Google, DORA metrics are four key measures for CI/CD maturity of a system and/or team.

## 1. Deployment Frequency

**Definition**: How often your team deploys code to production.

**Calculation**: Count of deployments per time period (per day, week, or month).

**What It Measures**: The pace at which your team can deliver changes to users.

## 2. Lead Time for Changes

**Definition**: How long it takes for code to go from being written to running in production.

**Calculation**: Time elapsed from code commit to successful deployment in production.

**What It Measures**: Your team's ability to respond quickly to needs or issues.

## 3. Change Failure Rate

**Definition**: The percentage of deployments that cause failures or require fixes.

**Calculation**: 100 * deployments with incidents / Total deployments

**What It Measures**: The quality and stability of your deployments.

## 4. Time to Restore Service

**Definition**: How long it takes to fix a problem in production.

**Calculation**: Time elapsed from incident detection to service restoration.

**What It Measures**: Team's ability to recover from incidents and minimize downtime.

---

These four simple generic metrics gives a quantitative measure of maturity of CI/CD setup for a system/team irrespective of any project/team specific tools, practices or processes.

## References
[Using the Four Keys to measure your DevOps performance](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance)
