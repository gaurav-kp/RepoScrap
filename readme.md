
# error log in table storage
```
using Microsoft.Azure.Cosmos.Table;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using System;

[ApiController]
[Route("api/[controller]")]
public class ErrorController : ControllerBase
{
    private readonly CloudTable errorLogTable;
    private readonly ILogger<ErrorController> logger;

    public ErrorController(ILogger<ErrorController> logger)
    {
        // Replace with your Azure Storage connection string and table name
        string storageConnectionString = "YourStorageConnectionString";
        string tableName = "ErrorLogs";

        // Create a CloudTable instance
        CloudStorageAccount storageAccount = CloudStorageAccount.Parse(storageConnectionString);
        CloudTableClient tableClient = storageAccount.CreateCloudTableClient(new TableClientConfiguration());
        errorLogTable = tableClient.GetTableReference(tableName);

        // Create the table if it doesn't exist
        errorLogTable.CreateIfNotExists();

        this.logger = logger;
    }

    [HttpGet("generate-error")]
    public IActionResult GenerateError()
    {
        try
        {
            // Simulate an error
            throw new Exception("This is a simulated error for logging.");

            // Additional business logic could be placed here
        }
        catch (Exception ex)
        {
            // Log the error
            LogErrorToTableStorage(ex);

            // Log the error to the application log
            logger.LogError(ex, "An error occurred while processing the request.");

            // Return a generic error response
            return StatusCode(500, "An error occurred. Please try again later.");
        }
    }

    private void LogErrorToTableStorage(Exception ex)
    {
        try
        {
            // Create an ErrorLog entity
            ErrorLog errorLog = new ErrorLog
            {
                PartitionKey = DateTime.UtcNow.ToString("yyyyMMdd"),
                RowKey = Guid.NewGuid().ToString(),
                Timestamp = DateTime.UtcNow,
                ErrorMessage = ex.Message,
                StackTrace = ex.StackTrace
            };

            // Insert the entity into the table
            TableOperation insertOperation = TableOperation.Insert(errorLog);
            errorLogTable.ExecuteAsync(insertOperation);
        }
        catch (Exception logEx)
        {
            // Log the secondary error (error during logging)
            logger.LogError(logEx, "An error occurred while logging the error to Azure Table Storage.");
        }
    }
}

public class ErrorLog : TableEntity
{
    public string ErrorMessage { get; set; }
    public string StackTrace { get; set; }
}


```

```
using Microsoft.AspNetCore.Mvc;
using DinkToPdf;
using DinkToPdf.Contracts;
using System.Collections.Generic;
using System.IO;

public class CustomerController : ControllerBase
{
    private readonly IConverter _pdfConverter;

    public CustomerController(IConverter pdfConverter)
    {
        _pdfConverter = pdfConverter;
    }

    [HttpPost]
    [Route("api/customers/report")]
    public IActionResult GenerateCustomerReport([FromBody] List<CustomerViewModel> customers)
    {
        var htmlContent = RenderRazorViewToString("Customer/CustomerReport.cshtml", customers);

        var globalSettings = new GlobalSettings
        {
            PaperSize = PaperKind.A4,
            Orientation = Orientation.Portrait,
            Margins = new MarginSettings { Top = 10, Bottom = 10, Left = 10, Right = 10 }
        };

        var objectSettings = new ObjectSettings
        {
            PagesCount = true,
            HtmlContent = htmlContent,
            WebSettings = { DefaultEncoding = "utf-8", UserStyleSheet = "wwwroot/css/style.css" }
        };

        var pdf = new HtmlToPdfDocument()
        {
            GlobalSettings = globalSettings,
            Objects = { objectSettings }
        };

        var file = _pdfConverter.Convert(pdf);
        return File(file, "application/pdf", "CustomerReport.pdf");
    }

    private string RenderRazorViewToString(string viewName, object model)
    {
        var viewEngine = HttpContext.RequestServices.GetService(typeof(ICompositeViewEngine)) as ICompositeViewEngine;

        using (var sw = new StringWriter())
        {
            var viewResult = viewEngine.FindView(ControllerContext, viewName, false);

            var viewContext = new ViewContext(
                ControllerContext,
                viewResult.View,
                new ViewDataDictionary(model),
                new TempDataDictionary(ControllerContext.HttpContext, TempData),
                sw,
                new HtmlHelperOptions()
            );

            viewResult.View.RenderAsync(viewContext);
            return sw.GetStringBuilder().ToString();
        }
    }
}



private string RenderRazorViewToString(string viewName, object model)
{
    var httpContext = new DefaultHttpContext { RequestServices = HttpContext.RequestServices };
    var actionContext = new ActionContext(httpContext, new RouteData(), new ActionDescriptor());

    using (var sw = new StringWriter())
    {
        var viewEngineResult = _viewEngine.FindView(actionContext, viewName, false);

        if (viewEngineResult.View == null)
        {
            throw new ArgumentNullException($"{viewName} does not match any available view");
        }

        var viewData = new ViewDataDictionary(new EmptyModelMetadataProvider(), new ModelStateDictionary())
        {
            Model = model
        };

        var tempData = new TempDataDictionary(actionContext.HttpContext, _tempDataProvider);

        var viewContext = new ViewContext(
            actionContext,
            viewEngineResult.View,
            viewData,
            tempData, // Pass the correct TempDataDictionary here
            sw,
            new HtmlHelperOptions()
        );

        var t = viewEngineResult.View.RenderAsync(viewContext);
        t.Wait();

        return sw.ToString();
    }
}


using System.Web.Mvc;

namespace YourWebAPIProject.Controllers
{
    public class MVCController : ApiController
    {
        public HttpResponseMessage Get()
        {
            // Render the MVC view as HTML
            var viewResult = ViewEngines.Engines.FindView(ControllerContext, "Hello", null);
            var writer = new StringWriter();
            var viewContext = new ViewContext(ControllerContext, viewResult.View, new ViewDataDictionary(), new TempDataDictionary(), writer);
            viewResult.View.Render(viewContext, writer);

            // Create an HTTP response with the HTML content
            var response = new HttpResponseMessage(HttpStatusCode.OK);
            response.Content = new StringContent(writer.ToString(), Encoding.UTF8, "text/html");

            return response;
        }
    }
}


```


# Example 2

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;
using Microsoft.AspNetCore.Mvc.ViewEngines;
using Microsoft.AspNetCore.Mvc.ViewFeatures;
using System.IO;

namespace YourWebAPIProject.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class MVCController : ControllerBase
    {
        private readonly ICompositeViewEngine _viewEngine;

        public MVCController(ICompositeViewEngine viewEngine)
        {
            _viewEngine = viewEngine;
        }

        [HttpGet]
        public IActionResult Get()
        {
            // Render the MVC view as HTML
            var viewResult = _viewEngine.FindView(ControllerContext, "Hello", false);
            var writer = new StringWriter();
            var viewContext = new ViewContext(ControllerContext, viewResult.View, new ViewDataDictionary(new EmptyModelMetadataProvider(), new ModelStateDictionary()), new TempDataDictionary(ControllerContext.HttpContext, _tempDataProvider), writer, new HtmlHelperOptions());

            viewResult.View.RenderAsync(viewContext);

            // Create an HTTP response with the HTML content
            return Content(writer.ToString(), "text/html");
        }
    }
}

```




#SQL

```sql

SELECT Id, LangCode, CtName
FROM your_table_name
WHERE (LangCode = 'fr' AND Id NOT IN (SELECT Id FROM your_table_name WHERE LangCode = 'en'))
   OR LangCode = 'en';


```
