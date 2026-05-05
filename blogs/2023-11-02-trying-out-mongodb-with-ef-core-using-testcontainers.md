---
title: "Trying out MongoDB with EF Core using Testcontainers"
url: "https://devblogs.microsoft.com/dotnet/efcore-mongodb/"
date: "Thu, 02 Nov 2023 15:00:00 +0000"
author: "Arthur Vickers"
feed_url: "https://devblogs.microsoft.com/dotnet/tag/entity-framework/feed/"
---
<p>Helping developers use both relational <em>and non-relational</em> databases effectively was one of the original tenets of EF Core. To this end, there has been an <a href="https://learn.microsoft.com/ef/core/providers/cosmos/">EF Core database provider for Azure Cosmos DB document databases</a> for many years now. Recently, the EF Core team has been collaborating with engineers from <a href="https://www.mongodb.com/">MongoDB</a> to bring support for MongoDB to EF Core. The initial result of this collaboration is the first <a href="https://www.mongodb.com/blog/post/mongodb-provider-entity-framework-core-now-available-public-preview">preview release of the MongoDB provider for EF Core</a>.</p>
<p>In this post, we will try out <a href="https://github.com/mongodb/mongo-efcore-provider">the MongoDB provider</a> for EF Core by using it to:</p>
<ul>
<li>Map a C# object model to documents in a MongoDB database</li>
<li>Use EF to save some documents to the database</li>
<li>Write LINQ queries to retrieve documents from the database</li>
<li>Make changes to a document and use EF’s change tracking to update the document</li>
</ul>
<p>The code shown in this post <a href="https://github.com/ajcvickers/MongoTestApp">can be found on GitHub</a>.</p>
<h2>Testcontainers</h2>
<p>It’s very easy to <a href="https://www.mongodb.com/atlas/database">get a MongoDB database in the cloud</a> that you can use to try things out. However, <a href="https://testcontainers.com/">Testcontainers</a> is another way to test code with different database systems which is particularly suited to:</p>
<ul>
<li>Running automated tests against the database</li>
<li>Creating standalone reproductions when reporting issues</li>
<li>Trying out new things with minimal setup</li>
</ul>
<p>Testcontainers are distributed as NuGet packages that take care of running a container containing a configured ready-to-use database system. The containers use Docker or a Docker-alternative to run, so this may need to be installed on your machine if you don’t already have it. See <a href="https://dotnet.testcontainers.org/"><em>Welcome to Testcontainers for .NET!</em></a> for more details. Other than starting Docker, you don&#8217;t need to do anything else except import the NuGet package.</p>
<h2>The C# project</h2>
<p>We&#8217;ll use a simple console application to try out MongoDB with EF Core. This project needs two package references:</p>
<ul>
<li><a href="https://www.nuget.org/packages/MongoDB.EntityFrameworkCore">MongoDB.EntityFrameworkCore</a> to install the EF Core provider. This package also transitives installs the common EF Core packages and the <a href="https://www.nuget.org/packages/MongoDB.Driver/">MongoDB.Driver</a> package which is used by the EF Provider to access the MongoDB database.</li>
<li><a href="https://www.nuget.org/packages/Testcontainers.MongoDb">Testcontainers.MongoDb</a> to install the pre-defined Testcontainer for MongoDB.</li>
</ul>
<p>The full <code>csproj</code> file looks like this:</p>
<pre><code class="language-xml">&lt;Project Sdk="Microsoft.NET.Sdk"&gt;

    &lt;PropertyGroup&gt;
        &lt;OutputType&gt;Exe&lt;/OutputType&gt;
        &lt;TargetFramework&gt;net7.0&lt;/TargetFramework&gt;
        &lt;ImplicitUsings&gt;enable&lt;/ImplicitUsings&gt;
        &lt;Nullable&gt;enable&lt;/Nullable&gt;
        &lt;RootNamespace /&gt;
    &lt;/PropertyGroup&gt;

    &lt;ItemGroup&gt;
        &lt;PackageReference Include="Testcontainers.MongoDB" Version="3.5.0" /&gt;
        &lt;PackageReference Include="MongoDB.EntityFrameworkCore" Version="7.0.0-preview.1" /&gt;
    &lt;/ItemGroup&gt;

&lt;/Project&gt;</code></pre>
<p>Remember, the full project is available to <a href="https://github.com/ajcvickers/MongoTestApp">download from GitHUb</a>.</p>
<h2>The object model</h2>
<p>We&#8217;ll map a simple object model of customers and their addresses:</p>
<pre><code class="language-csharp">public class Customer
{
    public Guid Id { get; set; }
    public required string Name { get; set; }
    public required Species Species { get; set; }
    public required ContactInfo ContactInfo { get; set; }
}

public class ContactInfo
{
    public required Address ShippingAddress { get; set; }
    public Address? BillingAddress { get; set; }
    public required PhoneNumbers Phones { get; set; }
}

public class PhoneNumbers
{
    public PhoneNumber? HomePhone { get; set; }
    public PhoneNumber? WorkPhone { get; set; }
    public PhoneNumber? MobilePhone { get; set; }
}

public class PhoneNumber
{
    public required int CountryCode { get; set; }
    public required string Number { get; set; }
}

public class Address
{
    public required string Line1 { get; set; }
    public string? Line2 { get; set; }
    public string? Line3 { get; set; }
    public required string City { get; set; }
    public required string Country { get; set; }
    public required string PostalCode { get; set; }
}

public enum Species
{
    Human,
    Dog,
    Cat
}</code></pre>
<p>Since MongoDB works with documents, we&#8217;re going to map this model to a top level Customer document, with the addresses and phone numbers embedded in this document. We&#8217;ll see how to do this in the next section.</p>
<h2>Creating the EF model</h2>
<p>EF works by <a href="https://learn.microsoft.com/ef/core/modeling/">building a model of the mapped CLR types</a>, such as those for <code>Customer</code>, etc. in the previous section. This model defines the relationships between types in the model, as well as how each type maps to the database.</p>
<p>Luckily there is not much to do here, since EF uses a set of model building conventions that generate a model based on input from both the model types and the database provider. This means that for relational databases, each type gets mapped to a different table by convention. For document databases like Azure CosmosDB and now MongoDB, only the top-level type (<code>Customer</code> in our example) is mapped to its own document. Other types referenced from the top-level types are, by-convention, included in the main document.</p>
<p>This means that the only thing EF needs to know to build a model is the top-level type, and that the MongoDB provider should be used. We do this by defining a type that extends from <code>DbContext</code>. For example:</p>
<pre><code class="language-csharp">public class CustomersContext : DbContext
{
    private readonly MongoClient _client;

    public CustomersContext(MongoClient client)
    {
        _client = client;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        =&gt; optionsBuilder.UseMongoDB(_client, "efsample");

    public DbSet&lt;Customer&gt; Customers =&gt; Set&lt;Customer&gt;();
}</code></pre>
<p>In this <code>DbContext</code> class:</p>
<ul>
<li><code>UseMongoDB</code> is called, passing in the client driver and the database name. This tells EF Core to use the MongoDB provider when building the model and accessing the database.</li>
<li>A <code>DbSet&lt;Customer&gt;</code> property that defines the top-level type for which documents should be modeled.</li>
</ul>
<p>We&#8217;ll see later how to create the <code>MongoClient</code> instance and use the <code>DbContext</code>. When we do, examining the <a href="https://learn.microsoft.com/ef/core/modeling/#debug-view">model DebugView</a> shows this:</p>
<pre><code class="language-text">Model: 
  EntityType: ContactInfo Owned
    Properties:
      CustomerId (no field, Guid) Shadow Required PK FK AfterSave:Throw
    Navigations:
      BillingAddress (Address) ToDependent ContactInfo.BillingAddress#Address (Address)
      Phones (PhoneNumbers) ToDependent PhoneNumbers
      ShippingAddress (Address) ToDependent ContactInfo.ShippingAddress#Address (Address)
    Keys:
      CustomerId PK
    Foreign keys:
      ContactInfo {'CustomerId'} -&gt; Customer {'Id'} Unique Ownership ToDependent: ContactInfo Cascade
  EntityType: ContactInfo.BillingAddress#Address (Address) CLR Type: Address Owned
    Properties:
      ContactInfoCustomerId (no field, Guid) Shadow Required PK FK AfterSave:Throw
      City (string) Required
      Country (string) Required
      Line1 (string) Required
      Line2 (string)
      Line3 (string)
      PostalCode (string) Required
    Keys:
      ContactInfoCustomerId PK
    Foreign keys:
      ContactInfo.BillingAddress#Address (Address) {'ContactInfoCustomerId'} -&gt; ContactInfo {'CustomerId'} Unique Ownership ToDependent: BillingAddress Cascade
  EntityType: ContactInfo.ShippingAddress#Address (Address) CLR Type: Address Owned
    Properties:
      ContactInfoCustomerId (no field, Guid) Shadow Required PK FK AfterSave:Throw
      City (string) Required
      Country (string) Required
      Line1 (string) Required
      Line2 (string)
      Line3 (string)
      PostalCode (string) Required
    Keys:
      ContactInfoCustomerId PK
    Foreign keys:
      ContactInfo.ShippingAddress#Address (Address) {'ContactInfoCustomerId'} -&gt; ContactInfo {'CustomerId'} Unique Ownership ToDependent: ShippingAddress Cascade
  EntityType: Customer
    Properties:
      Id (Guid) Required PK AfterSave:Throw ValueGenerated.OnAdd
      Name (string) Required
      Species (Species) Required
    Navigations:
      ContactInfo (ContactInfo) ToDependent ContactInfo
    Keys:
      Id PK
  EntityType: PhoneNumbers Owned
    Properties:
      ContactInfoCustomerId (no field, Guid) Shadow Required PK FK AfterSave:Throw
    Navigations:
      HomePhone (PhoneNumber) ToDependent PhoneNumbers.HomePhone#PhoneNumber (PhoneNumber)
      MobilePhone (PhoneNumber) ToDependent PhoneNumbers.MobilePhone#PhoneNumber (PhoneNumber)
      WorkPhone (PhoneNumber) ToDependent PhoneNumbers.WorkPhone#PhoneNumber (PhoneNumber)
    Keys:
      ContactInfoCustomerId PK
    Foreign keys:
      PhoneNumbers {'ContactInfoCustomerId'} -&gt; ContactInfo {'CustomerId'} Unique Ownership ToDependent: Phones Cascade
  EntityType: PhoneNumbers.HomePhone#PhoneNumber (PhoneNumber) CLR Type: PhoneNumber Owned
    Properties:
      PhoneNumbersContactInfoCustomerId (no field, Guid) Shadow Required PK FK AfterSave:Throw
      CountryCode (int) Required
      Number (string) Required
    Keys:
      PhoneNumbersContactInfoCustomerId PK
    Foreign keys:
      PhoneNumbers.HomePhone#PhoneNumber (PhoneNumber) {'PhoneNumbersContactInfoCustomerId'} -&gt; PhoneNumbers {'ContactInfoCustomerId'} Unique Ownership ToDependent: HomePhone Cascade
  EntityType: PhoneNumbers.MobilePhone#PhoneNumber (PhoneNumber) CLR Type: PhoneNumber Owned
    Properties:
      PhoneNumbersContactInfoCustomerId (no field, Guid) Shadow Required PK FK AfterSave:Throw
      CountryCode (int) Required
      Number (string) Required
    Keys:
      PhoneNumbersContactInfoCustomerId PK
    Foreign keys:
      PhoneNumbers.MobilePhone#PhoneNumber (PhoneNumber) {'PhoneNumbersContactInfoCustomerId'} -&gt; PhoneNumbers {'ContactInfoCustomerId'} Unique Ownership ToDependent: MobilePhone Cascade
  EntityType: PhoneNumbers.WorkPhone#PhoneNumber (PhoneNumber) CLR Type: PhoneNumber Owned
    Properties:
      PhoneNumbersContactInfoCustomerId (no field, Guid) Shadow Required PK FK AfterSave:Throw
      CountryCode (int) Required
      Number (string) Required
    Keys:
      PhoneNumbersContactInfoCustomerId PK
    Foreign keys:
      PhoneNumbers.WorkPhone#PhoneNumber (PhoneNumber) {'PhoneNumbersContactInfoCustomerId'} -&gt; PhoneNumbers {'ContactInfoCustomerId'} Unique Ownership ToDependent: WorkPhone Cascade</code></pre>
<p>Looking at this model, it can be seen that EF created <a href="https://learn.microsoft.com/ef/core/modeling/owned-entities">owned entity types</a> for the <code>ContactInfo</code>, <code>Address</code>, <code>PhoneNumber</code> and <code>PhoneNumbers</code> types, even though only the <code>Customer</code> type was referenced directly from the <code>DbContext</code>. These other types were discovered and configured by the model-building conventions.</p>
<h2>Create the MongoDB test container</h2>
<p>We now have a model and a <code>DbContext</code>. Next we need an actual MongoDB database, and this is where Testcontainers come in. There are Testcontainers available for many different types of database, and they all work in a very similar way. That is, a container is created using the appropriate <code>DbBuilder</code>, and then that container is started. For example:</p>
<pre><code class="language-csharp">await using var mongoContainer = new MongoDbBuilder()
    .WithImage("mongo:6.0")
    .Build();

await mongoContainer.StartAsync();</code></pre>
<p>And that&#8217;s it! We now have a configured, clean MongoDB instance running locally with which we can do what we wish, before just throwing it away.</p>
<h2>Save data to MongoDB</h2>
<p>Let&#8217;s use EF Core to write some data to the MongoDB database. To do this, we&#8217;ll need to create a <code>DbContext</code> instance, and for this we need a <code>MongoClient</code> instance from the underlying MongoDB driver. Often, in a real app, the <code>MongoClient</code> instance and the <code>DbContext</code> instance will be obtained using dependency injection. For the sake of simplicity, we&#8217;ll just <code>new</code> them up here:</p>
<pre><code class="language-csharp">var mongoClient = new MongoClient(mongoContainer.GetConnectionString());

await using (var context = new CustomersContext(mongoClient))
{
    // ...
}</code></pre>
<p>Notice that the Testcontainer instance provides the connection string we need to connect to our MongoDB test database.</p>
<p>To save a new <code>Customer</code> document, we&#8217;ll use <code>Add</code> to start tracking the document, and then call <code>SaveChangesAsync</code> to insert it into the database.</p>
<pre><code class="language-csharp">await using (var context = new CustomersContext(mongoClient))
{
    var customer = new Customer
    {
        Name = "Willow",
        Species = Species.Dog,
        ContactInfo = new()
        {
            ShippingAddress = new()
            {
                Line1 = "Barking Gate",
                Line2 = "Chalk Road",
                City = "Walpole St Peter",
                Country = "UK",
                PostalCode = "PE14 7QQ"
            },
            BillingAddress = new()
            {
                Line1 = "15a Main St",
                City = "Ailsworth",
                Country = "UK",
                PostalCode = "PE5 7AF"
            },
            Phones = new()
            {
                HomePhone = new() { CountryCode = 44, Number = "7877 555 555" },
                MobilePhone = new() { CountryCode = 1, Number = "(555) 2345-678" },
                WorkPhone = new() { CountryCode = 1, Number = "(555) 2345-678" }
            }
        }
    };

    context.Add(customer);
    await context.SaveChangesAsync();
}</code></pre>
<p>If we look at the JSON (actually, <a href="https://bsonspec.org/spec.html">BSON</a>, which is a more efficient binary representation for JSON documents) document created in the database, we can see it contains nested documents for all the contact information. This is different from what EF Core would do for a relational database, where each type would have been mapped to its own top-level table.</p>
<pre><code class="language-json">{
  "_id": "CSUUID(\"9a97fd67-515f-4586-a024-cf82336fc64f\")",
  "Name": "Willow",
  "Species": 1,
  "ContactInfo": {
    "BillingAddress": {
      "City": "Ailsworth",
      "Country": "UK",
      "Line1": "15a Main St",
      "Line2": null,
      "Line3": null,
      "PostalCode": "PE5 7AF"
    },
    "Phones": {
      "HomePhone": {
        "CountryCode": 44,
        "Number": "7877 555 555"
      },
      "MobilePhone": {
        "CountryCode": 1,
        "Number": "(555) 2345-678"
      },
      "WorkPhone": {
        "CountryCode": 1,
        "Number": "(555) 2345-678"
      }
    },
    "ShippingAddress": {
      "City": "Walpole St Peter",
      "Country": "UK",
      "Line1": "Barking Gate",
      "Line2": "Chalk Road",
      "Line3": null,
      "PostalCode": "PE14 7QQ"
    }
  }
}</code></pre>
<h2>Using LINQ queries</h2>
<p>EF Core supports <a href="https://learn.microsoft.com/ef/core/querying/">LINQ for querying data</a>. For example, to query a single customer:</p>
<pre><code class="language-csharp">using (var context = new CustomersContext(mongoClient))
{
    var customer = await context.Customers.SingleAsync(c =&gt; c.Name == "Willow");

    var address = customer.ContactInfo.ShippingAddress;
    var mobile = customer.ContactInfo.Phones.MobilePhone;
    Console.WriteLine($"{customer.Id}: {customer.Name}");
    Console.WriteLine($"    Shipping to: {address.City}, {address.Country} (+{mobile.CountryCode} {mobile.Number})");
}</code></pre>
<p>Running this code results in the following output:</p>
<pre><code class="language-text">336d4936-d048-469e-84c8-d5ebc17754ff: Willow
    Shipping to: Walpole St Peter, UK (+1 (555) 2345-678)</code></pre>
<p>Notice that the query pulled back the entire document, not just the <code>Customer</code> object, so we are able to access and print out the customer&#8217;s contact info without going back to the database.</p>
<p>Other LINQ operators can be used to perform filtering, etc. For example, to bring back all customers where the <code>Species</code> is <code>Dog</code>:</p>
<pre><code class="language-csharp">var customers = await context.Customers
    .Where(e =&gt; e.Species == Species.Dog)
    .ToListAsync();</code></pre>
<h2>Updating a document</h2>
<p>By default, <a href="https://learn.microsoft.com/ef/core/querying/tracking">EF tracks the object graphs returned from queries</a>. Then, when <code>SaveChanges</code> or <code>SaveChangesAsync</code> is called, EF detects any changes that have been made to the document and sends an update to MongoDB to update that document. For example:</p>
<pre><code class="language-csharp">using (var context = new CustomersContext(mongoClient))
{
    var baxter = (await context.Customers.FindAsync(baxterId))!;
    baxter.ContactInfo.ShippingAddress = new()
    {
        Line1 = "Via Giovanni Miani",
        City = "Rome",
        Country = "IT",
        PostalCode = "00154"
    };

    await context.SaveChangesAsync();
}</code></pre>
<p>In this case, we&#8217;re using <code>FindAsync</code> to query a customer by primary key&#8211;a LINQ query would work just as well. After that, we change the shipping address to Rome, and call <code>SaveChangesAsync</code>. EF detects that only the shipping address for a single document has been changed, and so sends a partial update to patch the updated address into the document stored in the MongoDB database.</p>
<h2>Going forward</h2>
<p>So far, the MongoDB provider for EF Core is only in its first preview. Full CRUD (creating, reading, updating, and deleting documents) is supported by this preview, but there are some limitations. See the <a href="https://github.com/mongodb/mongo-efcore-provider#readme">readme</a> on GitHub for more information, and for places to ask questions and file bugs.</p>
<h2>Learn more</h2>
<p>To learn more about EF Core and MongoDB:</p>
<ul>
<li>See the <a href="https://learn.microsoft.com/ef/core/">EF Core documentation</a> to learn more about using EF Core to access all kinds of databases.</li>
<li>See the <a href="https://www.mongodb.com/docs/">MongoDB documentation</a> to learn more about using MongoDB from any platform.</li>
<li>Watch <a href="https://www.youtube.com/live/Zat-ferrjro?si=WD2mjd2_dd3dquko">Introducing the MongoDB provider for EF Core</a> on the .NET Data Community Standup.</li>
<li>Watch the upcoming <a href="https://www.youtube.com/live/EQvRGG9Ine8?si=TkN5q_4ZiHPoxkJy">Announcing MongoDB Provider for Entity Framework Core</a> on the MongoDB livestream.</li>
</ul>
<h2>Summary</h2>
<p>We used <a href="https://testcontainers.com/">Testcontainers</a> to try out the <a href="https://github.com/mongodb/mongo-efcore-provider">first preview release of the MongoDB provider for EF Core</a>. Testcontainers allowed us to test MongoDB with very minimal setup, and we were able to create, query, and update documents in the MongoDB database using EF Core.</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/efcore-mongodb/">Trying out MongoDB with EF Core using Testcontainers</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
