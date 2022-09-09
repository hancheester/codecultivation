## Why I love Auto-Implemented Properties?

In C# 3.0 and later, auto-implemented properties make property-declaration more concise when no additional logic is required in the property accessors. They also enable client code to create objects. When you declare a property as shown in the following example, the compiler creates a private, anonymous backing field can only be accessed through the property’s get and set accessors.

```csharp
// Auto-Implemented Property for trivial get and set
public double TotalPurchases { get; set; }
```  

Equivalent to:

```csharp
// Auto-Implemented Property for trivial get and set
private double _totalPurchases;
public double TotalPurchases
{
  get { return _totalPurchases; }
  set { _totalPurchases = value; }
}
```

As you can see the above, it is a nice syntactic sugar which allows you to write shorter code.

Did you find this article useful? If you have any feedback or questions, please let me know in the comments below.

Thank you for reading and happy coding!