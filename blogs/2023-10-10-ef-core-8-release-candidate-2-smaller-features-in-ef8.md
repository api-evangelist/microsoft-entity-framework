---
title: "EF Core 8 Release Candidate 2: Smaller features in EF8"
url: "https://devblogs.microsoft.com/dotnet/announcing-ef8-rc2/"
date: "Tue, 10 Oct 2023 17:10:00 +0000"
author: "Arthur Vickers"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>The second release candidate of Entity Framework Core (EF Core) 8 is <a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/8.0.0-rc.2.23480.1">available on NuGet today</a>!</p>
<h2>Basic information</h2>
<p>EF Core 8, or just EF8, is the successor to EF Core 7, and is scheduled for release in November 2023, at the same time as .NET 8.</p>
<p>EF8 requires .NET 8 and this RC 2 release should be used with the <a href="https://dotnet.microsoft.com/next">.NET 8 RC 2 SDK</a>.</p>
<p>EF8 will align with .NET 8 as a long-term support (LTS) release. See the <a href="https://dotnet.microsoft.com/platform/support/policy/dotnet-core">.NET support policy</a> for more information.</p>
<h2>New in EF8</h2>
<p>In this post we&#8217;re going to take at a few of the smaller features included in EF8. Be sure to check out EF8 content from previous posts:</p>
<ul>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-rc1/">Complex types as value objects</a></li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-4/">Primitive collections and improved Contains</a></li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-1/#raw-sql-queries-for-unmapped-types">Raw SQL queries for unmapped types</a></li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-1/#lazy-loading-for-no-tracking-queries">Lazy-loading for no-tracking queries</a></li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-1/#dateonly-timeonly-supported-on-sql-server">DateOnly/TimeOnly supported on SQL Server</a></li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-2/#sql-server-hierarchyid">SQL Server HierarchyId</a></li>
<li><a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-2/#json-columns-for-sqlite">JSON Columns for SQLite</a></li>
</ul>
<h2>Sentinel values and database defaults</h2>
<p>Databases allow columns to be configured to generate a default value if no value is provided when inserting a row. This can be represented in EF using <code>HasDefaultValue</code> for constants:</p>
<pre><code class="language-csharp">b.Property(e =&gt; e.Status).HasDefaultValue("Hidden");</code></pre>
<p>Or <code>HasDefaultValueSql</code> for arbitrary SQL clauses:</p>
<pre><code class="language-csharp">b.Property(e =&gt; e.LeaseDate).HasDefaultValueSql("getutcdate()");</code></pre>
<p>In order for EF to make use of this, it must determine when and when not to send a value for the column. By default, EF uses the CLR default as a sentinel for this. That is, when the value of <code>Status</code> or <code>LeaseDate</code> in the examples above are the CLR defaults for these types, then EF <em>interprets that to mean that the property has not been set</em>, and so does not send a value to the database. This works well for reference types&#8211;for example, if the <code>string</code> property <code>Status</code> is <code>null</code>, then EF doesn&#8217;t send <code>null</code> to the database, but rather does not include any value so that the database default (<code>"Hidden"</code>) is used. Likewise, for the <code>DateTime</code> property <code>LeaseDate</code>, EF will not insert the CLR default value of <code>1/1/0001 12:00:00 AM</code>, but will instead omit this value so that database default is used.</p>
<p>However, in some cases the CLR default value is a valid value to insert. EF8 handles this by allowing the sentinel value for a colum to change. For example, consider an integer column configured with a database default:</p>
<pre><code class="language-csharp">b.Property(e =&gt; e.Credits).HasDefaultValueSql(10);</code></pre>
<p>In this case, we want the new entity to be inserted with the given number of credits, unless this is not specified, in which case 10 credits are assigned. However, this means that inserting a record with zero credits is not possible, since zero is the CLR default, and hence will cause EF to send no value. In EF8, this can be fixed by changing the sentinel for the property from zero (the CLR default) to <code>-1</code>:</p>
<pre><code class="language-csharp">b.Property(e =&gt; e.Credits).HasDefaultValueSql(10).HasSentinel(-1);</code></pre>
<p>EF will now only use the database default if <code>Credits</code> is set to <code>-1</code>; a value of zero will be inserted like any other amount.</p>
<p>Check out the <a href="https://www.youtube.com/live/GGDv_p4LAL8?si=6kbx53JjCH0OB_f6">.NET Data Community Standup</a> for a more in-dept discussion on this subject including examples showing how sentinels can help prevent bugs with <code>enum</code> and <code>bool</code> values.</p>
<h3>Tip: using a nullable backing field</h3>
<p>Another way to handle the same problem is to use a nullable backing field for the property. For example, instead of defining the <code>Credits</code> property as:</p>
<pre><code class="language-csharp">public class User
{
    public int Credits { get; set; }
}</code></pre>
<p>It can be defined as:</p>
<pre><code class="language-csharp">public class User
{
    private int? _credits;

    public int Credits
    {
        get =&gt; _credits ?? 0;
        set =&gt; _credits = value;
    }
}</code></pre>
<p>The backing field here will remain null <em>unless the property setter is actually called</em>. That is, the value of the backing field is a better indication of whether the property has been set or not than the CLR default of the property. This works out-of-the box with EF, since EF will use the backing field to read and write the property by default.</p>
<h2>Better ExecuteUpdate and ExecuteDelete</h2>
<p>SQL commands that perform updates and deletes, such as those generated by <code>ExecuteUpdate</code> and <code>ExecuteDelete</code> methods, must target a single database table. However, in EF7, <code>ExecuteUpdate</code> and <code>ExecuteDelete</code> did not support updates accessing multiple entity types <em>even when the query ultimately affected a single table</em>. EF8 removes this limitation. For example, consider a <code>Customer</code> entity type with <code>CustomerInfo</code> owned type:</p>
<pre><code class="language-csharp">[Table("Customers")]
public class Customer
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public required CustomerInfo CustomerInfo { get; set; }
}

[Owned]
public class CustomerInfo
{
    public string? Tag { get; set; }
}</code></pre>
<p>Both of these entity types map to the <code>Customers</code> table. However, the following bulk update fails on EF7 because it uses both entity types:</p>
<pre><code class="language-csharp">await context.Customers
    .Where(e =&gt; e.Name == name)
    .ExecuteUpdateAsync(
        s =&gt; s.SetProperty(b =&gt; b.CustomerInfo.Tag, "Tagged")
            .SetProperty(b =&gt; b.Name, b =&gt; b.Name + "_Tagged"));</code></pre>
<p>In EF8, this now translates to the following SQL when using Azure SQL:</p>
<pre><code class="language-sql">UPDATE [c]
SET [c].[Name] = [c].[Name] + N'_Tagged',
    [c].[CustomerInfo_Tag] = N'Tagged'
FROM [Customers] AS [c]
WHERE [c].[Name] = @__name_0</code></pre>
<p>Check out the <a href="https://www.youtube.com/live/GGDv_p4LAL8?si=6kbx53JjCH0OB_f6">.NET Data Community Standup</a> for other examples using <code>Union</code> queries and TPT inheritance mapping.</p>
<h2>Better use of <code>IN</code> queries</h2>
<p>When the Contains operator is used with a subquery, EF Core now generates better queries using SQL <code>IN</code> instead of <code>EXISTS</code>; aside from producing more readable SQL, in some cases this can result in dramatically faster queries. For example, consider the following LINQ query:</p>
<pre><code class="language-csharp">var blogsWithPosts = await context.Blogs
    .Where(b =&gt; context.Posts.Select(p =&gt; p.BlogId).Contains(b.Id))
    .ToListAsync();</code></pre>
<p>EF7 generates the following for PostgreSQL:</p>
<pre><code class="language-sql">SELECT b."Id", b."Name"
      FROM "Blogs" AS b
      WHERE EXISTS (
          SELECT 1
          FROM "Posts" AS p
          WHERE p."BlogId" = b."Id")</code></pre>
<p>Since the subquery references the external <code>Blogs</code> table (via <code>b."Id"</code>), this is a <em>correlated subquery</em>, meaning that the <code>Posts</code> subquery must be executed for each row in the <code>Blogs</code> table. In EF8, the following SQL is generated instead:</p>
<pre><code class="language-sql">SELECT b."Id", b."Name"
      FROM "Blogs" AS b
      WHERE b."Id" IN (
          SELECT p."BlogId"
          FROM "Posts" AS p
      )</code></pre>
<p>Since the subquery no longer references <code>Blogs</code>, it can be evaluated once, yielding massive performance improvements on most database systems. However, some database systems, most notably SQL Server, the database is able to optimize the first query to the second query so that the performance is the same.</p>
<p>Check out the <a href="https://www.youtube.com/live/GGDv_p4LAL8?si=6kbx53JjCH0OB_f6">.NET Data Community Standup</a> for discussion and additional examples of <code>IN</code> translations.</p>
<h2>Numeric rowversions for SQL Azure/SQL Server</h2>
<p>SQL Server automatic <a href="https://learn.microsoft.com/ef/core/saving/concurrency">optimistic concurrency</a> is handled using <a href="https://learn.microsoft.com/sql/t-sql/data-types/rowversion-transact-sql"><code>rowversion</code> columns</a>. A <code>rowversion</code> is an 8-byte opaque value passed between database, client, and server. By default, SqlClient exposes <code>rowversion</code> types as <code>byte[]</code>, despite mutable reference types being a bad match for <code>rowversion</code> semantics. In EF8, it is easy instead map <code>rowversion</code> columns to <code>long</code> or <code>ulong</code> properties. For example:</p>
<pre><code class="language-csharp">modelBuilder.Entity&lt;Blog&gt;()
    .Property(e =&gt; e.RowVersion)
    .HasConversion&lt;byte[]&gt;()
    .IsRowVersion();</code></pre>
<h2>Parentheses elimination</h2>
<p>Generating readable SQL is an important goal for EF Core. In EF8, the generated SQL is more readable through automatic elimination of unneeded parenthesis. For example, the following LINQ query:</p>
<pre><code class="language-csharp">await ctx.Customers  
    .Where(c =&gt; c.Id * 3 + 2 &gt; 0 &amp;&amp; c.FirstName != null || c.LastName != null)  
    .ToListAsync();  </code></pre>
<p>Translates to the following Azure SQL when using EF7:</p>
<pre><code class="language-sql">SELECT [c].[Id], [c].[City], [c].[FirstName], [c].[LastName], [c].[Street]
FROM [Customers] AS [c]
WHERE ((([c].[Id] * 3) + 2) &gt; 0 AND ([c].[FirstName] IS NOT NULL)) OR ([c].[LastName] IS NOT NULL)</code></pre>
<p>Which has been improved to the following when using EF8:</p>
<pre><code class="language-sql">SELECT [c].[Id], [c].[City], [c].[FirstName], [c].[LastName], [c].[Street]
FROM [Customers] AS [c]
WHERE ([c].[Id] * 3 + 2 &gt; 0 AND [c].[FirstName] IS NOT NULL) OR [c].[LastName] IS NOT NULL</code></pre>
<h2>Everything in EF8</h2>
<p>Overall, EF8 RC 2 contains all the major feature features we intend to ship in EF8, although further tweaks and bug fixes are coming for GA. These features include:</p>
<ul>
<li><a href="https://github.com/dotnet/efcore/issues/29424">Allow Multi-region or Application Preferred Regions in EF Core Cosmos</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/9906">Use C# structs or classes as value objects</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/31489">Support primitive collections in the compiled model</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/31414">Migrations and model snapshot for primitive collections</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/31365">Query: add support for projecting JSON entities that have been composed on</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/31355">SQLite: Add EF.Functions.Unhex</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30677">Add type mapping APIs to customize JSON value serialization/deserialization</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30408">SQL Server Index options SortInTempDB and DataCompression</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30965">Analyzer: warn (and code fix) for use of interpolation in SQL methods accepting raw strings</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30955">Translate Contains to IN with subquery instead of EXISTS where relevant</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30732">Allow inline primitive collections with parameters, translating to VALUES</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30708">Translate DateOnly.FromDateTime</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30604">Implement JSON serialization/deserialization via Utf8JsonReader/Utf8JsonWriter</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/13613">Update pattern for scaffolding column default constraints</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/31386">Use IN instead of EXISTS with ExecuteDelete and entity containment</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/31406">Allow ExecuteUpdate to update properties of multiple queries as long as the map to a single table</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/31364">Query: add support for projecting primitive collections from JSON entities</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/31100">Switch to storing enums as ints in JSON instead of strings</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30926">Translate DegreesToRadians</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30730">Metadata and type mapping support for primitive collections</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30727">JSON type representations and conversions to store types</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29755">Allow stripping away all model building code to reduce application size</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/28688">Json: add support for collection of primitive types inside JSON columns</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/28616">Support LINQ querying of non-primitive collections within JSON</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/8824">SQLite RevEng: Sample data to determine CLR type</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/701">Allow default value check in value generation to be customized</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/15070">Update handling of non-nullable store-generated properties</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/13617">IN() list queries are not parameterized, causing increased SQL Server CPU usage</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30704">Allow &#8216;unsharing&#8217; connection between contexts</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30684">Remove unneeded subquery and projection when using ordering without limit/offset in set operations</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30610">Make SequentialGuidValueGenerator non-allocating</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30426">Support querying over primitive collections</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30334">JSON/Sqlite: use -&gt; and -&gt;&gt; where possible when traversing JSON, rather than json_extract</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30072">Add Generic version of EntityTypeConfiguration Attribute</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29725">NativeAOT/trimming compatibility for Microsoft.Data.Sqlite</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29427">Map collections of primitive types to JSON column in relational database</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/28925">Translate DateTimeOffset.ToUnixTime(Seconds|Milliseconds)</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/27752">Allow pooling DbContext with singleton services</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/26560">Optional RestartSequenceOperation.StartValue</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/24896">Generate compiled relational model</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/24476">Global query filters produce too many parameters</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/30410">Optimize update path for single property JSON element</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29602">JSON columns can be used in compiled models</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/26767">Unneeded parentheses removed in SQL queries </a></li>
<li><a href="https://github.com/dotnet/efcore/issues/19129">Set operations are supported over non-entity projections with different facets</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/28816">Json: add support for Sqlite provider</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/365">SQL Server: Support hierarchyid</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29916">Configuration to opt out of occasionally problematic SaveChanges optimizations</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/28687">Add convention types for triggers</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/28648">Translate element access of a JSON array</a></li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#sql-queries-for-unmapped-types">Raw SQL queries for unmapped types</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/24507">Support the new BCL DateOnly and TimeOnly structs for SQL Server</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/17066">Translate ElementAt(OrDefault)</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/10787">Opt-out of lazy-loading for specific navigations</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/10042">Lazy-loading for no-tracking queries</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29121">Reverse engineer Synapse and Dynamics 365 TDS</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/10691">Set MaxLength on TPH discriminator property by convention</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/20839">Translate ToString() on a string column</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29476">Generic overload of ConventionSetBuilder.Remove</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29685">Lookup tracked entities by primary key, alternate key, or foreign key</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29758">Allow UseSequence and HiLo on non-key properties</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/29910">Pass query tracking behavior to materialization interceptor</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/27526">Use case-insensitive string key comparisons on SQL Server</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/24771">Allow value converters to change the DbType</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/13540">Resolve application services in EF services</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/12434">Numeric rowersion properties automatically convert to binary</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/24199">Allow transfer of ownership of DbConnection from application to DbContext</a></li>
<li><a href="https://github.com/dotnet/efcore/issues/18715">Provide more information when &#8216;No DbContext was found&#8217; error is generated</a></li>
</ul>
<h2>How to get EF8 RC 2</h2>
<p>EF8 is distributed exclusively as a set of NuGet packages. For example, to add the SQL Server provider to your project, you can use the following command using the dotnet tool:</p>
<pre><code class="language-bash">dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.0-rc.2.23480.1</code></pre>
<h2>Installing the EF8 Command Line Interface (CLI)</h2>
<p>The <code>dotnet-ef</code> tool must be installed before executing EF8 Core migration or scaffolding commands.</p>
<p>To install the tool globally, use:</p>
<pre><code class="language-bash">dotnet tool install --global dotnet-ef --version 8.0.0-rc.2.23480.1</code></pre>
<p>If you already have the tool installed, you can upgrade it with the following command:</p>
<pre><code class="language-bash">dotnet tool update --global dotnet-ef --version 8.0.0-rc.2.23480.1</code></pre>
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
<p>The post <a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-rc2/">EF Core 8 Release Candidate 2: Smaller features in EF8</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
