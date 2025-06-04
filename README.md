# MyWebAPI Project Setup Guide

*Complete step-by-step documentation of .NET 8.0 Web API with Entity Framework Core setup*

---

## 📋 Project Information

- **Project Name**: MyWebAPI
- **Framework**: .NET 8.0
- **Database**: SQL Server (Docker container)
- **ORM**: Entity Framework Core
- **Date Created**: June 4, 2025
- **Location**: `/Users/dhairya/RiderProjects/MyWebAPI`

---

## 🚀 Initial Project Setup

### 1. Create New Web API Project
```bash
cd RiderProjects/
dotnet new webapi -n MyWebAPI --framework net8.0
cd MyWebAPI
```

### 2. Add Entity Framework Packages
```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
```

### 3. Build Project
```bash
dotnet build
```

---

## ⚡ Entity Framework Tools Setup (Critical Issue Resolution)

### Problem Encountered
Initial attempts to use global EF tools failed due to version compatibility issues.

### Failed Attempts
```bash
# These commands failed:
dotnet tool install --global dotnet-ef
dotnet ef migrations add InitialCreate  # ❌ Failed
dotnet ef --version
dotnet tool uninstall --global dotnet-ef
dotnet tool install --global dotnet-ef --version 8.0.11
dotnet ef migrations add InitialCreate  # ❌ Still failed
export PATH="/Users/dhairya/.dotnet:$PATH" && dotnet ef migrations add InitialCreate  # ❌ Still failed
```

### ✅ Solution: Local EF Tools
```bash
# Create local tool manifest
dotnet new tool-manifest

# Install EF tools locally
dotnet tool install dotnet-ef

# Use local EF tools with dotnet dotnet-ef syntax
dotnet dotnet-ef migrations add InitialCreate  # ✅ Success!
dotnet dotnet-ef database update  # ✅ Success!
```

**Key Learning**: When global EF tools fail, use local tools with `dotnet dotnet-ef` syntax.

---

## 📁 Project Structure Created

```
MyWebAPI/
├── Controllers/
│   ├── ProductsController.cs      # Full CRUD API controller
│   └── WeatherForecastController.cs  # Default template
├── Data/
│   └── ApplicationDbContext.cs    # EF DbContext
├── Models/
│   ├── Product.cs                 # Product entity
│   └── WeatherForecast.cs         # Default template
├── Migrations/
│   ├── 20250604133317_InitialCreate.cs
│   ├── 20250604133317_InitialCreate.Designer.cs
│   └── ApplicationDbContextModelSnapshot.cs
├── Properties/
│   └── launchSettings.json
├── .config/
│   └── dotnet-tools.json          # Local tools manifest
├── appsettings.json               # Modified with connection string
├── appsettings.Development.json
├── Program.cs                     # Modified with EF and CORS
├── MyWebAPI.csproj
└── MyWebAPI.sln                   # Solution file
```

---

## 🗃️ Database Configuration

### Connection String (appsettings.json)
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1433;Database=WebApi;User Id=sa;Password=dhairya@3391;TrustServerCertificate=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### Entity Model (Models/Product.cs)
```csharp
using System.ComponentModel.DataAnnotations;

namespace MyWebAPI.Models;

public class Product
{
    public int Id { get; set; }
    
    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = string.Empty;
    
    [MaxLength(500)]
    public string? Description { get; set; }
    
    [Range(0.01, double.MaxValue)]
    public decimal Price { get; set; }
    
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public DateTime? UpdatedAt { get; set; }
}
```

### DbContext (Data/ApplicationDbContext.cs)
```csharp
using Microsoft.EntityFrameworkCore;
using MyWebAPI.Models;

namespace MyWebAPI.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options)
    {
    }
    
    public DbSet<Product> Products { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Configure Product entity
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(100);
            entity.Property(e => e.Description).HasMaxLength(500);
            entity.Property(e => e.Price).HasColumnType("decimal(18,2)");
            entity.Property(e => e.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
        });
    }
}
```

---

## 🛠️ Service Configuration (Program.cs)

```csharp
using Microsoft.EntityFrameworkCore;
using MyWebAPI.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Entity Framework Configuration
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// CORS Configuration for Development
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
            .AllowAnyHeader()
            .AllowAnyMethod();
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseCors("AllowAll");
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## 🎯 API Controller (Controllers/ProductsController.cs)

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MyWebAPI.Data;
using MyWebAPI.Models;

namespace MyWebAPI.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ApplicationDbContext _context;

    public ProductsController(ApplicationDbContext context)
    {
        _context = context;
    }

    // GET: api/Products
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
    {
        return await _context.Products.ToListAsync();
    }

    // GET: api/Products/5
    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetProduct(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null)
        {
            return NotFound();
        }
        return product;
    }

    // POST: api/Products
    [HttpPost]
    public async Task<ActionResult<Product>> PostProduct(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        return CreatedAtAction("GetProduct", new { id = product.Id }, product);
    }

    // PUT: api/Products/5
    [HttpPut("{id}")]
    public async Task<IActionResult> PutProduct(int id, Product product)
    {
        if (id != product.Id)
        {
            return BadRequest();
        }

        product.UpdatedAt = DateTime.UtcNow;
        _context.Entry(product).State = EntityState.Modified;

        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!ProductExists(id))
            {
                return NotFound();
            }
            throw;
        }
        return NoContent();
    }

    // DELETE: api/Products/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null)
        {
            return NotFound();
        }

        _context.Products.Remove(product);
        await _context.SaveChangesAsync();
        return NoContent();
    }

    private bool ProductExists(int id)
    {
        return _context.Products.Any(e => e.Id == id);
    }
}
```

---

## 🗄️ Database Migration Commands

```bash
# Create migration (using local EF tools)
dotnet dotnet-ef migrations add InitialCreate

# Apply migration to database
dotnet dotnet-ef database update

# Verify migration files created
ls -la Migrations/
```

---

## 📦 Solution File Setup

```bash
# Create solution file
dotnet new sln --name MyWebAPI

# Add project to solution
dotnet sln add MyWebAPI.csproj

# Verify solution
ls -la *.sln
```

---

## 🚀 Running the Project

```bash
# Run the application
dotnet run
```

**Access URLs:**
- API: `https://localhost:7101` or `http://localhost:5000`
- Swagger UI: `https://localhost:7101/swagger`

---

## 🌐 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/Products` | Get all products |
| GET | `/api/Products/{id}` | Get product by ID |
| POST | `/api/Products` | Create new product |
| PUT | `/api/Products/{id}` | Update existing product |
| DELETE | `/api/Products/{id}` | Delete product |

### Sample Product JSON
```json
{
  "name": "Sample Product",
  "description": "A sample product description",
  "price": 29.99
}
```

---

## 🔧 Key Troubleshooting Notes

### ❌ Global EF Tools Issue
**Problem**: `dotnet ef` commands failed even after global installation

**Root Cause**: Version compatibility issues between global EF tools and project

**Solution**: Use local EF tools instead
```bash
dotnet new tool-manifest
dotnet tool install dotnet-ef
dotnet dotnet-ef [command]  # Note the double 'dotnet'
```

### ✅ Local Tools Configuration
The `.config/dotnet-tools.json` file ensures consistent EF tool versions:
```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-ef": {
      "version": "8.0.11",
      "commands": ["dotnet-ef"]
    }
  }
}
```

---

## ✅ Project Status

- [x] .NET 8.0 Web API project created
- [x] Entity Framework Core with SQL Server configured
- [x] Product model with validation attributes
- [x] ApplicationDbContext with proper configuration
- [x] Full CRUD ProductsController implemented
- [x] Database migration created and applied
- [x] CORS configured for development
- [x] Swagger/OpenAPI documentation enabled
- [x] Local EF tools configured and working
- [x] Solution file created and configured
- [x] Project builds and runs successfully

---

## 📚 Next Steps

1. **Authentication & Authorization**: Add JWT authentication
2. **Validation**: Implement more robust model validation
3. **Logging**: Add structured logging with Serilog
4. **Testing**: Create unit and integration tests
5. **Docker**: Containerize the application
6. **CI/CD**: Set up GitHub Actions or Azure DevOps

---

*This guide documents the complete setup process, including the critical resolution of EF tools issues using local tooling instead of global installation.*

