---
layout: post
title: "Clean Database Migrations"
date: 2022-10-02
categories: [database, best-practices, software-engineering]
tags: [best-practices, database]
---

# Clean Database Migrations

There are a lot of guidelines about clean code but the database migration is an equally important part of the system usually less talked about.

A well-executed migration strategy can enable smooth deployments, minimize downtime, and prevent data disasters. On the other hand a hasty or poorly planned migration can lead to service outages, data loss, and late-night emergency debugging sessions that no developer wants to experience.

Here are some of my learnings in the context of migrations.

## Deploy Migrations Before Features

Perhaps the most crucial principle of database migrations is: Always deploy your migrations before the code that depends on them. This derisks the deployments

Major issue with DB migrations is that all test environments have very small set of data, so the migration overhead does not become clear on non prod environments.

However on production, because its a large data in the tables, it can cause long running migrations or even issues around locking of tables depending on the type of migration.

Deploying migration before feature derisks this so that the actual feature deployment has the schema already available and hence one less thing to worry about.

## Avoid drastic changes to the schema

There are some changes which are straightforward eg adding a new table, adding a new column as these are backward compatible changes from schema perspective.

However changes like renaming a table, renaming a column or even worst - changing column type - these are very tricky changes and should be avoided as much as possible.

This is because it impacts all of production data hence it increases risk of the deployment.

Also this means if it needs a rollback for any reason, this would again run the down block for the migration again impacting all data, meaning additional risk of the rollback.

Remember, a stable system should have least impact on users - much less due to a deployment because we may be doing multiple deployments a day.

## Method to the madness

### Changing column names
First evaluate the need of the operation. If the column has huge data and this really needs to be done, always add a new column and copy the data from old to new column. Newer records will need data populated in both old and new columns.

### Changing column types
This is very risky as reverting this is a complex operation. If this is really needed, then similar approach of adding new column and populating data for existing records works nicely.

### Deleting column
Once the column is not refereed in code anymore, the column could be deleted safely. Instead of hard deleting the column, evaluate if it will make sense to soft delete it eg rename it with a prefix `_deleted`

This is important in case you are using an ORM like SQLAlchemy which has schemas as classes.

If the column is deleted, then it can cause an issue with older migrations - when someone new joins a team, then they may face issues because the column does not exist.

### Changing default values
This does not impact existing values but needs to be tested clearly because if the column is optional, then rows which did not have value could suddenly get new value breaking the application

This approach offers several advantages:

* Backward compatibility: Old code can still run against the new schema.
* Gradual transition: You can migrate functionality piece by piece.
* Easy rollback: If something goes wrong and code needs to be rolled back, the original data structure is still intact.
* Lower risk of deployment

## A comprehensive Migration path

Let's put these principles together into a comprehensive strategy:

### Phase 1: Pre-Deployment
* Add new columns/tables with reasonable defaults
* Create necessary indexes on new structures
* Add any required constraints that won't impact existing data

### Phase 2: Feature Deployment
* Deploy code that writes to both old and new structures
* Begin background process to populate new structures with data from old ones

### Phase 3: Feature Transition
* Deploy code that reads from new structures but continues writing to both
* Verify data consistency between old and new structures

### Phase 4: Cleanup (Later)
* Deploy code that no longer references old structures
* Eventually remove old, unused structures (after confirming no dependencies)

This phased approach minimizes risk and ensures smooth transitions, even for complex schema changes.

## Additional best practises

1. **Make Migrations Idempotent**
   Migrations should be safe to run multiple times without causing errors or data corruption. Use conditionals like IF NOT EXISTS or check for the presence of objects before creating them.

2. **Test Migrations Thoroughly**
   Always test migrations on a copy of production data to verify:
   * Execution time (will it cause significant downtime?)
   * Space requirements (will you run out of disk space?)
   * Correct data transformation (is the new structure populated properly, what if values are null?)

3. **Have a Rollback Plan**
