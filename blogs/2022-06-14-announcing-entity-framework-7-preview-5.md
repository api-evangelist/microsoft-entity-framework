---
title: "Announcing Entity Framework 7 Preview 5"
url: "https://devblogs.microsoft.com/dotnet/announcing-ef7-preview5/"
date: "Tue, 14 Jun 2022 17:17:10 +0000"
author: "Jeremy Likness"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>Entity Framework 7 (EF7) Preview 5 has shipped with support for Table-per-Concrete type (TPC) mapping. This blog post will focus on TPC. There are several other enhancements included in Preview 5, such as:</p>
<ul>
<li><a href="https://github.com/dotnet/efcore/issues/26199">Support for AT TIME ZONE in SQL Server</a></li>
<li>Updates to command and connection interception (<a href="https://github.com/dotnet/efcore/issues/23087">#23087</a>, <a href="https://github.com/dotnet/efcore/issues/23085">#23085</a>, <a href="https://github.com/dotnet/efcore/issues/17261">#17261</a>)</li>
<li>Addition of the <a href="https://github.com/dotnet/efcore/issues/9621">delete behavior attribute</a></li>
</ul>
<p>Read the <a href="https://github.com/dotnet/efcore/issues?q=is%3Aissue+milestone%3A7.0.0-preview5+is%3Aclosed+label%3Atype-enhancement">full list of EF7 Preview 5 enhancements</a>.</p>
<h2>Table-per-concrete-type (TPC) mapping</h2>
<p>By default, EF Core maps an inheritance hierarchy of .NET types to a single database table. This is known as the table-per-hierarchy (TPH) mapping strategy. EF Core 5.0 introduced the table-per-type (TPT) strategy, which supports mapping each .NET type to a different database table. In EF Core 7.0 preview 5.0, we are excited to introduce the table-per-concrete-type (TPC) strategy. TPC also maps .NET types to different tables, but in a way that addresses some common performance issues with the TPT strategy.</p>
<p>In this post, we&#8217;ll start by describing the structure of TPH, TPT, and TPC mappings, then look at how these strategies can be configured in EF Core, and finally discuss the pros and cons of each approach.</p>
<h3>Mapping inheritance hierarchies</h3>
<p>Consider the following object-oriented domain model:</p>
<pre><code class="language-csharp">public abstract class Animal
{
    public int Id { get; set; }
    public string Species { get; set; }
}

public class FarmAnimal : Animal
{
    public decimal Value { get; set; }
}

public class Pet : Animal
{
    public string Name { get; set; }
}

public class Cat : Pet
{
    public string EducationLevel { get; set; }
}

public class Dog : Pet
{
    public string FavoriteToy { get; set; }
}</code></pre>
<p>If we are to retrieve some <code>Animal</code> object from the database, then we must know which type of animal it is. We don&#8217;t want to save a cat and then read it back as a dog, or vice versa. (I can tell you from experience that dogs generally don&#8217;t like to be treated as cats, and cats <em>certainly</em> don&#8217;t like to be treated as dogs!) So this means the type of animal&#8211;that is the actual class used when the animal was created in C#&#8211;must be saved to the database in some form.</p>
<p>Further, different information is associated with each <code>Animal</code> object depending on its type. For example, in our model, a farm animal has some monetary value but no name, while pets are priceless and named.</p>
<p>Inheritance mapping strategies (TPH, TPT, or TPC) define how this object-oriented type information and type-specific information are saved into a relational database, where inheritance is not a natural concept.</p>
<h4>The TPH strategy</h4>
<p>With the TPH strategy, a single table is created for all types in the hierarchy&#8211;hence the name &#8220;table-per-hierarchy&#8221;. This table contains a special column containing a &#8220;discriminator value&#8221;, which indicates the type of the object saved in each row. In addition, a column is created for every property of every type in the hierarchy. For example:</p>
<pre><code class="language-sql">CREATE TABLE [Animals] (
    [Id] int NOT NULL IDENTITY,
    [Species] nvarchar(max) NOT NULL,
    [Discriminator] nvarchar(max) NOT NULL,
    [Value] decimal(18,2) NULL,
    [Name] nvarchar(max) NULL,
    [EducationLevel] nvarchar(max) NULL,
    [FavoriteToy] nvarchar(max) NULL,
    CONSTRAINT [PK_Animals] PRIMARY KEY ([Id])
);</code></pre>
<p>Saving two cats, a dog, and a sheep to this table results in the following:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">Species</th>
<th style="text-align: left;">Discriminator</th>
<th style="text-align: left;">Value</th>
<th style="text-align: left;">Name</th>
<th style="text-align: left;">EducationLevel</th>
<th style="text-align: left;">FavoriteToy</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">1</td>
<td style="text-align: left;">Felis catus</td>
<td style="text-align: left;">Cat</td>
<td style="text-align: left;">NULL</td>
<td style="text-align: left;">Alice</td>
<td style="text-align: left;">MBA</td>
<td style="text-align: left;">NULL</td>
</tr>
<tr>
<td style="text-align: left;">2</td>
<td style="text-align: left;">Felis catus</td>
<td style="text-align: left;">Cat</td>
<td style="text-align: left;">NULL</td>
<td style="text-align: left;">Mac</td>
<td style="text-align: left;">BA</td>
<td style="text-align: left;">NULL</td>
</tr>
<tr>
<td style="text-align: left;">3</td>
<td style="text-align: left;">Canis familiaris</td>
<td style="text-align: left;">Dog</td>
<td style="text-align: left;">NULL</td>
<td style="text-align: left;">Toast</td>
<td style="text-align: left;">NULL</td>
<td style="text-align: left;">Mr. Squirrel</td>
</tr>
<tr>
<td style="text-align: left;">4</td>
<td style="text-align: left;">Ovis aries</td>
<td style="text-align: left;">FarmAnimal</td>
<td style="text-align: left;">100.00</td>
<td style="text-align: left;">NULL</td>
<td style="text-align: left;">NULL</td>
<td style="text-align: left;">NULL</td>
</tr>
</tbody>
</table>
<p>Notice that:</p>
<ul>
<li>The value in the <code>Discriminator</code> column indicates the type of C# object saved</li>
<li>There is a column for every property in the hierarchy</li>
<li>If the property does not exist for the type of the object saved, then the value in the database for that column is null</li>
</ul>
<blockquote><p>The TPH strategy requires that database columns be nullable for any property not defined in the root type of the hierarchy, even if that property is required. It is possible to create a database constraint for these columns to ensure the value is non-null whenever an instance with that property is saved, but this is not done automatically by EF Core. See <a href="https://github.com/dotnet/efcore/issues/20931">Issue #20931 on the EF Core GitHub repo</a> for more information.</p></blockquote>
<h4>The TPT strategy</h4>
<p>With the TPT strategy, a different table is created for every type in the hierarchy&#8211;hence the name &#8220;table-per-type&#8221;. The table itself is used to determine the type of the object saved, and each table contains only columns for the properties of that type. For example:</p>
<pre><code class="language-sql">CREATE TABLE [Animals] (
    [Id] int NOT NULL IDENTITY,
    [Species] nvarchar(max) NOT NULL,
    CONSTRAINT [PK_Animals] PRIMARY KEY ([Id])
);

CREATE TABLE [FarmAnimals] (
    [Id] int NOT NULL,
    [Value] decimal(18,2) NOT NULL,
    CONSTRAINT [PK_FarmAnimals] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_FarmAnimals_Animals_Id] FOREIGN KEY ([Id]) REFERENCES [Animals] ([Id])
);

CREATE TABLE [Pets] (
    [Id] int NOT NULL,
    [Name] nvarchar(max) NOT NULL,
    CONSTRAINT [PK_Pets] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Pets_Animals_Id] FOREIGN KEY ([Id]) REFERENCES [Animals] ([Id])
);

CREATE TABLE [Cats] (
    [Id] int NOT NULL,
    [EducationLevel] nvarchar(max) NOT NULL,
    CONSTRAINT [PK_Cats] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Cats_Pets_Id] FOREIGN KEY ([Id]) REFERENCES [Pets] ([Id])
);

CREATE TABLE [Dogs] (
    [Id] int NOT NULL,
    [FavoriteToy] nvarchar(max) NOT NULL,
    CONSTRAINT [PK_Dogs] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Dogs_Pets_Id] FOREIGN KEY ([Id]) REFERENCES [Pets] ([Id])
);</code></pre>
<p>Saving the same data into this database results in the following:</p>
<p>Animals table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">Species</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">1</td>
<td style="text-align: left;">Felis catus</td>
</tr>
<tr>
<td style="text-align: left;">2</td>
<td style="text-align: left;">Felis catus</td>
</tr>
<tr>
<td style="text-align: left;">3</td>
<td style="text-align: left;">Canis familiaris</td>
</tr>
<tr>
<td style="text-align: left;">4</td>
<td style="text-align: left;">Ovis aries</td>
</tr>
</tbody>
</table>
<p>FarmAnimals table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">Value</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">4</td>
<td style="text-align: left;">100.00</td>
</tr>
</tbody>
</table>
<p>Pets table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">Name</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">1</td>
<td style="text-align: left;">Alice</td>
</tr>
<tr>
<td style="text-align: left;">2</td>
<td style="text-align: left;">Mac</td>
</tr>
<tr>
<td style="text-align: left;">3</td>
<td style="text-align: left;">Toast</td>
</tr>
</tbody>
</table>
<p>Cats table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">EducationLevel</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">1</td>
<td style="text-align: left;">MBA</td>
</tr>
<tr>
<td style="text-align: left;">2</td>
<td style="text-align: left;">BA</td>
</tr>
</tbody>
</table>
<p>Dogs table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">FavoriteToy</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">3</td>
<td style="text-align: left;">Mr. Squirrel</td>
</tr>
</tbody>
</table>
<p>Notice that the data is saved in a normalized form, but that this means the information for a single object is spread across multiple tables.</p>
<h4>TPC mapping</h4>
<p>The TPC strategy is similar to the TPT strategy except that a different table is created for every concrete type in the hierarchy, but tables are not created for abstract types&#8211;hence the name &#8220;table-per-concrete-type&#8221;. As with TPT, the table itself indicates the type of the object saved. However, unlike TPT mapping, each table contains columns for every property in the concrete type <em>and its base types</em>. For example:</p>
<pre><code class="language-sql">CREATE TABLE [FarmAnimals] (
    [Id] int NOT NULL DEFAULT (NEXT VALUE FOR [AnimalIds]),
    [Species] nvarchar(max) NOT NULL,
    [Value] decimal(18,2) NOT NULL,
    CONSTRAINT [PK_FarmAnimals] PRIMARY KEY ([Id])
);

CREATE TABLE [Pets] (
    [Id] int NOT NULL DEFAULT (NEXT VALUE FOR [AnimalIds]),
    [Species] nvarchar(max) NOT NULL,
    [Name] nvarchar(max) NOT NULL,
    CONSTRAINT [PK_Pets] PRIMARY KEY ([Id])
);

CREATE TABLE [Cats] (
    [Id] int NOT NULL DEFAULT (NEXT VALUE FOR [AnimalIds]),
    [Species] nvarchar(max) NOT NULL,
    [Name] nvarchar(max) NOT NULL,
    [EducationLevel] nvarchar(max) NOT NULL,
    CONSTRAINT [PK_Cats] PRIMARY KEY ([Id])
);

CREATE TABLE [Dogs] (
    [Id] int NOT NULL DEFAULT (NEXT VALUE FOR [AnimalIds]),
    [Species] nvarchar(max) NOT NULL,
    [Name] nvarchar(max) NOT NULL,
    [FavoriteToy] nvarchar(max) NOT NULL,
    CONSTRAINT [PK_Dogs] PRIMARY KEY ([Id])
);</code></pre>
<p>Notice that:</p>
<ul>
<li>There is no table for <code>Animal</code>, since it is an abstract type in the object model. Remember that C# does not allow instances of abstract types, and there is therefore no situation where one will be saved to the database.</li>
<li>The mapping of properties in base types is repeated for each concrete type&#8211;for example, every table has a <code>Species</code> column, and both <code>Cats</code> and <code>Dogs</code> have a <code>Name</code> column.</li>
</ul>
<p>Saving the same data into this database results in the following:</p>
<p>FarmAnimals table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">Species</th>
<th style="text-align: left;">Value</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">4</td>
<td style="text-align: left;">Ovis aries</td>
<td style="text-align: left;">100.00</td>
</tr>
</tbody>
</table>
<p>Pets table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">Species</th>
<th style="text-align: left;">Name</th>
</tr>
</thead>
</table>
<p>Cats table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">Species</th>
<th style="text-align: left;">Name</th>
<th style="text-align: left;">EducationLevel</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">1</td>
<td style="text-align: left;">Felis catus</td>
<td style="text-align: left;">Alice</td>
<td style="text-align: left;">MBA</td>
</tr>
<tr>
<td style="text-align: left;">2</td>
<td style="text-align: left;">Felis catus</td>
<td style="text-align: left;">Mac</td>
<td style="text-align: left;">BA</td>
</tr>
</tbody>
</table>
<p>Dogs table:</p>
<table>
<thead>
<tr>
<th style="text-align: left;">Id</th>
<th style="text-align: left;">Species</th>
<th style="text-align: left;">Name</th>
<th style="text-align: left;">FavoriteToy</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">3</td>
<td style="text-align: left;">Canis familiaris</td>
<td style="text-align: left;">Toast</td>
<td style="text-align: left;">Mr. Squirrel</td>
</tr>
</tbody>
</table>
<p>Notice that, unlike with TPT mapping, all the information for a single object is contained in a single table.</p>
<h3>Configuring inheritance mappings in EF Core</h3>
<p>When mapping an inheritance hierarchy, all types in the hierarchy must be explicitly included in the model. This can be done by creating a <code>DbSet</code> property for the type on your <code>DbContext</code>:</p>
<pre><code class="language-csharp">public DbSet&lt;Animal&gt; Animals { get; set; }
public DbSet&lt;Pet&gt; Pets { get; set; }
public DbSet&lt;Cat&gt; Cats { get; set; }
public DbSet&lt;Dog&gt; Dogs { get; set; }
public DbSet&lt;FarmAnimal&gt; FarmAnimals { get; set; }</code></pre>
<p>Or by using the <code>Entity</code> method in <code>OnModelCreating</code>:</p>
<pre><code class="language-csharp">protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity&lt;Animal&gt;();
    modelBuilder.Entity&lt;Pet&gt;();
    modelBuilder.Entity&lt;Cat&gt;();
    modelBuilder.Entity&lt;Dog&gt;();
    modelBuilder.Entity&lt;FarmAnimal&gt;();
}</code></pre>
<blockquote><p>This is different from the legacy EF6 behavior, where derived types of mapped base types would sometimes be automatically discovered.</p></blockquote>
<p>Nothing else needs to be done to map the hierarchy as TPH, since it is the default strategy. However, you can make this explicit by calling <code>UseTphMappingStrategy</code> on the base type of the hierarchy. For example:</p>
<pre><code class="language-csharp">modelBuilder.Entity&lt;Animal&gt;().UseTphMappingStrategy();</code></pre>
<p>To use TPT instead, change this to <code>UseTptMappingStrategy</code>. For example:</p>
<pre><code class="language-csharp">modelBuilder.Entity&lt;Animal&gt;().UseTptMappingStrategy();</code></pre>
<p>Likewise, <code>UseTpcMappingStrategy</code> is used to configure TPC:</p>
<pre><code class="language-csharp">modelBuilder.Entity&lt;Animal&gt;().UseTpcMappingStrategy();</code></pre>
<p>In each case, the table name to use for each type can be configured using the <code>ToTable</code> builder method, or the <code>[Table]</code> attribute. However, this is only valid on types that are mapped to a table for the strategy being used. For example, the following code specifies the table names for TPC mapping:</p>
<pre><code class="language-csharp">modelBuilder.Entity&lt;Pet&gt;().ToTable("Pets");
modelBuilder.Entity&lt;Cat&gt;().ToTable("Cats");
modelBuilder.Entity&lt;Dog&gt;().ToTable("Dogs");
modelBuilder.Entity&lt;FarmAnimal&gt;().ToTable("FarmAnimals");</code></pre>
<p>No table name can be specified for <code>Animal</code> because it is not mapped to its own table when using the TPC strategy. Conversely, when using the TPH strategy, only the base type (<code>Animal</code>) can be given a table name.</p>
<blockquote><p>If multiple types in a hierarchy are given different table names, but no mapping strategy is explicitly specified, then the TPT strategy is used. This was the normal way to configure TPT prior to EF7.</p></blockquote>
<h3>Primary keys</h3>
<p>The inheritance mapping strategy chosen has consequences for how primary key values are generated and managed. Keys in TPH are easy, since each entity instance is represented by a single row in a single table. Any kind of key value generation can be used, and no additional constraints are needed.</p>
<p>For the TPT strategy, there is always a row in the table mapped to the base type of the hierarchy. Any kind of key generation can be used on this row. The keys for other tables are linked to this table using foreign key constraints. For example:</p>
<pre><code class="language-sql">CREATE TABLE [FarmAnimals] (
    [Id] int NOT NULL,
    [Value] decimal(18,2) NOT NULL,
    CONSTRAINT [PK_FarmAnimals] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_FarmAnimals_Animals_Id] FOREIGN KEY ([Id]) REFERENCES [Animals] ([Id])
);</code></pre>
<p>This ensures that same primary key value is used for a given entity in every table of the hierarchy.</p>
<p>This gets a bit more complicated when the TPC strategy is used. First, it&#8217;s important to understand that EF Core requires that all entities in a hierarchy must have a unique key value, even if the entities have different types. So, using our example model, a <code>Dog</code> cannot have the same <code>Id</code> key value as a <code>Cat</code>. Second, unlike TPT, there is no common table that can act as the single place where key values live and can be generated. This means a simple <code>Identity</code> column cannot be used.</p>
<p>For databases that support sequences, this key values can be generated by using a single sequence and referencing it in the default constraint for every table. This is the strategy used in the TPC tables shown above, where each table has the following:</p>
<pre><code class="language-sql">[Id] int NOT NULL DEFAULT (NEXT VALUE FOR [AnimalIds])</code></pre>
<p><code>AnimalIds</code> is a sequence created by EF Core migrations. The following model building code sets this up for SQL Server:</p>
<pre><code class="language-csharp">modelBuilder.HasSequence&lt;int&gt;("AnimalIds");

modelBuilder.Entity&lt;Animal&gt;()
    .UseTpcMappingStrategy()
    .Property(e =&gt; e.Id).HasDefaultValueSql("NEXT VALUE FOR [AnimalIds]");</code></pre>
<p>The syntax for the default constraint may be different for over database systems.</p>
<h3>Pros and cons of mapping strategies</h3>
<p>All of the above may be interesting, but how do you decide which strategy to use?</p>
<h4>TPH</h4>
<p>In almost all cases, TPH mapping works just fine, which is why it is the default. People are often concerned that the table can become very wide, with many columns only sparsely populated. While this can be true, it is rarely a problem with modern database systems. Query performance with TPH is always very good, mainly because no matter what query you write, only one table is ever needed to return results.</p>
<p>The most important performance differences of the different strategies stem from the SQL needed for different types of common query. To illustrate this, we will run the same three LINQ queries with each of the strategies. These queries are:</p>
<ol>
<li>A query that returns entities of all types in the hierarchy:</li>
</ol>
<pre><code class="language-csharp">context.Animals.Where(a =&gt; a.Species.StartsWith("F")).ToList();</code></pre>
<ol start="2">
<li>A query that returns entities from a subset of types in the hierarchy:</li>
</ol>
<pre><code class="language-csharp">context.Pets.Where(a =&gt; a.Species.StartsWith("F")).ToList();</code></pre>
<ol start="3">
<li>A query that returns only entities from a single leaf type in the hierarchy:</li>
</ol>
<pre><code class="language-csharp">context.Cats.Where(a =&gt; a.Species.StartsWith("F")).ToList();</code></pre>
<p>When using TPH, the SQL generated is simple and efficient in all cases. To help improve the performance of these queries, consider defining an index for faster filtering on the discriminator. In some scenarios this can a source of query slowdown compared to a table scan. However, note that SQL Server avoids using an index if it isn&#8217;t highly selective. For example, for 5 discriminator values over 1 million rows, SQL server prefers a table scan. Note that adding an index will also slows down updates, which may or may not be important.</p>
<p>Finally, if your database system supports it, then consider using sparse columns when the majority of rows will be null for that column.</p>
<ol>
<li>All types:</li>
</ol>
<pre><code class="language-sql">SELECT [a].[Id], [a].[Discriminator], [a].[Species], [a].[Value], [a].[Name], [a].[EducationLevel], [a].[FavoriteToy]
FROM [Animals] AS [a]
WHERE [a].[Species] LIKE N'F%'</code></pre>
<ol start="2">
<li>Subset of types:</li>
</ol>
<pre><code class="language-sql">SELECT [a].[Id], [a].[Discriminator], [a].[Species], [a].[Name], [a].[EducationLevel], [a].[FavoriteToy]
FROM [Animals] AS [a]
WHERE [a].[Discriminator] IN (N'Pet', N'Cat', N'Dog') AND ([a].[Species] LIKE N'F%')</code></pre>
<ol start="3">
<li>Leaf type:</li>
</ol>
<pre><code class="language-sql">SELECT [a].[Id], [a].[Discriminator], [a].[Species], [a].[Name], [a].[EducationLevel]
FROM [Animals] AS [a]
WHERE [a].[Discriminator] = N'Cat' AND ([a].[Species] LIKE N'F%')</code></pre>
<h4>TPT</h4>
<p>The TPT strategy is rarely a good choice. It is mainly used when it is considered important that the data is stored in a normalized form, which is in turn often the case for legacy existing databases or databases managed independently from the application development team.</p>
<p>The main issue with the TPT strategy is that almost all queries involve joining multiple tables because the data for any given entity instance is split across multiple tables.</p>
<p>Using the same queries again, we can see that querying for entities of all types requires all five tables to be joined:</p>
<pre><code class="language-sql">SELECT [a].[Id], [a].[Species], [f].[Value], [p].[Name], [c].[EducationLevel], [d].[FavoriteToy], CASE
    WHEN [d].[Id] IS NOT NULL THEN N'Dog'
    WHEN [c].[Id] IS NOT NULL THEN N'Cat'
    WHEN [p].[Id] IS NOT NULL THEN N'Pet'
    WHEN [f].[Id] IS NOT NULL THEN N'FarmAnimal'
END AS [Discriminator]
FROM [Animals] AS [a]
    LEFT JOIN [FarmAnimals] AS [f] ON [a].[Id] = [f].[Id]
    LEFT JOIN [Pets] AS [p] ON [a].[Id] = [p].[Id]
    LEFT JOIN [Cats] AS [c] ON [a].[Id] = [c].[Id]
    LEFT JOIN [Dogs] AS [d] ON [a].[Id] = [d].[Id]
WHERE [a].[Species] LIKE N'F%'</code></pre>
<blockquote><p>EF Core uses &#8220;discriminator synthesis&#8221; to determine which table the data comes from, and hence the correct type to use. This works because the LEFT JOIN returns nulls for the dependent ID column (the &#8220;sub-tables&#8221;) which aren&#8217;t the correct type. So for a dog, <code>[d].[Id]</code> will be non-null, and all the other (concrete) IDs will be null.</p></blockquote>
<p>Querying for entities of a subset of types still requires that the base table be joined, resulting in four tables being used:</p>
<pre><code class="language-sql">SELECT [a].[Id], [a].[Species], [p].[Name], [c].[EducationLevel], [d].[FavoriteToy], CASE
    WHEN [d].[Id] IS NOT NULL THEN N'Dog'
    WHEN [c].[Id] IS NOT NULL THEN N'Cat'
    WHEN [p].[Id] IS NOT NULL THEN N'Pet'
END AS [Discriminator]
FROM [Animals] AS [a]
    LEFT JOIN [Pets] AS [p] ON [a].[Id] = [p].[Id]
    LEFT JOIN [Cats] AS [c] ON [a].[Id] = [c].[Id]
    LEFT JOIN [Dogs] AS [d] ON [a].[Id] = [d].[Id]
WHERE ([d].[Id] IS NOT NULL OR [c].[Id] IS NOT NULL OR [p].[Id] IS NOT NULL) AND ([a].[Species] LIKE N'F%')</code></pre>
<p>And even querying for entities of just a single leaf type requires the tables for all the types that the leaf type derives from:</p>
<pre><code class="language-sql">SELECT [a].[Id], [a].[Species], [p].[Name], [c].[EducationLevel], CASE
    WHEN [c].[Id] IS NOT NULL THEN N'Cat'
END AS [Discriminator]
FROM [Animals] AS [a]
    LEFT JOIN [Pets] AS [p] ON [a].[Id] = [p].[Id]
    LEFT JOIN [Cats] AS [c] ON [a].[Id] = [c].[Id]
WHERE [c].[Id] IS NOT NULL AND ([a].[Species] LIKE N'F%')</code></pre>
<h4>TPC</h4>
<p>The TPC strategy is an improvement over TPT because it ensures that the information for a given entity instance is always stored in a single table. This means the TPC strategy can be useful when the mapped hierarchy is large and has many concrete (usually leaf) types, each with a large number of properties, and where only a small subset of types are used in most queries.</p>
<p>Using the same LINQ queries again, the SQL needed when querying for entities of all types is better than it was for TPT, since it requires one fewer table in the query. This is because there is no table for the abstract base type. In addition, <code>UNION ALL</code> is used instead of the <code>LEFT JOIN</code> needed for TPT. <code>UNION ALL</code> does not need to perform any matching between rows or de-duplication of rows, which makes it more efficient that the joins used in TPT queries.</p>
<p>All that being said, when compared to the SQL for TPH, the SQL for TPC in this case is still not great:</p>
<pre><code class="language-sql">SELECT [t].[Id], [t].[Species], [t].[Value], [t].[Name], [t].[EducationLevel], [t].[FavoriteToy], [t].[Discriminator]
FROM (
    SELECT [f].[Id], [f].[Species], [f].[Value], NULL AS [Name], NULL AS [EducationLevel], NULL AS [FavoriteToy], N'FarmAnimal' AS [Discriminator]
    FROM [FarmAnimals] AS [f]
    UNION ALL
    SELECT [p].[Id], [p].[Species], NULL AS [Value], [p].[Name], NULL AS [EducationLevel], NULL AS [FavoriteToy], N'Pet' AS [Discriminator]
    FROM [Pets] AS [p]
    UNION ALL
    SELECT [c].[Id], [c].[Species], NULL AS [Value], [c].[Name], [c].[EducationLevel], NULL AS [FavoriteToy], N'Cat' AS [Discriminator]
    FROM [Cats] AS [c]
    UNION ALL
    SELECT [d].[Id], [d].[Species], NULL AS [Value], [d].[Name], NULL AS [EducationLevel], [d].[FavoriteToy], N'Dog' AS [Discriminator]
    FROM [Dogs] AS [d]
) AS [t]
WHERE [t].[Species] LIKE N'F%'</code></pre>
<p>This is again the case when querying for entities of a subset of types:</p>
<pre><code class="language-sql">SELECT [t].[Id], [t].[Species], [t].[Name], [t].[EducationLevel], [t].[FavoriteToy], [t].[Discriminator]
FROM (
    SELECT [p].[Id], [p].[Species], [p].[Name], NULL AS [EducationLevel], NULL AS [FavoriteToy], N'Pet' AS [Discriminator]
    FROM [Pets] AS [p]
    UNION ALL
    SELECT [c].[Id], [c].[Species], [c].[Name], [c].[EducationLevel], NULL AS [FavoriteToy], N'Cat' AS [Discriminator]
    FROM [Cats] AS [c]
    UNION ALL
    SELECT [d].[Id], [d].[Species], [d].[Name], NULL AS [EducationLevel], [d].[FavoriteToy], N'Dog' AS [Discriminator]
    FROM [Dogs] AS [d]
) AS [t]
WHERE [t].[Species] IS NOT NULL AND ([t].[Species] LIKE N'F%')</code></pre>
<p>But TPC is <em>much better</em> than TPT when querying for entities of a single leaf type, since all the information for those entities comes from a single table:</p>
<pre><code class="language-sql">SELECT [c].[Id], [c].[Species], [c].[Name], [c].[EducationLevel]
FROM [Cats] AS [c]
WHERE [c].[Species] LIKE N'F%'</code></pre>
<p>These types of queries for single leaf types is where TPC really excels.</p>
<h3>Guidance</h3>
<p>In summary, the guidance for which mapping strategy to use is quite simple:</p>
<ul>
<li>If your code will mostly query for entities of a single leaf type, then use TPC. This is because:
<ul>
<li>The storage requirements are smaller, since there are no null columns and no discriminator.</li>
<li>No index is ever needed on the discriminator column, which would slow down updates and possibly also queries. An index may not be needed when using TPH either, but that depends on various factors.</li>
</ul>
</li>
<li>If your code will mostly query for entities of many types, such as writing queries against the base type, then lean towards TPH.
<ul>
<li>If your database system supports it (e.g. SQL Server), then consider using sparse columns for columns that will be rarely populated.</li>
</ul>
</li>
<li>Use TPT only if constrained to do so by external factors.</li>
</ul>
<h2>Prerequisites</h2>
<ul>
<li>EF7 currently targets .NET 6.</li>
<li>EF7 will not run on .NET Framework.</li>
</ul>
<p>EF7 is the successor to EF Core 6.0, not to be confused with <a href="https://github.com/dotnet/ef6">EF6</a>. If you are considering upgrading from EF6, please read our guide to <a href="https://docs.microsoft.com/ef/efcore-and-ef6/porting/">port from EF6 to EF Core</a>.</p>
<h2>How to get EF7 previews</h2>
<p>EF7 is distributed exclusively as a set of NuGet packages.
For example, to add the SQL Server provider to your project, you can use the following command using the dotnet tool:</p>
<pre><code class="language-bash">dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 7.0.0-preview.5.22302.2</code></pre>
<p>This following table links to the preview 5 versions of the EF Core packages and describes what they are used for.</p>
<table>
<thead>
<tr>
<th style="text-align: right;"><strong>Package</strong></th>
<th style="text-align: left;"><strong>Purpose</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore</a></td>
<td style="text-align: left;">The main EF Core package that is independent of specific database providers</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.SqlServer/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.SqlServer</a></td>
<td style="text-align: left;">Database provider for Microsoft SQL Server and SQL Azure</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite</a></td>
<td style="text-align: left;">SQL Server support for spatial types</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Sqlite/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Sqlite</a></td>
<td style="text-align: left;">Database provider for SQLite that includes the native binary for the database engine</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Sqlite.Core/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Sqlite.Core</a></td>
<td style="text-align: left;">Database provider for SQLite <em>without</em> a packaged native binary</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Sqlite.NetTopologySuite/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Sqlite.NetTopologySuite</a></td>
<td style="text-align: left;">SQLite support for spatial types</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Cosmos/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Cosmos</a></td>
<td style="text-align: left;">Database provider for Azure Cosmos DB</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.InMemory/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.InMemory</a></td>
<td style="text-align: left;">The in-memory database provider</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Tools/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Tools</a></td>
<td style="text-align: left;">EF Core PowerShell commands for the Visual Studio Package Manager Console; use this to integrate tools like <a href="https://docs.microsoft.com/ef/core/managing-schemas/scaffolding">scaffolding</a> and <a href="https://docs.microsoft.com/ef/core/managing-schemas/migrations/">migrations</a> with Visual Studio</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Design/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Design</a></td>
<td style="text-align: left;">Shared design-time components for EF Core tools</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Proxies/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Proxies</a></td>
<td style="text-align: left;">Lazy-loading and change-tracking proxies</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Abstractions/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Abstractions</a></td>
<td style="text-align: left;">Decoupled EF Core abstractions; use this for features like extended data annotations defined by EF Core</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Relational/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Relational</a></td>
<td style="text-align: left;">Shared EF Core components for relational database providers</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Analyzers/7.0.0-preview.5.22302.2">Microsoft.EntityFrameworkCore.Analyzers</a></td>
<td style="text-align: left;">C# analyzers for EF Core</td>
</tr>
</tbody>
</table>
<p>We also published the 7.0 preview 5 release of the <a href="https://www.nuget.org/packages/Microsoft.Data.Sqlite.Core/7.0.0-preview.5.22302.2">Microsoft.Data.Sqlite.Core</a> provider for <a href="https://docs.microsoft.com/dotnet/framework/data/adonet/ado-net-overview">ADO.NET</a>.</p>
<h2>Installing the EF7 Command Line Interface (CLI)</h2>
<p>Before you can execute EF7 Core migration or scaffolding commands, you&#8217;ll have to install the CLI package as either a global or local tool.</p>
<p>To install the preview tool globally, install with:</p>
<pre><code class="language-bash">dotnet tool install --global dotnet-ef --version 7.0.0-preview.5.22302.2 </code></pre>
<p>If you already have the tool installed, you can upgrade it with the following command:</p>
<pre><code class="language-bash">dotnet tool update --global dotnet-ef --version 7.0.0-preview.5.22302.2 </code></pre>
<p>It&#8217;s possible to use this new version of the EF7 CLI with projects that use older versions of the EF Core runtime.</p>
<h2>Daily builds</h2>
<p>EF7 previews are aligned with .NET 7 previews. These previews tend to lag behind the latest work on EF7. Consider using the <a href="https://github.com/aspnet/AspNetCore/blob/master/docs/DailyBuilds.md">daily builds</a> instead to get the most up-to-date EF7 features and bug fixes.</p>
<p>As with the previews, the daily builds require .NET 6.</p>
<h2>The .NET Data Community Standup</h2>
<p>The .NET data team is now live streaming every other Wednesday at 10am Pacific Time, 1pm Eastern Time, or 17:00 UTC. Join the stream to ask questions about the data-related topic of your choice, including the latest preview release.</p>
<ul>
<li><a href="https://aka.ms/efstandups">Watch our YouTube playlist</a> of previous shows</li>
<li><a href="https://live.dot.net">Visit the .NET Community Standup</a> page to preview upcoming shows</li>
<li><a href="https://github.com/dotnet/efcore/issues/22700">Submit your ideas</a> for a guest, product, demo, or other content to cover</li>
</ul>
<h2>Documentation and Feedback</h2>
<p>The starting point for all EF Core documentation is <a href="https://docs.microsoft.com/ef/">docs.microsoft.com/ef/</a>.</p>
<p>Please file issues found and any other feedback on the <a href="https://github.com/dotnet/efcore">dotnet/efcore GitHub repo</a>.</p>
<h2>Helpful Links</h2>
<p>The following links are provided for easy reference and access.</p>
<ul>
<li><a href="https://aka.ms/efstandups">EF Core Community Standup Playlist: https://aka.ms/efstandups</a></li>
<li><a href="https://aka.ms/efdocs">Main documentation: https://aka.ms/efdocs</a></li>
<li><a href="https://aka.ms/efcorefeedback">Issues and feature requests for EF Core: https://aka.ms/efcorefeedback</a></li>
<li><a href="https://aka.ms/efroadmap">Entity Framework Roadmap: https://aka.ms/efroadmap</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/27185">Bi-weekly updates: https://github.com/dotnet/efcore/issues/27185</a></li>
</ul>
<h2>Thank you from the team</h2>
<p>A big thank you from the EF team to everyone who has used and contributed to EF over the years!</p>
<p>Welcome to EF7.</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/announcing-ef7-preview5/">Announcing Entity Framework 7 Preview 5</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
