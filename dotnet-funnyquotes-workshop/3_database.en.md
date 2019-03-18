## Databases

### Run App locally and show db initialization via EF code first

1. Run MySQL locally via docker

    ```
    > docker pull mysql
    > docker run --name mysql -p 3306:3306/tcp -d -e MYSQL_ALLOW_EMPTY_PASSWORD=yes mysql:5.7
    ```

1. Connect to MySQL via CLI 

    ```
    > docker run -it --link mysql:mysql --rm mysql sh -c "exec mysql -h$MYSQL_PORT_3306_TCP_ADDR -P$MYSQL_PORT_3306_TCP_PORT -uroot -p$MYSQL_ENV_MYSQL_ROOT_PASSWORD"
    ```

1. Create empty database

    ````
    > CREATE DATABASE funnyquotes;
    ````

1. Launch FunnyQuotesServiceOwin locally, either from Visual Studio or Rider (must be on Windows)
1. Demonstrate database initialization by app startup by querying the database. From MySQL CLI:

    ```
    > USE funnyquotes;
    > SHOW Tables;
    > select * from FunnyQuotes;
    ```

1. Show migration table and it's content

    ```
    > DESCRIBE __MigrationHistory;
    > select MigrationId, ContextKey, ProductVersion From __MigrationHistory;
    ```

### Demo creating new migration

1. Change `FunnyQuotesCookieDatabase.FunnyQuote` class to add a new field:

    ```csharp
     public class FunnyQuote
     {
        public int Id { get; set; }
        public string Quote { get; set; }
        public long Views { get; set; }
     }
    ```

1. From Visual Studio Package Manager console, run 

    ```
    > EntityFramework\Add-Migration -Name ViewCountField -Project FunnyQuotesCookieDatabase -StartUpProject FunnyQuotesLegacyService
    ```

1. Highlight that startup project argument is used to determine the connection to the database in order to compare the schema. Web.Config must have ConnectionString section even if it's not used in normal course of the application to run this command.
    1. Show the new migration that was added under `FunnyQuotesCookieDatabase\Migrations`.
    1. Show that at this point the database is still based on old schema

    ```
    > DESCRIBE FunnyQuotes;
    ```
1. Run FunnyQuotesServiceOwin again and hit endpoint that uses the database to force migration to be applied

    http://localhost:61111/api/funnyquotes/random

1. Confirm that new migration is applied on the database.

    ```
    DESCRIBE FunnyQuotes;
    select MigrationId, ContextKey, ProductVersion From __MigrationHistory;
    ```

  Extract column should now be visible and new record in migration table

### Push to PCF
1. Provision a MySQL instance from marketplace. Use `mysql-funnyquotes` as name
1. Push Owin backend

    ```
    > cd FunnyQuotesServicesOwin
    > cf push FunnyQuotesServicesOwin -s windows2016 -b hwc_buildpack
    ```

1. Bind to MySQL service and restart
1. Confirm that everything works by hitting `/api/funnyquotes/random` endpoint
1. Open up `FunnyQuotesServicesOwin.Startup` class and explain use of Steeltoe Connectors to initialize db context. Highlight the following lines of code

    ```csharp
    builder.RegisterMySqlConnection(config);
    builder.Register(ctx => // register EF context
    {
        var connString = ctx.Resolve<IDbConnection>().ConnectionString;
        return new FunnyQuotesCookieDbContext(connString);
    });
    ```                
  Explain that helper methods exist when registering EF Core, but for EF 6.x IDbConnection get's auto configured, and we can feed it into EF registration as per above

1. Push FunnyQuotesLegacyService using manifest and how things can bind to services without needing to restart