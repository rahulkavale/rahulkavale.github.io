---
layout: post
title: "Good coding practices enabling SOC2 Compliance"
date: 2024-10-01
categories: [security, startups, compliance, best-practices]
tags: [soc2]
---

# Good Coding Practices enabling SOC2 Compliance

When working on a 0 to 1 product, lot of code specific changes are done to allow moving fast.
However, once customer starts using the service, additional business decisions can dictate the development workflow.
One such example is compliance - imagine a big new customer is ready to onboard, only issue is they want some complaince certificate eg SoC2 compliance.

Now, SoC2 includes lot of things - right from documentation of policies, infrastructure guidelines down to code guidelines.
While most of the things from such certifications are good for long term, some of them are general good practices and can help the team irrespective even from initial stage.

I am listing some of those changes below -

## 1. Separate User and Production User Accounts with Correct Access

SOC2 requires the principle of least privilege.
When it comes to DB access, a clear prod and non prod user separation is expected.
This is a straight forward change and immensely useful to protect against unexpected changes on prod.

## 2. Secret Keys in Secret Manager Instead of ENV Variables
Lot of times secrets like api keys etc are kept in Env variables without giving much thought.
However, this is risky as it can end up exposing critical details on production system.
Cleaner way is to keep secrets in secret manager.
Most of the cloud providers have secret manager and using it from start does not come as a blocker or very large dev effort.
Benefit is the values are kept safe with a trail for changes for those.

## 3. Database and Cache Not Exposed to Public
Usually when starting a new project, the storage systems like DB, Cache are kept private.
As team marches forward with features, this kind of gets sidelined - in fact, having a public DB becomes convenient as it allows debugging production issues easily.

However, this is very risky practice as it exposes the storage system to public access.

Apart from these easy changes in code, here are some similar infra level changes too -
## 4. Worker Nodes Not Exposed to Public
Worker processes should only accept tasks from authorized internal services, not directly from public endpoints.
## 5. Separate Network Groups for Production and Non-Production Services
## 6. Encrypted Storage -
Data in transit and Data at rest - both should be encrypted. All cloud providers have this feature and it is immensely helpful from security and compliance perspective.
## 7. Backups Enabled for Database
This is in general a good practice for a DB and another low hanging fruit from compliance perspective.
All cloud providers usually give option for this without much complexity to enable it.

## Conclusion
Compliance can come suddenly as a blocker to onboard large/enterprise clients and following such practises are low hanging fruits which improve stability of the system with some items from compliance already ticked off.
Want to share your SOC2 compliance journey? Drop a comment below!
