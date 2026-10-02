# Part 02: ตัวแปร ชนิดข้อมูล และ Operators
## Steps 11-20: ทำความเข้าใจข้อมูลใน C#

---

## Step 11: ชนิดข้อมูลพื้นฐาน (Primitive Data Types)

C# มีชนิดข้อมูลพื้นฐาน 2 ประเภทหลัก:

### Value Types (เก็บค่าโดยตรง)

```csharp
// ============================================
// INTEGER TYPES - จำนวนเต็ม
// ============================================

byte    byteVal  = 255;          // 0 ถึง 255 (8-bit unsigned)
sbyte   sbyteVal = -128;         // -128 ถึง 127 (8-bit signed)
short   shortVal = -32768;       // -32,768 ถึง 32,767 (16-bit)
ushort  ushortVal = 65535;       // 0 ถึง 65,535 (16-bit unsigned)
int     intVal   = -2147483648;  // -2,147,483,648 ถึง 2,147,483,647 (32-bit)
uint    uintVal  = 4294967295;   // 0 ถึง 4,294,967,295 (32-bit unsigned)
long    longVal  = -9223372036854775808L; // 64-bit signed
ulong   ulongVal = 18446744073709551615UL; // 64-bit unsigned

// ============================================
// FLOATING POINT TYPES - จำนวนทศนิยม
// ============================================

float   floatVal  = 3.14159f;    // 7 digits precision (32-bit)
double  doubleVal = 3.14159265358979; // 15-16 digits precision (64-bit)
decimal decimalVal = 3.14159265358979323846M; // 28-29 digits (128-bit, เหมาะกับเงิน)

// ============================================
// OTHER TYPES
// ============================================

bool    boolVal = true;          // true หรือ false
char    charVal = 'A';           // Unicode character (16-bit)

// ตัวอย่างการแสดงขนาดของแต่ละชนิด
Console.WriteLine($"int   : {sizeof(int)} bytes, range: {int.MinValue} to {int.MaxValue}");
Console.WriteLine($"long  : {sizeof(long)} bytes, range: {long.MinValue} to {long.MaxValue}");
Console.WriteLine($"float : {sizeof(float)} bytes");
Console.WriteLine($"double: {sizeof(double)} bytes");
Console.WriteLine($"decimal: {sizeof(decimal)} bytes");
```

### ตารางชนิดข้อมูล

| C# Type | .NET Type | Size | Range |
|---------|-----------|------|-------|
| `bool` | Boolean | 1 byte | true/false |
| `byte` | Byte | 1 byte | 0 - 255 |
| `sbyte` | SByte | 1 byte | -128 - 127 |
| `short` | Int16 | 2 bytes | -32,768 - 32,767 |
| `ushort` | UInt16 | 2 bytes | 0 - 65,535 |
| `int` | Int32 | 4 bytes | -2.1B - 2.1B |
| `uint` | UInt32 | 4 bytes | 0 - 4.3B |
| `long` | Int64 | 8 bytes | -9.2Q - 9.2Q |
| `ulong` | UInt64 | 8 bytes | 0 - 18.4Q |
| `float` | Single | 4 bytes | ±1.5E-45 - ±3.4E38 |
| `double` | Double | 8 bytes | ±5E-324 - ±1.7E308 |
| `decimal` | Decimal | 16 bytes | ±1E-28 - ±7.9E28 |
| `char` | Char | 2 bytes | U+0000 - U+FFFF |

---

## Step 12: ตัวแปร (Variables)

```csharp
// ============================================
// การประกาศตัวแปร
// ============================================

// รูปแบบ: <data_type> <variable_name> = <value>;

// ประกาศและกำหนดค่า
int age = 25;
string name = "สมชาย";
double price = 199.99;
bool isActive = true;

// ประกาศโดยไม่กำหนดค่า (ต้องกำหนดก่อนใช้)
int score;
score = 100; // กำหนดค่าภายหลัง

// ประกาศหลายตัวพร้อมกัน
int x = 1, y = 2, z = 3;

// var - Type Inference (C# จะ detect type เอง)
var message = "Hello";        // string
var number  = 42;             // int
var pi      = 3.14;           // double
var flag    = true;           // bool
var today   = DateTime.Now;   // DateTime

// ตรวจสอบ type ของ var
Console.WriteLine(message.GetType().Name);  // String
Console.WriteLine(number.GetType().Name);   // Int32
Console.WriteLine(pi.GetType().Name);       // Double

// ============================================
// ค่าคงที่ (Constants)
// ============================================

const double PI = 3.14159265358979;
const int MAX_SIZE = 100;
const string APP_NAME = "MyApp";

// PI = 4; // ERROR: ไม่สามารถเปลี่ยนค่า const ได้

// readonly - เปลี่ยนได้แค่ใน constructor
public class Circle
{
    private readonly double _radius;
    
    public Circle(double radius)
    {
        _radius = radius; // OK: กำหนดใน constructor
    }
    
    public double GetArea()
    {
        // _radius = 5; // ERROR: ไม่สามารถเปลี่ยนได้นอก constructor
        return PI * _radius * _radius;
    }
}
```

---

## Step 13: String - ชนิดข้อมูลข้อความ

```csharp
// ============================================
// การสร้าง String
// ============================================

string str1 = "Hello, World!";
string str2 = "สวัสดี โลก!";

// Verbatim string - ไม่ต้อง escape characters
string path = @"C:\Users\User\Documents\file.txt";
string multiline = @"บรรทัดที่ 1
บรรทัดที่ 2
บรรทัดที่ 3";

// Raw string literal (C# 11+)
string rawString = """
    Hello
    World
    """;

// ============================================
// String Interpolation
// ============================================

string firstName = "สมชาย";
string lastName = "ใจดี";
int birthYear = 1995;
int currentYear = DateTime.Now.Year;
int age = currentYear - birthYear;

// วิธีเก่า: String.Format
string msg1 = string.Format("ชื่อ: {0} {1}, อายุ: {2} ปี", firstName, lastName, age);

// วิธีใหม่: String Interpolation ($"")
string msg2 = $"ชื่อ: {firstName} {lastName}, อายุ: {age} ปี";

// Expressions ใน interpolation
string msg3 = $"ราคา: {1500 * 1.07:N2} บาท (รวม VAT)";
string msg4 = $"วันที่: {DateTime.Now:dd/MM/yyyy HH:mm:ss}";

Console.WriteLine(msg1);
Console.WriteLine(msg2);
Console.WriteLine(msg3);
Console.WriteLine(msg4);

// ============================================
// String Methods
// ============================================

string text = "  Hello, World!  ";

// การตัด whitespace
Console.WriteLine(text.Trim());         // "Hello, World!"
Console.WriteLine(text.TrimStart());    // "Hello, World!  "
Console.WriteLine(text.TrimEnd());      // "  Hello, World!"

// การเปลี่ยนขนาดตัวอักษร
Console.WriteLine(text.ToUpper());      // "  HELLO, WORLD!  "
Console.WriteLine(text.ToLower());      // "  hello, world!  "

// การค้นหา
Console.WriteLine(text.Contains("World"));   // True
Console.WriteLine(text.StartsWith("  He")); // True
Console.WriteLine(text.EndsWith("!  "));    // True
Console.WriteLine(text.IndexOf("World"));   // 9

// การแทนที่
Console.WriteLine(text.Replace("World", "C#")); // "  Hello, C#!  "

// การแบ่ง
string csv = "A,B,C,D,E";
string[] parts = csv.Split(',');
foreach (string part in parts)
{
    Console.Write($"{part} ");  // A B C D E
}

// การรวม
string[] words = { "Hello", "World", "C#" };
Console.WriteLine(string.Join(", ", words)); // Hello, World, C#

// การตรวจสอบ
Console.WriteLine(string.IsNullOrEmpty(""));       // True
Console.WriteLine(string.IsNullOrWhiteSpace("  ")); // True

// Substring
string str = "Hello, World!";
Console.WriteLine(str.Substring(7));      // "World!"
Console.WriteLine(str.Substring(7, 5));   // "World"

// String Length
Console.WriteLine(str.Length); // 13

// String Comparison
string s1 = "hello";
string s2 = "HELLO";
Console.WriteLine(s1 == s2);                               // False
Console.WriteLine(s1.Equals(s2, StringComparison.OrdinalIgnoreCase)); // True
Console.WriteLine(string.Compare(s1, s2, true));          // 0 (equal, case-insensitive)
```

---

## Step 14: Number Formatting

```csharp
// ============================================
// Number Literals
// ============================================

// Integer literals
int decimal_   = 1000000;      // ปกติ
int hexadecimal = 0x00_FF_00;  // Hexadecimal (สีเขียว RGB)
int binary      = 0b1000_0000; // Binary
int separated   = 1_000_000;   // Digit separator (อ่านง่ายขึ้น)

Console.WriteLine(hexadecimal); // 65280
Console.WriteLine(binary);      // 128

// Floating point literals
float f   = 1.5f;      // f suffix สำหรับ float
double d  = 1.5;       // default เป็น double
double d2 = 1.5d;      // d suffix สำหรับ double
decimal m = 1.5m;      // m suffix สำหรับ decimal

// Scientific notation
double avogadro = 6.022e23;  // 6.022 × 10^23
double electron = 1.6e-19;   // 1.6 × 10^-19

// ============================================
// Number Formatting
// ============================================

double amount = 1234567.89;

// Standard format specifiers
Console.WriteLine(amount.ToString("C"));      // ฿1,234,567.89 (Currency)
Console.WriteLine(amount.ToString("N2"));     // 1,234,567.89 (Number with 2 decimals)
Console.WriteLine(amount.ToString("F4"));     // 1234567.8900 (Fixed point)
Console.WriteLine(amount.ToString("E2"));     // 1.23E+006 (Exponential)
Console.WriteLine(amount.ToString("P"));      // 123,456,789.00% (Percent)
Console.WriteLine(amount.ToString("G"));      // 1234567.89 (General)

// Custom format
Console.WriteLine(amount.ToString("#,##0.00")); // 1,234,567.89
Console.WriteLine(amount.ToString("000000"));   // ไม่ถูก format ถ้าใหญ่กว่า pattern

// Format ใน string interpolation
Console.WriteLine($"{amount:C}");    // ฿1,234,567.89
Console.WriteLine($"{amount:N2}");   // 1,234,567.89
Console.WriteLine($"{amount:F2}");   // 1234567.89
Console.WriteLine($"{amount:0.00}"); // 1234567.89

// สำหรับเงินบาทไทย
System.Globalization.CultureInfo thaiCulture = new("th-TH");
Console.WriteLine(amount.ToString("C", thaiCulture)); // ฿1,234,567.89

// Hexadecimal
int colorValue = 255;
Console.WriteLine(colorValue.ToString("X"));  // FF
Console.WriteLine(colorValue.ToString("X6")); // 0000FF
Console.WriteLine($"{colorValue:X2}");         // FF
```

---

## Step 15: Type Conversion

```csharp
// ============================================
// Implicit Conversion (Widening) - ไม่มีการสูญเสียข้อมูล
// ============================================

byte  b = 100;
short s = b;    // byte → short (OK)
int   i = s;    // short → int (OK)
long  l = i;    // int → long (OK)
float f = l;    // long → float (อาจสูญเสีย precision เล็กน้อย)
double dd = f;  // float → double (OK)

Console.WriteLine($"byte: {b}, short: {s}, int: {i}, long: {l}");

// ============================================
// Explicit Conversion (Narrowing) - อาจสูญเสียข้อมูล
// ============================================

double pi = 3.14159;
int truncated = (int)pi;      // 3 (ตัดทศนิยมทิ้ง)
Console.WriteLine(truncated); // 3

long bigNumber = 9_999_999_999L;
int overflow = (int)bigNumber; // Overflow! ค่าอาจผิดพลาด
Console.WriteLine(overflow);   // 1410065407 (ผิดพลาด!)

// ============================================
// Safe Conversion ด้วย Convert class
// ============================================

string numStr = "42";
int parsed1 = Convert.ToInt32(numStr);      // 42
double parsed2 = Convert.ToDouble("3.14"); // 3.14
bool parsed3 = Convert.ToBoolean(1);       // True
string str = Convert.ToString(123);        // "123"

// Convert.ToXxx จะ throw exception ถ้า invalid
try
{
    int invalid = Convert.ToInt32("abc"); // FormatException!
}
catch (FormatException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}

// ============================================
// Safe Parsing ด้วย TryParse
// ============================================

// int.TryParse
string input1 = "123";
string input2 = "abc";

if (int.TryParse(input1, out int result1))
    Console.WriteLine($"Parsed: {result1}");    // Parsed: 123
else
    Console.WriteLine("Cannot parse");

if (int.TryParse(input2, out int result2))
    Console.WriteLine($"Parsed: {result2}");
else
    Console.WriteLine("Cannot parse 'abc'"); // Cannot parse 'abc'

// double.TryParse
if (double.TryParse("3.14", out double pi2))
    Console.WriteLine($"Pi = {pi2}");

// ============================================
// Checked Arithmetic (ป้องกัน Overflow)
// ============================================

try
{
    checked
    {
        int max = int.MaxValue;
        int overflow2 = max + 1; // OverflowException!
    }
}
catch (OverflowException)
{
    Console.WriteLine("Integer overflow detected!");
}

// ============================================
// Boxing และ Unboxing
// ============================================

// Boxing: value type → object
int num = 42;
object boxed = num;     // Boxing (ช้า เพราะสร้าง object ใหม่)
Console.WriteLine(boxed.GetType()); // System.Int32

// Unboxing: object → value type
int unboxed = (int)boxed; // Unboxing (ต้องระวัง InvalidCastException)
Console.WriteLine(unboxed); // 42

// ต้อง cast ให้ถูก type!
try
{
    double wrong = (double)boxed; // InvalidCastException!
}
catch (InvalidCastException)
{
    Console.WriteLine("Cannot unbox int as double");
}
```

---

## Step 16: Operators (ตัวดำเนินการ)

```csharp
// ============================================
// Arithmetic Operators - ตัวดำเนินการทางคณิตศาสตร์
// ============================================

int a = 10, b = 3;

Console.WriteLine($"a + b = {a + b}");   // 13 (บวก)
Console.WriteLine($"a - b = {a - b}");   // 7  (ลบ)
Console.WriteLine($"a * b = {a * b}");   // 30 (คูณ)
Console.WriteLine($"a / b = {a / b}");   // 3  (หาร - integer division)
Console.WriteLine($"a % b = {a % b}");   // 1  (หารเอาเศษ modulo)

// Integer vs Double division
Console.WriteLine($"10 / 3  = {10 / 3}");    // 3   (integer division)
Console.WriteLine($"10 / 3.0= {10 / 3.0}");  // 3.333... (double division)
Console.WriteLine($"10.0 / 3= {10.0 / 3}");  // 3.333...

// Math functions
Console.WriteLine(Math.Pow(2, 10));  // 1024 (2^10)
Console.WriteLine(Math.Sqrt(144));   // 12 (√144)
Console.WriteLine(Math.Abs(-42));    // 42 (ค่าสัมบูรณ์)
Console.WriteLine(Math.Round(3.567, 2)); // 3.57
Console.WriteLine(Math.Floor(3.9));  // 3 (ปัดลง)
Console.WriteLine(Math.Ceiling(3.1)); // 4 (ปัดขึ้น)
Console.WriteLine(Math.Max(10, 20)); // 20
Console.WriteLine(Math.Min(10, 20)); // 10

// ============================================
// Comparison Operators - ตัวดำเนินการเปรียบเทียบ
// ============================================

int x = 5, y = 10;

Console.WriteLine($"{x} == {y}: {x == y}"); // False
Console.WriteLine($"{x} != {y}: {x != y}"); // True
Console.WriteLine($"{x} < {y}: {x < y}");   // True
Console.WriteLine($"{x} > {y}: {x > y}");   // False
Console.WriteLine($"{x} <= {y}: {x <= y}"); // True
Console.WriteLine($"{x} >= {y}: {x >= y}"); // False

// ============================================
// Logical Operators - ตัวดำเนินการตรรกศาสตร์
// ============================================

bool isAdult = true;
bool hasID = true;
bool isVip = false;

// AND - ทั้งสองต้องเป็น true
Console.WriteLine($"isAdult && hasID = {isAdult && hasID}"); // True

// OR - อย่างน้อยหนึ่งต้องเป็น true
Console.WriteLine($"isAdult || isVip = {isAdult || isVip}"); // True

// NOT - ตรงข้าม
Console.WriteLine($"!isVip = {!isVip}"); // True

// Short-circuit evaluation
bool result = false && SomeExpensiveMethod(); // SomeExpensiveMethod ไม่ถูกเรียก
bool result2 = true || SomeExpensiveMethod(); // SomeExpensiveMethod ไม่ถูกเรียก

bool SomeExpensiveMethod()
{
    Console.WriteLine("Expensive method called!");
    return true;
}

// ============================================
// Assignment Operators
// ============================================

int n = 10;
n += 5;   // n = n + 5 = 15
n -= 3;   // n = n - 3 = 12
n *= 2;   // n = n * 2 = 24
n /= 4;   // n = n / 4 = 6
n %= 4;   // n = n % 4 = 2

Console.WriteLine($"n = {n}"); // 2

// ============================================
// Increment/Decrement Operators
// ============================================

int counter = 5;
Console.WriteLine(counter++); // 5 (แสดงก่อน แล้วค่อย +1)
Console.WriteLine(counter);   // 6

Console.WriteLine(++counter); // 7 (+1 ก่อน แล้วค่อยแสดง)
Console.WriteLine(counter);   // 7

Console.WriteLine(counter--); // 7 (แสดงก่อน แล้วค่อย -1)
Console.WriteLine(counter);   // 6

Console.WriteLine(--counter); // 5 (-1 ก่อน แล้วค่อยแสดง)
Console.WriteLine(counter);   // 5
```

---

## Step 17: Null และ Nullable Types

```csharp
// ============================================
// Null คืออะไร?
// ============================================

// Reference types สามารถเป็น null ได้
string? nullableString = null;
int[]? nullableArray = null;

// Value types ปกติไม่สามารถเป็น null
// int cantBeNull = null; // ERROR!

// Nullable value types (ใส่ ? ต่อท้าย type)
int? nullableInt = null;
double? nullableDouble = null;
bool? nullableBool = null;
DateTime? nullableDate = null;

Console.WriteLine(nullableInt.HasValue);  // False
Console.WriteLine(nullableInt == null);   // True

nullableInt = 42;
Console.WriteLine(nullableInt.HasValue);  // True
Console.WriteLine(nullableInt.Value);     // 42
Console.WriteLine((int)nullableInt);      // 42 (explicit cast)

// ============================================
// Null Operators
// ============================================

// Null-coalescing operator (??)
int? possiblyNull = null;
int definitelyNotNull = possiblyNull ?? 0; // ถ้า null ใช้ 0
Console.WriteLine(definitelyNotNull); // 0

string? nullStr = null;
string result = nullStr ?? "default value";
Console.WriteLine(result); // "default value"

// Null-coalescing assignment (??=)
string? name = null;
name ??= "Unknown"; // กำหนดค่าเฉพาะเมื่อ name เป็น null
Console.WriteLine(name); // "Unknown"

name ??= "John"; // จะไม่กำหนดเพราะ name ไม่ใช่ null แล้ว
Console.WriteLine(name); // "Unknown"

// Null-conditional operator (?.)
string? maybeNull = null;
int? length = maybeNull?.Length; // ไม่ throw NullReferenceException
Console.WriteLine(length); // (null)

maybeNull = "Hello";
length = maybeNull?.Length;
Console.WriteLine(length); // 5

// Chaining null-conditional
List<string>? list = null;
int? count = list?.Count;           // null
string? first = list?[0];           // null
string? upper = list?[0]?.ToUpper(); // null

// รวมกับ ??
string safeResult = list?[0] ?? "empty";
Console.WriteLine(safeResult); // "empty"

// Null-forgiving operator (!)
// ใช้เมื่อเราแน่ใจว่าไม่ใช่ null
string? possiblyNull2 = GetSomething();
string definite = possiblyNull2!; // บอก compiler ว่าไม่ใช่ null (ระวัง!)

string? GetSomething() => "Hello";

// ============================================
// Null Checking Patterns
// ============================================

object? obj = GetObject();

// Pattern 1: if null check
if (obj != null)
{
    Console.WriteLine(obj.ToString());
}

// Pattern 2: is null check (C# 7+)
if (obj is null)
{
    Console.WriteLine("obj is null");
}

// Pattern 3: is not null (C# 9+)
if (obj is not null)
{
    Console.WriteLine($"obj is: {obj}");
}

// Pattern 4: null guard (throw if null)
string? param = null;
ArgumentNullException.ThrowIfNull(param); // throw ถ้า null

object? GetObject() => new object();
```

---

## Step 18: Bitwise Operators

```csharp
// ============================================
// Bitwise Operators
// ============================================

byte a = 0b_0110_0101; // 101 ใน decimal
byte b = 0b_0011_1010; // 58 ใน decimal

// AND (&) - ทั้งสอง bit ต้องเป็น 1
byte andResult = (byte)(a & b);
Console.WriteLine($"a & b = {andResult:b8} = {andResult}"); // 00100000 = 32

// OR (|) - อย่างน้อยหนึ่ง bit เป็น 1
byte orResult = (byte)(a | b);
Console.WriteLine($"a | b = {orResult:b8} = {orResult}"); // 01111111 = 127

// XOR (^) - bit ต่างกันเป็น 1, bit เหมือนกันเป็น 0
byte xorResult = (byte)(a ^ b);
Console.WriteLine($"a ^ b = {xorResult:b8} = {xorResult}"); // 01011111 = 95

// NOT (~) - กลับทุก bit
byte notResult = (byte)(~a);
Console.WriteLine($"~a = {notResult:b8} = {notResult}"); // 10011010 = 154

// Left Shift (<<) - เลื่อน bit ไปทางซ้าย
byte leftShift = (byte)(a << 2);
Console.WriteLine($"a << 2 = {leftShift:b8} = {leftShift}"); // 10010100 (× 4)

// Right Shift (>>) - เลื่อน bit ไปทางขวา
byte rightShift = (byte)(a >> 2);
Console.WriteLine($"a >> 2 = {rightShift:b8} = {rightShift}"); // 00011001 (÷ 4)

// ============================================
// การใช้ Bitwise ในทางปฏิบัติ
// ============================================

// Flags Enum - ใช้ bit สำหรับ multiple choices
[Flags]
enum Permissions
{
    None    = 0b_0000, // 0
    Read    = 0b_0001, // 1
    Write   = 0b_0010, // 2
    Execute = 0b_0100, // 4
    Delete  = 0b_1000, // 8
    
    // ชุดรวม
    ReadWrite   = Read | Write,         // 3
    All         = Read | Write | Execute | Delete // 15
}

// กำหนด permissions
Permissions userPerms = Permissions.Read | Permissions.Write;
Console.WriteLine($"User permissions: {userPerms}"); // Read, Write

// ตรวจสอบ permission
bool canRead    = (userPerms & Permissions.Read) != 0;
bool canExecute = (userPerms & Permissions.Execute) != 0;

Console.WriteLine($"Can read: {canRead}");     // True
Console.WriteLine($"Can execute: {canExecute}"); // False

// ใช้ HasFlag (ดีกว่า)
Console.WriteLine(userPerms.HasFlag(Permissions.Read));    // True
Console.WriteLine(userPerms.HasFlag(Permissions.Execute)); // False

// เพิ่ม/ลบ permission
userPerms |= Permissions.Execute;  // เพิ่ม Execute
userPerms &= ~Permissions.Write;   // ลบ Write

Console.WriteLine($"Updated: {userPerms}"); // Read, Execute
```

---

## Step 19: Operator Precedence และ Expressions

```csharp
// ============================================
// Operator Precedence (ลำดับความสำคัญ)
// ============================================

// สูงสุด → ต่ำสุด:
// 1. () []  .  ++ -- (postfix)
// 2. ! ~ + - ++ -- (prefix) (type)
// 3. * / %
// 4. + -
// 5. << >>
// 6. < > <= >= is as
// 7. == !=
// 8. &
// 9. ^
// 10. |
// 11. &&
// 12. ||
// 13. ??
// 14. ?:
// 15. = += -= *= /= %= &= |= ^= <<= >>= ??=

int result1 = 2 + 3 * 4;        // 14 (คูณก่อน)
int result2 = (2 + 3) * 4;      // 20 (วงเล็บก่อน)
int result3 = 10 - 3 + 2;       // 9 (ซ้ายไปขวา)
bool result4 = true || false && false; // True (&& ก่อน ||)
bool result5 = (true || false) && false; // False

Console.WriteLine($"2 + 3 * 4 = {result1}");
Console.WriteLine($"(2 + 3) * 4 = {result2}");

// ============================================
// Conditional (Ternary) Operator
// ============================================

int age = 20;
string status = age >= 18 ? "ผู้ใหญ่" : "เยาวชน";
Console.WriteLine(status); // ผู้ใหญ่

// Nested ternary (ไม่แนะนำ - ยากอ่าน)
string grade = age >= 25 ? "อาวุโส" : age >= 18 ? "ผู้ใหญ่" : "เยาวชน";

// ดีกว่าที่จะใช้ if-else สำหรับ nested conditions
string betterGrade;
if (age >= 25)
    betterGrade = "อาวุโส";
else if (age >= 18)
    betterGrade = "ผู้ใหญ่";
else
    betterGrade = "เยาวชน";

// ============================================
// Switch Expression (C# 8+)
// ============================================

int day = 3;
string dayName = day switch
{
    1 => "จันทร์",
    2 => "อังคาร",
    3 => "พุธ",
    4 => "พฤหัสบดี",
    5 => "ศุกร์",
    6 => "เสาร์",
    7 => "อาทิตย์",
    _ => "ไม่ถูกต้อง"  // default case
};
Console.WriteLine(dayName); // พุธ

// Switch expression ด้วย pattern matching
object obj = 42;
string description = obj switch
{
    int n when n < 0   => $"ลบ: {n}",
    int n when n == 0  => "ศูนย์",
    int n when n > 0   => $"บวก: {n}",
    string s           => $"ข้อความ: {s}",
    null               => "null",
    _                  => $"ชนิดอื่น: {obj.GetType().Name}"
};
Console.WriteLine(description); // บวก: 42
```

---

## Step 20: การทำงานกับ DateTime

```csharp
// ============================================
// DateTime
// ============================================

// สร้าง DateTime
DateTime now = DateTime.Now;
DateTime utcNow = DateTime.UtcNow;
DateTime today = DateTime.Today;
DateTime specific = new DateTime(2025, 12, 31, 23, 59, 59);
DateTime fromString = DateTime.Parse("2025-01-01");

Console.WriteLine($"Now: {now}");
Console.WriteLine($"UTC: {utcNow}");
Console.WriteLine($"Today: {today:dd/MM/yyyy}");
Console.WriteLine($"Specific: {specific:yyyy-MM-dd HH:mm:ss}");

// Properties
Console.WriteLine($"Year: {now.Year}");
Console.WriteLine($"Month: {now.Month}");
Console.WriteLine($"Day: {now.Day}");
Console.WriteLine($"Hour: {now.Hour}");
Console.WriteLine($"Minute: {now.Minute}");
Console.WriteLine($"DayOfWeek: {now.DayOfWeek}");
Console.WriteLine($"DayOfYear: {now.DayOfYear}");

// คำนวณ DateTime
DateTime birthday = new DateTime(1995, 6, 15);
TimeSpan age2 = DateTime.Today - birthday;
int ageYears = (int)(age2.TotalDays / 365.25);
Console.WriteLine($"Age: {ageYears} years");

// เพิ่ม/ลด DateTime
DateTime nextWeek = now.AddDays(7);
DateTime lastMonth = now.AddMonths(-1);
DateTime inTwoHours = now.AddHours(2);

Console.WriteLine($"Next week: {nextWeek:dd/MM/yyyy}");
Console.WriteLine($"Last month: {lastMonth:dd/MM/yyyy}");

// Formatting
Console.WriteLine(now.ToString("dd/MM/yyyy"));        // 02/10/2025
Console.WriteLine(now.ToString("yyyy-MM-dd"));        // 2025-10-02
Console.WriteLine(now.ToString("HH:mm:ss"));          // 14:30:00
Console.WriteLine(now.ToString("dddd, d MMMM yyyy")); // Friday, 2 October 2025
Console.WriteLine(now.ToString("d"));                  // Short date
Console.WriteLine(now.ToString("D"));                  // Long date
Console.WriteLine(now.ToString("t"));                  // Short time
Console.WriteLine(now.ToString("T"));                  // Long time
Console.WriteLine(now.ToString("f"));                  // Full date/time
Console.WriteLine(now.ToString("s"));                  // Sortable: 2025-10-02T14:30:00
Console.WriteLine(now.ToString("o"));                  // ISO 8601: 2025-10-02T14:30:00.0000000+07:00

// Comparison
DateTime date1 = new DateTime(2025, 1, 1);
DateTime date2 = new DateTime(2025, 12, 31);
Console.WriteLine(date1 < date2);   // True
Console.WriteLine(date1.CompareTo(date2)); // -1 (date1 ก่อน date2)

// DateOnly และ TimeOnly (C# 6+)
DateOnly dateOnly = DateOnly.FromDateTime(now);
TimeOnly timeOnly = TimeOnly.FromDateTime(now);
Console.WriteLine($"Date: {dateOnly}");
Console.WriteLine($"Time: {timeOnly}");
```

---

## แบบฝึกหัด Part 02

### แบบฝึกหัดที่ 2.1: เครื่องคิดเลข
สร้างเครื่องคิดเลขที่รับตัวเลข 2 ตัวและ operator แล้วคำนวณผลลัพธ์

```csharp
Console.Write("ใส่ตัวเลขที่ 1: ");
// ... รับ input และคำนวณ
```

### แบบฝึกหัดที่ 2.2: ตรวจสอบปีอธิกสุรทิน
สร้างโปรแกรมตรวจสอบว่าปีที่ระบุเป็นปีอธิกสุรทิน (Leap Year) หรือไม่
- หารด้วย 4 ลงตัว และ
- ไม่หารด้วย 100 ลงตัว หรือ หารด้วย 400 ลงตัว

### แบบฝึกหัดที่ 2.3: แปลงหน่วยอุณหภูมิ
สร้างโปรแกรมแปลงอุณหภูมิระหว่าง Celsius, Fahrenheit, Kelvin

---

## เฉลยแบบฝึกหัด Part 02

### เฉลย 2.1: เครื่องคิดเลข

```csharp
using System;

Console.WriteLine("=== เครื่องคิดเลข ===\n");

Console.Write("ใส่ตัวเลขที่ 1: ");
if (!double.TryParse(Console.ReadLine(), out double num1))
{
    Console.WriteLine("ข้อมูลไม่ถูกต้อง");
    return;
}

Console.Write("ใส่ operator (+, -, *, /, %): ");
string op = Console.ReadLine() ?? "+";

Console.Write("ใส่ตัวเลขที่ 2: ");
if (!double.TryParse(Console.ReadLine(), out double num2))
{
    Console.WriteLine("ข้อมูลไม่ถูกต้อง");
    return;
}

double result = op switch
{
    "+" => num1 + num2,
    "-" => num1 - num2,
    "*" => num1 * num2,
    "/" when num2 != 0 => num1 / num2,
    "/" => double.NaN, // หารด้วยศูนย์
    "%" when num2 != 0 => num1 % num2,
    "%" => double.NaN,
    _ => double.NaN
};

if (double.IsNaN(result))
{
    if (num2 == 0)
        Console.WriteLine("Error: หารด้วยศูนย์ไม่ได้!");
    else
        Console.WriteLine($"Error: operator '{op}' ไม่รองรับ");
}
else
{
    Console.WriteLine($"\n{num1} {op} {num2} = {result:N4}");
}
```

### เฉลย 2.2: ปีอธิกสุรทิน

```csharp
using System;

Console.Write("ใส่ปี (ค.ศ.): ");
if (!int.TryParse(Console.ReadLine(), out int year))
{
    Console.WriteLine("ปีไม่ถูกต้อง");
    return;
}

// ตรวจสอบปีอธิกสุรทิน
bool isLeapYear = (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0);

// หรือใช้ method ของ .NET
bool isLeapYear2 = DateTime.IsLeapYear(year);

if (isLeapYear)
{
    Console.WriteLine($"ปี {year} เป็นปีอธิกสุรทิน (มี 366 วัน)");
    Console.WriteLine($"เดือนกุมภาพันธ์มี 29 วัน");
}
else
{
    Console.WriteLine($"ปี {year} ไม่ใช่ปีอธิกสุรทิน (มี 365 วัน)");
    Console.WriteLine($"เดือนกุมภาพันธ์มี 28 วัน");
}
```

### เฉลย 2.3: แปลงหน่วยอุณหภูมิ

```csharp
using System;

Console.WriteLine("=== แปลงหน่วยอุณหภูมิ ===\n");
Console.WriteLine("เลือกหน่วยที่ต้องการแปลง:");
Console.WriteLine("1. Celsius → Fahrenheit, Kelvin");
Console.WriteLine("2. Fahrenheit → Celsius, Kelvin");
Console.WriteLine("3. Kelvin → Celsius, Fahrenheit");
Console.Write("\nเลือก (1-3): ");

string choice = Console.ReadLine() ?? "1";

Console.Write("ใส่ค่าอุณหภูมิ: ");
if (!double.TryParse(Console.ReadLine(), out double temp))
{
    Console.WriteLine("ค่าไม่ถูกต้อง");
    return;
}

switch (choice)
{
    case "1": // Celsius
        double fahrenheit = (temp * 9.0 / 5.0) + 32;
        double kelvin = temp + 273.15;
        Console.WriteLine($"\n{temp}°C = {fahrenheit:F2}°F = {kelvin:F2}K");
        break;
        
    case "2": // Fahrenheit
        double celsius = (temp - 32) * 5.0 / 9.0;
        double kelvin2 = celsius + 273.15;
        Console.WriteLine($"\n{temp}°F = {celsius:F2}°C = {kelvin2:F2}K");
        break;
        
    case "3": // Kelvin
        if (temp < 0)
        {
            Console.WriteLine("Error: Kelvin ต้องมีค่า ≥ 0");
            break;
        }
        double celsius2 = temp - 273.15;
        double fahrenheit2 = (celsius2 * 9.0 / 5.0) + 32;
        Console.WriteLine($"\n{temp}K = {celsius2:F2}°C = {fahrenheit2:F2}°F");
        break;
        
    default:
        Console.WriteLine("ตัวเลือกไม่ถูกต้อง");
        break;
}
```

---

## สรุป Part 02

ในส่วนนี้เราได้เรียนรู้:

1. **Primitive Data Types** - int, long, float, double, decimal, bool, char
2. **Variables** - การประกาศ, var keyword
3. **Constants** - const, readonly
4. **Strings** - การสร้าง, interpolation, methods
5. **Number Formatting** - Format specifiers
6. **Type Conversion** - Implicit, Explicit, TryParse
7. **Operators** - Arithmetic, Comparison, Logical, Assignment
8. **Null Handling** - ??, ?., ??=
9. **Bitwise Operators** - &, |, ^, ~, <<, >>
10. **DateTime** - การสร้าง, คำนวณ, formatting

## ขั้นต่อไป

ใน Part 03 เราจะเรียนรู้เกี่ยวกับ:
- if/else statements
- switch statements
- Pattern matching
- Nested conditions

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 02 | Steps 11-20*
