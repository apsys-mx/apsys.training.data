# Documentación de capa de datos


## Estructura inicial del proyecto
![](img/step01.project_structure.png "Project structure")

## Definición del dominio


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


```c#
public class Author
{
    public string Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public IEnumerable<Book> Books { get; set; }
}
```