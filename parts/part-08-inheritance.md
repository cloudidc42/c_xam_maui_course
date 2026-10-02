# Part 08: Inheritance และ Polymorphism
## Steps 71-80: การสืบทอดและความหลากหลาย

---

## Step 71: Inheritance พื้นฐาน

```csharp
// ============================================
// Base Class (Parent)
// ============================================

public class Animal
{
    public string Name { get; }
    public int Age { get; }
    public string Sound { get; protected set; } = "...";
    
    public Animal(string name, int age)
    {
        Name = name;
        Age = age;
    }
    
    public virtual void MakeSound()
    {
        Console.WriteLine($"{Name} says: {Sound}");
    }
    
    public virtual string GetInfo()
        => $"{GetType().Name}: {Name} (age: {Age})";
    
    public override string ToString() => GetInfo();
}

// ============================================
// Derived Classes (Children)
// ============================================

public class Dog : Animal
{
    public string Breed { get; }
    
    public Dog(string name, int age, string breed) 
        : base(name, age) // เรียก constructor ของ parent
    {
        Breed = breed;
        Sound = "Woof!";
    }
    
    // Override method
    public override void MakeSound()
    {
        Console.WriteLine($"{Name} barks: {Sound} {Sound}");
    }
    
    public override string GetInfo()
        => $"{base.GetInfo()}, Breed: {Breed}"; // เรียก parent method
    
    public void Fetch(string item)
    {
        Console.WriteLine($"{Name} fetches the {item}!");
    }
}

public class Cat : Animal
{
    public bool IsIndoor { get; }
    
    public Cat(string name, int age, bool isIndoor = true) 
        : base(name, age)
    {
        IsIndoor = isIndoor;
        Sound = "Meow!";
    }
    
    public override void MakeSound()
    {
        Console.WriteLine($"{Name} says: {Sound}");
    }
    
    public void Purr()
    {
        Console.WriteLine($"{Name} purrs... *purrr*");
    }
}

public class Bird : Animal
{
    public bool CanFly { get; }
    
    public Bird(string name, int age, bool canFly = true) 
        : base(name, age)
    {
        CanFly = canFly;
        Sound = "Tweet!";
    }
    
    public override void MakeSound()
    {
        string action = CanFly ? "flies and tweets" : "tweets";
        Console.WriteLine($"{Name} {action}: {Sound}");
    }
}

// ============================================
// Polymorphism
// ============================================

var animals = new List<Animal>
{
    new Dog("บุญมา", 3, "Shiba Inu"),
    new Cat("มาลี", 2),
    new Bird("แก้ว", 1),
    new Dog("รัก", 5, "Golden Retriever"),
    new Cat("น้ำตาล", 4, false)
};

Console.WriteLine("=== All Animals ===");
foreach (Animal animal in animals)
{
    animal.MakeSound(); // เรียก method ของ subclass ที่ถูกต้อง
    Console.WriteLine(animal.GetInfo());
}

// Type checking
foreach (Animal animal in animals)
{
    if (animal is Dog dog)
        dog.Fetch("ball");
    else if (animal is Cat cat)
        cat.Purr();
}
```

---

## Step 72: Abstract Classes

```csharp
// ============================================
// Abstract Class
// ============================================

// Abstract class: ไม่สามารถ instantiate ได้โดยตรง
// แต่สามารถมี implementation บางส่วนได้
public abstract class Shape
{
    public string Color { get; set; } = "White";
    public string Name { get; }
    
    protected Shape(string name)
    {
        Name = name;
    }
    
    // Abstract method: ต้อง override ใน subclass
    public abstract double GetArea();
    public abstract double GetPerimeter();
    
    // Concrete method: มี implementation ใน base class
    public virtual void Draw()
    {
        Console.WriteLine($"Drawing {Name} (Color: {Color})");
        Console.WriteLine($"  Area: {GetArea():F2}");
        Console.WriteLine($"  Perimeter: {GetPerimeter():F2}");
    }
    
    // Template method pattern
    public void Describe()
    {
        Console.Write($"{Name}: ");
        PrintMeasurements(); // call abstract helper
        Console.WriteLine($"Color: {Color}");
    }
    
    protected virtual void PrintMeasurements()
    {
        Console.Write($"Area={GetArea():F2}, Perimeter={GetPerimeter():F2}, ");
    }
}

public class Circle2 : Shape
{
    public double Radius { get; }
    
    public Circle2(double radius, string color = "White") 
        : base("Circle")
    {
        Radius = radius;
        Color = color;
    }
    
    public override double GetArea() => Math.PI * Radius * Radius;
    
    public override double GetPerimeter() => 2 * Math.PI * Radius;
    
    protected override void PrintMeasurements()
    {
        Console.Write($"Radius={Radius}, ");
        base.PrintMeasurements();
    }
}

public class Rectangle2 : Shape
{
    public double Width { get; }
    public double Height { get; }
    
    public Rectangle2(double width, double height, string color = "White") 
        : base("Rectangle")
    {
        Width = width;
        Height = height;
        Color = color;
    }
    
    public override double GetArea() => Width * Height;
    public override double GetPerimeter() => 2 * (Width + Height);
}

public class Triangle : Shape
{
    public double A { get; }
    public double B { get; }
    public double C { get; }
    
    public Triangle(double a, double b, double c) : base("Triangle")
    {
        // Validate triangle inequality
        if (a + b <= c || b + c <= a || a + c <= b)
            throw new ArgumentException("Invalid triangle sides");
        A = a; B = b; C = c;
    }
    
    public override double GetArea()
    {
        // Heron's formula
        double s = GetPerimeter() / 2;
        return Math.Sqrt(s * (s - A) * (s - B) * (s - C));
    }
    
    public override double GetPerimeter() => A + B + C;
}

// ใช้งาน
var shapes = new List<Shape>
{
    new Circle2(5, "Red"),
    new Rectangle2(4, 6, "Blue"),
    new Triangle(3, 4, 5)
};

foreach (var shape in shapes)
{
    shape.Draw();
    Console.WriteLine();
}

// ============================================
// Abstract Factory Pattern
// ============================================

public abstract class UIComponent
{
    public abstract void Render();
    public abstract void HandleClick();
    
    public virtual string GetHtml() => $"<{GetType().Name.ToLower()} />";
}

public abstract class Button : UIComponent
{
    public string Text { get; }
    
    protected Button(string text) => Text = text;
    
    public override void HandleClick() 
        => Console.WriteLine($"Button '{Text}' clicked");
}

public class PrimaryButton : Button
{
    public PrimaryButton(string text) : base(text) { }
    
    public override void Render()
        => Console.WriteLine($"[PRIMARY BUTTON: {Text}]");
}

public class DangerButton : Button
{
    public DangerButton(string text) : base(text) { }
    
    public override void Render()
        => Console.WriteLine($"[DANGER BUTTON: {Text}]");
    
    public override void HandleClick()
    {
        Console.Write("Are you sure? ");
        base.HandleClick();
    }
}
```

---

## Step 73: Sealed Classes และ Override

```csharp
// ============================================
// sealed Class - ไม่สามารถ inherit ได้
// ============================================

public sealed class Singleton
{
    private static Singleton? _instance;
    private static readonly object _lock = new();
    
    private Singleton() { }
    
    public static Singleton Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    _instance ??= new Singleton();
                }
            }
            return _instance;
        }
    }
    
    public void DoWork() => Console.WriteLine("Singleton doing work...");
}

// Singleton._instance = ... // ไม่ได้ (private)
// class MySingleton : Singleton {} // ERROR: sealed

Singleton.Instance.DoWork();

// ============================================
// sealed Method - ป้องกัน override เพิ่มเติม
// ============================================

public class BaseClass
{
    public virtual void Method() => Console.WriteLine("Base");
}

public class MiddleClass : BaseClass
{
    public override sealed void Method()
    {
        Console.WriteLine("Middle - no more override!");
    }
}

public class ChildClass : MiddleClass
{
    // public override void Method() { } // ERROR: sealed
}

// ============================================
// new vs override
// ============================================

public class Parent
{
    public virtual void VirtualMethod()
        => Console.WriteLine("Parent.VirtualMethod");
    
    public void NormalMethod()
        => Console.WriteLine("Parent.NormalMethod");
}

public class Child : Parent
{
    // override: polymorphic (dynamic dispatch)
    public override void VirtualMethod()
        => Console.WriteLine("Child.VirtualMethod");
    
    // new: hide parent method (not polymorphic)
    public new void NormalMethod()
        => Console.WriteLine("Child.NormalMethod");
}

var child = new Child();
Parent parent = child; // upcasting

child.VirtualMethod();    // Child.VirtualMethod
parent.VirtualMethod();   // Child.VirtualMethod (polymorphic!)

child.NormalMethod();     // Child.NormalMethod
parent.NormalMethod();    // Parent.NormalMethod (not polymorphic - hides)
```

---

## Step 74: Interfaces

```csharp
// ============================================
// Interface Declaration
// ============================================

// Interface: contract ที่ class ต้อง implement
public interface IAnimal
{
    string Name { get; }
    int Age { get; }
    void MakeSound();
    
    // Default implementation (C# 8+)
    string GetInfo() => $"{Name} (age: {Age})";
}

public interface IMovable
{
    double Speed { get; }
    void Move(double distance);
    void Stop();
}

public interface ISwimmable : IMovable
{
    double SwimSpeed { get; }
    void Swim(double distance);
}

// ============================================
// Multiple Interface Implementation
// ============================================

public class Duck : IAnimal, IMovable, ISwimmable
{
    public string Name { get; }
    public int Age { get; }
    public double Speed => 5.0; // km/h walking
    public double SwimSpeed => 3.0; // km/h swimming
    
    private double _distanceTraveled;
    
    public Duck(string name, int age)
    {
        Name = name;
        Age = age;
    }
    
    public void MakeSound() => Console.WriteLine($"{Name}: Quack!");
    
    public void Move(double distance)
    {
        _distanceTraveled += distance;
        Console.WriteLine($"{Name} walks {distance}km");
    }
    
    public void Stop() => Console.WriteLine($"{Name} stops");
    
    public void Swim(double distance)
    {
        _distanceTraveled += distance;
        Console.WriteLine($"{Name} swims {distance}km at {SwimSpeed}km/h");
    }
    
    // สามารถ override default interface method ได้
    string IAnimal.GetInfo() => $"Duck: {Name} (age: {Age})";
}

var duck = new Duck("Donald", 3);
duck.MakeSound();
duck.Move(2.5);
duck.Swim(1.0);

// Polymorphism ผ่าน interface
IAnimal animal = duck;
Console.WriteLine(animal.GetInfo()); // Duck: Donald (age: 3)

ISwimmable swimmer = duck;
swimmer.Swim(0.5);

// ============================================
// Interface ที่ใช้บ่อยใน C#
// ============================================

// IDisposable - สำหรับ resource management
// IEnumerable<T> - สำหรับ iteration
// IComparable<T> - สำหรับ sorting
// IEquatable<T> - สำหรับ equality
// INotifyPropertyChanged - สำหรับ MVVM data binding

// ============================================
// Repository Interface Pattern
// ============================================

public interface IRepository<T, TId>
{
    Task<T?> GetByIdAsync(TId id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Func<T, bool> predicate);
    Task<T> AddAsync(T entity);
    Task<T> UpdateAsync(T entity);
    Task<bool> DeleteAsync(TId id);
    Task<int> CountAsync();
}

public interface IUserRepository : IRepository<User2, int>
{
    Task<User2?> GetByEmailAsync(string email);
    Task<IEnumerable<User2>> GetActiveUsersAsync();
}

public class User2
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public bool IsActive { get; set; }
}

// In-memory implementation
public class InMemoryUserRepository : IUserRepository
{
    private readonly List<User2> _users = new();
    private int _nextId = 1;
    
    public Task<User2?> GetByIdAsync(int id)
        => Task.FromResult(_users.FirstOrDefault(u => u.Id == id));
    
    public Task<IEnumerable<User2>> GetAllAsync()
        => Task.FromResult<IEnumerable<User2>>(_users.ToList());
    
    public Task<IEnumerable<User2>> FindAsync(Func<User2, bool> predicate)
        => Task.FromResult<IEnumerable<User2>>(_users.Where(predicate).ToList());
    
    public Task<User2> AddAsync(User2 user)
    {
        user.Id = _nextId++;
        _users.Add(user);
        return Task.FromResult(user);
    }
    
    public Task<User2> UpdateAsync(User2 user)
    {
        int idx = _users.FindIndex(u => u.Id == user.Id);
        if (idx >= 0) _users[idx] = user;
        return Task.FromResult(user);
    }
    
    public Task<bool> DeleteAsync(int id)
    {
        int removed = _users.RemoveAll(u => u.Id == id);
        return Task.FromResult(removed > 0);
    }
    
    public Task<int> CountAsync() => Task.FromResult(_users.Count);
    
    public Task<User2?> GetByEmailAsync(string email)
        => Task.FromResult(_users.FirstOrDefault(u => 
            u.Email.Equals(email, StringComparison.OrdinalIgnoreCase)));
    
    public Task<IEnumerable<User2>> GetActiveUsersAsync()
        => Task.FromResult<IEnumerable<User2>>(_users.Where(u => u.IsActive).ToList());
}
```

---

## Step 75: Covariance และ Contravariance

```csharp
// ============================================
// Covariance (out) - สามารถใช้ type ที่ specific กว่าได้
// ============================================

// IEnumerable<T> เป็น covariant
IEnumerable<Dog> dogs = new List<Dog>
{
    new Dog("Buddy", 2, "Lab"),
    new Dog("Max", 3, "Poodle")
};

// covariance: สามารถ assign IEnumerable<Dog> ให้ IEnumerable<Animal>
IEnumerable<Animal> animals2 = dogs; // OK เพราะ out T
foreach (Animal a in animals2)
    Console.WriteLine(a.Name);

// ============================================
// Contravariance (in) - สามารถใช้ type ที่ general กว่าได้
// ============================================

// IComparer<T> เป็น contravariant
var animalComparer = Comparer<Animal>.Create((a, b) => 
    string.Compare(a.Name, b.Name));

// contravariance: สามารถใช้ IComparer<Animal> กับ List<Dog>
var dogList = new List<Dog>
{
    new Dog("Zeus", 1, "German Shepherd"),
    new Dog("Apollo", 2, "Husky"),
    new Dog("Buddy", 3, "Lab")
};

// dogList.Sort(animalComparer); // OK เพราะ in T

// ============================================
// Custom Covariant Interface
// ============================================

public interface IProducer<out T>
{
    T Produce();
    IEnumerable<T> ProduceMany(int count);
}

public interface IConsumer<in T>
{
    void Consume(T item);
    void ConsumeMany(IEnumerable<T> items);
}

public class DogProducer : IProducer<Dog>
{
    public Dog Produce() => new Dog("New Dog", 0, "Mixed");
    
    public IEnumerable<Dog> ProduceMany(int count)
    {
        for (int i = 0; i < count; i++)
            yield return new Dog($"Dog {i}", i, "Mixed");
    }
}

// covariant: IProducer<Dog> สามารถ assign ให้ IProducer<Animal>
IProducer<Dog> dogProducer = new DogProducer();
IProducer<Animal> animalProducer = dogProducer; // OK

Animal newAnimal = animalProducer.Produce(); // ได้ Dog
```

---

## Step 76: Abstract Factory Pattern

```csharp
// ============================================
// Abstract Factory
// ============================================

// สำหรับสร้าง UI ที่ต่างกันตาม Platform
public abstract class UIFactory
{
    public abstract IButton CreateButton(string text);
    public abstract ITextField CreateTextField(string placeholder);
    public abstract ILabel CreateLabel(string text);
    
    // Factory method
    public static UIFactory GetFactory(string platform) => platform switch
    {
        "iOS"     => new IOSFactory(),
        "Android" => new AndroidFactory(),
        "Windows" => new WindowsFactory(),
        _         => new AndroidFactory()
    };
}

public interface IButton
{
    string Text { get; }
    void Click();
    void Render();
}

public interface ITextField
{
    string Placeholder { get; }
    string Value { get; set; }
    void Render();
}

public interface ILabel
{
    string Text { get; }
    void Render();
}

// iOS Implementation
public class IOSFactory : UIFactory
{
    public override IButton CreateButton(string text) => new IOSButton(text);
    public override ITextField CreateTextField(string placeholder) => new IOSTextField(placeholder);
    public override ILabel CreateLabel(string text) => new IOSLabel(text);
}

public class IOSButton : IButton
{
    public string Text { get; }
    public IOSButton(string text) => Text = text;
    public void Click() => Console.WriteLine($"[iOS] Button '{Text}' tapped");
    public void Render() => Console.WriteLine($"[iOS] Rendering button: [{Text}]");
}

public class IOSTextField : ITextField
{
    public string Placeholder { get; }
    public string Value { get; set; } = string.Empty;
    public IOSTextField(string placeholder) => Placeholder = placeholder;
    public void Render() => Console.WriteLine($"[iOS] TextField: _{Value}_ (placeholder: {Placeholder})");
}

public class IOSLabel : ILabel
{
    public string Text { get; }
    public IOSLabel(string text) => Text = text;
    public void Render() => Console.WriteLine($"[iOS] Label: {Text}");
}

// Android Implementation
public class AndroidFactory : UIFactory
{
    public override IButton CreateButton(string text) => new AndroidButton(text);
    public override ITextField CreateTextField(string placeholder) => new AndroidTextField(placeholder);
    public override ILabel CreateLabel(string text) => new AndroidLabel(text);
}

public class AndroidButton : IButton
{
    public string Text { get; }
    public AndroidButton(string text) => Text = text;
    public void Click() => Console.WriteLine($"[Android] Button '{Text}' clicked");
    public void Render() => Console.WriteLine($"[Android] Rendering button: <{Text}>");
}

public class AndroidTextField : ITextField
{
    public string Placeholder { get; }
    public string Value { get; set; } = string.Empty;
    public AndroidTextField(string placeholder) => Placeholder = placeholder;
    public void Render() => Console.WriteLine($"[Android] EditText: [{Value}] hint: {Placeholder}");
}

public class AndroidLabel : ILabel
{
    public string Text { get; }
    public AndroidLabel(string text) => Text = text;
    public void Render() => Console.WriteLine($"[Android] TextView: {Text}");
}

public class WindowsFactory : UIFactory
{
    public override IButton CreateButton(string text) => new AndroidButton(text); // reuse
    public override ITextField CreateTextField(string placeholder) => new AndroidTextField(placeholder);
    public override ILabel CreateLabel(string text) => new AndroidLabel(text);
}

// ใช้งาน
void RenderLoginForm(UIFactory factory)
{
    var titleLabel = factory.CreateLabel("เข้าสู่ระบบ");
    var usernameField = factory.CreateTextField("ชื่อผู้ใช้");
    var passwordField = factory.CreateTextField("รหัสผ่าน");
    var loginButton = factory.CreateButton("เข้าสู่ระบบ");
    var cancelButton = factory.CreateButton("ยกเลิก");
    
    titleLabel.Render();
    usernameField.Render();
    passwordField.Render();
    loginButton.Render();
    cancelButton.Render();
}

Console.WriteLine("=== iOS Form ===");
RenderLoginForm(UIFactory.GetFactory("iOS"));

Console.WriteLine("\n=== Android Form ===");
RenderLoginForm(UIFactory.GetFactory("Android"));
```

---

## Step 77: Composite Pattern

```csharp
// ============================================
// Composite Pattern - Tree Structure
// ============================================

// สำหรับ file system, UI hierarchy, organization structure

public abstract class FileSystemItem
{
    public string Name { get; }
    public string Path { get; protected set; }
    
    protected FileSystemItem(string name, string path)
    {
        Name = name;
        Path = path;
    }
    
    public abstract long GetSize();
    public abstract void Print(int depth = 0);
    
    protected static string GetIndent(int depth) 
        => new string(' ', depth * 2);
}

public class File2 : FileSystemItem
{
    public long Size { get; }
    public string Extension { get; }
    
    public File2(string name, string path, long size) 
        : base(name, path)
    {
        Size = size;
        Extension = System.IO.Path.GetExtension(name).ToLower();
    }
    
    public override long GetSize() => Size;
    
    public override void Print(int depth = 0)
    {
        string indent = GetIndent(depth);
        string sizeStr = FormatSize(Size);
        Console.WriteLine($"{indent}📄 {Name} ({sizeStr})");
    }
    
    private static string FormatSize(long bytes)
    {
        string[] units = { "B", "KB", "MB", "GB" };
        double size = bytes;
        int unit = 0;
        while (size >= 1024 && unit < units.Length - 1)
        {
            size /= 1024;
            unit++;
        }
        return $"{size:F1} {units[unit]}";
    }
}

public class Directory2 : FileSystemItem
{
    private readonly List<FileSystemItem> _items = new();
    
    public IReadOnlyList<FileSystemItem> Items => _items.AsReadOnly();
    
    public Directory2(string name, string path) : base(name, path) { }
    
    public void Add(FileSystemItem item)
    {
        _items.Add(item);
    }
    
    public void Remove(string name)
    {
        _items.RemoveAll(i => i.Name == name);
    }
    
    public FileSystemItem? Find(string name)
    {
        return _items.FirstOrDefault(i => i.Name == name) ??
               _items.OfType<Directory2>()
                     .Select(d => d.Find(name))
                     .FirstOrDefault(i => i != null);
    }
    
    public override long GetSize() => _items.Sum(i => i.GetSize());
    
    public override void Print(int depth = 0)
    {
        string indent = GetIndent(depth);
        Console.WriteLine($"{indent}📁 {Name}/ ({FormatCount()})");
        
        foreach (var item in _items.OrderBy(i => i is Directory2 ? 0 : 1)
                                   .ThenBy(i => i.Name))
        {
            item.Print(depth + 1);
        }
    }
    
    private string FormatCount()
    {
        int dirs = _items.OfType<Directory2>().Count();
        int files = _items.OfType<File2>().Count();
        return $"{dirs} folders, {files} files";
    }
}

// ทดสอบ
var root = new Directory2("MyProject", "/");
var src = new Directory2("src", "/src");
var tests = new Directory2("tests", "/tests");

src.Add(new File2("Program.cs", "/src/Program.cs", 2048));
src.Add(new File2("App.cs", "/src/App.cs", 1536));

var models = new Directory2("Models", "/src/Models");
models.Add(new File2("User.cs", "/src/Models/User.cs", 512));
models.Add(new File2("Product.cs", "/src/Models/Product.cs", 768));
src.Add(models);

tests.Add(new File2("UserTests.cs", "/tests/UserTests.cs", 1024));

root.Add(src);
root.Add(tests);
root.Add(new File2("README.md", "/README.md", 4096));

root.Print();
Console.WriteLine($"\nTotal size: {root.GetSize():N0} bytes");

var found = root.Find("User.cs");
Console.WriteLine($"Found: {found?.Path}");
```

---

## Step 78: Strategy Pattern

```csharp
// ============================================
// Strategy Pattern
// ============================================

// Define algorithm family
public interface ISortStrategy<T>
{
    void Sort(List<T> items);
    string Name { get; }
}

public class BubbleSort<T> : ISortStrategy<T> where T : IComparable<T>
{
    public string Name => "Bubble Sort";
    
    public void Sort(List<T> items)
    {
        int n = items.Count;
        for (int i = 0; i < n - 1; i++)
        {
            for (int j = 0; j < n - i - 1; j++)
            {
                if (items[j].CompareTo(items[j + 1]) > 0)
                    (items[j], items[j + 1]) = (items[j + 1], items[j]);
            }
        }
    }
}

public class QuickSort<T> : ISortStrategy<T> where T : IComparable<T>
{
    public string Name => "Quick Sort";
    
    public void Sort(List<T> items) => QuickSortHelper(items, 0, items.Count - 1);
    
    private static void QuickSortHelper(List<T> items, int low, int high)
    {
        if (low < high)
        {
            int pivot = Partition(items, low, high);
            QuickSortHelper(items, low, pivot - 1);
            QuickSortHelper(items, pivot + 1, high);
        }
    }
    
    private static int Partition(List<T> items, int low, int high)
    {
        T pivot = items[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++)
        {
            if (items[j].CompareTo(pivot) <= 0)
            {
                i++;
                (items[i], items[j]) = (items[j], items[i]);
            }
        }
        
        (items[i + 1], items[high]) = (items[high], items[i + 1]);
        return i + 1;
    }
}

public class Sorter<T> where T : IComparable<T>
{
    private ISortStrategy<T> _strategy;
    
    public Sorter(ISortStrategy<T> strategy) => _strategy = strategy;
    
    public void SetStrategy(ISortStrategy<T> strategy) => _strategy = strategy;
    
    public List<T> Sort(List<T> items)
    {
        var copy = new List<T>(items);
        var sw = System.Diagnostics.Stopwatch.StartNew();
        _strategy.Sort(copy);
        sw.Stop();
        Console.WriteLine($"{_strategy.Name}: {sw.ElapsedMilliseconds}ms");
        return copy;
    }
}

// ใช้งาน
var numbers = Enumerable.Range(1, 1000).OrderBy(_ => Guid.NewGuid()).ToList();

var sorter = new Sorter<int>(new BubbleSort<int>());
sorter.Sort(numbers);

sorter.SetStrategy(new QuickSort<int>());
sorter.Sort(numbers);
```

---

## Step 79: Observer Pattern

```csharp
// ============================================
// Observer Pattern
// ============================================

// Event ใน C# คือ implementation ของ Observer pattern

public class EventArgs2<T>
{
    public T Data { get; }
    public DateTime Timestamp { get; } = DateTime.Now;
    
    public EventArgs2(T data) => Data = data;
}

public class OrderService
{
    // Events
    public event EventHandler<EventArgs2<Order2>>? OrderCreated;
    public event EventHandler<EventArgs2<Order2>>? OrderUpdated;
    public event EventHandler<EventArgs2<int>>? OrderDeleted;
    
    private readonly List<Order2> _orders = new();
    
    public Order2 CreateOrder(string customerId, List<string> items)
    {
        var order = new Order2(
            _orders.Count + 1,
            customerId,
            items,
            DateTime.Now
        );
        _orders.Add(order);
        
        // Raise event
        OrderCreated?.Invoke(this, new EventArgs2<Order2>(order));
        
        return order;
    }
    
    public bool DeleteOrder(int id)
    {
        var order = _orders.FirstOrDefault(o => o.Id == id);
        if (order == null) return false;
        
        _orders.Remove(order);
        OrderDeleted?.Invoke(this, new EventArgs2<int>(id));
        return true;
    }
}

public record Order2(int Id, string CustomerId, List<string> Items, DateTime CreatedAt);

// Observers
public class EmailNotifier
{
    public void Subscribe(OrderService service)
    {
        service.OrderCreated += OnOrderCreated;
        service.OrderDeleted += OnOrderDeleted;
    }
    
    private void OnOrderCreated(object? sender, EventArgs2<Order2> e)
    {
        Console.WriteLine($"📧 Email: Order #{e.Data.Id} created for {e.Data.CustomerId}");
    }
    
    private void OnOrderDeleted(object? sender, EventArgs2<int> e)
    {
        Console.WriteLine($"📧 Email: Order #{e.Data} was deleted");
    }
}

public class AuditLogger
{
    private readonly List<string> _log = new();
    
    public void Subscribe(OrderService service)
    {
        service.OrderCreated += (_, e) => 
            Log($"ORDER_CREATED: #{e.Data.Id} by {e.Data.CustomerId}");
        service.OrderDeleted += (_, e) => 
            Log($"ORDER_DELETED: #{e.Data}");
    }
    
    private void Log(string message)
    {
        string entry = $"[{DateTime.Now:HH:mm:ss}] {message}";
        _log.Add(entry);
        Console.WriteLine($"📋 Audit: {entry}");
    }
    
    public IReadOnlyList<string> GetLog() => _log.AsReadOnly();
}

// ใช้งาน
var orderService = new OrderService();
var emailNotifier = new EmailNotifier();
var auditLogger = new AuditLogger();

emailNotifier.Subscribe(orderService);
auditLogger.Subscribe(orderService);

var order1 = orderService.CreateOrder("CUST001", new() { "Product A", "Product B" });
var order2 = orderService.CreateOrder("CUST002", new() { "Product C" });
orderService.DeleteOrder(order1.Id);
```

---

## Step 80: ตัวอย่างโปรเจกต์ - E-Commerce System

```csharp
// ============================================
// E-Commerce System ด้วย OOP
// ============================================

// Base Entity
public abstract class Entity<TId>
{
    public TId Id { get; protected set; }
    public DateTime CreatedAt { get; } = DateTime.Now;
    public DateTime UpdatedAt { get; protected set; } = DateTime.Now;
    
    protected Entity(TId id) => Id = id;
    
    protected void MarkUpdated() => UpdatedAt = DateTime.Now;
}

// Product Hierarchy
public abstract class ProductBase : Entity<int>
{
    public string Name { get; protected set; }
    public string Description { get; protected set; }
    public decimal Price { get; protected set; }
    public int Stock { get; protected set; }
    
    protected ProductBase(int id, string name, decimal price) 
        : base(id)
    {
        Name = name;
        Description = string.Empty;
        Price = price;
        Stock = 0;
    }
    
    public abstract decimal GetDiscountedPrice();
    
    public virtual bool CanPurchase(int quantity) => Stock >= quantity;
    
    public virtual void ReduceStock(int quantity)
    {
        if (!CanPurchase(quantity))
            throw new InvalidOperationException("Insufficient stock");
        Stock -= quantity;
        MarkUpdated();
    }
    
    public void AddStock(int quantity)
    {
        if (quantity <= 0) throw new ArgumentException("Quantity must be positive");
        Stock += quantity;
        MarkUpdated();
    }
}

public class PhysicalProduct : ProductBase
{
    public double Weight { get; } // kg
    public string Dimensions { get; } // "LxWxH cm"
    
    public PhysicalProduct(int id, string name, decimal price, double weight)
        : base(id, name, price)
    {
        Weight = weight;
        Dimensions = string.Empty;
    }
    
    public override decimal GetDiscountedPrice()
    {
        // No automatic discount for physical products
        return Price;
    }
    
    public decimal CalculateShipping(string destination)
    {
        // Simplified shipping calculation
        return destination == "Bangkok" ? 50m : 100m + (decimal)(Weight * 10);
    }
}

public class DigitalProduct : ProductBase
{
    public string DownloadUrl { get; private set; }
    public long FileSizeBytes { get; }
    
    public DigitalProduct(int id, string name, decimal price, long fileSizeBytes)
        : base(id, name, price)
    {
        FileSizeBytes = fileSizeBytes;
        DownloadUrl = string.Empty;
        Stock = int.MaxValue; // Digital products don't run out
    }
    
    public void SetDownloadUrl(string url)
    {
        DownloadUrl = url;
        MarkUpdated();
    }
    
    public override decimal GetDiscountedPrice() => Price * 0.9m; // 10% auto discount
    
    public override bool CanPurchase(int quantity) => true; // Always available
    
    public override void ReduceStock(int quantity) { } // No-op for digital
}

public class BundleProduct : ProductBase
{
    private readonly List<ProductBase> _products;
    
    public IReadOnlyList<ProductBase> Products => _products.AsReadOnly();
    
    public BundleProduct(int id, string name, List<ProductBase> products)
        : base(id, name, products.Sum(p => p.Price) * 0.8m) // 20% bundle discount
    {
        _products = products;
    }
    
    public override decimal GetDiscountedPrice() => Price; // Already discounted
    
    public override bool CanPurchase(int quantity)
        => _products.All(p => p.CanPurchase(quantity));
    
    public override void ReduceStock(int quantity)
    {
        foreach (var product in _products)
            product.ReduceStock(quantity);
    }
}

// Order System
public class OrderItem
{
    public ProductBase Product { get; }
    public int Quantity { get; }
    public decimal UnitPrice { get; }
    
    public decimal Subtotal => UnitPrice * Quantity;
    
    public OrderItem(ProductBase product, int quantity)
    {
        Product = product;
        Quantity = quantity;
        UnitPrice = product.GetDiscountedPrice();
    }
}

public enum OrderStatus { Pending, Confirmed, Processing, Shipped, Delivered, Cancelled }

public class Order3 : Entity<string>
{
    private readonly List<OrderItem> _items = new();
    
    public string CustomerId { get; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    
    public decimal Subtotal => _items.Sum(i => i.Subtotal);
    public decimal Tax => Subtotal * 0.07m;
    public decimal Total => Subtotal + Tax;
    
    public Order3(string customerId) 
        : base($"ORD{DateTime.Now:yyyyMMddHHmmss}{new Random().Next(1000, 9999)}")
    {
        CustomerId = customerId;
        Status = OrderStatus.Pending;
    }
    
    public void AddItem(ProductBase product, int quantity)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException("Cannot modify confirmed order");
        
        if (!product.CanPurchase(quantity))
            throw new InvalidOperationException($"Insufficient stock for {product.Name}");
        
        var existingItem = _items.FirstOrDefault(i => i.Product.Id == product.Id);
        if (existingItem != null)
            _items.Remove(existingItem);
        
        _items.Add(new OrderItem(product, quantity));
        MarkUpdated();
    }
    
    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException($"Cannot confirm order in {Status} status");
        
        if (!_items.Any())
            throw new InvalidOperationException("Cannot confirm empty order");
        
        // Reserve stock
        foreach (var item in _items)
            item.Product.ReduceStock(item.Quantity);
        
        Status = OrderStatus.Confirmed;
        MarkUpdated();
    }
    
    public void Cancel()
    {
        if (Status is OrderStatus.Delivered or OrderStatus.Cancelled)
            throw new InvalidOperationException($"Cannot cancel order in {Status} status");
        
        // Restore stock if confirmed
        if (Status == OrderStatus.Confirmed)
            foreach (var item in _items)
                item.Product.AddStock(item.Quantity);
        
        Status = OrderStatus.Cancelled;
        MarkUpdated();
    }
    
    public void PrintReceipt()
    {
        Console.WriteLine($"\n{'=', 45}");
        Console.WriteLine($"Order ID: {Id}");
        Console.WriteLine($"Customer: {CustomerId}");
        Console.WriteLine($"Status: {Status}");
        Console.WriteLine($"Date: {CreatedAt:dd/MM/yyyy HH:mm}");
        Console.WriteLine(new string('-', 45));
        
        foreach (var item in _items)
        {
            Console.WriteLine($"{item.Product.Name,-25} {item.Quantity,3} × {item.UnitPrice,8:C} = {item.Subtotal,10:C}");
        }
        
        Console.WriteLine(new string('-', 45));
        Console.WriteLine($"{"Subtotal:",-35} {Subtotal,10:C}");
        Console.WriteLine($"{"Tax (7%):",-35} {Tax,10:C}");
        Console.WriteLine($"{"Total:",-35} {Total,10:C}");
        Console.WriteLine(new string('=', 45));
    }
}

// ทดสอบ
var phone = new PhysicalProduct(1, "iPhone 16 Pro", 45900m, 0.227);
phone.AddStock(50);

var app = new DigitalProduct(2, "Pro App License", 990m, 52_428_800);
app.SetDownloadUrl("https://apps.apple.com/...");

var bundle = new BundleProduct(3, "iPhone + App Bundle", 
    new List<ProductBase> { phone, app });

var order = new Order3("CUST001");
order.AddItem(phone, 1);
order.AddItem(app, 2);
order.Confirm();
order.PrintReceipt();

Console.WriteLine($"\nPhone stock: {phone.Stock}"); // 49 (reduced by 1)
```

---

## สรุป Part 08

ในส่วนนี้เราได้เรียนรู้:

1. **Inheritance** - Base/Derived classes, base keyword
2. **Abstract Classes** - Abstract methods, Template method
3. **Sealed** - ป้องกัน inheritance, sealed methods
4. **new vs override** - Polymorphism vs hiding
5. **Interfaces** - Multiple implementation, Covariance
6. **Abstract Factory** - Platform-specific UI
7. **Composite Pattern** - Tree structures
8. **Strategy Pattern** - Interchangeable algorithms
9. **Observer Pattern** - Events and notifications
10. **Real Project** - E-Commerce with full OOP hierarchy

## ขั้นต่อไป

ใน Part 09 เราจะเรียนรู้เกี่ยวกับ:
- Interfaces ขั้นสูง
- Abstract Classes vs Interfaces
- Default interface implementations
- Interface segregation
- Dependency Inversion

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 08 | Steps 71-80*
