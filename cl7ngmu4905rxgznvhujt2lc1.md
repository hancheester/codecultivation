## How to use ASP.NET Identity with ASP.NET Web Service and ASP.NET MVC?

[ASP.NET Identity](https://www.asp.net/identity) is much more powerful than legacy ASP.NET Membership which includes the support of external authentication from 3rd party providers. In one of my projects, I have an ASP.NET MVC app communicating with another ASP.NET Web Service app which contains all core business logic and data access. The MVC application essentially is just another Presentation Layer which handles simple rule processing. While I was trying to [migrate to ASP.NET Identity](https://docs.microsoft.com/en-us/aspnet/identity/overview/migrations/migrating-an-existing-website-from-sql-membership-to-aspnet-identity), I was stuck on trying to decouple database access from ASP.NET Identity, specifically the `SignInManager` class in the MVC application.

Fortunately, there is a way to do it, thanks to Microsoft open source project which allows me to study the relevant codes and I managed to decouple them. To start with, I created a sample project which can be downloaded from my github for your reference. _One side note, as I do not need features like TFA (Two Factor Authentication), email confirmation and so on, this sample project only focus on registration, signin and signout functionalities. In future, I might improve the project further to include other features._

There are two main classes that we would often use them in our project:

*   `SignInManager` – provides APIs for user sign in
*   `UserManager` – provides APIs for managing user in persistence store (e.g. database)

Naturally, `UserManager` which needs access to database via `IUserStore` class should be in my ASP.NET Web Service app. However, to use `SignInManager` class in ASP.NET MVC app, you will need `UserManager` during construction. Obviously, we have a tight coupling issue here.

My approach to get around it is not to use `SignInManager` class at all by constructing our own implementation to authenticate users. First, just download the source code for Microsoft ASP.NET Identity from the link below:  
[https://archive.codeplex.com/?p=aspnetidentity](https://archive.codeplex.com/?p=aspnetidentity)

In `SignInManager.cs`, there is one method that we need to look at:

```csharp
    public virtual async Task PasswordSignInAsync(string userName, string password, bool isPersistent, bool shouldLockout)
    {
        if (UserManager == null)
        {
            return SignInStatus.Failure;
        }
        var user = await UserManager.FindByNameAsync(userName).WithCurrentCulture();
        if (user == null)
        {
            return SignInStatus.Failure;
        }
        if (await UserManager.IsLockedOutAsync(user.Id).WithCurrentCulture())
        {
            return SignInStatus.LockedOut;
        }
        if (await UserManager.CheckPasswordAsync(user, password).WithCurrentCulture())
        {
            await UserManager.ResetAccessFailedCountAsync(user.Id).WithCurrentCulture();
            return await SignInOrTwoFactor(user, isPersistent).WithCurrentCulture();
        }
        if (shouldLockout)
        {
            // If lockout is requested, increment access failed count which might lock out the user
            await UserManager.AccessFailedAsync(user.Id).WithCurrentCulture();
            if (await UserManager.IsLockedOutAsync(user.Id).WithCurrentCulture())
            {
                return SignInStatus.LockedOut;
            }
        }
        return SignInStatus.Failure;
    }
```

As you can see above, this method is using `UserManager` object to some validations and eventually, if everything goes well, it will sign in user via `SignInOrTwoFactor`.

```csharp
    private async Task SignInOrTwoFactor(TUser user, bool isPersistent)
    {
        var id = Convert.ToString(user.Id);
        if (await UserManager.GetTwoFactorEnabledAsync(user.Id).WithCurrentCulture()
            && (await UserManager.GetValidTwoFactorProvidersAsync(user.Id).WithCurrentCulture()).Count > 0
            && !await AuthenticationManager.TwoFactorBrowserRememberedAsync(id).WithCurrentCulture())
        {
            var identity = new ClaimsIdentity(DefaultAuthenticationTypes.TwoFactorCookie);
            identity.AddClaim(new Claim(ClaimTypes.NameIdentifier, id));
            AuthenticationManager.SignIn(identity);
            return SignInStatus.RequiresVerification;
        }
        await SignInAsync(user, isPersistent, false).WithCurrentCulture();
        return SignInStatus.Success;
    }    
```

`SignInOrTwoFactor` is just a simple method to sign out and sign in again with `AuthenticationManager` which comes from OWIN context itself. You can learn more about OWIN [here](http://owin.org/).

First, for both projects I use a DI container called [Autofac](https://autofac.org/). You can browse [here](http://autofaccn.readthedocs.io/en/latest/integration/mvc.html) and [here](http://autofaccn.readthedocs.io/en/latest/integration/wcf.html) to learn to use Autofac.

In Web Service project, I have 3 following classes:  
`AppDbContext` which derived from `IdentityDbContext<IdentityUser>`

```csharp
    public class AppDbContext : IdentityDbContext
    {
        public AppDbContext()
            : base ("name=AppDbContext")
        {
    
        }
    
        public AppDbContext(string connectionString)
            : base(connectionString)
        {
    
        }
    
        ...
    }    
```

`ApplicationUserManager` which derived from `UserManager<IdentityUser>`
```csharp
    public class ApplicationUserManager : UserManager
    {
        public ApplicationUserManager(ApplicationUserStore store)
            : base(store)
        {
            // Configure validation logic for usernames
            this.UserValidator = new UserValidator(this)
            {
                AllowOnlyAlphanumericUserNames = true,
                RequireUniqueEmail = true
            };
    
            // Configure validation logic for passwords
            this.PasswordValidator = new PasswordValidator
            {
                RequiredLength = 8,
            };
    
            // Configure user lockout defaults
            this.UserLockoutEnabledByDefault = true;
            this.DefaultAccountLockoutTimeSpan = TimeSpan.FromMinutes(5);
            this.MaxFailedAccessAttemptsBeforeLockout = 5;
        }
    }    
```

`ApplicationUserStore`
```csharp
    public class ApplicationUserStore : UserStore
    {
        public ApplicationUserStore(AppDbContext appDbContext)
            : base(appDbContext)
        {
    
        }
    }    
```

By refering to source code of `SignInManager` in `PasswordSignInAsync` method earlier, we have a method defined as below:
```csharp
    public IdentityLoginResult ValidateIdentityUser(string username, string password, bool shouldLockout)
    {
        var user = _userManager.FindByName(username);
        if (user == null)
            return new IdentityLoginResult { CustomerLoginResults = CustomerLoginResults.MemberNotExists };
    
        if (_userManager.IsLockedOut(user.Id))
            return new IdentityLoginResult { CustomerLoginResults = CustomerLoginResults.IsLockedOut };
    
        var account = _accountRepository.Table.Where(x => x.Email.ToLower() == user.Email.ToLower());
        if (account == null)
            return new IdentityLoginResult { CustomerLoginResults = CustomerLoginResults.AccountNotExists };
    
        if (_userManager.CheckPassword(user, password))
        {
            _userManager.ResetAccessFailedCount(user.Id);
    
            var userIdentity = _userManager.CreateIdentity(user, DefaultAuthenticationTypes.ApplicationCookie);
    
            return new IdentityLoginResult
            {
                ClaimsIdentity = userIdentity,
                CustomerLoginResults = CustomerLoginResults.Successful,
            };
        }
    
        if (shouldLockout)
        {
            _userManager.AccessFailed(user.Id);
            if (_userManager.IsLockedOut(user.Id))
                return new IdentityLoginResult { CustomerLoginResults = CustomerLoginResults.IsLockedOut };
        }
    
        return new IdentityLoginResult { CustomerLoginResults = CustomerLoginResults.WrongPassword };
    }    
```

Note that the workflow is very similar to `SignInManager.PasswordSignInAsync` method.

In MVC project, for `Startup` class
```csharp
    public partial class Startup
    {
        // For more information on configuring authentication, please visit https://go.microsoft.com/fwlink/?LinkId=301864
        public void ConfigureAuth(IAppBuilder app, IContainer container)
        {
            // Enable the application to use a cookie to store information for the signed in user
            // and to use a cookie to temporarily store information about a user logging in with a third party login provider
            // Configure the sign in cookie
            app.UseCookieAuthentication(new CookieAuthenticationOptions
            {
                AuthenticationType = DefaultAuthenticationTypes.ApplicationCookie,
                LoginPath = new PathString("/Account/Login"),
                Provider = new CookieAuthenticationProvider
                {
                    // Enables the application to validate the security stamp when the user logs in.
                    // This is a security feature which is used when you change a password or add an external login to your account.  
                    OnValidateIdentity = ApplicationSecurityStampValidator.OnValidateIdentity(
                        validateInterval: TimeSpan.FromMinutes(30),
                        regenerateIdentity: (manager, userId) => Task.FromResult(manager.CreateIdentity(userId)),
                        manager: DependencyResolver.Current.GetService())
                }
            });            
            app.UseExternalSignInCookie(DefaultAuthenticationTypes.ExternalCookie);
    
            // Enables the application to temporarily store user information when they are verifying the second factor in the two-factor authentication process.
            app.UseTwoFactorSignInCookie(DefaultAuthenticationTypes.TwoFactorCookie, TimeSpan.FromMinutes(5));
    
            // Enables the application to remember the second login verification factor such as phone or email.
            // Once you check this option, your second step of verification during the login process will be remembered on the device where you logged in from.
            // This is similar to the RememberMe option when you log in.
            app.UseTwoFactorRememberBrowserCookie(DefaultAuthenticationTypes.TwoFactorRememberBrowserCookie);
    
            // Uncomment the following lines to enable logging in with third party login providers
            //app.UseMicrosoftAccountAuthentication(
            //    clientId: "",
            //    clientSecret: "");
    
            //app.UseTwitterAuthentication(
            //   consumerKey: "",
            //   consumerSecret: "");
    
            //app.UseFacebookAuthentication(
            //   appId: "",
            //   appSecret: "");
    
            //app.UseGoogleAuthentication(new GoogleOAuth2AuthenticationOptions()
            //{
            //    ClientId = "",
            //    ClientSecret = ""
            //});
        }
    }    
```

Notice that instead of using default `SecurityStampValidator`, I created a custom security stamp validator which uses a DependencyResolver to resolve a service in order to call our web service app earlier. Even though I didn’t implment changing password or external login, its main purpose is to show that we can actually call web service while validating user identity.

There are 2 interfaces:

1.  `IAuthService` which contains common methods such as SignIn and SignOut which could be used by Form Authentication too.
2.  `IIdentityAuthService` which extends from `IAuthService` to support ASP.NET Identity features.

With all that classes in place, we have a working sample project which enables user to signin and signout via ASP.NET Web Service without using `SignInManager` at all.

For project source code, please download [here](https://github.com/hancheester/ntier-identity-without-signinmanager).  
[https://github.com/hancheester/ntier-identity-without-signinmanager](https://github.com/hancheester/ntier-identity-without-signinmanager)

Please let me know if you have any questions, comments, or concerns below.