# Simple programs

* [x] Proposed
* [x] Prototype: Started
* [ ] Implementation: Not Started
* [ ] Specification: Not Started

## Summary
[summary]: #summary

Allow a sequence of *statements* to occur right before the *namespace_member_declaration*s of a *compilation_unit* (i.e. source file).

The semantics are that if such a sequence of *statements* is present, the following type declaration, modulo the actual type name and the method name, would be emitted:

``` c#
static class Program
{
    static async Task Main()
    {
        // statements
    }
}
```

See also https://github.com/dotnet/csharplang/issues/3117.

## Motivation
[motivation]: #motivation

There's a certain amount of boilerplate surrounding even the simplest of programs,
because of the need for an explicit `Main` method. This seems to get in the way of
language learning and program clarity. The primary goal of the feature therefore is
to allow C# programs without unnecessary boilerplate around them, for the sake of
learners and the clarity of code.

## Detailed design
[design]: #detailed-design

### Syntax

The only additional syntax is allowing a sequence of *statement*s in a compilation unit,
just before the *namespace_member_declaration*s:

``` antlr
compilation_unit
    : extern_alias_directive* using_directive* global_attributes? statement* namespace_member_declaration*
    ;
```

In all but one *compilation_unit* the *statement*s must all be local function declarations. 

Example:

``` c#
// File 1 - any statements
if (args.Length == 0
    || !int.TryParse(args[0], out int n)
    || n < 0) return;
Console.WriteLine(Fib(n).curr);

// File 2 - only local functions
(int curr, int prev) Fib(int i)
{
    if (i == 0) return (1, 0);
    var (curr, prev) = Fib(i - 1);
    return (curr + prev, curr);
}
```

### Semantics

If any top-level statements are present in any compilation unit of the program, the meaning is as if
they were combined in the block body of a `Main` method of a `Program` class in the global namespace,
as follows:

``` c#
static class Program
{
    static async Task Main()
    {
        // File 1 statements
        // File 2 local functions
        // ...
    }
}
```

Note that the names "Program" and "Main" are used only for illustrations purposes, actual names used by
compiler are implementation dependent and neither the type, nor the method can be referenced by name from
source code.

The method is designated as the entry point of the program. Explicitly declared methods that by convention 
could be considered as an entry point candidates are ignored. A warning is reported when that happens. It is
an error to specify `-main:<type>` compiler switch.

If any one compilation unit has statements other than local function declarations, statements from that
compilation unit occur first. This causes it to be legal for local functions in one file to reference
local variables in another. The order of statement contributions (which would all be local functions)
from other compilation units is undefined.

If `await` expressions and other async operations are omitted, no warning is produced. Instead the
signature of the generated entry point method is equivalent to 
``` c#
    static void Main()
```

The example above would yield the following `$Main` method declaration:

``` c#
static class $Program
{
    static void $Main()
    {
        // Statements from File 1
        if (args.Length == 0
            || !int.TryParse(args[0], out int n)
            || n < 0) return;
        Console.WriteLine(Fib(n).curr);
        
        // Local functions from File 2
        (int curr, int prev) Fib(int i)
        {
            if (i == 0) return (1, 0);
            var (curr, prev) = Fib(i - 1);
            return (curr + prev, curr);
        }
    }
}
```

At the same time an example like this:
``` c#
// File 1
await System.Threading.Tasks.Task.Delay(1000);
System.Console.WriteLine("Hi!");
```

would  yield:
``` c#
static class $Program
{
    static async Task $Main()
    {
        // Statements from File 1
        await System.Threading.Tasks.Task.Delay(1000);
        System.Console.WriteLine("Hi!");
    }
}
```

### Scope of top-level local variables and local functions

Even though top-level local variables and functions are "wrapped" 
into the generated entry point method, they should still be in scope throughout the program.
For the purpose of simple-name evaluation, once the global namespace is reached:
- First, an attempt is made to evaluate the name within the generated entry point method and 
  only if this attempt fails 
- The "regular" evaluation within the global namespace declaration is performed. 

This could lead to name shadowing of namespaces and types declared within the global namespace
as well as to shadowing of imported names.

If the simple name evaluation occurs outside of the top-level statements and the evaluation
yields a top-level local variable or function, that should lead to an error.

In this way we protect our future ability to better address "Top-level functions" (scenario 2 
in https://github.com/dotnet/csharplang/issues/3117), and are able to give useful diagnostics 
to users who mistakenly believe them to be supported.

