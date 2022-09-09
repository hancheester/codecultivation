## How to mock ASP.NET Core Identity?

I was having a bit of issue when I tried to test my class which needs `UserManager<ApplicationUser>` object and `RoleManager<IdentityRole>` object in constructor method. Both classes are from new membership system called ASP.NET Core Identity. You can learn more from [Introduction to Identity on ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity)

When I first googled for my problem, some people use Fake class to test their method which results with more classes in unit test project to maintain. I rather try to avoid that.

The solution that I have used [Moq](https://github.com/moq/moq4) for .NET mocking framework. Without creating fake class, I can instantiate a Mock object by passing in mocked arguments into the intended type contructor method.

Please see example below:
```csharp
    var mockUserManager = new Mock>(
                         new Mock>().Object,
                         new Mock>().Object,
                         new Mock>().Object,
                         new IUserValidator[0],
                         new IPasswordValidator[0],
                         new Mock().Object,
                         new Mock().Object,
                         new Mock().Object,
                         new Mock>>().Object);
     return mockUserManager.Object;

    var mockRoleManager = new Mock>(
                         new Mock>().Object,
                         new IRoleValidator[0],
                         new Mock().Object,
                         new Mock().Object,
                         new Mock>>().Object);
     return mockRoleManager.Object;
```

As observed above, ASP.NET Identity’s `UserManager<TUser>` takes quite a number of parameters if you look at the class definition. And there is no parameterless constructor in `UserManager<TUser>`, otherwise, you will have an error message saying _Could not find a parameterless constructor._ like I did when trying to instantiate this class without constructor parameters. By the way, I was using [Castle Windsor](http://www.castleproject.org/projects/windsor/) for Inversion of Control (IOC) container for my unit testing project for injecting required arguments into contructor methods.

So by using Mocked object from Moq, I would be able to use `Mock<UserManager<TUser>().Setup(x => ...).Return(...)` in all my test methods.

Did you find this article useful? If you have any feedback or questions, please let me know in the comments below.

Thank you for reading and happy coding!