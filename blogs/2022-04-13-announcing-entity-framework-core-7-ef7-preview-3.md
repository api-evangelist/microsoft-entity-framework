---
title: "Announcing Entity Framework Core 7 (EF7) Preview 3"
url: "https://devblogs.microsoft.com/dotnet/announcing-entity-framework-7-preview-3/"
date: "Wed, 13 Apr 2022 17:16:28 +0000"
author: "Jeremy Likness"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>Today, the .NET data team announces the third preview release of
<a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/7.0.0-preview.3.22175.1">EF Core 7.0 (EF7)</a>.
In addition to bug fixes and foundation work for larger features, we are pleased to announce the initial preview of scaffolding (database-first) templates. This preview also includes changes to the update pipeline to improve performance and streamline the generated SQL, and support for TPC in migrations.
Be sure to read the <a href="https://docs.microsoft.com/ef/core/what-is-new/ef-core-7.0/plan">full plan for EF7</a> to learn what&#8217;s on the roadmap.</p>
<p>You can also view the <a href="https://github.com/dotnet/efcore/issues?q=is%3Aclosed+is%3Aissue+milestone%3A7.0.0-preview3+">full list of issues addressed in EF7 Preview 3</a>.</p>
<h2>Improvements to the update pipeline</h2>
<p>Several improvements to the update pipeline are now part of Preview 3, including:</p>
<ul>
<li><a href="https://github.com/dotnet/efcore/pull/27573">Improve SQL Server insertion logic</a> (also make RETURNING the default INSERT strategy for retrieving db-generated values for other providers).</li>
<li><a href="https://github.com/dotnet/efcore/pull/27663">Use RETURNING/OUTPUT clause for UPDATE/DELETE</a></li>
<li><a href="https://github.com/dotnet/efcore/pull/27584">Refactor ReaderModificationCommandBatch</a></li>
<li><a href="https://github.com/dotnet/efcore/pull/27696">Reimplement MaxBatchSize as a pre-check</a></li>
</ul>
<h2>Take control of your DbContext</h2>
<p>Preview 3 introduces the ability to control how EF7 reverse engineers or scaffolds classes for database-first projects using <a href="https://docs.microsoft.com/visualstudio/modeling/code-generation-and-t4-text-templates">T4 templates</a>.
Do you prefer &#8220;null bang&#8221; setters? Property initializers? Constructor initialization? All these customizations are now possible. In fact, you are not limited to generating
the &#8220;traditional&#8221; DbContext and entity classes. Anything is possible, including using the templates to generate documentation.</p>
<p>The best way to learn about this new feature is to watch our recent community standup: <a href="https://youtu.be/x2nh1vZBsHE">Database-first with T4 templates in EF7</a>. The video
begins with an introduction to T4 templates for those of you who are not familiar with them. The EF7 feature is introduced about 23 minutes in. In addition to generating custom code, the demo shows how to create markdown using Mermaid syntax to generate ERD diagrams.</p>
<p>Code like this:</p>
<pre><code class="language-mermaid">
```mermaid
erDiagram
    ORDERMASTER ||--o{ ORDERDETAIL : owns
    ORDERDETAIL ||--|{ LINE-ITEM : contains
    ORDERMASTER }|..|{ CUSTOMER : uses
```</code></pre>
<p>Produces diagrams like this:</p>
<p><figure class="wp-caption aligncenter" id="attachment_39431"><a href="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2022/04/mermaidexample.png"><img alt="An ERD diagram" class="size-full wp-image-39431" height="527" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2022/04/mermaidexample.png" width="400" /></a><figcaption class="wp-caption-text" id="figcaption_attachment_39431">Mermaid ERD Diagram</figcaption></figure></p>
<p>You can get started in 3 steps:</p>
<ul>
<li>Include the Preview 3 <code>Microsoft.EntityFrameworkCore.Design</code> package in your project (this will also work with the daily builds). </li>
<li>Install or update the <code>dotnet-ef</code> tool either globally or locally using a <a href="https://docs.microsoft.com/dotnet/core/tools/global-tools#install-a-local-tool">tool manifest</a>.</li>
<li>Create the <code>DbContext.t4</code> and <code>EntityType.t4</code> T4 templates in a folder named <code>CodeTemplates</code>. EF7 will pick these up by convention.</li>
</ul>
<p>For more details, watch the <a href="https://youtu.be/x2nh1vZBsHE">community standup demo</a>.</p>
<h2>Prerequisites</h2>
<ul>
<li>EF7 currently targets .NET 6. This will likely be updated to .NET 7 as we near the release. </li>
<li>EF7 will not run on .NET Framework.</li>
</ul>
<p>EF7 is the successor to EF Core 6.0, not to be confused with <a href="https://github.com/dotnet/ef6">EF6</a>. If you are considering upgrading from EF6, please read our guide to <a href="https://docs.microsoft.com/ef/efcore-and-ef6/porting/">port from EF6 to EF Core</a>.</p>
<hr />
<h2>How to get EF7 previews</h2>
<p>EF7 is distributed exclusively as a set of NuGet packages.
For example, to add the SQL Server provider to your project, you can use the following command using the dotnet tool:</p>
<pre><code class="language-bash">dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 7.0.0-preview.3.22175.1</code></pre>
<p>This following table links to the preview 3 versions of the EF Core packages and describes what they are used for.</p>
<table>
<thead>
<tr>
<th style="text-align: right;"><strong>Package</strong></th>
<th style="text-align: left;"><strong>Purpose</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore</a></td>
<td style="text-align: left;">The main EF Core package that is independent of specific database providers</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.SqlServer/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.SqlServer</a></td>
<td style="text-align: left;">Database provider for Microsoft SQL Server and SQL Azure</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite</a></td>
<td style="text-align: left;">SQL Server support for spatial types</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Sqlite/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Sqlite</a></td>
<td style="text-align: left;">Database provider for SQLite that includes the native binary for the database engine</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Sqlite.Core/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Sqlite.Core</a></td>
<td style="text-align: left;">Database provider for SQLite <em>without</em> a packaged native binary</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Sqlite.NetTopologySuite/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Sqlite.NetTopologySuite</a></td>
<td style="text-align: left;">SQLite support for spatial types</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Cosmos/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Cosmos</a></td>
<td style="text-align: left;">Database provider for Azure Cosmos DB</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.InMemory/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.InMemory</a></td>
<td style="text-align: left;">The in-memory database provider</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Tools/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Tools</a></td>
<td style="text-align: left;">EF Core PowerShell commands for the Visual Studio Package Manager Console; use this to integrate tools like <a href="https://docs.microsoft.com/ef/core/managing-schemas/scaffolding">scaffolding</a> and <a href="https://docs.microsoft.com/ef/core/managing-schemas/migrations/">migrations</a> with Visual Studio</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Design/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Design</a></td>
<td style="text-align: left;">Shared design-time components for EF Core tools</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Proxies/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Proxies</a></td>
<td style="text-align: left;">Lazy-loading and change-tracking proxies</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Abstractions/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Abstractions</a></td>
<td style="text-align: left;">Decoupled EF Core abstractions; use this for features like extended data annotations defined by EF Core</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Relational/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Relational</a></td>
<td style="text-align: left;">Shared EF Core components for relational database providers</td>
</tr>
<tr>
<td style="text-align: right;"><a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Analyzers/7.0.0-preview.3.22175.1">Microsoft.EntityFrameworkCore.Analyzers</a></td>
<td style="text-align: left;">C# analyzers for EF Core</td>
</tr>
</tbody>
</table>
<p>We also published the 7.0 preview 3 release of the <a href="https://www.nuget.org/packages/Microsoft.Data.Sqlite.Core/7.0.0-preview.3.22175.1">Microsoft.Data.Sqlite.Core</a> provider for <a href="https://docs.microsoft.com/dotnet/framework/data/adonet/ado-net-overview">ADO.NET</a>.</p>
<h2>Installing the EF7 Command Line Interface (CLI)</h2>
<p>Before you can execute EF7 Core migration or scaffolding commands, you&#8217;ll have to install the CLI package as either a global or local tool.</p>
<p>To install the preview tool globally, install with:</p>
<pre><code class="language-bash">dotnet tool install --global dotnet-ef --version 7.0.0-preview.3.22175.1</code></pre>
<p>If you already have the tool installed, you can upgrade it with the following command:</p>
<pre><code class="language-bash">dotnet tool update --global dotnet-ef --version 7.0.0-preview.3.22175.1</code></pre>
<p>It&#8217;s possible to use this new version of the EF7 CLI with projects that use older versions of the EF Core runtime.</p>
<h2>Daily builds</h2>
<p>EF7 previews are aligned with .NET 7 previews. These previews tend to lag behind the latest work on EF7. Consider using the <a href="https://github.com/aspnet/AspNetCore/blob/master/docs/DailyBuilds.md">daily builds</a> instead to get the most up-to-date EF7 features and bug fixes.</p>
<p>As with the previews, the daily builds require .NET 6.</p>
<h2>The .NET Data Community Standup</h2>
<p>The .NET data team is now live streaming every other Wednesday at 10am Pacific Time, 1pm Eastern Time, or 17:00 UTC. Join the stream to ask questions about the data-related topic of your choice, including the latest preview release. </p>
<ul>
<li><a href="https://aka.ms/efstandups">Watch our YouTube playlist</a> of previous shows</li>
<li><a href="https://dotnet.microsoft.com/platform/community/standup">Visit the .NET Community Standup</a> page to preview upcoming shows</li>
<li><a href="https://github.com/dotnet/efcore/issues/22700">Submit your ideas</a> for a guest, product, demo, or other content to cover</li>
</ul>
<h2>Documentation and Feedback</h2>
<p>The starting point for all EF Core documentation is <a href="https://docs.microsoft.com/ef/">docs.microsoft.com/ef/</a>.</p>
<p>Please file issues found and any other feedback on the <a href="https://github.com/dotnet/efcore">dotnet/efcore GitHub repo</a>.</p>
<h2>Helpful Links</h2>
<p>The following links are provided for easy reference and access.</p>
<p>EF Core Community Standup Playlist:
<a href="https://aka.ms/efstandups" rel="noopener" target="_blank">https://aka.ms/efstandups</a></p>
<p>Main documentation:
<a href="https://aka.ms/efdocs" rel="noopener" target="_blank">https://aka.ms/efdocs</a></p>
<p>Issues and feature requests for EF Core:
<a href="https://aka.ms/efcorefeedback" rel="noopener" target="_blank">https://aka.ms/efcorefeedback</a></p>
<p>Entity Framework Roadmap:
<a href="https://aka.ms/efroadmap" rel="noopener" target="_blank">https://aka.ms/efroadmap</a></p>
<p>Bi-weekly updates:
<a href="https://github.com/dotnet/efcore/issues/27185" rel="noopener" target="_blank">https://github.com/dotnet/efcore/issues/27185</a></p>
<h2>Thank you from the team</h2>
<p>A big thank you from the EF team to everyone who has used and contributed to EF over the years!</p>
<p>Welcome to EF7.</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/announcing-entity-framework-7-preview-3/">Announcing Entity Framework Core 7 (EF7) Preview 3</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
