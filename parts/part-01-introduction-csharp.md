# Part 01: บทนำ C# และ .NET Ecosystem
## Steps 1-10: ทำความรู้จักกับ C# และ .NET

---

## Step 1: C# คืออะไร?

C# (อ่านว่า "C Sharp") คือภาษาโปรแกรมมิ่งที่พัฒนาโดย Microsoft ในปี 2000 โดย Anders Hejlsberg เป็นภาษาที่:

- **Object-Oriented** - รองรับการเขียนโปรแกรมเชิงวัตถุอย่างสมบูรณ์
- **Type-Safe** - ตรวจสอบชนิดข้อมูลตั้งแต่ Compile-time
- **Managed Language** - มี Garbage Collector จัดการหน่วยความจำ
- **Cross-platform** - รันได้บน Windows, macOS, Linux, iOS, Android

### ประวัติความเป็นมา

```
C# 1.0 (2002) - เวอร์ชันแรก
C# 2.0 (2005) - เพิ่ม Generics, Nullable types
C# 3.0 (2007) - เพิ่ม LINQ, Lambda expressions
C# 4.0 (2010) - เพิ่ม Dynamic typing
C# 5.0 (2012) - เพิ่ม Async/Await
C# 6.0 (2015) - String interpolation, Expression-bodied members
C# 7.0 (2017) - Pattern matching, Tuples
C# 8.0 (2019) - Nullable reference types, Async streams
C# 9.0 (2020) - Records, Init-only properties
C# 10.0 (2021) - Global using, File-scoped namespaces
C# 11.0 (2022) - Raw string literals, Generic attributes
C# 12.0 (2023) - Primary constructors, Collection expressions
C# 13.0 (2024) - params collections, New lock type
```

---

## Step 2: .NET Ecosystem คืออะไร?

.NET คือ Platform สำหรับพัฒนาซอฟต์แวร์ที่ครอบคลุมหลากหลายประเภทแอปพลิเคชัน

```
.NET Ecosystem
├── .NET Runtime (CLR - Common Language Runtime)
│   ├── Just-In-Time Compiler (JIT)
│   ├── Garbage Collector
│   └── Memory Manager
├── .NET Base Class Library (BCL)
│   ├── System.Collections
│   ├── System.IO
│   ├── System.Net
│   └── System.Threading
└── Application Frameworks
    ├── ASP.NET Core (Web)
    ├── .NET MAUI (Mobile/Desktop)
    ├── Blazor (WebAssembly)
    ├── WPF/WinForms (Windows Desktop)
    └── Unity (Game Development)
```

### เวอร์ชันของ .NET

| เวอร์ชัน | ปีที่ออก | สถานะ | LTS |
|---------|---------|-------|-----|
| .NET Framework 4.8 | 2019 | Maintenance | ไม่ใช่ |
| .NET Core 3.1 | 2019 | End of Life | ใช่ |
| .NET 5 | 2020 | End of Life | ไม่ใช่ |
| .NET 6 | 2021 | End of Life | ใช่ |
| .NET 7 | 2022 | End of Life | ไม่ใช่ |
| .NET 8 | 2023 | **Active** | **ใช่** |
| .NET 9 | 2024 | Active | ไม่ใช่ |
| .NET 10 | 2025 | Active | ใช่ |

> **แนะนำ**: ใช้ .NET 8 หรือ .NET 9 สำหรับโปรเจกต์ใหม่

---

## Step 3: ทำไมต้องเรียน C# สำหรับ Mobile Development?

### เปรียบเทียบกับภาษาอื่น

| ภาษา | Platform | ข้อดี | ข้อเสีย |
|------|---------|-------|---------|
| **C# / .NET MAUI** | iOS, Android, Windows, macOS | Code เดียว ทุก platform | ต้องเรียน C# |
| Swift | iOS, macOS เท่านั้น | Performance สูง | ใช้ได้เฉพาะ Apple |
| Kotlin | Android เท่านั้น | Native Android | ไม่ cross-platform |
| Flutter (Dart) | iOS, Android, Web | UI สวย | Ecosystem เล็กกว่า |
| React Native (JS) | iOS, Android | ใช้ JS ได้เลย | Performance ต่ำกว่า |

### ข้อดีของ C# / .NET MAUI

1. **เขียนครั้งเดียว รันได้ทุก Platform**
2. **Performance ใกล้เคียง Native**
3. **Ecosystem ขนาดใหญ่** - NuGet มีมากกว่า 300,000 packages
4. **Microsoft Support** - มี enterprise support
5. **Salary ดี** - เป็นที่ต้องการในตลาดงาน

---

## Step 4: การติดตั้ง Development Environment

### Windows

#### 1. ติดตั้ง Visual Studio 2022

```
1. ไปที่ https://visualstudio.microsoft.com/downloads/
2. เลือก "Community" (ฟรี)
3. ในหน้า Installer เลือก Workloads:
   ✅ .NET Multi-platform App UI development
   ✅ .NET desktop development
   ✅ ASP.NET and web development
4. คลิก Install
5. รอจนติดตั้งเสร็จ (ประมาณ 30-60 นาที)
```

#### 2. ตรวจสอบการติดตั้ง

เปิด Command Prompt หรือ PowerShell แล้วพิมพ์:

```bash
# ตรวจสอบ .NET version
dotnet --version
# ควรแสดง: 8.x.x หรือ 9.x.x

# ตรวจสอบ SDK ทั้งหมด
dotnet --list-sdks

# ตรวจสอบ Runtime ทั้งหมด
dotnet --list-runtimes
```

### macOS

#### 1. ติดตั้ง .NET SDK

```bash
# ติดตั้งผ่าน Homebrew (แนะนำ)
brew install --cask dotnet-sdk

# หรือ ดาวน์โหลดจาก
# https://dotnet.microsoft.com/download
```

#### 2. ติดตั้ง Visual Studio for Mac หรือ VS Code

```bash
# ติดตั้ง VS Code
brew install --cask visual-studio-code

# ติดตั้ง Extensions ที่จำเป็น
code --install-extension ms-dotnettools.csharp
code --install-extension ms-dotnettools.csdevkit
```

#### 3. ติดตั้ง Xcode (สำหรับ iOS Development)

```bash
# ติดตั้งจาก App Store
# หรือ
xcode-select --install
```

---

## Step 5: สร้างโปรแกรมแรก - Hello World

### วิธีที่ 1: ผ่าน Command Line

```bash
# สร้าง folder สำหรับโปรเจกต์
mkdir HelloWorld
cd HelloWorld

# สร้างโปรเจกต์ใหม่
dotnet new console --name HelloWorld

# เข้าไปในโปรเจกต์
cd HelloWorld

# รันโปรแกรม
dotnet run
```

### ผลลัพธ์ที่ได้:
```
Hello, World!
```

### วิธีที่ 2: ผ่าน Visual Studio 2022

```
1. เปิด Visual Studio 2022
2. คลิก "Create a new project"
3. เลือก "Console App" แล้วคลิก Next
4. ตั้งชื่อโปรเจกต์ "HelloWorld"
5. เลือก .NET 8.0 แล้วคลิก Create
6. กด F5 หรือคลิก Run
```

### โค้ด Hello World แบบต่างๆ

```csharp
// วิธีที่ 1: Top-level statements (C# 9+) - Modern style
Console.WriteLine("Hello, World!");
Console.WriteLine("ยินดีต้อนรับสู่การเรียน C#!");

// วิธีที่ 2: Traditional style
using System;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
            Console.WriteLine("ยินดีต้อนรับสู่การเรียน C#!");
        }
    }
}

// วิธีที่ 3: รับ Input จากผู้ใช้
Console.Write("กรุณาใส่ชื่อของคุณ: ");
string name = Console.ReadLine();
Console.WriteLine($"สวัสดี {name}! ยินดีต้อนรับสู่การเรียน C#!");
```

---

## Step 6: โครงสร้างโปรเจกต์ C# Console Application

```
HelloWorld/
├── HelloWorld.csproj    ← ไฟล์ config โปรเจกต์
├── Program.cs           ← ไฟล์โค้ดหลัก
├── obj/                 ← ไฟล์ temporary (auto-generated)
└── bin/                 ← ไฟล์ executable (auto-generated)
```

### ไฟล์ HelloWorld.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <!-- กำหนด output type -->
    <OutputType>Exe</OutputType>
    
    <!-- กำหนด target framework -->
    <TargetFramework>net8.0</TargetFramework>
    
    <!-- เปิดใช้ Nullable reference types -->
    <Nullable>enable</Nullable>
    
    <!-- เปิดใช้ Implicit global usings -->
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
```

---

## Step 7: การ Comment โค้ด

```csharp
// Single-line comment - ใช้สำหรับ comment บรรทัดเดียว
// นี่คือ comment บรรทัดเดียว

/* Multi-line comment
   ใช้สำหรับ comment หลายบรรทัด
   อธิบายโค้ดที่ซับซ้อน
*/

/// <summary>
/// XML Documentation Comment
/// ใช้สำหรับสร้าง API documentation อัตโนมัติ
/// </summary>
/// <param name="name">ชื่อของผู้ใช้</param>
/// <returns>ข้อความต้อนรับ</returns>
string Greet(string name)
{
    return $"สวัสดี {name}!";
}

// ตัวอย่างการใช้ comment อย่างมีประสิทธิภาพ
// BAD: อธิบายสิ่งที่เห็นได้อยู่แล้ว
int x = 5; // กำหนดค่า x เป็น 5

// GOOD: อธิบาย "ทำไม" ไม่ใช่ "อะไร"
int maxRetries = 5; // จำกัดไม่ให้ retry มากเกินไปเพื่อป้องกัน infinite loop
```

---

## Step 8: Namespace และ Using Statements

```csharp
// Using statement - บอก C# ว่าจะใช้ namespace ไหน
using System;                          // พื้นฐาน
using System.Collections.Generic;     // List, Dictionary, etc.
using System.IO;                       // File operations
using System.Linq;                     // LINQ queries
using System.Net.Http;                 // HTTP requests
using System.Text;                     // String builder
using System.Threading.Tasks;         // Async operations

// Global using (C# 10+) - ใช้ได้ทั้งโปรเจกต์
// ใส่ในไฟล์ GlobalUsings.cs
global using System;
global using System.Collections.Generic;

// Namespace declaration
namespace MyApp.Models
{
    public class User
    {
        public string Name { get; set; }
    }
}

// File-scoped namespace (C# 10+) - ทำให้โค้ดดูสะอาดขึ้น
namespace MyApp.Models;

public class Product
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}
```

---

## Step 9: การรัน, Build, และ Publish โปรแกรม

### คำสั่ง dotnet CLI พื้นฐาน

```bash
# สร้างโปรเจกต์ใหม่
dotnet new console -n MyApp        # Console App
dotnet new classlib -n MyLibrary   # Class Library
dotnet new webapi -n MyApi         # Web API
dotnet new maui -n MyMauiApp       # MAUI App

# Build โปรเจกต์
dotnet build

# รันโปรเจกต์
dotnet run

# รัน Tests
dotnet test

# Publish สำหรับ Production
dotnet publish -c Release -o ./publish

# เพิ่ม NuGet Package
dotnet add package Newtonsoft.Json
dotnet add package Microsoft.EntityFrameworkCore

# ลิสต์ Package ที่ใช้อยู่
dotnet list package

# ลบ Package
dotnet remove package Newtonsoft.Json

# Restore Packages
dotnet restore
```

### การ Publish แบบต่างๆ

```bash
# Publish แบบ Framework-dependent (ต้องมี .NET installed)
dotnet publish -c Release

# Publish แบบ Self-contained (ไม่ต้องมี .NET installed)
dotnet publish -c Release -r win-x64 --self-contained true

# Publish แบบ Single File
dotnet publish -c Release -r win-x64 --self-contained true \
  -p:PublishSingleFile=true

# Publish สำหรับ Platform ต่างๆ
dotnet publish -r win-x64       # Windows 64-bit
dotnet publish -r win-x86       # Windows 32-bit
dotnet publish -r linux-x64     # Linux 64-bit
dotnet publish -r osx-x64       # macOS Intel
dotnet publish -r osx-arm64     # macOS Apple Silicon
```

---

## Step 10: ทำความเข้าใจ C# Compilation Process

```
Source Code (.cs)
      ↓
   Compiler (csc/Roslyn)
      ↓
   IL Code (Intermediate Language)
   + Metadata
      ↓
   Assembly (.dll/.exe)
      ↓
   CLR (Common Language Runtime)
      ↓
   JIT Compiler (Just-In-Time)
      ↓
   Native Machine Code
      ↓
   Execution
```

### ตัวอย่างการ Compile และรัน

```csharp
// Program.cs - โค้ดที่เราเขียน (Source Code)
using System;

// คลาส Program ถูก compile เป็น IL
class Program
{
    // Method Main เป็น entry point ของโปรแกรม
    static void Main(string[] args)
    {
        // แสดงข้อมูล Runtime
        Console.WriteLine($"Runtime Version: {System.Runtime.InteropServices.RuntimeEnvironment.GetRuntimeDirectory()}");
        Console.WriteLine($"OS: {System.Runtime.InteropServices.RuntimeInformation.OSDescription}");
        Console.WriteLine($"Architecture: {System.Runtime.InteropServices.RuntimeInformation.ProcessArchitecture}");
        
        // แสดง args ที่ส่งมา
        Console.WriteLine($"\nCommand-line arguments ({args.Length} items):");
        for (int i = 0; i < args.Length; i++)
        {
            Console.WriteLine($"  [{i}]: {args[i]}");
        }
    }
}
```

---

## แบบฝึกหัด Part 01

### แบบฝึกหัดที่ 1.1: Hello World หลายภาษา
สร้างโปรแกรมที่แสดงข้อความ "Hello" ใน 5 ภาษาที่แตกต่างกัน

```csharp
// TODO: เติมโค้ดของคุณที่นี่
Console.WriteLine("Hello, World!");     // อังกฤษ
// เพิ่มภาษาอื่นๆ อีก 4 ภาษา
```

### แบบฝึกหัดที่ 1.2: Interactive Greeting
สร้างโปรแกรมที่:
1. ถามชื่อผู้ใช้
2. ถามอายุ
3. แสดงข้อความต้อนรับพร้อมชื่อและอายุ

```csharp
// TODO: เติมโค้ดของคุณที่นี่
Console.Write("ชื่อของคุณคือ: ");
string name = Console.ReadLine();
// เพิ่ม code สำหรับถามอายุและแสดงผล
```

### แบบฝึกหัดที่ 1.3: System Information
สร้างโปรแกรมที่แสดงข้อมูลระบบ:
- วันและเวลาปัจจุบัน
- OS ที่ใช้
- .NET Version
- Computer Name

```csharp
// TODO: เติมโค้ดของคุณที่นี่
Console.WriteLine($"วันที่: {DateTime.Now:dd/MM/yyyy HH:mm:ss}");
// เพิ่ม code สำหรับแสดงข้อมูลอื่นๆ
```

---

## เฉลยแบบฝึกหัด Part 01

### เฉลย 1.1: Hello World หลายภาษา

```csharp
using System;

// Hello World ใน 5 ภาษา
Console.WriteLine("สวัสดี ชาวโลก!");          // ไทย
Console.WriteLine("Hello, World!");              // อังกฤษ
Console.WriteLine("Hola, Mundo!");              // สเปน
Console.WriteLine("Bonjour, Monde!");           // ฝรั่งเศส
Console.WriteLine("Hallo, Welt!");              // เยอรมัน
Console.WriteLine("こんにちは、世界！");         // ญี่ปุ่น
Console.WriteLine("你好，世界！");               // จีน

Console.WriteLine("\n--- กด Enter เพื่อออกจากโปรแกรม ---");
Console.ReadLine();
```

### เฉลย 1.2: Interactive Greeting

```csharp
using System;

Console.WriteLine("=== โปรแกรมต้อนรับ ===\n");

// ถามชื่อ
Console.Write("ชื่อของคุณคือ: ");
string name = Console.ReadLine() ?? "ไม่ทราบชื่อ";

// ถามอายุ
Console.Write("อายุของคุณคือ: ");
string ageInput = Console.ReadLine() ?? "0";

// แปลงเป็น integer
if (int.TryParse(ageInput, out int age))
{
    Console.WriteLine($"\nสวัสดี {name}!");
    Console.WriteLine($"คุณมีอายุ {age} ปี");
    
    if (age < 18)
        Console.WriteLine("คุณยังเป็นผู้เยาว์");
    else if (age < 60)
        Console.WriteLine("คุณอยู่ในวัยทำงาน");
    else
        Console.WriteLine("คุณอยู่ในวัยเกษียณ");
}
else
{
    Console.WriteLine($"\nสวัสดี {name}! (ไม่สามารถอ่านอายุได้)");
}

Console.WriteLine("\n--- กด Enter เพื่อออกจากโปรแกรม ---");
Console.ReadLine();
```

### เฉลย 1.3: System Information

```csharp
using System;
using System.Runtime.InteropServices;

Console.WriteLine("=== ข้อมูลระบบ ===\n");

// วันและเวลาปัจจุบัน
Console.WriteLine($"วันที่และเวลา: {DateTime.Now:dddd, dd MMMM yyyy HH:mm:ss}");
Console.WriteLine($"UTC: {DateTime.UtcNow:yyyy-MM-dd HH:mm:ss} UTC");

// OS ที่ใช้
Console.WriteLine($"\nOS: {RuntimeInformation.OSDescription}");
Console.WriteLine($"Architecture: {RuntimeInformation.ProcessArchitecture}");
Console.WriteLine($"Framework: {RuntimeInformation.FrameworkDescription}");

// Computer Name
Console.WriteLine($"\nComputer Name: {Environment.MachineName}");
Console.WriteLine($"User: {Environment.UserName}");
Console.WriteLine($"Domain: {Environment.UserDomainName}");

// Hardware Info
Console.WriteLine($"\nCPU Count: {Environment.ProcessorCount}");
Console.WriteLine($"Working Set: {Environment.WorkingSet / 1024 / 1024} MB");
Console.WriteLine($"OS 64-bit: {Environment.Is64BitOperatingSystem}");

// Current Directory
Console.WriteLine($"\nCurrent Directory: {Environment.CurrentDirectory}");
Console.WriteLine($".NET Version: {Environment.Version}");

Console.WriteLine("\n--- กด Enter เพื่อออกจากโปรแกรม ---");
Console.ReadLine();
```

---

## สรุป Part 01

ในส่วนนี้เราได้เรียนรู้:

1. **C# คืออะไร** - ภาษา OOP ที่พัฒนาโดย Microsoft
2. **.NET Ecosystem** - Platform สำหรับพัฒนาแอปหลากหลายประเภท
3. **ทำไมต้องเรียน C#** - Cross-platform, Performance, Enterprise-grade
4. **การติดตั้ง** - Visual Studio 2022 และ .NET SDK
5. **Hello World** - โปรแกรมแรกของเรา
6. **โครงสร้างโปรเจกต์** - .csproj, Program.cs
7. **Comments** - Single-line, Multi-line, XML Documentation
8. **Namespaces** - การจัดการ code ให้เป็นระเบียบ
9. **dotnet CLI** - คำสั่งพื้นฐาน
10. **Compilation Process** - Source Code → IL → Native Code

## ขั้นต่อไป

ใน Part 02 เราจะเรียนรู้เกี่ยวกับ:
- ชนิดข้อมูลพื้นฐาน (Data Types)
- ตัวแปร (Variables)
- ค่าคงที่ (Constants)
- ตัวดำเนินการ (Operators)
- Type Conversion

---

*หลักสูตร C# Xamarin/.NET MAUI | Part 01 | Steps 1-10*
