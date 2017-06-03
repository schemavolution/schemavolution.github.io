---
layout: childpage
title: DbVolver Roadmap
---

The goal of DbVolver is to bring the benefits of partial ordering to managing database migrations. These include:

- Supporting parallel development on teams
- Optimizing database deployment scripts
- Organizing data model specifications

This project does not assume Entity Framework as the data access platform, but it works best with either EF6 or EF Core.

In the following feature lists ~~strikethrough~~ indicates that the feature is complete.

## Specifications

The minimum viable product includes specifications for all valid column types and common operations. Support will be added for:

1. Drop column/table/index/foreign key
2. Rename column/table
3. Automatic rename of index/foreign key to preserve convention

## Customizations

The fluent interface would necessarily be limited to the most common migrations. Support for custom SQL would be required for all scenarios. Specifically, the following will be considered:

1. Views - list source tables/columns as prerequisites
2. Stored procedures - list source tables/columns as prerequisites
3. ~~Less common column data types: e.g. VARCHAR instead of NVARCHAR - list table as prerequisite~~
4. ~~Generic custom SQL - list prerequisites based on intention~~

## Verification

Ensure that deployment will be successful before making the attempt. Situations to look for:

1. Name collisions - force making a rename a prerequisite of one of the migrations
2. Foreign key data type alignment

## Tooling

Even though the tool does not require Entity Framework, it works best with EF. The tooling is especially improved when combined with EF. Tooling support will include:

1. Update-Database (or similar) to apply migrations: -Force to migrate down to a previous version
2. Verify-Migrations to run verification on the model, compare it with the EF model, and compare it with the database
3. Add-Migration (or similar) to generate code based on EF model
4. Unit test to compare the migration model with the EF model, and fail on mismatch
5. Conversion of an EF Migrations project to DbVolver, including mapping the __MigrationHistory table at deploy time

## Data Migration

Initially DbVolver is all about the schema. But there is a real opportunity to address data migration as well. The tool can generate a SQL MERGE statement based on specified rows that migrates data from one structure to another. Layer a data migration into the specification between a Create (new structure) and Drop (old structure), and the tool will decide if the data migration needs to take place. If the target database does not yet have the old structure, then the data migration can be skipped altogether.

## Domain Driven Design

An initial concept of DDD is in the early versions of the project. This could be fleshed out to provide an optional set of tools for applying DDD conventions to the data model. For example, an AggregateRoot is a single entity with a primary key and child entities; references to the primary keys of the child entities are disallowed. Entity and ValueType could be represented by different C# types, and enforce different sets of conventions. This area will become more clear as DbVolver is used in more production scenarios.