---
title: "Authentication With Jwt"
date: 2022-07-19T22:19:01+02:00
draft: true
---

In this post we will look into authentication with JSON Web Tokens (JWTs) in .NET 6. If you don’t know what a JWT is, I strongly recommend you read this introduction first. It will make the rest of the post a lot easier to understand.

First things first, create a new C# project of the type ASP.NET Core Web API. Use the default settings, which means that you don’t select any Authentication type under Additional information. We will add everything ourselves.

Once the project is created, right click on the Controllers folder and select **Add > Controller**. Use the template API Controller – Empty. Name the new controller TokenController.cs.

## Creating the controller
Add the bottom of the TokenController, add these two private methods for validating the user, and generating the token. Notice also the record in the very bottom. If you haven’t heard of Records yet, you can read this tutorial.

```csharp 
private static bool IsValidUser(string username, string password, out UserAccount user)
{
    // This is where you would look the user up in the database.
    user = new UserAccount(Id: Guid.NewGuid(), 
                            Username: "Admin", 
                            Email: "admin@example.com");
 
    return true;
}
 
private static string GenerateToken(UserAccount user)
{
    var claims = new List<Claim>
    {
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new Claim(ClaimTypes.Name, user.Username),
        new Claim(ClaimTypes.Email, user.Email),
        new Claim(JwtRegisteredClaimNames.Exp, 
            new DateTimeOffset(DateTime.Now)
                   .ToUnixTimeSeconds().ToString())
};
 
    var header = new JwtHeader(
        new SigningCredentials(
            new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes("ThisKeyMustBeAtLeast16Characters")), 
                SecurityAlgorithms.HmacSha256));
 
    var payload = new JwtPayload(claims);
 
    var token = new JwtSecurityToken(header, payload);
 
    return new JwtSecurityTokenHandler().WriteToken(token);
}

private record UserAccount(Guid Id, string Username, string Email);
```

The `IsValidUser()` method is where you would normally look up the user in the database, but that is out of scope for this post. The reason I have chosen to return the user via the out parameter modifier is that we need it to generate the claims, in case of a successful login.

The generate token method will convert the user information to a list of claims and store it in a signed JWT. Notice that the secret key must be at least 16 characters. In the example I have put in directly in the code. Do NOT do this in a production scenario, as it would mean you push your key to your source control. Preferably use something like Azure Key Vault. But at very least a config file, which isn’t pushed to source control.

With these two methods in place, the post method can now be added, still to our controller.

```csharp 
[HttpPost]
public IActionResult GetToken(LoginRequest loginRequest)
{
    if (IsValidUser(loginRequest.Username, 
                    loginRequest.Password, 
                    out UserAccount user))
    {
        string token = GenerateToken(user);
        return Ok(token);
    }
    else
    {
        return BadRequest("Invalid username and/or password.");
    }
}
 
public record LoginRequest(string Username, string Password);
```

If the username and password is correct (it always is in our example) this will return a 200 OK response along with the token. Once again, notice the record in the bottom. The reason for this, rather than just have username and password as input for the GetToken method, is that it takes the username and password out of the request URL and put it into the request body.

Since this basically just returns a string, we could run this as it is, without any changes to the Program.cs class. Unfortunately, as it is a post request, it can be a bit tricky to test, unless you have something like Postman. Later on Swagger will be added, for easy testing.

One last thing which must be added to the TokenController is an end point which requires authorization. This will be used to test if our token is working as intended.

```csharp 
[HttpGet]
[Authorize]
public IActionResult VerifyToken()
{
    // This end point is just used to verify that we are authorized correctly.
    return Ok("Congratulations, you are authorized to see this.");
}
```

With this end point, the controller is now done. However, if you try to access the end point at this time, you will get a 401 Unauthorized response, even if you send the correct token. This is because we haven’t configured the authentication yet. In other words, your code doesn’t know what type of token it is, or what to do with it.

## Configure services
To let the program know that we are using JWT and how it’s signed, we need to add authentication to the services. This is done in the Program.cs file (in earlier versions it was in Startup.cs). Add the following code near the top of the file, just before `builder.Services.AddControllers()`.

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
}).AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters()
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("ThisKeyMustBeAtLeast16Characters")),
        ValidateIssuer = false,
        ValidateAudience = false,
        ValidateLifetime = true,
        ClockSkew = TimeSpan.FromMinutes(5)
    };
});
```

Notice that the key is the same as earlier. If this is not the case, your API won’t be able to verify the JWT. The last part we need to add is at the bottom of the Program.cs file, right after `app.UseHttpRedirection()`.

```csharp 
app.UseAuthentication();
app.UseAuthorization();
```

This enables Authentication and Authorization for our API. The order of the two is important, it won’t work the other way around. If you are having trouble remembering the correct order, just remember than you need to authenticate before the app can know if you are authorized.

## Configure Swagger (optional)
**If you are using another tool for testing your API, like Postman, you can skip this step.**
This last part has nothing to do with the authentication. But by configuring Swagger, it will be a lot easier to verify that the authentication is working as intended. If you used .NET 6 and the default settings when creating the project, Swagger should already be installed. Otherwise, you may have to install the various dependencies (which is not in the scope of this post).

Add the code snippet below to your Program.cs file right after `builder.Sercvices.AddControllers()`.

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc(
    "v1", new OpenApiInfo
    {
        Title = "Golf Score API",
        Version = "v1"
    });
 
    // This section allows submitting the token with your request in Swagger.
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "Bearer",
        BearerFormat = "JWT",
        Description = "JWT Authorization header using the Bearer scheme."
    });
 
    // This section allows submitting the token with your request in Swagger.
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

Also make sure that the the following two lines are added right before app.MapControllers(), near the bottom of the Program.cs file.

```csharp 
app.UseSwagger();
app.UseSwaggerUI();
```

With this in place, we should be able to test the authentication.

## Verify the authentication
Run the program and go to <https://localhost:7051/swagger/index.html>, if it doesn’t send you there automatically. You should be able to see two or three end points (the third one is the weather forecast controller, which comes with the template).

You can ignore the WeatherForecast end point. It is auto generated by the template.
The first step is to obtain a token via the POST method for /api/Token. The request body must contain the username and password, formated as shown below. You don’t need to change the username or password as we made the validation method return true no matter what.

```
{
  "username": "string",
  "password": "string"
}
```

When you execute the request, you get the a response looking something like the one shown in the image below. The cryptic string is the encoded token. Copy it as it is used as input for the next step.

The response after posting a login request via Swagger.
As said. Copy the token and then, at the GET method, click the lock at the top right of the panel. This will open a new window where the token can be posted.

Once you have pasted the token and clicked Authorize, the lock icon changes from open to locked. This means that it will now use the token for authorization. With this in place you can execute and (hopefully) get a 200 Ok response, which means that the authorization worked as intended.

The request was successful. Without the token a 401 Unauthorized would have been returned.
If you think this was a lot, for the simplest possible implementation of JWT authentication, I’m inclined to agree. But on the other hand, I don’t see how it would be a whole lot simpler. After all, authentication is quite complicated, considering that if it’s done well, the user hardly notices it. I know that I left a lot of unanswered questions, but I hope this post can help you get started on a difficult topic.

##S Additional resources
**YouTube video:** [Upgrading to .NET Core: Adding JWT Authentication to Our API – A TimCo Retail Manager video (by Tim Corey)](https://www.youtube.com/watch?v=9QU_y7-VsC8)

*My primary inspiration for this blog. Tim describes JWT authentication in great details (as always). The video is a few years old and I had to change a few things to make it work in my example.*

**YouTube video:**
[What Is JWT and Why Should You Use JWT (by Web Dev Simplified)](https://www.youtube.com/watch?v=7Q17ubqLfaM)

*A relatively short but great introduction to JWT. Both about what it is, but also how they are advantageous over other types of authentication.*

**Website:** [jwt.io](https://jwt.io/)

*A great website where you can play around with a JWT and see how it changes the encoded token. And how changing the encoded token makes it invalid. Also has a great written introduction to JWT.*