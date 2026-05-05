---
title: "Entity Framework Core 8 (EF8) is available today"
url: "https://devblogs.microsoft.com/dotnet/announcing-ef8/"
date: "Tue, 14 Nov 2023 16:00:00 +0000"
author: "Arthur Vickers"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>Entity Framework Core (EF Core) 8 is <a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/8.0.0">available on NuGet today</a>!</p>
<h2>Basic information</h2>
<p>EF Core 8, or just EF8, is the successor to EF Core 7. EF8 requires .NET 8. It will not work with .NET 6 or 7, or with any version of .NET Framework.</p>
<p>EF8 aligns with .NET 8 as a long-term support (LTS) release. See the <a href="https://dotnet.microsoft.com/platform/support/policy/dotnet-core">.NET support policy</a> for more information.</p>
<p>The following sections give an overview of the major enhancements in EF8. In total, EF8 ships with <a href="https://github.com/dotnet/efcore/issues?q=is%3Aissue+milestone%3A8.0.0+is%3Aclosed+label%3Aclosed-fixed+label%3Atype-enhancement">117 enhancements and new features, both large and small</a>, as well as <a href="https://github.com/dotnet/efcore/issues?q=is%3Aissue+milestone%3A8.0.0+is%3Aclosed+label%3Aclosed-fixed+label%3Atype-bug">128 bug fixes</a>.</p>
<blockquote>
<p><strong>TIP</strong>
Full details of all new EF8 features can be found in the <a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew"><em>What&#8217;s New in EF8</em></a> documentation. All the code is available in <a href="https://github.com/dotnet/EntityFramework.Docs">runnable samples on GitHub</a>.</p>
</blockquote>
<h2>Value objects using Complex Types</h2>
<p>Prior to EF8, there was no good way to map objects that are structured to hold multiple values, but do not have a key defining identity. For example, <code>Address</code>, <code>Coordinate</code>. <a href="https://learn.microsoft.com/ef/core/modeling/owned-entities">Owned types</a> can be used, but since owned types are actually entity types, they have semantics based on a key value, even when that key value is hidden.</p>
<p>EF8 now supports &#8220;Complex Types&#8221; to cover this type of &#8220;value object&#8221;. Complex type objects:</p>
<ul>
<li>Are not identified or tracked by key value.</li>
<li>Must be defined as part of an entity type.  (In other words, you cannot have a <code>DbSet</code> of a complex type.)</li>
<li>Can be either .NET <a href="https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-types">value types</a> or <a href="https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/reference-types">reference types</a> (Owned types must reference types.)</li>
<li>Instances can be shared by multiple properties. (Owned type instances cannot be shared.)</li>
</ul>
<h3>Simple example</h3>
<p>For example, consider an <code>Address</code> type:</p>
<pre><code class="language-csharp">public class Address
{
    public required string Line1 { get; set; }
    public string? Line2 { get; set; }
    public required string City { get; set; }
    public required string Country { get; set; }
    public required string PostCode { get; set; }
}</code></pre>
<p><code>Address</code> is then used in three places in a simple customer/orders model:</p>
<pre><code class="language-csharp">public class Customer
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public required Address Address { get; set; }
    public List&lt;Order&gt; Orders { get; } = new();
}

public class Order
{
    public int Id { get; set; }
    public required string Contents { get; set; }
    public required Address ShippingAddress { get; set; }
    public required Address BillingAddress { get; set; }
    public Customer Customer { get; set; } = null!;
}</code></pre>
<p>Let&#8217;s create and save a customer with their address:</p>
<pre><code class="language-csharp">var customer = new Customer
{
    Name = "Willow",
    Address = new() { Line1 = "Barking Gate", City = "Walpole St Peter", Country = "UK", PostCode = "PE14 7AV" }
};

context.Add(customer);
await context.SaveChangesAsync();</code></pre>
<p>This results in the following row being inserted into the database:</p>
<pre><code class="language-sql">INSERT INTO [Customers] ([Name], [Address_City], [Address_Country], [Address_Line1], [Address_Line2], [Address_PostCode])
OUTPUT INSERTED.[Id]
VALUES (@p0, @p1, @p2, @p3, @p4, @p5);</code></pre>
<p>Notice that the complex types do not get their own tables. Instead, they are saved inline to columns of the <code>Customers</code> table. This matches the table sharing behavior of owned types.</p>
<blockquote>
<p><strong>NOTE</strong>
We don&#8217;t plan to allow complex types to be mapped to their own table. However, in a future release, we do plan to allow the complex type to be saved as a JSON document in a single column. Vote for <a href="https://github.com/dotnet/efcore/issues/31252">Issue #31252</a> if this is important to you.</p>
</blockquote>
<p>Now let&#8217;s say we want to ship an order to a customer and use the customer&#8217;s address as both the default billing an shipping address. The natural way to do this is to copy the <code>Address</code> object from the <code>Customer</code> into the <code>Order</code>. For example:</p>
<pre><code class="language-csharp">customer.Orders.Add(
    new Order { Contents = "Tesco Tasty Treats", BillingAddress = customer.Address, ShippingAddress = customer.Address, });

await context.SaveChangesAsync();</code></pre>
<p>With complex types, this works as expected, and the address is inserted into the <code>Orders</code> table:</p>
<pre><code class="language-sql">INSERT INTO [Orders] ([Contents], [CustomerId],
    [BillingAddress_City], [BillingAddress_Country], [BillingAddress_Line1], [BillingAddress_Line2], [BillingAddress_PostCode],
    [ShippingAddress_City], [ShippingAddress_Country], [ShippingAddress_Line1], [ShippingAddress_Line2], [ShippingAddress_PostCode])
OUTPUT INSERTED.[Id]
VALUES (@p0, @p1, @p2, @p3, @p4, @p5, @p6, @p7, @p8, @p9, @p10, @p11);</code></pre>
<p>If we try the same thing with owned types, EF warns and then throws:</p>
<pre><code class="language-text">warn: 8/20/2023 12:48:01.678 CoreEventId.DuplicateDependentEntityTypeInstanceWarning[10001] (Microsoft.EntityFrameworkCore.Update) 
      The same entity is being tracked as different entity types 'Order.BillingAddress#Address' and 'Customer.Address#Address' with defining navigations. If a property value changes, it will result in two store changes, which might not be the desired outcome.
fail: 8/20/2023 12:48:01.709 CoreEventId.SaveChangesFailed[10000] (Microsoft.EntityFrameworkCore.Update) 
      An exception occurred in the database while saving changes for context type 'NewInEfCore8.ComplexTypesSample+CustomerContext'.
      System.InvalidOperationException: Cannot save instance of 'Order.ShippingAddress#Address' because it is an owned entity without any reference to its owner. Owned entities can only be saved as part of an aggregate also including the owner entity.
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.InternalEntityEntry.PrepareToSave()</code></pre>
<p>This is because a single instance of the <code>Address</code> entity type (with the same hidden key value) is being used for three <em>different</em> entity instances. On the other hand, sharing the same instance between complex properties is allowed, and so the code works as expected when using complex types.</p>
<h3>Immutable records as complex types</h3>
<p>The .NET type used for a complex type in the EF model can be:</p>
<ul>
<li>.NET <a href="https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-types">value types</a> or <a href="https://learn.microsoft.com/dotnet/csharp/language-reference/keywords/reference-types">reference types</a></li>
<li>Mutable or Immutable</li>
<li>A C# <a href="https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record">record type</a> </li>
</ul>
<p>The <a href="https://aka.ms/ef8-new"><em>What&#8217;s new in EF8</em></a> documentation covers all of these possibilities, but <a href="https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/struct#record-struct">immutable record value types</a> are usually the best fit for representing complex types.  </p>
<p>For example, let&#8217;s use the following immutable struct record to represent the address: </p>
<pre><code class="language-csharp">public readonly record struct Address(string Line1, string? Line2, string City, string Country, string PostCode);</code></pre>
<p>We can now use the <code>with</code> syntax to update the <code>customer.Address</code> property with a new object that has one or more properties changes. For example: </p>
<p>The code for changing the address now looks the same as when using immutable class record:</p>
<pre><code class="language-csharp">customer.Address = customer.Address with { Line1 = "Peacock Lodge" };

await context.SaveChangesAsync();</code></pre>
<h3>Current limitations</h3>
<p>Complex types represent a significant investment across the EF stack. We were not able to make everything work in this release, but we plan to close some of the gaps in a future release. Make sure to vote (<img alt="👍" class="wp-smiley" src="https://s.w.org/images/core/emoji/17.0.2/72x72/1f44d.png" style="height: 1em;" />) on the appropriate GitHub issues if fixing any of these limitations is important to you.</p>
<p>Complex type limitations in EF8 include:</p>
<ul>
<li>Support collections of complex types. (<a href="https://github.com/dotnet/efcore/issues/31237">Issue #31237</a>)</li>
<li>Allow complex type properties to be null. (<a href="https://github.com/dotnet/efcore/issues/31376">Issue #31376</a>)</li>
<li>Map complex type properties to JSON columns. (<a href="https://github.com/dotnet/efcore/issues/31252">Issue #31252</a>)</li>
<li>Constructor injection for complex types. (<a href="https://github.com/dotnet/efcore/issues/31621">Issue #31621</a>)</li>
<li>Add seed data support for complex types. (<a href="https://github.com/dotnet/efcore/issues/31254">Issue #31254</a>)</li>
<li>Map complex type properties for the Cosmos provider. (<a href="https://github.com/dotnet/efcore/issues/31253">Issue #31253</a>)</li>
<li>Implement complex types for the in-memory database. (<a href="https://github.com/dotnet/efcore/issues/31464">Issue #31464</a>)</li>
</ul>
<h3>More information on complex types</h3>
<p>For more information on complex types, see:</p>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#value-objects-using-complex-types">What&#8217;s new in EF8: Complex types</a> in the EF documentation.</li>
<li><a href="https://www.youtube.com/live/H-soJYqWSds?si=lbq9OsdsPn6BnwXf">.NET Data Community Standup recording</a> on YouTube.</li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-rc1/">Complex types as value objects</a> on the .NET blog.</li>
</ul>
<h2>Primitive collections</h2>
<p>A persistent question when using relational databases is what to do with collections of primitive types; that is, lists or arrays of integers, date/times, strings, and so on. If you&#8217;re using PostgreSQL, then its easy to store these things using PostgreSQL&#8217;s <a href="https://www.postgresql.org/docs/current/arrays.html">built-in array type</a>. For other databases, a common approach is to serialize the primitive collection into a type that is handled by the database&#8211;for example, serialize to and from a string with comma delimiters.</p>
<p>EF8 now includes built-in support for this kind of mapping, using JSON as the serialization format. JSON works well for this since modern relational databases include built-in mechanisms for querying and manipulating JSON, such that the JSON column can, effectively, be treated as a table when needed, without the overhead of actually creating that table.</p>
<h3>Primitive collection properties</h3>
<p>EF Core can map ordered collections of primitive types to a JSON column in the database. The collection property must be typed as <code>IEnumerable&lt;T&gt;</code>, where <code>T</code> is a primitive type, and at runtime the collection object must implement <code>IList&lt;T&gt;</code>, indicating that it is ordered and supports random access. </p>
<p>For example, all properties in the following entity type are mapped to JSON columns by convention:</p>
<pre><code class="language-csharp">public class PrimitiveCollections
{
    public IList&lt;DateOnly&gt; Dates { get; set; }
    public uint[] UnsignedInts { get; set; }
    public List&lt;bool&gt; Booleans { get; set; }
    public List&lt;Uri&gt; Urls { get; set; }
    public IEnumerable&lt;int&gt; Ints { get; set; } // Must be an IList&lt;int&gt;() at runtime.
    public ICollection&lt;string&gt; Strings { get; set; } // Must be an IList&lt;int&gt;() at runtime.
}</code></pre>
<p>Let&#8217;s look at a query that makes use of a column containing a list of dates. For example, using this entity type to represent a British Public House:</p>
<pre><code class="language-csharp">public class Pub
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public required string[] Beers { get; set; }
    public List&lt;DateOnly&gt; DaysVisited { get; private set; } = new();
}</code></pre>
<p>We can write a query to find pubs visited this year:</p>
<pre><code class="language-csharp">var thisYear = DateTime.Now.Year;
var pubsVisitedThisYear = await context.Pubs
    .Where(e =&gt; e.DaysVisited.Any(v =&gt; v.Year == thisYear))
    .Select(e =&gt; e.Name)
    .ToListAsync();</code></pre>
<p>This translates to the following on SQL Server:</p>
<pre><code class="language-sql">SELECT [p].[Name]
FROM [Pubs] AS [p]
WHERE EXISTS (
    SELECT 1
    FROM OPENJSON([p].[DaysVisited]) AS [d]
    WHERE DATEPART(year, CAST([d].[value] AS date)) = @__thisYear_0)</code></pre>
<p>EF is using the <a href="https://learn.microsoft.com/sql/t-sql/functions/OPENJSON-transact-sql?view=sql-server-ver16">SQL Server OPENJSON function</a> to parse the JSON saved into the <code>DaysVisited</code> column and treat it like a table. Notice that the query makes use of the date-specific function <code>DATEPART</code> here because EF <em>knows that the primitive collection contains dates</em>. It might not seem like it, but this is actually really important. Because EF knows what&#8217;s in the collection, it can generate appropriate SQL to use the typed values with parameters, functions, other columns etc.</p>
<h3>Primitive collections in JSON documents</h3>
<p>Primitive collections embedded in <a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-7.0/whatsnew#json-columns">an owned entity type to a column containing a JSON document</a>, which was introduced in EF7, can be persisted and queried in the same way.</p>
<p>For example, the following query extracts data from the JSON document, including use of sub-queries into the primitive collections contained in the document:</p>
<pre><code class="language-csharp">var walksWithADrink = await context.Walks.Select(
    w =&gt; new
    {
        WalkName = w.Name,
        PubName = w.ClosestPub.Name,
        WalkLocationTag = w.Visits.LocationTag,
        PubLocationTag = w.ClosestPub.Visits.LocationTag,
        Count = w.Visits.DaysVisited.Count(v =&gt; w.ClosestPub.Visits.DaysVisited.Contains(v)),
        TotalCount = w.Visits.DaysVisited.Count
    }).ToListAsync();</code></pre>
<p>This translates to the following on SQL Server:</p>
<pre><code class="language-sql">SELECT [w].[Name] AS [WalkName], [p].[Name] AS [PubName], JSON_VALUE([w].[Visits], '$.LocationTag') AS [WalkLocationTag], JSON_VALUE([p].[Visits], '$.LocationTag') AS [PubLocationTag], (
    SELECT COUNT(*)
    FROM OPENJSON(JSON_VALUE([w].[Visits], '$.DaysVisited')) AS [d]
    WHERE EXISTS (
        SELECT 1
        FROM OPENJSON(JSON_VALUE([p].[Visits], '$.DaysVisited')) AS [d0]
        WHERE CAST([d0].[value] AS date) = CAST([d].[value] AS date) OR ([d0].[value] IS NULL AND [d].[value] IS NULL))) AS [Count], (
    SELECT COUNT(*)
    FROM OPENJSON(JSON_VALUE([w].[Visits], '$.DaysVisited')) AS [d1]) AS [TotalCount]
FROM [Walks] AS [w]
INNER JOIN [Pubs] AS [p] ON [w].[ClosestPubId] = [p].[Id]</code></pre>
<h3>Better Contains queries</h3>
<p>The use of JSON to represent primitive collections has opened several new query translations that make use of the JSON capabilities of relational databases to create what are effectively inline, temporary tables of values. This is very powerful. For example, consider the following entity type:</p>
<pre><code class="language-csharp">public class DogWalk
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public Terrain Terrain { get; set; }
    public List&lt;DateOnly&gt; DaysVisited { get; private set; } = new();
    public Pub? ClosestPub { get; set; }
}

public enum Terrain
{
    Forest,
    River,
    Hills,
    Village,
    Park,
    Beach,
}</code></pre>
<p>Using this model, we can write simple <code>Contains</code> query to find all walks with one of several different terrains:</p>
<pre><code class="language-csharp">var terrains = new[] { Terrain.River, Terrain.Beach, Terrain.Park };
var walksWithTerrain = await context.Walks
    .Where(e =&gt; terrains.Contains(e.Terrain))
    .Select(e =&gt; e.Name)
    .ToListAsync();</code></pre>
<p>This is already translated by current versions of EF Core by inlining the values to look for. For example, when using SQL Server:</p>
<pre><code class="language-sql">SELECT [w].[Name]
FROM [Walks] AS [w]
WHERE [w].[Terrain] IN (1, 5, 4)</code></pre>
<p>However, this strategy does not work well with database query caching; see <a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-4/">Announcing EF8 Preview 4</a> on the .NET Blog for a discussion of the issue.</p>
<p>For EF8, the default is now to pass the list of terrains as a single parameter containing a JSON collection. For example:</p>
<pre><code class="language-none">@__terrains_0='[1,5,4]'</code></pre>
<p>The query then uses <code>OPENJSON</code> on SQL Server:</p>
<pre><code class="language-sql">SELECT [w].[Name]
FROM [Walks] AS [w]
WHERE EXISTS (
    SELECT 1
    FROM OPENJSON(@__terrains_0) AS [t]
    WHERE CAST([t].[value] AS int) = [w].[Terrain])</code></pre>
<p>Or <code>json_each</code> on SQLite:</p>
<pre><code class="language-sql">SELECT "w"."Name"
FROM "Walks" AS "w"
WHERE EXISTS (
    SELECT 1
    FROM json_each(@__terrains_0) AS "t"
    WHERE "t"."value" = "w"."Terrain")</code></pre>
<h3>More information on primitive collections</h3>
<p>For more information on complex types, see:</p>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#primitive-collections">What&#8217;s new in EF8: Primitive collections</a> in the EF documentation.</li>
<li><a href="https://www.youtube.com/live/AUS2OZjsA2I?si=tpcPyw9FxmGn6kwq">.NET Data Community Standup recording</a> on YouTube.</li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-4/">Primitive collections and improved Contains</a> on the .NET blog.</li>
</ul>
<h2>Enhancements to JSON column mapping</h2>
<p>EF8 includes improvements to the <a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-7.0/whatsnew#json-columns">JSON column mapping support introduced in EF7</a>.</p>
<h3>Translate element access into JSON arrays</h3>
<p>EF8 supports indexing in JSON arrays when executing queries. For example, the following query checks whether the first two updates were made before a given date.</p>
<pre><code class="language-csharp">var cutoff = DateOnly.FromDateTime(DateTime.UtcNow - TimeSpan.FromDays(365));
var updatedPosts = await context.Posts
    .Where(
        p =&gt; p.Metadata!.Updates[0].UpdatedOn &lt; cutoff
             &amp;&amp; p.Metadata!.Updates[1].UpdatedOn &lt; cutoff)
    .ToListAsync();</code></pre>
<p>This translates into the following SQL when using SQL Server:</p>
<pre><code class="language-sql">SELECT [p].[Id], [p].[Archived], [p].[AuthorId], [p].[BlogId], [p].[Content], [p].[Discriminator], [p].[PublishedOn], [p].[Title], [p].[PromoText], [p].[Metadata]
FROM [Posts] AS [p]
WHERE CAST(JSON_VALUE([p].[Metadata],'$.Updates[0].UpdatedOn') AS date) &lt; @__cutoff_0
  AND CAST(JSON_VALUE([p].[Metadata],'$.Updates[1].UpdatedOn') AS date) &lt; @__cutoff_0</code></pre>
<h3>JSON Columns for SQLite and PostgreSQL</h3>
<p>EF7 introduced support for mapping to JSON columns when using Azure SQL/SQL Server. EF8 extends this support to SQLite databases, and the <a href="https://www.nuget.org/packages/Npgsql.EntityFrameworkCore.PostgreSQL/">Npgsql.EntityFrameworkCore.PostgreSQL</a> EF Core provider brings this same support to PostgreSQL databases. As for the SQL Server support, this includes:</p>
<ul>
<li>Mapping of aggregates built from .NET types to JSON documents stored in columns</li>
<li>Queries into JSON columns, such as filtering and sorting by the elements of the documents</li>
<li>Queries that project elements out of the JSON document into results</li>
<li>Updating and saving changes to JSON documents</li>
</ul>
<p>The existing <a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-7.0/whatsnew#json-columns">documentation from What&#8217;s New in EF7</a> provides detailed information on JSON mapping, queries, and updates. This documentation now also applies to SQLite and PostgreSQL.</p>
<h2>HierarchyId in .NET and EF Core</h2>
<p>Azure SQL and SQL Server have a special data type called <a href="https://learn.microsoft.com/sql/t-sql/data-types/hierarchyid-data-type-method-reference?view=sql-server-ver16"><code>hierarchyid</code></a> that is used to store <a href="https://learn.microsoft.com/sql/relational-databases/hierarchical-data-sql-server?view=sql-server-ver16">hierarchical data</a>. In this case, &#8220;hierarchical data&#8221; essentially means data that forms a tree structure, where each item can have a parent and/or children. Examples of such data are:</p>
<ul>
<li>An organizational structure</li>
<li>A file system</li>
<li>A set of tasks in a project</li>
<li>A taxonomy of language terms</li>
<li>A graph of links between Web pages</li>
</ul>
<p>The database is then able to run queries against this data using its hierarchical structure. For example, a query can find ancestors and dependents of given items, or find all items at a certain depth in the hierarchy.</p>
<h3>Modeling hierarchies</h3>
<p>The <code>HierarchyId</code> type can be used for properties of an entity type. For example, assume we want to model the paternal family tree of some fictional <a href="https://en.wikipedia.org/wiki/Halfling">halflings</a>. In the entity type for <code>Halfling</code>, a <code>HierarchyId</code> property can be used to locate each halfling in the family tree.</p>
<pre><code class="language-csharp">public class Halfling
{
    public Halfling(HierarchyId pathFromPatriarch, string name, int? yearOfBirth = null)
    {
        PathFromPatriarch = pathFromPatriarch;
        Name = name;
        YearOfBirth = yearOfBirth;
    }

    public int Id { get; private set; }
    public HierarchyId PathFromPatriarch { get; set; }
    public string Name { get; set; }
    public int? YearOfBirth { get; set; }
}</code></pre>
<p>In this case, the family tree is rooted with the patriarch of the family. Each halfling can be traced from the patriarch down the tree using its <code>PathFromPatriarch</code> property. SQL Server uses a compact binary format for these paths, but it is common to parse to and from a human-readable string representation when when working with code. In this representation, the position at each level is separated by a <code>/</code> character. For example, consider the family tree in the diagram below:</p>
<p><img alt="Halfling family tree" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2023/11/familytree.png" /></p>
<h3>Querying hierarchies</h3>
<p>The following query uses <code>GetAncestor</code> to find the direct ancestor of a halfling, given that halfling&#8217;s name:</p>
<pre><code class="language-csharp">async Task&lt;Halfling?&gt; FindDirectAncestor(string name)
    =&gt; await context.Halflings
        .SingleOrDefaultAsync(
            ancestor =&gt; ancestor.PathFromPatriarch == context.Halflings
                .Single(descendent =&gt; descendent.Name == name).PathFromPatriarch
                .GetAncestor(1));</code></pre>
<p>This translates to the following SQL:</p>
<pre><code class="language-sql">SELECT TOP(2) [h].[Id], [h].[Name], [h].[PathFromPatriarch], [h].[YearOfBirth]
FROM [Halflings] AS [h]
WHERE [h].[PathFromPatriarch] = (
    SELECT TOP(1) [h0].[PathFromPatriarch]
    FROM [Halflings] AS [h0]
    WHERE [h0].[Name] = @__name_0).GetAncestor(1)</code></pre>
<p>Running this query for the halfling &#8220;Bilbo&#8221; returns &#8220;Bungo&#8221;.</p>
<h3>Updating hierarchies</h3>
<p>The normal <a href="https://learn.microsoft.com/ef/core/change-tracking/">change tracking</a> and <a href="https://learn.microsoft.com/ef/core/saving/basic">SaveChanges</a> mechanisms can be used to update <code>hierarchyid</code> columns.</p>
<p>For example, I&#8217;m sure we all remember the scandal of SR 1752 (a.k.a. &#8220;LongoGate&#8221;) when DNA testing revealed that Longo was not in fact the son of Mungo, but actually the son of Ponto! One fallout from this scandal was that the family tree needed to be re-written. In particular, Longo and all his descendents needed to be re-parented from Mungo to Ponto. <code>GetReparentedValue</code> can be used to do this. For example, first &#8220;Longo&#8221; and all his descendents are queried:</p>
<pre><code class="language-csharp">var longoAndDescendents = await context.Halflings.Where(
        descendent =&gt; descendent.PathFromPatriarch.IsDescendantOf(
            context.Halflings.Single(ancestor =&gt; ancestor.Name == "Longo").PathFromPatriarch))
    .ToListAsync();</code></pre>
<p>Then <code>GetReparentedValue</code> is used to update the <code>HierarchyId</code> for Longo and each descendent, followed by a call to <code>SaveChangesAsync</code>:</p>
<pre><code class="language-csharp">foreach (var descendent in longoAndDescendents)
{
    descendent.PathFromPatriarch
        = descendent.PathFromPatriarch.GetReparentedValue(
            mungo.PathFromPatriarch, ponto.PathFromPatriarch)!;
}

await context.SaveChangesAsync();</code></pre>
<pre><code class="language-sql">SET NOCOUNT ON;
UPDATE [Halflings] SET [PathFromPatriarch] = @p0
OUTPUT 1
WHERE [Id] = @p1;
UPDATE [Halflings] SET [PathFromPatriarch] = @p2
OUTPUT 1
WHERE [Id] = @p3;
UPDATE [Halflings] SET [PathFromPatriarch] = @p4
OUTPUT 1
WHERE [Id] = @p5;</code></pre>
<p>Using these parameters:</p>
<pre><code class="language-text"> @p1='9',
 @p0='0x7BC0' (Nullable = false) (Size = 2) (DbType = Object),
 @p3='16',
 @p2='0x7BD6' (Nullable = false) (Size = 2) (DbType = Object),
 @p5='23',
 @p4='0x7BD6B0' (Nullable = false) (Size = 3) (DbType = Object)</code></pre>
<blockquote>
<p><strong>NOTE</strong>
The parameters values for <code>HierarchyId</code> properties are sent to the database in their compact, binary format.</p>
</blockquote>
<p>Following the update, querying for the descendents of &#8220;Mungo&#8221; returns &#8220;Bungo&#8221;, &#8220;Belba&#8221;, &#8220;Linda&#8221;, &#8220;Bingo&#8221;, &#8220;Bilbo&#8221;, &#8220;Falco&#8221;, and &#8220;Poppy&#8221;, while querying for the descendents of &#8220;Ponto&#8221; returns &#8220;Longo&#8221;, &#8220;Rosa&#8221;, &#8220;Polo&#8221;, &#8220;Otho&#8221;, &#8220;Posco&#8221;, &#8220;Prisca&#8221;, &#8220;Lotho&#8221;, &#8220;Ponto&#8221;, &#8220;Porto&#8221;, &#8220;Peony&#8221;, and &#8220;Angelica&#8221;.</p>
<h3>More information hierarchies and EF Core</h3>
<p>For more information on mapping hierarchies with EF Core, see:</p>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#hierarchyid-in-net-and-ef-core">What&#8217;s new in EF8: HierarchyId</a> in the EF documentation.</li>
<li><a href="https://www.youtube.com/live/pmnHGWYpCfg?si=NOrlBo1TVKmT7biO">.NET Data Community Standup recording</a> on YouTube.</li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-2/">Lite and familiar</a> on the .NET blog.</li>
</ul>
<h2>Raw SQL queries for unmapped types</h2>
<p>EF7 introduced <a href="https://learn.microsoft.com/ef/core/querying/sql-queries#querying-scalar-(non-entity)-types">raw SQL queries returning scalar types</a>. This is enhanced in EF8 to include raw SQL queries returning any mappable CLR type, without including that type in the EF model.</p>
<p>Queries using unmapped types are executed using <a href="https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.relationaldatabasefacadeextensions.sqlquery">SqlQuery</a> or <a href="https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.relationaldatabasefacadeextensions.sqlqueryraw">SqlQueryRaw</a> The former uses string interpolation to parameterize the query, which helps ensure that all non-constant values are parameterized. </p>
<p>The types used for SQL queries must have a property for every value in the result set, but do not need to match any specific table in the database. For example, the following type represents only a subset of information for each post, and includes the blog name, which comes from the <code>Blogs</code> table:</p>
<pre><code class="language-csharp">public class PostSummary
{
    public string BlogName { get; set; }
    public string PostTitle { get; set; }
    public DateOnly PublishedOn { get; set; }
}</code></pre>
<p>Instances of this type can be returned using <code>SqlQuery</code>:</p>
<pre><code class="language-csharp">var summaries =
    await context.Database.SqlQuery&lt;PostSummary&gt;(
            @$"SELECT b.Name AS BlogName, p.Title AS PostTitle, p.PublishedOn
            FROM Posts AS p
            INNER JOIN Blogs AS b ON p.BlogId = b.Id")
        .ToListAsync();</code></pre>
<h2>More features in EF8</h2>
<p>The <em>What&#8217;s new in EF8</em> documentation covers some additional interesting enhancements in EF8, including:</p>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#lazy-loading-for-no-tracking-queries">Lazy-loading for no-tracking queries</a></li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#opt-out-of-lazy-loading-for-specific-navigations">Opt-out of lazy-loading for specific navigations</a></li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#lookup-tracked-entities-by-primary-alternate-or-foreign-key">Lookup tracked entities by primary, alternate, or foreign key</a></li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#discriminator-columns-have-max-length">Discriminator columns have max length</a></li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#dateonlytimeonly-supported-on-sql-server">DateOnly/TimeOnly supported on SQL Server</a></li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#enhancements-to-math-translations">Enhancements to Math translations</a></li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#checking-for-pending-model-changes">Checking for pending model changes</a></li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew#enhancements-to-sqlite-scaffolding">Enhancements to SQLite scaffolding</a></li>
</ul>
<h2>How to get EF8</h2>
<p>EF8 is distributed exclusively as a set of NuGet packages. For example, to add the SQL Server provider to your project, you can use the following command using the dotnet tool:</p>
<pre><code class="language-bash">dotnet add package Microsoft.EntityFrameworkCore.SqlServer</code></pre>
<h2>Installing the EF8 Command Line Interface (CLI)</h2>
<p>The <code>dotnet-ef</code> tool must be installed before executing EF8 Core migration or scaffolding commands.</p>
<p>To install the tool globally, use:</p>
<pre><code class="language-bash">dotnet tool install --global dotnet-ef</code></pre>
<p>If you already have the tool installed, you can upgrade it with the following command:</p>
<pre><code class="language-bash">dotnet tool update --global dotnet-ef</code></pre>
<h2>The .NET Data Community Standup</h2>
<p>The .NET data access team is now live streaming every other Wednesday at 10am Pacific Time, 1pm Eastern Time, or 18:00 UTC. Join the stream learn and ask questions about many .NET Data related topics.</p>
<ul>
<li><a href="https://aka.ms/efstandups">Watch our YouTube playlist</a> of previous shows</li>
<li><a href="https://live.dot.net">Visit the .NET Community Standup</a> page to preview upcoming shows</li>
<li><a href="https://github.com/dotnet/efcore/issues/22700">Submit your ideas</a> for a guest, product, demo, or other content to cover</li>
</ul>
<h2>Documentation and Feedback</h2>
<p>The starting point for all EF Core documentation is <a href="https://docs.microsoft.com/ef/">docs.microsoft.com/ef/</a>. Please file issues found and any other feedback on the <a href="https://github.com/dotnet/efcore">dotnet/efcore GitHub repo</a>.</p>
<h2>Helpful Links</h2>
<p>The following links are provided for easy reference and access.</p>
<ul>
<li>EF Core Community Standup Playlist: <a href="https://aka.ms/efstandups">aka.ms/efstandups</a></li>
<li>Main documentation: <a href="https://aka.ms/efdocs">aka.ms/efdocs</a></li>
<li>What&#8217;s New in EF Core 8: <a href="https://aka.ms/ef8-new">aka.ms/ef8-new</a></li>
<li>What&#8217;s New in EF Core 7: <a href="https://aka.ms/ef7-new">aka.ms/ef7-new</a></li>
<li>Issues and feature requests for EF Core: <a href="https://github.com/dotnet/efcore/issues">github.com/dotnet/efcore/issues</a></li>
<li>Entity Framework Roadmap: <a href="https://aka.ms/efroadmap">aka.ms/efroadmap</a></li>
</ul>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/announcing-ef8/">Entity Framework Core 8 (EF8) is available today</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
