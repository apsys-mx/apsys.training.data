# UnitOfWork y Repositories

Ahora que tenemos un dominio, y un esquema de base de datos, necesitamos crear los elementos que nos permitan realizar las operaciones de almacenamiento y lectura de nuestras entidades la base de datos

Esta guia explica como implementar el UnitOfWork usando el ORM <a href="https://nhibernate.info/">NHibernate</a>, que por su madurez y versatilidad a sigo la opción por default en los proyectos de APSYS. 

Sin embargo, la implementación del UnitOfWork puede realizarse con otros ORM's como Microsoft Entity Framework, o incluso sin el uso de ORM alguno, generando las sentencias SQL directamente en código o a través de procedimientos almacenados

## Definición de UnitOfWork
El primer paso consiste en crear las interfaces que definirán las operaciones de nuestras entidades hacia la base de datos.

Abre el proyecto `apsys.training.bookstore.repositories.csproj` e instala los siguientes paquetes

```
Install-Package apsys.repository.core
```
Posteriormente, definiremos la interface de nuestro `UnitOfWork` como se muestra a continuación

```c#
public interface IUnitOfWork
{
    void Commit();
    void Rollback();
}
```
Esta primer definición solo contiene los métodos que nos permitirá ejecutar la transacción de los cambios realizados en todas nuestras entidades de manera atómica, consistente y perdurable, o por el contrario, revertirlos en caso de un error


## Definición de AuthorsRepository

Ahora agregaremos la definicion de nuestro primer repositorio que se encargará de realizar las operaciones *CRUD* de la entidad `Authors` en la base de datos

```c#
public interface IAuthorsRepository: IRepository<Author>
{
}
```

Como podrás observar, `IAuthorsRepository` hereda de la interface `IRepository<T>`, contenida en el paquete `apsys.repository.core`.

Esta interface `IRepository` ha sido diseñada para definir las operaciones CRUD básicas como `Add`, `Remove`, `Get` y deberá ser usada en todos los repositorios de nuestra aplicación

> La interface IRepository define un tipo de dato genérico `<T>` para encapsular la definición de cualquier entidad de cualquier dominio. Para mayor información puede consultar la referencia <a href="https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/types/generics">https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/types/generics</a>

Una vez definido nuestro primer repositorio es necesario incluirlo en nuestro `UnitOfWork`. Abre el archivo `IUnitOfWork.cs` y modifica como se muestra a continuación

```c#
public interface IUnitOfWork
{
    IAuthorsRepository Authors { get; }
    void Commit();
    void Rollback();
}
```

> Nota que la declaración de la propiedad `Authors` en `IUnitOfWork` es *read-only*

## UnitOfWork con NHibernate

Con la definición de nuestras interfaces, podemos realizar la implementación usando NHibernate. Para eso necesitamos abrir el proyecto `apsys.training.bookstore.repositories.nhibernate` e instalar los siguientes paquetes

```
Install-Package apsys.repository.core
Install-Package apsys.repository.nhibernate.core
Install-Package NHibernate
Install-Package FluentNHibernate
```

Agrega una clase llamada `UnitOfWork.cs` y realiza la implementación de la inteface `IUnitOfWork` como se muestra a continuación

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

Nota que en el constructor recibe una entidad de tipo `ISession`. Esta entidad pertenece a *NHibernate* e implementa a su vez el patrón de UnitOfWork. Nuestros repositorios usarán esta entidad para realizar todas sus operaciones hacia la base de datos

> <a href="https://nhibernate.info/doc/nhibernate-reference/architecture.html">https://nhibernate.info/doc/nhibernate-reference/architecture.html</a> 

## AuthorsRepository con NHibernate

Crea una clase llamada `AuthorsRepository` e implementa `IAuthorsRepository` como se muestra a continuación

```c#
public class AuthorsRepository: Repository<Author>, IAuthorsRepository 
{
    public AuthorsRepository(ISession session) 
        : base(session)
    {
    }
}
```

Nota que la clase `AuthorsRepository` hereda de la clase `Repository<T>` contenida en el paquete `apsys.repository.nhibernate.core`.

Esta clase `Repository` implementa las operaciones CRUD usando la sesión de NHibernate

Una vez definida nuestro repositorio, debemos inicializarlo en el UnitOfWork. Abre el archivo `UnitOfWork.cs` y modificalo como se muestra a continuación

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

## Mapper de Author

NHibernate necesita saber como mapear nuestras entidades de dominio contra la base de datos. Este mapeo lo realizamos a través de clases que heredan de `ClassMapping`

Abre el proyecto `apsys.training.bookstore.repositories.nhibernate` y agrega una carpeta llamada `mappers`. Dentro de esta carpeta agrega una clase llamada `AuthorsMappers` e implementa como se muestra a continuación

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
