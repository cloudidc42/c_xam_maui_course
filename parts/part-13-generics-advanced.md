# Part 13: Generics ขั้นสูง
## Steps 121-130: Generic Patterns และ Type System

---

## Step 121: Generic Constraints ขั้นสูง

```csharp
// ============================================
// Generic Constraints ทุกชนิด
// ============================================

// where T : class - reference type
public class RefContainer<T> where T : class
{
    public T? Value { get; set; } // nullable ref allowed
}

// where T : struct - value type
public class ValueContainer<T> where T : struct
{
    public T Value { get; set; }
    public T? NullableValue { get; set; }
}

// where T : new() - มี parameterless constructor
public T CreateDefault<T>() where T : new() => new T();

// where T : SomeClass - inherit from specific class
public void Process<T>(T item) where T : Stream
{
    item.Flush();
}

// where T : ISomeInterface - implement interface
public T GetMax<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

// where T : unmanaged - value type without managed references (C# 7.3+)
public unsafe void WriteToBuffer<T>(T value, byte* buffer) where T : unmanaged
{
    *(T*)buffer = value;
}

// Multiple constraints
public class EntityRepository<T> 
    where T : class, IEntity, new()
{
    public T Create()
    {
        var entity = new T();
        entity.Id = Guid.NewGuid();
        return entity;
    }
    
    public void Save(T entity) 
        where T : IValidatable // can't do this - constraint on method not class
    { }
}

public interface IEntity { Guid Id { get; set; } }

// ============================================
// Combining Constraints
// ============================================

public class ServiceBase<TService, TRepository, TEntity>
    where TEntity : class, IEntity, new()
    where TRepository : IAsyncRepository<TEntity, Guid>
    where TService : class
{
    protected readonly TRepository Repository;
    
    protected ServiceBase(TRepository repository) => Repository = repository;
}
```

---

## Step 122: Generic Type Inference

```csharp
// ============================================
// Type Inference
// ============================================

// ไม่ต้องระบุ type argument เมื่อ compiler infer ได้
public static T[] RepeatN<T>(T value, int count)
{
    var result = new T[count];
    Array.Fill(result, value);
    return result;
}

// Inferred:
var ints = RepeatN(42, 5);       // T = int
var strs = RepeatN("hello", 3);   // T = string

// ============================================
// Generic Factory Methods
// ============================================

public class Container<T>
{
    private readonly T _value;
    private Container(T value) => _value = value;
    
    // Factory method enables inference
    public static Container<T> Of(T value) => new(value);
    
    public Container<TResult> Map<TResult>(Func<T, TResult> f) 
        => Container<TResult>.Of(f(_value));
    
    public T Get() => _value;
}

// ใช้งาน
var c = Container.Of(42);                          // Container<int>
var doubled = c.Map(x => x * 2);                  // Container<int>
var stringified = c.Map(x => x.ToString());        // Container<string>

// ============================================
// Generic Delegate Inference
// ============================================

Func<int, string> toString = x => x.ToString();
Func<string, bool> hasLength = s => s.Length > 3;

// Compose using generics
Func<T1, T3> Compose<T1, T2, T3>(Func<T1, T2> f, Func<T2, T3> g)
    => x => g(f(x));

var isLong = Compose(toString, hasLength); // Func<int, bool>
Console.WriteLine(isLong(1234)); // true (4 chars)
Console.WriteLine(isLong(12));   // false (2 chars)
```

---

## Step 123: Variance (Covariance/Contravariance)

```csharp
// ============================================
// Covariance (out)
// ============================================

// IEnumerable<out T> is covariant
// สามารถใช้ IEnumerable<Dog> แทน IEnumerable<Animal>

public class Animal2 { public string Sound { get; set; } = "..."; }
public class Dog : Animal2 { public Dog() { Sound = "Woof"; } }
public class Cat : Animal2 { public Cat() { Sound = "Meow"; } }

IEnumerable<Dog> dogs = new List<Dog> { new Dog(), new Dog() };
IEnumerable<Animal2> animals = dogs; // OK! IEnumerable<Dog> -> IEnumerable<Animal2>

// Custom covariant interface
public interface IProducer<out T>
{
    T Produce();
}

public class DogProducer : IProducer<Dog>
{
    public Dog Produce() => new Dog();
}

IProducer<Animal2> producer = new DogProducer(); // OK! covariant
Animal2 animal = producer.Produce(); // Returns Dog, but typed as Animal2

// ============================================
// Contravariance (in)
// ============================================

// Action<in T> is contravariant
// สามารถใช้ Action<Animal> แทน Action<Dog>

Action<Animal2> makeSound = a => Console.WriteLine(a.Sound);
Action<Dog> dogAction = makeSound; // OK! contravariant

dogAction(new Dog()); // Prints "Woof"

// Custom contravariant interface
public interface IConsumer<in T>
{
    void Consume(T item);
}

public class AnimalConsumer : IConsumer<Animal2>
{
    public void Consume(Animal2 item) => Console.WriteLine($"Processing: {item.Sound}");
}

IConsumer<Dog> dogConsumer = new AnimalConsumer(); // OK! contravariant
dogConsumer.Consume(new Dog());

// ============================================
// Practical Example: Comparison
// ============================================

public class AnimalComparer : IComparer<Animal2>
{
    public int Compare(Animal2? x, Animal2? y)
        => string.Compare(x?.Sound, y?.Sound);
}

// AnimalComparer can be used for dogs (contravariance)
IComparer<Dog> dogComparer = new AnimalComparer();
var dogs2 = new List<Dog> { new Dog(), new Dog() };
dogs2.Sort(dogComparer); // works!
```

---

## Step 124: Generic Math (C# 11+)

```csharp
// ============================================
// Generic Math (INumber<T>)
// ============================================

using System.Numerics;

// Generic Sum ที่ทำงานกับ int, double, decimal, etc.
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T total = T.Zero;
    foreach (var v in values)
        total += v;
    return total;
}

// ใช้งาน
var ints2 = new[] { 1, 2, 3, 4, 5 };
var doubles = new[] { 1.1, 2.2, 3.3 };
var decimals2 = new[] { 1.5m, 2.5m, 3.5m };

Console.WriteLine(Sum(ints2));     // 15
Console.WriteLine(Sum(doubles));   // 6.6
Console.WriteLine(Sum(decimals2)); // 7.5

// Generic Statistics
public static class Stats<T> where T : INumber<T>
{
    public static T Average(IEnumerable<T> values)
    {
        var list = values.ToList();
        if (list.Count == 0) return T.Zero;
        return Sum(list) / T.CreateChecked(list.Count);
    }
    
    public static (T Min, T Max) MinMax(IEnumerable<T> values)
    {
        var list = values.ToList();
        return (list.Min()!, list.Max()!);
    }
}

Console.WriteLine(Stats<int>.Average(ints2));              // 3
Console.WriteLine(Stats<double>.Average(doubles));          // 2.2
var (min, max) = Stats<decimal>.MinMax(decimals2);
Console.WriteLine($"Min={min}, Max={max}");

// Generic Vector
public struct Vector2D<T> where T : INumber<T>
{
    public T X { get; }
    public T Y { get; }
    
    public Vector2D(T x, T y) { X = x; Y = y; }
    
    public static Vector2D<T> operator +(Vector2D<T> a, Vector2D<T> b)
        => new(a.X + b.X, a.Y + b.Y);
    
    public static Vector2D<T> operator *(Vector2D<T> v, T scalar)
        => new(v.X * scalar, v.Y * scalar);
    
    public T Magnitude() => T.Sqrt(X * X + Y * Y);
    
    public override string ToString() => $"({X}, {Y})";
}
```

---

## Step 125: Generic Data Structures

```csharp
// ============================================
// Generic Stack
// ============================================

public class GenericStack<T>
{
    private T[] _items;
    private int _count;
    
    public GenericStack(int initialCapacity = 16)
    {
        _items = new T[initialCapacity];
    }
    
    public int Count => _count;
    public bool IsEmpty => _count == 0;
    
    public void Push(T item)
    {
        EnsureCapacity();
        _items[_count++] = item;
    }
    
    public T Pop()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        var item = _items[--_count];
        _items[_count] = default!; // clear reference
        return item;
    }
    
    public T Peek()
    {
        if (IsEmpty) throw new InvalidOperationException("Stack is empty");
        return _items[_count - 1];
    }
    
    public bool TryPop(out T item)
    {
        if (IsEmpty) { item = default!; return false; }
        item = Pop();
        return true;
    }
    
    private void EnsureCapacity()
    {
        if (_count == _items.Length)
        {
            var newItems = new T[_items.Length * 2];
            Array.Copy(_items, newItems, _count);
            _items = newItems;
        }
    }
}

// ============================================
// Generic Queue (Circular Buffer)
// ============================================

public class CircularBuffer<T>
{
    private readonly T[] _buffer;
    private int _head, _tail, _count;
    
    public CircularBuffer(int capacity)
    {
        _buffer = new T[capacity];
    }
    
    public int Capacity => _buffer.Length;
    public int Count => _count;
    public bool IsFull => _count == Capacity;
    public bool IsEmpty => _count == 0;
    
    public void Enqueue(T item)
    {
        if (IsFull) throw new InvalidOperationException("Buffer is full");
        _buffer[_tail] = item;
        _tail = (_tail + 1) % Capacity;
        _count++;
    }
    
    public T Dequeue()
    {
        if (IsEmpty) throw new InvalidOperationException("Buffer is empty");
        var item = _buffer[_head];
        _buffer[_head] = default!;
        _head = (_head + 1) % Capacity;
        _count--;
        return item;
    }
    
    public T Peek() 
    {
        if (IsEmpty) throw new InvalidOperationException("Buffer is empty");
        return _buffer[_head];
    }
}

// ============================================
// Generic Graph
// ============================================

public class Graph<TNode, TWeight> 
    where TWeight : IComparable<TWeight>
{
    private readonly Dictionary<TNode, List<(TNode To, TWeight Weight)>> _adjacency = new();
    
    public void AddNode(TNode node)
    {
        if (!_adjacency.ContainsKey(node))
            _adjacency[node] = new();
    }
    
    public void AddEdge(TNode from, TNode to, TWeight weight)
    {
        AddNode(from);
        AddNode(to);
        _adjacency[from].Add((to, weight));
    }
    
    public IEnumerable<TNode> BFS(TNode start)
    {
        var visited = new HashSet<TNode>();
        var queue = new Queue<TNode>();
        queue.Enqueue(start);
        visited.Add(start);
        
        while (queue.Count > 0)
        {
            var node = queue.Dequeue();
            yield return node;
            
            foreach (var (neighbor, _) in _adjacency.GetValueOrDefault(node, new()))
            {
                if (visited.Add(neighbor))
                    queue.Enqueue(neighbor);
            }
        }
    }
}
```

---

## Step 126: Generic Caching

```csharp
// ============================================
// Type-safe Generic Cache
// ============================================

public class TypedCache
{
    private readonly Dictionary<string, object> _store = new();
    private readonly Dictionary<string, DateTime> _expiry = new();
    
    public void Set<T>(string key, T value, TimeSpan? expiry = null)
    {
        _store[key] = value!;
        if (expiry.HasValue)
            _expiry[key] = DateTime.Now.Add(expiry.Value);
    }
    
    public bool TryGet<T>(string key, out T? value)
    {
        value = default;
        
        if (_expiry.TryGetValue(key, out var expiryTime) && expiryTime < DateTime.Now)
        {
            _store.Remove(key);
            _expiry.Remove(key);
            return false;
        }
        
        if (_store.TryGetValue(key, out var obj) && obj is T typedValue)
        {
            value = typedValue;
            return true;
        }
        
        return false;
    }
    
    public T GetOrAdd<T>(string key, Func<T> factory, TimeSpan? expiry = null)
    {
        if (TryGet<T>(key, out var cached))
            return cached!;
        
        var value = factory();
        Set(key, value, expiry);
        return value;
    }
    
    public async Task<T> GetOrAddAsync<T>(
        string key, 
        Func<Task<T>> factory, 
        TimeSpan? expiry = null)
    {
        if (TryGet<T>(key, out var cached))
            return cached!;
        
        var value = await factory();
        Set(key, value, expiry);
        return value;
    }
}

// ใช้งาน
var cache = new TypedCache();

cache.Set("users", new[] { "Alice", "Bob" }, TimeSpan.FromMinutes(5));
cache.Set("config", new { MaxConnections = 10, Timeout = 30 }, TimeSpan.FromHours(1));

if (cache.TryGet<string[]>("users", out var users))
    Console.WriteLine($"Cached users: {string.Join(", ", users!)}");

var expensiveData = await cache.GetOrAddAsync("expensive", async () =>
{
    await Task.Delay(1000); // simulate slow operation
    return "computed value";
}, TimeSpan.FromMinutes(10));
```

---

## Step 127: Functional Generics

```csharp
// ============================================
// Maybe Monad (Option Type)
// ============================================

public readonly struct Maybe<T>
{
    private readonly T? _value;
    private readonly bool _hasValue;
    
    private Maybe(T value) { _value = value; _hasValue = true; }
    
    public static Maybe<T> Some(T value) => new(value);
    public static Maybe<T> None => new();
    
    public bool HasValue => _hasValue;
    public T Value => _hasValue ? _value! : throw new InvalidOperationException("No value");
    
    public Maybe<TResult> Map<TResult>(Func<T, TResult> mapper)
        => _hasValue ? Maybe<TResult>.Some(mapper(_value!)) : Maybe<TResult>.None;
    
    public Maybe<TResult> Bind<TResult>(Func<T, Maybe<TResult>> binder)
        => _hasValue ? binder(_value!) : Maybe<TResult>.None;
    
    public T GetValueOrDefault(T defaultValue)
        => _hasValue ? _value! : defaultValue;
    
    public void Match(Action<T> onSome, Action onNone)
    {
        if (_hasValue) onSome(_value!);
        else onNone();
    }
    
    public TResult Match<TResult>(Func<T, TResult> onSome, Func<TResult> onNone)
        => _hasValue ? onSome(_value!) : onNone();
    
    public static implicit operator Maybe<T>(T value) => Some(value);
    
    public override string ToString() => _hasValue ? $"Some({_value})" : "None";
}

// Extension methods for Maybe
public static class MaybeExtensions
{
    public static Maybe<T> ToMaybe<T>(this T? value) where T : class
        => value != null ? Maybe<T>.Some(value) : Maybe<T>.None;
    
    public static Maybe<T> ToMaybe<T>(this T? value) where T : struct
        => value.HasValue ? Maybe<T>.Some(value.Value) : Maybe<T>.None;
    
    public static Maybe<TValue> TryGetValue<TKey, TValue>(
        this Dictionary<TKey, TValue> dict, TKey key) where TKey : notnull
        => dict.TryGetValue(key, out var value) ? Maybe<TValue>.Some(value) : Maybe<TValue>.None;
}

// ใช้งาน
var users2 = new Dictionary<int, string>
{
    { 1, "Alice" }, { 2, "Bob" }
};

var user = users2.TryGetValue(1)
    .Map(name => name.ToUpper())
    .Map(name => $"Hello, {name}!");

user.Match(
    msg => Console.WriteLine(msg),       // "Hello, ALICE!"
    () => Console.WriteLine("Not found")
);

var missing = users2.TryGetValue(999)
    .Map(name => name.ToUpper());

Console.WriteLine(missing); // "None"
Console.WriteLine(missing.GetValueOrDefault("Unknown")); // "Unknown"
```

---

## Step 128: Generic Repository Pattern

```csharp
// ============================================
// Complete Generic Repository
// ============================================

public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    IQueryable<T> Apply(IQueryable<T> query);
}

public abstract class Specification<T> : ISpecification<T>
{
    public abstract bool IsSatisfiedBy(T entity);
    
    public virtual IQueryable<T> Apply(IQueryable<T> query)
        => query.Where(e => IsSatisfiedBy(e));
    
    public Specification<T> And(Specification<T> other)
        => new AndSpecification<T>(this, other);
    
    public Specification<T> Or(Specification<T> other)
        => new OrSpecification<T>(this, other);
    
    public Specification<T> Not()
        => new NotSpecification<T>(this);
}

public class AndSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public AndSpecification(Specification<T> left, Specification<T> right)
    { _left = left; _right = right; }
    public override bool IsSatisfiedBy(T entity) 
        => _left.IsSatisfiedBy(entity) && _right.IsSatisfiedBy(entity);
}

public class OrSpecification<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public OrSpecification(Specification<T> left, Specification<T> right)
    { _left = left; _right = right; }
    public override bool IsSatisfiedBy(T entity) 
        => _left.IsSatisfiedBy(entity) || _right.IsSatisfiedBy(entity);
}

public class NotSpecification<T> : Specification<T>
{
    private readonly Specification<T> _inner;
    public NotSpecification(Specification<T> inner) => _inner = inner;
    public override bool IsSatisfiedBy(T entity) => !_inner.IsSatisfiedBy(entity);
}

// Product Specifications
public record Product7(int Id, string Name, string Category, decimal Price, bool IsActive);

public class ActiveProductSpec : Specification<Product7>
{
    public override bool IsSatisfiedBy(Product7 p) => p.IsActive;
}

public class CategorySpec : Specification<Product7>
{
    private readonly string _category;
    public CategorySpec(string category) => _category = category;
    public override bool IsSatisfiedBy(Product7 p) 
        => p.Category.Equals(_category, StringComparison.OrdinalIgnoreCase);
}

public class PriceRangeSpec : Specification<Product7>
{
    private readonly decimal _min, _max;
    public PriceRangeSpec(decimal min, decimal max) { _min = min; _max = max; }
    public override bool IsSatisfiedBy(Product7 p) => p.Price >= _min && p.Price <= _max;
}

// ใช้งาน Specification Pattern
var products3 = new List<Product7>
{
    new(1, "iPhone 15", "Electronics", 35000, true),
    new(2, "Samsung S24", "Electronics", 30000, false),
    new(3, "Nike Air", "Shoes", 4500, true),
    new(4, "iPad Pro", "Electronics", 42000, true),
    new(5, "Adidas Ultra", "Shoes", 3800, true),
};

var activeElectronics = new ActiveProductSpec()
    .And(new CategorySpec("Electronics"));

var affordableShoes = new CategorySpec("Shoes")
    .And(new PriceRangeSpec(0, 5000));

var results3 = products3.Where(p => activeElectronics.IsSatisfiedBy(p));
Console.WriteLine("Active Electronics:");
results3.ToList().ForEach(p => Console.WriteLine($"  {p.Name}: {p.Price:C}"));

var shoesResult = products3.Where(p => affordableShoes.IsSatisfiedBy(p));
Console.WriteLine("\nAffordable Shoes:");
shoesResult.ToList().ForEach(p => Console.WriteLine($"  {p.Name}: {p.Price:C}"));
```

---

## Step 129: Generic Event System

```csharp
// ============================================
// Generic Event Bus
// ============================================

public interface IEvent { DateTime OccurredAt { get; } }

public abstract record BaseEvent : IEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record OrderPlacedEvent(int OrderId, string CustomerId, decimal Total) : BaseEvent;
public record OrderShippedEvent(int OrderId, string TrackingNumber) : BaseEvent;
public record UserRegisteredEvent(int UserId, string Email) : BaseEvent;

public class EventBus
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();
    
    public void Subscribe<TEvent>(Action<TEvent> handler) where TEvent : IEvent
    {
        var type = typeof(TEvent);
        if (!_handlers.ContainsKey(type))
            _handlers[type] = new();
        _handlers[type].Add(handler);
    }
    
    public void Subscribe<TEvent>(Func<TEvent, Task> handler) where TEvent : IEvent
    {
        var type = typeof(TEvent);
        if (!_handlers.ContainsKey(type))
            _handlers[type] = new();
        _handlers[type].Add(handler);
    }
    
    public void Unsubscribe<TEvent>(Action<TEvent> handler) where TEvent : IEvent
    {
        if (_handlers.TryGetValue(typeof(TEvent), out var handlers))
            handlers.Remove(handler);
    }
    
    public async Task PublishAsync<TEvent>(TEvent @event) where TEvent : IEvent
    {
        if (!_handlers.TryGetValue(typeof(TEvent), out var handlers))
            return;
        
        var tasks = handlers.Select(h => h switch
        {
            Action<TEvent> action => Task.Run(() => action(@event)),
            Func<TEvent, Task> func => func(@event),
            _ => Task.CompletedTask
        });
        
        await Task.WhenAll(tasks);
    }
}

// ใช้งาน
var bus = new EventBus();

bus.Subscribe<OrderPlacedEvent>(async e =>
{
    await Task.Delay(10);
    Console.WriteLine($"[Email] Sending confirmation for order #{e.OrderId}");
});

bus.Subscribe<OrderPlacedEvent>(e =>
{
    Console.WriteLine($"[Inventory] Reserving items for order #{e.OrderId}");
});

bus.Subscribe<UserRegisteredEvent>(async e =>
{
    await Task.Delay(10);
    Console.WriteLine($"[Welcome Email] Sending to {e.Email}");
});

await bus.PublishAsync(new OrderPlacedEvent(1001, "C001", 35000));
await bus.PublishAsync(new UserRegisteredEvent(501, "somchai@gmail.com"));
```

---

## Step 130: Generic Pipeline

```csharp
// ============================================
// Generic Pipeline Builder
// ============================================

public interface IPipelineStep<TInput, TOutput>
{
    Task<TOutput> ExecuteAsync(TInput input, CancellationToken ct = default);
}

public class Pipeline<T>
{
    private readonly List<Func<T, CancellationToken, Task<T>>> _steps = new();
    
    public Pipeline<T> AddStep(Func<T, CancellationToken, Task<T>> step)
    {
        _steps.Add(step);
        return this;
    }
    
    public Pipeline<T> AddStep(Func<T, T> step)
    {
        _steps.Add((x, _) => Task.FromResult(step(x)));
        return this;
    }
    
    public async Task<T> ExecuteAsync(T input, CancellationToken ct = default)
    {
        T current = input;
        foreach (var step in _steps)
            current = await step(current, ct);
        return current;
    }
}

// Pipeline สำหรับ string processing
var textPipeline = new Pipeline<string>()
    .AddStep(text => text.Trim())
    .AddStep(text => text.ToLower())
    .AddStep(text => System.Text.RegularExpressions.Regex.Replace(text, @"\s+", " "))
    .AddStep(async (text, ct) =>
    {
        await Task.Delay(10, ct); // simulate async operation
        return text.Replace("bad", "***");
    });

string dirtyText = "  Hello   World   bad   Word  ";
string clean = await textPipeline.ExecuteAsync(dirtyText);
Console.WriteLine($"Original: '{dirtyText}'");
Console.WriteLine($"Processed: '{clean}'");

// Image processing pipeline (concept)
public record ImageData(byte[] Pixels, int Width, int Height);

var imagePipeline = new Pipeline<ImageData>()
    .AddStep(img => ResizeImage(img, 800, 600))
    .AddStep(img => ConvertToGrayscale(img))
    .AddStep(async (img, ct) => await CompressAsync(img, ct));

static ImageData ResizeImage(ImageData img, int w, int h) 
    => img with { Width = w, Height = h };
static ImageData ConvertToGrayscale(ImageData img) => img; // simplified
static async Task<ImageData> CompressAsync(ImageData img, CancellationToken ct)
{
    await Task.Delay(10, ct);
    return img;
}
```

---

## สรุป Part 13

ใน Part 13 เราได้เรียนรู้:

1. **Generic Constraints** - class, struct, new(), unmanaged, interface
2. **Type Inference** - Compiler infers generic arguments
3. **Variance** - Covariance (out), Contravariance (in)
4. **Generic Math** - INumber<T> (C# 11+)
5. **Generic Data Structures** - Stack, Circular Buffer, Graph
6. **Generic Caching** - TypedCache with expiry
7. **Maybe Monad** - Option type for null safety
8. **Specification Pattern** - Composable business rules
9. **Generic Event Bus** - Type-safe events
10. **Generic Pipeline** - Composable transformations

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 13 | Steps 121-130*
