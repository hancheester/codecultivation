## Generate absolute URLs in ASP.NET MVC

Sometimes it’s easier to create relative URL in development using `Url.Content("~/yourimage.png")`. It seems to be quicker to code this way. However, there are times that I need to generate absolute URL for public facing websites where SEO matters. Quickly google “why absolute url matters in seo”, there is quite a number of blogs or articles promoting the use of absolute URLs. For example, you can read an article from [Moz.com](https://moz.com/blog/relative-vs-absolute-urls-whiteboard-friday) which has a good explanation.

At first, I tried to search for solution to generate them in ASP.NET MVC. Some of them are not quite what I wished for. Thus, it gave me opportunity to create extension method for the `System.Web.Mvc.UrlHelper` class which will generate absolute URL relative to current HTTP content’s Request.Url property. Also, as we all know that Google prefers HTTPS, I also include an optional parameter ‘forceHttps’ if I would like it in HTTPS format. Look at the following class definition below:

```csharp
    public static class UrlHelperExtensions
     {
         public static string GenerateAbsoluteUrl(this UrlHelper helper, string path, bool forceHttps = false)
         {
             const string HTTPS = "https";
             var uri = helper.RequestContext.HttpContext.Request.Url;
             var scheme = forceHttps ? HTTPS : uri.Scheme;
             var host = uri.Host;
             var port = (forceHttps || uri.Scheme == HTTPS) ? string.Empty : (uri.Port == 80 ? string.Empty : ":" + uri.Port);
         return string.Format("{0}://{1}{2}/{3}", scheme, host, port, string.IsNullOrEmpty(path) ? string.Empty : path.TrimStart('/')); }
     }
```

Now, let’s say my website is _/_ and my relative URL for image is _/media/product/img.jpg_. In my ASP.NET MVC Razor view `.cshtml`, the usage is as below:

```html
    <img src="@Url.GenerateAbsoluteUrl("/media/product/img.jpg")" alt="Your Image">
```

The generated HTML would be:

```html
    <img src="https://codecultivation.com/media/product/img.jpg" alt="Your Image">
```

That’s all. Hope it can help someone looking for ways to build absolute URL.

Please let me know if you have any questions, comments, or concerns below.

Thanks.