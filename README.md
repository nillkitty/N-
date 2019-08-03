# **Convenience and Uniformity**
## **Try Expressions**
You can now create a **try expression**.     The expression following the `try` operator is evaluated and if an exception is thrown during evaluation, `default` is returned.
``` csharp
Net.IPAddress ip = try Net.IPAddress.Parse(ipTextBox.Text);
if (ip == null) {
    MessageBox.Show("Invalid IP address!");
}
```
You can also have try blocks without a `catch` or `finally`.   These are functionally equivalent to having an empty `catch` block in C#.
``` csharp
try {
   BestEffortTask();
}
Console.WriteLine("Task attempt completed");
```
## **XML Literals**
VB.NET has had these for decades.
``` csharp
var xml = <customer id="2">
             <name>John Doe</name>
             <email>john.doe@example.net</email>
          </customer>;

// Update some values
xml.<name>.Value = "John Doe, Jr.";
xml.@id = 3;
xml.Add(<company>Doe and Son Donuts</company>);

Debug.Write(xml.GetType().Name);    // Output: System.Xml.XElement
```
## **DateTime Literals**
This is common in other languages as well.
``` csharp
public long GetUnixTime(DateTime date)
{
    DateTime epoch = #1/1/1970#;
    return (long)date.Subtract(epoch).TotalSeconds;
}
```

## **Arbitrary Restrictions Removed**
* You can now reassign the value of local parameters that are not declared `in` or `readonly`, as well as loop iterator variables
* `try`, `catch`, `finally`, `using`, `lock`, etc. no longer require code blocks,  they can be used on single statements

## **Arbitrary Operator Creation**

# **Property Changes**
## **Property-Scoped Variables**
If you have a variable that should only be modified through a property,  you can define the variable within the scope of the property to effectively hide it from being used by unrelated code without accessing it through the property.
``` csharp
public class Office {
    public string Name {
        string name;

        get => name;
        set {
            if (value?.Length > 30) throw new ArgumentException("Property value too long.");
            name = value;
        }
    }
}
```
Note that even in the most basic validation case,  a lot of boilerplate code is required just to marshal between the internal storage variable and the `value` keyword or the return value of the `get` block.   This brings us to the next feature...

## **Improved Auto Properties**
In C#, if a property defines a non-automatic getter, it may not use an automatic setter, and vice-versa.   We can re-write the previous example like this:

``` csharp
public class Office {
     public string Name {
        get;
        set {
            if (value?.Length > 30) throw new ArgumentException("Property value too long.");
            return value;
        }
     }
}
```
If you write a setter that returns a value,  instead of exiting the setter,  it sets the storage variable of the property to the supplied value and exits.   If you use `return` without a value, or an exception is thrown, the `set` does not impact the storage variable.

Some more examples:
``` csharp
public partial class NutritionFacts {
    public int Calories 
    {
        get; 
        set => Math.Max(0, value);
    }
    public IngredientList Ingredients 
    {
        bool cached;
        get
        {  
            if (cached) return value;
            value = GetIngredients();
            cached = true;
            return value;
        }
        set { ValidateIngredients(value); cached = true; return value; }
    }
    public int DateCode 
    {
        DateTime value;  /* Declares the storage variable as a DateTime */
        get => value.ToString("yyMM");
        set => DateTime.Parse(value, "yyMM");
    } = 1905; /* And you can still set an initial value (which calls the setter) */

    public int DateCode2
    {
        /* Setting the initial value here sets the storage variable only;
         * No setter code is called */
        DateTime value = #5/1/2019#;

        get => value.ToString("yyMM");
        set => DateTime.Parse(value, "yyMM");
    }


}
```



# **Code Security Changes**
## **Access Modifier Overloading**
In traditional C#, access modifiers are not taken into account when resolving a method call to a particular overload.   Thus, you cannot do:
``` csharp
public void Log(string text) 
{ 
    if (!AllowUserLogEntries) return;   // Silened
    if (UserLogCount++ >= UserLogThreshold) return; // Throttled

    Platform.Current.Log(Name, text, this);
}
internal void Log(string text) 
{ 
    // Internal components are never restricted from logging
    Platform.Current.Log(Name, text, this);
}
```
This will result in a compilation error because the method signatures are otherwise identical.   Thus, you cannot have a `Log` for public use and a `Log` for internal use without explicitly naming them different things.   You might want to have special behavior when the method is called from outside your assembly versus inside, such as in this case (where security or feature checks should take place when called from unknown code);  but the only way to do that in C# is to define two differently-named methods with different access modifiers.

In N#,  however,  this code is perfectly valid.    If a call to `Log` is made from the same assembly, the `internal` method body is used,  if a call is made to `Log` outside the current assembly, the `public` one is used and the additional checks occur.  

## **Code Security Blocks**
In the above example, we provided two implementations for `Log` with differing access modifiers,  but in many cases the two implementations sharing the same name typically do similar (or identical) things.   **Code security blocks** allow you to guard a block of code based on the caller's access level.  

``` csharp
/* Unified function */
public void Log(string text)  
{
    public {
       if (!AllowUserLogEntries) return;   // Silened
       if (UserLogCount++ >= UserLogThreshold) return; // Throttled
    }

    Platform.Current.Log(Name, text, this);
}
```
| Block | Description |
|-------|-------|
| `base` | The caller is the base class of this class
| `this` | The caller is in the same class, and in an instance member of the same instance
| `private` | The caller is in the same class and in a different instance (or either member is static)
| `protected` | The caller is in a derived class of this one
| `internal` | The caller is in the same assembly as this one
| `public` | The caller is in an external assembly

Access modifiers can be combined (as an **or** relationship):
``` csharp
public internal {
    // Code when called from outside this class
}
protected private this {
   // Code when called from this or a derived class
}
base {
   // Code when called by our base class
}
```
## **Security Modifiers for Arguments**
Take the following example:
``` csharp
public void EnterFunction(string functionName, InternalSecurityContext context)
{
   internal private this {
      if (context != null) context.PushCurrent();
   } 

   if (context == null) 
      context = InternalSecurityContext.Current;

   public {
      context.DoSecurityCheck(this, functionName);
   }

   Log($"Entered:  {functionName}");
}
```
With code security blocks,  we can ensure that calls to our API functions from other assemblies carry out necessary security checks while saving drastic code by allowing is to no longer need both an internal and private version of each method or property that has a behavioral difference.   Additionally,  this function allows internal callers to supply an explicit security context.

There are, however, two issues with this code, as-is:

1. Because the method signature is `public`, the `context` parameter is visible to public consumers of this API even though the value of `context` is not used by `public` code.   Additionally,  if the `InternalSecurityContext` type is `internal`, this code can't be built because it would be exposing an `internal` type to through a `public` method.

2. The `this` or `private` paths of the code security blocks will always execute because this function is intended to be called at the beginning of other functions on this class;  it's the *caller of the caller* in which we're interested in this case,  so we need a way to make the call to this function transparent to the code security evaluation within the method.

N# solves both problems right in the method signature.   Here's the new signature:
``` csharp
public passthru void EnterFunction(string functionName, internal InternalSecurityContext context)
{
   // ** snip **
}
```
The new `passthru` modifier indicates that calls to this function won't update the current **security level**;   code security blocks such as `private { }` will be working on the caller of the method calling this function.   This is typically used for small utility functions that are really dealing with the nature of the function calling them (such as in this case, a tracing function).

The `internal` modifier on the `context` argument does exactly what you'd expect:
* Classes outside the `internal` scope don't see this parameter, and it's value is set to `default` within the scope of the method.
* The type of the argument can be less public than the method signature itself, as long as it's not more public than the modifier used on it.
* Default argument values are used if an argument is hidden due to code security

**Note:** Multiple methods with otherwise the same signature cannot vary only by argument modifiers

## **Better Type Constraints**
In N#, you can use several new types of type constraints for generic types which will result in compile-time and/or run-time checking of the validity of the types passed into it.

``` csharp
public bool DisposeEnumerators(IEnumerable<t> input) where t : IEnumerator<>, IDisposable { }
/* t must implement IEnumerator as well IDisposable */

public bool PublishToInternet(t object) where t : public { }
/* t must be a public type */

public bool LicenseComponent(t component) where t :: Microsoft.Office { }
/* t must be a type in namespace Microsoft.Office */

```

## **Better pattern matching**
N# has several new patterns to make classifying weakly typed objects more efficient to process.
``` csharp
if (x is [Serializable] attr) {
    /* x is of a type that is marked with a SerializableAttribute (which attr now holds a reference to) */
}

if (ex is Exception e && e is :: System.Net.Sockets) {
    Debug.WriteLine($"Exception in the socket library: {e}");
}
```
You can also use type constraints as patterns.
``` csharp
void CloneAndThrow<t>(t exception) where t : Exception {
    if (exception is : new(string, Exception)) {
        /* The type t has a constructor that accepts a string an Exception */
        throw new t(exception.Message, exception);
    } else if (exception is : new(string)) {
       /* The type t has a constructor that accepts a single string */
       throw new t(exception.Message);
   } else if (exception is : new()) {
      /* The type t has a parameterless constructor */
       throw new t();
   }
}
```
## **Regex literals and the 'like' operator**
A token beginning with `/`, and ending with `/`, followed by one or more alphabetic flag characters, is interpretted as a literal regular expression and can be assigned to any variable that `System.Text.RegularExpressions.Regex` can be assigned to.

``` csharp
var x = /abc/; // regex
if (x.Match("abc")) Console.WriteLine($"Match at {x.Index}"); 

var y = "ab?"; // wildcard

if ("abc" like x) { /* Same as (x.Match("abc")?.Success) */ }
if ("abc" like y) { /* Similar to this usage in VB.NET when used against a string */ }
```
The `like` operator can also be used to compare two types.  `a like b` is true if `a` and `b` are both `System.Type` and `a` is assignable to `b`.

``` csharp
object CreateCustomLogEntry(LogEntry e, Type logType) {

    if (logType like string) {
       return e.ToString();
    } else if (logType is : new(LogEntry e)) {
       return Activator.CreateInstance(logType, new object[] { e });
    }
}
```

# **Aspect Oriented Programming**
## **Pointcuts and join points**
A **pointcut** is a block of code tied to a **join point**.  Consider the following code, which makes calls to the `EnterFunction` function we used an example above:
``` csharp
public int Connect(string hostname, int port) {
    EnterFunction(nameof(Connect));
    LogSocketActivity(nameof(Connect), $"{hostname}:{port}");

   // ...
}
public bool Disconnect(int reason) {
    EnterFunction(nameof(Disconnect));
    LogSocketActivity(nameof(Disonnect), "");

   // ...
}
public bool Reconnect() {
    EnterFunction(nameof(Reconnect));
    LogSocketActivity(nameof(Reconnect), "");

   // ...
}
public int SendMessage(string text) {
    EnterFunction(nameof(SendMessage));
    
  // ...
}
```
As you can see we're using `EnterFunction` to do some trace logging as well as security checks;  wouldn't it be nice if we could just "hook-up" the `EnterFunction` method to the "beginning of every public function in this class"?  Well you can with N#:
``` csharp
before public (*) {
   EnterFunction(here.Method.Name);
}
```
This called an **pointcut** and it can be written as expression-bodied as well:
``` csharp
before public *Connect(*) 
     => LogSocketActivity(here.Method.Name, here.Arguments.ToString());
```
And likewise there's the `after` keyword:
``` csharp
after public (*) {
   ExitFunction(here.Method.Name);
}
```
The **before** and **after** keywords indicate that you are declaring a pointcut;  the signature of the pointcut determines which **join points** the pointcut will handle.   There are several ways pointcuts can be declared:

``` csharp
// Before all function calls and property accessers
before * { }   

// Before all function calls only
before *(*) { }

// Before all parameterless function calls
before *() { }

// Before all property accesses
before *[*] { }

// Before all parameterless property accesses
before *[*] { }

// Before all property "gets" or "sets"
before *[*] get { }
before *[*] set { }

// Before indexer "gets" or "sets"
before this[*] get { }
before this[*] set { }

// Before all public functions
before public *(*) { }

// After any constructor runs
after new(*) { }

// After the parameterless constructor runs
after new() { }

// Before the destructor/finalizer runs
before ~() { }

// After a public property "set" occurs
after *[*] set => NotifyPropertyChanged();
```
Inside of a pointcut:
* Execution reaching the end of the pointcut or a `break` statement will return to the original flow, or to the next pointcut if multiple pointcuts match a join point.
* A `return` statement will return out of the original context (property accessor or function body) as if it were used directly within the method body or property accessor;  can only be used when a pointcut defines an explicit return type.

## **Explicit Joinpoints**
In addition to function calls and property accesses, you can define arbitrary join points in your code and then create pointcuts to target code to run `before` or `after` them.

When a join point is encountered,  pointcuts are run in the following order:
* All before calls, from most-recently defined to least-recently defined
* All after calls, from least-recently defined to most-recently defined

Join points are declared using the `join` keyword;
``` csharp
public bool ProcessRecord() {
   double Amount = Employee.PayrollBalance;
   if (Amount > 0) {
      join PayingEmployee(Employee, Amount);
      Employee.Pay(Amount);
      return true;
   }
   return false;
}
```
Join points declare a name (`PayingEmployee`) in this case, and one or more optional arguments of any type (in this case `Employee` and `Amount` ar passed as arguments)

A pointcut can be tied to this join point as follows:
``` csharp
before bool ProcessRecord.PayingEmployee(Employee e, double amt) {
   if (amt > 50000.00) {
      return false;   // Returns out of the ProcessRecord() function
   } 
}
```
## **Aspects**
So far we've been declaring our pointcuts within the same class as the join points,  but we could also create an **aspect** to house pointcuts that are **cross-cutting**, in other words, they are ambient and are associated more with a system-wide service rather than any one specific object type.  Behaviors like logging, security checks, auditing, diagnostics, and performance recording are examples of cross-cutting functions.

Aspects can be used to define pointcuts that can target join points anywhere in the same namespace.  An aspect is defined using the `aspect` keyword.  Aspects do not become Types and thus have no access modifiers.
``` csharp
aspect Logging {
   before public *(*) { 
       /* Called before all public function calls on all types in the namespace */
       Debug.WriteLine($"Enter: {here.Method.Name}");
   }

  before public Contoso.Sample (*) {
      /* Called before all public function calls on all types in the Conoso.Sample namespace
         only */
  }

  before public Contoso.Sample.*(*) {
      /* Called before all public function calls on all types in the Conoso.Sample namespace
         and nested namespaces */
  }

  before public t.*(*) where t : class {
      /* Called before all public function calls on class (reference) types */
  }

  before public t.*(*) where t :: Contoso.Sample {
      /* Called before all public function calls on class (reference) types only
         within the Contoso.Sample namespace */
  }

}
```

