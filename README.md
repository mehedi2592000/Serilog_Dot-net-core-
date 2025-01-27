# Serilog in .NET Core

There are multiple ways to save log messages:
1. Local file save
2. Save to a database

### Install Required Libraries
```bash
# Add the following packages to your project:
dotnet add package Serilog.AspNetCore   
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File      # For local file save
dotnet add package Serilog.Sinks.Seq       # For Seq dashboard
dotnet add package Serilog.Sinks.MSSqlServer # For saving logs in a database
```

---

### Use `Program.cs` File
```csharp
var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(configuration) // Load configuration from appsettings.json
    .CreateLogger();

// Use Serilog for logging
builder.Host.UseSerilog();
```

---

### Use `appsettings.json` File
```json
{
  "Serilog": {
    "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.File", "Serilog.Sinks.Seq", "Serilog.Sinks.Email", "Serilog.Sinks.MSSqlServer" ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Error"
      }
    },
    "WriteTo": [
      { "Name": "Console" },
      {
        "Name": "File",                 // Write to a local file and save a new file every day
        "Args": {
          "path": "Logs/myapp-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7,
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "Seq",                  // Add logger dashboard
        "Args": {
          "serverUrl": "http://localhost:5341",
          "apiKey": "your-api-key",
          "restrictedToMinimumLevel": "Information"
        }
      },
      {
        "Name": "MSSqlServer",         // Save logs to the database
        "Args": {
          "connectionString": "Server=.;Database=LogsDB;Integrated Security=True;",
          "tableName": "Logs",
          "autoCreateSqlTable": true,
          "restrictedToMinimumLevel": "Information"
        }
      }
    ],
    "Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId", "UserIdEnricher" ],
    "Filter": [
      {
        "Name": "ByExcluding",
        "Args": {
          "expression": "StartsWith(SourceContext, 'Microsoft')"
        }
      }
    ]
  }
}
```

---

### Create Database Table for MSSqlServer Sink
```sql
CREATE TABLE Logs (
    Id INT IDENTITY PRIMARY KEY,
    Message NVARCHAR(MAX),
    MessageTemplate NVARCHAR(MAX),
    Level NVARCHAR(128),
    TimeStamp DATETIME,
    Exception NVARCHAR(MAX),
    Properties NVARCHAR(MAX)
);
```

---

### Seq Docker Configuration
```bash
docker run -d --name seq -p 5341:80 -p 5342:443 -e ACCEPT_EULA=Y datalust/seq:latest
```

### we can Add Custom Enrichers
``` code
using Serilog.Core;
using Serilog.Events;

public class UserIdEnricher : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var userId = "12345"; // Replace with actual user ID (e.g., from HttpContext)
        var userIdProperty = propertyFactory.CreateProperty("UserId", userId);
        logEvent.AddPropertyIfAbsent(userIdProperty);
    }
}

```

```
"Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId", "UserIdEnricher" ]
```
