# EF Core LINQ Query Cache: How Thousands of Queries Can Kill Your Memory

Your application runs perfectly fine with 100 concurrent users. Response times are snappy, memory usage is stable, and everything looks great in your monitoring dashboards. Then you hit 1,000+ users, and suddenly your application starts throwing `OutOfMemoryException`s seemingly out of nowhere. Sound familiar?

If you're using Entity Framework Core with LINQ queries, you might be facing one of the most insidious performance issues in .NET applications: unbounded query compilation cache growth. This problem often goes completely unnoticed during development and even in staging environments, only to rear its ugly head when your application faces real-world traffic patterns.

## The Hidden Danger: EF Core Query Compilation Cache

To understand this problem, we first need to understand how Entity Framework Core handles LINQ queries under the hood.

### What is Query Compilation Cache?

When you write a LINQ query like this:

```csharp
var user = context.Users.Where(u => u.Id == userId).FirstOrDefault();
```

EF Core doesn't just translate this to SQL on the fly every time. Instead, it goes through a compilation process:

1. **Parse the LINQ expression tree** - EF analyzes your query structure
2. **Generate SQL** - The expression tree is converted to database-specific SQL
3. **Cache the compiled query** - The compiled query plan is stored in memory for reuse
4. **Execute** - The actual SQL is executed with your parameters

This caching mechanism is brilliant for performance - it means that subsequent executions of the "same" query skip the expensive compilation step and go straight to execution.

### When Caching Goes Terribly Wrong

The problem occurs when EF Core thinks you're writing "different" queries when you're actually trying to write the same query with different parameters. Each "unique" query structure gets its own cache entry, and if you're not careful, you can end up with thousands or even millions of cached query plans eating up your application's memory.

Here's the kicker: **EF Core generates cache keys based on the entire query structure, including inline values, not just the logical query pattern.**

This means that these two queries, which are logically identical, will create separate cache entries:

```csharp
// Cache entry #1
var user1 = context.Users.Where(u => u.Name == "John").FirstOrDefault();

// Cache entry #2  
var user2 = context.Users.Where(u => u.Name == "Jane").FirstOrDefault();
```

Multiply this by thousands of unique user names, product IDs, or search terms, and you've got a memory leak that will bring your application to its knees.

## Common Patterns That Cause Cache Bloat

Let me show you the most common ways developers accidentally create query cache bloat. I've seen every one of these patterns cause production outages.

### 1. String Interpolation and Concatenation in LINQ

This is probably the most common culprit:

```csharp
// 🚫 BAD: Creates a new cache entry for every userId
public async Task<User> GetUserByDynamicName(int userId)
{
    return await context.Users
        .Where(u => u.Name == $"User_{userId}")
        .FirstOrDefaultAsync();
}

// ✅ GOOD: Reuses the same cache entry
public async Task<User> GetUserByDynamicName(int userId)
{
    var userName = $"User_{userId}";
    return await context.Users
        .Where(u => u.Name == userName)
        .FirstOrDefaultAsync();
}
```

The first version will create a separate cache entry for "User_1", "User_2", "User_3", etc. With 10,000 users, you'll have 10,000 cache entries for essentially the same query.

### 2. Inline Collections in Queries

This one catches even experienced developers:

```csharp
// 🚫 BAD: Each different array creates a new cache entry
public async Task<List<Order>> GetOrdersByStatuses(int customerId)
{
    var statuses = GetActiveStatuses(); // Returns different arrays each time
    return await context.Orders
        .Where(o => o.CustomerId == customerId && 
                   new[] { 1, 2, 3 }.Contains(o.StatusId)) // Inline array!
        .ToListAsync();
}

// ✅ GOOD: Use a variable for the collection
public async Task<List<Order>> GetOrdersByStatuses(int customerId)
{
    var activeStatusIds = new[] { 1, 2, 3 };
    return await context.Orders
        .Where(o => o.CustomerId == customerId && 
                   activeStatusIds.Contains(o.StatusId))
        .ToListAsync();
}
```

### 3. DateTime.Now and Other "Unique" Values in Queries

This is a subtle but devastating pattern:

```csharp
// 🚫 BAD: Each execution creates a "different" query
public async Task<List<Event>> GetRecentEvents()
{
    return await context.Events
        .Where(e => e.CreatedAt > DateTime.Now.AddDays(-7))
        .ToListAsync();
}

// ✅ GOOD: Calculate the value outside the query
public async Task<List<Event>> GetRecentEvents()
{
    var weekAgo = DateTime.Now.AddDays(-7);
    return await context.Events
        .Where(e => e.CreatedAt > weekAgo)
        .ToListAsync();
}
```

Because `DateTime.Now` returns a different value each time, EF Core sees this as a completely different query structure.

### 4. Dynamic Query Building

This is where things get really dangerous:

```csharp
// 🚫 BAD: Dynamic where clauses
public async Task<List<Product>> SearchProducts(string name, decimal? minPrice, int? categoryId)
{
    var query = context.Products.AsQueryable();
    
    if (!string.IsNullOrEmpty(name))
        query = query.Where(p => p.Name.Contains(name)); // Different cache entry per name
    
    if (minPrice.HasValue)
        query = query.Where(p => p.Price >= minPrice.Value); // Different cache entry per price
    
    if (categoryId.HasValue)
        query = query.Where(p => p.CategoryId == categoryId.Value); // Different cache entry per category
    
    return await query.ToListAsync();
}
```

This innocent-looking method can generate 2³ = 8 different query structures, and if you're using inline values, the number explodes exponentially.

## Real-World Example: The E-commerce Search Meltdown

Let me share a war story from a project I worked on. We had an e-commerce platform with a sophisticated search feature that allowed users to filter products by multiple criteria: category, price range, brand, ratings, and free text search.

### The Implementation (That Nearly Killed Us)

The search method looked something like this:

```csharp
public async Task<List<Product>> SearchProducts(SearchRequest request)
{
    var query = context.Products.AsQueryable();
    
    if (!string.IsNullOrEmpty(request.SearchTerm))
        query = query.Where(p => p.Name.Contains(request.SearchTerm) || 
                                p.Description.Contains(request.SearchTerm));
    
    if (request.MinPrice > 0)
        query = query.Where(p => p.Price >= request.MinPrice);
    
    if (request.MaxPrice > 0 && request.MaxPrice < decimal.MaxValue)
        query = query.Where(p => p.Price <= request.MaxPrice);
    
    if (request.CategoryIds?.Any() == true)
        query = query.Where(p => request.CategoryIds.Contains(p.CategoryId));
    
    if (request.BrandIds?.Any() == true)
        query = query.Where(p => request.BrandIds.Contains(p.BrandId));
    
    return await query.OrderBy(p => p.Name).ToListAsync();
}
```

Looks reasonable, right? It worked great in development and even handled our staging load tests without issues.

### The Production Disaster

Three months after launch, during a peak shopping season, our application started experiencing random `OutOfMemoryException`s. The symptoms were baffling:

- **Memory usage climbed steadily** over hours, never decreasing
- **No obvious memory leaks** in our application code
- **Performance degraded** over time, even for simple operations
- **Application restarts** temporarily fixed the issue

Our monitoring showed memory usage climbing from 2GB at startup to over 8GB before crashing, with no correlation to user activity patterns.

### The Diagnosis

After days of profiling with dotMemory and PerfView, we discovered that our application was holding hundreds of thousands of compiled query objects in memory. The EF Core query compilation cache had grown to over 300,000 unique entries!

Here's what was happening:

1. **Every unique search term** created a new cache entry
2. **Every price combination** created a new cache entry  
3. **Every category/brand filter combination** created a new cache entry
4. **All search parameters together** multiplied the cache entries exponentially

With thousands of unique search terms and dozens of possible filter combinations, we were generating millions of unique query cache keys over time.

## Diagnosing Query Cache Issues

If you suspect you're dealing with query cache bloat, here's how to diagnose it:

### Memory Profiling

Use a memory profiler like dotMemory, PerfView, or Visual Studio Diagnostic Tools to look for:

- **Large numbers of `QueryCompilationContext` objects**
- **Growing collections of `CompiledQueryCacheKey` objects**  
- **Memory that never gets released**, even after GC

### EF Core Query Logging

Enable detailed query logging to see what queries are being generated:

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(connectionString)
        .LogTo(Console.WriteLine, LogLevel.Information)
        .EnableSensitiveDataLogging(); // Be careful with this in production!
}
```

Look for patterns where you see slight variations of the same query being logged repeatedly.

### Custom Cache Monitoring

You can access EF Core's internal cache metrics (though this requires some reflection):

```csharp
public class QueryCacheMonitor
{
    public static int GetCacheSize(DbContext context)
    {
        var serviceProvider = context.GetService<IServiceProvider>();
        var compiledQueryCache = serviceProvider.GetService<ICompiledQueryCache>();
        
        // Use reflection to access internal cache size
        var cacheField = compiledQueryCache.GetType()
            .GetField("_cache", BindingFlags.NonPublic | BindingFlags.Instance);
        
        if (cacheField?.GetValue(compiledQueryCache) is IDictionary cache)
        {
            return cache.Count;
        }
        
        return -1;
    }
}
```

## Solutions and Best Practices

Now let's fix these issues. Here are the strategies that actually work in production:

### 1. Always Parameterize Your Queries

The golden rule: **Never use inline values in LINQ queries**. Always extract values into variables:

```csharp
// 🚫 BAD
var products = context.Products
    .Where(p => p.CategoryId == 5 && p.Price > 100m)
    .ToList();

// ✅ GOOD
var categoryId = 5;
var minPrice = 100m;
var products = context.Products
    .Where(p => p.CategoryId == categoryId && p.Price > minPrice)
    .ToList();
```

### 2. Use Compiled Queries for Hot Paths

For frequently executed queries, use compiled queries to bypass the cache entirely:

```csharp
private static readonly Func<MyDbContext, int, User> GetUserById =
    EF.CompileQuery((MyDbContext context, int id) => 
        context.Users.FirstOrDefault(u => u.Id == id));

private static readonly Func<MyDbContext, string, IEnumerable<Product>> SearchProductsByName =
    EF.CompileQuery((MyDbContext context, string searchTerm) => 
        context.Products.Where(p => p.Name.Contains(searchTerm)));

// Usage
public async Task<User> GetUser(int id)
{
    return GetUserById(context, id);
}
```

### 3. Restructure Dynamic Queries

Instead of building complex dynamic queries, use a more structured approach:

```csharp
// ✅ BETTER: Predefined query structures
public async Task<List<Product>> SearchProducts(SearchRequest request)
{
    // Use a single parameterized query structure
    return await context.Products
        .Where(p => (string.IsNullOrEmpty(request.SearchTerm) || 
                    p.Name.Contains(request.SearchTerm)) &&
                   (request.MinPrice == null || p.Price >= request.MinPrice) &&
                   (request.MaxPrice == null || p.Price <= request.MaxPrice) &&
                   (request.CategoryIds == null || request.CategoryIds.Contains(p.CategoryId)))
        .ToListAsync();
}
```

This creates only one cache entry regardless of which parameters are provided.

### 4. Implement Query Cache Size Limits

While EF Core doesn't provide a built-in way to limit cache size, you can implement your own caching layer:

```csharp
public class BoundedQueryCache : ICompiledQueryCache
{
    private readonly ICompiledQueryCache _inner;
    private readonly ConcurrentDictionary<object, DateTime> _accessTimes;
    private readonly int _maxSize;
    
    public BoundedQueryCache(ICompiledQueryCache inner, int maxSize = 1000)
    {
        _inner = inner;
        _maxSize = maxSize;
        _accessTimes = new ConcurrentDictionary<object, DateTime>();
    }
    
    public TResult GetOrAddQuery<TResult>(object cacheKey, Func<TResult> compiler)
    {
        // Implement LRU eviction logic here
        if (_accessTimes.Count > _maxSize)
        {
            EvictOldestEntries();
        }
        
        _accessTimes[cacheKey] = DateTime.UtcNow;
        return _inner.GetOrAddQuery(cacheKey, compiler);
    }
    
    private void EvictOldestEntries()
    {
        // Remove 25% of oldest entries
        // Implementation details omitted for brevity
    }
}
```

### 5. Optimize DbContext Lifecycle

Use shorter-lived DbContext instances and consider connection pooling:

```csharp
// ✅ GOOD: Shorter context lifetimes
services.AddDbContextPool<MyDbContext>(options => 
    options.UseSqlServer(connectionString), 
    poolSize: 128);

// Use in controllers/services with scoped lifetime
public class ProductService
{
    private readonly IDbContextFactory<MyDbContext> _contextFactory;
    
    public ProductService(IDbContextFactory<MyDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }
    
    public async Task<List<Product>> GetProducts()
    {
        using var context = _contextFactory.CreateDbContext();
        return await context.Products.ToListAsync();
    }
}
```

## Prevention Strategies

### Code Review Checklist

Add these items to your code review process:

- [ ] No string interpolation or concatenation in LINQ queries
- [ ] No inline collections (arrays, lists) in Where clauses
- [ ] No DateTime.Now, Guid.NewGuid(), or other "unique" values in queries
- [ ] Dynamic queries use consistent parameterized structures
- [ ] Frequently called queries use compiled queries

### Static Analysis

Create custom Roslyn analyzers or use tools like SonarQube with custom rules to catch these patterns automatically.

### Performance Testing

Include query cache growth in your load testing scenarios:

```csharp
[Fact]
public async Task SearchProducts_DoesNotCauseMemoryLeak()
{
    var initialMemory = GC.GetTotalMemory(true);
    
    // Execute 10,000 different search queries
    for (int i = 0; i < 10000; i++)
    {
        await productService.SearchProducts(new SearchRequest 
        { 
            SearchTerm = $"Product {i}" 
        });
    }
    
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();
    
    var finalMemory = GC.GetTotalMemory(false);
    var memoryIncrease = finalMemory - initialMemory;
    
    // Memory increase should be reasonable (< 50MB for this test)
    Assert.True(memoryIncrease < 50_000_000, 
        $"Memory increased by {memoryIncrease:N0} bytes, indicating possible cache bloat");
}
```

## Monitoring in Production

Set up monitoring and alerts for query cache health:

### Application Metrics

```csharp
public class QueryCacheHealthCheck : IHealthCheck
{
    private readonly IDbContextFactory<MyDbContext> _contextFactory;
    
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, 
        CancellationToken cancellationToken = default)
    {
        try
        {
            using var dbContext = _contextFactory.CreateDbContext();
            var cacheSize = QueryCacheMonitor.GetCacheSize(dbContext);
            
            if (cacheSize > 10000) // Threshold based on your needs
            {
                return HealthCheckResult.Degraded(
                    $"Query cache size is high: {cacheSize} entries");
            }
            
            return HealthCheckResult.Healthy($"Query cache size: {cacheSize} entries");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Failed to check query cache", ex);
        }
    }
}
```

### Memory Dashboards

Track these metrics in your monitoring system (Application Insights, Datadog, etc.):

- **Total memory usage over time**
- **EF Core-related object counts**  
- **Query execution patterns**
- **Context creation/disposal rates**

## Conclusion

EF Core's query compilation cache is a powerful performance optimization that can also become your worst enemy if not handled carefully. The key takeaways are:

1. **Always parameterize your queries** - Never use inline values in LINQ expressions
2. **Be extremely careful with dynamic query building** - Each variation creates a new cache entry
3. **Monitor your cache growth** - Set up alerts and health checks
4. **Test under realistic load** - Cache bloat often doesn't appear until you hit production scale
5. **Use compiled queries for hot paths** - Skip the cache entirely for frequently executed queries

Remember: **Performance is not just about making things fast; it's also about making sure they stay fast under load.** A query that runs in 50ms but creates a memory leak is infinitely worse than a query that runs in 100ms reliably.

The patterns I've shown you are based on real production issues I've encountered over years of working with EF Core in high-traffic applications. Don't let query cache bloat be the reason your application goes down during peak traffic.

Take some time to audit your existing codebase for these patterns. Your future self (and your on-call rotation) will thank you.

---

*Have you encountered EF Core query cache issues in your applications? I'd love to hear about your experiences and solutions in the comments.*

**Tags:** Entity Framework Core, Performance, Memory Management, LINQ, .NET, Database Optimization