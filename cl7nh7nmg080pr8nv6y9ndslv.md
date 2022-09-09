## My journey of using Join, GroupJoin and GroupBy – Part 2

As I continued doing my migration process to convert my existing project from ASP.NET Web Form to ASP.NET MVC using Entity Framework from my previous post, I had abit of issue with SQL LEFT JOIN. Basically, the SQL keywords return all records from _left table_ and matched records from _right table_. `NULL` is returned from right side if there is no match.

I had 3 tables that I needed to join them together. The _left table_ is called Line Item. The rest are _right tables_ which are ProductPrices and Colours. Take a look at the SQL statement below:

    SELECT L.ProductId, L.Name, PP.Size, C.Value, PP.Stock, SUM(L.PendingQuantity), COUNT(DISTINCT L.OrderId) 
     FROM LineItem L WITH(NOLOCK)
         LEFT JOIN ProductPrices PP WITH(NOLOCK) ON L.ProductPriceId = PP.Id
         LEFT JOIN Colours C WITH(NOLOCK) ON PP.ColourId = C.Id
     WHERE L.StatusCode IN ('AS', 'SW')
     GROUP BY L.ProductId, L.Name, PP.Size, C.Value, PP.Stock
     ORDER BY MIN(L.CreatedOnUtc)

From [previous post](https://codecultivation.com/my-journey-of-using-join-groupjoin-and-groupby-part-1/) which mentioned about (INNER) JOIN to return records that have matching values in all tables, only LineItem table would need to return all records. The other 2 tables will return `NULL` if there is no matching value. In LINQ, the operation that can perform the task is called GroupJoin. Take a look at Queryable.GroupJoin syntax below:

    GroupJoin(IQueryable, IEnumerable,
     Expression>, Expression>,
     Expression,TResult>>)

I noticed that GroupJoin is very similar to Join in LINQ. Let’s take a look again at Queryable.Join syntax below:

    Join<TOuter,TInner,TKey,TResult>(IQueryable<TOuter>, IEnumerable<TInner>, Expression<Func<TOuter,TKey>>, Expression<Func<TInner,TKey>>, Expression<Func<TOuter,TInner,TResult>>)

Both have same type parameters, and the first 4 parameters are identical too. The key difference between these 2 methods is in the last parameter.

For Join, the last parameter is:  
`Expression<Func<TOuter,TInner,TResult>>`

For GroupJoin, the last parameter is:  
`Expression<Func<TOuter,IEnumerable<TInner>,TResult>>`

Both last parameters act as `resultSelector` meaning LINQ will pass results into a function to create result element. In GroupJoin, from `IEnumerable<TInner>` I know that the collection of matching elements could be zero or more which behaves similar to SQL LEFT JOIN.

    LineItems
    .GroupJoin(ProductPrices,
        l => l.ProductPriceId,
        pp => pp.Id, 
        (l, pp) => new { l, pp = pp.DefaultIfEmpty() })

Noticed that I used `DefaultIfEmpty()` to indicate that if sequence of product price is empty then return type parameter default value, in this case default value for class type is `null`. Now, if I left join another table again, like below:

    LineItems
    .GroupJoin(ProductPrices, 
        l => l.ProductPriceId, 
        pp => pp.Id, 
        (l, pp) => new { l, pp = pp.DefaultIfEmpty() })
    .GroupJoin(Colours, 
        x => x.pp.ColourId, 
        c => c.Id, 
        (x, c) => new { l_p = x, c = c.DefaultIfEmpty() })

LINQPad throwed me an error below:  
`CS1061 'IEnumerable<ProductPrices>' does not contain a definition for 'ColourId' and no extension method 'ColourId' accepting a first argument of type 'IEnumerable<ProductPrices>' could be found (press F4 to add a using directive or assembly reference)`

Remember `IEnumerable<TInner>` in `Expression<Func<TOuter,IEnumerable<TInner>,TResult>>`? In this case, `IEnumerable<ProductPrices>` obviously doesn’t have any method called ColourId. The result elements from first GroupJoin is an anonymous type `new { LineItem l, IEnumerable<ProductPrice> pp }` which are the result of one-to-many projection function. In order to find the matching value from Colour table, I needed to flatten the collection into one single-dimensional sequence first. In this case, I would need to use SelectMany in LINQ. Take a look at the SelectMany syntax:

    SSelectMany<TSource,TResult>(IQueryable<TSource>, Expression<Func<TSource,IEnumerable<TResult>>>)

The result would give me a sequence of product prices with repeated line item assigned to them in one-to-one relationship.

In LINQPad,

    LineItems
    .GroupJoin(ProductPrices, 
        l => l.ProductPriceId, 
        pp => pp.Id, 
        (l, pp) => new { l, pp = pp.DefaultIfEmpty() })
    .SelectMany(l_pp => l_pp.pp.Select(x => new { l = l_pp.l, pp = x }))
    .GroupJoin(Colours, 
        x => x.pp.ColourId, 
        c => c.Id, 
        (x, c) => new { l_p = x, c = c.DefaultIfEmpty() })
    .SelectMany(l_pp_c => l_pp_c.c.Select(x => new { l = l_pp_c.l_p.l, pp = l_pp_c.l_p.pp, c = x }))
    .Select(x => new { x.l, x.pp, x.c })

When I chose to view in SQL in LINQPad,

    SELECT [t0].[Id], [t0].[OrderId], [t0].[ProductPriceId], [t0].[ProductId], [t0].[Name], ...
    FROM [LineItem] AS [t0]
    LEFT OUTER JOIN (
        SELECT 1 AS [test], [t1].[Id], [t1].[ProductId], [t1].[Price], ...
        FROM [ProductPrices] AS [t1]
        ) AS [t2] ON [t0].[ProductPriceId] = [t2].[Id]
    LEFT OUTER JOIN (
        SELECT 1 AS [test], [t3].[Id], [t3].[BrandId], [t3].[Value], [t3].[ColourFilename], [t3].[ThumbnailFilename]
        FROM [Colours] AS [t3]
        ) AS [t4] ON [t2].[ColourId] = ([t4].[Id])

With that one-dimensional sequence, I was ready to group the results. As I already learned to use GroupBy from previous post, take a look at LINQ statement below:

    LineItems
    .GroupJoin(ProductPrices, l => l.ProductPriceId, pp => pp.Id, (l, pp) => new { l, pp = pp.DefaultIfEmpty() })
    .SelectMany(l_pp => l_pp.pp.Select(x => new { l = l_pp.l, pp = x }))
    .GroupJoin(Colours, x => x.pp.ColourId, c => c.Id, (x, c) => new { l_p = x, c = c.DefaultIfEmpty() })
    .SelectMany(l_pp_c => l_pp_c.c.Select(x => new { l = l_pp_c.l_p.l, pp = l_pp_c.l_p.pp, c = x }))
    .Select(x => new { x.l, x.pp, x.c })
    .Where(x => new string[] { "AS", "SW" }.Contains(x.l.StatusCode))
    .GroupBy(x => new { x.l.ProductId, x.l.Name, x.pp.Size, x.c.Value, x.pp.Stock })

Together with SQL SUM(), COUNT() and MIN(), the final LINQ statement as below:

    LineItems
    .GroupJoin(ProductPrices, l => l.ProductPriceId, pp => pp.Id, (l, pp) => new { l, pp = pp.DefaultIfEmpty() })
    .SelectMany(l_pp => l_pp.pp.Select(x => new { l = l_pp.l, pp = x }))
    .GroupJoin(Colours, x => x.pp.ColourId, c => c.Id, (x, c) => new { l_p = x, c = c.DefaultIfEmpty() })
    .SelectMany(l_pp_c => l_pp_c.c.Select(x => new { l = l_pp_c.l_p.l, pp = l_pp_c.l_p.pp, c = x }))
    .Select(x => new { x.l, x.pp, x.c })
    .Where(x => new string[] { "AS", "SW" }.Contains(x.l.StatusCode))
    .GroupBy(x => new { x.l.ProductId, x.l.Name, x.pp.Size, x.c.Value, x.pp.Stock })
    .Select(x => new { Group = x.Key, OrderPlaced = x.Min(y => y.l.CreatedOnUtc), OrderCount = x.Select(z => z.l.OrderId).Distinct().Count(), Quantity = x.Sum(w => w.l.PendingQuantity) })
    .OrderBy(x => x.OrderPlaced)

### Conclusion

IMHO, LINQ does a good job in trying to bridge the gap between objects and data. Doing so, I can really just focus using C# without adding foreign language in my project. However, there is one setback I would like to point out. Though LINQ can help make my life easier but not the performance itself. If I convert the above LINQ statement into SQL in LINQPad, I got the following SQL statements:

    -- Region Parameters
    DECLARE @p0 NVarChar(1000) = 'AS'
    DECLARE @p1 NVarChar(1000) = 'SW'
    -- EndRegion
    SELECT [t9].[ProductId], [t9].[Name], [t9].[value3] AS [Size], ...
    FROM (
        SELECT [t4].[ProductId], [t4].[Name], [t4].[value3], [t4].[value22], [t4].[value32], [t4].[value], (
            SELECT COUNT(*)
            FROM (
                SELECT DISTINCT [t5].[OrderId]
                FROM [LineItem] AS [t5]
                LEFT OUTER JOIN [ProductPrices] AS [t6] ON [t5].[ProductPriceId] = [t6].[Id]
                LEFT OUTER JOIN [Colours] AS [t7] ON [t6].[ColourId] = ([t7].[Id])
                WHERE ([t4].[ProductId] = [t5].[ProductId]) AND ([t4].[Name] = [t5].[Name]) AND ((([t4].[value3] IS NULL) ... AND ((([t4].[value32] IS NULL) AND ([t6].[Stock] IS NULL)) OR (([t4].[value32] IS NOT NULL) AND ([t6].[Stock] IS NOT NULL) AND ([t4].[value32] = [t6].[Stock]))))) AND ([t5].[StatusCode] IN (@p0, @p1))
                ) AS [t8]
            ) AS [value2], [t4].[value2] AS [value23]
        FROM (
            SELECT MIN([t3].[CreatedOnUtc]) AS [value], SUM([t3].[PendingQuantity]) AS [value2], [t3].[ProductId], [t3].[Name], [t3].[value] AS [value3], [t3].[value2] AS [value22], [t3].[value3] AS [value32]
            FROM (
                SELECT [t0].[ProductId], [t0].[Name], [t1].[Size] AS [value], [t2].[Value] AS [value2], [t1].[Stock] AS [value3], [t0].[StatusCode], [t0].[CreatedOnUtc], [t0].[PendingQuantity]
                FROM [LineItem] AS [t0]
                LEFT OUTER JOIN [ProductPrices] AS [t1] ON [t0].[ProductPriceId] = [t1].[Id]
                LEFT OUTER JOIN [Colours] AS [t2] ON [t1].[ColourId] = ([t2].[Id])
                ) AS [t3]
            WHERE [t3].[StatusCode] IN (@p0, @p1)
            GROUP BY [t3].[ProductId], [t3].[Name], [t3].[value], [t3].[value2], [t3].[value3]
            ) AS [t4]
        ) AS [t9]
    ORDER BY [t9].[value]

I have got a lot of SELECTs compared to my original SQL statement! Will this have more or less the same performance? I doubt it. Sometimes, I would rather to use Stored Proc then LINQ, though I still have abit of SQL in my project but I would rather to prefer performance over zero SQL in my project.