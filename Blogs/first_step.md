# .NET 6 for API development - First Step

This tutorial teaches the basics of building a web API with .NET 6 and ASP.NET 6.

In this tutorial, you will learn how to:

* Create a web API project.
* Add a model class.
* Scaffold a controller.
* Configure routing, URL paths, and return values.

At the end, you will have a web API that can:

* Call postgresql stored procedure to validate user.
* Call another API to trigger one time passcode.
* Return a JSON response.

## Overview

| API | Description | Request Body | Response Body |
| ----------- | ----------- | ----------- | ----------- |
| POST /wx/v1/loyalty/rewards/customer-hub/cards/login | Login and send one time passcode to email or mobile | LoginRequestDto | LoginResponseDto |

![First step: architecture](first_step_architecture.svg)

## Prerequisites

* [Visual Studio Code](https://code.visualstudio.com/download)
* [C# for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp)
* [.NET 6.0 SDK](https://dotnet.microsoft.com/download/dotnet/6.0)

## Create a web project

* Open Visual Studio Code
* Open the integrated terminal
* Change directories (`cd`) to the folder that will contain the project folder.
* Run the following commands:
```
dotnet new webapi -o wx-api-rewards-customer-hub
cd wx-api-rewards-customer-hub
code -r ../wx-api-rewards-customer-hub
```
* These commands:
  * Create a new web API project and open it in Visual Studio Code.
  * When a dialog box asks if you want to add required assets to the project, select **Yes**.

## Test the project

* Trust the HTTPS development certificate by running the following command:
```
dotnet dev-certs https --trust
```
* Start the Kestrel web server with hot reload
```
dotnet watch run
```

The Swagger page `/swagger/index.html` is displayed. Select **GET** > **Try it out** > **Execute**. The page displays:

* The curl command to test the WeatherForecast API.
* The URL to test the WeatherForecast API.
* The response code, body, and headers.
* A drop down list box with media types and the example value and schema.

## Use multiple environments
The file `Properties/launchSettings.json` has important environmental settings for web server, launch url, etc.

```
"profiles": {
  "dev": {
    "commandName": "Project",
    "dotnetRunMessages": true,
    "launchBrowser": true,
    "launchUrl": "swagger",
    "applicationUrl": "https://localhost:3500",
    "environmentVariables": {
      "ASPNETCORE_ENVIRONMENT": "Development"
    }
  },
  "prod": {
    "commandName": "Project",
    "dotnetRunMessages": true,
    "launchBrowser": false,
    "applicationUrl": "https://localhost:3500",
    "environmentVariables": {
      "ASPNETCORE_ENVIRONMENT": "Production"
    }
  },
  "IIS Express": {
    "commandName": "IISExpress",
    "launchBrowser": true,
    "environmentVariables": {
      "ASPNETCORE_ENVIRONMENT": "Development"
    }
  }
}
```

By Default, dotnet runs the first profile, run the following command to explicitly select profile:
```
dotnet run --launch-profile "dev"
```
For this example, dev/prod both listens on port 3500, using Kestrel as the web server, dev will launch user's default browser and display swagger. 

For details please refer to Microsoft's official document - [Use multiple environments in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/environments?view=aspnetcore-6.0).

## Add a model class
A *model* is a set of classes that represent the data that the app manages.
- Add a folder named *Models*.
- Add a `Member.cs` file to the Models folder with the following code:
```C#
namespace wx_api_rewards_customer_hub.Models;

public class Member
{
    public string CRN { get; set; }
    public string CardNumber { get; set; }
    public string Email { get; set; }
    public string Mobile { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```
Model classes can go anywhere in the project, but the *Models* folder is used by convention.

## Add DTO classes
A DTO(Data Transfer Object) may be used to:

- Prevent over-posting.
- Hide properties that clients are not supposed to view.
- Omit some properties in order to reduce payload size.
- Flatten object graphs that contain nested objects. Flattened object graphs can be more convenient for clients.

Add `LoginMemberDTO` class into `Member.cs`, this will be used as the incoming request object:
```C#
public class LoginMemberDTO
{
    public string Preferred { get; set; }
    public string Email { get; set; }
    public string Mobile { get; set; }
}
```

Add `OTPTokenDTO` class into `Member.cs`, this will be used as the embedded reponse object of `Response` type:
```C#
public class OTPTokenDTO
{
    public string Token { get; set; }
}
```

Add `Response.cs`, this type is a simple wrapper of actual responses, to provide a general API interface with `data` as root:
```C#
public class Response<T>
{
    public T? Data { get; set; }
}
```
The OTPTokenDTO JSON response with this wrapper(`Response<OTPTokenDTO>>`) looks like:
```JSON
{
    "data": {
        "token": "TEST OTP Token"
    }
}
```

## Scaffold a controller
```
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet tool install -g dotnet-aspnet-codegenerator
```
Let's have a look at the generated controller code:
```C#
using Microsoft.AspNetCore.Mvc;

namespace wx_api_rewards_customer_hub.Controllers
{
    [Route("wx/v1/loyalty/rewards/customer-hub/[controller]/[action]")]
    [ApiController]
    public class CardsController : ControllerBase
    {
    }
}
```
The generated code:
- Uses [`Microsoft.AspNetCore.Mvc`](https://docs.microsoft.com/en-us/aspnet/core/mvc/overview?view=aspnetcore-6.0) web app framework.
- Marks the class with the [`[ApiController]`](https://docs.microsoft.com/en-us/aspnet/core/web-api/?view=aspnetcore-6.0#apicontroller-attribute)] attribute. This attribute indicates that the controller responds to web API requests.
- Adds a routing with the [[Route("wx/v1/loyalty/rewards/customer-hub/[controller]/[action]")]](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/routing?view=aspnetcore-6.0#verb) attribute. This attribute indicates that the controller responds to web API requests with path `/wx/v1/loyalty/rewards/customer-hub/Cards/[Action]`.
- Action method name will be used to match the `[Action]` component of the path.
- All path components are case insensitive. 

## Generate a default `.gitignore` then initialize the git repo
```
dotnet new gitignore
git init
```
`appsettings.*.json` will provide app configurations, e.g. database connection parameters. Add them into .gitignore file for security reason:
```
echo "\n# appsettings\n**/appsettings*" >> .gitignore
git add .
git commit
```

## Add an action method
```C#
[HttpPost]
public async Task<ActionResult<Response<OTPTokenDTO>>> Login(LoginMemberDTO loginMemberDTO)
{
    OTPTokenDTO result = new OTPTokenDTO
    {
        Token = "TEST OTP Token"
    };
    return Created(string.Empty, new Response<OTPTokenDTO>
    {
        Data = result
    });
}
```
This action method reponds to `POST /wx/v1/loyalty/rewards/customer-hub/cards/login` requests.
It also:
- Takes a `LoginMemberDTO` object.
- Responds a `Response<OTPTokenDTO>>` object.
- Creates a `201 Created` HTTP status code.
- Creates an empty url in `Location` header.

## Add basic validation
Add `using System.ComponentModel.DataAnnotations;` and `[Required]` to all properties of `LoginRequestDTO`:
```C#
public class LoginMemberDTO
{
    [Required]
    public string Preferred { get; set; }
    [Required]
    public string Email { get; set; }
    [Required]
    public string Mobile { get; set; }
}

```
This change will display these properties as required in swagger:
![First step: required](first_step_required_property.png)

The actual validations is based on C# type information, for this example, because all properties are not-nullable types(`string` instead of `string?`), if any one of them is missing in the incoming json, like this:
```JOSN
{
    "email": "",
    "mobile": null
}
```
The following HTTP 400 Bad Request error is returned:
```JSON
{
    "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
    "title": "One or more validation errors occurred.",
    "status": 400,
    "traceId": "00-3ea681402c4fe48d48a33b163529e2ca-0f33f06c49349c9f-00",
    "errors": {
        "Email": [
            "The Email field is required."
        ],
        "Mobile": [
            "The Mobile field is required."
        ],
        "Preferred": [
            "The Preferred field is required."
        ]
    }
}
```
What if given wrong type? E.g. using int instead of string for `Preferred`:
```JSON
{
    "preferred": 123,
    "email": "sean.linxs@gmail.com",
    "mobile": "0431071979"
}
```
Then the whole incoming object will be marked invalid:
```JSON
{
    "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
    "title": "One or more validation errors occurred.",
    "status": 400,
    "traceId": "00-21a04d54426e7ca524978fa8d2247b78-afe2bd5f33ac3777-00",
    "errors": {
        "loginMemberDTO": [
            "The loginMemberDTO field is required."
        ],
        "$.preferred": [
            "The JSON value could not be converted to System.String. Path: $.preferred | LineNumber: 1 | BytePositionInLine: 20."
        ]
    }
}
```

## Add postgresql service
[`Npgsql`](https://www.npgsql.org/index.html) is an open source ADO.NET Data Provider for PostgreSQL, it allows programs written in C#, Visual Basic, F# to access the PostgreSQL database server. It is implemented in 100% C# code, is free and is open source.

Add `Npgsql` package:
```
dotnet add package Npgsql
```
[Configuration in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-6.0) has details about various ways to inject application configurations:
- Settings files, such as `appsettings.json`
- Environment variables
- Azure Key Vault
- Azure App Configuration
- Command-line arguments
- Custom providers, installed or created
- Directory files
- In-memory .NET objects

Since json is both expressive and accurate, this tutorial will use `appsettings.json` for configurations.

Create Development environment database connection paramaters in `appsettings.Development.json`:
```
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "CDP": {
    "Host": "localhost",
    "Port": "8888",
    "Database": "lcdm",
    "Username": "api_mandy",
    "Password": "drXXAA24uUtjRcnH66pt"
  }
}
```
The next step is to add a service class `CdpService` and inject into controller `CardsController`, please refer to the [DI](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-6.0) document for details.

I will discuss dotnet 6 and database in a separate blog, for now, simply create a `Services` folder, then create a file `CdpService.cs` underneath:
```C#
using Npgsql;
using wx_api_rewards_customer_hub.Models;

public enum CDP_SOURCE
{
    CPORTAL = 1,
    MANDY = 2
}

public interface ICdpService
{
    Task<LoginResultDTO> Login(LoginMemberDTO loginRequestDTO);
}

public class CdpService : ICdpService
{
    private readonly IConfiguration _configuration;
    private readonly string _connString;

    public CdpService(IConfiguration configuration)
    {
        _configuration = configuration;
        var host = _configuration["CDP:Host"];
        var port = _configuration["CDP:Port"];
        var database = _configuration["CDP:Database"];
        var username = _configuration["CDP:Username"];
        var password = _configuration["CDP:Password"];
        _connString = $"Host={host};Port={port};Database={database};Username={username};Password={password}";
    }

    public async Task<LoginResultDTO> Login(LoginMemberDTO loginRequestDTO)
    {
        await using var _conn = new NpgsqlConnection(_connString);
        await _conn.OpenAsync();

        await using var cmd = new NpgsqlCommand("SELECT * FROM rtapi.sp_get_mandy_order_card($1, $2, $3, $4, $5, $6)", _conn);
        cmd.Parameters.AddWithValue(loginRequestDTO.Preferred == PreferredDevice.Email ? loginRequestDTO.Email : DBNull.Value);
        cmd.Parameters.AddWithValue(loginRequestDTO.Preferred == PreferredDevice.Mobile ? loginRequestDTO.Mobile : DBNull.Value);
        cmd.Parameters.AddWithValue(DateTime.Now);
        cmd.Parameters.AddWithValue((int)CDP_SOURCE.CPORTAL);
        cmd.Parameters.AddWithValue(CDP_SOURCE.CPORTAL.ToString());
        cmd.Parameters.AddWithValue(DBNull.Value);

        await using var reader = await cmd.ExecuteReaderAsync();

        await reader.ReadAsync();
        return new LoginResultDTO
        {
            ErrorCode = reader.IsDBNull(0) ? null : reader.GetString(0),
            ErrorMessage = reader.IsDBNull(1) ? null : reader.GetString(1),
            Result = reader.GetFieldValue<int>(2),
            CRN = reader.IsDBNull(3) ? null : reader.GetString(3),
            CardNumber = reader.IsDBNull(4) ? null : reader.GetString(4),
            Email = reader.IsDBNull(5) ? null : reader.GetString(5),
            Mobile = reader.IsDBNull(6) ? null : reader.GetString(6),
            FirstName = reader.IsDBNull(7) ? null : reader.GetString(7),
            LastName = reader.IsDBNull(8) ? null : reader.GetString(8)
        };
    }
}
```
Then register it in `Program.cs`:
```C#
...
// Add services to the container.
builder.Services.AddScoped<ICdpService, CdpService>();
...
```