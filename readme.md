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


```
