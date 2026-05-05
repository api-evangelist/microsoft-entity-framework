---
title: "EF Core 8 Preview 2: Lite and familiar"
url: "https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-2/"
date: "Tue, 14 Mar 2023 10:02:00 +0000"
author: "Arthur Vickers"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>The second preview of Entity Framework Core (EF Core) 8 is <a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/8.0.0-preview.2.23128.3">available on NuGet today</a>!</p>
<h2>Basic information</h2>
<p>EF Core 8, or just EF8, is the successor to EF Core 7, and is scheduled for release in November 2023, at the same time as .NET 8.</p>
<p>EF8 previews currently target .NET 6, and can therefore be used with either .NET 6 (LTS) or .NET 7. This will likely be updated to .NET 8 as we near release.</p>
<p>EF8 will align with .NET 8 as a long-term support (LTS) release. See the <a href="https://dotnet.microsoft.com/platform/support/policy/dotnet-core">.NET support policy</a> for more information.</p>
<h2>New in EF8 Preview 2</h2>
<p>The following sections give an overview of two exciting enhancements available in EF8 Preview 2: support for JSON columns in SQLite databases and HierarchyId in SQL Server/Azure SQL databases. EF8 Preview 2 also ships several <a href="https://github.com/dotnet/efcore/issues?q=is%3Aissue+milestone%3A8.0.0-preview2+is%3Aclosed">smaller bug fixes and enhancements</a>, as well as <a href="https://github.com/dotnet/efcore/issues?q=is%3Aissue+milestone%3A8.0.0-preview1+is%3Aclosed">more than 60 bug fixes and enhancements from preview 1</a>.</p>
<blockquote><p><strong>TIP</strong>
Full details of all new EF8 features can be found in the <a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-8.0/whatsnew">What&#8217;s New in EF8</a> documentation. All the code is available in <a href="https://github.com/dotnet/EntityFramework.Docs">runnable samples on GitHub</a>.</p></blockquote>
<h3>JSON Columns for SQLite</h3>
<p>EF7 introduced support for mapping to JSON columns when using Azure SQL/SQL Server. EF8 extends this support to SQLite databases. As for the SQL Server support, this includes</p>
<ul>
<li>Mapping of aggregates built from .NET types to JSON documents stored in SQLite columns</li>
<li>Queries into JSON columns, such as filtering and sorting by the elements of the documents</li>
<li>Queries that project elements out of the JSON document into results</li>
<li>Updating and saving changes to JSON documents</li>
</ul>
<p>The existing <a href="https://learn.microsoft.com/ef/core/what-is-new/ef-core-7.0/whatsnew#json-columns">documentation from What&#8217;s New in EF7</a> provides detailed information on JSON mapping, queries, and updates. This documentation now also applies to SQLite.</p>
<blockquote><p><strong>TIP</strong>
The code shown in the EF7 documentation has been updated to also run on SQLite can can be found in <a href="https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore8/JsonColumnsSample.cs">JsonColumnsSample.cs</a>.</p></blockquote>
<h4>Queries into JSON columns</h4>
<p>Queries into JSON columns on SQLite use the <code>json_extract</code> function. For example, the &#8220;authors in Chigley&#8221; query from the documentation referenced above:</p>
<pre><code class="language-csharp">var authorsInChigley = await context.Authors
    .Where(author =&gt; author.Contact.Address.City == "Chigley")
    .ToListAsync();</code></pre>
<p>Is translated to the following SQL when using SQLite:</p>
<pre><code class="language-sql">SELECT "a"."Id", "a"."Name", "a"."Contact"
FROM "Authors" AS "a"
WHERE json_extract("a"."Contact", '$.Address.City') = 'Chigley'</code></pre>
<h4>Updating JSON columns</h4>
<p>For updates, EF uses the <code>json_set</code> function on SQLite. For example, when updating a single property in a document:</p>
<pre><code class="language-csharp">var arthur = await context.Authors.SingleAsync(author =&gt; author.Name.StartsWith("Arthur"));

arthur.Contact.Address.Country = "United Kingdom";

await context.SaveChangesAsync();</code></pre>
<p>EF generates the following parameters:</p>
<pre><code class="language-text">info: 3/10/2023 10:51:33.127 RelationalEventId.CommandExecuted[20101] (Microsoft.EntityFrameworkCore.Database.Command)
      Executed DbCommand (0ms) [Parameters=[@p0='["United Kingdom"]' (Nullable = false) (Size = 18), @p1='4'], CommandType='Text', CommandTimeout='30']</code></pre>
<p>Which use the <code>json_set</code> function on SQLite:</p>
<pre><code class="language-sql">UPDATE "Authors" SET "Contact" = json_set("Contact", '$.Address.Country', json_extract(@p0, '$[0]'))
WHERE "Id" = @p1
RETURNING 1;</code></pre>
<h3>SQL Server HierarchyId</h3>
<p>Azure SQL and SQL Server have a special data type called <a href="https://learn.microsoft.com/sql/t-sql/data-types/hierarchyid-data-type-method-reference"><code>hierarchyid</code></a> that is used to store <a href="https://learn.microsoft.com/sql/relational-databases/hierarchical-data-sql-server">hierarchical data</a>. In this case, &#8220;hierarchical data&#8221; essentially means data that forms a tree structure, where each item can have a parent and/or children. Examples of such data are:</p>
<ul>
<li>An organizational structure</li>
<li>A file system</li>
<li>A set of tasks in a project</li>
<li>A taxonomy of language terms</li>
<li>A graph of links between Web pages</li>
</ul>
<p>The database is then able to run queries against this data using its hierarchical structure. For example, a query can find ancestors and dependents of given items, or find all items at a certain depth in the hierarchy.</p>
<h4>HierarchyId support in .NET and EF Core</h4>
<p>Official support for the SQL Server <code>hierarchyid</code> type has only recently come to modern .NET platforms (i.e. &#8220;.NET Core&#8221;). This support is in the form of the <a href="https://www.nuget.org/packages/Microsoft.SqlServer.Types">Microsoft.SqlServer.Types</a> NuGet package, which brings in low-level SQL Server-specific types. In this case, the low-level type is called <code>SqlHierarchyId</code>.</p>
<p>At the next level, a new <a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.SqlServer.Abstractions">Microsoft.EntityFrameworkCore.SqlServer.Abstractions</a> package has been introduced, which includes a higher-level <code>HierarchyId</code> type intended for use in entity types.</p>
<blockquote><p><strong>TIP</strong>
The <code>HierarchyId</code> type is more idiomatic to the norms of .NET than <code>SqlHierarchyId</code>, which is instead modeled after how .NET Framework types are hosted inside the SQL Server database engine. <code>HierarchyId</code> is designed to work with EF Core, but it can also be used outside of EF Core in other applications. The <code>Microsoft.EntityFrameworkCore.SqlServer.Abstractions</code> package doesn&#8217;t reference any other packages, and so has minimal impact on deployed application size and dependencies.</p></blockquote>
<p>Use of <code>HierarchyId</code> for EF Core functionality such as queries and updates requires the <a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.SqlServer.HierarchyId">Microsoft.EntityFrameworkCore.SqlServer.HierarchyId</a> package. This package brings in <code>Microsoft.EntityFrameworkCore.SqlServer.Abstractions</code> and <code>Microsoft.SqlServer.Types</code> as transitive dependencies, and so is often the only package needed. Once the package is installed, use of <code>HierarchyId</code> is enabled by calling <code>UseHierarchyId</code> as part of the application&#8217;s call to <code>UseSqlServer</code>. For example:</p>
<pre><code class="language-csharp">options.UseSqlServer(
    connectionString,
    x =&gt; x.UseHierarchyId());</code></pre>
<blockquote><p><strong>NOTE</strong>
Unofficial support for <code>hierarchyid</code> in EF Core has been available for many years via the <a href="https://www.nuget.org/packages/EntityFrameworkCore.SqlServer.HierarchyId">EntityFrameworkCore.SqlServer.HierarchyId</a> package. This package has been maintained as a collaboration between the community and the EF team. Now that there is official support for <code>hierarchyid</code> in .NET, the code from this community package forms, with the permission of the original contributors, the basis for the official package described here. Many thanks to all those involved over the years, including <a href="https://github.com/aljones">@aljones</a>, <a href="https://github.com/cutig3r">@cutig3r</a>, <a href="https://github.com/huan086">@huan086</a>, <a href="https://github.com/kmataru">@kmataru</a>, <a href="https://github.com/mehdihaghshenas">@mehdihaghshenas</a>, and <a href="https://github.com/vyrotek">@vyrotek</a></p></blockquote>
<h4>Modeling hierarchies</h4>
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
<blockquote><p><strong>TIP</strong>
The code shown here and in the examples below comes from <a href="https://github.com/dotnet/EntityFramework.Docs/tree/main/samples/core/Miscellaneous/NewInEFCore8/HierarchyIdSample.cs">HierarchyIdSample.cs</a>.</p>
<p><strong>TIP</strong>
If desired, <code>HierarchyId</code> is suitable for use as a key property type.</p></blockquote>
<p>In this case, the family tree is rooted with the patriarch of the family. Each halfling can be traced from the patriarch down the tree using its <code>PathFromPatriarch</code> property. SQL Server uses a compact binary format for these paths, but it is common to parse to and from a human-readable string representation when when working with code. In this representation, the position at each level is separated by a <code>/</code> character. For example, consider the family tree in the diagram below:</p>
<p><img alt="Halfling family tree" src="./familytree.png" /></p>
<p>In this tree:</p>
<ul>
<li>Balbo is at the root of the tree, represented by <code>/</code>.</li>
<li>Balbo has five children, represented by <code>/1/</code>, <code>/2/</code>, <code>/3/</code>, <code>/4/</code>, and <code>/5/</code>.</li>
<li>Balbo&#8217;s first child, Mungo, also has five children, represented by <code>/1/1/</code>, <code>/1/2/</code>, <code>/1/3/</code>, <code>/1/4/</code>, and <code>/1/5/</code>. Notice that the <code>HierarchyId</code> for Balbo (<code>/1/</code>) is the prefix for all his children.</li>
<li>Similarly, Balbo&#8217;s third child, Ponto, has two children, represented by <code>/3/1/</code> and <code>/3/2/</code>. Again the each of these children is prefixed by the <code>HierarchyId</code> for Ponto, which is represented as <code>/3/</code>.</li>
<li>And so on down the tree&#8230;</li>
</ul>
<p>The following code inserts this family tree into a database using EF Core:</p>
<pre><code class="language-csharp">await AddRangeAsync(
    new Halfling(HierarchyId.Parse("/"), "Balbo", 1167),
    new Halfling(HierarchyId.Parse("/1/"), "Mungo", 1207),
    new Halfling(HierarchyId.Parse("/2/"), "Pansy", 1212),
    new Halfling(HierarchyId.Parse("/3/"), "Ponto", 1216),
    new Halfling(HierarchyId.Parse("/4/"), "Largo", 1220),
    new Halfling(HierarchyId.Parse("/5/"), "Lily", 1222),
    new Halfling(HierarchyId.Parse("/1/1/"), "Bungo", 1246),
    new Halfling(HierarchyId.Parse("/1/2/"), "Belba", 1256),
    new Halfling(HierarchyId.Parse("/1/3/"), "Longo", 1260),
    new Halfling(HierarchyId.Parse("/1/4/"), "Linda", 1262),
    new Halfling(HierarchyId.Parse("/1/5/"), "Bingo", 1264),
    new Halfling(HierarchyId.Parse("/3/1/"), "Rosa", 1256),
    new Halfling(HierarchyId.Parse("/3/2/"), "Polo"),
    new Halfling(HierarchyId.Parse("/4/1/"), "Fosco", 1264),
    new Halfling(HierarchyId.Parse("/1/1/1/"), "Bilbo", 1290),
    new Halfling(HierarchyId.Parse("/1/3/1/"), "Otho", 1310),
    new Halfling(HierarchyId.Parse("/1/5/1/"), "Falco", 1303),
    new Halfling(HierarchyId.Parse("/3/2/1/"), "Posco", 1302),
    new Halfling(HierarchyId.Parse("/3/2/2/"), "Prisca", 1306),
    new Halfling(HierarchyId.Parse("/4/1/1/"), "Dora", 1302),
    new Halfling(HierarchyId.Parse("/4/1/2/"), "Drogo", 1308),
    new Halfling(HierarchyId.Parse("/4/1/3/"), "Dudo", 1311),
    new Halfling(HierarchyId.Parse("/1/3/1/1/"), "Lotho", 1310),
    new Halfling(HierarchyId.Parse("/1/5/1/1/"), "Poppy", 1344),
    new Halfling(HierarchyId.Parse("/3/2/1/1/"), "Ponto", 1346),
    new Halfling(HierarchyId.Parse("/3/2/1/2/"), "Porto", 1348),
    new Halfling(HierarchyId.Parse("/3/2/1/3/"), "Peony", 1350),
    new Halfling(HierarchyId.Parse("/4/1/2/1/"), "Frodo", 1368),
    new Halfling(HierarchyId.Parse("/4/1/3/1/"), "Daisy", 1350),
    new Halfling(HierarchyId.Parse("/3/2/1/1/1/"), "Angelica", 1381));

await SaveChangesAsync();</code></pre>
<blockquote><p><strong>TIP</strong>
If needed, decimal values can be used to create new nodes between two existing nodes. For example, <code>/3/2.5/2/</code> goes between <code>/3/2/2/</code> and <code>/3/3/2/</code>.</p></blockquote>
<h4>The <code>HierarchyId</code> type</h4>
<p><code>HierarchyId</code> exposes several methods that can be used in LINQ queries.</p>
<table>
<thead>
<tr>
<th>Method</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>GetAncestor(int n)</code></td>
<td>Gets the node <code>n</code> levels up the hierarchical tree.</td>
</tr>
<tr>
<td><code>GetDescendant(HierarchyId? child1, HierarchyId? child2)</code></td>
<td>Gets the value of a descendant node that is greater than <code>child1</code> and less than <code>child2</code>.</td>
</tr>
<tr>
<td><code>GetLevel()</code></td>
<td>Gets the level of this node in the hierarchical tree.</td>
</tr>
<tr>
<td><code>GetReparentedValue(HierarchyId? oldRoot, HierarchyId? newRoot)</code></td>
<td>Gets a value representing the location of a new node that has a path from <code>newRoot</code> equal to the path from <code>oldRoot</code> to this, effectively moving this to the new location.</td>
</tr>
<tr>
<td><code>IsDescendantOf(HierarchyId? parent)</code></td>
<td>Gets a value indicating whether this node is a descendant of <code>parent</code>.</td>
</tr>
</tbody>
</table>
<p>In addition, the operators <code>==</code>, <code>!=</code>, <code>&lt;</code>, <code>&lt;=</code>, <code>&gt;</code> and <code>&gt;=</code> can be used.</p>
<h4>Example: Get entities at a given level in the tree</h4>
<p>The following query uses <code>GetLevel</code> to return all halflings at a given level in the family tree:</p>
<pre><code class="language-csharp">var generation = await context.Halflings.Where(halfling =&gt; halfling.PathFromPatriarch.GetLevel() == level).ToListAsync();</code></pre>
<p>This translates to the following SQL:</p>
<pre><code class="language-sql">SELECT [h].[Id], [h].[Name], [h].[PathFromPatriarch], [h].[YearOfBirth]
FROM [Halflings] AS [h]
WHERE [h].[PathFromPatriarch].GetLevel() = @__level_0</code></pre>
<p>Running this in a loop we can get the halflings for every generation:</p>
<pre><code class="language-text">Generation 0: Balbo
Generation 1: Mungo, Pansy, Ponto, Largo, Lily
Generation 2: Bungo, Belba, Longo, Linda, Bingo, Rosa, Polo, Fosco
Generation 3: Bilbo, Otho, Falco, Posco, Prisca, Dora, Drogo, Dudo
Generation 4: Lotho, Poppy, Ponto, Porto, Peony, Frodo, Daisy
Generation 5: Angelica</code></pre>
<h4>Example: Get the direct ancestor of an entity</h4>
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
<h4>Example: Get the direct descendents of an entity</h4>
<p>The following query also uses <code>GetAncestor</code>, but this time to find the direct descendents of a halfling, given that halfling&#8217;s name:</p>
<pre><code class="language-csharp">IQueryable&lt;Halfling&gt; FindDirectDescendents(string name)
    =&gt; context.Halflings.Where(
        descendent =&gt; descendent.PathFromPatriarch.GetAncestor(1) == context.Halflings
            .Single(ancestor =&gt; ancestor.Name == name).PathFromPatriarch);</code></pre>
<p>This translates to the following SQL:</p>
<pre><code class="language-sql">SELECT [h].[Id], [h].[Name], [h].[PathFromPatriarch], [h].[YearOfBirth]
FROM [Halflings] AS [h]
WHERE [h].[PathFromPatriarch].GetAncestor(1) = (
    SELECT TOP(1) [h0].[PathFromPatriarch]
    FROM [Halflings] AS [h0]
    WHERE [h0].[Name] = @__name_0)</code></pre>
<p>Running this query for the halfling &#8220;Mungo&#8221; returns &#8220;Bungo&#8221;, &#8220;Belba&#8221;, &#8220;Longo&#8221;, and &#8220;Linda&#8221;.</p>
<h4>Example: Get all ancestors of an entity</h4>
<p><code>GetAncestor</code> is useful for searching up or down a single level, or, indeed, a specified number of levels. On the other hand, <code>IsDescendantOf</code> is useful for finding all ancestors or dependents. For example, the following query uses <code>IsDescendantOf</code> to find the all the ancestors of a halfling, given that halfling&#8217;s name:</p>
<pre><code class="language-csharp">IQueryable&lt;Halfling&gt; FindAllAncestors(string name)
    =&gt; context.Halflings.Where(
            ancestor =&gt; context.Halflings
                .Single(
                    descendent =&gt;
                        descendent.Name == name
                        &amp;&amp; ancestor.Id != descendent.Id)
                .PathFromPatriarch.IsDescendantOf(ancestor.PathFromPatriarch))
        .OrderByDescending(ancestor =&gt; ancestor.PathFromPatriarch.GetLevel());</code></pre>
<blockquote><p><strong>IMPORTANT</strong>
<code>IsDescendantOf</code> returns true for itself, which is why it is filtered out in the query above.</p></blockquote>
<p>This translates to the following SQL:</p>
<pre><code class="language-sql">SELECT [h].[Id], [h].[Name], [h].[PathFromPatriarch], [h].[YearOfBirth]
FROM [Halflings] AS [h]
WHERE (
    SELECT TOP(1) [h0].[PathFromPatriarch]
    FROM [Halflings] AS [h0]
    WHERE [h0].[Name] = @__name_0 AND [h].[Id] &lt;&gt; [h0].[Id]).IsDescendantOf([h].[PathFromPatriarch]) = CAST(1 AS bit)
ORDER BY [h].[PathFromPatriarch].GetLevel() DESC</code></pre>
<p>Running this query for the halfling &#8220;Bilbo&#8221; returns &#8220;Bungo&#8221;, &#8220;Mungo&#8221;, and &#8220;Balbo&#8221;.</p>
<h4>Example: Get all descendents of an entity</h4>
<p>The following query also uses <code>IsDescendantOf</code>, but this time to all the descendents of a halfling, given that halfling&#8217;s name:</p>
<pre><code class="language-csharp">IQueryable&lt;Halfling&gt; FindAllDescendents(string name)
    =&gt; context.Halflings.Where(
            descendent =&gt; descendent.PathFromPatriarch.IsDescendantOf(
                context.Halflings
                    .Single(
                        ancestor =&gt;
                            ancestor.Name == name
                            &amp;&amp; descendent.Id != ancestor.Id)
                    .PathFromPatriarch))
        .OrderBy(descendent =&gt; descendent.PathFromPatriarch.GetLevel());</code></pre>
<p>This translates to the following SQL:</p>
<pre><code class="language-sql">SELECT [h].[Id], [h].[Name], [h].[PathFromPatriarch], [h].[YearOfBirth]
FROM [Halflings] AS [h]
WHERE [h].[PathFromPatriarch].IsDescendantOf((
    SELECT TOP(1) [h0].[PathFromPatriarch]
    FROM [Halflings] AS [h0]
    WHERE [h0].[Name] = @__name_0 AND [h].[Id] &lt;&gt; [h0].[Id])) = CAST(1 AS bit)
ORDER BY [h].[PathFromPatriarch].GetLevel()</code></pre>
<p>Running this query for the halfling &#8220;Mungo&#8221; returns &#8220;Bungo&#8221;, &#8220;Belba&#8221;, &#8220;Longo&#8221;, &#8220;Linda&#8221;, &#8220;Bingo&#8221;, &#8220;Bilbo&#8221;, &#8220;Otho&#8221;, &#8220;Falco&#8221;, &#8220;Lotho&#8221;, and &#8220;Poppy&#8221;.</p>
<h4>Example: Finding a common ancestor</h4>
<p>One of the most common questions asked about this particular family tree is, &#8220;who is the common ancestor of Frodo and Bilbo?&#8221; We can use <code>IsDescendantOf</code> to write such a query:</p>
<pre><code class="language-csharp">async Task&lt;Halfling?&gt; FindCommonAncestor(Halfling first, Halfling second)
    =&gt; await context.Halflings
        .Where(
            ancestor =&gt; first.PathFromPatriarch.IsDescendantOf(ancestor.PathFromPatriarch)
                        &amp;&amp; second.PathFromPatriarch.IsDescendantOf(ancestor.PathFromPatriarch))
        .OrderByDescending(ancestor =&gt; ancestor.PathFromPatriarch.GetLevel())
        .FirstOrDefaultAsync();</code></pre>
<p>This translates to the following SQL:</p>
<pre><code class="language-sql">SELECT TOP(1) [h].[Id], [h].[Name], [h].[PathFromPatriarch], [h].[YearOfBirth]
FROM [Halflings] AS [h]
WHERE @__first_PathFromPatriarch_0.IsDescendantOf([h].[PathFromPatriarch]) = CAST(1 AS bit)
  AND @__second_PathFromPatriarch_1.IsDescendantOf([h].[PathFromPatriarch]) = CAST(1 AS bit)
ORDER BY [h].[PathFromPatriarch].GetLevel() DESC</code></pre>
<p>Running this query with &#8220;Bilbo&#8221; and &#8220;Frodo&#8221; tells us that their common ancestor is &#8220;Balbo&#8221;.</p>
<h4>Example: Re-parenting a sub-hierarchy</h4>
<p>I&#8217;m sure we all remember the scandal of SR 1752 (a.k.a. &#8220;LongoGate&#8221;) when DNA testing revealed that Longo was not in fact the son of Mungo, but actually the son of Ponto! One fallout from this scandal was that the family tree needed to be re-written. In particular, Longo and all his descendents needed to be re-parented from Mungo to Ponto. <code>GetReparentedValue</code> can be used to do this. For example, first &#8220;Longo&#8221; and all his descendents are queried:</p>
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
<p>This results in the following database update:</p>
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
<blockquote><p><strong>NOTE</strong>
The parameters values for <code>HierarchyId</code> properties are sent to the database in their compact, binary format.</p></blockquote>
<p>Following the update, querying for the descendents of &#8220;Mungo&#8221; returns &#8220;Bungo&#8221;, &#8220;Belba&#8221;, &#8220;Linda&#8221;, &#8220;Bingo&#8221;, &#8220;Bilbo&#8221;, &#8220;Falco&#8221;, and &#8220;Poppy&#8221;, while querying for the descendents of &#8220;Ponto&#8221; returns &#8220;Longo&#8221;, &#8220;Rosa&#8221;, &#8220;Polo&#8221;, &#8220;Otho&#8221;, &#8220;Posco&#8221;, &#8220;Prisca&#8221;, &#8220;Lotho&#8221;, &#8220;Ponto&#8221;, &#8220;Porto&#8221;, &#8220;Peony&#8221;, and &#8220;Angelica&#8221;.</p>
<h2>How to get EF8 Preview 2</h2>
<p>EF8 is distributed exclusively as a set of NuGet packages. For example, to add the SQL Server provider to your project, you can use the following command using the dotnet tool:</p>
<pre><code class="language-bash">dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.0-preview.2.23128.3</code></pre>
<h2>Installing the EF8 Command Line Interface (CLI)</h2>
<p>The <code>dotnet-ef</code> tool must be installed before executing EF8 Core migration or scaffolding commands.</p>
<p>To install the tool globally, use:</p>
<pre><code class="language-bash">dotnet tool install --global dotnet-ef --version 8.0.0-preview.2.23128.3</code></pre>
<p>If you already have the tool installed, you can upgrade it with the following command:</p>
<pre><code class="language-bash">dotnet tool update --global dotnet-ef --version 8.0.0-preview.2.23128.3</code></pre>
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
<p>The post <a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-preview-2/">EF Core 8 Preview 2: Lite and familiar</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
