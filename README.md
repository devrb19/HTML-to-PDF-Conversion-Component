# HTML-to-PDF Conversion Component

A reusable HTML-to-PDF component for **ASP.NET MVC 5 / .NET Framework 4.8**. It accepts either a webpage URL or raw HTML, renders it with Microsoft Edge in headless mode, stores the resulting PDF through a storage abstraction, and returns a document result to the calling application.

> **Current status:** The local development proof of concept has successfully generated PDFs from both a webpage URL and raw HTML. This is a working demo, not yet a production-hardened public conversion service.

## Technology choices

- C# and .NET Framework 4.8
- ASP.NET MVC 5
- Microsoft Edge (headless command-line rendering)
- .NET Framework APIs such as `System.Diagnostics` and `System.IO`
- No third-party PDF DLL or PDF-generation NuGet package

Microsoft Edge is an external runtime prerequisite. It is the rendering and PDF printing engine; it is not bundled into the Core class library.

## Solution layout

```text
HtmlToPdfReusable.sln
├── HtmlToPdf.Core
│   ├── Contracts
│   │   ├── IHtmlToPdfService.cs
│   │   └── IPdfStorage.cs
│   ├── Models
│   │   ├── HtmlToPdfRequest.cs
│   │   └── PdfDocumentResult.cs
│   └── Services
│       ├── FileSystemPdfStorage.cs
│       └── HtmlToPdfService.cs
└── HtmlToPdf.MvcDemo
    ├── Controllers
    │   └── PdfController.cs
    ├── Infrastructure
    │   └── PdfServiceFactory.cs
    ├── Models
    │   └── PdfDemoViewModel.cs
    ├── Views
    │   └── Pdf
    │       └── Index.cshtml
    └── Web.config
```

The MVC project references `HtmlToPdf.Core`. The Core library has no dependency on MVC, allowing other .NET Framework applications to consume it.

## How it works

1. The MVC form submits either a URL or raw HTML.
2. `PdfController` builds an `HtmlToPdfRequest` and obtains the services from `PdfServiceFactory`.
3. `HtmlToPdfService` validates the input and creates a unique temporary working directory and Edge profile.
4. For raw HTML, the service writes a temporary `.html` file. For a URL, it opens the supplied HTTP/HTTPS address.
5. Microsoft Edge runs headlessly and prints the page to a temporary PDF.
6. The service validates the PDF signature and passes its bytes to `IPdfStorage`.
7. `FileSystemPdfStorage` saves the PDF under a GUID-based physical filename and returns the document ID.
8. The MVC demo reads the stored bytes and returns a download using the requested display filename.
9. Temporary conversion files are deleted in the service's `finally` block.

## Configuration

The current demo configuration is in `HtmlToPdf.MvcDemo/Web.config`:

```xml
<add key="HtmlToPdf.EdgeExecutablePath"
     value="C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" />
<add key="HtmlToPdf.StoragePath"
     value="~/App_Data/GeneratedPdfs" />
<add key="HtmlToPdf.TempPath"
     value="~/App_Data/HtmlToPdfTemp" />
<add key="HtmlToPdf.TimeoutMilliseconds"
     value="120000" />
```

`PdfServiceFactory` maps `~/` paths to the web application's physical path and expands environment variables in other paths.

### Storage location

`~/App_Data/GeneratedPdfs` is the ASP.NET application's **server-side App_Data folder**. It is not Windows `%APPDATA%` or `%LOCALAPPDATA%`. When deployed, the files reside on the IIS server alongside the deployed application. Keep the folder protected by IIS/ASP.NET request restrictions and NTFS permissions.

The temporary `HtmlToPdfTemp` directory is for short-lived conversion artifacts and should normally be empty after a completed conversion. The physical PDF name is a GUID such as `a83d...pdf`; the download name (for example, `StudentReport.pdf`) is returned separately in `PdfDocumentResult.FileName`.

## Run the demo

1. Open `HtmlToPdfReusable.sln` in Visual Studio.
2. Confirm both projects target **.NET Framework 4.8**.
3. Confirm `HtmlToPdf.MvcDemo` has a project reference to `HtmlToPdf.Core`.
4. Confirm the Edge executable path in `Web.config` exists on the machine running the app.
5. Set `HtmlToPdf.MvcDemo` as the startup project.
6. Build the solution.
7. Run with F5 and open `/Pdf/Index` on the local development site.
8. Test a URL in the URL field, or paste raw HTML into the HTML field. Supply one input type at a time.
9. For raw HTML, `PdfDemoViewModel.Html` must have `[AllowHtml]` so MVC request validation permits HTML markup in that one property.
10. Check `HtmlToPdf.MvcDemo/App_Data/GeneratedPdfs` for a GUID-named PDF after conversion.

## Base URL

Base URL is optional. Leave it empty for self-contained HTML, inline CSS, and fully qualified resource URLs. Supply it when raw HTML uses relative resource paths, such as `images/logo.png` or `css/site.css`, so the browser can resolve those paths against the base address.

## Common issue: ASP.NET request validation

If MVC reports “A potentially dangerous Request.Form value was detected” when raw HTML is submitted, add `using System.Web.Mvc;` and `[AllowHtml]` to the `Html` property on `PdfDemoViewModel`. Do not disable request validation globally. `[AllowHtml]` permits the input through model binding; it does not sanitize or make the HTML safe.

## Production readiness — not complete

Do **not** expose the demo conversion endpoint as a public service yet. Before production, implement and test at least:

- SSRF protection for URL navigation, DNS resolution, redirects, loopback/private/link-local IPs, and internal hostnames. URL scheme validation alone is not sufficient.
- A clear policy for raw HTML scripts, remote images/styles/fonts, file URLs, and external resource loading.
- Authentication and authorization for conversion and document retrieval; `Request.IsLocal` is only a local-development guard.
- File-size, HTML-size, concurrency, rate, and timeout limits.
- Appropriate IIS application-pool identity permissions for Edge, the temporary profile, temp folder, and storage folder; test Edge headless execution under the real pool identity.
- A retention and cleanup policy for stored PDFs and failed/stale temporary folders.
- Safe operational logging and error handling without returning sensitive internal exception details to clients.
- Testing for authenticated pages, redirects, fonts, images, long pages, concurrency, and Edge updates. The current converter does not pass through the end user's browser cookies/session.

The MVC demo currently checks `Request.IsLocal` in the POST action. That does not replace SSRF controls and should not be treated as an authorization strategy for a deployed service.

## Documentation

See [`docs/IMPLEMENTATION-GUIDE.md`](docs/IMPLEMENTATION-GUIDE.md) for the detailed step-by-step implementation walkthrough.
