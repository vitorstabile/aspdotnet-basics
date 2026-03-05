# ASP.NET Core — A Comprehensive Guide for Beginners

<h1 align="center">
    <img alt="ASP.NET Core" title="ASP.NET Core" src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/ee/.NET_Core_Logo.svg/256px-.NET_Core_Logo.svg.png" width="200px" />
</h1>

<p align="center">
  <a href="#about">About</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#contents">Contents</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#section-1">Section 1 — How ASP.NET Core Works</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#section-2">Section 2 — Middleware</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#section-3">Section 3 — Web API</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#section-4">Section 4 — Entity Framework Core</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#section-5">Section 5 — Authentication & Security</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#section-6">Section 6 — Advanced Topics</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#section-7">Section 7 — Integration Testing</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#section-8">Section 8 — Final Project</a>&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
  <a href="#references">References</a>
</p>

---

## :bookmark_tabs: About <a name = "about"></a>

This guide is a structured, step-by-step roadmap to build modern web APIs and web applications with **ASP.NET Core** — covering the request pipeline, controllers, routing, EF Core, JWT authentication, and production-ready patterns.

This guide is **Part 3** of a three-part series:

| Part | Guide | Focus |
|------|-------|-------|
| 1 | README-csharp | The C# language |
| 2 | README-dotnetcore | The .NET platform & runtime |
| **3** | **README-aspnetcore (this)** | **ASP.NET Core web development** |

> **Prerequisites:** C# fundamentals (Part 1) and .NET Core basics (Part 2).

> **How to use this guide:** Read each section in order. Every topic includes a brief explanation, key concepts, and practical code examples. Each section ends with exercises to reinforce your knowledge.

---

## :books: Contents <a name = "contents"></a>

| Section | Topic |
|---------|-------|
| [Section 1](#section-1) | How ASP.NET Core Works |
| [Section 2](#section-2) | Middleware |
| [Section 3](#section-3) | Building Web APIs |
| [Section 4](#section-4) | Entity Framework Core & Databases |
| [Section 5](#section-5) | Authentication & Security |
| [Section 6](#section-6) | Advanced Topics |
| [Section 7](#section-7) | Integration Testing |
| [Section 8](#section-8) | Final Project |

---

## Section 1 — How ASP.NET Core Works <a name = "section-1"></a>

### 1.1 What is ASP.NET Core?

**ASP.NET Core** is Microsoft's cross-platform, high-performance, open-source framework for building:
- **Web APIs** — REST APIs consumed by mobile apps, SPAs, other services
- **MVC Web Apps** — server-rendered HTML with Razor views
- **Razor Pages** — page-focused, simpler than MVC
- **Blazor** — C# on the browser via WebAssembly
- **SignalR** — real-time communication (WebSockets)
- **gRPC** — high-performance RPC services

---

### 1.2 The Request Pipeline

Every HTTP request travels through a sequential **middleware pipeline** before reaching the application logic:

```
HTTP Request
    │
    ▼
┌─────────────────────────────────────────┐
│  Exception Handling Middleware           │  ← catches unhandled exceptions
├─────────────────────────────────────────┤
│  HTTPS Redirection Middleware            │  ← HTTP → HTTPS
├─────────────────────────────────────────┤
│  Static Files Middleware                 │  ← serves wwwroot files
├─────────────────────────────────────────┤
│  Routing Middleware                      │  ← matches URL to endpoint
├─────────────────────────────────────────┤
│  CORS Middleware                         │  ← Cross-Origin Resource Sharing
├─────────────────────────────────────────┤
│  Authentication Middleware               │  ← Who are you? (parse JWT)
├─────────────────────────────────────────┤
│  Authorization Middleware                │  ← Are you allowed?
├─────────────────────────────────────────┤
│  Controller / Endpoint                   │  ← your application code
└─────────────────────────────────────────┘
    │
    ▼
HTTP Response  ← travels back up through the same pipeline
```

> **Order matters.** Middleware registered first wraps all subsequent ones. `UseAuthentication()` must come before `UseAuthorization()`.

---

### 1.3 Program.cs — Application Bootstrap

```csharp
// Program.cs — the entire app setup in one file (.NET 6+)
var builder = WebApplication.CreateBuilder(args);

// ═══════════════════════════════════════════════════════════
// PHASE 1: Register services into the DI container
// ═══════════════════════════════════════════════════════════
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddMemoryCache();

// Database
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

// Application services
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddSingleton<ITokenService, TokenService>();

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(/* options */);
builder.Services.AddAuthorization();

// CORS
builder.Services.AddCors(options =>
    options.AddPolicy("AllowFrontend",
        p => p.WithOrigins("http://localhost:3000")
              .AllowAnyMethod()
              .AllowAnyHeader()));

// ═══════════════════════════════════════════════════════════
// PHASE 2: Build the WebApplication
// ═══════════════════════════════════════════════════════════
var app = builder.Build();

// ═══════════════════════════════════════════════════════════
// PHASE 3: Configure the middleware pipeline
// ═══════════════════════════════════════════════════════════
app.UseMiddleware<GlobalExceptionMiddleware>(); // must be first

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("AllowFrontend");
app.UseAuthentication();   // before Authorization
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

### 1.4 Minimal APIs (alternative to controllers)

```csharp
// Minimal API — great for small services or microservices
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Define endpoints directly
app.MapGet("/api/products", async (IProductService svc) =>
    Results.Ok(await svc.GetAllAsync()));

app.MapGet("/api/products/{id:int}", async (int id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

app.MapPost("/api/products", async (CreateProductDto dto, IProductService svc) =>
{
    var product = await svc.CreateAsync(dto);
    return Results.CreatedAtRoute("GetProduct", new { id = product.Id }, product);
})
.WithName("CreateProduct")
.RequireAuthorization();

app.MapDelete("/api/products/{id:int}", async (int id, IProductService svc) =>
{
    await svc.DeleteAsync(id);
    return Results.NoContent();
})
.RequireAuthorization("AdminOnly");

app.Run();
```

---

### :pencil: Section 1 — Exercises

1. Create a new `webapi` project. Examine `Program.cs` and identify each middleware registered by default. Look up what each one does.
2. Add a `GET /api/info` endpoint (Minimal API style) that returns the app name, environment, and current UTC time as JSON.
3. Draw the request pipeline for your project: list each middleware in order and describe what it does to the request and response.

---

## Section 2 — Middleware <a name = "section-2"></a>

### 2.1 Middleware Fundamentals

Each middleware component:
1. Receives the `HttpContext`
2. Optionally processes the **request** (before `next`)
3. Calls `next` to pass control to the following middleware
4. Optionally processes the **response** (after `next` returns)

```csharp
// Inline middleware with app.Use
app.Use(async (HttpContext context, RequestDelegate next) =>
{
    // ← before: request processing
    Console.WriteLine($"→ {context.Request.Method} {context.Request.Path}");

    await next(context);  // ← call the next middleware

    // ← after: response processing
    Console.WriteLine($"← {context.Response.StatusCode}");
});

// Terminal middleware — does NOT call next (short-circuits)
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from terminal middleware.");
});

// Branch on path with app.Map
app.Map("/health", healthApp =>
{
    healthApp.Run(async ctx =>
        await ctx.Response.WriteAsJsonAsync(new { status = "healthy", time = DateTime.UtcNow }));
});

// Branch conditionally with app.MapWhen
app.MapWhen(
    ctx => ctx.Request.Headers.ContainsKey("X-Debug"),
    debugApp => debugApp.Use(async (ctx, next) =>
    {
        ctx.Response.Headers["X-Debug-Info"] = "Debug mode active";
        await next(ctx);
    })
);
```

---

### 2.2 Custom Middleware Class

For production middleware, always use a class:

```csharp
// Middleware/RequestTimingMiddleware.cs
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    // Singleton-lifetime services go in the constructor
    public RequestTimingMiddleware(RequestDelegate next,
        ILogger<RequestTimingMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    // Scoped/Transient services go in InvokeAsync parameters
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();

        await _next(context);

        sw.Stop();

        _logger.LogInformation(
            "{Method} {Path} responded {StatusCode} in {ElapsedMs}ms",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            sw.ElapsedMilliseconds);
    }
}

// Register with extension method (clean Program.cs)
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(
        this IApplicationBuilder app) =>
        app.UseMiddleware<RequestTimingMiddleware>();
}

// Program.cs
app.UseRequestTiming();
```

---

### 2.3 Global Exception Handling Middleware

```csharp
// Middleware/GlobalExceptionMiddleware.cs
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next,
        ILogger<GlobalExceptionMiddleware> logger)
    {
        _next   = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Unhandled exception for {Method} {Path}",
                context.Request.Method,
                context.Request.Path);

            await HandleExceptionAsync(context, ex);
        }
    }

    private static Task HandleExceptionAsync(HttpContext context, Exception ex)
    {
        (int status, string message) = ex switch
        {
            NotFoundException         => (404, ex.Message),
            ValidationException       => (400, ex.Message),
            ConflictException         => (409, ex.Message),
            ForbiddenException        => (403, ex.Message),
            UnauthorizedAccessException => (401, "Unauthorized."),
            _                         => (500, "An unexpected error occurred.")
        };

        context.Response.ContentType = "application/json";
        context.Response.StatusCode  = status;

        return context.Response.WriteAsJsonAsync(new
        {
            statusCode = status,
            message,
            path = context.Request.Path.Value,
            timestamp = DateTime.UtcNow
        });
    }
}

// Register FIRST so it catches all exceptions
app.UseMiddleware<GlobalExceptionMiddleware>();
```

---

### :pencil: Section 2 — Exercises

1. Write a `CorrelationIdMiddleware` that checks for an `X-Correlation-ID` request header. If absent, generate a `Guid`. Set it on the response header and add it to the log context via `LogContext.PushProperty`.
2. Create a `MaintenanceModeMiddleware` that reads a `MaintenanceMode:Enabled` config flag. When `true`, short-circuit all requests with `503 Service Unavailable` and a JSON message.
3. Build a `RateLimitingMiddleware` (simple in-memory version) that limits each IP address to 60 requests per minute. Return `429 Too Many Requests` when exceeded.

---

## Section 3 — Building Web APIs <a name = "section-3"></a>

### 3.1 Controllers and Routing

```csharp
// [ApiController] enables:
//   - automatic 400 response on ModelState errors
//   - binding source inference ([FromBody], [FromQuery], etc.)
//   - problem details responses
[ApiController]
[Route("api/[controller]")]          // [controller] = "Products"
public class ProductsController : ControllerBase
{
    // GET  api/products
    [HttpGet]
    public IActionResult GetAll() { ... }

    // GET  api/products/5
    [HttpGet("{id:int}", Name = "GetProduct")]
    public IActionResult GetById(int id) { ... }

    // GET  api/products/search?name=laptop&minPrice=100&maxPrice=500
    [HttpGet("search")]
    public IActionResult Search(
        [FromQuery] string? name,
        [FromQuery] decimal? minPrice,
        [FromQuery] decimal? maxPrice,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20) { ... }

    // POST api/products
    [HttpPost]
    public IActionResult Create([FromBody] CreateProductDto dto) { ... }

    // PUT  api/products/5
    [HttpPut("{id:int}")]
    public IActionResult Update(int id, [FromBody] UpdateProductDto dto) { ... }

    // PATCH api/products/5/toggle-active
    [HttpPatch("{id:int}/toggle-active")]
    public IActionResult ToggleActive(int id) { ... }

    // DELETE api/products/5
    [HttpDelete("{id:int}")]
    public IActionResult Delete(int id) { ... }
}
```

**Route constraints:**

```csharp
{id:int}          // integer only
{id:int:min(1)}   // integer >= 1
{name:alpha}      // letters only
{slug:regex(^[a-z0-9-]+$)}  // regex
{id:guid}         // GUID format
```

---

### 3.2 Model Validation with Data Annotations

```csharp
using System.ComponentModel.DataAnnotations;

public class CreateProductDto
{
    [Required(ErrorMessage = "Name is required.")]
    [StringLength(100, MinimumLength = 2,
        ErrorMessage = "Name must be between 2 and 100 characters.")]
    public string Name { get; set; } = string.Empty;

    [Range(0.01, 999_999.99,
        ErrorMessage = "Price must be between 0.01 and 999,999.99.")]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue,
        ErrorMessage = "Stock cannot be negative.")]
    public int Stock { get; set; }

    [Required]
    [Range(1, int.MaxValue, ErrorMessage = "CategoryId must be a valid ID.")]
    public int CategoryId { get; set; }

    [StringLength(500)]
    public string? Description { get; set; }
}

// [ApiController] returns 400 automatically when ModelState is invalid:
// {
//   "errors": {
//     "Name": ["Name is required."],
//     "Price": ["Price must be between 0.01 and 999,999.99."]
//   }
// }
```

---

### 3.3 FluentValidation (advanced validation)

```bash
dotnet add package FluentValidation.AspNetCore
```

```csharp
// Validators/CreateProductValidator.cs
public class CreateProductValidator : AbstractValidator<CreateProductDto>
{
    private readonly IProductRepository _repo;

    public CreateProductValidator(IProductRepository repo)
    {
        _repo = repo;

        RuleFor(x => x.Name)
            .NotEmpty()
            .MinimumLength(2).WithMessage("Name must be at least 2 characters.")
            .MaximumLength(100)
            .MustAsync(BeUniqueName).WithMessage("A product with this name already exists.");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than zero.");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0);

        RuleFor(x => x.CategoryId)
            .GreaterThan(0)
            .MustAsync(CategoryExist).WithMessage("Category does not exist.");
    }

    private async Task<bool> BeUniqueName(string name, CancellationToken ct)
        => !await _repo.ExistsByNameAsync(name);

    private async Task<bool> CategoryExist(int categoryId, CancellationToken ct)
        => await _repo.CategoryExistsAsync(categoryId);
}

// Program.cs
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductValidator>();
```

---

### 3.4 Full CRUD Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(IProductService service,
                               ILogger<ProductsController> logger)
    {
        _service = service;
        _logger  = logger;
    }

    /// <summary>Returns a paginated, filterable list of products.</summary>
    [HttpGet]
    [AllowAnonymous]
    [ProducesResponseType(typeof(PagedResult<ProductDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll(
        [FromQuery] ProductFilterDto filter,
        CancellationToken ct)
    {
        var result = await _service.GetAllAsync(filter, ct);
        Response.Headers["X-Total-Count"] = result.TotalCount.ToString();
        return Ok(result);
    }

    /// <summary>Returns a product by its ID.</summary>
    [HttpGet("{id:int}", Name = "GetProduct")]
    [AllowAnonymous]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(int id, CancellationToken ct)
    {
        var product = await _service.GetByIdAsync(id, ct);
        // NotFoundException is handled by GlobalExceptionMiddleware → 404
        return Ok(product);
    }

    /// <summary>Creates a new product.</summary>
    [HttpPost]
    [Authorize]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Create(
        [FromBody] CreateProductDto dto,
        CancellationToken ct)
    {
        var product = await _service.CreateAsync(dto, ct);
        _logger.LogInformation("Product {Id} created by {User}", product.Id,
            User.FindFirstValue(ClaimTypes.Email));
        return CreatedAtRoute("GetProduct", new { id = product.Id }, product);
    }

    /// <summary>Updates an existing product.</summary>
    [HttpPut("{id:int}")]
    [Authorize]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(
        int id,
        [FromBody] UpdateProductDto dto,
        CancellationToken ct)
    {
        await _service.UpdateAsync(id, dto, ct);
        return NoContent();
    }

    /// <summary>Deletes a product. Requires Admin role.</summary>
    [HttpDelete("{id:int}")]
    [Authorize(Roles = "Admin")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id, CancellationToken ct)
    {
        await _service.DeleteAsync(id, ct);
        return NoContent();
    }
}
```

---

### 3.5 Swagger / OpenAPI

```csharp
// Program.cs
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title       = "Products API",
        Version     = "v1",
        Description = "A RESTful API for managing products and categories.",
        Contact     = new OpenApiContact
        {
            Name  = "Dev Team",
            Email = "dev@myapi.com",
            Url   = new Uri("https://github.com/mycompany/myapi")
        }
    });

    // Enable XML comments in Swagger UI
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    c.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFile));

    // Add JWT Bearer support to Swagger UI
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name        = "Authorization",
        Type        = SecuritySchemeType.Http,
        Scheme      = "Bearer",
        BearerFormat = "JWT",
        In          = ParameterLocation.Header,
        Description = "Enter your JWT token. Example: eyJhbGci..."
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                    { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
            },
            Array.Empty<string>()
        }
    });
});

// .csproj — enable XML docs generation
// <GenerateDocumentationFile>true</GenerateDocumentationFile>
// <NoWarn>$(NoWarn);1591</NoWarn>
```

---

### :pencil: Section 3 — Exercises

1. Build a `CategoriesController` with full CRUD. Implement `GET /api/categories/{id}/products` that returns all products in a category with pagination.
2. Add a `ProductFilterDto` with `Name`, `MinPrice`, `MaxPrice`, `CategoryId`, `Page`, and `PageSize`. Implement filtering and pagination in the service layer.
3. Configure Swagger with XML documentation. Add `/// <summary>` comments to every controller action and verify they appear in the Swagger UI.

---

## Section 4 — Entity Framework Core & Databases <a name = "section-4"></a>

### 4.1 Setup and Entities

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet tool install --global dotnet-ef
```

```csharp
// Models/Category.cs
public class Category
{
    public int    Id          { get; set; }
    public string Name        { get; set; } = string.Empty;
    public string? Description { get; set; }
    public bool   IsActive    { get; set; } = true;

    public ICollection<Product> Products { get; set; } = new List<Product>();
}

// Models/Product.cs
public class Product
{
    public int      Id          { get; set; }
    public string   Name        { get; set; } = string.Empty;
    public string?  Description { get; set; }
    public decimal  Price       { get; set; }
    public int      Stock       { get; set; }
    public bool     IsActive    { get; set; } = true;
    public DateTime CreatedAt   { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt  { get; set; }

    public int       CategoryId { get; set; }
    public Category  Category   { get; set; } = null!;
}

// Data/AppDbContext.cs
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product>  Products   { get; set; }
    public DbSet<Category> Categories { get; set; }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.Entity<Product>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Name).IsRequired().HasMaxLength(100);
            e.Property(p => p.Price).HasColumnType("decimal(18,2)");
            e.Property(p => p.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
            e.HasIndex(p => p.Name);
            e.HasIndex(p => p.CategoryId);

            e.HasOne(p => p.Category)
             .WithMany(c => c.Products)
             .HasForeignKey(p => p.CategoryId)
             .OnDelete(DeleteBehavior.Restrict);
        });

        builder.Entity<Category>(e =>
        {
            e.HasKey(c => c.Id);
            e.Property(c => c.Name).IsRequired().HasMaxLength(50);
        });

        // Seed data
        builder.Entity<Category>().HasData(
            new Category { Id = 1, Name = "Electronics",  IsActive = true },
            new Category { Id = 2, Name = "Accessories",  IsActive = true },
            new Category { Id = 3, Name = "Books",        IsActive = true }
        );
    }
}

// Program.cs
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));
```

---

### 4.2 Migrations

```bash
# Create the initial migration
dotnet ef migrations add InitialCreate

# Apply to database
dotnet ef database update

# After changing models
dotnet ef migrations add AddProductDescription
dotnet ef database update

# See what SQL will be executed
dotnet ef migrations script

# Roll back
dotnet ef database update PreviousMigrationName

# Remove last unapplied migration
dotnet ef migrations remove
```

**Auto-apply migrations on startup (useful for dev/tests):**

```csharp
// Program.cs
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync(); // applies any pending migrations
}
```

---

### 4.3 Repository Pattern

```csharp
// Repositories/Interfaces/IProductRepository.cs
public interface IProductRepository
{
    Task<(IEnumerable<Product> Items, int TotalCount)> GetAllAsync(
        ProductFilterDto filter, CancellationToken ct = default);
    Task<Product?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<bool> ExistsByNameAsync(string name, CancellationToken ct = default);
    Task<Product> CreateAsync(Product product, CancellationToken ct = default);
    Task UpdateAsync(Product product, CancellationToken ct = default);
    Task DeleteAsync(Product product, CancellationToken ct = default);
}

// Repositories/ProductRepository.cs
public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context) => _context = context;

    public async Task<(IEnumerable<Product> Items, int TotalCount)> GetAllAsync(
        ProductFilterDto filter, CancellationToken ct = default)
    {
        var query = _context.Products
                            .Include(p => p.Category)
                            .AsQueryable();

        // Filtering
        if (!string.IsNullOrWhiteSpace(filter.Name))
            query = query.Where(p => p.Name.Contains(filter.Name));

        if (filter.MinPrice.HasValue)
            query = query.Where(p => p.Price >= filter.MinPrice.Value);

        if (filter.MaxPrice.HasValue)
            query = query.Where(p => p.Price <= filter.MaxPrice.Value);

        if (filter.CategoryId.HasValue)
            query = query.Where(p => p.CategoryId == filter.CategoryId.Value);

        if (filter.IsActive.HasValue)
            query = query.Where(p => p.IsActive == filter.IsActive.Value);

        // Count before pagination
        var total = await query.CountAsync(ct);

        // Ordering + Pagination
        var items = await query
            .OrderBy(p => p.Name)
            .Skip((filter.Page - 1) * filter.PageSize)
            .Take(filter.PageSize)
            .ToListAsync(ct);

        return (items, total);
    }

    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
        => await _context.Products
                         .Include(p => p.Category)
                         .FirstOrDefaultAsync(p => p.Id == id, ct);

    public async Task<bool> ExistsByNameAsync(string name, CancellationToken ct = default)
        => await _context.Products.AnyAsync(p => p.Name == name, ct);

    public async Task<Product> CreateAsync(Product product, CancellationToken ct = default)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync(ct);
        return product;
    }

    public async Task UpdateAsync(Product product, CancellationToken ct = default)
    {
        product.UpdatedAt = DateTime.UtcNow;
        _context.Products.Update(product);
        await _context.SaveChangesAsync(ct);
    }

    public async Task DeleteAsync(Product product, CancellationToken ct = default)
    {
        _context.Products.Remove(product);
        await _context.SaveChangesAsync(ct);
    }
}
```

---

### :pencil: Section 4 — Exercises

1. Add a `User` entity to track which user created each product. Add `CreatedByUserId` to `Product` and configure the relationship.
2. Write a repository method `GetTopSellingAsync(int count)` that returns the `n` products with the highest stock, ordered descending.
3. Implement soft-delete: instead of removing records, set `IsDeleted = true`. Add a global query filter in `OnModelCreating` so deleted records are always excluded automatically.

---

## Section 5 — Authentication & Security <a name = "section-5"></a>

### 5.1 JWT Authentication Flow

```
POST /api/auth/login  { email, password }
         ↓
   Server validates credentials
         ↓
   Server generates JWT token:
     Header.Payload.Signature
         ↓
   Client stores token
         ↓
   Client sends: Authorization: Bearer <token>
         ↓
   Server validates token on every request
         ↓
   Access granted / denied
```

---

### 5.2 Setting Up JWT

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package BCrypt.Net-Next
```

```json
// appsettings.json
{
  "Jwt": {
    "Key": "your-super-secret-key-at-least-32-chars-!!",
    "Issuer": "https://myapi.com",
    "Audience": "https://myapi.com",
    "ExpirationMinutes": 60
  }
}
```

```csharp
// Program.cs
var jwtKey = builder.Configuration["Jwt:Key"]!;

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer              = builder.Configuration["Jwt:Issuer"],
            ValidAudience            = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey         = new SymmetricSecurityKey(
                                           Encoding.UTF8.GetBytes(jwtKey)),
            ClockSkew                = TimeSpan.Zero
        };

        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                if (ctx.Exception is SecurityTokenExpiredException)
                    ctx.Response.Headers["Token-Expired"] = "true";
                return Task.CompletedTask;
            }
        };
    });
```

---

### 5.3 Token Service and Auth Controller

```csharp
// Services/TokenService.cs
public interface ITokenService
{
    string GenerateToken(int userId, string email, string role);
}

public class TokenService : ITokenService
{
    private readonly IConfiguration _config;
    public TokenService(IConfiguration config) => _config = config;

    public string GenerateToken(int userId, string email, string role)
    {
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_config["Jwt:Key"]!));

        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub,  userId.ToString()),
            new Claim(JwtRegisteredClaimNames.Email, email),
            new Claim(JwtRegisteredClaimNames.Jti,  Guid.NewGuid().ToString()),
            new Claim(ClaimTypes.NameIdentifier,     userId.ToString()),
            new Claim(ClaimTypes.Email,              email),
            new Claim(ClaimTypes.Role,               role),
        };

        var token = new JwtSecurityToken(
            issuer:   _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims:   claims,
            expires:  DateTime.UtcNow.AddMinutes(
                          _config.GetValue<int>("Jwt:ExpirationMinutes")),
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256)
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}

// Controllers/AuthController.cs
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;

    public AuthController(IAuthService authService) => _authService = authService;

    /// <summary>Register a new user.</summary>
    [HttpPost("register")]
    [AllowAnonymous]
    [ProducesResponseType(typeof(AuthResponseDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> Register(
        [FromBody] RegisterDto dto, CancellationToken ct)
    {
        var response = await _authService.RegisterAsync(dto, ct);
        return CreatedAtAction(nameof(GetProfile), null, response);
    }

    /// <summary>Login and receive a JWT token.</summary>
    [HttpPost("login")]
    [AllowAnonymous]
    [ProducesResponseType(typeof(AuthResponseDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> Login(
        [FromBody] LoginDto dto, CancellationToken ct)
    {
        var response = await _authService.LoginAsync(dto, ct);
        return Ok(response);
    }

    /// <summary>Get the current user's profile.</summary>
    [HttpGet("profile")]
    [Authorize]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public IActionResult GetProfile()
    {
        return Ok(new
        {
            Id    = User.FindFirstValue(ClaimTypes.NameIdentifier),
            Email = User.FindFirstValue(ClaimTypes.Email),
            Role  = User.FindFirstValue(ClaimTypes.Role)
        });
    }
}
```

---

### 5.4 Authorization

```csharp
// ─── Role-based ────────────────────────────────────────────
[Authorize]                          // any authenticated user
[Authorize(Roles = "Admin")]         // Admin only
[Authorize(Roles = "Admin,Manager")] // Admin OR Manager
[AllowAnonymous]                     // override — no auth needed

// ─── Policy-based ──────────────────────────────────────────
// Register policies in Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly",          p => p.RequireRole("Admin"));
    options.AddPolicy("CanManageProducts",  p => p.RequireRole("Admin", "Manager"));
    options.AddPolicy("PremiumUser",        p =>
        p.RequireClaim("SubscriptionLevel", "Premium", "Enterprise"));
    options.AddPolicy("InternalService",    p =>
        p.RequireAssertion(ctx =>
            ctx.User.HasClaim("scope", "internal-api")));
});

[Authorize(Policy = "CanManageProducts")]
[HttpPost]
public async Task<IActionResult> Create([FromBody] CreateProductDto dto) { ... }

// ─── Resource-based authorization ─────────────────────────
// Useful when authorization depends on the specific resource
public class ProductAuthorizationHandler
    : AuthorizationHandler<OwnerRequirement, Product>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OwnerRequirement requirement,
        Product resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (resource.OwnerId.ToString() == userId ||
            context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}
```

---

### :pencil: Section 5 — Exercises

1. Implement `POST /api/auth/register` that stores the user in the database with a BCrypt-hashed password. Return a JWT on success.
2. Implement `POST /api/auth/login` that validates credentials and returns a JWT with `Id`, `Email`, and `Role` claims.
3. Add a `POST /api/auth/refresh` endpoint that accepts an expired (but otherwise valid) token and returns a new one. Hint: disable lifetime validation in `TokenValidationParameters` for this endpoint only.

---

## Section 6 — Advanced Topics <a name = "section-6"></a>

### 6.1 Caching

```csharp
// ─── In-Memory Cache ───────────────────────────────────────
builder.Services.AddMemoryCache();

public class ProductService : IProductService
{
    private readonly IProductRepository _repo;
    private readonly IMemoryCache _cache;
    private const string AllProductsCacheKey = "products:all";

    public async Task<IEnumerable<ProductDto>> GetAllAsync(CancellationToken ct)
    {
        if (!_cache.TryGetValue(AllProductsCacheKey,
                out IEnumerable<ProductDto>? cached))
        {
            var products = await _repo.GetAllAsync(ct);
            cached = products.Select(MapToDto);

            _cache.Set(AllProductsCacheKey, cached, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
                SlidingExpiration               = TimeSpan.FromMinutes(2)
            });
        }
        return cached!;
    }

    private void InvalidateCache()
    {
        _cache.Remove(AllProductsCacheKey);
    }
}

// ─── Response Caching ──────────────────────────────────────
builder.Services.AddResponseCaching();
app.UseResponseCaching();

[HttpGet]
[AllowAnonymous]
[ResponseCache(Duration = 30, VaryByQueryKeys = new[] { "name", "page" })]
public async Task<IActionResult> GetAll([FromQuery] ProductFilterDto filter) { ... }
```

---

### 6.2 Pagination

```csharp
// DTOs/PagedResult.cs
public class PagedResult<T>
{
    public IEnumerable<T> Items     { get; set; } = Enumerable.Empty<T>();
    public int            Page      { get; set; }
    public int            PageSize  { get; set; }
    public int            TotalCount { get; set; }
    public int            TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool           HasNext   => Page < TotalPages;
    public bool           HasPrevious => Page > 1;
}

// Usage in controller
[HttpGet]
public async Task<IActionResult> GetAll([FromQuery] ProductFilterDto filter)
{
    var result = await _service.GetPagedAsync(filter);
    Response.Headers["X-Total-Count"] = result.TotalCount.ToString();
    Response.Headers["X-Total-Pages"] = result.TotalPages.ToString();
    return Ok(result);
}
```

---

### 6.3 CORS

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("Development", policy =>
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader());

    options.AddPolicy("Production", policy =>
        policy.WithOrigins(
                  "https://myapp.com",
                  "https://www.myapp.com")
              .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH")
              .WithHeaders("Content-Type", "Authorization")
              .AllowCredentials()
              .SetPreflightMaxAge(TimeSpan.FromMinutes(10)));
});

app.UseCors(app.Environment.IsDevelopment() ? "Development" : "Production");
```

---

### 6.4 Background Services

```csharp
// Worker that runs on a schedule
public class ProductCleanupWorker : BackgroundService
{
    private readonly ILogger<ProductCleanupWorker> _logger;
    private readonly IServiceScopeFactory _scopeFactory;

    public ProductCleanupWorker(ILogger<ProductCleanupWorker> logger,
                                 IServiceScopeFactory scopeFactory)
    {
        _logger      = logger;
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("ProductCleanupWorker started.");

        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Running cleanup at {Time}", DateTime.UtcNow);

            // Must create a scope to use Scoped services in a Singleton worker
            using var scope = _scopeFactory.CreateScope();
            var repo = scope.ServiceProvider.GetRequiredService<IProductRepository>();
            // ... cleanup logic

            await Task.Delay(TimeSpan.FromHours(24), stoppingToken);
        }
    }
}

// Register
builder.Services.AddHostedService<ProductCleanupWorker>();
```

---

### 6.5 HttpClient and IHttpClientFactory

```csharp
// Register typed HTTP client
builder.Services.AddHttpClient<IExternalProductService, ExternalProductService>(client =>
{
    client.BaseAddress = new Uri("https://api.external.com/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddTransientHttpErrorPolicy(p =>
    p.WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))));

// Typed HTTP client service
public class ExternalProductService : IExternalProductService
{
    private readonly HttpClient _client;

    public ExternalProductService(HttpClient client) => _client = client;

    public async Task<IEnumerable<ExternalProduct>> GetProductsAsync()
    {
        var response = await _client.GetFromJsonAsync<IEnumerable<ExternalProduct>>(
            "products");
        return response ?? Enumerable.Empty<ExternalProduct>();
    }
}
```

---

### :pencil: Section 6 — Exercises

1. Add in-memory caching to `GET /api/products`. Bust the cache on every create, update, and delete. Verify with logging that the cache is hit on the second request.
2. Implement proper pagination for all list endpoints. Include `X-Total-Count`, `X-Total-Pages`, and pagination metadata in the response body.
3. Create a `DatabaseSeedWorker` (BackgroundService) that checks on startup if the database is empty and seeds it with initial data. Run it only in Development.

---

## Section 7 — Integration Testing <a name = "section-7"></a>

### 7.1 WebApplicationFactory

Integration tests test the real HTTP pipeline from request to response, using an in-memory server.

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

```csharp
// Tests/Integration/CustomWebApplicationFactory.cs
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove the real DbContext
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.RemoveAll<AppDbContext>();

            // Add in-memory database
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("IntegrationTestDb"));

            // Replace external services with test doubles
            services.RemoveAll<IEmailService>();
            services.AddScoped<IEmailService, FakeEmailService>();
        });

        builder.UseEnvironment("Testing");
    }
}

// Tests/Integration/ProductsControllerTests.cs
public class ProductsControllerTests
    : IClassFixture<CustomWebApplicationFactory>, IAsyncLifetime
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory _factory;

    public ProductsControllerTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client  = factory.CreateClient();
    }

    // Seed / reset database before each test
    public async Task InitializeAsync()
    {
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.EnsureDeletedAsync();
        await db.Database.EnsureCreatedAsync();
        // seed test data
        db.Categories.Add(new Category { Id = 1, Name = "Electronics" });
        await db.SaveChangesAsync();
    }
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task GetAll_ReturnsOkWithEmptyList()
    {
        var response = await _client.GetAsync("/api/products");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var body = await response.Content.ReadFromJsonAsync<PagedResult<ProductDto>>();
        body!.Items.Should().BeEmpty();
    }

    [Fact]
    public async Task Create_WithValidData_Returns201AndLocation()
    {
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", GetTestToken("Admin"));

        var dto = new CreateProductDto
            { Name = "Test Laptop", Price = 999m, Stock = 5, CategoryId = 1 };

        var response = await _client.PostAsJsonAsync("/api/products", dto);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var created = await response.Content.ReadFromJsonAsync<ProductDto>();
        created!.Name.Should().Be("Test Laptop");
    }

    [Fact]
    public async Task Delete_WithoutAuth_Returns401()
    {
        var response = await _client.DeleteAsync("/api/products/1");
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task Delete_AsUser_Returns403()
    {
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", GetTestToken("User"));

        var response = await _client.DeleteAsync("/api/products/1");
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }

    private string GetTestToken(string role)
    {
        // Build a real JWT with the test configuration
        var config = _factory.Services.GetRequiredService<IConfiguration>();
        var tokenSvc = _factory.Services.GetRequiredService<ITokenService>();
        return tokenSvc.GenerateToken(1, "test@test.com", role);
    }
}
```

---

### :pencil: Section 7 — Exercises

1. Write integration tests for all CRUD endpoints of your `ProductsController`. Cover: happy path, not found, validation errors, and unauthorized access.
2. Write an integration test for `POST /api/auth/register` → `POST /api/auth/login` → `GET /api/auth/profile` as a full flow.
3. Add a test that verifies pagination: insert 25 products, request `?page=2&pageSize=10`, and verify exactly 10 items and the correct `X-Total-Count` header.

---

## Section 8 — Final Project <a name = "section-8"></a>

### 8.1 Products REST API — Project Structure

```
ProductsApi/
├── src/
│   └── ProductsApi/
│       ├── Controllers/
│       │   ├── ProductsController.cs
│       │   ├── CategoriesController.cs
│       │   └── AuthController.cs
│       ├── Models/
│       │   ├── Product.cs
│       │   ├── Category.cs
│       │   └── User.cs
│       ├── DTOs/
│       │   ├── Product/
│       │   │   ├── CreateProductDto.cs
│       │   │   ├── UpdateProductDto.cs
│       │   │   └── ProductDto.cs
│       │   ├── Auth/
│       │   │   ├── LoginDto.cs
│       │   │   ├── RegisterDto.cs
│       │   │   └── AuthResponseDto.cs
│       │   └── Common/
│       │       ├── PagedResult.cs
│       │       └── ProductFilterDto.cs
│       ├── Services/
│       │   ├── Interfaces/
│       │   └── Implementations/
│       ├── Repositories/
│       │   ├── Interfaces/
│       │   └── Implementations/
│       ├── Data/
│       │   ├── AppDbContext.cs
│       │   └── Migrations/
│       ├── Middleware/
│       │   └── GlobalExceptionMiddleware.cs
│       ├── Extensions/
│       │   └── ServiceCollectionExtensions.cs
│       ├── Validators/
│       ├── appsettings.json
│       ├── appsettings.Development.json
│       ├── Program.cs
│       └── ProductsApi.csproj
└── tests/
    └── ProductsApi.Tests/
        ├── Unit/Services/
        ├── Integration/Controllers/
        └── ProductsApi.Tests.csproj
```

---

### 8.2 Feature Checklist

**Core API:**
- [ ] Full CRUD for `Products` with EF Core + SQLite
- [ ] `Categories` with one-to-many relationship to `Products`
- [ ] Filtering: `name`, `minPrice`, `maxPrice`, `categoryId`, `isActive`
- [ ] Pagination with `page` / `pageSize` query params and metadata headers

**Quality:**
- [ ] FluentValidation for all request DTOs
- [ ] Global exception middleware returning consistent error JSON
- [ ] Structured logging with `ILogger<T>` on all key operations
- [ ] In-memory caching on `GET /api/products` with cache invalidation
- [ ] `CancellationToken` propagated through all async methods

**Security:**
- [ ] `POST /api/auth/register` — BCrypt hashing, uniqueness check
- [ ] `POST /api/auth/login` — returns JWT with `Id`, `Email`, `Role`
- [ ] `GET` endpoints are `[AllowAnonymous]`
- [ ] `POST` / `PUT` require `[Authorize]`
- [ ] `DELETE` requires `[Authorize(Roles = "Admin")]`

**Documentation:**
- [ ] Swagger UI with XML doc comments on all endpoints
- [ ] JWT Bearer auth configured in Swagger UI

**Tests:**
- [ ] Unit tests for all service methods (xUnit + Moq + FluentAssertions)
- [ ] Integration tests for all controller endpoints (WebApplicationFactory)
- [ ] ≥ 80% code coverage

---

### 8.3 API Endpoints Summary

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/auth/register` | Public | Register new user |
| `POST` | `/api/auth/login` | Public | Login → JWT |
| `GET` | `/api/auth/profile` | Auth | Current user info |
| `GET` | `/api/products` | Public | List (filterable + paginated) |
| `GET` | `/api/products/{id}` | Public | Get by ID |
| `POST` | `/api/products` | Auth | Create product |
| `PUT` | `/api/products/{id}` | Auth | Update product |
| `DELETE` | `/api/products/{id}` | Admin | Delete product |
| `GET` | `/api/categories` | Public | List categories |
| `GET` | `/api/categories/{id}` | Public | Get category by ID |
| `POST` | `/api/categories` | Admin | Create category |
| `GET` | `/api/categories/{id}/products` | Public | Products in a category |

---

## :books: References <a name = "references"></a>

| Resource | Link |
|----------|------|
| ASP.NET Core Documentation | [docs.microsoft.com/aspnet/core](https://docs.microsoft.com/en-us/aspnet/core) |
| ASP.NET Core Developer Roadmap | [roadmap.sh/aspnet-core](https://roadmap.sh/aspnet-core) |
| .NET and C# Beginners Roadmap | [roadmap.sh/ai/course](https://roadmap.sh/ai/course/net-and-c-for-beginners-a-comprehensive-guide) |
| Entity Framework Core | [docs.microsoft.com/ef/core](https://docs.microsoft.com/en-us/ef/core/) |
| ASP.NET Core Roadmap (MoienTajik) | [github.com/MoienTajik/AspNetCore-Developer-Roadmap](https://github.com/MoienTajik/AspNetCore-Developer-Roadmap) |
| JWT.io | [jwt.io](https://jwt.io/) |
| FluentValidation | [docs.fluentvalidation.net](https://docs.fluentvalidation.net/) |
| AutoMapper | [automapper.org](https://automapper.org/) |
| Polly (Resilience) | [thepollyproject.org](https://thepollyproject.org/) |
| xUnit Testing | [xunit.net](https://xunit.net/) |
| FluentAssertions | [fluentassertions.com](https://fluentassertions.com/) |

---

## :page_with_curl: License

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

This project is licensed under the MIT License.

---

<p align="center">Made with ❤️ for ASP.NET Core developers worldwide</p>
