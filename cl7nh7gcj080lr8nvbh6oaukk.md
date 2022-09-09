## My journey of using Join, GroupJoin and GroupBy – Part 1

As I was doing a migration process to convert existing project from ASP.NET Web Form to ASP.NET MVC using Entity Framework, I came across a bunch of SQL statements. Yeah, I know, we should not have any SQL statements in C# codes. Thus, while I was going thru all my codes to remove all SQL statements.

Now, let’s come back here. Most of the SQL statements were relatively easy to handle until I came across 2 types of SQL operations which gave me a bit of head scratch. They are SQL (INNER) JOIN and GROUP BY which are pretty common in SQL statements.

I needed to join 3 tables grouped by an identifier and trying to find the minimum and maximum prices for each product. Take a look at the original SQL statement below:

    SELECT PC.ProductId, MIN(PP.Price), MAX(PP.Price) 
    FROM ProductPrices PP WITH(NOLOCK)
        JOIN ProductCategories PC WITH(NOLOCK) ON PP.ProductId = PC.ProductId
        JOIN Products P WITH(NOLOCK) ON PC.ProductId = P.Id
    WHERE PC.CategoryId IN (76, 77)
        AND P.Enabled = 1
        AND PP.Enabled = 1
    GROUP BY PC.ProductId

Fortunately, LINQ was the answer to remove SQL statement from my C# codes. LINQ was added to .NET Framework 3.5 and you read more LINQ here. From the SQL statement, I divided my problem into 3 stages. It was worth to mention that I was using a fantastic tool called [LINQPad](http://www.linqpad.net/) to help in this process. It helped testing my LINQ statements before applying them in my C# codes. To read more about this tool, please [read here](http://www.linqpad.net/) or [download the free edition here](http://www.linqpad.net/Download.aspx) to start with.

### 1st stage: SQL Join

The first problem I needed to solve was to find a way to join all 3 tables. Noticed that I used (INNER) JOIN here to retrieve records that have matching values in all 3 tables. In LINQ, I found out that Queryable.Join is one of the ways to achieve my goal here. Take a look at the Queryable.Join syntax below:

    Join<TOuter,TInner,TKey,TResult>(IQueryable<TOuter>, IEnumerable<TInner>, Expression<Func<TOuter,TKey>>, Expression<Func<TInner,TKey>>, Expression<Func<TOuter,TInner,TResult>>)

`TOuter` is the type of elements of the first sequence or in other words, a table which is going to call this method to join.

`TInner` is the type of elements of the second sequence or in other words, a target table which is going to be passed in in Join method.

Both `Expression<Func<TOuter, TKey>>` and `Expression<Func<TInner, TKey>>` serve the same purpose which is to determine which field in sequence as join key. For instance, I know that both tables Products and ProductCategory can be joined using ProductId.

Finally, `Expression<Func<TOuter,TInner,TResult>>` is where Join method will pass in results into this function to let me create result elements.

So my LINQ statement at this stage as below:

    ProductPrices
    .Join(ProductCategories, pp => pp.ProductId, pc => pc.ProductId, (pp, pc) => new { pp, pc })
    .Join(Products, pp_pc => pp_pc.pp.ProductId, p => p.Id, (pp_pc, p) => new { pp_pc.pp, pp_pc.pc, p })
    .Where(pp_pc_p => new int[] {76, 77}.Contains(pp_pc_p.pc.CategoryId))
    .Where(pp_pc_p => pp_pc_p.p.Enabled == true)
    .Where(pp_p => pp_p.pp.Enabled == true)

Now, if I copy and paste the above in LINQPad, and select to view in SQL.

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286873965/YDhUv-0JI.png)](https://codecultivation.files.wordpress.com/2020/11/linqpad-view-in-sql.png)

I have the SQL auto-generated as below:

    -- Region Parameters
    DECLARE @p0 Int = 76
    DECLARE @p1 Int = 77
    -- EndRegion
    SELECT [t0].[Id], [t0].[ProductId], [t0].[Price], [t0].[Size], [t0].[Stock], ...
    FROM [ProductPrices] AS [t0]
    INNER JOIN [ProductCategory] AS [t1] ON [t0].[ProductId] = [t1].[ProductId]
    INNER JOIN [Products] AS [t2] ON [t0].[ProductId] = [t2].[Id]
    WHERE ([t0].[Enabled] = 1) AND ([t2].[Enabled] = 1) AND ([t1].[CategoryId] IN (@p0, @p1))

### 2nd stage: SQL GROUP BY

My second problem that I needed to solve was to find a way to group those records by ProductId. In LINQ, I chose Queryable.GroupBy to do this task. Take a look at the syntax below:

    GroupBy<TSource,TKey,TElement,TResult>(IQueryable<TSource>, Expression<Func<TSource,TKey>>, Expression<Func<TSource,TElement>>, Expression<Func<TKey,IEnumerable<TElement>,TResult>>)

`TSource` is a type of elements of source or in other words, a sequence result which is going to call this method to be grouped by.

`Expression<Func<TSource,TKey>>` is a function to extract the key of each element or in other words, to indicate which field in a table to group by. Essentially, this is the main function in GroupBy.

`Expression<Func<TSource,TElement>>` is a function to map each source element to an element in an `IGrouping<TKey, TElement>` or in other words, I wanted to group tables by ProductId (`TKey`) and the result of the grouping in SQL select is Price (`TElement`) which is going to be used for minimum and maximum value for next selector.

`Expression<Func<TKey,IEnumerable<TElement>,TResult>>` is a function to create a result value from each group or in other words, for each product which has 1 or more prices in `IGrouping<TKey, TElement>` format, I would be able to find maximum and minimum price using `Min()` and `Max()`.

So my LINQ statement at this stage as below:

    ProductPrices
    .Join(ProductCategories, pp => pp.ProductId, pc => pc.ProductId, (pp, pc) => new { pp, pc })
    .Join(Products, pp_pc => pp_pc.pp.ProductId, p => p.Id, (pp_pc, p) => new { pp_pc.pp, pp_pc.pc, p })
    .Where(pp_pc_p => new int[] {76, 77}.Contains(pp_pc_p.pc.CategoryId))
    .Where(pp_pc_p => pp_pc_p.p.Enabled == true)
    .Where(pp_pc_p => pp_pc_p.pp.Enabled == true)
    .GroupBy(pp_pc_p => pp_pc_p.pc.ProductId,
    		 pp_pc_p => pp_pc_p.pp.Price,
    		 (productId, prices) => new { productId, max = prices.Max(), min = prices.Min() })

### 3rd stage: SQL MIN() and MAX()

If I execute the LINQ statements above in LINQPad, it throws me an error saying that `NotSupportedException: Parameterless aggregate operator 'Max' is not supported over projections.`

I was wondering why was this happening? According to [Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/api/system.linq.queryable.max?view=netframework-4.7.1#System_Linq_Queryable_Max__1_System_Linq_IQueryable___0__), `Max<TSource>(IQueryable<TSource>)` should return the maximum value in a sequence of Decimal values. However, I realized that each price was part of projected values. As the exception had already told me that prices were really just some projected values, the correct method should be `Max<TSource,TResult>(IQueryable<TSource>, Expression<Func<TSource,TResult>>)` which invokes a transform function `Func<TSource,TResult>` to find me the maximum value.

So with some slight changes, my LINQ statement is as below:

    ProductPrices
    .Join(ProductCategories, pp => pp.ProductId, pc => pc.ProductId, (pp, pc) => new { pp, pc })
    .Join(Products, pp_pc => pp_pc.pp.ProductId, p => p.Id, (pp_pc, p) => new { pp_pc.pp, pp_pc.pc, p })
    .Where(pp_pc_p => new int[] {76, 77}.Contains(pp_pc_p.pc.CategoryId))
    .Where(pp_pc_p => pp_pc_p.p.Enabled == true)
    .Where(pp_pc_p => pp_pc_p.pp.Enabled == true)
    .GroupBy(pp_pc_p => pp_pc_p.pc.ProductId,
             pp_pc_p => pp_pc_p.pp.Price,
             (productId, prices) => new { productId, max = prices.Max(p => p), min = prices.Min(p => p) })

Now, if I copy and paste above in LINQPad, and select to view in SQL, I have the SQL auto-generated as below which is the same SQL written by me:

    -- Region Parameters
    DECLARE @p0 Int = 76
    DECLARE @p1 Int = 77
    -- EndRegion
    SELECT MAX([t0].[Price]) AS [max], MIN([t0].[Price]) AS [min], [t1].[ProductId] AS [productId]
    FROM [ProductPrices] AS [t0]
    INNER JOIN [ProductCategory] AS [t1] ON [t0].[ProductId] = [t1].[ProductId]
    INNER JOIN [Products] AS [t2] ON [t0].[ProductId] = [t2].[Id]
    WHERE ([t0].[Enabled] = 1) AND ([t2].[Enabled] = 1) AND ([t1].[CategoryId] IN (@p0, @p1))
    GROUP BY [t1].[ProductId]

Stay tuned.