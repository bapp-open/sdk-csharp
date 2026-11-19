# BAPP Auto API Client — C# / .NET

Official C# / .NET client for the [BAPP Auto API](https://www.bapp.ro). Provides a
simple, consistent interface for authentication, entity CRUD, and task execution.

Targets .NET Standard 2.0 and .NET 8 — works on .NET Framework 4.6.1+, .NET Core and .NET 5+. Every method is async and returns `System.Text.Json.Nodes` objects. Method names follow .NET conventions (`ListAsync`, `RunTaskAsync`, ...); the long-running `run_task_async` is `RunTaskAndWaitAsync`.

## Getting Started

### 1. Install

```bash
dotnet add package Bapp.ApiClient
```

### 2. Create a client

```csharp
using Bapp.ApiClient;

using var client = new BappApiClient(new BappApiClientOptions { Token = "your-api-key" });
```

### 3. Make your first request

```csharp
// List with filters
var countries = await client.ListAsync("core.country",
    new Dictionary<string, string> { ["page"] = "1", ["search"] = "Romania" });
Console.WriteLine($"{countries.Count} total, next: {countries.Next}");
foreach (var c in countries)
    Console.WriteLine(c["name"]);

// Get by ID
var country = await client.GetAsync("core.country", "42");

// Create (dictionary, JsonNode or any serializable object)
var created = await client.CreateAsync("core.country",
    new Dictionary<string, object?> { ["name"] = "Romania", ["code"] = "RO" });

// Update (full)
await client.UpdateAsync("core.country", "42", new { name = "Romania", code = "RO" });

// Patch (partial)
await client.PatchAsync("core.country", "42", new { code = "RO" });

// Delete
await client.DeleteAsync("core.country", "42");
```

## Authentication

The client supports **Token** (API key) and **Bearer** (JWT / OAuth) authentication.
Token auth already includes a tenant binding, so you don't need to specify `tenant` separately.

```csharp
// Static API token (tenant is included in the token)
var client = new BappApiClient(new BappApiClientOptions { Token = "your-api-key" });

// Bearer (JWT / OAuth)
var client = new BappApiClient(new BappApiClientOptions { Bearer = "eyJhbG...", Tenant = "1" });

// With IHttpClientFactory (the injected HttpClient is not disposed by the SDK)
var client = new BappApiClient(new BappApiClientOptions { Token = "your-api-key", HttpClient = httpClient });
```

## Configuration

`tenant` and `app` can be changed at any time after construction:

```csharp
client.Tenant = "2";
client.App = "wms";
```

## API Reference

### Client options

| Option | Description | Default |
|--------|-------------|---------|
| `token` | Static API token (`Token <value>`) — includes tenant | — |
| `bearer` | Bearer / JWT token | — |
| `host` | API base URL | `https://panel.bapp.ro/api` |
| `tenant` | Tenant ID (`x-tenant-id` header) | `None` |
| `app` | App slug (`x-app-slug` header) | `"account"` |
| `timeout` | HTTP request timeout (seconds) | `30` |
| `max_retries` | Max retries on transient errors (5xx, 429, connection) | `3` |

### Methods

| Method | Description |
|--------|-------------|
| `me()` | Get current user profile |
| `get_app(app_slug)` | Get app configuration by slug |
| `list(content_type, **filters)` | List entities (paginated) |
| `get(content_type, id)` | Get a single entity |
| `create(content_type, data)` | Create an entity |
| `update(content_type, id, data)` | Full update (PUT) |
| `patch(content_type, id, data)` | Partial update (PATCH) |
| `delete(content_type, id)` | Delete an entity |
| `list_introspect(content_type)` | Get list view metadata |
| `detail_introspect(content_type)` | Get detail view metadata |
| `get_document_views(record)` | Extract available views from a record |
| `get_document_url(record, output?, label?, variation?)` | Build a render/download URL |
| `get_document_content(record, output?, label?, variation?)` | Fetch document bytes (PDF, HTML, JPG) |
| `download_document(record, dest, output?, label?, variation?)` | Stream document to file (memory-efficient) |
| `list_tasks()` | List available task codes |
| `detail_task(code)` | Get task configuration |
| `run_task(code, payload?)` | Execute a task |
| `run_task_async(code, payload?)` | Run a long-running task and poll until done |

### Paginated responses

`list()` returns the results directly as a list/array. Pagination metadata is
available as extra attributes:

- `count` — total number of items across all pages
- `next` — URL of the next page (or `null`)
- `previous` — URL of the previous page (or `null`)

## File Uploads

When data contains file objects, the client automatically switches from JSON to
`multipart/form-data`. Mix regular fields and files in the same call:

```csharp
// BappFile values inside a dictionary switch the request to multipart/form-data
await client.CreateAsync("myapp.document", new Dictionary<string, object?>
{
    ["name"] = "Report",
    ["file"] = BappFile.FromPath("report.pdf", "application/pdf"),
});

// From bytes or a stream
await client.PatchAsync("myapp.document", "42", new Dictionary<string, object?>
{
    ["file"] = BappFile.FromBytes("data.csv", bytes, "text/csv"),
});

// Also works with tasks
await client.RunTaskAsync("myapp.import_data", new Dictionary<string, object?>
{
    ["format"] = "csv",
    ["file"] = BappFile.FromStream("data.csv", stream),
});
```

## Document Views

Records may include `public_view` and/or `view_token` fields with JWT tokens
for rendering documents (invoices, orders, reports, etc.) as HTML, PDF, or images.

The SDK normalises both formats and builds the correct URL automatically:

```csharp
var order = await client.GetAsync("company_order.order", "42");

// Get a PDF download URL (auto-detects public_view vs view_token)
var url = client.GetDocumentUrl(order!, output: "pdf");

// Pick a specific view by label
url = client.GetDocumentUrl(order!, output: "html", label: "Comanda interna");

// Use a variation
url = client.GetDocumentUrl(order!, output: "pdf", variation: "v4");

// Fetch the actual content as bytes
var pdfBytes = await client.GetDocumentContentAsync(order!, output: "pdf");

// Or stream it straight to a file
await client.DownloadDocumentAsync(order!, "order.pdf", output: "pdf");

// Enumerate all available views
foreach (var v in BappApiClient.GetDocumentViews(order!))
    Console.WriteLine($"{v.Label} {v.Type} {v.Variations?.Count}");
```

`get_document_views()` returns a list of normalised view entries with `label`,
`token`, `type` (`"public_view"` or `"view_token"`), `variations`, and
`default_variation`. Use it to enumerate available views (e.g. for a dropdown).

## Tasks

Tasks are server-side actions identified by a dotted code (e.g. `myapp.export_report`).

```csharp
var tasks = await client.ListTasksAsync();

var cfg = await client.DetailTaskAsync("myapp.export_report");

// Run without payload (GET)
var result = await client.RunTaskAsync("myapp.export_report");

// Run with payload (POST)
var result = await client.RunTaskAsync("myapp.export_report", new { format = "csv" });

// Long-running task: polls until finished (cross-SDK run_task_async)
var done = await client.RunTaskAndWaitAsync("myapp.export_report", new { format = "csv" },
    pollInterval: TimeSpan.FromSeconds(2), timeout: TimeSpan.FromMinutes(2));
Console.WriteLine(done?["file"]);
```

### Long-running tasks

Some tasks run asynchronously on the server. When triggered, they return an `id`
that can be polled via `bapp_framework.taskdata`. Use `run_task_async()` to
handle this automatically — it polls until `finished` is `true` and returns the
final task data (which includes a `file` URL when the task produces a download).

## License

MIT
