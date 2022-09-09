## How to access your localhost website from external LAN?

I was trying to access my localhost ASP.NET Core MVC website from another computer which sits in different LAN. Thus, I tried to google around and it does show some good solutions, however, I would need to refer to multiple places to get the final solution. So, I thought I could just compile them in one single post, in case there are some people looking for one-stop solution. FYI, This solution is only for Windows 7 or Vista.

### 1\. Configure Windows Firewall

In my case, I will need to make sure that my Windows 7 firewall to open for that port number, for instance, 51841. You will need to add a rule in your Windows Firewall in Inbound Rules like below. Just select New Rule…

![Windows Firewall](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286844390/cbiJTIM9G.png)
Choose Port and click Next button

![Windows Firewall Port](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286845799/5tCmvhVqo.png)

Choose TCP and enter 51841 (or whatever port number you wish) in Specific local ports

![Windows Firewall Port Value](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286847129/zKBbFencV.png)

Choose Allow the connection

![Windows Firewall Action](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286848440/70-_3xTNw.png)

For profiles, it depends on your environment. In my case, I will check all three.

![Windows Firewall Profile](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286849820/7cSAXEW2N.png)

And finally, pick a name for this rule. In my case, I will put down my website name for easy reference later.

![Windows Firewall Name](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286851102/cHNOZ5_uE.png)

Once added, your new inbound rule will appear in the list.

![WIndows Firewall New Rule](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286852381/xypq6Fh5y.png)

### 2\. Enable remote connections on IIS Express

By default, remote connections are disabled on IIS Express. To enable it, simply execute the following command from an administrative prompt (Right click and choose Run as administrator):

```batch
    netsh http add urlacl url=http://*:51841/ user=everyone
```

In the command, replace 51841 with your own port number. `netsh` is a powerful tool to modify network configuration of computer. If you are interested, learn more from this [article](https://docs.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-contexts).

### 3\. Modify applicatonhost.config

However, you may come across the HTTP 503 Invalid hostname error. To resolve this, locate **$(solutionDir).vsconfigapplicationhost.config** file. (solutionDir) would your ASP.NET Core MVC project folder path. To be able to see _.vs_ folder, make sure you allow hidden folder to be visible.

Locate `<binding protocol="http" bindingInformation=":51841:localhost" />`  
and replace with `<binding protocol="http" bindingInformation="*:51841:*" />`

Once it’s done, it’s better for you to restart your Visual Studio.

And that’s it, you should be able to browse your website from anywhere, for example, **[http://%5Bmypcname%5D:51841/](http://%5Bmypcname%5D:51841/)**

Please let me know if you have any questions, comments, or concerns below.