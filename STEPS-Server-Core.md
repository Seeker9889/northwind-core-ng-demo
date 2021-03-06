# Steps for creating .NET Core Breeze Server

Here are some steps to follow to create a new .NET Core + EntityFramework backend, using Breeze to handle the data management.

Later, we'll work on a client that talks to our server.

We will assume that you've already created the database and the **NorthwindCore** directory,
following the steps in the [STEPS](./STEPS.md) document.

For the server, we'll start with an empty directory, and implement a Breeze API that
our client can use to query and update data in the database.  Along the way we will:

- Create a .NET Core solution
- Create C# entity classes from the database using EF
- Create an API for interacting with the entity model
- Create metadata from the entity model

## Create the .NET Core solution

Here we create the Visual Studio solution and the backend projects.  For this demo, we will have two projects: the model project, which implements the data model, and the server project, which implements the API.

1. Create the server project.  

    - In Visual Studio, select File / New / Project...
    - Choose project type .NET Core / **ASP.NET Core Web Application**
    - Set the project Name to **NorthwindServer**
    - Set the Location to the **NorthwindCore** directory that you created earlier
    - Make sure "Create directory for solution" is checked
    - Click OK, which takes you to the next dialog
    - Choose **ASP.NET Core 2.2** target
    - Choose **Empty** web application
    - Click OK

2. Create the model project

    - Right-click on the solution and choose Add / New Project... (or, from the top menu, select File / Add / New Project... )
    - Choose project type .NET Standard / **Class Library (.NET Standard).**
    - Set the project Name to **NorthwindModel**
    - Set the Location to the **NorwindCore/NorthwindServer** directory that was created for the solution
    - Click OK

3. Add a reference to the model project

    - Right-click on the **NorthwindServer** project and select Add / Reference...
    - Choose Projects / Solution and check **NorthwindModel**
    - Click OK
    - Set **NorthwindServer** as the startup project if it is not already.

4. Add Nuget packages to the server project.  These support the data API that we will create using Breeze.

    - Select Tools / Nuget Package Manager / Package Manager Console
    - Set Default project to **NorthwindServer**
    - `Install-Package Breeze.AspNetCore.NetCore`
    - `Install-Package Breeze.Persistence.EFCore`

5. Add Nuget packages to the model project.  These support using Entity Framework
for data access and data model creation.

    - Select Tools / Nuget Package Manager / Package Manager Console
    - Set Default project to **NorthwindModel**
    - `Install-Package Bricelam.EntityFrameworkCore.Pluralizer`
    - `Install-Package Microsoft.EntityFrameworkCore.SqlServer`

## Create the data model

We will create the data model classes from the database schema.  For this, we need a connection string for connecting to the database.  The commands below assume a local SQL Express instance; if you are using a separate server, you will need to change the Data Source and the security information in the connection string.

1. Select Tools / Nuget Package Manager / Package Manager Console
2. Set Default project to **NorthwindModel**
3. `Scaffold-DbContext "Data Source=.;Initial Catalog=NorthwindCore;Integrated Security=True;MultipleActiveResultSets=True" Microsoft.EntityFrameworkCore.SqlServer -OutputDir Models -UseDatabaseNames -Force`
    - (The command above should all be on one line)

Now you should have a **Models** folder in the **NorthwindModel** project, which contains classes for each of the NorthwindCore database tables, and a `NorthwindCoreContext.cs` that contains the EF DbContext class for accessing the database.

If you change the database schema, you can re-run the Scaffold-DbContext command to re-build the model classes.

## Create the Persistence Manager

The PersistenceManager is a Breeze class that wraps the DbContext to provide Breeze data management.  We will create it in the **NorthwindServer** project

1. Right-click on the **NorthwindServer** project and select Add / Class...
2. Name the class `NorthwindCorePersistenceManager.cs` and click Add.  This will create the class file.
3. In the class file, remove the default `using` statements and replace them with 
```
using Breeze.Persistence.EFCore;
using NorthwindModel.Models;
```
4. Make the class extend EFPersistenceManager
```
public class NorthwindCorePersistenceManager : EFPersistenceManager<NorthwindCoreContext>
```
5. Add a constructor to create it from our DbContext
```
public NorthwindCorePersistenceManager(NorthwindCoreContext dbContext) : base(dbContext) {}
```

## Add a Breeze controller

We will add a controller class that will be the interface between the client and server.

1. Right-click on the **NorthwindServer** project and select Add / New Item...
2. Choose type **API Controller Class**
3. Set the Name to `BreezeController.cs`
4. Click Add.  This will create the controller class with some default methods.
5. Delete all the existing methods in the BreezeController class
6. Update the class attributes so it can perform Breeze queries:

```
using Breeze.AspNetCore;
using Breeze.Persistence;
using Microsoft.AspNetCore.Mvc;
using Newtonsoft.Json.Linq;
using NorthwindModel.Models;
using System.Linq;
...
  [Route("api/[controller]/[action]")]
  [BreezeQueryFilter]
  public class BreezeController : Controller
```

### Add the Persistence Manager to the BreezeController

Add a new `persistenceManager` field to the `BreezeController` class, and add a constructor that takes a NorthwindCoreContext and sets the `persistenceManager` field.  This will be called by dependency injection.
```
  private NorthwindCorePersistenceManager persistenceManager;
  public BreezeController(NorthwindCoreContext dbContext)
  {
      persistenceManager = new NorthwindCorePersistenceManager(dbContext);
  }
```

### Add query methods to the BreezeController

Add a HttpGet method returning `IQueryable<>` for each of the `Customer`, `Order`, `Product`, and `Supplier` types in the data model.  We won't do one for `OrderItem` because we will only query those with and `Order`
```
  [HttpGet]
  public IQueryable<Customer> Customers()
  {
      return persistenceManager.Context.Customer;
  }
  [HttpGet]
  public IQueryable<Order> Orders()
  {
      return persistenceManager.Context.Order;
  }
  [HttpGet]
  public IQueryable<Product> Products()
  {
      return persistenceManager.Context.Product;
  }
  [HttpGet]
  public IQueryable<Supplier> Suppliers()
  {
      return persistenceManager.Context.Supplier;
  }
```
### Add a SaveChanges method to the BreezeController

This HttpPost method will be called by the client to create/update/delete entities.
```
  [HttpPost]
  public ActionResult<SaveResult> SaveChanges([FromBody] JObject saveBundle)
  {
      return persistenceManager.SaveChanges(saveBundle);
  }
```
## Compile Time!

Now make sure the solution compiles.  Don't run it yet, there are a few more things to add.

## Add connection string

Add the connection string to the `appsettings.json` file
```
  "ConnectionStrings": {
    "NorthwindCore": "Data Source=.;Initial Catalog=NorthwindCore;Integrated Security=True;MultipleActiveResultSets=True"
  },
```

## Configure the Startup class

Edit the Startup.cs file in the **NorthwindServer** project.

First we add the needed `using` statements:

```
using Breeze.AspNetCore;
using Breeze.Core;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Newtonsoft.Json.Serialization;
using NorthwindModel.Models;
```

We will need access to the `appsettings.json` configuration, so we will need to add a constructor to accept an IConfiguration instance:
```
        private IConfiguration configuration;
        public Startup(IConfiguration configuration)
        {
            this.configuration = configuration;
        }
```

In the `ConfigureServices` method, we need to 
1. Enable MVC, so our `BreezeController` class can be used to handle requests
2. Set JSON serialization options so the client-side Breeze can send and receive entities
3. Add an exception filter, so errors are communicated to the Breeze client
4. Add the DbContext to dependency injection, so our BreezeController can receive it

add some MVC options to let the Breeze client communicate with the server:
```
public void ConfigureServices(IServiceCollection services)
{
    var mvcBuilder = services.AddMvc();

    mvcBuilder.AddJsonOptions(opt => {
        // Set Breeze defaults for entity serialization
        var ss = JsonSerializationFns.UpdateWithDefaults(opt.SerializerSettings);
        if (ss.ContractResolver is DefaultContractResolver resolver)
        {
            resolver.NamingStrategy = null;  // remove json camelCasing; names are converted on the client.
        }
    });
    // Add Breeze exception filter to send errors back to the client
    mvcBuilder.AddMvcOptions(o => { o.Filters.Add(new GlobalExceptionFilter()); });

    // Add DbContext using connection string
    var connectionString = configuration.GetConnectionString("NorthwindCore");
    services.AddDbContext<NorthwindCoreContext>(options => options.UseSqlServer(connectionString));
}
```
In the `Configure` method, remove the existing `app.Run()` statement.  Then configure CORS and turn on MVC:
```
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    // Allow any host - development only!
    app.UseCors(builder => builder
        .AllowAnyHeader()
        .AllowAnyMethod()
        .SetIsOriginAllowed((host) => true)
        .AllowCredentials()
    );

    app.UseMvc();
}
```

## Configure the Program class to generate metadata

To make client-side development easier, we are going to generate Breeze _metadata_ from our entity model.  This metadata represents the structure of our entities, and will be shared between the server and the client.
Later we will use the metadata to generate TypeScript classes representing our entity types.

When we run our .NET Core application from the command line, it starts in the `Main` method of `Program.cs`.
So we will change the `Main` method to produce metadata output, instead of starting the server, when we give it a "metadata" command-line parameter:
```
public static void Main(string[] args)
{
    if (args.Length > 0 && args[0].Contains("metadata"))
    {
        // Generate metadata and exit
        var dbContext = new NorthwindCoreContext();
        var persistenceManager = new NorthwindCorePersistenceManager(dbContext);
        var metadata = persistenceManager.Metadata();
        Console.Out.WriteLine(metadata);
    }
    else
    {
        // Start web server
        CreateWebHostBuilder(args).Build().Run();
    }
}
```
## Generate the metadata

Compile the solution, then open a command prompt in the `NorthwindCore/NorthwindServer` directory.  Then run:

`dotnet NorthwindServer\bin\Debug\netcoreapp2.2\NorthwindServer.dll metadata > metadata.json`

_NOTE: Switch separators to `/` for OSX or Linux._

That will create a `metadata.json` file in the `NorthwindServer` directory.  The file
contains a big blob of JSON that represents the type information that EF knows about our data model.:
```
{"structuralTypes":[{"shortName":"Customer","namespace":"NorthwindModel.Models","autoGeneratedKeyType":"Identity","defaultResourceName":"Customer",...
```

We'll use the `metadata.json` file later when we create the web client.

## Testing the server

Compile and run the solution.  It should open a browser and attempt to open a page on the default port, e.g http://localhost:33028 ,
but it may return a 404 error because we haven't enabled static pages.  That's okay; let's test a Breeze query.

Change the URL to http://localhost:{port}/api/breeze/customers

_You can see the configured ports in NorthwindServer\Properties\launchSettings.json_

Now you should get a JSON result containing all the rows from the Customers table in the NorthwindCore database:
```
[
  {
    "$id": "1",
    "$type": "NorthwindModel.Models.Customer, NorthwindModel",
    "Id": 1,
    "FirstName": "Maria",
    "LastName": "Anders",
    "City": "Berlin",
    "Country": "Germany",
    "Phone": "030-0074321",
    "Orders": []
  },
  {
    "$id": "2",
    "$type": "NorthwindModel.Models.Customer, NorthwindModel",
    "Id": 2,
    "FirstName": "Ana",
    "LastName": "Trujillo",
    "City": "M\u00e9xico D.F.",
    "Country": "Mexico",
    "Phone": "(5) 555-4729",
    "Orders": []
  }...
]
```
If you don't get the result above, review the previous steps and try to see where you may have gone wrong.

Note that each Customer entity has an `$id` and `$type` property.  

- The `$id` is for resolving circular references - each entity appear only once in the JSON tree, and if the entity is referenced again in the tree, it will be replaced by a `$ref` that references the `$id`.

- The `$type` identifies the entity type.  The Breeze client will use this to determine what type of entity to create on the client as a result of the query.

If you don't see the `$id` and `$type`, go back to the `Startup.cs` file and make sure the settings are correct.

# Create the client

If everything looks good on the server, we are ready to work on the client application.

For an Angular app, see [STEPS-Client-Angular](./STEPS-Client-Angular.md).

<!-- For a React app see [STEPS-Client-React](./STEPS-Client-React.md). -->

<!-- For a Vue app see, [STEPS-Client-Vue](./STEPS-Client-Vue.md). -->

