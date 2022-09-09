## Peculiar case of “Unable to connect to web server ‘IIS Express'”

There was one particular ASP.NET Core project that I have left it for quite a long time. Therefore, I would need to refresh my memory in order to understand how it works by first running it in debugging mode in Visual Studio 2017. To my suprise, as soon as I pressed F5, there was a popup error as below:

![Error](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286816747/9u6QEc17s.png)

Of course, like everyone else, I would first check my **launchSettings.json**

```json
    {
      "iisSettings": {
        "windowsAuthentication": false,
        "anonymousAuthentication": true,
        "iisExpress": {
          "applicationUrl": "http://localhost:51841",
          "sslPort": 44353
        }
      },
      "profiles": {
        "IIS Express": {
          "commandName": "IISExpress",
          "launchBrowser": true,
          "environmentVariables": {
            "ASPNETCORE_ENVIRONMENT": "Development"
          }
        },
        "WebApplication5": {
          "commandName": "Project",
          "launchBrowser": true,
          "environmentVariables": {
            "ASPNETCORE_ENVIRONMENT": "Development"
          },
          "applicationUrl": "https://localhost:5001;http://localhost:5000"
        }
      }
    }    
```

It just looked fine for me. So, what could go wrong?

### Method 1 (Remove .vs folder)

After searching in Google, quite a number of people suggesting to remove **.vs** folder from the project and let Visual Studio to auto generate the .vs folder (especially the **applicationhost.config** in **config** folder and it should work after that. It didn’t work for me.

![Visual Studio Folder](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286817862/djQZPpVVZb.png)

In case you ask where to find **.vs** folder, simply make sure to show hidden folder or files in _Folder Options_ menu.

![Show Hidden](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286819378/oMgOTvt0i.png)

### Method 2 (Run as administrator)

The second way that should work is to run Visual Studio as administrator. Simply right-click on Visual Studio shortcut and choose _Run as administrator_

![Runas](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286820924/evtrRjljm.png)

However, I was not happy with both methods above. I think there must be some other hidden reason behind this.

### Method 3 (Diagnostic approach)

First, I need to find out what if I can have some sort of log file or error message. Unable to connect to web server ‘IIS Express’ doesn’t help. Visual Studio in this case didn’t really help me much to give more information about this issue.

After doing some digging around on Internet, I came across a management console built for IIS Express called [Jexus Manager](https://www.jexusmanager.com/) that apparently provides some good diagnostic reports about your website.

![Jexus Manager](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286822203/qn43fueSH.png)

To begin, choose the option _Connect to a Server_ like below.

![Connect Server](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286823428/BAwDDxjBo.png)

In _Choose Server Type_ screen, make sure to choose _Visual Studio IIS Express Configuration File_ like below.

![Server Type](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286824775/EzVFfbW9w.png)

And then, choose the targeted Visual Studio solution file (.sln) like below.

![VS Solution](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286826358/jU4PRNfdv.png)

Make sure option _From Visual Studio .vs folder_ is selected.

![Configure src](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286828340/-W4aX9sqt.png)

Simply click _Finish_ button to complete the process.

![Connection Name](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286829971/Z0RoCliR9.png)

Finally, I would see a new connection was created in _Connections_ pane.

![Connections](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286831407/yXx72wntk.png)

Normally, we could ignore the first _WebSite1_ as it is just a default website. Select the next website and I would see some options in _Actions_ pane.

![Actions](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286832605/mxvTcqDdZ.png)

At first, all those Diagnostics reports didn’t really help me much here. However, when I chose to _Start_ the website (under _Manage Website_ section), something interesting popped up.

![Message](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286834102/k6MScjjKg.png)

Now, this was something interesting for me. I had to admit that I did not understand all those special numbers and process names, but fortunately the final two lines helped me to go further. Apparently, I could execute **iisexpress.exe** in Command Prompt to investigate further! At this point of time, I was happy and nervous at the same time as it proved my lack of knowledge on IIS Express but was happy to learn something new too.

Let’s open Command Prompy and navigate to _C:Program FilesIIS Express_. To start IIS Express, simply execute the command line below.

```batch
    iisexpress.exe /config:"c:[your project path].vsconfigapplicationhost.config" /site:WebApplication5 /trace:error    
```

The output in Command Prompt was as below.

![IIS Express Error](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286835635/9qqJXuSYj.png)

Noticed that IIS Express complained it was failed to register URL “[http://localhost:51841/](http://localhost:51841/)” as access was denied. Subsequently, it was failed to register “[https://localhost:44353/](https://localhost:44353/)” as it cannot create a file when that file already exists. I knew that the first error message was a clue but I just couldn’t understand why.

Also, I noticed that there was a notification popup from IIS Express too.

![IIS Notify](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286836839/IVZ_ElaUP.png)

When clicked, I could see _IIS Express Notifications_ screen.

![IS Express Notifications](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286838412/8PFTi8X6t.png)

The message in notification further clarified that I needed administrative privileges to bind to hostname or port and article [Handling URL binding failures in IIS Express](https://docs.microsoft.com/en-us/iis/extensions/using-iis-express/handling-url-binding-failures-in-iis-express) helped me alot in understanding how IIS Express handling URL binding. In this article, it clearly states that you need administrative privileges to configure HTTP.sys (an OS component that handles HTTP and SSL traffic for both IIS and IIS Express) for the following scenarios,

*   Using reserved ports such as 80 or 443
*   Serving external traffic
*   Using SSL

To configure HTTP.sys, I needed to use [netsh.exe](http://) utility. After going thru the article, I noticed that it has alot to do with urlacl switch. urlacl stands for URL Access Control List which creates reservation for particular group of users. You can read more about it [here](https://serverfault.com/questions/822207/how-does-url-reservation-actually-work-in-windows-particularly-the-acls).

At this point, I was interested to find out the list of reservation entries in my computer by running `netsh http show urlacl` and I saw something like this below.

![urlacl](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286839767/XtcdWBDVZ.png)

To my surprise, I managed to find one entry with **Reserved URL: [http://\*.51841/](http://*.51841/)**  
How on earth could that happen!? I presume only administrator could do that and I don’t think Visual Studio will be able to add the entry by itself. After some intense investigations, I laughed at myself as the culprit was me myself all this time. Why? It was a leftover from my previous article [How to access your localhost website from external LAN?](https://codecultivation.com/how-to-access-your-localhost-website-from-external/) I should have done the cleanup after the article. I created a problem from my own solution!

At the end, I simply removed the entry from the reservation list and IIS Express started to work fine again. The command line is as below.

```batch
    netsh http delete urlacl url=http://*:51841/
```

### Conclusion

In the end, I was having alot fun while investigating the problem and I was glad to manage to solve the issue by diagnostic approach. First, quick (temporary) solution is not a good solution, I should always aim for permanent solution. Next, it is always important to keep learning while solving problems as it would further improve my understanding. Last but not least, cleanup process is super important if you don’t want to have any (not so pleasant) surprises and it’s always a best practise to restore to original state after you have done the work.

Please let me know if you have any questions, comments, or concerns below.