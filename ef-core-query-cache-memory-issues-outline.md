# EF Core LINQ Query Cache: How Thousands of Queries Can Kill Your Memory

## Article Outline

### Introduction
- Hook: "Your application runs fine with 100 users, but crashes with OutOfMemoryException at 1000+ users"
- The hidden danger of EF Core's query compilation cache
- Why this problem often goes unnoticed in development

### The Problem: EF Core Query Compilation Cache
- **What is query compilation cache?**
  - EF translates LINQ to SQL
  - Compiled queries are cached by query structure
  - Cache key generation process
  
- **When caching goes wrong**
  - Dynamic queries that generate unique cache keys
  - Each "different" query creates a new cache entry
  - Memory grows unbounded with query variations

### Common Patterns That Cause Cache Bloat

#### 1. String Interpolation in LINQ Queries
```csharp
// BAD: Creates a new cache entry for each userId
var user = context.Users.Where(u => u.Name == $"User_{userId}").FirstOrDefault();

// GOOD: Parameterized query reuses cache entry
var user = context.Users.Where(u => u.Name == userName).FirstOrDefault();
```

#### 2. Dynamic Where Clauses with Concatenation
- Building queries with string concatenation
- Each combination creates unique cache entries
- Example: Search filters, dynamic reporting queries

#### 3. Inline Collections in Queries
```csharp
// BAD: Each different array creates new cache entry
var orders = context.Orders.Where(o => new[] { 1, 2, 3 }.Contains(o.StatusId));

// GOOD: Use variables
var statusIds = new[] { 1, 2, 3 };
var orders = context.Orders.Where(o => statusIds.Contains(o.StatusId));
```

#### 4. DateTime.Now and Guid.NewGuid() in Queries
- Each execution creates a "different" query
- Cache grows with every request

### Real-World Example: The Production Incident
- **Scenario**: E-commerce search with dynamic filters
- **Symptoms**: 
  - Memory usage climbing steadily
  - OutOfMemoryException after hours/days
  - Performance degradation over time
- **Root cause**: Search query generating thousands of cache entries
- **Impact**: Application restarts, user complaints, revenue loss

### Diagnosing Query Cache Issues

#### Memory Profiling
- Using dotMemory/PerfView to identify EF Core objects
- Recognizing query cache memory patterns
- Tools for monitoring cache growth

#### EF Core Diagnostics
```csharp
// Enable query logging
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information)
                  .EnableSensitiveDataLogging();
}
```

#### Custom Cache Monitoring
- Accessing internal cache metrics
- Building alerts for cache size
- Tracking unique query patterns

### Solutions and Best Practices

#### 1. Parameterize Your Queries
- Always use variables instead of inline values
- Proper parameter binding techniques
- Code examples showing before/after

#### 2. Query Cache Size Limits
```csharp
// Set maximum cache size
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(connectionString, options => 
    {
        options.QueryCache = new QueryCache(maxSize: 1000);
    });
}
```

#### 3. Compiled Queries for Hot Paths
```csharp
private static readonly Func<MyDbContext, int, User> GetUserById =
    EF.CompileQuery((MyDbContext context, int id) => 
        context.Users.FirstOrDefault(u => u.Id == id));
```

#### 4. Query Splitting Strategies
- Break complex dynamic queries into simpler, cacheable parts
- Use multiple queries instead of one complex query
- Balance between network roundtrips and memory usage

#### 5. DbContext Lifecycle Management
- Shorter DbContext lifetimes
- Periodic context recreation
- Connection pooling considerations

### Advanced Techniques

#### Custom Query Cache Implementation
- Building your own caching layer
- Redis for distributed query caching
- Cache invalidation strategies

#### Expression Tree Analysis
- Understanding how EF builds cache keys
- Tools for analyzing query complexity
- Optimization techniques

### Prevention Strategies

#### Development Practices
- Code review checklist for LINQ queries
- Static analysis rules
- Unit tests for query cache behavior

#### Performance Testing
- Load testing with realistic query patterns
- Memory profiling in staging environments
- Automated cache growth monitoring

#### Monitoring in Production
- Application metrics and alerts
- Query performance tracking
- Memory usage dashboards

### Conclusion
- Key takeaways: Always parameterize, monitor cache growth, test under load
- The hidden costs of "innocent" LINQ queries
- Performance is a feature, not an afterthought
- Call to action: Audit your existing queries

### Additional Resources
- EF Core documentation links
- Memory profiling tools
- Performance testing frameworks
- Community discussions and GitHub issues

---

## Article Metadata
- **Target Audience**: Senior .NET developers, architects, DevOps engineers
- **Reading Time**: ~15 minutes
- **Difficulty**: Intermediate to Advanced
- **Tags**: EF Core, Performance, Memory Management, LINQ, .NET, Database
- **SEO Keywords**: "EF Core memory leak", "LINQ query cache", "OutOfMemoryException EF Core", "Entity Framework performance"