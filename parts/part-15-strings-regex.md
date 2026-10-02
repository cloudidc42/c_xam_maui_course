# Part 15: Strings และ Regular Expressions
## Steps 141-150: การจัดการข้อความขั้นสูง

---

## Step 141: String Fundamentals

```csharp
// ============================================
// String Immutability
// ============================================

string s1 = "Hello";
string s2 = s1; // s2 points to same string object
s1 = s1 + " World"; // creates NEW string, s1 now points to it
Console.WriteLine(s2); // "Hello" - ยังคงเดิม

// String interning
string a = "hello";
string b = "hello";
Console.WriteLine(object.ReferenceEquals(a, b)); // true (interned)

string c = new string(new char[] { 'h', 'e', 'l', 'l', 'o' });
Console.WriteLine(object.ReferenceEquals(a, c)); // false
Console.WriteLine(a == c);                       // true (value equal)

// ============================================
// String Methods
// ============================================

string text = "  Hello, สวัสดี World!  ";

// Trimming
Console.WriteLine(text.Trim());         // "Hello, สวัสดี World!"
Console.WriteLine(text.TrimStart());    // "Hello, สวัสดี World!  "
Console.WriteLine(text.TrimEnd());      // "  Hello, สวัสดี World!"
Console.WriteLine(text.Trim('!', ' ')); // "Hello, สวัสดี World"

// Case
Console.WriteLine(text.ToUpper()); // UPPER CASE
Console.WriteLine(text.ToLower()); // lower case
Console.WriteLine(text.ToUpperInvariant()); // culture-independent

// Searching
string search = "Hello, สวัสดี World!";
Console.WriteLine(search.Contains("สวัสดี"));           // true
Console.WriteLine(search.StartsWith("Hello"));          // true
Console.WriteLine(search.EndsWith("!"));                // true
Console.WriteLine(search.IndexOf("สวัสดี"));            // 7
Console.WriteLine(search.LastIndexOf("l"));             // 3
Console.WriteLine(search.IndexOfAny(new[]{'!','?'})); // last punctuation

// Substring
Console.WriteLine(search.Substring(7, 6));   // "สวัสดี"
Console.WriteLine(search[7..13]);            // "สวัสดี" (Range operator)
Console.WriteLine(search[^1]);               // "!" (from end)

// Replace
Console.WriteLine(search.Replace("Hello", "Hi")); // "Hi, สวัสดี World!"
Console.WriteLine(search.Replace(",", ";")); // "Hello; สวัสดี World!"

// Split/Join
string csv = "a,b,c,d,e";
string[] parts = csv.Split(',');
string rejoined = string.Join(" - ", parts); // "a - b - c - d - e"

// Split with options
string messy = "  hello   world  foo  ";
string[] words = messy.Split(new[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);
// ["hello", "world", "foo"]

// ============================================
// String Comparison
// ============================================

string s3 = "hello";
string s4 = "Hello";

// Case-insensitive
bool equal1 = string.Equals(s3, s4, StringComparison.OrdinalIgnoreCase); // true
bool equal2 = s3.Equals(s4, StringComparison.CurrentCultureIgnoreCase);  // true

// Compare
int cmp1 = string.Compare(s3, s4, StringComparison.Ordinal);         // not 0
int cmp2 = string.Compare(s3, s4, StringComparison.OrdinalIgnoreCase); // 0

// Contains with comparison
bool has = "Hello World".Contains("hello", StringComparison.OrdinalIgnoreCase); // true

// ============================================
// String Interpolation ขั้นสูง
// ============================================

decimal price = 12345.678m;
DateTime date = new DateTime(2024, 3, 15);

// Format specifiers
Console.WriteLine($"Price: {price:C}");        // Price: ฿12,345.68
Console.WriteLine($"Price: {price:N2}");       // Price: 12,345.68
Console.WriteLine($"Price: {price:F3}");       // Price: 12345.678
Console.WriteLine($"Date: {date:yyyy-MM-dd}"); // Date: 2024-03-15
Console.WriteLine($"Date: {date:D}");          // Date: Friday, March 15, 2024
Console.WriteLine($"Hex: {255:X2}");           // Hex: FF
Console.WriteLine($"Binary: {42:B8}");         // Binary: 00101010 (C# 12+)

// Alignment
Console.WriteLine($"{"Name",-15} {"Score",8}");
Console.WriteLine($"{"Alice",-15} {95.5,8:F1}");
Console.WriteLine($"{"Bob",-15} {87.3,8:F1}");
```

---

## Step 142: StringBuilder

```csharp
using System.Text;

// ============================================
// StringBuilder - Mutable String
// ============================================

// สำหรับการต่อ string จำนวนมาก
// ❌ Bad: O(n²) - creates many string objects
string result = "";
for (int i = 0; i < 10000; i++)
    result += i.ToString(); // ช้ามาก!

// ✅ Good: O(n) - single buffer
var sb = new StringBuilder(capacity: 50000);
for (int i = 0; i < 10000; i++)
    sb.Append(i);
string result2 = sb.ToString();

// ============================================
// StringBuilder Methods
// ============================================

var builder = new StringBuilder();

// Append
builder.Append("Hello");
builder.Append(", ");
builder.Append("World");
builder.AppendLine(); // แถวใหม่
builder.AppendLine("New line");
builder.AppendFormat("{0}: {1:C}", "Price", 1234.56m);

// Insert
builder.Insert(0, "START: ");

// Remove
builder.Remove(0, 7); // ลบ "START: "

// Replace
builder.Replace("Hello", "Hi");

// Length
Console.WriteLine($"Length: {builder.Length}");

// Access chars
builder[0] = 'h'; // lowercase first char

Console.WriteLine(builder.ToString());

// ============================================
// StringBuilder สำหรับ HTML Generation
// ============================================

public static string GenerateHtmlTable(
    IEnumerable<string> headers, 
    IEnumerable<IEnumerable<string>> rows)
{
    var sb = new StringBuilder();
    sb.AppendLine("<table>");
    
    // Header
    sb.AppendLine("  <thead><tr>");
    foreach (var h in headers)
        sb.AppendLine($"    <th>{EscapeHtml(h)}</th>");
    sb.AppendLine("  </tr></thead>");
    
    // Body
    sb.AppendLine("  <tbody>");
    foreach (var row in rows)
    {
        sb.AppendLine("    <tr>");
        foreach (var cell in row)
            sb.AppendLine($"      <td>{EscapeHtml(cell)}</td>");
        sb.AppendLine("    </tr>");
    }
    sb.AppendLine("  </tbody>");
    sb.AppendLine("</table>");
    
    return sb.ToString();
}

static string EscapeHtml(string text)
    => text.Replace("&", "&amp;").Replace("<", "&lt;").Replace(">", "&gt;");

// ใช้งาน
string html = GenerateHtmlTable(
    new[] { "ชื่อ", "อายุ", "GPA" },
    new[]
    {
        new[] { "อลิส", "22", "3.8" },
        new[] { "บ็อบ", "20", "3.2" },
    }
);
Console.WriteLine(html);
```

---

## Step 143: String Formatting

```csharp
// ============================================
// String.Format
// ============================================

// Basic
string msg = string.Format("Hello, {0}! You are {1} years old.", "สมชาย", 30);

// Named (requires record/object - use interpolation instead)
// C# 10+: constant interpolation
const string prefix = "LOG";
// const string logFormat = $"{prefix}: {{0}}"; // not supported - runtime values only

// Composite formatting
Console.WriteLine("{0,-20} {1,10:C} {2,8:F2}", "Product Name", 12345.67m, 3.14);

// ============================================
// Custom Format Providers
// ============================================

public class ThaiCurrencyFormat : IFormatProvider, ICustomFormatter
{
    public object? GetFormat(Type? formatType)
        => formatType == typeof(ICustomFormatter) ? this : null;
    
    public string Format(string? format, object? arg, IFormatProvider? formatProvider)
    {
        if (arg is decimal value && format == "THB")
            return $"฿{value:N2}";
        
        return arg?.ToString() ?? string.Empty;
    }
}

var provider = new ThaiCurrencyFormat();
string price = string.Format(provider, "{0:THB}", 12345.67m); // ฿12,345.67
Console.WriteLine(price);

// ============================================
// Span<char> สำหรับ Zero-allocation Strings
// ============================================

// ไม่ allocate string ใหม่
ReadOnlySpan<char> span = "Hello, World!".AsSpan();
ReadOnlySpan<char> hello = span[..5];       // "Hello"
ReadOnlySpan<char> world = span[7..12];     // "World"

Console.WriteLine(hello.ToString()); // ต้อง ToString เมื่อต้องการ string
Console.WriteLine(hello.SequenceEqual("Hello")); // true - no allocation

// Parse numbers without substring
ReadOnlySpan<char> numStr = "  123  ".AsSpan().Trim();
int.TryParse(numStr, out int num); // No intermediate string!
Console.WriteLine(num); // 123

// ============================================
// String.Create - Efficient string creation
// ============================================

static string CreateWithSpan(int[] numbers)
{
    int totalLength = numbers.Sum(n => n.ToString().Length) + numbers.Length - 1;
    
    return string.Create(totalLength, numbers, (span, nums) =>
    {
        int pos = 0;
        for (int i = 0; i < nums.Length; i++)
        {
            if (i > 0) span[pos++] = ',';
            nums[i].TryFormat(span[pos..], out int written);
            pos += written;
        }
    });
}

string csv2 = CreateWithSpan(new[] { 1, 23, 456, 7890 }); // "1,23,456,7890"
```

---

## Step 144: Regular Expressions

```csharp
using System.Text.RegularExpressions;

// ============================================
// Regex Basics
// ============================================

// Compiled regex (ดีกว่าสำหรับ reuse)
var emailRegex = new Regex(
    @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$",
    RegexOptions.Compiled | RegexOptions.IgnoreCase);

// Source generators (C# 10+ - best performance)
[GeneratedRegex(@"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$",
    RegexOptions.IgnoreCase)]
private static partial Regex EmailRegexGenerated();

// IsMatch
bool isValidEmail = emailRegex.IsMatch("user@example.com"); // true
bool isValidEmail2 = emailRegex.IsMatch("not-an-email");    // false

// Match - ค้นหาครั้งแรก
var match = Regex.Match("Phone: 02-345-6789", @"\d{2}-\d{3}-\d{4}");
if (match.Success)
    Console.WriteLine($"Found: {match.Value}"); // "02-345-6789"

// Matches - ค้นหาทั้งหมด
var text = "Call 02-234-5678 or 081-234-5678 or 02-345-9876";
var phoneRegex = new Regex(@"(\d{2,3})-(\d{3})-(\d{4})");

foreach (Match m in phoneRegex.Matches(text))
{
    Console.WriteLine($"Full: {m.Value}");
    Console.WriteLine($"  Area: {m.Groups[1].Value}");
    Console.WriteLine($"  Middle: {m.Groups[2].Value}");
    Console.WriteLine($"  Last: {m.Groups[3].Value}");
}

// ============================================
// Named Groups
// ============================================

var dateRegex = new Regex(
    @"(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})");

var dateMatch = dateRegex.Match("Today is 2024-03-15");
if (dateMatch.Success)
{
    int year = int.Parse(dateMatch.Groups["year"].Value);
    int month = int.Parse(dateMatch.Groups["month"].Value);
    int day = int.Parse(dateMatch.Groups["day"].Value);
    Console.WriteLine($"Date: {day}/{month}/{year}");
}
```

---

## Step 145: Advanced Regex

```csharp
// ============================================
// Replace with Regex
// ============================================

string html2 = "<p>Hello <b>World</b></p>";

// Remove HTML tags
string noHtml = Regex.Replace(html2, @"<[^>]+>", "");
Console.WriteLine(noHtml); // "Hello World"

// Replace with MatchEvaluator
string template = "Dear {{Name}}, your order #{{OrderId}} is ready.";
var vars = new Dictionary<string, string>
{
    { "Name", "สมชาย" },
    { "OrderId", "ORD-12345" }
};

string replaced = Regex.Replace(template, @"\{\{(\w+)\}\}", match =>
{
    string key = match.Groups[1].Value;
    return vars.TryGetValue(key, out string? value) ? value : match.Value;
});

Console.WriteLine(replaced);
// "Dear สมชาย, your order #ORD-12345 is ready."

// ============================================
// Split with Regex
// ============================================

// แยกด้วย whitespace หลายชนิด
string text2 = "word1   word2\tword3\nword4";
string[] words2 = Regex.Split(text2, @"\s+");
// ["word1", "word2", "word3", "word4"]

// แยกที่ข้อความแต่เก็บ delimiter
string sentence = "Hello. World! How? Are you.";
string[] parts2 = Regex.Split(sentence, @"(?<=[.!?])\s*");
// ["Hello.", "World!", "How?", "Are you."]

// ============================================
// Lookahead/Lookbehind
// ============================================

// Positive lookahead: ตามด้วย
var beforeDot = new Regex(@"\w+(?=\.)");
var priceRegex = new Regex(@"\d+(?=\s*บาท)"); // number before "บาท"

foreach (Match m in priceRegex.Matches("ราคา 500 บาท และ 1000 บาท"))
    Console.WriteLine($"Price: {m.Value}");

// Negative lookahead: ไม่ตามด้วย
var notPassword = new Regex(@"\b(?!password)\w+\b");

// Positive lookbehind: ก่อนหน้าด้วย
var afterDollar = new Regex(@"(?<=\$)\d+");
foreach (Match m in afterDollar.Matches("Pay $100 and $200"))
    Console.WriteLine($"Amount: {m.Value}");

// ============================================
// Regex Patterns สำหรับ Thai
// ============================================

// Thai characters
var thaiOnlyRegex = new Regex(@"^[฀-๿\s]+$");
Console.WriteLine(thaiOnlyRegex.IsMatch("สวัสดี")); // true
Console.WriteLine(thaiOnlyRegex.IsMatch("Hello สวัสดี")); // false

// Thai phone (081-234-5678 or 02-234-5678)
var thaiPhone = new Regex(@"^(0[689]\d-\d{3}-\d{4}|0[2-8]-\d{3}-\d{4})$");
Console.WriteLine(thaiPhone.IsMatch("081-234-5678")); // true
Console.WriteLine(thaiPhone.IsMatch("02-234-5678")); // true

// Thai ID card (13 digits)
var thaiId = new Regex(@"^\d{13}$");

// Thai bank account
var thaiAccount = new Regex(@"^\d{3}-\d-\d{5}-\d$"); // BBL format
```

---

## Step 146: Text Processing Library

```csharp
// ============================================
// Complete Text Processing Library
// ============================================

public static class TextUtils
{
    // Email validation
    private static readonly Regex _emailRegex = new(
        @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$",
        RegexOptions.Compiled | RegexOptions.IgnoreCase);
    
    // URL validation
    private static readonly Regex _urlRegex = new(
        @"^https?://[\w-]+(\.[\w-]+)+([\w.,@?^=%&:/~+#-]*[\w@?^=%&/~+#-])?$",
        RegexOptions.Compiled | RegexOptions.IgnoreCase);
    
    // Thai phone
    private static readonly Regex _thaiPhoneRegex = new(
        @"^(0[689]\d{8}|0[2-8]\d{7})$",
        RegexOptions.Compiled);
    
    public static bool IsValidEmail(string? email)
        => !string.IsNullOrWhiteSpace(email) && _emailRegex.IsMatch(email);
    
    public static bool IsValidUrl(string? url)
        => !string.IsNullOrWhiteSpace(url) && _urlRegex.IsMatch(url);
    
    public static bool IsValidThaiPhone(string? phone)
    {
        if (string.IsNullOrWhiteSpace(phone)) return false;
        string normalized = phone.Replace("-", "").Replace(" ", "");
        return _thaiPhoneRegex.IsMatch(normalized);
    }
    
    // Slug generation
    public static string ToSlug(string text)
    {
        string lower = text.ToLowerInvariant().Trim();
        string noSpecial = Regex.Replace(lower, @"[^a-z0-9\s-]", "");
        string noSpaces = Regex.Replace(noSpecial, @"\s+", "-");
        return Regex.Replace(noSpaces, @"-{2,}", "-").Trim('-');
    }
    
    // Truncate with ellipsis
    public static string Truncate(string text, int maxLength, string suffix = "...")
    {
        if (text.Length <= maxLength) return text;
        return text[..(maxLength - suffix.Length)] + suffix;
    }
    
    // Word count
    public static int WordCount(string text)
        => Regex.Matches(text.Trim(), @"\S+").Count;
    
    // Extract URLs
    public static IEnumerable<string> ExtractUrls(string text)
        => Regex.Matches(text, @"https?://\S+").Select(m => m.Value);
    
    // Mask sensitive data
    public static string MaskEmail(string email)
    {
        var match = Regex.Match(email, @"^(.{2})(.*)(@.*)$");
        if (!match.Success) return email;
        string visible = match.Groups[1].Value;
        string hidden = new string('*', match.Groups[2].Length);
        return visible + hidden + match.Groups[3].Value;
    }
    
    public static string MaskPhone(string phone)
    {
        string digits = Regex.Replace(phone, @"\D", "");
        if (digits.Length < 4) return phone;
        return "***-***-" + digits[^4..];
    }
    
    // Template rendering
    public static string Render(string template, Dictionary<string, object?> data)
        => Regex.Replace(template, @"\{\{(\w+)\}\}", match =>
        {
            string key = match.Groups[1].Value;
            return data.TryGetValue(key, out object? value) 
                ? value?.ToString() ?? "" 
                : match.Value;
        });
    
    // HTML encode/decode
    public static string HtmlEncode(string text)
        => System.Net.WebUtility.HtmlEncode(text);
    
    public static string HtmlDecode(string html)
        => System.Net.WebUtility.HtmlDecode(html);
    
    // Camel/Pascal/Snake case
    public static string ToCamelCase(string text)
    {
        string pascal = ToPascalCase(text);
        return char.ToLower(pascal[0]) + pascal[1..];
    }
    
    public static string ToPascalCase(string text)
        => string.Concat(
            Regex.Split(text, @"[_\s-]+")
                .Where(w => w.Length > 0)
                .Select(w => char.ToUpper(w[0]) + w[1..].ToLower())
        );
    
    public static string ToSnakeCase(string text)
        => Regex.Replace(
            Regex.Replace(text, @"([A-Z])([A-Z][a-z])", "$1_$2"),
            @"([a-z])([A-Z])", "$1_$2")
            .ToLower();
}

// ใช้งาน
Console.WriteLine(TextUtils.IsValidEmail("test@example.com")); // true
Console.WriteLine(TextUtils.IsValidEmail("not-an-email"));     // false

Console.WriteLine(TextUtils.ToSlug("Hello World! How Are You?"));
// "hello-world-how-are-you"

Console.WriteLine(TextUtils.MaskEmail("somchai@gmail.com"));
// "so*****@gmail.com"

Console.WriteLine(TextUtils.MaskPhone("081-234-5678"));
// "***-***-5678"

string rendered = TextUtils.Render(
    "Dear {{Name}}, your total is {{Amount}}.",
    new() { { "Name", "สมชาย" }, { "Amount", "฿1,500" } }
);
Console.WriteLine(rendered);

Console.WriteLine(TextUtils.ToCamelCase("hello_world_foo")); // helloWorldFoo
Console.WriteLine(TextUtils.ToPascalCase("hello_world_foo")); // HelloWorldFoo
Console.WriteLine(TextUtils.ToSnakeCase("HelloWorldFoo")); // hello_world_foo
```

---

## Step 147: Natural Language Processing (NLP) พื้นฐาน

```csharp
// ============================================
// Text Analysis
// ============================================

public class TextAnalyzer
{
    private static readonly HashSet<string> _stopWords = new(StringComparer.OrdinalIgnoreCase)
    {
        "the", "a", "an", "and", "or", "but", "in", "on", "at", "to",
        "for", "of", "with", "by", "from", "is", "are", "was", "were",
        "ที่", "และ", "ใน", "มี", "ของ", "โดย"
    };
    
    public Dictionary<string, int> WordFrequency(string text)
    {
        var words = Regex.Matches(text.ToLower(), @"\b[a-zA-Zก-๙]+\b")
            .Select(m => m.Value)
            .Where(w => !_stopWords.Contains(w) && w.Length > 1);
        
        return words
            .GroupBy(w => w)
            .ToDictionary(g => g.Key, g => g.Count());
    }
    
    public IEnumerable<string> TopWords(string text, int n)
        => WordFrequency(text)
            .OrderByDescending(kv => kv.Value)
            .Take(n)
            .Select(kv => $"{kv.Key}: {kv.Value}");
    
    public double ReadabilityScore(string text)
    {
        var sentences = Regex.Split(text, @"[.!?]+").Where(s => s.Length > 0);
        var words = Regex.Matches(text, @"\b\w+\b");
        var syllables = words.Sum(w => EstimateSyllables(w.Value));
        
        int sentenceCount = sentences.Count();
        int wordCount = words.Count;
        
        if (sentenceCount == 0 || wordCount == 0) return 0;
        
        // Flesch Reading Ease (simplified)
        return 206.835 - 1.015 * (wordCount / sentenceCount) 
                       - 84.6 * (syllables / wordCount);
    }
    
    private int EstimateSyllables(string word)
    {
        word = word.ToLower();
        int count = Regex.Matches(word, @"[aeiouy]+").Count;
        if (word.EndsWith("e")) count--;
        return Math.Max(1, count);
    }
    
    public IEnumerable<string> ExtractSentences(string text)
        => Regex.Split(text, @"(?<=[.!?])\s+")
            .Select(s => s.Trim())
            .Where(s => s.Length > 0);
    
    public string Summarize(string text, int maxSentences = 3)
    {
        var sentences = ExtractSentences(text).ToList();
        var wordFreq = WordFrequency(text);
        
        // Score sentences by word frequency
        var scored = sentences
            .Select(s => new
            {
                Sentence = s,
                Score = Regex.Matches(s.ToLower(), @"\b\w+\b")
                    .Sum(m => wordFreq.GetValueOrDefault(m.Value, 0))
            })
            .OrderByDescending(x => x.Score)
            .Take(maxSentences)
            .OrderBy(x => sentences.IndexOf(x.Sentence)); // preserve order
        
        return string.Join(" ", scored.Select(x => x.Sentence));
    }
}

// ใช้งาน
var analyzer = new TextAnalyzer();

string article = """
    C# is a modern programming language. It supports object-oriented programming.
    C# was created by Microsoft and runs on the .NET platform.
    The language has many features including generics, LINQ, and async/await.
    C# is widely used for building desktop, web, and mobile applications.
    """;

Console.WriteLine("Top words:");
analyzer.TopWords(article, 5).ToList().ForEach(Console.WriteLine);

Console.WriteLine($"\nReadability: {analyzer.ReadabilityScore(article):F1}");
Console.WriteLine($"\nSummary: {analyzer.Summarize(article, 2)}");
```

---

## Step 148: Encoding และ Unicode

```csharp
using System.Text;

// ============================================
// Encoding
// ============================================

// UTF-8 (ค่าเริ่มต้นใน .NET)
string thai = "สวัสดี ชาวโลก";
byte[] utf8Bytes = Encoding.UTF8.GetBytes(thai);
Console.WriteLine($"UTF-8 bytes: {utf8Bytes.Length}"); // ภาษาไทย = 3 bytes/char

// UTF-16 (ที่ .NET ใช้ใน memory)
byte[] utf16Bytes = Encoding.Unicode.GetBytes(thai);
Console.WriteLine($"UTF-16 bytes: {utf16Bytes.Length}"); // 2 bytes/char

// ภาษาไทยใน ASCII จะเป็น '?'
byte[] asciiBytes = Encoding.ASCII.GetBytes(thai);
Console.WriteLine($"ASCII bytes: {Encoding.ASCII.GetString(asciiBytes)}"); // ???

// Decode back
string decoded = Encoding.UTF8.GetString(utf8Bytes);
Console.WriteLine(decoded == thai); // true

// ============================================
// BOM (Byte Order Mark)
// ============================================

// UTF-8 with BOM (Windows Notepad default)
var utf8WithBom = new UTF8Encoding(encoderShouldEmitUTF8Identifier: true);
var utf8NoBom = new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);

// Writing with BOM
await File.WriteAllTextAsync("with_bom.txt", "สวัสดี", utf8WithBom);
await File.WriteAllTextAsync("no_bom.txt", "สวัสดี", utf8NoBom);

// ============================================
// Unicode String Operations
// ============================================

// String.Length ไม่เท่ากับจำนวน Unicode characters
string emoji = "Hello 😀 World";
Console.WriteLine($"Length: {emoji.Length}"); // 14 (😀 = 2 chars in .NET)

// StringInfo สำหรับ Unicode-aware length
var info = new System.Globalization.StringInfo(emoji);
Console.WriteLine($"Text elements: {info.LengthInTextElements}"); // 13

// Enumerate Unicode characters
var enumerator = System.Globalization.StringInfo.GetTextElementEnumerator(emoji);
int count = 0;
while (enumerator.MoveNext())
{
    count++;
    string element = enumerator.GetTextElement();
    // Console.Write($"[{element}]");
}
Console.WriteLine($"True char count: {count}");

// ============================================
// Base64 Encoding
// ============================================

string original = "This is a secret message! สวัสดี";
byte[] bytes = Encoding.UTF8.GetBytes(original);

string base64 = Convert.ToBase64String(bytes);
Console.WriteLine($"Base64: {base64}");

byte[] decoded2 = Convert.FromBase64String(base64);
string back = Encoding.UTF8.GetString(decoded2);
Console.WriteLine($"Decoded: {back}");

// URL-safe Base64
string urlSafe = base64.Replace('+', '-').Replace('/', '_').TrimEnd('=');
```

---

## Step 149: String Parsing

```csharp
// ============================================
// Parsing สร้างสรรค์
// ============================================

// TryParse pattern
public static class Parser
{
    public static bool TryParseDate(string text, out DateOnly date)
    {
        // รองรับหลายรูปแบบ
        string[] formats = { "yyyy-MM-dd", "dd/MM/yyyy", "d/M/yyyy", "MM-dd-yyyy" };
        
        foreach (var format in formats)
        {
            if (DateOnly.TryParseExact(text, format, 
                System.Globalization.CultureInfo.InvariantCulture,
                System.Globalization.DateTimeStyles.None, out date))
                return true;
        }
        
        date = default;
        return false;
    }
    
    public static bool TryParseRange(string text, out (int Min, int Max) range)
    {
        range = default;
        var match = Regex.Match(text, @"^(\d+)\s*[-~]\s*(\d+)$");
        if (!match.Success) return false;
        
        range = (int.Parse(match.Groups[1].Value), int.Parse(match.Groups[2].Value));
        return range.Min <= range.Max;
    }
    
    public static T Parse<T>(string text) where T : IParsable<T>
        => T.Parse(text, null);
    
    public static bool TryParse<T>(string text, out T? result) where T : IParsable<T>
        => T.TryParse(text, null, out result);
}

// ============================================
// Building DSL (Domain Specific Language)
// ============================================

public class QueryParser
{
    // Parse: "age > 18 AND major = CS OR major = IT"
    private static readonly Regex _conditionRegex = new(
        @"(\w+)\s*(=|!=|>|>=|<|<=)\s*(\w+)",
        RegexOptions.IgnoreCase);
    
    public static Func<Dictionary<string, string>, bool> Parse(string query)
    {
        var tokens = Regex.Split(query.Trim(), @"\s+(AND|OR)\s+", 
            RegexOptions.IgnoreCase);
        var operators = Regex.Matches(query, @"\b(AND|OR)\b", RegexOptions.IgnoreCase)
            .Select(m => m.Value.ToUpper())
            .ToList();
        
        var conditions = tokens
            .Where((_, i) => i % 1 == 0)
            .Select(ParseCondition)
            .ToList();
        
        return record =>
        {
            bool result = conditions[0](record);
            for (int i = 0; i < operators.Count && i + 1 < conditions.Count; i++)
            {
                if (operators[i] == "AND")
                    result = result && conditions[i + 1](record);
                else
                    result = result || conditions[i + 1](record);
            }
            return result;
        };
    }
    
    private static Func<Dictionary<string, string>, bool> ParseCondition(string condition)
    {
        var match = _conditionRegex.Match(condition);
        if (!match.Success) return _ => true;
        
        string field = match.Groups[1].Value;
        string op = match.Groups[2].Value;
        string value = match.Groups[3].Value;
        
        return record =>
        {
            if (!record.TryGetValue(field, out string? actual)) return false;
            
            int? numCompare = null;
            if (double.TryParse(actual, out double a) && double.TryParse(value, out double b))
                numCompare = a.CompareTo(b);
            
            return op switch
            {
                "=" => actual.Equals(value, StringComparison.OrdinalIgnoreCase),
                "!=" => !actual.Equals(value, StringComparison.OrdinalIgnoreCase),
                ">" => numCompare > 0,
                ">=" => numCompare >= 0,
                "<" => numCompare < 0,
                "<=" => numCompare <= 0,
                _ => false
            };
        };
    }
}
```

---

## Step 150: Internationalization (i18n)

```csharp
using System.Globalization;

// ============================================
// Culture-aware Formatting
// ============================================

decimal price2 = 12345.67m;
DateTime now = DateTime.Now;

// Thai culture
var thai2 = new CultureInfo("th-TH");
Console.WriteLine(price2.ToString("C", thai2));    // ฿12,345.67
Console.WriteLine(now.ToString("D", thai2));       // วันศุกร์ที่ 15 มีนาคม 2024

// US culture
var us = new CultureInfo("en-US");
Console.WriteLine(price2.ToString("C", us));       // $12,345.67
Console.WriteLine(now.ToString("D", us));          // Friday, March 15, 2024

// Japan
var jp = new CultureInfo("ja-JP");
Console.WriteLine(price2.ToString("C", jp));       // ¥12,346
Console.WriteLine(now.ToString("D", jp));          // 2024年3月15日

// Parsing culture-specific numbers
string thaiNum = "12,345.67";
string europeanNum = "12.345,67"; // European format

decimal parsed1 = decimal.Parse(thaiNum, us);
decimal parsed2 = decimal.Parse(europeanNum, new CultureInfo("de-DE"));

// ============================================
// Resource Files (Concept)
// ============================================

// In real MAUI app:
/*
// Resources/Strings.resx (default - English)
// Resources/Strings.th.resx (Thai)

// Usage:
// string greeting = Resources.Strings.Greeting;
// Changes automatically based on device language
*/

// Simple resource dictionary
public class LocalizedStrings
{
    private static readonly Dictionary<string, Dictionary<string, string>> _strings = new()
    {
        ["en"] = new()
        {
            ["greeting"] = "Hello!",
            ["farewell"] = "Goodbye!",
            ["yes"] = "Yes",
            ["no"] = "No",
        },
        ["th"] = new()
        {
            ["greeting"] = "สวัสดี!",
            ["farewell"] = "ลาก่อน!",
            ["yes"] = "ใช่",
            ["no"] = "ไม่",
        },
    };
    
    private readonly string _lang;
    
    public LocalizedStrings(string lang = "en") 
        => _lang = _strings.ContainsKey(lang) ? lang : "en";
    
    public string Get(string key, string defaultValue = "")
    {
        if (_strings.TryGetValue(_lang, out var dict) && dict.TryGetValue(key, out var val))
            return val;
        
        // Fallback to English
        return _strings["en"].GetValueOrDefault(key, defaultValue);
    }
    
    public string this[string key] => Get(key);
}

// ใช้งาน
var en = new LocalizedStrings("en");
var th = new LocalizedStrings("th");

Console.WriteLine($"English: {en["greeting"]}"); // Hello!
Console.WriteLine($"ภาษาไทย: {th["greeting"]}"); // สวัสดี!
```

---

## สรุป Part 15

ใน Part 15 เราได้เรียนรู้:

1. **String Fundamentals** - Immutability, Methods, Comparison
2. **StringBuilder** - Efficient string building
3. **String Formatting** - Format providers, Span<char>
4. **Regex Basics** - Match, Replace, Split
5. **Advanced Regex** - Named groups, Lookahead, Lookbehind
6. **Text Processing Library** - Validation, Masking, Template
7. **Text Analysis** - Word frequency, Readability
8. **Encoding** - UTF-8, Base64
9. **String Parsing** - TryParse, DSL
10. **Internationalization** - Culture, Localization

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 15 | Steps 141-150*
