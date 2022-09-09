## Using Debug Diagnostics Tool To Analyze Hang Issue

Just arrived at the office, without breakfast, I had complaint from boss that our website was very slow. Looking at the website, it was very slow indeed, only after a good few minutes, the website only started to load some texts barely while you still can see the rolling icon on web browser’s tab.

After looking at application log files, I still couldn’t find any issue. In fact, the last recorded data was 2 days ago. Tried to connect to database, it worked. Did a couple of SQL queries to see if I could retrieve data, it worked too. Next, let me tried to check worker process for application pool. It would take a snapshot of current requests associated with the application pool. To my surprise, some of the requests have taken more than 2 minutes and still running.

![Worker processes](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286863265/nhDlIELFi.png)

It left me no choice but to restart the application pool and website. It still having the same issue. This time, I created a dump file (.DMP file) and used an IIS Debug Diagnostics Tool which can be [downloaded here for free](https://www.microsoft.com/en-us/download/details.aspx?id=49924) to analyze hang issue. After a while, a HTML report popped up in Internet Explorer.

![Debug Diagnostic Tool Report](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286864747/5fFgtc4zv.png)

It gave me a nice easy to read report, and it seemed that there was thread not releasing a .NET lock. By examining .NET call stack, it was stuck trying to make a call to our web services.

Let’s go to Web Service server.

After looking at Task Manager -> Details tab, w3wp.exe for web service was having very high CPU usage! Nearly 50% of CPU was detected for this web worker. Created a dump file and analyzed using the same diagnostics tool. The report showed that it was taking a long time to retrieve an object (BrandFeaturedItem, it’s a hint!) from cache. At first, I thought it was due to our own cache engine which is running as a Window Service. The purpose of having cache engine is to enable us to have distributed cache. Restarted the service and waited to see what happened next. Apparently, it seemed to help a bit with speed, but shortly after that, it showed high CPU usage again.

In my mind, I thought perhaps this time I could use the diagnostics tool again to analyze .NET memory issue. In the report, it showed me the most memory consuming .NET object types, very nice. Out of those object types, I saw BrandFeaturedItem again! There were 544309 objects! It was not normal as I would expect to have no more than few hundreds. Without hesitation, I quickly ran a SQL query to retrieve rows from BrandFeaturedItem table in database.

![SQL query to retrieve rows from BrandFeaturedItem table in database](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286865988/B6qqQ1bCe.png)

More than 20 thousands rows in this table. Let me see if there were any duplicate rows by running SQL query below.

```sql
    SELECT BrandId, ProductId, FeaturedItemType, COUNT(*) FROM BrandFeaturedItem
    GROUP BY BrandId, ProductId, FeaturedItemType
    HAVING COUNT(*) > 1
```

There were a lot of duplicate rows indeed. Somewhere in application was creating duplicate rows which I would sort it out later after fixing this urgent issue. Now, I needed to remove all duplicate rows leaving one copy of each. Using [Common Table Expressions](https://technet.microsoft.com/en-us/library/ms190766(v=sql.105).aspx) to create a temporary result set, I would be able to get all duplicate rows leaving only one copy of each.

```sql
    WTIH BrandFeaturedItemCTE (BrandId, ProductId, FeaturedItemType)
    AS 
    (
    	SELECT BrandId, ProductId, FeaturedItemType FROM BrandFeaturedItem
    	GROUP BY BrandId, ProductId, FeaturedItemType
    	HAVING COUNT(*) > 1
    )
    
    DELETE FROM BrandFeaturedItem
    WHERE Id IN (
    	SELECT Id FROM BrandFeaturedItemCTE cte
    	LEFT JOIN BrandFeaturedItem bfi ON cte.BrandId = bfi.BrandId 
    		AND cte.ProductId = bfi.ProductId 
    		AND cte.FeaturedItemType = bfi.FeaturedItemType
    )
    AND Id NOT IN (
    	SELECT MAX(Id) FROM BrandFeaturedItem
    	GROUP BY BrandId, ProductId, FeaturedItemType
    	HAVING COUNT(*) > 1
    )
```

Now, all duplicate rows were gone. By fixing the urgent speed issue, the website now loaded quickly as before. Relieved, now it’s time for me to look at the application which could be another post itself to explain how I finally fixed a bug.

Thanks to IIS Debug Diagnostics Tool, they made my life so much easier to search for hang issue. Unlike before, I would need to open Command Prompt to get me a dump file using [ADPlus](https://support.microsoft.com/en-us/help/286350/how-to-use-adplus-vbs-to-troubleshoot-hangs-and-crashes) and run [WindDbg](https://developer.microsoft.com/en-us/windows/hardware/download-windbg) to evaluate all the threads and stacks.

The process of debugging hang issue was roughly like below.

1.  Retrieve process id from IIS
    *   iisapp
2.  Create crash dump using process id
    *   Goto c:program filesdebugging tools for windows (x86)
    *   adplus -hang -p \[process id\] -o c:applicationscrashdump
3.  File -> Symbol File Path ->
    *   SRV_c:symbols_[http://msdl.microsoft.com/download/symbols](http://msdl.microsoft.com/download/symbols)
4.  Run command in windbg
    *   .load c:windowsmicrosoft.netframeworkv4.0.30319sos.dll
5.  Evaluate all threads
    *   ~\* e !clrstack
6.  Finding which thread is holding the log, look for first number at “Info”
    *   !syncblk
7.  Goto the thread \[number\]
    *   ~\[number\]s
8.  Under the thread number context 0:\[number\]>
    *   !clrstack

Now, I just need to use IIS Debug Diagnostics Tool and it will just get me a nice easy-to-read report which will show me where to look for. Timewise, it saves me lot of efforts. However, knowing how to use ADPlus and WinDbg is a plus, just in case the tool is not available to use.

Did you find this article useful? If you have any feedback or questions, please let me know in the comments below.

Thank you for reading and happy debugging!