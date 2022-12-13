# module-12-rest-api
Module 12 Lab: Http and REST
This is a hands-on exercise where you will be creating a simple RESTFul web api using C# and .Net Core. In this example you will be creating an e-commerce Api that allows clients to query product inventory and make alterations to that inventory. We will be creating test data in a database that our Api will communicate with via various endpoints. You will learn how to implement various REST endpoints, add api versioning, and generate documentation. This application will only be running on your local machine so these endpoints won’t be publicly reachable via the internet. You can work with a partner, but you may want to each implement your own solution so that you can have your own code for future reference.

Creating a project repository in Github
Create a new repository in Github. You can follow the instructions here for a refresher on how to create a repo.
Name your project repo module-12-rest-api and create it with a readme file and .gitignore file (use the VisualStudio template).
Creating a .Net Core Web Api project
Open up Visual Studio and under the Get Started section on the pop-up, click Clone a repository.
On the next screen, select the Github option under the Browse a repository section. You may need to sign in to your Github account and grant permissions.
After you have logged in, you should be able to see all of your Github repositories including the one you just created. Click on it and select the appropriate local path to clone the repo to and clone it.
Once visual studio finishes cloning and opening the folder, go to File -> Project to create a new project. You should now see the project templates dialog. Search for “api” and select the ASP.NET Core Web API template.
Finish creating the project by naming it ProductApi and make sure it gets created in the root folder of your cloned repo folder. Make sure to uncheck Place solution and project in the same directory and Configure for HTTPS. Once the project is created, click on the solution explorer and open the .sln file.
The template may have created some extra files that are unnecessary for this assignment so check with the mentor about which files can be deleted.
Creating a database for the Api
In order to implement a product inventory Api, we need a place to store the data that can be retrieved by the api (i.e. a database). We don’t want to have to create and setup a new database just for this exercise, fortunately, there is an in-memory database available via the NuGet Package source that we will be using to store information about our products. You can read more about EntityFrameworkCore and the commands available here.

In visual studio go to Project -> Manage NuGet Packages.... Click on the Browse tab and make sure that the Package source is set to nuget.org. In the search bar look for Microsoft.EntityFrameworkCore.InMemory, select the package and on the version drop-down menu select version 5.0.13 then click Install. Accept any prompts and verify there are not any build failures.
Open the solution explorer tab, right click your project and select Add -> New Folder. Name the folder Models.
Create and add a new class to the Models folder called Product.cs.
Copy the following code to your class (don’t overwrite the namespace). Import any required packages that Visual Studio suggests:
public class Product
{
  [Key]
  [Required]
  [Display(Name = "productNumber")]
  public string ProductNumber { get; set; }
 
  [Required]
  [Display(Name = "name")]
  public string Name { get; set; }
 
  [Required]
  [Range(10, 90)]
  [Display(Name = "price")]
  public double? Price { get; set; }
 
  [Required]
  [Display(Name = "department")]
  public string Department { get; set; }
}
We need to create a class for interacting with the database. Create a new folder in the project called Daos. Create a new class in this folder called ProductDao.cs. Copy the following code to the new class and import any necessary packages:
public class ProductContext : DbContext
{
  public ProductContext(DbContextOptions<ProductContext> options) : base(options)
  {
  }
 
  public DbSet<Product> Products { get; set; }
}
Because we are injecting the database context dependency, we need to configure our .net core application to enable dependency injection. Open the Startup.cs file and add the following the ConfigureServices method:
services
  .AddDbContext<ProductContext>(opt => 
              opt.UseInMemoryDatabase("Products"));
We also need to create an extension method that allows us to iterate through our products much more efficiently. Create a new folder called Extensions and then create a new class in the folder called EnumerableExtensions.cs. Copy the following code to the class:
public  static class EnumerableExtensions
{
  public static IEnumerable<T> Times<T>(this int count, Func<int, T> func)
  {
    for (var i = 1; i <= count; i++) yield return func.Invoke(i);
  }
}
Lastly, we need to create a class that seeds the data (e.g. initializes the data with values). We are going to do this via a static class. This class will loop through a list of items to create 500 different products. The names are picked at random with a department and price. Each product gets an “id” that servers as the primary key which comes from department, name, and product id. Create a class in the models folder called ProductSeed.cs and copy the following code:
public static class ProductSeed
{
  public static void InitData(ProductContext context)
  {
    var rnd = new Random();
 
    var adjectives = new [] { "Small", "Ergonomic", "Rustic", 
                                        "Smart", "Sleek" };
    var materials = new [] { "Steel", "Wooden", "Concrete", "Plastic",
                                       "Granite", "Rubber" };
    var names = new [] { "Chair", "Car", "Computer", "Pants", "Shoes" };
    var departments = new [] { "Books", "Movies", "Music", 
                                       "Games", "Electronics" };
 
    context.Products.AddRange(500.Times(x =>
    {
      var adjective = adjectives[rnd.Next(0, 5)];
      var material = materials[rnd.Next(0, 5)];
      var name = names[rnd.Next(0, 5)];
      var department = departments[rnd.Next(0, 5)];
      var productId = $"{x, -3:000}";
 
      return new Product
      {
        ProductNumber = 
           $"{department.First()}{name.First()}{productId}",
        Name = $"{adjective} {material} {name}",
        Price = (double) rnd.Next(1000, 9000) / 100,
        Department = department
      };
    }));
 
    context.SaveChanges();
  }
}
Creating HTTP endpoints for our Api
Now that we have a .net core project configured to work with an in-memory database that we are seeding the data for, we need to start adding REST endpoints to our api that allow for clients to perform various operations with the database. This is achieved by adding a Controller class and corresponding methods for each endpoint. Additionally, an Api will follow some form of versioning convention. This allows the maintainers of the api to add new functionality (e.g. new endpoints) that are part of a separate “version” or set of routes in the api. This allows clients of the Api to continue using the Api without having to upgrade to the next version right away. Versioning also allows for backwards compatibility assuming the older version endpoints remain functional.

Let’s add the Microsoft’s versioning package via NuGet. In Visual Studio, click Project -> Manage NuGet Packages.... Open the Browse tab and search for Microsoft.AspNetCore.Mvc.Versioning, click the package and install the latest version. You will also need to add the following dependency injection to your Startup.cs class under the
If you don’t already have a Controllers folder in your project, create one now. Next, create a new class in the folder called ProductsController.cs. The ProductsController is going to inheret from the ControllerBase class that contains some boilerplate functionality for api controller. We’re also going to inject our database context (e.g. ProductDao) into the controller. Lastly, we are going to call our InitData method of the ProductSeed class to produce an initial seed of the data. Copy the code below to your newly created class:
[ApiController]
[ApiVersion("1.0")]
[Route("v{version:apiVersion}/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
  private readonly ProductContext _context;
 
  public ProductsController(ProductContext context)
  {
    _context = context;
 
    if (_context.Products.Any()) return;
 
    ProductSeed.InitData(context);
  }
}
We have a controller created now, but we still haven’t created any endpoints. Let’s add a GET endpoint which will return a list of Products that are ordered by product number. We are going to be using Linq to perform operations on collections using a concise syntax (read more about Linq here). Add the following code to your ProductsController class (be sure to put it inside of the public class block):
[HttpGet]
[Route("")]
[ProducesResponseType(StatusCodes.Status200OK)]
public ActionResult<IQueryable<Product>> GetProducts()
{
  var result = _context.Products as IQueryable<Product>;
 
  return Ok(result
    .OrderBy(p => p.ProductNumber));
}
We’re now ready to start testing our Api. Make sure that the ProductApi project is set to the default startup project in VS. Press F5 or click the Launch button in VS to build and run the project. Depending on how the VS template was created, you might need to configure the project. Right click on the project in the Solution Explorer and select Properties. Next, select the debug tab and under the Launch Browser section, make sure there is no url present. Save the file and run your project.
If you setup everything correctly you shouldn’t see any build errors and a new browser window should pop up but most likely you will see a page with an HTTTP error 404 indicating that no web page (e.g. url) was not found. THis is most likely because the url in the browser is defaulting to http://localhost:37724 which isn’t defined in our api. We need to change the url to point to our endpoint which is http://localhost:37724/v1/products. You should now see a a giant array of JSON objects containing a product and it’s metadata. Feel free to hit the endpoint with Postman as well. Because we are doing a simple GET operation, we don’t need to provide a body in the request- that is all of the information needed for the request is present in the url.
Now that we have data being returned from our endpoint, it’s important to consider how we want to display and limit those requests because as the size of the data grows we need to be cognizant about scaling and performance demands. We can limit the number of products being returned for every request by replacing the following return statement in the GetProducts method on the controller with the following code:
return Ok(result
    .OrderBy(p => p.ProductNumber)
    .Take(15));
Because this endpoint does not require a resource identifier (e.g. some kind of product id), we are returning all of the products but we can certainly add the option for clients to filter the data being returned. This can be accomplished by adding a query parameter to the GET request. A query param is just a key and value that are added to the request url by starting with a “?” followed by the name of the query param and an assigned value. Any additional query params can be appended by adding “&” along with another key value pair (e.g. /v1/products?discount=15&department=electronics).
We need to add an argument to our GET method in order to take in a string called department that comes from the query params, we then needs to filter the result object to only contain products that belong to that department. Your GetProducts method should look like the following:
[HttpGet]
[Route("")]
[ProducesResponseType(StatusCodes.Status200OK)]
public ActionResult<IQueryable<Product>> GetProducts([FromQuery] string department)
{
    var result = _context.Products as IQueryable<Product>;

    if (!string.IsNullOrEmpty(department))
    {
        result = result.Where(p => p.Department.StartsWith(department, StringComparison.InvariantCultureIgnoreCase));
    }

    return Ok(result
        .OrderBy(p => p.ProductNumber)
        .Take(15));
}
Build and run the project. In the browser, go to http://localhost:37724/v1/products?department=electronics. You should now see a much smaller list being returned that only contains products belonging to the “electronics” department. Feel free to experiment with different department values for the query param.
Adding Api Documentation using Swagger
Now that our Api is taking shape, we want to be have a concise way to communicate what our endpoints are and how they should be used to other developers. Ideally, a client of your api should be able to retrieve all the information required to interact with the Api without having to open the project’s code. Fortunately, we can use the Swagger package to generate not only Api documentation but also a UI that allows us to interact with the endpoints directly without the use of external software. Swagger uses the attributes defined in the controllers (e.g. the things in brackets above each method such as [ProducesResponseType(StatusCodes.Status200OK)] to generates living documentation as the controllers are modified.).

In Visual Studio, click Project -> Manage NuGet Packages..., click the browse tab and search for Swashbuckle.AspNetCore and install the latest version.
Like the other NuGet packages we added previously, we need to update the Startup.cs file to include our dependency injection. Add the following code to the ConfigureServices method in your Startup.cs class.
services.AddSwaggerGen(c => c.SwaggerDoc("v1", new OpenApiInfo
{
  Title = "Products",
  Description = "The ultimate e-commerce store for all your needs",
  Version = "v1"
}));
We also need to update the Configure method in the Startup.cs class so that our application knows to use Swagger. Your Configure method should look like the following:
public void Configure(IApplicationBuilder app)
{
    app.UseSwagger();
    app.UseSwaggerUI(opt => opt.SwaggerEndpoint("/swagger/v1/swagger.json", "Products v1"));

    app.UseRouting();

    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
Build and run the project and navigate to http://localhost:37724/swagger and you should now see the swagger page with information about our Api and endpoints. You should see a Products drop-down (this is our controller), with the available endpoints. Click on the GET drop-down and click the Try it out button to allow you to input data. Notice how there are parameters (department and version) that we can set. Make note of which parameters are required and which are not. Enter a version and department, then click the Execute button to send the GET request. You should see a response with some information such as the request url that was used, as well as the response body, and response headers.
Another awesome feature of Swagger is that it provides an example of what a successful response with data looks like. We can also expand the schema section further down and see the data format for our Product resource which is handy for developers that are needing to use the Api and want to know exactly what the requests or response data looks like.
Everytime we run our application, we have to manually navigate to the swagger page. To avoid this, we can set the default startup route by right-clicking the project in Solution Explorer and selecting Properties. Under the debug tab, on the Launch browser field, add swagger/ as a URL. Build and run the app and now you should see the browser open up on the swagger page.
Adding additional REST endpoints with verbs
We now have an api with some basic functionality that allows us to fetch and filter some data. However, we might want clients to be able to update resources. These updates could be changes in department, price, or availability. We can add this functionality via REST verbs such as POST, PUT, PATCH, and DELETE.

The Post endpoint takes in a JSON body in the request that contains information about a new product and adds it to the database. This method is not idemopotent because it creates new resources when invoked. Add the following method to the ProductsController.cs class:
[HttpPost]
[ProducesResponseType(StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public ActionResult<Product> PostProduct([FromBody] Product product)
{
  try
  {
    _context.Products.Add(product);
    _context.SaveChanges();
 
    return new CreatedResult($"/products/{product.ProductNumber.ToLower()}", product);
  }
  catch (Exception e)
  {
    // Typically an error log is produced here
    return ValidationProblem(e.Message);
  }
}
We just added a Post method that attempts to add the product object in the request to the database context and then returns a successful Created (200) response if the update was successful otherwise it catchs the exception and returns a Bad Response (400) with the exception message. Build and run the application and notice how our new POST endpoint shows up in Swagger. Expand that endpoint and click Try it out. Notice how it already includes a sample request with all of the appropriate properties (these are not always required and don’t have to match the same number of fields in the database).
Add a version number and some sample values for a new product (try adding one with new department and verify you can fetch the product based on the department via the GET endpoint). Note: because we are using an in-memory database, if you make alterations to the database via the endpoints and then stop running the application you will lose those updates!
Try creating a request with bad data (e.g. incompatible data types for the properties, incorrect request body format, etc.) and verify you get the correct 400 response code.
Notice in the headers section for the response, you should get a 201 code and also have a Location for the new resource (e.g. product/bc916). This is giving us the exact resource location that we just created. We can add a new GET endpoint that allows us to fetch a product based on it’s product number.
Add the following additional GET method to the ProductsController.cs class. Notice in the method signature, we are passing in a [FromRoute] attribute rather than a [FromQuery] attribute. This is because we are including the resource identifier (e.g. product id) as part of the route rather than as a query param. We then use Linq to filter the list of all products based on the supplied product id. Build and run the app and perform another post so that you can get a product id and include that product id in a new GET request and verify the correct product is returned. Congratulations you just created your first API!!!
Final Steps
Once you are finished with the lab and (optional) additional exercises, push your project to your remote Github repo.
Feel free to update the readme file to include any relevant notes you think may be useful in the future. You will be coming back to these labs for guidance when you start implementing your final project.
Additional Exercises
We’ve added the ability to create records in our database via exposed endpoints, but you typically want to perform some sort of data validation prior to updating. Add some data validation around the endpoints that alter the database so that not only do request need to be in the correct format, but the the values of their data may need to meet some criteria (e.g. prices below or above a certain amount aren’t allowed).
Implement endpoints for PATCH, PUT, and DELETE operations. Refer to the EntityFrameworkCore documentation linked earlier for the appropriate commands (e.g. _context.SaveChanges();). Don’t over-think the problem, review what each REST verb does and use Linq queries to fetch the data and make the corresponding changes to the database.
Implement new endpoints for a different Api version. What features or changes do you want the new version of this endpoint to have?
Add better exception handling around areas that may be prone to issues.
Modify the Swagger hub config to include more detailed information about your api and endpoints.
