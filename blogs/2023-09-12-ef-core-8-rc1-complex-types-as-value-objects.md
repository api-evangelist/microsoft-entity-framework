---
title: "EF Core 8 RC1: Complex types as value objects"
url: "https://devblogs.microsoft.com/dotnet/announcing-ef8-rc1/"
date: "Tue, 12 Sep 2023 17:10:00 +0000"
author: "Arthur Vickers"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>The first release candidate of Entity Framework Core (EF Core) 8 is <a href="https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/8.0.0-rc.1.23419.6">available on NuGet today</a>!</p>
<h2>Basic information</h2>
<p>EF Core 8, or just EF8, is the successor to EF Core 7, and is scheduled for release in November 2023, at the same time as .NET 8.</p>
<p>EF8 requires .NET 8 and this RC1 release should be used with the <a href="https://dotnet.microsoft.com/next">.NET 8 RC1 SDK</a>.</p>
<p>EF8 will align with .NET 8 as a long-term support (LTS) release. See the <a href="https://dotnet.microsoft.com/platform/support/policy/dotnet-core">.NET support policy</a> for more information.</p>
<h2>New in EF8</h2>
<p>EF8 RC1 contains all the major feature features we intend to ship in EF8, although further tweaks and bug fixes are coming for RC2. These features include:</p>
<ul>
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
<p>In this post we&#8217;re going to take a more detailed look at <a href="https://github.com/dotnet/efcore/issues/9906">using complex types to represent value objects</a>.</p>
<h2>Complex types as value objects</h2>
<p>Objects saved to the database can be split into three broad categories:</p>
<ul>
<li>Objects that are unstructured and hold a single value. For example, <code>int</code>, <code>Guid</code>, <code>string</code>, <code>IPAddress</code>. These are (somewhat loosely) called &#8220;primitive types&#8221;.</li>
<li>Objects that are structured to hold multiple values, and where the identity of the object is defined by a key value. For example, <code>Blog</code>, <code>Post</code>, <code>Customer</code>. These are called &#8220;entity types&#8221;.</li>
<li>Objects that are structured to hold multiple values, but the object has no key defining identity. For example, <code>Address</code>, <code>Coordinate</code>.</li>
</ul>
<p>Prior to EF8, there was no good way to map the third type of object. <a href="https://learn.microsoft.com/ef/core/modeling/owned-entities">Owned types</a> can be used, but since owned types are actually entity types, they have a key and semantics based on the value of that key, even when the key value is hidden.</p>
<p>EF8 now supports &#8220;Complex Types&#8221; to cover this third type of object. Complex types in EF Core are very similar to complex types in EF6, but there are some differencee. Complex type objects:</p>
<ul>
<li>Are not identified or tracked by key value.</li>
<li>Must be defined as part of an entity type. (In other words, you cannot have a <code>DbSet</code> of a complex type.)</li>
<li>Can be either .NET <a href="https://review.learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-types">value types</a> or <a href="https://review.learn.microsoft.com/dotnet/csharp/language-reference/keywords/reference-types">reference types</a>. (EF6 only supports reference types.)</li>
<li>Can share the same instance across multiple properties. (See below for more details. EF6 does not allow sharing.)</li>
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
<p>So far you might be saying, &#8220;but I could do this with owned types!&#8221; However, the &#8220;entity type&#8221; semantics of owned types quickly get in th way. For example, running the code above with owned types results in a slew of warnings and then an error:</p>
<pre><code class="language-text">warn: 8/20/2023 12:48:01.678 CoreEventId.DuplicateDependentEntityTypeInstanceWarning[10001] (Microsoft.EntityFrameworkCore.Update) 
      The same entity is being tracked as different entity types 'Order.BillingAddress#Address' and 'Customer.Address#Address' with defining navigations. If a property value changes, it will result in two store changes, which might not be the desired outcome.
warn: 8/20/2023 12:48:01.687 CoreEventId.DuplicateDependentEntityTypeInstanceWarning[10001] (Microsoft.EntityFrameworkCore.Update) 
      The same entity is being tracked as different entity types 'Order.ShippingAddress#Address' and 'Customer.Address#Address' with defining navigations. If a property value changes, it will result in two store changes, which might not be the desired outcome.
warn: 8/20/2023 12:48:01.687 CoreEventId.DuplicateDependentEntityTypeInstanceWarning[10001] (Microsoft.EntityFrameworkCore.Update)
      The same entity is being tracked as different entity types 'Order.ShippingAddress#Address' and 'Order.BillingAddress#Address' with defining navigations. If a property value changes, it will result in two store changes, which might not be the desired outcome.
fail: 8/20/2023 12:48:01.709 CoreEventId.SaveChangesFailed[10000] (Microsoft.EntityFrameworkCore.Update) 
      An exception occurred in the database while saving changes for context type 'NewInEfCore8.ComplexTypesSample+CustomerContext'.
      System.InvalidOperationException: Cannot save instance of 'Order.ShippingAddress#Address' because it is an owned entity without any reference to its owner. Owned entities can only be saved as part of an aggregate also including the owner entity.
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.InternalEntityEntry.PrepareToSave()
         ...</code></pre>
<p>This is because a single instance of the <code>Address</code> entity type (with the same hidden key value) is being used for three <em>different</em> entity instances. On the other hand, sharing the same instance between complex properties is allowed, and so the code works as expected when using complex types.</p>
<h3>Configuration of complex types</h3>
<p>Complex types must be configured in the model using either <a href="https://learn.microsoft.com/ef/core/modeling/#use-data-annotations-to-configure-a-model">mapping attributes</a> or by calling <a href="https://learn.microsoft.com/ef/core/modeling/#use-fluent-api-to-configure-a-model"><code>ComplexProperty</code> API in <code>OnModelCreating</code></a>. Complex types are not discovered by convention.</p>
<p>For example, the <code>Address</code> type can be configured using the <a href="https://review.learn.microsoft.com/dotnet/api/system.componentmodel.dataannotations.schema.complextypeattribute"><code>ComplexTypeAttribute</code></a>:</p>
<pre><code class="language-csharp">[ComplexType]
public class Address
{
    public required string Line1 { get; set; }
    public string? Line2 { get; set; }
    public required string City { get; set; }
    public required string Country { get; set; }
    public required string PostCode { get; set; }
}</code></pre>
<p>Or in <code>OnModelCreating</code>:</p>
<pre><code class="language-csharp">protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity&lt;Customer&gt;()
        .ComplexProperty(e =&gt; e.Address);

    modelBuilder.Entity&lt;Order&gt;(b =&gt;
    {
        b.ComplexProperty(e =&gt; e.BillingAddress);
        b.ComplexProperty(e =&gt; e.ShippingAddress);
    });
}</code></pre>
<h3>Mutability</h3>
<p>In the example above, we ended up with the same <code>Address</code> instance used in three places. This is allowed and doesn&#8217;t cause any issues for EF Core when using complex types. However, sharing instances of the same reference type means that if a property value on the instance is modified, then that change will be reflected in all three usages. For example, following on from above, let&#8217;s change <code>Line1</code> of the customer address and save the changes:</p>
<pre><code class="language-csharp">customer.Address.Line1 = "Peacock Lodge";
await context.SaveChangesAsync();</code></pre>
<p>This results in the following update to the database when using SQL Server:</p>
<pre><code class="language-sql">UPDATE [Customers] SET [Address_Line1] = @p0
OUTPUT 1
WHERE [Id] = @p1;
UPDATE [Orders] SET [BillingAddress_Line1] = @p2, [ShippingAddress_Line1] = @p3
OUTPUT 1
WHERE [Id] = @p4;</code></pre>
<p>Notice that all three <code>Line1</code> columns have changed, since they are all sharing the same instance. This is usually not what we want.</p>
<blockquote>
<p><strong>TIP</strong>
If order addresses should change automatically when the customer address changes, then consider mapping the address as an entity type. <code>Order</code> and <code>Customer</code> can then safely reference the same address instance (which is now identified by a key) via a navigation property.</p>
</blockquote>
<p>A good way to deal with issues like this it to make the type immutable. Indeed, this immutability often natural when a type is a good candidate for being a complex type. For example, it usually makes sense to supply a complex new <code>Address</code> object rather than to just mutate, say, the country while leaving the rest the same. C# records are a great fit for this!</p>
<h3>Immutable record</h3>
<p>C# 9 introduced <a href="https://review.learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record?branch=main">record types</a>, which makes creating and using immutable objects easier. For example, the <code>Address</code> object can be made a record type:</p>
<pre><code class="language-csharp">public record Address(string Line1, string? Line2, string City, string Country, string PostCode);</code></pre>
<p>It now becomes very easy to update the immutable recored with just one property value changed. For example:</p>
<pre><code class="language-csharp">customer.Address = customer.Address with { Line1 = "Peacock Lodge" };

await context.SaveChangesAsync();</code></pre>
<h3>Immutable struct record</h3>
<p>C# 10 introduced <code>struct record</code> types, which makes it easy to create and work with immutable struct records like it is with immutable class records. For example, we can define <code>Address</code> as an immutable struct record:</p>
<pre><code class="language-csharp">public readonly record struct Address(string Line1, string? Line2, string City, string Country, string PostCode);</code></pre>
<p>The code for changing the address now looks the same as when using an immutable class record:</p>
<pre><code class="language-csharp">customer.Address = customer.Address with { Line1 = "Peacock Lodge" };

await context.SaveChangesAsync();</code></pre>
<h3>Nested complex types</h3>
<p>A complex type can contain properties of other complex types. For example, let&#8217;s use our <code>Address</code> complex type from above together with a <code>PhoneNumber</code> complex type, and nest them both inside another complex type:</p>
<pre><code class="language-csharp">public record Address(string Line1, string? Line2, string City, string Country, string PostCode);

public record PhoneNumber(int CountryCode, long Number);

public record Contact
{
    public required Address Address { get; init; }
    public required PhoneNumber HomePhone { get; init; }
    public required PhoneNumber WorkPhone { get; init; }
    public required PhoneNumber MobilePhone { get; init; }
}</code></pre>
<p>We&#8217;re using immutable records here, since these are a good match for the semantics of our complex types, but nesting of complex types can be done with any flavor of .NET type.</p>
<blockquote>
<p><strong>NOTE</strong>
We&#8217;re not using a primary constructor for the <code>Contact</code> type because EF Core does not yet support constructor injection of complex type values. Vote for <a href="https://github.com/dotnet/efcore/issues/31621">Issue #31621</a> if this is important to you.</p>
</blockquote>
<p>We will add <code>Contact</code> as a property of the <code>Customer</code>:</p>
<pre><code class="language-csharp">public class Customer
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public required Contact Contact { get; set; }
    public List&lt;Order&gt; Orders { get; } = new();
}</code></pre>
<p>And <code>PhoneNumber</code> as properties of the <code>Order</code>:</p>
<pre><code class="language-csharp">public class Order
{
    public int Id { get; set; }
    public required string Contents { get; set; }
    public required PhoneNumber ContactPhone { get; set; }
    public required Address ShippingAddress { get; set; }
    public required Address BillingAddress { get; set; }
    public Customer Customer { get; set; } = null!;
}</code></pre>
<p>Configuration of nested complex types can again be achieved using <a href="https://learn.microsoft.com/dotnet/api/system.componentmodel.dataannotations.schema.complextypeattribute"><code>ComplexTypeAttribute</code></a>:</p>
<pre><code class="language-csharp">[ComplexType]
public record Address(string Line1, string? Line2, string City, string Country, string PostCode);

[ComplexType]
public record PhoneNumber(int CountryCode, long Number);

[ComplexType]
public record Contact
{
    public required Address Address { get; init; }
    public required PhoneNumber HomePhone { get; init; }
    public required PhoneNumber WorkPhone { get; init; }
    public PhoneNumrequired ber MobilePhone { get; init; }
}</code></pre>
<p>Or in <code>OnModelCreating</code>:</p>
<pre><code class="language-csharp">protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity&lt;Customer&gt;(
        b =&gt;
        {
            b.ComplexProperty(
                e =&gt; e.Contact,
                b =&gt;
                {
                    b.ComplexProperty(e =&gt; e.Address);
                    b.ComplexProperty(e =&gt; e.HomePhone);
                    b.ComplexProperty(e =&gt; e.WorkPhone);
                    b.ComplexProperty(e =&gt; e.MobilePhone);
                });
        });

    modelBuilder.Entity&lt;Order&gt;(
        b =&gt;
        {
            b.ComplexProperty(e =&gt; e.ContactPhone);
            b.ComplexProperty(e =&gt; e.BillingAddress);
            b.ComplexProperty(e =&gt; e.ShippingAddress);
        });
}</code></pre>
<h3>Queries</h3>
<p>Properties of complex types on entity types are treated like any other non-navigation property of the entity type. This means that they are always loaded when the entity type is loaded. This is also true of any nested complex type properties. For example, querying for a customer:</p>
<pre><code class="language-csharp">var customer = await context.Customers.FirstAsync(e =&gt; e.Id == customerId);</code></pre>
<p>Is translated to the following SQL when using SQL Server:</p>
<pre><code class="language-sql">SELECT TOP(1) [c].[Id], [c].[Name], [c].[Contact_Address_City], [c].[Contact_Address_Country],
    [c].[Contact_Address_Line1], [c].[Contact_Address_Line2], [c].[Contact_Address_PostCode],
    [c].[Contact_HomePhone_CountryCode], [c].[Contact_HomePhone_Number], [c].[Contact_MobilePhone_CountryCode],
    [c].[Contact_MobilePhone_Number], [c].[Contact_WorkPhone_CountryCode], [c].[Contact_WorkPhone_Number]
FROM [Customers] AS [c]
WHERE [c].[Id] = @__customerId_0</code></pre>
<p>Notice two things from this SQL:</p>
<ul>
<li>Everything is returned to populate the customer <em>and</em> all the nested <code>Contact</code>, <code>Address</code>, and <code>PhoneNumber</code> complex types.</li>
<li>All the complex type values are stored as columns in the table for the entity type. Complex types are never mapped to separate tables.</li>
</ul>
<h4>Projections</h4>
<p>Complex types can be projected from a query. For example, selecting just the shipping address from an order:</p>
<pre><code class="language-csharp">var shippingAddress = await context.Orders
    .Where(e =&gt; e.Id == orderId)
    .Select(e =&gt; e.ShippingAddress)
    .SingleAsync();</code></pre>
<p>This translates to the following when using SQL Server:</p>
<pre><code class="language-sql">SELECT TOP(2) [o].[ShippingAddress_City], [o].[ShippingAddress_Country], [o].[ShippingAddress_Line1],
    [o].[ShippingAddress_Line2], [o].[ShippingAddress_PostCode]
FROM [Orders] AS [o]
WHERE [o].[Id] = @__orderId_0</code></pre>
<p>Note that projections of complex types cannot be tracked, since complex type objects have no identity to use for tracking.</p>
<h3>Use in predicates</h3>
<p>Members of complex types can be used in predicates. For example, finding all the orders going to a certain city:</p>
<pre><code class="language-csharp">var city = "Walpole St Peter";
var walpoleOrders = await context.Orders.Where(e =&gt; e.ShippingAddress.City == city).ToListAsync();</code></pre>
<p>Which translates to the following SQL on SQL Server:</p>
<pre><code class="language-sql">SELECT [o].[Id], [o].[Contents], [o].[CustomerId], [o].[BillingAddress_City], [o].[BillingAddress_Country],
    [o].[BillingAddress_Line1], [o].[BillingAddress_Line2], [o].[BillingAddress_PostCode],
    [o].[ContactPhone_CountryCode], [o].[ContactPhone_Number], [o].[ShippingAddress_City],
    [o].[ShippingAddress_Country], [o].[ShippingAddress_Line1], [o].[ShippingAddress_Line2],
    [o].[ShippingAddress_PostCode]
FROM [Orders] AS [o]
WHERE [o].[ShippingAddress_City] = @__city_0</code></pre>
<p>A full complex type instance can also be used in predicates. For example, finding all customers with a given phone number:</p>
<pre><code class="language-csharp">var phoneNumber = new PhoneNumber(44, 7777555777);
var customersWithNumber = await context.Customers
    .Where(
        e =&gt; e.Contact.MobilePhone == phoneNumber
             || e.Contact.WorkPhone == phoneNumber
             || e.Contact.HomePhone == phoneNumber)
    .ToListAsync();</code></pre>
<p>This translates to the following SQL when using SQL Server:</p>
<pre><code class="language-sql">SELECT [c].[Id], [c].[Name], [c].[Contact_Address_City], [c].[Contact_Address_Country], [c].[Contact_Address_Line1],
     [c].[Contact_Address_Line2], [c].[Contact_Address_PostCode], [c].[Contact_HomePhone_CountryCode],
     [c].[Contact_HomePhone_Number], [c].[Contact_MobilePhone_CountryCode], [c].[Contact_MobilePhone_Number],
     [c].[Contact_WorkPhone_CountryCode], [c].[Contact_WorkPhone_Number]
FROM [Customers] AS [c]
WHERE ([c].[Contact_MobilePhone_CountryCode] = @__entity_equality_phoneNumber_0_CountryCode
    AND [c].[Contact_MobilePhone_Number] = @__entity_equality_phoneNumber_0_Number)
OR ([c].[Contact_WorkPhone_CountryCode] = @__entity_equality_phoneNumber_0_CountryCode
    AND [c].[Contact_WorkPhone_Number] = @__entity_equality_phoneNumber_0_Number)
OR ([c].[Contact_HomePhone_CountryCode] = @__entity_equality_phoneNumber_0_CountryCode
    AND [c].[Contact_HomePhone_Number] = @__entity_equality_phoneNumber_0_Number)</code></pre>
<p>Notice that equality is performed by expanding out each member of the complex type. This aligns with complex types having no key for identity and hence a complex type instance is equal to another complex type instance if and only if all their members are equal. Notice that this also aligns with equality for C# records.</p>
<h3>Manipulation of complex type values</h3>
<p>EF8 provides access to tracking information such as the current and original values of complex types and whether or not a property value has been modified. The API complex types is an extension of <a href="https://learn.microsoft.com/ef/core/change-tracking/entity-entries">the change tracking API already used for entity types</a>.</p>
<p>The <code>ComplexProperty</code> methods of <a href="https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.changetracking.entityentry">EntityEntry</a> return a entry for an entire complex object. For example, to get the current value of the <code>Order.BillingAddress</code>:</p>
<pre><code class="language-csharp">var billingAddress = context.Entry(order)
    .ComplexProperty(e =&gt; e.BillingAddress)
    .CurrentValue;</code></pre>
<p>A call to <code>Property</code> can be added to access a property of the complex type. For example to get the current value of just the billing post code:</p>
<pre><code class="language-csharp">var postCode = context.Entry(order)
    .ComplexProperty(e =&gt; e.BillingAddress)
    .Property(e =&gt; e.PostCode)
    .CurrentValue;</code></pre>
<p>Nested complex types are accessed using nested calls to <code>ComplexProperty</code>. For example, to get the city from the nested <code>Address</code> of the <code>Contact</code> on a <code>Customer</code>:</p>
<pre><code class="language-csharp">var currentCity = context.Entry(customer)
    .ComplexProperty(e =&gt; e.Contact)
    .ComplexProperty(e =&gt; e.Address)
    .Property(e =&gt; e.City)
    .CurrentValue;</code></pre>
<p>Other methods are available for reading and changing state. For example, <a href="https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.changetracking.propertyentry.ismodified#microsoft-entityframeworkcore-changetracking-propertyentry-ismodified"><code>PropertyEntry.IsModified</code></a> can be used to set a property of a complex type as modified:</p>
<pre><code class="language-csharp">context.Entry(customer)
    .ComplexProperty(e =&gt; e.Contact)
    .ComplexProperty(e =&gt; e.Address)
    .Property(e =&gt; e.PostCode)
    .IsModified = true;</code></pre>
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
<h2>How to get EF8 RC 1</h2>
<p>EF8 is distributed exclusively as a set of NuGet packages. For example, to add the SQL Server provider to your project, you can use the following command using the dotnet tool:</p>
<pre><code class="language-bash">dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.0-rc.1.23419.6</code></pre>
<h2>Installing the EF8 Command Line Interface (CLI)</h2>
<p>The <code>dotnet-ef</code> tool must be installed before executing EF8 Core migration or scaffolding commands.</p>
<p>To install the tool globally, use:</p>
<pre><code class="language-bash">dotnet tool install --global dotnet-ef --version 8.0.0-rc.1.23419.6</code></pre>
<p>If you already have the tool installed, you can upgrade it with the following command:</p>
<pre><code class="language-bash">dotnet tool update --global dotnet-ef --version 8.0.0-rc.1.23419.6</code></pre>
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
<li><a href="https://devblogs.microsoft.com/visualstudio/visual-studio-2022-17-8-preview-2-has-arrived/">Visual Studio 2022 17.8 Preview 2 Release</a></li>
<li>EF Core Community Standup Playlist: <a href="https://aka.ms/efstandups">aka.ms/efstandups</a></li>
<li>Main documentation: <a href="https://aka.ms/efdocs">aka.ms/efdocs</a></li>
<li>What&#8217;s New in EF Core 8: <a href="https://aka.ms/ef8-new">aka.ms/ef8-new</a></li>
<li>What&#8217;s New in EF Core 7: <a href="https://aka.ms/ef7-new">aka.ms/ef7-new</a></li>
<li>Issues and feature requests for EF Core: <a href="https://github.com/dotnet/efcore/issues">github.com/dotnet/efcore/issues</a></li>
<li>Entity Framework Roadmap: <a href="https://aka.ms/efroadmap">aka.ms/efroadmap</a></li>
<li>Bi-weekly updates: <a href="https://aka.ms/ef-news">aka.ms/ef-news</a></li>
</ul>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/announcing-ef8-rc1/">EF Core 8 RC1: Complex types as value objects</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
