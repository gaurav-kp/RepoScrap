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
