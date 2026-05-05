---
title: "EF Core 8 Preview 1: Raw, lazy, and on-time"
url: "https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-1/"
date: "Tue, 21 Feb 2023 18:00:00 +0000"
author: "Arthur Vickers"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>The first preview of Entity Framework Core (EF Core) 8 is <a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/8.0.0-preview.1.23111.4">available on NuGet today</a>!</p>
<h2>Basic information</h2>
<p>EF Core 8, or just EF8, is the successor to EF Core 7, and is scheduled for release in November 2023, at the same time as .NET 8.</p>
<p>EF8 currently targets .NET 6. This will likely be updated to .NET 8 as we near release.</p>
<p>EF8 will align with .NET 8 as a long-term support (LTS) release. See the <a href="https://dotnet.microsoft.com/platform/support/policy/dotnet-core">.NET support policy</a> for more information.</p>
<h2>EF8 themes</h2>
<p>Before looking at the new features in preview 1, let&#8217;s take a quick look at the plan for large investments in data access for .NET 8. These investments are covered by five themes:</p>
<h3>Highly requested features</h3>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#json-columns">JSON columns</a>
Build on EF7 JSON support to further power the document/relational hybrid pattern.</li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#value-objects">Value objects</a>
Applications can use DDD-style value objects in EF models.</li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#sql-queries-for-unmapped-types">SQL queries for unmapped types</a>
Applications can execute more types of SQL query without dropping down to ADO.NET or using third-party libraries.</li>
</ul>
<h3>Cloud native and devices</h3>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#aot-and-trimming-with-ef-core">AOT and trimming with EF Core</a>
Small, fast-starting EF Core applications with no dynamic code generation.</li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#aot-and-trimming-for-adonet">AOT and trimming for ADO.NET</a>
Low-level data access can be used in cloud native applications.</li>
</ul>
<h3>Performance</h3>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#woodstar">Woodstar</a>
Fast, fully managed access to SQL Server and Azure SQL for .NET applications.</li>
</ul>
<h3>Visual Tooling</h3>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#first-class-t4-templates-in-visual-studio">First-class T4 templates in Visual Studio</a>
Leverage T4 templating across multiple areas in Visual Studio.</li>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#ef-core-database-first-in-visual-studio">EF Core Database First in Visual Studio</a>
Out-of-the-box Database First tooling in Visual Studio.</li>
</ul>
<h3>Developer experience</h3>
<ul>
<li><a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/plan#theme-developer-experience">Make EF Core better</a>
Improve the developer experience be making many small improvements to EF Core</li>
</ul>
<h3>Find out more and give feedback</h3>
<p>Your feedback on planning is important. Please comment on <a href="https://github.com/dotnet/efcore/issues/29853">GitHub Issue #29853</a> with any feedback or general suggestions about the plan. The best way to indicate the importance of an issue is to vote (<img alt="👍" class="wp-smiley" src="https://s.w.org/images/core/emoji/17.0.2/72x72/1f44d.png" style="height: 1em;" />) for that <a href="https://github.com/dotnet/efcore/issues">issue on GitHub</a>. This data will then feed into the <a href="core/what-is-new/release-planning">planning process</a> for the next release.</p>
<h2>New in EF8 preview 1</h2>
<p>The following sections give an overview of three particular enhancements available in EF8 preview 1: Raw SQL queries for unmapped types, lazy-loading for no-tracking queries, and <code>DateOnly</code>/<code>TimeOnly</code> support for SQL Server/Azure SQL. In total, EF8 preview 1 ships with <a href="https://github.com/dotnet/efcore/issues?q=is%3Aissue+milestone%3A8.0.0-preview1+is%3Aclosed">more than 60 improvements, both bug fixes and enhancements</a>.</p>
<blockquote>
<p><strong>TIP</strong>
Full details of all new EF8 features can be found in the <a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew">What&#8217;s New in EF8</a> documentation. All the code is available in <a href="https://github.com/dotnet/EntityFramework.Docs">runnable samples on GitHub</a>.</p>
</blockquote>
<h3>Raw SQL queries for unmapped types</h3>
<p>EF7 introduced <a href="https://learn.microsoft.com/ef/core/querying/sql-queries#querying-scalar-non-entity-types">raw SQL queries returning scalar types</a>. This is enhanced in EF8 to include raw SQL queries returning any mappable CLR type, without including that type in the EF model.</p>
<blockquote>
<p><strong>TIP</strong>
The code shown here comes from <a href="https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore8/RawSqlSample.cs">RawSqlSample.cs</a>.</p>
</blockquote>
<p>Queries using unmapped types are executed using <a href="https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.relationaldatabasefacadeextensions.sqlquery">SqlQuery</a> or <a href="https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.relationaldatabasefacadeextensions.sqlqueryraw">SqlQueryRaw</a>. The former uses string interpolation to parameterize the query, which helps ensure that all non-constant values are parameterized. For example, consider the following database table:</p>
<pre><code class="language-sql">CREATE TABLE [Posts] (
    [Id] int NOT NULL IDENTITY,
    [Title] nvarchar(max) NOT NULL,
    [Content] nvarchar(max) NOT NULL,
    [PublishedOn] date NOT NULL,
    [BlogId] int NOT NULL,
);</code></pre>
<p><code>SqlQuery</code> can be used to query this table and return instances of a <code>BlogPost</code> type with properties corresponding to the columns in the table:</p>
<p>For example:</p>
<pre><code class="language-csharp">public class BlogPost
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public DateOnly PublishedOn { get; set; }
    public int BlogId { get; set; }
}</code></pre>
<p>For example:</p>
<pre><code class="language-csharp">var start = new DateOnly(2022, 1, 1);
var end = new DateOnly(2023, 1, 1);
var postsIn2022 =
    await context.Database
        .SqlQuery&lt;BlogPost&gt;($"SELECT * FROM Posts as p WHERE p.PublishedOn &gt;= {start} AND p.PublishedOn &lt; {end}")
        .ToListAsync();</code></pre>
<p>When using SQL Server, this query is parameterized and executed as:</p>
<pre><code class="language-sql">SELECT * FROM Posts as p WHERE p.PublishedOn &gt;= @p0 AND p.PublishedOn &lt; @p1</code></pre>
<p>The type used for query results can contain common mapping constructs supported by EF Core, such as parameterized constructors and mapping attributes. For example:</p>
<pre><code class="language-csharp">public class BlogPost
{
    public BlogPost(string blogTitle, string content, DateOnly publishedOn)
    {
        BlogTitle = blogTitle;
        Content = content;
        PublishedOn = publishedOn;
    }

    public int Id { get; private set; }

    [Column("Title")]
    public string BlogTitle { get; set; }

    public string Content { get; set; }
    public DateOnly PublishedOn { get; set; }
    public int BlogId { get; set; }
}</code></pre>
<blockquote>
<p><strong>NOTE</strong>
Types used in this way do not have keys defined and cannot have relationships to other types. Types with relationships must be mapped in the model.</p>
</blockquote>
<p>One nice feature of <code>SqlQuery</code> is that it returns an <code>IQueryable</code> which can be composed on using LINQ. For example, a &#8216;Where&#8217; clause can be added to the query above:</p>
<pre><code class="language-csharp">var summariesIn2022 =
    await context.Database.SqlQuery&lt;PostSummary&gt;(
            @$"SELECT b.Name AS BlogName, p.Title AS PostTitle, p.PublishedOn
               FROM Posts AS p
               INNER JOIN Blogs AS b ON p.BlogId = b.Id")
        .Where(p =&gt; p.PublishedOn &gt;= cutoffDate &amp;&amp; p.PublishedOn &lt; end)
        .ToListAsync();</code></pre>
<p>This is executed as:</p>
<pre><code class="language-sql">SELECT [n].[BlogName], [n].[PostTitle], [n].[PublishedOn]
FROM (
         SELECT b.Name AS BlogName, p.Title AS PostTitle, p.PublishedOn
         FROM Posts AS p
                  INNER JOIN Blogs AS b ON p.BlogId = b.Id
     ) AS [n]
WHERE [n].[PublishedOn] &gt;= @__cutoffDate_1 AND [n].[PublishedOn] &lt; @__end_2</code></pre>
<p>At this point it is worth remembering that all of the above can be done completely in LINQ without the need to write any SQL. This includes returning instances of an unmapped type like <code>PostSummary</code>. For example, the preceding query can be written in LINQ as:</p>
<pre><code class="language-csharp">var summariesByLinq =
    await context.Posts.Select(
            p =&gt; new PostSummary
            {
                BlogName = p.Blog.Name,
                PostTitle = p.Title,
                PublishedOn = p.PublishedOn,
            })
        .Where(p =&gt; p.PublishedOn &gt;= start &amp;&amp; p.PublishedOn &lt; end)
        .ToListAsync();</code></pre>
<p>Which translates to much cleaner SQL:</p>
<pre><code class="language-sql">SELECT [b].[Name] AS [BlogName], [p].[Title] AS [PostTitle], [p].[PublishedOn]
FROM [Posts] AS [p]
INNER JOIN [Blogs] AS [b] ON [p].[BlogId] = [b].[Id]
WHERE [p].[PublishedOn] &gt;= @__start_0 AND [p].[PublishedOn] &lt; @__end_1</code></pre>
<blockquote>
<p><strong>TIP</strong>
EF is able to generate cleaner SQL when it is responsible for the entire query than it is when composing over user-supplied SQL because, in the former case, the full semantics of the query are available to EF.</p>
</blockquote>
<p>So far, all the queries have been executed directly against tables. <code>SqlQuery</code> can also be used to return results from a view without mapping the view type in the EF model. For example:</p>
<pre><code class="language-csharp">var summariesFromView =
    await context.Database.SqlQuery&lt;PostSummary&gt;(
            @$"SELECT * FROM PostAndBlogSummariesView")
        .Where(p =&gt; p.PublishedOn &gt;= cutoffDate &amp;&amp; p.PublishedOn &lt; end)
        .ToListAsync();</code></pre>
<p>Likewise, <code>SqlQuery</code> can be used for the results of a function:</p>
<pre><code class="language-csharp">var summariesFromFunc =
    await context.Database.SqlQuery&lt;PostSummary&gt;(
            @$"SELECT * FROM GetPostsPublishedAfter({cutoffDate})")
        .Where(p =&gt; p.PublishedOn &lt; end)
        .ToListAsync();</code></pre>
<p>The returned <code>IQueryable</code> can be composed upon when it is the result of a view or function, just like it can be for the result of a table query. Stored procedures can be also be executed using <code>SqlQuery</code>, but most databases do not support further composition. For example:</p>
<pre><code class="language-csharp">var summariesFromStoredProc =
    await context.Database.SqlQuery&lt;PostSummary&gt;(
            @$"exec GetRecentPostSummariesProc")
        .ToListAsync();</code></pre>
<h3>Lazy-loading for no-tracking queries</h3>
<p>EF8 adds support for <a href="https://learn.microsoft.com/ef/core/querying/related-data/lazy">lazy-loading of navigations</a> on entities that are not being tracked by the <code>DbContext</code>. This means a no-tracking query can be followed by lazy-loading of navigations on the entities returned by the no-tracking query.</p>
<blockquote>
<p><strong>TIP</strong>
The code for the lazy-loading examples shown below comes from <a href="https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore8/LazyLoadingSample.cs">LazyLoadingSample.cs</a>.</p>
</blockquote>
<p>For example, consider a no-tracking query for blogs:</p>
<pre><code class="language-csharp">var blogs = await context.Blogs.AsNoTracking().ToListAsync();</code></pre>
<p>If <code>Blog.Posts</code> is configured for lazy-loading, for example, using lazy-loading proxies, then accessing <code>Posts</code> will cause it to load from the database:</p>
<pre><code class="language-csharp">Console.WriteLine();
Console.Write("Choose a blog: ");
if (int.TryParse(ReadLine(), out var blogId))
{
    Console.WriteLine("Posts:");
    foreach (var post in blogs[blogId - 1].Posts)
    {
        Console.WriteLine($"  {post.Title}");
    }
}</code></pre>
<p>EF8 also reports whether or not a given navigation is loaded for entities not tracked by the context. For example:</p>
<pre><code class="language-csharp">foreach (var blog in blogs)
{
    if (context.Entry(blog).Collection(e =&gt; e.Posts).IsLoaded)
    {
        Console.WriteLine($" Posts for blog '{blog.Name}' are loaded.");
    }
}</code></pre>
<p>There are a few important considerations when using lazy-loading in this way:</p>
<ul>
<li>Lazy-loading will only succeed until the <code>DbContext</code> used to query the entity is disposed.</li>
<li>Entities queried in this way a reference to their <code>DbContext</code>, even though they are not tracked by it. Care should be taken to avoid memory leaks if the entity instances will have long lifetimes.</li>
<li>Explicitly detaching the entity by setting its state to <code>EntityState.Detached</code> severs the reference to the <code>DbContext</code> and lazy-loading will no longer work.</li>
<li>Remember that all lazy-loading uses synchronous I/O, since there is no way to access a property in an asynchronous manner.</li>
</ul>
<p>Lazy-loading from untracked entities works for both <a href="https://learn.microsoft.com/ef/core/querying/related-data/lazy#lazy-loading-with-proxies">lazy-loading proxies</a> and <a href="https://learn.microsoft.com/ef/core/querying/related-data/lazy#lazy-loading-without-proxies">lazy-loading without proxies</a>.</p>
<h3>DateOnly/TimeOnly supported on SQL Server</h3>
<p>The <a href="https://learn.microsoft.com/dotnet/api/system.dateonly">System.DateOnly</a> and <a href="https://learn.microsoft.com/dotnet/api/system.timeonly">System.TimeOnly</a> types were introduced in .NET 6 and have been supported for several database providers (e.g. SQLite, MySQL, and PostgreSQL) since their introduction. For SQL Server, the recent release of a <a href="https://www.nuget.org/packages/Microsoft.Data.SqlClient/">Microsoft.Data.SqlClient</a> package targeting .NET 6 has allowed <a href="https://github.com/dotnet/SqlClient/pull/1813">ErikEJ to add support for these types at the ADO.NET level</a>. This in turn paved the way for support in EF8 for <code>DateOnly</code> and <code>TimeOnly</code> as properties in entity types.</p>
<blockquote>
<p><strong>TIP</strong>
<code>DateOnly</code> and <code>TimeOnly</code> can be used in EF Core 6 and 7 using the <a href="https://www.nuget.org/packages/ErikEJ.EntityFrameworkCore.SqlServer.DateOnlyTimeOnly">ErikEJ.EntityFrameworkCore.SqlServer.DateOnlyTimeOnly</a> community package from <a href="https://github.com/ErikEJ">@ErikEJ</a>.</p>
</blockquote>
<p>For example, consider the following EF model for British schools:</p>
<pre><code class="language-csharp">public class School
{
    public int Id { get; set; }
    public string Name { get; set; } = null!;
    public DateOnly Founded { get; set; }
    public List&lt;Term&gt; Terms { get; } = new();
    public List&lt;OpeningHours&gt; OpeningHours { get; } = new();
}

public class Term
{
    public int Id { get; set; }
    public string Name { get; set; } = null!;
    public DateOnly FirstDay { get; set; }
    public DateOnly LastDay { get; set; }
    public School School { get; set; } = null!;
}

[Owned]
public class OpeningHours
{
    public OpeningHours(DayOfWeek dayOfWeek, TimeOnly? opensAt, TimeOnly? closesAt)
    {
        DayOfWeek = dayOfWeek;
        OpensAt = opensAt;
        ClosesAt = closesAt;
    }

    public DayOfWeek DayOfWeek { get; private set; }
    public TimeOnly? OpensAt { get; set; }
    public TimeOnly? ClosesAt { get; set; }
}</code></pre>
<blockquote>
<p><strong>TIP</strong>
The code shown here comes from <a href="https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore8/DateOnlyTimeOnlySample.cs">DateOnlyTimeOnlySample.cs</a>.</p>
<p><strong>NOTE</strong>
This model represents only British schools and stores times as local (GMT) times. Handling different timezones would complicate this code significantly. Note that using <code>DateTimeOffset</code> would not help here, since opening and closing times have different offsets depending whether daylight saving time is active or not.</p>
</blockquote>
<p>These entity types map to the following tables when using SQL Server or Azure SQL. Notice that the <code>DateOnly</code> properties map to <code>date</code> columns, and the <code>TimeOnly</code> properties map to <code>time</code> columns.</p>
<pre><code class="language-sql">CREATE TABLE [Schools] (
    [Id] int NOT NULL IDENTITY,
    [Name] nvarchar(max) NOT NULL,
    [Founded] date NOT NULL,
    CONSTRAINT [PK_Schools] PRIMARY KEY ([Id]));

CREATE TABLE [OpeningHours] (
    [SchoolId] int NOT NULL,
    [Id] int NOT NULL IDENTITY,
    [DayOfWeek] int NOT NULL,
    [OpensAt] time NULL,
    [ClosesAt] time NULL,
    CONSTRAINT [PK_OpeningHours] PRIMARY KEY ([SchoolId], [Id]),
    CONSTRAINT [FK_OpeningHours_Schools_SchoolId] FOREIGN KEY ([SchoolId]) REFERENCES [Schools] ([Id]) ON DELETE CASCADE);

CREATE TABLE [Term] (
    [Id] int NOT NULL IDENTITY,
    [Name] nvarchar(max) NOT NULL,
    [FirstDay] date NOT NULL,
    [LastDay] date NOT NULL,
    [SchoolId] int NOT NULL,
    CONSTRAINT [PK_Term] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Term_Schools_SchoolId] FOREIGN KEY ([SchoolId]) REFERENCES [Schools] ([Id]) ON DELETE CASCADE);</code></pre>
<p>Queries using <code>DateOnly</code> and <code>TimeOnly</code> work in the expected manner. For example, the following LINQ query finds schools that are currently open:</p>
<pre><code class="language-csharp">openSchools = await context.Schools
    .Where(
        s =&gt; s.Terms.Any(
                 t =&gt; t.FirstDay &lt;= today
                      &amp;&amp; t.LastDay &gt;= today)
             &amp;&amp; s.OpeningHours.Any(
                 o =&gt; o.DayOfWeek == dayOfWeek
                      &amp;&amp; o.OpensAt &lt; time &amp;&amp; o.ClosesAt &gt;= time))
    .ToListAsync();</code></pre>
<p>This query translates to the following SQL, as shown by calling <a href="https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.entityframeworkqueryableextensions.toquerystring">ToQueryString()</a> on the IQueryable object:</p>
<pre><code class="language-sql">DECLARE @__today_0 date = '2023-02-07';
DECLARE @__dayOfWeek_1 int = 2;
DECLARE @__time_2 time = '19:53:40.4798052';

SELECT [s].[Id], [s].[Founded], [s].[Name], [o0].[SchoolId], [o0].[Id], [o0].[ClosesAt], [o0].[DayOfWeek], [o0].[OpensAt]
FROM [Schools] AS [s]
LEFT JOIN [OpeningHours] AS [o0] ON [s].[Id] = [o0].[SchoolId]
WHERE EXISTS (
    SELECT 1
    FROM [Term] AS [t]
    WHERE [s].[Id] = [t].[SchoolId] AND [t].[FirstDay] &lt;= @__today_0 AND [t].[LastDay] &gt;= @__today_0) AND EXISTS (
    SELECT 1
    FROM [OpeningHours] AS [o]
    WHERE [s].[Id] = [o].[SchoolId] AND [o].[DayOfWeek] = @__dayOfWeek_1 AND [o].[OpensAt] &lt; @__time_2 AND [o].[ClosesAt] &gt;= @__time_2)
ORDER BY [s].[Id], [o0].[SchoolId]</code></pre>
<h2>How to get EF8 preview 1</h2>
<p>EF8 is distributed exclusively as a set of NuGet packages. For example, to add the SQL Server provider to your project, you can use the following command using the dotnet tool:</p>
<pre><code class="language-bash">dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.0-preview.1.23111.4</code></pre>
<h2>Installing the EF8 Command Line Interface (CLI)</h2>
<p>The <code>dotnet-ef</code> tool must be installed before executing EF8 Core migration or scaffolding commands.</p>
<p>To install the tool globally, use:</p>
<pre><code class="language-bash">dotnet tool install --global dotnet-ef --version 8.0.0-preview.1.23111.4</code></pre>
<p>If you already have the tool installed, you can upgrade it with the following command:</p>
<pre><code class="language-bash">dotnet tool update --global dotnet-ef --version 8.0.0-preview.1.23111.4</code></pre>
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
<li>What&#8217;s New in EF Core 8: <a href="https://aka.ms/ef7-new">aka.ms/ef7-new</a></li>
<li>What&#8217;s New in EF Core 7: <a href="https://aka.ms/ef8-new">aka.ms/ef8-new</a></li>
<li>Issues and feature requests for EF Core: <a href="https://github.com/dotnet/efcore/issues">github.com/dotnet/efcore/issues</a></li>
<li>Entity Framework Roadmap: <a href="https://aka.ms/efroadmap">aka.ms/efroadmap</a></li>
<li>Bi-weekly updates: <a href="https://aka.ms/ef-news">aka.ms/ef-news</a></li>
</ul>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-1/">EF Core 8 Preview 1: Raw, lazy, and on-time</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
