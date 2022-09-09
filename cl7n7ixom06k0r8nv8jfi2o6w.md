## Benchmark Programs in C#

## Overview
**Problem.** 
You are wondering if the C# programming language is fast. You question whether it is worth benchmarking programming languages. What else can you learn from careful benchmarks?  
**Solution.** Here’s a list of benchmarking notes using the C# programming language that touches on some important things about benchmarking.

## Tips
- Use the programs here to perform micro-benchmarks.
- Contextualize these results in the scale of other operations.
- Do not spend 99% of your time optimizing code that runs 1% of the time.

## Benchmarking using C# #
Let's start with simple benchmarking using console application.  
Open a command prompt by pressing `Windows log key + R` and enter `cmd`.  

Enter the following commands to create a project called `mybenchmark`:
```batch
dotnet new console --name mybenchmark
```  

Open in Visual Studio Code:

```batch
cd mybenchmark
code .
```  

Open `Programs.cs` file and replace with code below:

```csharp
using System.Diagnostics;

const int _max = 1000000; 

var s1 = Stopwatch.StartNew(); 
for (int i = 0; i < _max; i++) 
{ 
    //Your custom code here
} 
s1.Stop(); 

var s2 = Stopwatch.StartNew(); 
for (int i = 0; i < _max; i++) 
{ 
    //Your custom code here
} 
s2.Stop(); 

Console.WriteLine(((double)(s1.ElapsedMilliseconds * 1000 * 1000) / _max).ToString("0.00") + " ns"); 
Console.WriteLine(((double)(s2.ElapsedMilliseconds * 1000 * 1000) / _max).ToString("0.00") + " ns"); 

Console.Read(); 

```  

Back to the command prompt, enter the following command to build the project *(make sure you are still in the project directory)*:
```batch
dotnet build
```  

Execute `.exe` directly to see the output:
```batch
.\bin\Debug\net6.0\mybenchmark.exe
```

Output *(your output result may vary)*
> 1.70 ns  
1.64 ns  

Here, we measure elapsed time of 2 `for` loops. Inside each loop, we put our custom code in each loop blocks. It can be a function name, code snippet from another code, etc, depending on your needs.  It is not necessary to have 2 loops, in fact, we should have more loops to measure average performance.

Note that we use `Stopwatch' instance to measure elapsed time and it is one of the common ways to measure performance. To check total milliseconds, we simply refer to a property `.ElapsedMilliseconds` of a `Stopwatch` class instance.  

You can also change value of `_max` depending how slow the iterations are. It is recommended to start with smaller value and push the limit up.

Here we look at the kinds of loops that Dot Net Perls benchmarks are usually done with. This simple framework is what the author does his experiments with. He currently uses Vista which has much more accurate Stopwatch measurements in my experience. You will want to change m depending on the code of each iteration; it is best to start smaller and push the limit up.

However, there is a small problem here. We build our application using *Debug* build configuration. *Debug* is a default build configuration when we run `dotnet build`. Instead, we should use *Release* configuration when we really want to measure the actual performance of our code. The output from *Release* build configuration has minimal symbolic debug information and is fully optimized. 

To build in *Release* configuration, enter the following command:

```batch
dotnet publish --configuration Release
```  

Execute `.exe` in *publish* folder to see the output:
```batch
.\bin\Release\net6.0\publish\mybenchmark.exe
```

Output
> 0.28 ns  
0.24 ns  

From the output, we can see that elapsed time has been drastically reduced. However, It doesn't mean that our application has better performance. It only shows us the actual elapsed time when we deploy our code to production. `Release` configuration will give us almost-to-production metric so that we can properly fine tune our custom code.

##  Optimise too early

Most of the time during development, we shouldn't optimise our code too early, especially on code that is not run often. If we optimise too early, we are only going to make it harder to understand and maintain.

Is benchmarking important? Yes, however, it is often not important in a practical sense for your current project. It encourages you to examine your code and find out what it is really doing.

## Natural languages

Here we look at optimization from another point of view. As I have progressed as a writer, I have worked on making my **English** clearer and shorter to read. I think of this as optimization in the same way as with the C# language.

###  Slow English
>What I want to show to you next is this code, which adds one to a variable in each iteration of the loop.  

Example:
```csharp
var j = 0;
for(int i = 0; i < 10; i++)
{
    j = j + 1;
}
```

### Fast English
>The loop increments the variable.  

Example:
```csharp
int j;
for(j = 0; j < 10; j++){}
```

Which would you rather read?  

## Summary
Here we looked at example code for **benchmarking** the C# programming language. Benchmarking encourages careful thinking about your code. It saves milli\nanoseconds from your software but greatly improves the depth of your understanding. 

Did you find this article useful? If you have any feedback or questions, please let me know in the comments below.

Thank you for reading and happy coding!