Current status: **Proof of Concept**

You can't yet use this in production.

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

Create a `DatabaseEvolver` and provide the **master** connection string and the genome.

```csharp
class DatabaseConfig
{
    public static void Configure(HttpServerUtility server)
    {
        var master = new SqlConnectionStringBuilder
        {
            DataSource = @"(LocalDB)\MSSQLLocalDB",
            InitialCatalog = "master",
            IntegratedSecurity = true
        };

        string fileName = server.MapPath("~/App_Data/Mathematicians.mdf");
        string databaseName = "Mathematicians";
        var evolver = new DatabaseEvolver(
            databaseName,
            fileName,
            master.ConnectionString,
            new Genome());
        evolver.EvolveDatabase();
    }
}
```

Put the above class in `App_Start`, and call `DatabaseConfig.Configure(Server)` from `Application_Start` in `Global.asax.cs`.

In the near future, you will also be able to do this from PowerShell:

```powershell
Evolve-Database -DatabaseName Mathematicians -MasterConnectionString "Data Source=(local);Initial Catalog=master;Integrated Security=True;"
```

Or you can take a look at the SQL script before you apply it. Just apply the `-Script` flag to the command.

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

## How are genes expressed as SQL scripts? (a.k.a. phenotype)

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

## Rolling back

While it's best to only add genes, sometimes you will decide to take them away. Maybe you've only evolved your development database, and you don't want those genes getting into test and production.

If you try to evolve a database that already has genes that are no longer in your genome, the process will fail. You can force the `DatabaseEvolver` to roll back those genes:

```csharp
var evolver = new DatabaseEvolver(
    databaseName,
    fileName,
    master.ConnectionString,
    new Genome());
evolver.DevolveDatabase();
evolver.EvolveDatabase();
```

**Warning:** Devolving a database may cause data loss. For example, if you started with the full `Mathematician` table and deleted the `BirthYear` and `DeathYear` genes, `DevolveDatabase` would run the following script:

```sql
ALTER TABLE [Mathematicians].[dbo].[Mathematician]
    DROP COLUMN [DeathYear]
ALTER TABLE [Mathematicians].[dbo].[Mathematician]
    DROP COLUMN [BirthYear]
```

In the near future, you will be able to do this in PowerShell using the `-Force` switch:

```powershell
Evolve-Database -DatabaseName Mathematicians -MasterConnectionString "Data Source=(local);Initial Catalog=master;Integrated Security=True;" -Force
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