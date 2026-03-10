# Lane Knowledge: VB.NET → C# / ASP.NET Core MVC

Framework-specific migration guidance for VB.NET + WebForms / WinForms to C# + ASP.NET Core MVC.

## Syntax Map Table

| VB.NET Syntax | C# Equivalent | Notes |
|--------------|---------------|-------|
| `Module ModuleName` | `public static class ModuleName` | All members become `static` |
| `Dim x As Integer` | `int x;` or `var x = 0;` | Use `var` when type is obvious |
| `Dim x As New List(Of String)()` | `var x = new List<string>();` | Generic syntax changes |
| `Sub MethodName()` | `void MethodName()` | No return value |
| `Function MethodName() As String` | `string MethodName()` | Return type before name |
| `ReDim Preserve arr(n)` | `Array.Resize(ref arr, n + 1)` | 0-indexed; new size = n+1 elements |
| `With obj ... End With` | Use object directly in scope block | Or local variable `var o = obj;` |
| `If ... Then ... ElseIf ... Else ... End If` | `if (...) { } else if (...) { } else { }` | Braces required |
| `For i = 0 To n - 1` | `for (int i = 0; i < n; i++)` | |
| `For Each item In collection` | `foreach (var item in collection)` | |
| `Try ... Catch ex As Exception ... Finally ... End Try` | `try { } catch (Exception ex) { } finally { }` | |
| `String.Format("{0}", x)` | `$"{x}"` | Prefer C# interpolation |
| `& ` (string concat operator) | `+` or `$"..."` | |
| `Is Nothing` | `== null` | |
| `IsNot Nothing` | `!= null` | Or `is not null` (C# 9+) |
| `Not condition` | `!condition` | |
| `AndAlso` | `&&` | |
| `OrElse` | `\|\|` | |
| `CBool(x)` / `CInt(x)` / `CStr(x)` | `(bool)x` / `(int)x` / `x.ToString()` | Or `Convert.To*` |
| `Optional param As Type = default` | `Type param = default` | C# default parameters |
| `ByRef param As Type` | `ref Type param` | Pass by reference |
| `ByVal param As Type` | `Type param` | Pass by value (default in C#) |
| `Property Name As Type` | `public Type Name { get; set; }` | Auto-property |
| `Inherits BaseClass` | `: BaseClass` | |
| `Implements IInterface` | `: IInterface` | |
| `MustInherit` class | `abstract` class | |
| `MustOverride` method | `abstract` method | |
| `Overrides` method | `override` method | |
| `Overridable` method | `virtual` method | |
| `Shared` member | `static` member | |
| `ReadOnly` property/field | `readonly` field / get-only property | |
| `Friend` access modifier | `internal` | |
| `Protected Friend` | `protected internal` | |

## WebForms → ASP.NET Core MVC

| WebForms Pattern | ASP.NET Core MVC Equivalent |
|-----------------|------------------------------|
| `Page.aspx` + `Page.aspx.vb` codebehind | `Controller.cs` + `View.cshtml` (Razor) |
| `Page_Load(sender, e)` | Controller action method |
| `Button1_Click(sender, e)` | Controller action with `[HttpPost]` + `[ValidateAntiForgeryToken]` |
| `Session["key"]` | `IHttpContextAccessor.HttpContext.Session["key"]` or `TempData["key"]` |
| `Response.Redirect("url")` | `return RedirectToAction("ActionName", "ControllerName")` |
| `Request.QueryString["id"]` | Action parameter `int id` (model binding) |
| `Request.Form["field"]` | `[FromForm] string field` or model binding |
| `GridView.DataSource = ...` | `return View(model)` → iterate in Razor with `@foreach` |
| `Label1.Text = "..."` | Razor `@Model.PropertyName` |
| `TextBox1.Text` | `@Html.TextBoxFor(m => m.Property)` or `<input asp-for="Property">` |
| `DropDownList1.SelectedValue` | `<select asp-for="Property" asp-items="...">` |
| `ValidationSummary` | `<div asp-validation-summary="All">` |
| Global.asax `Application_Start` | `Program.cs` / `Startup.cs` service registration |
| `web.config` `<appSettings>` | `appsettings.json` + `IConfiguration` |
| `ScriptManager` + UpdatePanel | JavaScript fetch API / AJAX |
| Master Pages | `_Layout.cshtml` |
| User Controls (.ascx) | Partial views or Razor Components |

## String Handling Patterns

```vb
' VB.NET
Dim msg As String = "Hello, " & name & "! You have " & count.ToString() & " items."
```
```csharp
// C#
string msg = $"Hello, {name}! You have {count} items.";
```

## VB Optional Parameters → C# Default Parameters

```vb
' VB.NET
Public Function GetOrders(Optional ByVal status As String = "ACTIVE",
                           Optional ByVal page As Integer = 1) As List(Of Order)
```
```csharp
// C#
public List<Order> GetOrders(string status = "ACTIVE", int page = 1)
```

## Context7 Lookup Map

When implementing, use Context7 to look up:
- `ASP.NET Core MVC` — controllers, actions, model binding, routing
- `.NET Core migration` — `IConfiguration`, `appsettings.json`, DI container (`IServiceCollection`)
- `C# syntax` — pattern matching, records, nullable reference types, LINQ
- `Razor Pages` — if migrating to Razor Pages instead of MVC
- `ASP.NET Core Identity` — if migrating Forms Authentication
- `Entity Framework Core` — if migrating ADO.NET or LINQ to SQL

## Quality Gate Checklist

Before approving the transform phase, verify:
- [ ] All VB.NET-specific syntax converted (no `Dim`, `Sub`, `Function`, `Imports` remaining)
- [ ] `Module` classes converted to `static class`
- [ ] WebForms codebehind events converted to controller actions with proper HTTP verb attributes
- [ ] `Session["key"]` replaced with `IHttpContextAccessor` or `TempData` patterns
- [ ] `Response.Redirect` replaced with `RedirectToAction` or `Redirect`
- [ ] `web.config` `<appSettings>` migrated to `appsettings.json` + `IConfiguration`
- [ ] ViewState patterns removed — use `TempData` for redirect data, model binding for form data
- [ ] `Global.asax` startup code moved to `Program.cs` / `Startup.cs`
- [ ] All `Optional` parameters converted to C# default parameter syntax
- [ ] String concatenation using `&` replaced with `$"..."` interpolation
- [ ] Null checks using `Is Nothing` / `IsNot Nothing` updated to `== null` / `!= null`

## Common Pitfalls

1. **1-based arrays in VB.NET**: VB.NET supports `Option Base 1` — check all array indexing. C# is always 0-based.
2. **Late binding (`Object` type)**: VB.NET `Option Strict Off` allows late binding. In C#, make all types explicit or use `dynamic` sparingly.
3. **Date/Time formatting**: VB.NET `Format(date, "dd/MM/yyyy")` → C# `date.ToString("dd/MM/yyyy")`.
4. **Integer division**: VB.NET `\` is integer division; `/` is floating point. In C#, `/` on `int` is integer division.
5. **Error handling**: VB.NET `On Error GoTo` → C# `try/catch`. No direct equivalent for `Resume Next`.
6. **Event wiring**: WebForms auto-wires events by naming convention. In MVC, events become POST actions with `[HttpPost]`.
7. **ViewState**: WebForms ViewState is implicit state management. In MVC, all state must be explicit (hidden fields, TempData, DB).
