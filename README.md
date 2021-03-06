﻿Current status: **Beta**

Create a new assembly. Install the NuGet package:

```powershell
Install-Package Schemavolution.EntityFramework
```

Create a `Genome` class and implement `IGenome` as described below. In the Package Manager Console, run:

```powershell
Evolve-Database
```

And watch your database evolve.

## Your database's genome

Create a class that implements `IGenome`. This class has one method, `AddGenes`. Create a schema (or use `dbo`):

```csharp
public void AddGenes(DatabaseSpecification db)
{
    var dbo = db.UseSchema("dbo");

    var mathematician = DefineMathematician(dbo);

    DefineContribution(dbo, mathematician);
}
```

The database schema is composed of tiny steps -- genes -- that each make small changes to its structure. One kind of gene creates a table:

```csharp
private static PrimaryKeySpecification DefineMathematician(SchemaSpecification schema)
{
    var table = schema.CreateTable("Mathematician");

    // ...
}
```

Another one creates a column:

```csharp
private static PrimaryKeySpecification DefineMathematician(SchemaSpecification schema)
{
    // ...

    var mathematicianId = table.CreateIdentityColumn("MathematicianId");
    var name = table.CreateStringColumn("Name", 100);
    var birthYear = table.CreateIntColumn("BirthYear");
    var deathYear = table.CreateIntColumn("DeathYear", nullable: true);

    // ...
}
```

Another one creates a primary key:

```csharp
private static PrimaryKeySpecification DefineMathematician(SchemaSpecification schema)
{
    // ...

    var pk = table.CreatePrimaryKey(mathematicianId);

    return pk;
}
```

You can group genes together into chromosomes -- methods that define entire tables:

```csharp
private static void DefineContribution(SchemaSpecification schema, PrimaryKeySpecification mathematicianKey)
{
    var table = schema.CreateTable("Contribution");

    var contributionId = table.CreateIdentityColumn("ContributionId");
    var mathematicianId = table.CreateIntColumn("MathematicianId");

    // ...

    var pk = table.CreatePrimaryKey(contributionId);
}
```

Create an index:

```csharp
private static void DefineContribution(SchemaSpecification schema, PrimaryKeySpecification mathematicianKey)
{
    // ...

    var indexMathematicianId = table.CreateIndex(mathematicianId);

    // ...
}
```

Create a foreign key:

```csharp
private static void DefineContribution(SchemaSpecification schema, PrimaryKeySpecification mathematicianKey)
{
    // ...

    var fkMathematician = indexMathematicianId.CreateForeignKey(mathematicianKey);

    // ...
}
```

Organize these genes into chromosomes to keep them nice and neat. Put them in functions. Group them in classes. The organization is up to you. They don't need to be put in sequential order.

## Evolving your database

After adding some genes, you are ready to evolve your database. This process will apply any new genes that have been defined since the last evolution.

You can evolve the database from the Package Manager Console. Select the assembly containing the genome as the default project, and set the Web application as the startup project. Then run:

```powershell
Evolve-Database
```

Your genome will be applied to the database described in `web.config` of the startup project.

## Gene splicing (a.k.a. merging)

You're probably working with a team on your database application. Different members of your team will be editing different parts of the genome at the same time. What happens when they merge their code?

You will have applied one set of genes to your development database before receiving their edits. They will have applied a different set of genes before receiving yours. As long as the two don't conflict, then Schemavolution will splice the two sets together. Their edits will apply cleanly to your database, and yours will apply to theirs.

When will gene splicing not work?

- If you choose the same table names
- If you are working on the same table and choose the same column names
- If you both define a different primary key for the same table

So in general, it will usually work fine.

## Mutating your schema

If you make a mistake, do not delete a gene! Instead, create a new gene that mutates the schema. Rename a table:

```csharp
private static void DefineContribution(SchemaSpecification schema, PrimaryKeySpecification mathematicianKey)
{
    // ...

    var paper = table.RenameTable("Paper");
}
```

Rename a column:

```csharp
private static PrimaryKeySpecification DefineMathematician(SchemaSpecification schema)
{
    // ...

    var yearOfBirth = birthYear.RenameColumn("YearOfBirth");

    // ...
}
```

Or drop a table or column:

```csharp
private static PrimaryKeySpecification DefineMathematician(SchemaSpecification schema)
{
    // ...

    deathDate.DropColumn();

    // ...
}
```

As long as you always add genes, database deployment will be successful. To any environment. In any order. But if you delete one, then you will have to force the tool to roll back the gene, which could result in data loss.

## Phenotype (a.k.a generated SQL scripts)

Genes are expressed as a phenotype (SQL script) when you evolve the datbase schema. These SQL scripts are generated at the time of evolution. A set of related genes combined with the current structure of the database produces the desired script.

If you evolve a database that does not yet have a Mathematicians table, for example, the generated SQL script will create the table in full:

```sql
CREATE TABLE [Mathematicians].[dbo].[Mathematician](
    [MathematicianId] INT IDENTITY (1,1) NOT NULL,
    CONSTRAINT [PK_Mathematician] PRIMARY KEY CLUSTERED ([MathematicianId]),
    [Name] NVARCHAR(100) NOT NULL,
    [BirthYear] INT NOT NULL,
    [DeathYear] INT NULL)
```

But what if you previously evolved the database with just these genes:

```csharp
private static PrimaryKeySpecification DefineMathematician(SchemaSpecification schema)
{
    var table = schema.CreateTable("Mathematician");

    var mathematicianId = table.CreateIdentityColumn("MathematicianId");
    var name = table.CreateStringColumn("Name", 100);

    var pk = table.CreatePrimaryKey(mathematicianId);

    return pk;
}
```

And then later added these:

```csharp
private static PrimaryKeySpecification DefineMathematician(SchemaSpecification schema)
{
    // ...

    var birthYear = table.CreateIntColumn("BirthYear");
    var deathYear = table.CreateIntColumn("DeathYear", nullable: true);

    // ...
}
```

Then you would get this script on the first evolution:

```sql
CREATE TABLE [Mathematicians].[dbo].[Mathematician](
    [MathematicianId] INT IDENTITY (1,1) NOT NULL,
    CONSTRAINT [PK_Mathematician] PRIMARY KEY CLUSTERED ([MathematicianId]),
    [Name] NVARCHAR(100) NOT NULL)
```

And this one on the second:

```sql
ALTER TABLE [Mathematicians].[dbo].[Mathematician]
    ADD [BirthYear] INT NOT NULL
    CONSTRAINT [DF_Mathematician_BirthYear] DEFAULT (0)
ALTER TABLE [Mathematicians].[dbo].[Mathematician]
    DROP CONSTRAINT [DF_Mathematician_BirthYear]
ALTER TABLE [Mathematicians].[dbo].[Mathematician]
    ADD [DeathYear] INT NULL
```

Schemavolution will generate the optimal SQL script to apply the genes that have been added since the last evolution.

## CRISPR/Cas9 gene editing (a.k.a rolling back)

While it's best to only add genes, sometimes you will decide to edit or delete them. Maybe you've only evolved your development database, and you don't want those genes getting into test and production.

If you try to evolve a database that already has genes that are different or no longer in your genome, the process will fail. You can force `Evolve-Database` to roll back those genes:

```powershell
Evolve-Database -Force
```

**Warning:** Devolving a database may cause data loss. For example, if you started with the full `Mathematician` table and deleted the `BirthYear` and `DeathYear` genes, `Evolve-Database -Force` would run the following script:

```sql
ALTER TABLE [Mathematicians].[dbo].[Mathematician]
    DROP COLUMN [DeathYear]
ALTER TABLE [Mathematicians].[dbo].[Mathematician]
    DROP COLUMN [BirthYear]
```

## Partial order

The reason that database migrations are so finicky is that they have to be applied in a specific order. This makes it difficult for members of a development team to work on different migrations at the same time. Usually, they have to resolve collisions by backing out and reapplying their changes. This ensures that changes can be serialized: applied in a specific linear order.

But if you examine the dependencies between migrations, you will find that most of the time parallel migrations can be applied in either order. That's because at their core, these changes are actually partially ordered. It's just that our tools don't know about that partial order.

If a set of migrations are totally ordered, then for any pair of migrations, I can tell which one has to happen before the other. This is usually done by comparing their sequence numbers. If you can make this comparison, then your set is totally ordered: it is a sequence.

If a set is partially ordered, however, then for any pair of migrations, I might be able to tell which comes first, or I might not. This seems less useful, but in fact it gives you more options. If a set of migrations is partially ordered, then that means that different developers can be working on different migrations in parallel. They each think that their migrations came first. By the time they integrate their migrations, they can be applied in either order.

## Yet another migration tool?

What about Entity Framework Migrations? FluentMigrator? DbUp? RoundhousE? Django?

All of these tools assume a totally ordered sequence of migrations. Total ordering makes merging hard. Just Google merging in any of these projects and you will see how difficult it is.

Schemavolution is the first tool that defines a partial order of migrations. I don't expect it to be the last, but until then, enjoy the perks that only partial order can give you.

* Simple merging on multi-developer teams
* Elimination of unnecessary intermediate steps
* Aggregation of migrations for optimal change scripts
* Consolidation of migrations by table
* Organization of code the way you want

You are in complete control of your database migrations, but you don't have to manage their dependencies and order anymore.