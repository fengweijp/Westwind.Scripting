# Westwind CSharp Scripting
### Dynamically compile and execute CSharp code at runtime

This small `CSharpScripting` class provides an easy way to compile and execute C# on the fly at runtime using the .NET compiler services. This is ideal to provide support for addin's and application automation tasks that are user configurable.

The library supports both the latest Roslyn compiler and classic CSharp compilation.

> #### Requires Roslyn Code Providers for your Project
> If you want to use Roslyn compilation you have to make sure you add the `Microsoft.CodeDom.CompilerServices` NuGet Package to your project to provide the required compiler binaries for your application. This should be added to the application's start project.
>
> Note that this adds a sizable chunk of files to your application's output folder in the `roslyn` folder. If you don't want this you can use the classic compiler, at the cost of not having access to C# 6+ features.

## Usage
Using the `CSharpScriptExecution` class is very easy. It works by letting you provide either a simple code snippet that can optionally `return` a value, or a complete method signature with a method header and return statement. You can also provide multiple method that can be called explicitly using the `InvokeMethod()` operation.

You can add **Assembly References** and **Namespaces** via the `AddReferece()` and `AddNamespace()` methods.

Script Execution is gated rather than throwing directly to provide 

### Executing a Code Snippet
A code snippet can be **any block of .NET code** that can be executed. You can pass any number of parameters to the snippets which are accessible via a `parameters` object array. You can optionally `return` a value by providing a `return` statement.


```cs
var script = new CSharpScriptExecution()
{
    SaveGeneratedCode = true,
    CompilerMode = ScriptCompilerModes.Roslyn
};
script.AddDefaultReferencesAndNamespaces();

//script.AddAssembly("Westwind.Utilities.dll");
//script.AddNamespace("Westwind.Utilities");

var code = $@"
// Check some C# 6+ lang features
var s = new {{ name = ""Rick""}}; // anonymous types
Console.WriteLine(s?.name);       // null propagation

// pick up and cast parameters
int num1 = (int) @0;   // same as parameters[0];
int num2 = (int) @1;   // same as parameters[1];

// string templates
var result = $""{{num1}} + {{num2}} = {{(num1 + num2)}}"";
Console.WriteLine(result);

return result;
";

string result = script.ExecuteCode(code,10,20) as string;

Console.WriteLine($"Result: {result}");
Console.WriteLine($"Error: {script.Error}");
Console.WriteLine(script.ErrorMessage);
Console.WriteLine(script.GeneratedClassCodeWithLineNumbers);

Assert.IsFalse(script.Error, script.ErrorMessage);
Assert.IsTrue(result.Contains(" = 30"));
```

Note that the `return` in your code snippet is optional - you can just run code without a result value. Any parameters you pass in can be accessed either via `parameters[0]`, `parameters[1]` etc. or using a simpler string representation of `@0`, `@1`.

### Evaluating an expression
If you want to evaluate a single expression, there's a shortcut `Evalute()` method that works pretty much the same:

```cs
var script = new CSharpScriptExecution()
{
    SaveGeneratedCode = true,
};
script.AddDefaultReferencesAndNamespaces();

// Full syntax
//object result = script.Evaluate("(decimal) parameters[0] + (decimal) parameters[1]", 10M, 20M);

// Numbered parameter syntax is easier
object result = script.Evaluate("(decimal) @0 + (decimal) @1", 10M, 20M);

Console.WriteLine($"Result: {result}");
Console.WriteLine($"Error: {script.Error}");
Console.WriteLine(script.ErrorMessage);
Console.WriteLine(script.GeneratedClassCode);

Assert.IsFalse(script.Error, script.ErrorMessage);
Assert.IsTrue(result is decimal, script.ErrorMessage);
```            

This method is a shortcut wrapper and simply wraps your code into a single line `return {exp};` statement.

### Executing a Method
Another way to execute code is to provide a full method body which is a little more explicit and makes it easier to reference parameters passed in. 

```csharp
var script = new CSharpScriptExecution()
{
    SaveGeneratedCode = true,
    CompilerMode = ScriptCompilerModes.Roslyn,
    GeneratedClassName = "HelloWorldTestClass"
};
script.AddDefaultReferencesAndNamespaces();

string code = $@"
public string HelloWorld(string name)
{{
string result = $""Hello {{name}}. Time is: {{DateTime.Now}}."";
return result;
}}";

string result = script.ExecuteMethod(code,"HelloWorld","Rick") as string;

Console.WriteLine($"Result: {result}");
Console.WriteLine($"Error: {script.Error}");
Console.WriteLine(script.ErrorMessage);
Console.WriteLine(script.GeneratedClassCode);

Assert.IsFalse(script.Error);
Assert.IsTrue(result.Contains("Hello Rick"));
```

### More than just a Single Method
Note that you can provide more than a method in the code block - you can provide an entire **class body** including additional methods, properties/fields, events and so on. Effectively you can build out an entire class this way. After intial execution you can access the `ObjectInstance` member and use either Reflection or `dynamic` to access the functionality on that class.

```cs
var script = new CSharpScriptExecution()
{
    SaveGeneratedCode = true,
    CompilerMode = ScriptCompilerModes.Roslyn
};
script.AddDefaultReferencesAndNamespaces();

// Class body with multiple methods and properties
string code = $@"
public string HelloWorld(string name)
{{
string result = $""Hello {{name}}. Time is: {{DateTime.Now}}."";
return result;
}}

public string GoodbyeName {{ get; set; }}

public string GoodbyeWorld()
{{
string result = $""Goodbye {{GoodbyeName}}. Time is: {{DateTime.Now}}."";
return result;
}}
";

string result = script.ExecuteMethod(code, "HelloWorld", "Rick") as string;

Console.WriteLine($"Result: {result}");
Console.WriteLine($"Error: {script.Error}");
Console.WriteLine(script.ErrorMessage);
Console.WriteLine(script.GeneratedClassCode);

Assert.IsFalse(script.Error);
Assert.IsTrue(result.Contains("Hello Rick"));

// You can pick up the ObjectInstance of the generated class
// Make dynamic for easier access
dynamic instance = script.ObjectInstance;

instance.GoodbyeName = "Markus";
result = instance.GoodbyeWorld();

Console.WriteLine($"Result: {result}");
Assert.IsTrue(result.Contains("Goodbye Markus"));
```