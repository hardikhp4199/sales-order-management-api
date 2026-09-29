# Sales Order API

A production-minded ASP.NET Core Web API sample using **.NET 10**, **Entity Framework Core 10**, and **SQL Server**. The project demonstrates a multi-table sales-order workflow with asynchronous CRUD operations, DTO validation, transactions, database constraints, global error handling, OpenAPI documentation, and a maintainable feature-based structure.

> **Framework choice:** Use .NET 10 for a new repository. It is an LTS release, and EF Core 10 requires the .NET 10 SDK/runtime. If your hosting environment is currently standardised on .NET 8, change `net10.0` to `net8.0` and use matching 8.x EF Core packages.

## Business scenario

A sales order belongs to one customer and contains one or more order items. Each item references a product and stores the sale-time unit price. Creating or updating an order recalculates its total inside a database transaction.

## Technology

- .NET 10 / C# 14
- ASP.NET Core controller-based Web API
- Entity Framework Core 10
- SQL Server
- OpenAPI
- xUnit-ready test structure

# DotNet Sales Order API

Production-ready ASP.NET Core .NET 10 Web API using SQL Server and Entity Framework Core.

## Database Relationship

Customer
    |
    | 1:N
    |
SalesOrder
    |
    | 1:N
    |
SalesOrderItem
    |
    | N:1
    |
Product

## Features

- .NET 10 Web API
- SQL Server
- Entity Framework Core 10
- Multi-table CRUD Operations
- Validation
- Transactions
- Concurrency Handling
- Health Checks
- Swagger / OpenAPI
- Global Exception Handling
- Production-Oriented Structure

## Solution structure

```text
SalesOrderApi/
├── database/
│   └── 001-create-sales-order-db.sql
├── src/
│   └── SalesOrder.Api/
│       ├── Common/
│       │   └── GlobalExceptionHandler.cs
│       ├── Controllers/
│       │   ├── ProductsController.cs
│       │   └── SalesOrdersController.cs
│       ├── Data/
│       │   ├── Configurations/
│       │   │   ├── CustomerConfiguration.cs
│       │   │   ├── ProductConfiguration.cs
│       │   │   ├── SalesOrderConfiguration.cs
│       │   │   └── SalesOrderItemConfiguration.cs
│       │   └── SalesDbContext.cs
│       ├── Features/
│       │   ├── Products/
│       │   │   ├── ProductDtos.cs
│       │   │   └── ProductService.cs
│       │   └── SalesOrders/
│       │       ├── SalesOrderDtos.cs
│       │       └── SalesOrderService.cs
│       ├── Models/
│       │   ├── Customer.cs
│       │   ├── Product.cs
│       │   ├── SalesOrder.cs
│       │   └── SalesOrderItem.cs
│       ├── Properties/
│       │   └── launchSettings.json
│       ├── Program.cs
│       ├── appsettings.json
│       └── SalesOrder.Api.csproj
├── tests/
│   └── SalesOrder.Api.Tests/
├── .gitignore
├── SalesOrderApi.sln
└── README.md
```

This deliberately uses one deployable API project with feature folders. It avoids premature multi-project complexity while preserving clear boundaries between HTTP, business logic, persistence, and domain models. Split into separate projects only when independently reusable boundaries genuinely emerge.

# 1. Database details

## Database name

`SalesOrderDb`

## Tables

| Table | Purpose | Main relationships |
|---|---|---|
| `Customers` | Customer master data | One customer has many sales orders |
| `Products` | Product catalogue and stock | One product can appear in many order items |
| `SalesOrders` | Order header | Belongs to one customer |
| `SalesOrderItems` | Order lines | Belongs to one order and one product |

## SQL Server script

Create `database/001-create-sales-order-db.sql`:

```sql
IF DB_ID(N'SalesOrderDb') IS NULL
BEGIN
    CREATE DATABASE SalesOrderDb;
END;
GO

USE SalesOrderDb;
GO

CREATE TABLE dbo.Customers
(
    CustomerId       INT IDENTITY(1,1) NOT NULL,
    CustomerCode     NVARCHAR(20) NOT NULL,
    Name             NVARCHAR(150) NOT NULL,
    Email            NVARCHAR(255) NULL,
    IsActive         BIT NOT NULL CONSTRAINT DF_Customers_IsActive DEFAULT (1),
    CreatedAtUtc     DATETIME2(0) NOT NULL CONSTRAINT DF_Customers_CreatedAtUtc DEFAULT (SYSUTCDATETIME()),
    CONSTRAINT PK_Customers PRIMARY KEY (CustomerId),
    CONSTRAINT UQ_Customers_CustomerCode UNIQUE (CustomerCode)
);
GO

CREATE UNIQUE INDEX UX_Customers_Email
ON dbo.Customers(Email)
WHERE Email IS NOT NULL;
GO

CREATE TABLE dbo.Products
(
    ProductId        INT IDENTITY(1,1) NOT NULL,
    Sku              NVARCHAR(40) NOT NULL,
    Name             NVARCHAR(150) NOT NULL,
    UnitPrice        DECIMAL(18,2) NOT NULL,
    StockQuantity    INT NOT NULL,
    IsActive         BIT NOT NULL CONSTRAINT DF_Products_IsActive DEFAULT (1),
    CreatedAtUtc     DATETIME2(0) NOT NULL CONSTRAINT DF_Products_CreatedAtUtc DEFAULT (SYSUTCDATETIME()),
    UpdatedAtUtc     DATETIME2(0) NULL,
    RowVersion       ROWVERSION NOT NULL,
    CONSTRAINT PK_Products PRIMARY KEY (ProductId),
    CONSTRAINT UQ_Products_Sku UNIQUE (Sku),
    CONSTRAINT CK_Products_UnitPrice CHECK (UnitPrice >= 0),
    CONSTRAINT CK_Products_StockQuantity CHECK (StockQuantity >= 0)
);
GO

CREATE TABLE dbo.SalesOrders
(
    SalesOrderId     BIGINT IDENTITY(1,1) NOT NULL,
    OrderNumber      NVARCHAR(30) NOT NULL,
    CustomerId       INT NOT NULL,
    OrderDateUtc     DATETIME2(0) NOT NULL,
    Status           NVARCHAR(20) NOT NULL,
    TotalAmount      DECIMAL(18,2) NOT NULL,
    CreatedAtUtc     DATETIME2(0) NOT NULL CONSTRAINT DF_SalesOrders_CreatedAtUtc DEFAULT (SYSUTCDATETIME()),
    UpdatedAtUtc     DATETIME2(0) NULL,
    RowVersion       ROWVERSION NOT NULL,
    CONSTRAINT PK_SalesOrders PRIMARY KEY (SalesOrderId),
    CONSTRAINT UQ_SalesOrders_OrderNumber UNIQUE (OrderNumber),
    CONSTRAINT FK_SalesOrders_Customers FOREIGN KEY (CustomerId)
        REFERENCES dbo.Customers(CustomerId),
    CONSTRAINT CK_SalesOrders_Status CHECK (Status IN ('Draft','Confirmed','Cancelled')),
    CONSTRAINT CK_SalesOrders_TotalAmount CHECK (TotalAmount >= 0)
);
GO

CREATE INDEX IX_SalesOrders_CustomerId_OrderDateUtc
ON dbo.SalesOrders(CustomerId, OrderDateUtc DESC);
GO

CREATE TABLE dbo.SalesOrderItems
(
    SalesOrderItemId BIGINT IDENTITY(1,1) NOT NULL,
    SalesOrderId     BIGINT NOT NULL,
    ProductId        INT NOT NULL,
    Quantity         INT NOT NULL,
    UnitPrice        DECIMAL(18,2) NOT NULL,
    LineTotal AS CONVERT(DECIMAL(18,2), Quantity * UnitPrice) PERSISTED,
    CONSTRAINT PK_SalesOrderItems PRIMARY KEY (SalesOrderItemId),
    CONSTRAINT FK_SalesOrderItems_SalesOrders FOREIGN KEY (SalesOrderId)
        REFERENCES dbo.SalesOrders(SalesOrderId) ON DELETE CASCADE,
    CONSTRAINT FK_SalesOrderItems_Products FOREIGN KEY (ProductId)
        REFERENCES dbo.Products(ProductId),
    CONSTRAINT UQ_SalesOrderItems_Order_Product UNIQUE (SalesOrderId, ProductId),
    CONSTRAINT CK_SalesOrderItems_Quantity CHECK (Quantity > 0),
    CONSTRAINT CK_SalesOrderItems_UnitPrice CHECK (UnitPrice >= 0)
);
GO

CREATE INDEX IX_SalesOrderItems_ProductId
ON dbo.SalesOrderItems(ProductId);
GO

INSERT INTO dbo.Customers (CustomerCode, Name, Email)
VALUES ('CUST-001', 'Northwind Retail Ltd', 'orders@northwind.example');

INSERT INTO dbo.Products (Sku, Name, UnitPrice, StockQuantity)
VALUES
('LAP-001', 'Business Laptop', 899.99, 25),
('MON-001', '27-inch Monitor', 229.50, 40),
('KEY-001', 'Mechanical Keyboard', 79.99, 60);
GO
```

# 2. Project setup

## Prerequisites

- .NET 10 SDK
- SQL Server 2022 or later, SQL Server Developer, or a compatible Azure SQL database
- SQL Server Management Studio or Azure Data Studio
- Git
- Visual Studio 2022 or Visual Studio Code with C# tooling

Verify the SDK:

```bash
dotnet --info
dotnet --list-sdks
```

## Create the solution

```bash
mkdir SalesOrderApi
cd SalesOrderApi

dotnet new sln -n SalesOrderApi
mkdir src tests database

dotnet new webapi --use-controllers -n SalesOrder.Api -o src/SalesOrder.Api -f net10.0
dotnet new xunit -n SalesOrder.Api.Tests -o tests/SalesOrder.Api.Tests -f net10.0

dotnet sln add src/SalesOrder.Api/SalesOrder.Api.csproj
dotnet sln add tests/SalesOrder.Api.Tests/SalesOrder.Api.Tests.csproj
dotnet add tests/SalesOrder.Api.Tests/SalesOrder.Api.Tests.csproj reference src/SalesOrder.Api/SalesOrder.Api.csproj
```

## Install packages

```bash
dotnet add src/SalesOrder.Api package Microsoft.EntityFrameworkCore.SqlServer --version 10.*
dotnet add src/SalesOrder.Api package Microsoft.EntityFrameworkCore.Design --version 10.*
dotnet add src/SalesOrder.Api package Microsoft.AspNetCore.OpenApi --version 10.*

dotnet tool install --global dotnet-ef --version 10.*
```

For .NET 8, use `-f net8.0` and matching `8.*` package/tool versions. Do not mix EF Core major versions.

## Project file

`src/SalesOrder.Api/SalesOrder.Api.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <InvariantGlobalization>false</InvariantGlobalization>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.*" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.*">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="10.*" />
  </ItemGroup>
</Project>
```

> For reproducible production builds, replace floating `10.*` versions with the exact approved versions after the first restore.

## Connection string

`src/SalesOrder.Api/appsettings.json`

```json
{
  "ConnectionStrings": {
    "SalesDb": "Server=localhost;Database=SalesOrderDb;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=False"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

Do not commit production credentials. Use user secrets locally:

```bash
cd src/SalesOrder.Api
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:SalesDb" "Server=localhost;Database=SalesOrderDb;User Id=YOUR_USER;Password=YOUR_PASSWORD;TrustServerCertificate=True"
cd ../..
```

# 3. Application code

## Domain models

`Models/Customer.cs`

```csharp
namespace SalesOrder.Api.Models;

public sealed class Customer
{
    public int CustomerId { get; set; }
    public required string CustomerCode { get; set; }
    public required string Name { get; set; }
    public string? Email { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAtUtc { get; set; }
    public ICollection<SalesOrder> SalesOrders { get; set; } = [];
}
```

`Models/Product.cs`

```csharp
namespace SalesOrder.Api.Models;

public sealed class Product
{
    public int ProductId { get; set; }
    public required string Sku { get; set; }
    public required string Name { get; set; }
    public decimal UnitPrice { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
    public byte[] RowVersion { get; set; } = [];
    public ICollection<SalesOrderItem> SalesOrderItems { get; set; } = [];
}
```

`Models/SalesOrder.cs`

```csharp
namespace SalesOrder.Api.Models;

public sealed class SalesOrder
{
    public long SalesOrderId { get; set; }
    public required string OrderNumber { get; set; }
    public int CustomerId { get; set; }
    public DateTime OrderDateUtc { get; set; }
    public required string Status { get; set; }
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAtUtc { get; set; }
    public DateTime? UpdatedAtUtc { get; set; }
    public byte[] RowVersion { get; set; } = [];
    public Customer Customer { get; set; } = null!;
    public ICollection<SalesOrderItem> Items { get; set; } = [];
}
```

`Models/SalesOrderItem.cs`

```csharp
namespace SalesOrder.Api.Models;

public sealed class SalesOrderItem
{
    public long SalesOrderItemId { get; set; }
    public long SalesOrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal LineTotal { get; private set; }
    public SalesOrder SalesOrder { get; set; } = null!;
    public Product Product { get; set; } = null!;
}
```

## EF Core context

`Data/SalesDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using SalesOrder.Api.Models;

namespace SalesOrder.Api.Data;

public sealed class SalesDbContext(DbContextOptions<SalesDbContext> options)
    : DbContext(options)
{
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<SalesOrder.Api.Models.SalesOrder> SalesOrders => Set<SalesOrder.Api.Models.SalesOrder>();
    public DbSet<SalesOrderItem> SalesOrderItems => Set<SalesOrderItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
        => modelBuilder.ApplyConfigurationsFromAssembly(typeof(SalesDbContext).Assembly);
}
```

## Entity configurations

`Data/Configurations/CustomerConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using SalesOrder.Api.Models;

namespace SalesOrder.Api.Data.Configurations;

public sealed class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("Customers");
        builder.HasKey(x => x.CustomerId);
        builder.Property(x => x.CustomerCode).HasMaxLength(20).IsRequired();
        builder.Property(x => x.Name).HasMaxLength(150).IsRequired();
        builder.Property(x => x.Email).HasMaxLength(255);
        builder.HasIndex(x => x.CustomerCode).IsUnique();
    }
}
```

`Data/Configurations/ProductConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using SalesOrder.Api.Models;

namespace SalesOrder.Api.Data.Configurations;

public sealed class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(x => x.ProductId);
        builder.Property(x => x.Sku).HasMaxLength(40).IsRequired();
        builder.Property(x => x.Name).HasMaxLength(150).IsRequired();
        builder.Property(x => x.UnitPrice).HasPrecision(18, 2);
        builder.Property(x => x.RowVersion).IsRowVersion();
        builder.HasIndex(x => x.Sku).IsUnique();
    }
}
```

`Data/Configurations/SalesOrderConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using SalesOrder.Api.Models;

namespace SalesOrder.Api.Data.Configurations;

public sealed class SalesOrderConfiguration : IEntityTypeConfiguration<SalesOrder.Api.Models.SalesOrder>
{
    public void Configure(EntityTypeBuilder<SalesOrder.Api.Models.SalesOrder> builder)
    {
        builder.ToTable("SalesOrders");
        builder.HasKey(x => x.SalesOrderId);
        builder.Property(x => x.OrderNumber).HasMaxLength(30).IsRequired();
        builder.Property(x => x.Status).HasMaxLength(20).IsRequired();
        builder.Property(x => x.TotalAmount).HasPrecision(18, 2);
        builder.Property(x => x.RowVersion).IsRowVersion();
        builder.HasIndex(x => x.OrderNumber).IsUnique();
        builder.HasOne(x => x.Customer)
            .WithMany(x => x.SalesOrders)
            .HasForeignKey(x => x.CustomerId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

`Data/Configurations/SalesOrderItemConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using SalesOrder.Api.Models;

namespace SalesOrder.Api.Data.Configurations;

public sealed class SalesOrderItemConfiguration : IEntityTypeConfiguration<SalesOrderItem>
{
    public void Configure(EntityTypeBuilder<SalesOrderItem> builder)
    {
        builder.ToTable("SalesOrderItems");
        builder.HasKey(x => x.SalesOrderItemId);
        builder.Property(x => x.UnitPrice).HasPrecision(18, 2);
        builder.Property(x => x.LineTotal)
            .HasComputedColumnSql("CONVERT(decimal(18,2), [Quantity] * [UnitPrice])", stored: true);
        builder.HasIndex(x => new { x.SalesOrderId, x.ProductId }).IsUnique();
        builder.HasOne(x => x.SalesOrder)
            .WithMany(x => x.Items)
            .HasForeignKey(x => x.SalesOrderId)
            .OnDelete(DeleteBehavior.Cascade);
        builder.HasOne(x => x.Product)
            .WithMany(x => x.SalesOrderItems)
            .HasForeignKey(x => x.ProductId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

## DTOs

`Features/Products/ProductDtos.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace SalesOrder.Api.Features.Products;

public sealed record ProductResponse(
    int ProductId, string Sku, string Name, decimal UnitPrice,
    int StockQuantity, bool IsActive, string RowVersion);

public sealed class UpsertProductRequest
{
    [Required, StringLength(40)]
    public string Sku { get; init; } = string.Empty;

    [Required, StringLength(150)]
    public string Name { get; init; } = string.Empty;

    [Range(0, 9999999999999999.99)]
    public decimal UnitPrice { get; init; }

    [Range(0, int.MaxValue)]
    public int StockQuantity { get; init; }

    public bool IsActive { get; init; } = true;

    public string? RowVersion { get; init; }
}
```

`Features/SalesOrders/SalesOrderDtos.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace SalesOrder.Api.Features.SalesOrders;

public sealed class SalesOrderLineRequest
{
    [Range(1, int.MaxValue)]
    public int ProductId { get; init; }

    [Range(1, 100000)]
    public int Quantity { get; init; }
}

public sealed class CreateSalesOrderRequest
{
    [Range(1, int.MaxValue)]
    public int CustomerId { get; init; }

    [MinLength(1)]
    public List<SalesOrderLineRequest> Items { get; init; } = [];
}

public sealed class UpdateSalesOrderRequest
{
    [Range(1, int.MaxValue)]
    public int CustomerId { get; init; }

    [Required]
    public string RowVersion { get; init; } = string.Empty;

    [MinLength(1)]
    public List<SalesOrderLineRequest> Items { get; init; } = [];
}

public sealed record SalesOrderItemResponse(
    long SalesOrderItemId, int ProductId, string Sku, string ProductName,
    int Quantity, decimal UnitPrice, decimal LineTotal);

public sealed record SalesOrderResponse(
    long SalesOrderId, string OrderNumber, int CustomerId, string CustomerName,
    DateTime OrderDateUtc, string Status, decimal TotalAmount,
    string RowVersion, IReadOnlyCollection<SalesOrderItemResponse> Items);
```

## Product service

`Features/Products/ProductService.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using SalesOrder.Api.Data;
using SalesOrder.Api.Models;

namespace SalesOrder.Api.Features.Products;

public interface IProductService
{
    Task<IReadOnlyList<ProductResponse>> GetAllAsync(CancellationToken ct);
    Task<ProductResponse?> GetByIdAsync(int id, CancellationToken ct);
    Task<ProductResponse> CreateAsync(UpsertProductRequest request, CancellationToken ct);
    Task<ProductResponse?> UpdateAsync(int id, UpsertProductRequest request, CancellationToken ct);
    Task<bool> DeleteAsync(int id, CancellationToken ct);
}

public sealed class ProductService(SalesDbContext db) : IProductService
{
    public async Task<IReadOnlyList<ProductResponse>> GetAllAsync(CancellationToken ct) =>
        await db.Products.AsNoTracking().OrderBy(x => x.Name)
            .Select(x => Map(x)).ToListAsync(ct);

    public async Task<ProductResponse?> GetByIdAsync(int id, CancellationToken ct)
    {
        var product = await db.Products.AsNoTracking()
            .SingleOrDefaultAsync(x => x.ProductId == id, ct);
        return product is null ? null : Map(product);
    }

    public async Task<ProductResponse> CreateAsync(UpsertProductRequest request, CancellationToken ct)
    {
        var sku = request.Sku.Trim().ToUpperInvariant();
        if (await db.Products.AnyAsync(x => x.Sku == sku, ct))
            throw new InvalidOperationException($"Product SKU '{sku}' already exists.");

        var product = new Product
        {
            Sku = sku,
            Name = request.Name.Trim(),
            UnitPrice = request.UnitPrice,
            StockQuantity = request.StockQuantity,
            IsActive = request.IsActive,
            CreatedAtUtc = DateTime.UtcNow
        };

        db.Products.Add(product);
        await db.SaveChangesAsync(ct);
        return Map(product);
    }

    public async Task<ProductResponse?> UpdateAsync(int id, UpsertProductRequest request, CancellationToken ct)
    {
        var product = await db.Products.SingleOrDefaultAsync(x => x.ProductId == id, ct);
        if (product is null) return null;

        if (string.IsNullOrWhiteSpace(request.RowVersion))
            throw new ArgumentException("RowVersion is required when updating a product.");

        db.Entry(product).Property(x => x.RowVersion).OriginalValue =
            Convert.FromBase64String(request.RowVersion);

        product.Sku = request.Sku.Trim().ToUpperInvariant();
        product.Name = request.Name.Trim();
        product.UnitPrice = request.UnitPrice;
        product.StockQuantity = request.StockQuantity;
        product.IsActive = request.IsActive;
        product.UpdatedAtUtc = DateTime.UtcNow;

        await db.SaveChangesAsync(ct);
        return Map(product);
    }

    public async Task<bool> DeleteAsync(int id, CancellationToken ct)
    {
        var product = await db.Products.SingleOrDefaultAsync(x => x.ProductId == id, ct);
        if (product is null) return false;

        var used = await db.SalesOrderItems.AnyAsync(x => x.ProductId == id, ct);
        if (used)
            throw new InvalidOperationException("A product used by an order cannot be deleted; deactivate it instead.");

        db.Products.Remove(product);
        await db.SaveChangesAsync(ct);
        return true;
    }

    private static ProductResponse Map(Product x) => new(
        x.ProductId, x.Sku, x.Name, x.UnitPrice, x.StockQuantity,
        x.IsActive, Convert.ToBase64String(x.RowVersion));
}
```

## Sales order service

`Features/SalesOrders/SalesOrderService.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using SalesOrder.Api.Data;
using SalesOrder.Api.Models;
using OrderEntity = SalesOrder.Api.Models.SalesOrder;

namespace SalesOrder.Api.Features.SalesOrders;

public interface ISalesOrderService
{
    Task<IReadOnlyList<SalesOrderResponse>> GetAllAsync(CancellationToken ct);
    Task<SalesOrderResponse?> GetByIdAsync(long id, CancellationToken ct);
    Task<SalesOrderResponse> CreateAsync(CreateSalesOrderRequest request, CancellationToken ct);
    Task<SalesOrderResponse?> UpdateAsync(long id, UpdateSalesOrderRequest request, CancellationToken ct);
    Task<bool> DeleteDraftAsync(long id, string rowVersion, CancellationToken ct);
}

public sealed class SalesOrderService(SalesDbContext db) : ISalesOrderService
{
    public async Task<IReadOnlyList<SalesOrderResponse>> GetAllAsync(CancellationToken ct)
    {
        var orders = await BaseQuery().OrderByDescending(x => x.OrderDateUtc).ToListAsync(ct);
        return orders.Select(Map).ToList();
    }

    public async Task<SalesOrderResponse?> GetByIdAsync(long id, CancellationToken ct)
    {
        var order = await BaseQuery().SingleOrDefaultAsync(x => x.SalesOrderId == id, ct);
        return order is null ? null : Map(order);
    }

    public async Task<SalesOrderResponse> CreateAsync(CreateSalesOrderRequest request, CancellationToken ct)
    {
        ValidateNoDuplicateProducts(request.Items);
        await using var transaction = await db.Database.BeginTransactionAsync(ct);

        var customerExists = await db.Customers
            .AnyAsync(x => x.CustomerId == request.CustomerId && x.IsActive, ct);
        if (!customerExists) throw new ArgumentException("Active customer was not found.");

        var products = await LoadProductsAsync(request.Items.Select(x => x.ProductId), ct);
        ValidateProducts(request.Items, products);

        var order = new OrderEntity
        {
            OrderNumber = $"SO-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid():N}"[..20].ToUpperInvariant(),
            CustomerId = request.CustomerId,
            OrderDateUtc = DateTime.UtcNow,
            Status = "Draft",
            CreatedAtUtc = DateTime.UtcNow,
            Items = request.Items.Select(line =>
            {
                var product = products[line.ProductId];
                return new SalesOrderItem
                {
                    ProductId = line.ProductId,
                    Quantity = line.Quantity,
                    UnitPrice = product.UnitPrice
                };
            }).ToList()
        };

        order.TotalAmount = order.Items.Sum(x => x.Quantity * x.UnitPrice);
        db.SalesOrders.Add(order);
        await db.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct);

        return (await GetByIdAsync(order.SalesOrderId, ct))!;
    }

    public async Task<SalesOrderResponse?> UpdateAsync(
        long id, UpdateSalesOrderRequest request, CancellationToken ct)
    {
        ValidateNoDuplicateProducts(request.Items);
        await using var transaction = await db.Database.BeginTransactionAsync(ct);

        var order = await db.SalesOrders.Include(x => x.Items)
            .SingleOrDefaultAsync(x => x.SalesOrderId == id, ct);
        if (order is null) return null;
        if (order.Status != "Draft")
            throw new InvalidOperationException("Only draft orders can be updated.");

        db.Entry(order).Property(x => x.RowVersion).OriginalValue =
            Convert.FromBase64String(request.RowVersion);

        var customerExists = await db.Customers
            .AnyAsync(x => x.CustomerId == request.CustomerId && x.IsActive, ct);
        if (!customerExists) throw new ArgumentException("Active customer was not found.");

        var products = await LoadProductsAsync(request.Items.Select(x => x.ProductId), ct);
        ValidateProducts(request.Items, products);

        db.SalesOrderItems.RemoveRange(order.Items);
        order.CustomerId = request.CustomerId;
        order.Items = request.Items.Select(line => new SalesOrderItem
        {
            ProductId = line.ProductId,
            Quantity = line.Quantity,
            UnitPrice = products[line.ProductId].UnitPrice
        }).ToList();
        order.TotalAmount = order.Items.Sum(x => x.Quantity * x.UnitPrice);
        order.UpdatedAtUtc = DateTime.UtcNow;

        await db.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct);
        return (await GetByIdAsync(id, ct))!;
    }

    public async Task<bool> DeleteDraftAsync(long id, string rowVersion, CancellationToken ct)
    {
        var order = await db.SalesOrders.SingleOrDefaultAsync(x => x.SalesOrderId == id, ct);
        if (order is null) return false;
        if (order.Status != "Draft")
            throw new InvalidOperationException("Only draft orders can be deleted.");

        db.Entry(order).Property(x => x.RowVersion).OriginalValue = Convert.FromBase64String(rowVersion);
        db.SalesOrders.Remove(order);
        await db.SaveChangesAsync(ct);
        return true;
    }

    private IQueryable<OrderEntity> BaseQuery() => db.SalesOrders.AsNoTracking()
        .Include(x => x.Customer)
        .Include(x => x.Items).ThenInclude(x => x.Product)
        .AsSplitQuery();

    private async Task<Dictionary<int, Product>> LoadProductsAsync(
        IEnumerable<int> productIds, CancellationToken ct) =>
        await db.Products.Where(x => productIds.Contains(x.ProductId))
            .ToDictionaryAsync(x => x.ProductId, ct);

    private static void ValidateNoDuplicateProducts(IEnumerable<SalesOrderLineRequest> items)
    {
        if (items.GroupBy(x => x.ProductId).Any(x => x.Count() > 1))
            throw new ArgumentException("An order cannot contain duplicate product lines.");
    }

    private static void ValidateProducts(
        IEnumerable<SalesOrderLineRequest> items, IReadOnlyDictionary<int, Product> products)
    {
        foreach (var line in items)
        {
            if (!products.TryGetValue(line.ProductId, out var product) || !product.IsActive)
                throw new ArgumentException($"Active product {line.ProductId} was not found.");
            if (line.Quantity > product.StockQuantity)
                throw new InvalidOperationException($"Insufficient stock for product {product.Sku}.");
        }
    }

    private static SalesOrderResponse Map(OrderEntity x) => new(
        x.SalesOrderId, x.OrderNumber, x.CustomerId, x.Customer.Name,
        x.OrderDateUtc, x.Status, x.TotalAmount,
        Convert.ToBase64String(x.RowVersion),
        x.Items.Select(i => new SalesOrderItemResponse(
            i.SalesOrderItemId, i.ProductId, i.Product.Sku, i.Product.Name,
            i.Quantity, i.UnitPrice, i.Quantity * i.UnitPrice)).ToList());
}
```

## Controllers

`Controllers/ProductsController.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using SalesOrder.Api.Features.Products;

namespace SalesOrder.Api.Controllers;

[ApiController]
[Route("api/products")]
public sealed class ProductsController(IProductService service) : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IReadOnlyList<ProductResponse>>> GetAll(CancellationToken ct)
        => Ok(await service.GetAllAsync(ct));

    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductResponse>> GetById(int id, CancellationToken ct)
    {
        var product = await service.GetByIdAsync(id, ct);
        return product is null ? NotFound() : Ok(product);
    }

    [HttpPost]
    public async Task<ActionResult<ProductResponse>> Create(
        UpsertProductRequest request, CancellationToken ct)
    {
        var product = await service.CreateAsync(request, ct);
        return CreatedAtAction(nameof(GetById), new { id = product.ProductId }, product);
    }

    [HttpPut("{id:int}")]
    public async Task<ActionResult<ProductResponse>> Update(
        int id, UpsertProductRequest request, CancellationToken ct)
    {
        var product = await service.UpdateAsync(id, request, ct);
        return product is null ? NotFound() : Ok(product);
    }

    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id, CancellationToken ct)
        => await service.DeleteAsync(id, ct) ? NoContent() : NotFound();
}
```

`Controllers/SalesOrdersController.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using SalesOrder.Api.Features.SalesOrders;

namespace SalesOrder.Api.Controllers;

[ApiController]
[Route("api/sales-orders")]
public sealed class SalesOrdersController(ISalesOrderService service) : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IReadOnlyList<SalesOrderResponse>>> GetAll(CancellationToken ct)
        => Ok(await service.GetAllAsync(ct));

    [HttpGet("{id:long}")]
    public async Task<ActionResult<SalesOrderResponse>> GetById(long id, CancellationToken ct)
    {
        var order = await service.GetByIdAsync(id, ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    public async Task<ActionResult<SalesOrderResponse>> Create(
        CreateSalesOrderRequest request, CancellationToken ct)
    {
        var order = await service.CreateAsync(request, ct);
        return CreatedAtAction(nameof(GetById), new { id = order.SalesOrderId }, order);
    }

    [HttpPut("{id:long}")]
    public async Task<ActionResult<SalesOrderResponse>> Update(
        long id, UpdateSalesOrderRequest request, CancellationToken ct)
    {
        var order = await service.UpdateAsync(id, request, ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpDelete("{id:long}")]
    public async Task<IActionResult> Delete(
        long id, [FromHeader(Name = "If-Match")] string rowVersion, CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(rowVersion))
            return BadRequest(new ProblemDetails { Title = "If-Match row version is required." });

        return await service.DeleteDraftAsync(id, rowVersion.Trim('"'), ct)
            ? NoContent()
            : NotFound();
    }
}
```

## Global exception handler

`Common/GlobalExceptionHandler.cs`

```csharp
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace SalesOrder.Api.Common;

public sealed class GlobalExceptionHandler(
    IProblemDetailsService problemDetailsService,
    ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        logger.LogError(exception, "Unhandled request error. TraceId: {TraceId}", httpContext.TraceIdentifier);

        var (status, title) = exception switch
        {
            ArgumentException => (StatusCodes.Status400BadRequest, "Invalid request"),
            InvalidOperationException => (StatusCodes.Status409Conflict, "Business rule conflict"),
            DbUpdateConcurrencyException => (StatusCodes.Status409Conflict, "The record was changed by another request"),
            DbUpdateException => (StatusCodes.Status409Conflict, "Database constraint conflict"),
            FormatException => (StatusCodes.Status400BadRequest, "Invalid row version"),
            _ => (StatusCodes.Status500InternalServerError, "Unexpected server error")
        };

        httpContext.Response.StatusCode = status;
        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            ProblemDetails = new ProblemDetails
            {
                Status = status,
                Title = title,
                Detail = status == 500 ? null : exception.Message,
                Instance = httpContext.Request.Path
            }
        });
    }
}
```

## Program.cs

`src/SalesOrder.Api/Program.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using SalesOrder.Api.Common;
using SalesOrder.Api.Data;
using SalesOrder.Api.Features.Products;
using SalesOrder.Api.Features.SalesOrders;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddOpenApi();
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();

builder.Services.AddDbContext<SalesDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("SalesDb"),
        sql => sql.EnableRetryOnFailure(5, TimeSpan.FromSeconds(10), null)));

builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<ISalesOrderService, SalesOrderService>();

builder.Services.AddHealthChecks()
    .AddDbContextCheck<SalesDbContext>();

var app = builder.Build();

app.UseExceptionHandler();
app.UseHttpsRedirection();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.MapHealthChecks("/health");
app.MapControllers();
app.Run();

public partial class Program;
```

# 4. Create and run the database

## Option A: Run the supplied SQL script

Run `database/001-create-sales-order-db.sql` in SQL Server Management Studio or Azure Data Studio, then:

```bash
dotnet restore
dotnet build --no-restore
dotnet run --project src/SalesOrder.Api
```

## Option B: Use EF Core migrations

If EF Core owns the schema, do not also run the table-creation part of the SQL script.

```bash
dotnet ef migrations add InitialCreate \
  --project src/SalesOrder.Api \
  --startup-project src/SalesOrder.Api \
  --output-dir Data/Migrations

dotnet ef database update \
  --project src/SalesOrder.Api \
  --startup-project src/SalesOrder.Api
```

# 5. API contract

| Method | Endpoint | Purpose | Success |
|---|---|---|---|
| `GET` | `/api/products` | List products | `200` |
| `GET` | `/api/products/{id}` | Get one product | `200`, `404` |
| `POST` | `/api/products` | Create product | `201` |
| `PUT` | `/api/products/{id}` | Update product | `200`, `404`, `409` |
| `DELETE` | `/api/products/{id}` | Delete unused product | `204`, `404`, `409` |
| `GET` | `/api/sales-orders` | List orders with customer/items | `200` |
| `GET` | `/api/sales-orders/{id}` | Get one order | `200`, `404` |
| `POST` | `/api/sales-orders` | Create draft order | `201` |
| `PUT` | `/api/sales-orders/{id}` | Replace draft order details | `200`, `404`, `409` |
| `DELETE` | `/api/sales-orders/{id}` | Delete draft using `If-Match` | `204`, `404`, `409` |
| `GET` | `/health` | Database health check | `200`, `503` |

## Create a product

```http
POST /api/products
Content-Type: application/json

{
  "sku": "MOU-001",
  "name": "Wireless Mouse",
  "unitPrice": 34.99,
  "stockQuantity": 100,
  "isActive": true
}
```

## Create a sales order

```http
POST /api/sales-orders
Content-Type: application/json

{
  "customerId": 1,
  "items": [
    { "productId": 1, "quantity": 1 },
    { "productId": 2, "quantity": 2 }
  ]
}
```

## Update a sales order

Use the latest Base64 `rowVersion` returned by GET/POST:

```http
PUT /api/sales-orders/1
Content-Type: application/json

{
  "customerId": 1,
  "rowVersion": "AAAAAAAAB9E=",
  "items": [
    { "productId": 1, "quantity": 2 },
    { "productId": 3, "quantity": 1 }
  ]
}
```

## Delete a draft sales order

```http
DELETE /api/sales-orders/1
If-Match: "AAAAAAAAB9E="
```

# 6. Verification commands

```bash
dotnet format --verify-no-changes
dotnet build -warnaserror
dotnet test
```

OpenAPI JSON is available in Development at `/openapi/v1.json`.

# 7. Production checklist

Before production deployment, add the following:

- Microsoft Entra ID or another approved authentication provider
- Policy-based authorisation for read/write/admin operations
- API versioning and pagination for collection endpoints
- Structured logging without patient, customer, credential, or payment data
- Secret storage through environment variables, Azure Key Vault, or the approved platform
- Integration tests against SQL Server, plus unit tests for business rules
- Rate limiting, request-size limits, CORS allow-listing, and security headers
- Idempotency for create-order requests if clients may retry
- Stock reservation or stock deduction rules with suitable transaction isolation
- CI checks for restore, build, tests, formatting, dependency scanning, and migrations
- Exact NuGet versions instead of floating versions

# 8. Git setup

```bash
git init
dotnet new gitignore
git add .
git commit -m "feat: initialise sales order API"
git branch -M main
git remote add origin <YOUR_GITHUB_REPOSITORY_URL>
git push -u origin main
```

## Suggested repository description

> Production-minded ASP.NET Core .NET 10 Web API sample with SQL Server, EF Core, multi-table sales-order CRUD, validation, transactions, concurrency control, OpenAPI, and clean feature-based organisation.

## Licence

Add the licence approved for your repository. For public sample code, MIT is a common choice, but confirm organisational requirements before publishing.
