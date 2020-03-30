## Part 1: Setting up an Odata feed in .Net Core

### Options
There are a few frameworks available that can be used for this purpose, one of them being RESTier. But after agreeing on a “cloud native Azure centric” approach together with the client, and since we decided to develop the backend with the LTS supported .Net Core 2.1, I felt that a framework published by Microsoft was the preferable way to go if there was nothing talking against it.

### Prerequisites
The Odata feeds in the APIs should of course also be versioned, resulting in the following packages being installed to the project. (The versions below are the ones that were used in the .Net Core 2.1 APIs to deliver the expected results. There are some newer versions available but I will post the versions that were actually used as they are the ones that have been tested in the real world scenario.)
So I added the following NuGet packages into the ASP.NET Core project:
Install-Package Microsoft.AspNetCore.OData -Version 7.1.0
Install-Package Microsoft.AspNetCore.OData.Versioning -Version 3.0.0-beta1
Install-Package Microsoft.AspNetCore.OData.Versioning.ApiExplorer -Version 3.0.0-beta1

In the APIs I also used Swashbuckle to automatically generate Open API compliable API specification. This was specified early in project and I cannot not stress enough that Swagger documentation should be part of all APIs, it’s makes debugging easier as well as importing them into cloud resources like Azure API Management.
However, by default just installing Swashbuckle will not give full support for OData v4 support in OData controllers which was of course needed. Therefore, I also installed this package is also needed to extend Swashbuckle with the necessary OData support.
Install-Package Swashbuckle.OData -Version 3.5.0

### Setup
Since this is a .Net Core project, the services needs to be added and configured.
The APIs in question are of course versioned, so the OData operations should be too of course. Versioning the OData feeds can be a tricky endeavour, but this great guide by Chris Martinez https://github.com/microsoft/aspnet-api-versioning/wiki/API-Versioning-with-OData helped greatly, making me ending with the following configuration in Startup.cs. Only including code relevant for OData and the versioning of it.
(The order is important when adding services to the IServiceCollection in Startup.cs, AddMvc needs to be above the Addition of services.AddOData, otherwise you can get various errors.)

ConfigureServices:
...
            services.AddMvc();
            services.AddApiVersioning(options => options.ReportApiVersions = true);
            services.AddOData().EnableApiVersioning();
            services.AddODataApiExplorer(
                options =>
                {
                    // Add the versioned api explorer, which also adds IApiVersionDescriptionProvider service
                    // Note: the specified format code will format the version as "'v'major[.minor][-status]"
                    options.GroupNameFormat = "'v'VVV";

                    // Note: this option is only necessary when versioning by url segment. the SubstitutionFormat
                    // Can also be used to control the format of the API version in route templates
                    options.SubstituteApiVersionInUrl = true;

                    options.DefaultApiVersion = new ApiVersion(1, 0);
                });
 
           services.AddSwaggerGen(
                options =>
                {
                    // Resolve the IApiVersionDescriptionProvider service
                    // Note: that we have to build a temporary service provider here because one has not been created yet
                    var provider = services.BuildServiceProvider().GetRequiredService<IApiVersionDescriptionProvider>();

                    // Add a swagger document for each discovered API version
                    // Note: you might choose to skip or document deprecated API versions differently
                    foreach (var description in provider.ApiVersionDescriptions)
                    {
                        options.SwaggerDoc(description.GroupName, CreateInfoForApiVersion(description));
                    }

                    // Add a custom operation filter which sets default values
                    options.OperationFilter<SwaggerDefaultValues>();

                    // Integrate xml comments
                    options.IncludeXmlComments(XmlCommentsFilePath);
                    options.DocumentFilter<AddOdataMetadataEndpointsFilter>();

                    options.OperationFilter<PreventDuplicateConsumeFilter>();
                });
...

To avoid errors when Swashbuckle tries to generate the API documentation, we need to explicitly specify the OData MIME types as supported media types when adding MvcCore to the IServiceCollection.
...
            // Workaround: https://github.com/OData/WebApi/issues/1177
            services.AddMvcCore(options =>
            {
                foreach (var outputFormatter in options.OutputFormatters.OfType<ODataOutputFormatter>().Where(_ => _.SupportedMediaTypes.Count == 0))
                {
                    outputFormatter.SupportedMediaTypes.Add(new MediaTypeHeaderValue("application/prs.odatatestxx-odata"));
                }
                foreach (var inputFormatter in options.InputFormatters.OfType<ODataInputFormatter>().Where(_ => _.SupportedMediaTypes.Count == 0))
                {
                    inputFormatter.SupportedMediaTypes.Add(new MediaTypeHeaderValue("application/prs.odatatestxx-odata"));
                }
            });
...

Configuration:
...
            app.Use(async (context, next) => {
                context.Request.EnableRewind();
                await next();
            });

            app.UseMiddleware<TokenHandlingMiddleware>();

            app.UseHttpsRedirection();
            app.UseODataQueryStringFixer(); 
            app.UseMvc(routeBuilder =>
            {
                // Add support for all Odata filters.
                routeBuilder.Select().Expand().Filter().OrderBy().MaxTop(100).Count();
                routeBuilder.MapVersionedODataRoutes("odata", "odata/Odata.svc", modelBuilder.GetEdmModels());

                // Workaround: https://github.com/OData/WebApi/issues/1175
                routeBuilder.EnableDependencyInjection();
            }); 
...

In UseMvc we add support for the OData query filter and specifies which route we want for the OData feed, in this case “/odata/Odata.svc”.
The OdataQueryStringFixer above is an extension to enable case insensitive filtering. I will post it in its entirety below:

...
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using System.Text.RegularExpressions;
using System.Threading.Tasks;

namespace customer.order.API.Extensions
{
    public class ODataQueryStringFixer : IMiddleware
    {
        private static readonly Regex ReplaceToLowerRegex =
            new Regex(@"contains\((?<columnName>\w+),");

        public Task InvokeAsync(HttpContext context, RequestDelegate next)
        {
            var input = context.Request.QueryString.Value;
            var replacement = @"contains(tolower($1),";
            context.Request.QueryString =
                new QueryString(ReplaceToLowerRegex.Replace(input, replacement));

            return next(context);
        }
    }

    public static class ODataQueryStringFixerExtensions
    {
        public static IApplicationBuilder UseODataQueryStringFixer(this IApplicationBuilder app)
        {
            return app.UseMiddleware<ODataQueryStringFixer>();
        }
    }
}
...

### EDM
OData requires us to declare entities which can be used as OData resources. Since there was the possibility of potentially several EDM models added over time, I decided to apply the model configurations in its own class inheriting from Microsoft.AspNet.Odata.Builder.IModelConfiguration, negating the need completely of having any EDM related code in Startup.cs. (This class also becomes a good place to add additional logic in the future when new API versions are added.)
I will share the class in its entirety below:

...
using customer.order.API.Configuration;
using customer.order.API.Model;
using Microsoft.AspNet.OData.Builder;
using Microsoft.AspNetCore.Mvc;

namespace customer.invoice.API.Configuration
{
    /// <summary>
    /// Represents the model configuration for order.
    /// </summary>
    public class OrderModelConfiguration : IModelConfiguration
    {
        /// <summary>
        /// Applies model configurations using the provided builder for the specified API version.
        /// </summary>
        /// <param name="builder">The <see cref="ODataModelBuilder">builder</see> used to apply configurations.</param>
        /// <param name="apiVersion">The <see cref="ApiVersion">API version</see> associated with the <paramref name="builder"/>.</param>
        public void Apply(ODataModelBuilder builder, ApiVersion apiVersion)
        {
            EntityTypeConfiguration<Order> order = builder.EntitySet<Order>("Orders").EntityType.HasKey(o => o.OrderId);
        }
    }
}
...

### Odata controller
At this point in time we are starting to get ready to actually create the OdataController that will serve the OData feed to the consumers.

In our case, we have differentiation between standard business users and internal user in the client company. Logic to check whether a user is internal or not resides in the custom http middleare class that saves the information in the httpContext, thus why the HttpContext is injected into the OdataController constructor. Since this is a microservice architecture I have tried to keep each services as self-contained as possible to the point where I tried to keep every 
Since the controller name is used by Swashbuckle we specify a name, summary annotiations what response types the controller will return, all to provider a production worthy API specification.
The single most important line of code below is probably the [EnableQuery] as it makes the feed queryable, one of the most important features to fulfill the business need and one of the biggest differentiators compared to a static Excel export.

...
using customer.order.API.Extensions;
using customer.order.API.Infrastructure.Repositories;
using customer.order.API.Model;
using Microsoft.AspNet.OData;
using Microsoft.AspNet.OData.Routing;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using System.Collections.Generic;
using System.Threading.Tasks;
using static Microsoft.AspNetCore.Http.StatusCodes;

namespace customer.order.API.Controllers.Odata
{
    [ApiVersion("1.0")]
    [ODataRoutePrefix("Orders")]
    [ControllerName("OrderOdata")]
    public class OrderOdataController : ODataController
    {
        private readonly IOrderDataRepository _orderDataRepository;
        private readonly IHttpContextAccessor _httpContext;

        public OrderOdataController(IOrderDataRepository orderDataRepository, IHttpContextAccessor httpContext)
        {
            _orderDataRepository = orderDataRepository;
            _httpContext = httpContext;
        }

        /// <summary>
        /// GetOrdersOdata
        /// </summary>
        /// <remarks>
        /// Get customer orders.
        /// </remarks>
        /// <returns>A queryable list of orders.</returns>
        /// <response code="200">The orders were successfully retrieved.</response>
        /// <response code="204">No content could be found for the internal user.</response>
        /// <response code="400">Invalid request (missing customerId).</response>
        /// <response code="500">The request could not be processed.</response>
        [EnableQuery]
        [ProducesResponseType(typeof(ODataValue<IEnumerable<Order>>), Status200OK)]
        [ProducesResponseType(Status204NoContent)]
        [ProducesResponseType(Status400BadRequest)]
        [ProducesResponseType(Status500InternalServerError)]
        [ODataRoute]
        [HttpGet]
        public async Task<IActionResult> GetOrdersOdataAsync()
        {
            string customerId = _httpContext.CurrentCustomerId();
            bool internalAAD = _httpContext.IsInternalAADUser();

            if (string.IsNullOrEmpty(customerId) && internalAAD == false)
            {
                return BadRequest(customerId);
            }
            else if (string.IsNullOrEmpty(customerId) && internalAAD == true)
            {
                return NoContent();
            }

            return Ok(await _orderDataRepository.GetOrders(customerId));
        }
    }
}
...

### Meta data
The above are the options for the Swagger API documentation generation. Of particular note here is the options.DocumentFilter<AddOdataMetadataEndpointsFilter>(); line. By default Swagger does not include OData meta operations, something that will make us loose valuable time in the long run, and in addition I like the Swagger file to complete list of the operations, definitions, paths etc. of an API. To solve this, an operation filter is needed. In the filter below I construct the meta data operation and applies it to the Swagger documentation so that it's included when downloaded from i.e. from the UI. (In part 2 we will see one of the reasions why it's good to include this.)
...
using Swashbuckle.AspNetCore.Swagger;
using Swashbuckle.AspNetCore.SwaggerGen;
using System.Collections.Generic;

namespace customer.order.API.Extensions.Swagger
{
    public class AddOdataMetadataEndpointsFilter : IDocumentFilter
    {
        public void Apply(SwaggerDocument swaggerDoc, DocumentFilterContext context)
        {
            swaggerDoc.Paths.Add("/odata/Odata.svc/$metadata", new PathItem
            {
                Get = new Operation
                {
                    OperationId = "GetMetadata",
                    Responses = new Dictionary<string, Response>()
                    {
                        { "200" , new Response() {
                            Description = "OK", Schema = new Schema() {
                                Type = "string"
                            }
                        }
                        }
                    },
                    Tags = new List<string>() { "OdataMetadata" }
                }
            });
        }
    }
}
...
            
### Next

This is only the start though. How can you prompt a user to open a feed in i.e. Excel or PowerBi with minimal effort? How can the feed be secured with appropriate authentication with minimal hassle for the user? What needs to be considered when consuming an ODAta feed through Azure API Management? I will cover these topics in the following articles.
 
