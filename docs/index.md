# Capa de datos
Las aplicaciones de backend desarrolladas generalmente siguen un organización de capas como la que se muestra en la imagen a continuación

![](img/step00.layers_of_application.png "Application layers")

El presente material establece una guía sobre como inicializar, configurar, verificar e implementar una capa de acceso a datos para los proyectos de APSYS. Para ejemplificar de manera práctica, se desarrollará el código requerido para un proyecto llamado `bookstore`.

> Esta guía explica como realizar la implementación usando los patrones de diseño `unitOfWork` y `repository`. Para mayor documentación sobre estos patrones de diseño, puedes consultar las referencias: 

> __https://martinfowler.com/eaaCatalog/unitOfWork.html__
> __https://martinfowler.com/eaaCatalog/repository.html__

## Inicialización de un proyecto

Crea una solución en blanco usando Visual Studio 2017 o 2019, con el nombre `apsys.training.bookstore.sln`. Dentro de esa solución agrega los siguientes proyecto, organizados como se muestra a continuación

| Proyecto                                         | Tipo de proyecto      | Carpeta   |
| -----                                            | ------                | -----     |
| apsys.training.bookstore                         | Biblioteca de clases  | 02.domain |
| apsys.training.bookstore.repositories            | Biblioteca de clases  | 01.data   |
| apsys.training.bookstore.repositories.nhibernate | Biblioteca de clases  | 01.data   |
| apsys.training.bookstore.repositories.nhibernate | Biblioteca de clases  | 01.data   |
| apsys.training.bookstore.repositories.testing    | NUnit                 | 01.data   |
| apsys.training.bookstore.migrations              | Aplicación de consola | 00.tools  |

![](img/step01.project_structure.png "Project structure")

## Creación de la base de datos

Para persistir la información, en esta guìa usaremos `Microsoft SQL Server Express` sin embargo cualquier base de datos relacional, como `MySQL`, `Postgress`, pueden ser usadas, realizando los ajustes requeridos en la configuración

Usando `Microsoft SQL Server Management Studio`, crea una base de datos con el nombre `BookStore`

![](img/step02.add_database.png "Database")

## Definición del dominio
La capa de datos se encarga de persistir y recuperar nuestras entidades del dominio de una base de datos, por lo que necesitamos de primera instancia entidades en nuestro dominio. 

Agrega las siguientes clases a el proyecto `apsys.training.bookstore`

```c#
public class Author
{
    public string Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public IEnumerable<Book> Books { get; set; }
}
```

```c#
public class Book
{
    public string Id { get; set; }
    public string Title { get; set; }
    public string ISBN { get; set; }
    public string Genre { get; set; }
    public DateTime PublishDate { get; set; }
    public Author Author { get; set; }
}
```

## Creación del esquema de datos

Ahora que tenemos un dominio definido, podemos crear las tablas en nuestra base de datos donde se almacenará su información. Para mantener el esquema de la base de datos bajo control de versiones, usaremos una herramienta llamada Migrator.NET, asi como una serie de paquetes adicionales


> Todo cambio realizado en el esquema de la base de datos deberá ser realizado a través de una migración para asegurar la correcta distribución de estos en los servidores de prueba, producción y ambientes de desarollo del resto del equipo de trabajo


Abre el proyecto `apsys.training.bookstore.migrations` e instala los siguientes paquetes

```
Install-Package FluentMigrator -Version 3.3.1
Install-Package FluentMigrator.Extensions.SqlServer -Version 3.3.1
Install-Package FluentMigrator.Runner -Version 3.3.1
```

### Agregar tabla _Authors_

Vamos a crear nuestra primer tabla que almacenará la información de nuestra entidad `Author`. En el mismo proyecto, agrega una clase llamada `M01_CreateAuthorsTable` como se muestra a continuación

```c#
[Migration(1)]
public class M01_CreateAuthorsTable: Migration
{
    public override void Up()
    {
        Create.Table("Authors")
            .WithColumn("Id").AsString().PrimaryKey()
            .WithColumn("FirstName").AsString().NotNullable()
            .WithColumn("LastName").AsString().NotNullable();
    }

    public override void Down()
    {
        Delete.Table("Authors");
    }
}
```

El método `Up` deberá contener los cambios a realizar en nuestro esquema de la base de datos. En este caso, agregamos una tabla llamada `Authors` con sus campos correspondientes. El método `Down` deberá contener el código que permita deshacer los cambios realizados en el método `Up`. En este caso, la eliminación de la tabla `Authors`

### Ejecutando las migraciones
Para ver reflejados los cambios en nuestra base de datos, debemos ejecutar nuestras migraciones. Abre el archivo `Program.cs` del proyecto de migraciones y escribe el siguiente código

```c#
public class Program
{
    static int Main()
    {
        try
        {
            CommandLineArgs parameter = new CommandLineArgs();
            if (!parameter.ContainsKey("cnn"))
                throw new ArgumentException("No [cnn] parameter received. You need pass the connection string in order to execute the migrations");
            
            string connectionString = parameter["cnn"];
            var serviceProvider = CreateServices(connectionString);
            using var scope = serviceProvider.CreateScope();
            UpdateDatabase(scope.ServiceProvider);
            return (int)ExitCode.Success;
        }
        catch (Exception ex)
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"Error updating the database schema: {ex.Message}");
            Console.ResetColor();
            return (int)ExitCode.UnknownError;
        }
    }

    private static IServiceProvider CreateServices(string connectionString)
    {
        return new ServiceCollection()
            .AddFluentMigratorCore()
            .ConfigureRunner(rb => rb
                .AddSqlServer2016()
                .WithGlobalConnectionString(connectionString)
                .ScanIn(typeof(M01_CreateAuthorsTable).Assembly).For.Migrations())
            .AddLogging(lb => lb.AddFluentMigratorConsole())
            .BuildServiceProvider(false);
    }

    private static void UpdateDatabase(IServiceProvider serviceProvider)
    {
        var runner = serviceProvider.GetRequiredService<IMigrationRunner>();
        runner.MigrateUp();
    }

}
    
enum ExitCode
{
    Success = 0,
    UnknownError = 1
}
    
class CommandLineArgs : Dictionary<string, string>
{
    private const string Pattern = @"\/(?<argname>\w+):(?<argvalue>.+)";
    private readonly Regex _regex = new Regex(Pattern, RegexOptions.IgnoreCase | RegexOptions.Compiled);

    public CommandLineArgs()
    {
        var args = Environment.GetCommandLineArgs();
        foreach (var match in args.Select(arg => _regex.Match(arg)).Where(m => m.Success))
            this.Add(match.Groups["argname"].Value, match.Groups["argvalue"].Value);
    }
}
```

## Unit of Work

```
Install-Package apsys.repository.core
```

```c#
public interface IUnitOfWork
{
    void Commit();
    void Rollback();
}
```

```c#
public interface IAuthorsRepository: IRepository<Author>
{
}
```

```c#
public interface IUnitOfWork
{
    IAuthorsRepository Authors { get; }
    void Commit();
    void Rollback();
}
```

## Implementing Unit of Work with NHibernate

```
Install-Package apsys.repository.core
Install-Package apsys.repository.nhibernate.core
Install-Package NHibernate
Install-Package FluentNHibernate
```

```c#
public class UnitOfWork: IUnitOfWork
{
    private readonly ISession _session;
    private ITransaction _transaction;

    public UnitOfWork(ISession session)
    {
        _session = session;
        this._transaction = session.BeginTransaction();
    }
    
    public IAuthorsRepository Authors { get; }
    
    public void Commit()
    {
        if (this._transaction != null && this._transaction.IsActive)
            this._transaction.Commit();
        else
            throw new TransactionException("The actual transaction is not longer active");
    }

    public void Rollback()
    {
        if (this._transaction != null && this._transaction.IsActive)
            this._transaction.Rollback();
        else
            throw new TransactionException("The actual transaction is not longer active");
    }
}
```

```c#
public class AuthorsRepository: Repository<Author>,  IAuthorsRepository 
{
    public AuthorsRepository(ISession session) 
        : base(session)
    {
    }
}
```


```c#
public class UnitOfWork: IUnitOfWork
{    
    ...

    public UnitOfWork(ISession session)
    {
        _session = session;
        this._transaction = session.BeginTransaction();
        this.Authors = new AuthorsRepository(this._session);
    }    
    ...
}
```

```c#
public class AuthorsMappers: ClassMapping<Author>
{

    public AuthorsMappers()
    {
        Table("ApplicationRoles");
        Id(x => x.Id, x =>
        {
            x.Generator(Generators.Assigned);
            x.Column("Id");
        });
        Property(x => x.FirstName, x =>
        {
            x.Column("FirstName");
        });            
        Property(x => x.LastName, x =>
        {
            x.Column("LastName");
        });
    }
}
```

## Testing unit of work

```
Install-Package FluentAssertions
Install-Package NUnit
Install-Package NUnit3TestAdapter
```


