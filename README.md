📦 YourSolutionName
 ┣ 📂 src
 ┃ ┣ 📂 Application            # Business logic (Use Cases)
 ┃ ┃ ┣ 📂 Interfaces           # Contracts for repositories/services
 ┃ ┃ ┣ 📂 DTOs                 # Data Transfer Objects
 ┃ ┃ ┣ 📂 Features             # Use Cases (CQRS: Commands/Queries)
 ┃ ┃ ┃ ┣ 📂 Products           # Example Feature
 ┃ ┃ ┃ ┃ ┣ 📄 CreateProductCommand.cs
 ┃ ┃ ┃ ┃ ┣ 📄 GetProductQuery.cs
 ┃ ┃ ┃ ┃ ┣ 📄 DeleteProductCommand.cs
 ┃ ┃ ┃ ┃ ┣ 📄 UpdateProductCommand.cs
 ┃ ┣ 📂 Domain                 # Core Entities (Enterprise Business Rules)
 ┃ ┃ ┣ 📂 Entities             # Core Entities
 ┃ ┃ ┃ ┗ 📄 Product.cs
 ┃ ┃ ┣ 📂 Enums                # Enum Definitions
 ┃ ┃ ┣ 📂 ValueObjects         # Value Objects
 ┃ ┃ ┣ 📂 Events               # Domain Events
 ┃ ┣ 📂 Infrastructure         # External Implementations (Persistence, Services)
 ┃ ┃ ┣ 📂 Persistence          # Database (EF Core)
 ┃ ┃ ┃ ┣ 📄 AppDbContext.cs
 ┃ ┃ ┃ ┣ 📄 ProductRepository.cs
 ┃ ┃ ┃ ┣ 📄 UnitOfWork.cs
 ┃ ┃ ┣ 📂 Identity             # Authentication & Authorization
 ┃ ┃ ┣ 📂 Services             # External Services (Email, SMS, Logging)
 ┃ ┃ ┣ 📂 Configurations       # Configuration Classes
 ┃ ┣ 📂 WebAPI                 # Presentation Layer (Controllers, Middleware)
 ┃ ┃ ┣ 📂 Controllers          # API Controllers
 ┃ ┃ ┃ ┗ 📄 ProductsController.cs
 ┃ ┃ ┣ 📂 Middleware           # Custom Middlewares (Error Handling, Logging)
 ┃ ┃ ┣ 📂 Filters              # Action Filters
 ┃ ┃ ┣ 📂 Extensions           # Extension Methods
 ┃ ┃ ┣ 📄 Program.cs           # Entry Point
 ┃ ┃ ┣ 📄 appsettings.json      # Config File
 ┣ 📂 tests                    # Unit & Integration Tests
 ┃ ┣ 📂 Application.Tests      # Testing Business Logic
 ┃ ┣ 📂 WebAPI.Tests           # Testing API Endpoints
 ┃ ┣ 📂 Infrastructure.Tests   # Testing Persistence Layer
 ┣ 📄 README.md                # Documentation
 ┣ 📄 .gitignore               # Git Ignore
 ┣ 📄 docker-compose.yml        # Docker Config (Optional)



🛠️ Explanation of Each Layer
1️⃣ Domain Layer (Core)
Contains Entities, Enums, Value Objects, and Domain Events.
Does NOT depend on any other layer.
Represents the core business logic.
✅ Example Product.cs (Domain Entity):

csharp
Copy
Edit
public class Product
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public int Stock { get; private set; }

    public Product(string name, decimal price, int stock)
    {
        Name = name;
        Price = price;
        Stock = stock;
    }

    public void UpdateStock(int amount)
    {
        if (amount < 0) throw new InvalidOperationException("Stock cannot be negative");
        Stock = amount;
    }
}
2️⃣ Application Layer
Contains CQRS Handlers, Interfaces, DTOs, and Use Cases.
Implements business logic but does not know about infrastructure.
Uses Dependency Injection.
✅ Example CreateProductCommand.cs (CQRS - Command Handler):

csharp
Copy
Edit
public record CreateProductCommand(string Name, decimal Price, int Stock) : IRequest<int>;

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    private readonly IProductRepository _repository;

    public CreateProductCommandHandler(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<int> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        var product = new Product(request.Name, request.Price, request.Stock);
        await _repository.AddAsync(product);
        return product.Id;
    }
}
3️⃣ Infrastructure Layer
Implements Repositories, Database Context, Authentication, and External APIs.
Uses EF Core or any other persistence provider.
✅ Example ProductRepository.cs (EF Core Repository):

csharp
Copy
Edit
public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Product?> GetByIdAsync(int id) =>
        await _context.Products.FindAsync(id);

    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
    }
}
4️⃣ WebAPI Layer
The presentation layer containing Controllers, Middleware, and Filters.
✅ Example ProductsController.cs:

csharp
Copy
Edit
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<IActionResult> CreateProduct([FromBody] CreateProductCommand command)
    {
        var productId = await _mediator.Send(command);
        return CreatedAtAction(nameof(GetProductById), new { id = productId }, command);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetProductById(int id)
    {
        var product = await _mediator.Send(new GetProductQuery(id));
        return product is not null ? Ok(product) : NotFound();
    }
}
5️⃣ Tests Layer
Contains Unit Tests, Integration Tests, and API Tests.
✅ Example ProductServiceTests.cs (Unit Test):

csharp
Copy
Edit
public class ProductServiceTests
{
    private readonly Mock<IProductRepository> _mockRepo;
    private readonly CreateProductCommandHandler _handler;

    public ProductServiceTests()
    {
        _mockRepo = new Mock<IProductRepository>();
        _handler = new CreateProductCommandHandler(_mockRepo.Object);
    }

    [Fact]
    public async Task CreateProduct_ShouldReturnProductId()
    {
        // Arrange
        var command = new CreateProductCommand("Laptop", 999.99m, 10);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        Assert.True(result > 0);
    }
}
🎯 Benefits of This Structure
✅ Scalability → Easy to add new features without breaking existing ones.
✅ Maintainability → Each layer is independent, making debugging and updates easier.
✅ Testability → Supports unit and integration testing by following SOLID principles.
✅ Separation of Concerns → Business logic, API, and data layers are well-separated.
