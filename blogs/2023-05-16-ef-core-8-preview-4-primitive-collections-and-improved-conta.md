---
title: "EF Core 8 Preview 4: Primitive collections and improved Contains"
url: "https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-4/"
date: "Tue, 16 May 2023 17:04:00 +0000"
author: "Shay Rojansky"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>The fourth preview of Entity Framework Core (EF Core) 8 is <a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/8.0.0-preview.4.23259.3">available on NuGet today</a>!</p>
<h2>Basic information</h2>
<p>EF Core 8, or just EF8, is the successor to EF Core 7, and is scheduled for release in November 2023, at the same time as .NET 8.</p>
<p>EF8 previews currently target .NET 6, and can therefore be used with either .NET 6 (LTS) or .NET 7. This will likely be updated to .NET 8 as we near release.</p>
<p>EF8 will align with .NET 8 as a long-term support (LTS) release. See the <a href="https://dotnet.microsoft.com/platform/support/policy/dotnet-core">.NET support policy</a> for more information.</p>
<h2>New in EF8 Preview 4</h2>
<p>The fourth preview version of EF Core 8.0 preview4 includes some exciting new capabilities in query translation, as well as an important performance optimization. Let&#8217;s dive in!</p>
<h3>Translating LINQ Contains with an inline collection</h3>
<p>In EF&#8217;s quest to translate more and more LINQ queries to SQL, we sometimes encounter odd and problematic corner cases. Let&#8217;s examine such a case, which also happens to be related to a highly-voted EF performance issue. Starting from something simple, imagine you have a bunch of Blogs, and want to query out two Blogs whose names you know. You could use the following LINQ query to do so:</p>
<pre><code class="language-c#">var blogs = await context.Blogs
    .Where(b =&gt; new[] { "Blog1", "Blog2" }.Contains(b.Name))
    .ToArrayAsync();</code></pre>
<p>This would cause the followed SQL query to be generated on SQL Server:</p>
<pre><code class="language-sql">SELECT [b].[Id], [b].[Name]
FROM [Blogs] AS [b]
WHERE [b].[Name] IN (N'Blog1', N'Blog2')</code></pre>
<p>Looks great! The LINQ Contains operator has a matching SQL construct &#8211; the IN expression &#8211; which provides us with a perfect translation. However, the names in this query are embedded as constants into the LINQ query &#8211; and therefore also into the SQL query, via what I&#8217;ll refer to as an <em>inline collection</em> (that&#8217;s the <code>new[] { ... }</code> part): the collection is specified within the query itself, in line. In many cases, we can&#8217;t do that: the Blog names are sometimes available only as a variable, since we read them from some other source, possibly even from another EF LINQ query.</p>
<h3>Translating LINQ Contains with a parameter collection</h3>
<p>So what happens when we try to do the same, but embedding a variable within the query instead of an inline collection?</p>
<pre><code class="language-c#">var names = new[] { "Blog1", "Blog2" };

var blogs = await context.Blogs
    .Where(b =&gt; names.Contains(b.Name))
    .ToArrayAsync();</code></pre>
<p>When a variable such as <code>names</code> is embedded in a query, EF usually sends it as-is via a database parameter. This works well in most cases, but for this particular case, databases simply don&#8217;t support using the IN expression with a parameter. In other words, the following isn&#8217;t valid SQL:</p>
<pre><code class="language-sql">SELECT [b].[Id], [b].[Name]
FROM [Blogs] AS [b]
WHERE [b].[Name] IN @names</code></pre>
<p>More broadly, relational databases don&#8217;t really have the concept of a &#8220;list&#8221; or of a &#8220;collection&#8221;; they generally work with logically unordered, structured sets such as tables. SQL Server does allow sending <a href="https://learn.microsoft.com/sql/relational-databases/tables/use-table-valued-parameters-database-engine">table-valued parameters</a>, but that involves various complications which make this an inappropriate solution (e.g. the table type must be defined in advanced before querying, with its specific structure).</p>
<p>The one exception to this is PostgreSQL, which fully supports the concept of arrays: you can have an int array column in a table, query into it, and send an array as a parameter, just like you can with any other database type. This allows the EF PostgreSQL provider to perform the following translation:</p>
<pre><code class="language-sql">Executed DbCommand (10ms) [Parameters=[@__names_0={ 'Blog1', 'Blog2' } (DbType = Object)], CommandType='Text', CommandTimeout='30']

SELECT b."Id", b."Name"
FROM "Blogs" AS b
WHERE b."Name" = ANY (@__names_0)</code></pre>
<p>This is very similar to the inline collection translation above with IN, but uses the PostgreSQL-specific ANY construct, which can accept an array type. Leveraging this, we pass the array of blog names as a SQL parameter directly to ANY &#8211; that&#8217;s <code>@__names_0</code> &#8211; and get the perfect translation. But what can we do for other databases, where this does not exist?</p>
<p>Up to now, all versions of EF have provided the following translation:</p>
<pre><code class="language-sql">SELECT [b].[Id], [b].[Name]
FROM [Blogs] AS [b]
WHERE [b].[Name] IN (N'Blog1', N'Blog2')</code></pre>
<p>But wait, this looks suspiciously familiar &#8211; it&#8217;s the inline collection translation we saw above! And indeed, since we couldn&#8217;t parameterize the array, we simply embedded its values &#8211; as constants &#8211; into the SQL query. While .NET variables in EF LINQ queries usually become SQL parameters, in this particular case the variable has disappeared, and its contents have been inserted directly into the SQL.</p>
<p>This has the unfortunate consequence that the SQL produced by EF varies for different array contents &#8211; a pretty abnormal situation! Usually, when you run the same LINQ query over and over again &#8211; changing only parameter values &#8211; EF sends the exact same SQL to the database. This is vital for good performance: SQL Server caches SQL, performing expensive query planning only the first time a particular SQL is seen (a similar SQL cache is implemented in the database driver for PostgreSQL). In addition, EF itself has an internal SQL cache for its queries, and this SQL variance makes caching impossible, leading to further EF overhead for each and every query.</p>
<p>But crucially, the negative performance impact of constantly varying SQLs goes beyond this particular query. SQL Server (and Npgsql) can only cache a certain number of SQLs; at some point, they have to get rid of old entries to avoid using too much memory. If you frequently use Contains with a variable array, each individual invocation causees valuable cache entries to be taken at the database, for SQLs that will most probably never be used (since they have the specific array values baked in). That means you&#8217;re also evicting cache entries for other, important SQLs that will need to be used, and requiring them to be re-planned again and again.</p>
<p>In short &#8211; not great! In fact, this performance issue is the <a href="https://github.com/dotnet/efcore/issues/13617">second most highly-voted issue</a> in the EF Core repo; and as with most performance problems, your application may be suffering from it without you knowing about it. We clearly need a better solution for translating the LINQ Contains operator when the collection is a parameter.</p>
<h3>Using OPENJSON to translate parameter collections</h3>
<p>Let&#8217;s see what SQL preview4 generates for this LINQ query:</p>
<pre><code class="language-sql">Executed DbCommand (49ms) [Parameters=[@__names_0='["Blog1","Blog2"]' (Size = 4000)], CommandType='Text', CommandTimeout='30']

SELECT [b].[Id], [b].[Name]
FROM [Blogs] AS [b]
WHERE EXISTS (
    SELECT 1
    FROM OPENJSON(@__names_0) AS [n]
    WHERE [n].[value] = [b].[Name])</code></pre>
<p>This SQL is a completely different beast indeed; but even without understanding exactly what&#8217;s going on, we can already see that the blog names are passed as a parameter, represented via <code>@__names_0</code> in the SQL &#8211; similar to our PostgreSQL translation above. So how does this work?</p>
<p>Modern databases have built-in support for JSON; although the specifics vary from database to database, all support some basic forms of parsing and querying JSON directly in SQL. One of SQL Server&#8217;s JSON capabilities is the OPENJSON function: this is a &#8220;table-valued function&#8221; which accepts a JSON document, and returns a standard, relational rowset from its contents. For example, the following SQL query:</p>
<pre><code class="language-sql">SELECT * FROM OPENJSON('["one", "two", "three"]');</code></pre>
<p>Returns the following rowset:</p>
<table>
<thead>
<tr>
<th>[key]</th>
<th>value</th>
<th>type</th>
</tr>
</thead>
<tbody>
<tr>
<td>0</td>
<td>one</td>
<td>1</td>
</tr>
<tr>
<td>1</td>
<td>two</td>
<td>1</td>
</tr>
<tr>
<td>2</td>
<td>three</td>
<td>2</td>
</tr>
</tbody>
</table>
<p>The input JSON array has effectively been transformed into a relational &#8220;table&#8221;, which can then be queried with the usual SQL operators. EF makes use of this to solve the &#8220;parameter collection&#8221; problem:</p>
<ol>
<li>We convert your .NET array variable into a JSON array&#8230;</li>
<li>We send that JSON array as a simple SQL nvarchar parameter&#8230;</li>
<li>We use the OPENJSON function to unpack the parameter&#8230;</li>
<li>And we use an EXISTS subquery to check if any of the elements match the Blog&#8217;s name.</li>
</ol>
<p>This achieves our goal of having a single, non-varying SQL for different values in the .NET array, and resolves the SQL caching problem. Importantly, when viewed on its own, this new translation may actually run a bit <em>slower</em> than the previous one &#8211; SQL Server can sometimes execute the previous IN translation more efficiently than it can the new translation; when exactly this happens depends on the number of elements in the array. But the crucial bit is that no matter how fast this <em>particular</em> query runs, it no longer causes <em>other</em> queries to be evicted from the SQL cache, negatively affecting your application as a whole.</p>
<blockquote><p>We are looking into further optimizations for the OPENJSON-based translation above &#8211; the preview4 implementation is just the first version of this feature. Stay tuned for further performance improvements in this area.</p></blockquote>
<h3>Older versions of SQL Server</h3>
<p>The OPENJSON function was introduced in SQL Server 2016 (13.x); while that&#8217;s quite an old version, it&#8217;s still supported, and we don&#8217;t want to break its users by relying on it. Therefore, we&#8217;ve introduced a general way for you to tell EF which SQL Server is being targeted &#8211; this will allow us to take advantage of newer features while preserving backwards compatibility for users on older versions. To do this, simply call the new <code>UseCompatibilityLevel</code> method when configuring your context options:</p>
<pre><code class="language-c#">protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    =&gt; optionsBuilder
        .UseSqlServer(@"&lt;CONNECTION STRING&gt;", o =&gt; o.UseCompatibilityLevel(120));</code></pre>
<p>The <code>120</code> argument is the desired <a href="https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql-compatibility-level#compatibility_level--160--150--140--130--120--110--100--90--80-">SQL Server compatibility level</a>; <code>120</code> corresponds to SQL Server 2014 (12.x). When this is done, EF will generate the previous translation, embedding the array&#8217;s contents into an IN expression.</p>
<h3>Queryable primitive collection columns</h3>
<p>I could stop here, and we&#8217;d already have a nice feature, resolving a long-standing performance issue. But let&#8217;s go further! The solution to Contains above supports representing a primitive collection as a JSON array, and then using that collection like any other table in the query. The above translation of Contains is just a very specific case of that &#8211; but we can do much, much more.</p>
<p>Let&#8217;s say that each Blog is also associated to a collection of Tags. In classical relational modeling, we&#8217;d represent this as a many-to-many relationship between a Blogs table and a Tags table, using a BlogTags join table to link the two together; and EF Core supports this mapping very well (<a href="https://learn.microsoft.com/ef/core/modeling/relationships/many-to-many">see docs</a>). But this traditional modeling can be a bit heavy, requiring two additional tables and JOINs, and a .NET type to wrap your simple string Tag. Let&#8217;s try to look at this from a different angle.</p>
<p>Since EF now supports primitive collections, we can simply add a string array property to our Blog type:</p>
<pre><code class="language-c#">public class Blog
{
    public int Id { get; set; }
    // ...
    public string[] Tags { get; set; }
}</code></pre>
<p>This causes EF to generate the following table:</p>
<pre><code class="language-sql">CREATE TABLE [Blogs] (
    [Id] int NOT NULL IDENTITY,
    -- ...
    [Tags] nvarchar(max) NULL,
);</code></pre>
<p>Our new Tags properties is now mapped to a single <code>nvarchar(max)</code> property in the database. You can now add a Blog with some tags:</p>
<pre><code class="language-c#">context.Blogs.Add(new Blog { Name = "Blog1", Tags = new[] { "Tag1", "Tag2" } });
await context.SaveChangesAsync();</code></pre>
<p>&#8230; and EF will automatically encode your Tags .NET array as a JSON array string in the database:</p>
<pre><code class="language-sql">Executed DbCommand (47ms) [Parameters=[@p0='foo' (Nullable = false) (Size = 4000), @p1='["Tag1","Tag2"]' (Size = 4000)], CommandType='Text', CommandTimeout='30']

INSERT INTO [Blogs] ([Name], [Tags])
OUTPUT INSERTED.[Id]
VALUES (@p0, @p1);</code></pre>
<p>Similarly, when reading a Blog from the database, EF will automatically decode the JSON array and populate your .NET array property. That&#8217;s all pretty nifty &#8211; but people have been doing this for quite some time by defining a <a href="https://learn.microsoft.com/ef/core/modeling/value-conversions">value converter</a> on their array properties. In fact, our value converter documentation has an <a href="https://learn.microsoft.com/ef/core/modeling/value-conversions">example</a> showing exactly this. So what&#8217;s the big deal?</p>
<p>Just as we used a SQL EXISTS subquery to translate the LINQ Contains operator, EF now allows you to use arbitrary LINQ operators over such primitive collection columns &#8211; just as if they were regular DbSets; in other words, primitive collections are now fully queryable. For example, to find all Blogs which have a certain Tag, you can now use the following LINQ query:</p>
<pre><code class="language-c#">var blogs = await context.Blogs
    .Where(b =&gt; b.Tags.Contains("Tag1"))
    .ToArrayAsync();</code></pre>
<p>&#8230; which EF translates to the following:</p>
<pre><code class="language-sql">SELECT [b].[Id], [b].[Name], [b].[Tags]
FROM [Blogs] AS [b]
WHERE EXISTS (
    SELECT 1
    FROM OPENJSON([b].[Tags]) AS [t]
    WHERE [t].[value] = N'Tag1')</code></pre>
<p>That&#8217;s the exact same SQL we saw above for a parameter &#8211; but applied to a column! But let&#8217;s do something fancier: what if, instead of querying for all Blogs which have a certain Tag, we want to query for Blogs which have <em>multiple</em> Tags? This can now be done with the following LINQ query:</p>
<pre><code class="language-c#">var tags = new[] { "Tag1", "Tag2" };

var blogs = await context.Blogs
    .Where(b =&gt; b.Tags.Intersect(tags).Count() &gt;= 2)
    .ToArrayAsync();</code></pre>
<p>This leverages more sophisticated LINQ operators: we intersect each Blog&#8217;s Tags with a parameter collection, and query out the Blogs where there are at least two matches. This translates to the following:</p>
<pre><code class="language-sql">Executed DbCommand (48ms) [Parameters=[@__tags_0='["Tag1","Tag2"]' (Size = 4000)], CommandType='Text', CommandTimeout='30']

SELECT [b].[Id], [b].[Name], [b].[Tags]
FROM [Blogs] AS [b]
WHERE (
    SELECT COUNT(*)
    FROM (
        SELECT [t].[value]
        FROM OPENJSON([b].[Tags]) AS [t] -- column collection
        INTERSECT
        SELECT [t1].[value]
        FROM OPENJSON(@__tags_0) AS [t1] -- parameter collection
    ) AS [t0]) &gt;= 2</code></pre>
<p>That&#8217;s quite a mouthful &#8211; but we&#8217;re using the same basic mechanisms: we perform an intersection between the column primitive collection (<code>[b].[Tags]</code>) and the parameter primitive collection (<code>@__tags_0</code>), using OPENJSON to unpack the JSON array strings into rowsets.</p>
<p>Let&#8217;s look at one last example. Since we encode primitive collections as JSON arrays, these collections are <em>naturally ordered</em>. This is an atypical situation within relationl databases &#8211; relational sets are always logically unordered, and an ORDER BY clause must be used in order to get any deterministic ordering.</p>
<p>Now, a list of Tags is typically an unordered bag: we don&#8217;t care which Tag comes first. But let&#8217;s assume, for the sake of this example, that your Blogs&#8217; Tags are ordered, with more &#8220;important&#8221; Tags coming first. In such a situation, it may make sense to query all Blogs with a certain value as their <em>first</em> Tag:</p>
<pre><code class="language-c#">var blogs = await context.Blogs
    .Where(b =&gt; b.Tags[0] == "Tag1")
    .ToArrayAsync();</code></pre>
<p>This currently generates the following SQL:</p>
<pre><code class="language-sql">SELECT [b].[Id], [b].[Name], [b].[Tags]
FROM [Blogs] AS [b]
WHERE (
    SELECT [t].[value]
    FROM OPENJSON([b].[Tags]) AS [t]
    ORDER BY CAST([t].[key] AS int)
    OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY) = N'Tag1'</code></pre>
<p>EF generates an ORDER BY clause to make sure that the JSON array&#8217;s natural ordering is preserved, and then uses limits to get the first element. This over-elaborate SQL has already been improved, and later previews will generate the following tighter SQL instead:</p>
<pre><code class="language-sql">SELECT [b].[Id], [b].[Name], [b].[Tags]
FROM [Blogs] AS [b]
WHERE JSON_VALUE([b].[Tags], '$[0]') = N'Tag1'</code></pre>
<p>To summarize, you can now use the full range of LINQ operators on primitive collections &#8211; whether they&#8217;re a column or a parameter. This opens up exciting translation possibilities for queries which were never translatable before; we&#8217;re looking forward to seeing the kind of queries you&#8217;ll use with this!</p>
<blockquote><p>Before using JSON-based primitive collections, carefully consider indexing and query performance. Most database allow indexing at least some forms of querying into JSON documents; but arbitrary, complex queries such as the intersect above would likely not be able to use an index. In some cases, traditional relational modeling (e.g. many-to-many) may be more appropriate.</p>
<p>We mentioned above that PostgreSQL has native support for arrays, so there&#8217;s no need to resort to JSON array encoding when dealing with primitive collections there. Instead, primitive array collections are (by default) mapped to arrays, and the PostgreSQL <a href="https://www.postgresql.org/docs/current/functions-array.html#id-1.5.8.25.6.2.2.17.1.1.1">unnest</a> function is used to expand the native array to a rowset.</p></blockquote>
<h3>And one last thing: queryable inline collections</h3>
<p>We discussed columns and parameters containing primitive collections, but we left out one last type &#8211; inline collections. You may remember that we started this post with the following LINQ query:</p>
<pre><code class="language-c#">var blogs = await context.Blogs
    .Where(b =&gt; new[] { "Blog1", "Blog2" }.Contains(b.Name))
    .ToArrayAsync();</code></pre>
<p>The <code>new[] { ... }</code> bit in the query represents an <em>inline collection</em>. Up to now, EF supported these only in some very restricted scenarios, such as with the Contains operator. Preview 4 now brings full support for queryable inline collections, allowing you to use the full range of LINQ operators on them as well.</p>
<p>As an example query, let&#8217;s challenge ourselves and do something a bit more complicated. The following query searches for Blogs which have at least one Tag that starts with either <code>a</code> or <code>b</code>:</p>
<pre><code class="language-c#">var blogs = await context.Blogs
    .Where(b =&gt; new[] { "a%", "b%" }
        .Any(pattern =&gt; b.Tags.Any(tag =&gt; EF.Functions.Like(tag, pattern))))
    .ToArrayAsync();</code></pre>
<p>Note that the inline collection of patterns &#8211; <code>new[] { "a%", "b%" }</code> &#8211; is composed over with the Any operator. This now translates to the following SQL:</p>
<pre><code class="language-sql">SELECT [b].[Id], [b].[Name], [b].[Tags]
FROM [Blogs] AS [b]
WHERE EXISTS (
    SELECT 1
    FROM (VALUES (CAST(N'a%' AS nvarchar(max))), (N'b%')) AS [v]([Value]) -- inline collection
    WHERE EXISTS (
        SELECT 1
        FROM OPENJSON([b].[Tags]) AS [t] -- column collection
        WHERE [t].[value] LIKE [v].[Value]))</code></pre>
<p>The interesting bit is the &#8220;inline collection&#8221; line. Unlike with parameter and column collections, we don&#8217;t need to resort to JSON arrays and OPENJSON: SQL already has a universal mechanism for specifying inline tables via the <code>VALUES</code> expression. This completes the picture &#8211; EF now supports querying into any kind of primitive collection, be it a column, a parameter or an inline collection.</p>
<h3>What&#8217;s supported and what&#8217;s not</h3>
<p>The fourth preview brings primitive collection support for SQL Server and SQLite; the PostgreSQL provider will also be updated to support them. However, as indicated above, this is the first wave of work on primitive collections &#8211; expect further improvements in coming versions. Specifically:</p>
<ul>
<li>Primitive collections inside owned JSON entities aren&#8217;t supported yet.</li>
<li>Certain primitive data types aren&#8217;t yet supported on certain providers; this is the case with spatial types, for example.</li>
<li>We may optimize the SQL around OPENJSON to make querying more efficient.</li>
</ul>
<h2>How to get EF8 Preview 4</h2>
<p>EF8 is distributed exclusively as a set of NuGet packages. For example, to add the SQL Server provider to your project, you can use the following command using the dotnet tool:</p>
<pre><code class="language-bash">dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.0-preview.4.23259.3</code></pre>
<h2>Installing the EF8 Command Line Interface (CLI)</h2>
<p>The <code>dotnet-ef</code> tool must be installed before executing EF8 Core migration or scaffolding commands.</p>
<p>To install the tool globally, use:</p>
<pre><code class="language-bash">dotnet tool install --global dotnet-ef --version 8.0.0-preview.4.23259.3</code></pre>
<p>If you already have the tool installed, you can upgrade it with the following command:</p>
<pre><code class="language-bash">dotnet tool update --global dotnet-ef --version 8.0.0-preview.4.23259.3</code></pre>
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
<li>Bi-weekly updates: <a href="https://aka.ms/ef-news">aka.ms/ef-news</a></li>
</ul>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-4/">EF Core 8 Preview 4: Primitive collections and improved Contains</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
