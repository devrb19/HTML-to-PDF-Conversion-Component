# HTML-to-PDF Reusable Component — Step-by-Step Implementation Guide

## 1. Goal and scope

Build a reusable component that converts either:

- an existing webpage URL (`http://` or `https://`), or
- a raw HTML string

into a PDF document in an ASP.NET MVC 5 application.

The component is separated from the MVC demo so that another .NET Framework application can reference the Core project without taking a dependency on ASP.NET MVC.

### Design decisions

| Area | Decision |
|---|---|
| Application framework | .NET Framework 4.8 |
| Web application | ASP.NET MVC 5 |
| PDF renderer | Microsoft Edge headless command line |
| Third-party PDF library | None |
| Conversion input | URL or raw HTML (exactly one per request) |
| Initial storage | Server-side `App_Data/GeneratedPdfs` |
| Temporary files | Server-side `App_Data/HtmlToPdfTemp` |
| Storage abstraction | `IPdfStorage` |

Edge is a separately installed Microsoft browser/runtime prerequisite. The Core library starts its executable; it does not contain the browser or a PDF DLL.

## 2. Prerequisites

- Visual Studio with the **Class Library (.NET Framework)** and **ASP.NET Web Application (.NET Framework)** project templates.
- .NET Framework 4.8 targeting pack/developer pack.
- Microsoft Edge installed on the machine running the application.
- Permission for the application identity to execute Edge and create/read/delete its working and storage files.

### Verified development environment

- Edge executable: `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`
- Edge version observed during the proof of concept: `156.0.4314.58`
- The standalone PowerShell smoke test created and opened a sample PDF.
- The MVC demo subsequently generated PDFs from both a URL and raw HTML.

The Edge path and version above describe the development test machine; verify the executable path and supported behavior on each deployment server.

## 3. Create the solution and projects

1. In Visual Studio, create a **Blank Solution** named `HtmlToPdfReusable`.
2. Add a **Class Library (.NET Framework)** project named `HtmlToPdf.Core`.
3. Set the target framework to **.NET Framework 4.8**.
4. Delete the default `Class1.cs` file.
5. Add three folders to Core: `Contracts`, `Models`, and `Services`.
6. Add an **ASP.NET Web Application (.NET Framework)** project named `HtmlToPdf.MvcDemo`, also targeting .NET Framework 4.8.
7. Select the **MVC** template and **No Authentication** for the demo.
8. Add a project reference from `HtmlToPdf.MvcDemo` to `HtmlToPdf.Core` (Add Reference → Projects → Solution).
9. Set `HtmlToPdf.MvcDemo` as the startup project.

### Expected solution structure

```text
HtmlToPdfReusable
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

## 4. Core project: contracts and models

Create the following files inside `HtmlToPdf.Core`.

### 4.1 `Contracts/IHtmlToPdfService.cs`

```csharp
using HtmlToPdf.Core.Models;

namespace HtmlToPdf.Core.Contracts
{
    public interface IHtmlToPdfService
    {
        PdfDocumentResult Convert(HtmlToPdfRequest request);
    }
}
```

This is the public conversion entry point consumed by applications.

### 4.2 `Contracts/IPdfStorage.cs`

```csharp
namespace HtmlToPdf.Core.Contracts
{
    public interface IPdfStorage
    {
        string Save(byte[] pdfBytes);

        byte[] Read(string documentId);

        bool Delete(string documentId);
    }
}
```

This isolates storage from conversion. A different implementation can later store documents in cloud object storage without changing the conversion contract.

### 4.3 `Models/HtmlToPdfRequest.cs`

```csharp
namespace HtmlToPdf.Core.Models
{
    public class HtmlToPdfRequest
    {
        public string Url { get; set; }
        public string Html { get; set; }
        public string BaseUrl { get; set; }
        public string FileName { get; set; }
    }
}
```

Supply exactly one of `Url` or `Html`. `BaseUrl` is optional and is useful when raw HTML contains relative image, stylesheet, or other resource references. `FileName` is the browser download name, not the physical storage name.

### 4.4 `Models/PdfDocumentResult.cs`

```csharp
namespace HtmlToPdf.Core.Models
{
    public class PdfDocumentResult
    {
        public string DocumentId { get; set; }
        public string FileName { get; set; }
        public string ContentType { get; set; }
        public long Length { get; set; }
        public System.DateTime CreatedUtc { get; set; }
    }
}
```

The result exposes a document ID and metadata, not the physical file-system path.

## 5. Core project: file-system storage

Create `Services/FileSystemPdfStorage.cs` implementing `IPdfStorage`.

Responsibilities:

1. Accept a configured root folder in its constructor and create it if missing.
2. On `Save`, reject null/empty bytes, create a GUID document ID, save as `<guid>.pdf`, and return the ID.
3. On `Read`, validate the ID with `Guid.TryParseExact(documentId, "N", ...)`, locate the corresponding PDF, and return its bytes.
4. On `Delete`, validate the ID and delete the file if it exists.
5. Do not accept arbitrary file paths or use the caller's display filename to form the physical path.

The implemented class follows this design. Keep the GUID validation and path construction inside the storage implementation; do not let callers provide a path to read or delete.

## 6. Core project: HTML-to-PDF conversion service

Create `Services/HtmlToPdfService.cs` implementing `IHtmlToPdfService`.

Its constructor receives:

- an `IPdfStorage` implementation,
- the full path to the Edge executable,
- a temporary root directory, and
- a positive timeout in milliseconds.

Its `Convert` method follows this sequence:

1. Reject a null request.
2. Require exactly one non-empty input: URL or HTML.
3. For URLs, accept only absolute HTTP/HTTPS addresses and reject embedded credentials.
4. Sanitize the display filename and ensure it ends with `.pdf`.
5. Create a unique temporary working directory and a per-conversion Edge profile directory.
6. For raw HTML, write UTF-8 HTML to a temporary `input.html`; if Base URL is present, insert a `<base href="...">` element into the HTML head.
7. Start Edge with headless printing arguments and a dedicated profile directory.
8. Enforce the configured timeout. Check that the output file exists.
9. Read the PDF bytes and check that they begin with `%PDF-`.
10. Save the bytes using `IPdfStorage.Save` and build `PdfDocumentResult`.
11. Attempt to delete the per-conversion working directory in a `finally` block.

The converter uses a unique temporary profile per conversion to avoid sharing one Edge profile between overlapping requests. The initial implementation attempts cleanup even when a conversion fails.

### Important caveat

The service currently validates the URL scheme but does **not** implement comprehensive SSRF protection. A server-side browser can reach internal services and follow redirects. This must be fixed before accepting untrusted URLs in production.

## 7. MVC application: configuration

In `HtmlToPdf.MvcDemo/Web.config`, add these keys inside the existing `<appSettings>` element (do not create a second `<appSettings>` section):

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

`120000` milliseconds is 120 seconds. Adjust it deliberately after testing representative documents.

### 7.1 What `App_Data` means

`~/App_Data` is the ASP.NET application's own data folder. It is physically located under the deployed website directory on the server. It is **not** Windows `%APPDATA%` or `%LOCALAPPDATA%`. During local development it appears under the local project/application directory; under IIS it appears under the deployed application's physical directory.

The current demo uses:

- `App_Data/GeneratedPdfs` for durable PDF files;
- `App_Data/HtmlToPdfTemp` for transient conversion work.

Temporary per-conversion subfolders are removed in the converter's `finally` block. Do not expect the temp folder to retain completed HTML or PDF files.

For production, consider moving durable storage outside the website's deployed content directory and explicitly configuring NTFS ACLs for the application-pool identity. Confirm that direct HTTP requests cannot retrieve files from the chosen storage location.

## 8. MVC application: service factory

Create `Infrastructure/PdfServiceFactory.cs`.

The factory:

1. Reads the four settings from `ConfigurationManager.AppSettings`.
2. Resolves `~/` paths with `HostingEnvironment.MapPath`.
3. Expands environment variables for non-application-relative paths.
4. Parses and validates the timeout.
5. Creates `FileSystemPdfStorage`.
6. Creates `HtmlToPdfService` with that storage implementation and the Edge/temp/timeout settings.

This keeps MVC-specific configuration and dependency construction outside the Core library.

## 9. MVC application: form model and raw HTML request validation

Create `Models/PdfDemoViewModel.cs` with properties for `Url`, `Html`, `BaseUrl`, and `FileName`.

**Important:** ASP.NET MVC request validation blocks HTML markup in form values by default. Since this demo intentionally accepts raw HTML, add `[AllowHtml]` to the `Html` property only:

```csharp
using System.Web.Mvc;

namespace HtmlToPdf.MvcDemo.Models
{
    public class PdfDemoViewModel
    {
        public string Url { get; set; }

        [AllowHtml]
        public string Html { get; set; }

        public string BaseUrl { get; set; }
        public string FileName { get; set; }

        public PdfDemoViewModel()
        {
            FileName = "document.pdf";
        }
    }
}
```

Do not disable request validation globally just to support this one field. `[AllowHtml]` only permits the value through MVC request validation; it does not sanitize HTML, prevent script execution, or make arbitrary HTML safe.

## 10. MVC application: controller and view

Create `Controllers/PdfController.cs` with:

- `GET Index`: returns the conversion form.
- `POST Generate`: validates that the user supplied either URL or HTML, creates an `HtmlToPdfRequest`, obtains the Core services from the factory, calls `Convert`, reads the stored bytes by document ID, and returns the PDF as a browser download.

The demo POST action includes `Request.IsLocal` as a local-development guard. This is not a production authentication/authorization control and is not a substitute for SSRF defenses.

Create `Views/Pdf/Index.cshtml` with a form containing:

- Webpage URL input;
- Raw HTML textarea;
- optional Base URL input;
- PDF display filename input;
- anti-forgery token;
- Generate PDF submit button.

The page should clearly instruct the user to provide either a URL or raw HTML, not both.

## 11. Build and run

1. Save all files.
2. Select **Build → Build Solution**.
3. Set `HtmlToPdf.MvcDemo` as the startup project.
4. Press F5.
5. Open `/Pdf/Index` using the local application URL.
6. First test the URL input with a public webpage that does not require login.
7. Then leave URL blank and test raw HTML.
8. Verify that the browser downloads a PDF.
9. Inspect `HtmlToPdf.MvcDemo/App_Data/GeneratedPdfs` for a GUID-named `.pdf` file.

### Example raw HTML test

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>HTML to PDF Test</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 32px; }
        h1 { color: #1f4e79; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #999; padding: 8px; }
        th { background: #1f4e79; color: white; }
    </style>
</head>
<body>
    <h1>HTML to PDF Works!</h1>
    <p>This document was generated from raw HTML.</p>
    <table>
        <tr><th>Subject</th><th>Marks</th></tr>
        <tr><td>C# Programming</td><td>92</td></tr>
        <tr><td>Database Management</td><td>88</td></tr>
    </table>
</body>
</html>
```

For this example, Base URL is not required because styles are inline and there are no relative resource paths.

## 12. Base URL: when it is required

Base URL is optional.

| HTML contains | Base URL |
|---|---|
| Plain text, headings, tables | Not required |
| Inline `<style>` block | Not required |
| Fully qualified `https://...` image/resource URL | Usually not required for path resolution |
| Relative URL such as `images/logo.png` | Required to resolve the relative path |
| Relative stylesheet such as `css/site.css` | Required to resolve the relative path |

Example: if the HTML contains `images/logo.png`, a Base URL such as `https://example.com/` lets the browser resolve it as `https://example.com/images/logo.png`.

The server must be able to reach external resources. Authenticated pages may not render as the user's browser does because the converter does not automatically reuse the user's cookies or session.

## 13. Common issues and fixes

### ASP.NET says a potentially dangerous Request.Form value was detected

Cause: MVC request validation rejected raw HTML in the form post.

Fix: use `using System.Web.Mvc;` and `[AllowHtml]` on `PdfDemoViewModel.Html` only. Rebuild and retry.

### URL conversion works but raw HTML fails before entering the action

Check that `[AllowHtml]` is on the bound `Html` property in the actual view model used by `PdfController.Generate`.

### PDF downloads but cannot be found by its download filename

The requested filename is the browser download name. The stored file is named with a GUID. Look in `App_Data/GeneratedPdfs` for a GUID-named PDF.

### `HtmlToPdfTemp` appears empty

That is expected after a successful conversion. The service deletes each per-conversion working directory in `finally`.

### Images or CSS are missing from raw HTML

Check whether resource paths are relative. Supply a Base URL where appropriate, and ensure the IIS/application server can reach those resource hosts.

### Edge works from an interactive PowerShell session but not under IIS

The IIS application-pool identity is different from the interactive Windows user. Verify executable access, application/temp/storage directory permissions, user-profile loading requirements, and headless Edge operation under the actual pool identity.

## 14. Production hardening checklist

The current implementation is a working development proof of concept. Before production exposure, complete and test the following:

- [ ] **SSRF protection:** Resolve and validate destination IPs; deny loopback, private, link-local, multicast, and other prohibited ranges; revalidate every redirect and DNS resolution; block internal hostnames and metadata endpoints. Prefer an allowlist where possible.
- [ ] **Navigation/resource policy:** Control top-level navigation and subresource requests from raw HTML, including scripts, images, fonts, stylesheets, frames, `file:` URLs, data URLs, and redirects. Restrict outbound network access at the server/network layer as defense in depth.
- [ ] **Raw HTML policy:** Decide whether scripts are allowed. Consider sanitization or disabling JavaScript for untrusted HTML. `[AllowHtml]` is not sanitization.
- [ ] **Authentication/authorization:** Protect conversion and document retrieval; ensure users can only retrieve documents they are authorized to access. Do not rely on `Request.IsLocal` as the security boundary.
- [ ] **Resource limits:** Enforce maximum request and HTML sizes, per-user quotas, bounded concurrency, rate limiting, CPU/memory controls, and conversion timeouts.
- [ ] **IIS permissions:** Test with the actual application-pool identity. Grant least-privilege access to Edge, temporary profile and temp directories, and PDF storage. Decide how Edge user data/profile directories are managed.
- [ ] **Private storage:** Consider storing outside the web root; configure NTFS ACLs; verify files cannot be served directly; encrypt at rest if required by data classification.
- [ ] **Retention:** Define cleanup/retention for generated PDFs and stale temp folders; monitor disk usage.
- [ ] **Observability:** Add structured logging, correlation IDs, safe error responses, and diagnostics for Edge exit codes and stderr without leaking physical paths or sensitive HTML.
- [ ] **Robustness:** Test crashes, timeouts, process termination, concurrent requests, large/long documents, page size/margins, redirects, remote assets, and Edge updates.
- [ ] **Authenticated content:** Define how to handle URLs requiring authentication; the current implementation does not inherit browser cookies/session state.

## 15. Current status and next engineering tasks

### Verified in the development proof of concept

- Edge headless PDF smoke test succeeded.
- MVC URL-to-PDF conversion succeeded.
- MVC raw-HTML-to-PDF conversion succeeded after applying `[AllowHtml]` to the HTML view-model property.
- Generated PDFs were found in `App_Data/GeneratedPdfs`.
- Temporary conversion files are designed to be cleaned up after each conversion.

### Not yet verified or implemented

- Deployment and execution under a real IIS application-pool identity.
- Production SSRF/resource-request controls.
- Production authentication/authorization and per-user document ownership.
- Storage retention and cleanup job.
- Cloud storage implementation of `IPdfStorage`.
- Authenticated URL conversion and cookie/session handling.

Treat those items as future work, not completed features.
