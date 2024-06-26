# Notes of Entity Framework Core In Action (2nd Edition)

## 1. Introduction to Entity Framework Core

EF Core is designed as an *object-relational mapper (O/RM)*. O/RMs work by mapping between two worlds: the relational database, with its own API, and the object-oriented software world of classes and software code.



### The Downsides of O/RMs

1. *Object-relational impedance mismatch*. Database servers and object-oriented software use different principles; databases use primary keys to define that a row is unique, whereas .NET class instances are, by default, considered unique by their reference.

2. EF Core "hides" the database so well that you can sometimes forget about the database underneath. This problem can cause you to write code that would work well in C# but doesn’t work for a database.

   ````c#
   public string FullName => $"{FirstName} {LastName}";
   ````

   An expression body property such as the one just shown is the right thing to do in C#, but the same property would throw an exception if you tried to filter or order on that property, because EF Core needs a FullName column in the table so that it can apply an SQL `WHERE` or `ORDER` command at the database level.



### First Ef Core Application



![](./diagrams/svg/01_01_first_ef_core_application_tables.drawio.svg)



````c#
// EF Core maps .NET classes to database tables.
public class Book
{
    // A class needs a primary key.
    // We are using an EF Core naming convention that tells EF Core that the property BookId is the primary key.
    public int BookId { get; set; }
    
    // these properties are mapped to the table's columns.
    public string Title { get; set; }
    public string Description { get; set; }
    public DateTime PublishedOn { get; set; }
    
    // The AuthorId foreign key is used in the database to link a row in the Books table to a row in the Author table.
    public int AuthorId { get; set; }
    
    // The Author property is an EF Core navigational property. EF Core uses this on a save to see whether the Book has an Author class attached. If so, it sets the foreign key, AuthorId.
    // Upon loading a Book class, the method Include will fill this property with the Author class that's linked to this book class by using the foreign key, AuthorId.
    public Author Author { get; set; }
}

public class Author
{
    // Note that the foreign key in the Book class has the same name.
    public int AuthorId { get; set; }
    public string Name { get; set; }
    public string WebUrl { get; set; }
}
````

````c#
// DbContext class holds the information and configuration for accessing database.
public class AppDbContext : DbContext
{
    private const string ConnectionString = "Server=(localdb)\mssqllocaldb;Database=FirstEfCoreDb;Trusted_Connection=True";
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer(connectionString);
    }
    
    // By creating a property called Books of type DbSet<Book>, you tell EF Core that these's a database table named Books, and it has the columns and keys as found in the Book class.
    public DbSet<Book> Books { get; set; }
    
    // Database has a table called Author, but purposely didn't create a property for that table.
    // EF Core finds that table by finding a navigational property of type Author in the Book class.
}
````

Two main parts of the application’s DbContext created for the application. First, the setting of the database options defines what type of database to use and where it can be found. Second, the `DbSet<T>` property (or properties) tell(s) EF Core what classes should be mapped to the database.

EF Core will create a model of the database your classes map to. First, it looks at the classes you have defined via the `DbSet<T>` properties; then it looks down all the references to other classes. Using these classes, EF Core can work out the default model of the database. But then it runs the `OnModelCreating` method in the application’s DbContext, which you can override to add your specific commands to configure the database the way you want it.

The following text provides a more detailed description of the process:

* EF Core looks at the application’s DbContext and finds all the public `DbSet<T>` properties. From this data, it defines the initial name for the one table it finds: Books.
* EF Core looks through all the classes referred to in `DbSet<T>` and looks at its properties to work out the column names, types, and so forth. It also looks for special attributes on the class and/or properties that provide extra modeling information.
* EF Core looks for any classes that the `DbSet<T>` classes refer to. In our case, the `Book` class has a reference to the `Author` class, so EF Core scans that class too. It carries out the same search on the properties of the `Author` class as it did on the `Book` class in step 2. It also takes the class name, `Author`, as the table name.
* For the last input to the modeling process, EF Core runs the virtual method `OnModelCreating` inside the application’s DbContext. In this simple application, you don’t override the `OnModelCreating` method, but if you did, you could provide extra information via a fluent API to do more configuration of the modeling.
* EF Core creates an internal model of the database based on all the information it gathered. This database model is cached so that later accesses will be quicker. Then this model is used for performing all database accesses.



````c#
// The code to read all the books and output them to the console
public static void ListAll()
{
    using(var db = new AppDbContext())
    {
        // Reads all the books. AsNoTracking indicates that this access is read-only.
        // The include causes the author information to be loaded with each book.
        foreach(var book in db.Books.AsNoTracking()
               .Include(book => book.Author))
        {
            var webUrl = book.Author.WebUrl ?? "- no web URL given -";
            Console.WriteLine($"{book.Title} by {book.Author.Name}");
            Console.WriteLine("	" + "Published on " + $"{book.PublishedOn:dd-MMM-yyyy}" + $". {webUrl}");
        }
    }
}
````



![](./diagrams/images/01_01_ef_core_executes_database_query.png)



````sql
-- SQL command produced to read Books and Author
SELECT [b].[BookId],
[b].[AuthorId],
[b].[Description],
[b].[PublishedOn],
[b].[Title],
[a].[AuthorId],
[a].[Name],
[a].[WebUrl]
FROM [Books] AS [b]
INNER JOIN [Author] AS [a] ON
[b].[AuthorId] = [a].[AuthorId]
````



````c#
// The code to update the author’s WebUrl of the book Quantum Networking
public static void ChangeWubUrl()
{
    Console.WriteLine("New Quantum Networking WebUrl > ");
    var newWebUrl = Console.ReadLine();
    
    using(var db = new AppDbContext())
    {
        var singleBook = db.Books
            .Include(book => book.Author)
            .Single(book => book.Title == "Quantum Networking");
        
        singleBook.Author.WebUrl = newWebUrl;
        db.SaveChanges();
        Console.WriteLine("... SavedChanges called.");
    }
    
    ListAll();
}
````



![](./diagrams/images/01_02_ef_core_update_data.png)



1. The application uses a LINQ query to find a single book with its author information. EF Core turns the LINQ query into an SQL command to read the rows where the Title is Quantum Networking, returning an instance of both the `Book` and the `Author` classes, and checks that only one row was found.
2. The LINQ query doesn’t include the `.AsNoTracking` method you had in the previous read versions, so the query is considered to be a *tracked query*. Therefore, EF Core creates a tracking snapshot of the data loaded.
3. Then the code changes the `WebUrl` property in the Author class of the book. When `SaveChanges` is called, the Detect Changes stage compares all the classes that were returned from a tracked query with the tracking snapshot. From this, it can detect what has changed—in this case, only the `WebUrl` property of the `Author` class, which has a primary key of 3.
4. As a change is detected, EF Core starts a transaction. Every database update is done as an *atomic unit*: if multiple changes to the database occur, either they all succeed, or they all fail. This fact is important, because a relational database could get into a bad state if only part of an update were applied.
5. The update request is converted by the database provider to an SQL command that does the update. If the SQL command is successful, the transaction is committed, and the `SaveChanges` method returns; otherwise, an exception is raised.





## 2. Querying The Database

* Modeling three main types of database relationships
* Creating and changing a database via migration
* Defining and creating an application DbContext
* Loading related data
* Splitting complex queries into subqueries



### The Book App

* One-to-one relationship - PriceOffer to a Book
* One-to-many relationship - Book with Reviews
* Many-to-many relationship - Books linked to Authors and Books linked to Tags



#### One-to-one Relationship: Priceoffer to a book

A book can have a promotional price applied to it with an optional row in the Price- Offer, which is an example of a one-to-one relationship. (Technically, the relationship is one-to-zero-or-one, but EF Core handles it the same way.)

To calculate the final price of the book, you need to check for a row in the PriceOffer table that’s linked to the Books via a foreign key. If such a row is found, the NewPrice supersedes the price for the original book, and the PromotionalText is shown onscreen, as in this example:

​	*$40 $30 Our summertime price special, for this week only!*



![](./diagrams/svg/02_01_one_to_one_price_offer_to_book.drawio.svg)



#### One-to-many relationship: Reviews to a book

You want to allow customers to review a book; they can give a book a star rating and optionally leave a comment. Because a book may have no reviews or many (unlimited) reviews, you need to create a table to hold that data. In this example, you’ll call the table Review. The Books table has a one-to-many relationship to the Review table.

In the Summary display, you need to count the number of reviews and work out the average star rating to show a summary. Here’s a typical onscreen display you might produce from this one-to-many relationship:

​	*Votes 4.5 by 2 customers*



![](./diagrams/svg/02_02_one_to_many_reviews_to_book.drawio.svg)



#### Many-to-many relationship: Manually configured

Books can be written by one or more authors, and an author may write one or more books. The link between the Books and Authors tables is called a many-to-many relationship, which in this case needs a linking table to achieve this relationship.
In this case, you create your own linking table with an Order value in it because the names of the authors in a book must be displayed in a specific order.

A typical onscreen display from the many-to-many relationship would look like this:

​	*by Dino Esposito, Andrea Saltarello*



![](./diagrams/svg/02_03_many_to_many_authors_books.drawio.svg)



#### Many-to-many relationship: Autoconfigured by EF Core

Books can be tagged with different categories - such as Microsoft .NET, Linux, Web, and so on - to help the customer to find a book on the topic they are interested in. A category might be applied to multiple books, and a book might have one or more categories, so a many-to-many linking table is needed. But unlike in the previous `BookAuthor` linking table, the tags don’t have to be ordered, which makes the linking table simpler.

A typical onscreen display from a many-to-many relationship would look like this:

​	*Categories: Microsoft .NET, Web*



![](./diagrams/svg/02_04_many_to_many_tags_books.drawio.svg)



#### Other relationship types

* Owned Type class: Useful for adding grouped data, such as an `Address` class, to an entity class. The `Address` class is linked to the main entity, but your code can copy around the `Address` class rather than copying individual `Street`, `City`, `State`, and related properties.
* Table splitting: Maps multiple classes to one table. You could have a summary class with the basic properties in it and a detailed class containing all the data, for example, which would give you a quicker load of the summary data.
* Table per hierarchy (TPH): Useful for groups of data that are similar. If you have a lot of data with only a few differences, such as a list of animals, you can have a base `Animal` class that `Dog`, `Cat`, and `Snake` classes can inherit, with per-type properties such as `LengthOfTail` for `Dog` and `Cat` and a `Venomous` flag for the `Snake`. EF Core maps all the classes to one table, which can be more efficient.
* Table per type (TPT): Useful for groups of data that have dissimilar data. TPT is the opposite of TPH, in which each class has its own table. Following the `Animal` example for TPH, the TPT version would map the `Dog`, `Cat`, and `Snake` classes to three different tables in the database.



![](./diagrams/svg/02_05_book_app_all_tables.drawio.svg)



#### The classes that EF Core maps to the database

There created five .NET classes to map to the six tables in the database. These classes are called `Book`, `PriceOffer`, `Review`, `Tag`, `Author`, and `BookAuthor` for the many-to-manylinking table, and they are referred to as *entity classes* to show that they’re mapped by EF Core to the database. From the software point of view, there’s nothing special about entity classes. They’re normal .NET classes, sometimes referred to as plain old CLR objects (POCOs). The term entity class identifies the class as one that EF Core has mapped to the database.



````c#
// The Book class, mapped to the Books table in the database
public class Book // The Book class contains the main book information.
{
    public int BookId { get; set; } // We use EF Core’s By Convention configuration to define the primary key of this entity class, so we use <ClassName>Id, and because the property is of type int, EF Core assumes that the database will use the SQL IDENTITY command to create a unique key when a new row is added.
    public string Title { get; set; }
    public string Description { get; set; }
    public DateTime PublishedOn { get; set; }
    public string Publisher { get; set; }
    public decimal Price { get; set; }
    public string ImageUrl { get; set; }
    
    //-----------------------------------------------
    //relationships

    public PriceOffer Promotion { get; set; } // Link to the optional one-toone PriceOffer relationship.
    public ICollection<Review> Reviews { get; set; } // There can be zero to many reviews of the book.
    public ICollection<Tag> Tags { get; set; } // EF's automatic many-to-many relationship to the Tag entity class
    public ICollection<BookAuthor> AuthorsLink { get; set; } // Provides a link to the many-to-many linking table that links to the Authors of this book
}
````



* In the Book App, when we have navigational properties that are collections, we use the type `ICollection<T>`. We do so because the new eager loading sort capability can return a sorted collection, and the default `HashSet` definition says it holds only a collection "whose elements are in no particular order." But there is a performance cost to not using `HashSet` when your navigational properties contain a large collection.



### Creating the application’s DbContext

To access the database, you need to do the following:

1. Define your application’s DbContext, which you do by creating a class and inheriting from EF Core’s `DbContext` class.
2. Create an instance of that class every time you want to access the database.

The key class you need to use EF Core is the application’s `DbContext`. You define this class by inheriting EF Core’s `DbContext` class and adding various properties to allow your software to access the database tables. It also contains methods you can override to access other features in EF Core, such as configuring the database modeling.



````c#
public EfCoreContext : DbContext // Any application DbContext must inherit from the EF Core’s DbContext class.
{
    // These public properties of type DbSet<T> are mapped by EF Core to tables in your database, using the name of the property as the table name. You can query these tables via LINQ methods on a property.
    public DbSet<Book> Books { get; set; }
    public DbSet<Author> Authors { get; set; }
    public DbSet<Tag> Tags { get; set; }
    public DbSet<PriceOffer> PriceOffers { get; set; }
    // The classes, such as Book, Author, Tag and PriceOffer, are entity classes. Their properties are mapped to columns in the appropriate database table.
    
    
    public EfCoreContext(DbContextOptions<EfCoreContext> options) : base(options) // For your ASP.NET Core application, you need a constructor to set up the database options. This allows your application to define what sort of database it is and where it’s located.
    {
    }
    
    protected override void OnModelCreating(DbModelBuilder modelBuilder)
    {
        //... code left out
    }
}

// The application’s DbContext is the key class in accessing the database. This figure shows the main parts of an application’s DbContext, starting with its inheriting EF Core’s DbContext, which brings in lots of code and features. You have to add some properties with the class DbSet<T> that map your classes to a database table with the same name as the property name you use. The other parts are the constructor, which handles setting up the database options, and the OnModelCreating method, which you can override to add your own configuration commands and set up the database the way you want.
````

````c#
// Creating an instance of the application’s DbContext to access the database
const string connection = "Data Source=(localdb)\\mssqllocaldb;Database=EfCoreInActionDb.Chapter02;Integrated Security=True;"; // The connection string, with its format dictated by the sort of database provider and hosting you’re using
var optionsBuilder = new DbContextOptionsBuilder<EfCoreContext>(); // You need an EF Core DbContextOptionsBuilder<> instance to set the options you need.

optionsBuilder.UseSqlServer(connection); // You’re accessing an SQL Server database and using the UseSqlServer method from the Microsoft.EntityFramework- Core.SqlServer library, and this method needs the database connection string.
var options = optionsBuilder.Options;
using (var context = new EfCoreContext(options)) // Creates the all-important EfCoreContext, using the options you’ve set up. You use a using statement because the DbContext is disposable.
{
    var bookCount = context.Books.Count(); // Uses the DbContext to find out the number of books in the database
    //... etc.
}
````



At the end of this listing, you create an instance of `EfCoreContext` inside a using statement because DbContext has an IDisposable interface and therefore should be disposed after you’ve used it.



### Understanding database queries

````c#
context.Books									// Application’s DbContext property access
    .Where(p => p.Title.StartsWith("Quantum")	// A series of LINQ and/or EF Core commands
    .ToList();									// An execute command
// The three parts of an EF Core database query, with example code.
````

Until a final execute command is applied at the end of the sequence of LINQ commands, the LINQ is held as a series of commands in what is called an *expression tree*, which means that it hasn’t been executed on the data yet.

At this point, your LINQ query will be converted to database commands and sent to the database. If you want to build high-performance database queries, you want all your LINQ commands for filtering, sorting, paging, and so on to come before you call an execute command. Therefore, your filter, sort, and other LINQ commands will be run inside the database, which improves the performance of your query.



````c#
context.Books.AsNoTracking()
    .Where(p => p.Title.StartsWith("Quantum")).ToList();
````

This query has the EF Core’s `AsNoTracking` method added to the LINQ query. As well as making the query read-only, the `AsNoTracking` method improves the performance of the query by turning off certain EF Core features.



### Loading related data

You can load data in four ways: eager loading, explicit loading, select loading, and lazy loading. 

You need to be aware that EF Core won’t load any relationships in an entity class unless you ask it to. This default behavior of not loading relationships is correct, because it means that EF Core minimizes the database accesses. If you want to load a relationship, you need to add code to tell EF Core to do that.



#### Eager loading: Loading relationships with the primary entity class

The first approach to loading related data is *eager loading*, which entails telling EF Core to load the relationship in the same query that loads the primary entity class. Eager loading is specified via two fluent methods, `Include` and `ThenInclude`. The next listing shows the loading of the first row of the Books table as an instance of the `Book` entity class and the eager loading of the single relationship, `Reviews`.



````c#
// Eager loading of first book with the corresponding Reviews relationship
var firstBook = context.Books
    .Include(book => book.Reviews) // Gets a collection of Review class instances, which may be an empty collection
    .FirstOrDefault(); // Takes the first book or null if there are no books in the database
````

If you look at the SQL command that this EF Core query creates, shown in the following snippet, you’ll see two SQL commands. The first command loads the first row in the Books table. The second loads the reviews, where the foreign key, `BookId`, has the same value as the first `Books` row primary key:

````sql
SELECT "t"."BookId", "t"."Description", "t"."ImageUrl",
	"t"."Price", "t"."PublishedOn", "t"."Publisher",
	"t"."Title", "r"."ReviewId", "r"."BookId",
	"r"."Comment", "r"."NumStars", "r"."VoterName"
FROM (
    SELECT "b"."BookId", "b"."Description", "b"."ImageUrl",
	    "b"."Price", "b"."PublishedOn", "b"."Publisher", "b"."Title"
    FROM "Books" AS "b"
    LIMIT 1
) AS "t"
LEFT JOIN "Review" AS "r" ON "t"."BookId" = "r"."BookId"
ORDER BY "t"."BookId", "r"."ReviewId"
````



````c#
// Eager loading of the Book class and all the related data
var firstBook = context.Books
    .Include(book => book.AuthorsLink) // The first Include gets a collection of BookAuthor.
    	.ThenInclude(bookAuthor => bookAuthor.Author) // Gets the next link—in this case, the link to the author
    .Include(book => book.Reviews) // Gets a collection of Review class instances, which may be an empty collection
    .Include(book => book.Tags) // Loads and directly accesses the Tags
    .Include(book => book.Promotion) // Loads any optional PriceOffer class, if one is assigned
    .FirstOrDefault(); // Loads any optional PriceOffer class, if one is assigned
````

`Include` followed by `ThenInclude`, is the standard way of accessing relationships that go deeper than a first-level relationship. You can go to any depth with multiple `ThenIncludes`, one after the other.

If you use the direct linking of many-to-many relationships, you don’t need `ThenInclude` to load the second-level relationship because the property directly accesses the other end of the many-to-many relationship via the `Tags` property, which is of type `ICollection<Tag>`. This approach can simplify the use of a many-to-many relationship as long you don’t need some data in the linking table, such as the `Order` property in the `BookAuthor` linking entity class used to order the `Book`’s `Authors` correctly.

If the relationship doesn’t exist (such as the optional `PriceOffer` class pointed to by the Promotion property in the `Book` class), `Include` doesn’t fail; it simply doesn’t load anything, or in the case of collections, it returns an empty collection (a valid collection with zero entries). The same rule applies to `ThenInclude`: if the previous `Include` or `ThenInclude` was empty, subsequent `ThenIncludes` are ignored. If you don’t `Include` a collection, it is null by default.

The advantage of eager loading is that EF Core will load all the data referred to by the `Include` and `ThenInclude` in an efficient manner, using a minimum of database accesses, or *database round-trips*. This type of loading to be useful in relational updates in which I need to update an existing relationship. Also eager loading to be useful in business logic.
The downside is that eager loading loads *all* the data, even when you don’t need part of it. The book list display, for example, doesn’t need the book description, which could be quite large.



You can use the ability to sort or filter the related entities when you use the `Include` or `ThenInclude` methods. This capability is helpful if you want to load only a subset of the related data (such as only `Reviews` with five stars) and/or to order the included entities (such as ordering the `AuthorsLink` collection against the Order property). The only LINQ commands you can use in the `Include` or `ThenInclude` methods are `Where`, `OrderBy`, `OrderByDescending`, `ThenBy`, `ThenByDescending`, `Skip`, and `Take`, but those commands are all you need for sorting and filtering.

````c#
// Sorting and filtering when using Include or ThenInclude
var firstBook = context.Books
    .Include(book => book.AuthorsLink					// Sort example: On the eager loading of the AuthorsLink collection, you sort the
    	.OrderBy(bookAuthor => bookAuthor.Order))		// BookAuthors so that the Authors will be in the correct order to display.
    	.ThenInclude(bookAuthor => bookAuthor.Author)
    .Include(book => book.Reviews						// Filter example. Here, you load only
        .Where(review => review.NumStars == 5))			// the Reviews with a star rating of 5.
    .Include(book => book.Promotion)
    .First();
````



#### Explicit loading: Loading relationships after the primary entity class

The second approach to loading data is *explicit loading*. After you’ve loaded the primary entity class, you can explicitly load any other relationships you want. First, it loads the `Book`; then it uses explicit-loading commands to read all the relationships.

````c#
// Explicit loading of the Book class and related data
var firstBook = context.Books.First(); // Reads in the first book on its own
context.Entry(firstBook)
    .Collection(book => book.AuthorsLink).Load(); // Explicity loads the linking table, BookAuthor
foreach (var authorLink in firstBook.AuthorsLink) // To load all the possible authors, the code has to loop through all the BookAuthor entries...
{
    context.Entry(authorLink) // …and load each linked Author class.
        .Reference(bookAuthor =>
        	bookAuthor.Author).Load();
}

context.Entry(firstBook) // Loads all the reviews
    .Collection(book => book.Tags).Load(); // Loads the Tags
context.Entry(firstBook) // Loads the optional PriceOffer class
    .Reference(book => book.Promotion).Load();
````



Alternatively, explicit loading can be used to apply a query to the relationship instead of loading the relationship. You can use any standard LINQ command after the `Query` method, such as `Where` or `OrderBy`.

````c#
// Explicit loading of the Book class with a refined set of related data
var firstBook = context.Books.First(); // Reads in the first book on its own
var numReviews = context.Entry(firstBook) // Executes a query to count reviews for this book
    .Collection(book => book.Reviews)
    .Query().Count();
var starRatings = context.Entry(firstBook) // Executes a query to get all the star ratings for the book
    .Collection(book => book.Reviews)
    .Query().Select(review => review.NumStars)
    .ToList();
````



The advantage of explicit loading is that you can load a relationship of an entity class later. This technique useful when I’m using a library that loads only the primary entity class, and need one of its relationships. Explicit loading can also be useful when you need that related data in only some circumstances. You might also find explicit loading to be useful in complex business logic because you can leave the job of loading the specific relationships to the parts of the business logic that need it. The downside of explicit loading is more database round trips, which can be inefficient. If you know up front the data you need, eager loading the data is usually more efficient because it takes fewer database round trips to load the relationships.



#### Select loading: Loading specific parts of primary entity class and any relationships

The third approach to loading data is using the LINQ `Select` method to pick out the data you want, which calls *select loading*. The next listing shows the use of the `Select` method to select a few standard properties from the `Book` class and execute specific code inside the query to get the count of customer reviews for this book.



````c#
// Select of the Book class picking specific properties and one calculation
var books = context.Books
    .Select(book => new // Uses the LINQ Select keyword and creates an anonymous type to hold the results
    	{
            book.Title,
            book.Price,
            NumReviews = book.Reviews.Count, // Runs a query that counts the number of reviews
        }
).ToList();
````

The advantage of this approach is that only the data you need is loaded, which can be more efficient if you don’t need all the data. Only one SQL `SELECT` command is required to get all that data, which is also efficient in terms of database round trips. EF Core turns the `p.Reviews.Count` part of the query into an SQL command, so that count is done inside the database, as you can see in the following snippet of the SQL created by EF Core:

````sql
SELECT "b"."Title", "b"."Price", (
    SELECT COUNT(*)
    FROM "Review" AS "r"
    WHERE "b"."BookId" = "r"."BookId") AS "NumReviews"
FROM "Books" AS "b"
````



#### Lazy loading: Loading relationships as required

*Lazy loading* makes writing queries easy, but it has a bad effect on database performance. Lazy loading does require some changes to your DbContext or your entity classes, but after you make those changes, reading is easy; if you access a navigational property that isn’t loaded, EF Core will execute a database query to load that navigational property.

You can set up lazy loading in either of two ways:

* Adding the Microsoft.EntityFrameworkCore.Proxies library when configuring your DbContext
* Injecting a lazy loading method into the entity class via its constructor

The first option is simple but locks you into setting up lazy loading for all the relationships. The second option requires you to write more code but allows you to pick which relationships use lazy loading.

To configure the simple lazy loading approach, you must do two things:

* Add the keyword `virtual` before *every* property that is a relationship.
* Add the method `UseLazyLoadingProxies` when setting up your DbContext.

So the converted Book entity type to the simple lazy loading approach would look like the following code snippet, with the virtual keyword added to the navigational properties:

````c#
public class BookLazy
{
    public int BookLazyId { get; set; }
    //… Other properties left out for clarity

    public virtual PriceOffer Promotion { get; set; }
    public virtual ICollection<Review> Reviews { get; set; }
    public virtual ICollection<BookAuthor> AuthorsLink { get; set; }
}
````

Using the EF Core’s Proxy library has a limitation: you must make every relational property virtual; otherwise, EF Core will throw an exception when you use the DbContext.



The second part is adding the EF Core’s Proxy library to the application that sets up the DbContext and then adding the `UseLazyLoadingProxies` to the configuring of the DbContext.

````c#
var optionsBuilder =
    new DbContextOptionsBuilder<EfCoreContext>();
optionsBuilder
    .UseLazyLoadingProxies()
    .UseSqlServer(connection);
var options = optionsBuilder.Options;

using (var context = new EfCoreContext(options))
````

When you have configured lazy loading in your entity classes and in the way you create the DbContext, reading relationships is simple; you don’t need extra `Include` methods in your query because the data is loaded from the database when your code accesses that relationship property.

````c#
// Lazy loading of BookLazy’s Reviews navigational property
var book = context.BookLazy.Single(); // Gets an instance of the BookLazy entity class that has configured its Reviews property to use lazy loading
var reviews = book.Reviews.ToList(); // When the Reviews property is accessed, EF Core will read in the reviews from the database.
````

Creates two database accesses. The first access loads the `BookLazy` data without any properties, and the second happens when you access `BookLazy`’s `Reviews` property.



Many developers find lazy loading to be useful, but you can want to avoid it because of its performance issues. There is time overhead for every access to the database server, so the best approach is to minimize the number of calls to the database server. But lazy loading (and explicit loading) can create lots of database accesses, making the query slow and causing the database server to work harder.

Even if you have set up a relational property for lazy loading, you can get better performance by adding an `Include` on a virtual relational property. The lazy loading will see that the property has been loaded and not load it again.



### Using client vs. server evaluation: Adapting data at the last stage of a query

EF Core has a feature called *client vs. server evaluation*, which allows you to run code at the last stage of the query (that is, the final `Select` part in your query) that can’t be converted to database commands. EF Core runs these non-server-runnable commands after the data has come back from the database.

The client vs. server evaluation feature gives you the opportunity to adapt/change the data within the last part of the query, which can save you from having to apply an extra step after the query. You use client vs. server evaluation to create a comma-delimited list of the authors of a book. If you didn’t use client vs. server evaluation for that task, you would need to (a) send back a list of all the Author names and (b) add an extra step after the query, using a `foreach` section to apply a `string.Join` to each book’s authors.



If your LINQ queries can’t be converted to database commands, EF Core will throw an `InvalidOperationException`, with a message containing the words `could not be translated`. The trouble is that you get that error only when you try that query - and you don’t want that error to happen in production!



For the list display of the books in the Book App, you need to (a) extract all the authors’ names, in order, from the Authors table and (b) turn them into one string with commas between names. Here’s an example that loads two properties, `BookId` and `Title`, in the normal manner, and a third property, `AuthorsString`, that uses client vs. server evaluation.

````c#
// Select query that includes a non-SQL command, string.Join
var firstBook = context.Books
    .Select(book => new
    {
        book.BookId,
        book.Title,
        AuthorsString = string.Join(", ", // string.Join is executed on the client in software.
        	book.AuthorsLink
            .OrderBy(ba => ba.Order)
            .Select(ba => ba.Author.Name))
    }
).First();
````



![](./diagrams/images/02_01_client_vs_server_evaluation.png)



Using client vs. server evaluation on a property means that you cannot use that property in any LINQ command that would produce database commands, such as any commands that sort or filter that property. If you do, you will get an `InvalidOperationException`, with a message that contains the words `could not be translated`. If you tried to sort or filter on the `AuthorsString`, you would get the `could not be translated` exception.



### Building complex queries

You’re going to build a query to list all the books in the Book App, with a range of features including sorting, filtering, and paging.
You could build the book display by using eager loading. First, you’d load all the data; then, in the code, you’d combine the authors, calculate the price, calculate the average votes, and so on. The problem with that approach is that (a) you are loading data you don’t need and (b) sorting and filtering have to be done in software. For this chapter’s Book App, which has approximately 50 books, you could eager-load all the books and relationships into memory and then sort or filter them in software, but that approach wouldn’t work for real!

This figure is complicated because the queries needed to get all the data are complicated. With this diagram in mind, let’s look at how to build the book select query. You start with the class you’re going to put the data in. This type of class, which exists only to bring together the exact data you want, is referred to in various ways. In ASP.NET, it is referred to as a ViewModel, but that term also has other connotations and uses; therefore, we refer to this type of class as a *Data Transfer Object (DTO)*. DTOs is "object that is used to encapsulate data, and send it from one subsystem of an application to another".



````c#
// The DTO BookListDto
public class BookListDto
{
    public int BookId { get; set; } // You need the primary key if the customer clicks the entry to buy the book.
    public string Title { get; set; }
    public DateTime PublishedOn { get; set; } // Although the publication date isn’t shown, you’ll want to sort by it, so you have to include it.
    public decimal Price { get; set; } // The normal selling price of the book
    public decimal ActualPrice { get; set; } // Selling price—either the normal price or the promotional.NewPrice if present
    public string PromotionPromotionalText { get; set; } // Promotional text to show whether there’s a new price
    public string AuthorsOrdered { get; set; } // String to hold the comma-delimited list of authors’ names
    public int ReviewsCount { get; set; } // Number of people who reviewed the book
    public double? ReviewsAverageVotes { get; set; } // Average of all the votes, null if no votes
    public string[] TagStrings { get; set; } // The Tag names (that is the categories) for this book
}
````

To work with EF Core’s select loading, the class that’s going to receive the data must have a default constructor (which you can create without providing any properties to the constructor), the class must not be static, and the properties must have public setters.

Next, you’ll build a select query that fills in every property in `BookListDto`. Because you want to use this query with other query parts, such as sort, filter, and paging, you’ll use the `IQueryable<T>` type to create a method called `MapBookToDto` that takes in `IQueryable<Book>` and returns `IQueryable<BookListDto>`.

````c#
// The Select query to fill BookListDto
public static IQueryable<BookListDto> MapBookToDto(this IQueryable<Book> books) // Takes in IQueryable<Book> and returns IQueryable<BookListDto>
{
    return books.Select(book => new BookListDto
    {
        BookId = book.BookId,
        Title = book.Title,
        Price = book.Price,
        PublishedOn = book.PublishedOn,
        ActualPrice = book.Promotion == null // Calculates the selling price, which is the normal price, or the promotion price if that relationship exists
            ? book.Price
            : book.Promotion.NewPrice,
        PromotionPromotionalText = book.Promotion == null // PromotionalText depends on whether a PriceOffer exists for this book
            ? null
            : book.Promotion.PromotionalText,
        AuthorsOrdered = string.Join(", ", // Obtains an array of authors’ names, in the right order. You’re using client vs. server evaluation because you want the author names combined into one string.
        	book.AuthorsLink
            	.OrderBy(ba => ba.Order)
                .Select(ba => ba.Author.Name)),
        ReviewsCount = book.Reviews.Count, // You need to calculate the number of reviews.
        ReviewsAverageVotes = book.Reviews.Select(review => // To get EF Core to turn the LINQ average into the SQL AVG command, you need to cast the NumStars to (double?).
        	(double?) review.NumStars).Average(),
        TagStrings = book.Tags // Array of Tag names (categories) for this book
            .Select(x => x.TagId).ToArray(),
    });
}
````

The `MapBookToDto` method uses the Query Object pattern; the method takes in `IQueryable<T>` and outputs `IQueryable<T>`, which allows you to encapsulate a query, or part of a query, in a method. That way, the query is isolated in one place, which makes it easier to find, debug, and performance-tune. You’ll use the Query Object pattern for the sort, filter, and paging parts of the query too.



Query Objects are useful for building queries such as the book list in this example, but alternative approaches exist, such as the Repository pattern.



The `MapBookToDto` method is also what .NET calls an *extension method*. Extension methods allow you to chain Query Objects together.



A method can become an extension method if (a) it’s declared in a static class, (b) the method is static, and (c) the first parameter has the keyword `this` in front of it.



Query Objects take in a `IQueryable<T1>` input and return `IQueryable<T2>`, so you’re adding LINQ commands to the original `IQueryable<T1>` input. You can add another Query Object to the end, or if you want to execute the query, add an execute command such as ToList to execute the query.



### Adding sorting, filtering, and paging

#### Sorting books by price, publication date, and customer ratings

Sorting in LINQ is done by the methods `OrderBy` and `OrderByDescending`. You create a Query Object called `OrderBooksBy` as an extension method, as shown in the next listing. You’ll see that in addition to the `IQueryable<BookListDto>` parameter, this method takes in an enum parameter that defines the type of sort the user wants.

````c#
// The OrderBooksBy Query Object method
public static IQueryable<BookListDto> OrderBooksBy (this IQueryable<BookListDto> books, OrderByOptions orderByOptions)
{
    switch (orderByOptions)
    {
        case OrderByOptions.SimpleOrder:
            return books.OrderByDescending(x => x.BookId); // Because of paging, you always need to sort. You default-sort on the primary key, which is fast.
        case OrderByOptions.ByVotes:
            return books.OrderByDescending(x => x.ReviewsAverageVotes); // Orders the book by votes. Books without any votes (null return) go at the bottom.
        case OrderByOptions.ByPublicationDate:
            return books.OrderByDescending(x => x.PublishedOn); // Orders by publication date, with the latest books at the top
        case OrderByOptions.ByPriceLowestFirst:					// Orders by actual price,
            return books.OrderBy(x => x.ActualPrice);			// which takes into account
        case OrderByOptions.ByPriceHighestFirst:				// any promotional price—both
            return books.OrderByDescending(x => x.ActualPrice);	// lowest first and highest first
        default:
            throw new ArgumentOutOfRangeException(nameof(orderByOptions), orderByOptions, null);
    }
}
````

Calling the `OrderBooksBy` method returns the original query with the appropriate LINQ sort command added to the end. You pass this query on to the next Query Object, or if you’ve finished, you call a command to execute the code, such as `ToList`.

Even if the user doesn’t select a sort, you’ll still sort because you’ll be using paging, providing only a page at a time rather than all the data, and SQL requires the data to be sorted to handle paging. The most efficient sort is on the primary key, so you sort on that key.



#### Filtering books by publication year, categories, and customer ratings

````c#
// The code to produce a list of the years when books are published
var result = _db.Books
    .Where(x => x.PublishedOn <= DateTime.UtcNow.Date)
    .Select(x => x.PublishedOn.Year)
    .Distinct() // The Distinct method returns a list of each year a book was published.
    .OrderByDescending(x => x.PublishedOn) // Orders the published. years, with newest year at the top
    .Select(x => new DropdownTuple // Use two client/server evaluations to turn the values into strings.
    {
        Value = x.ToString(),
        Text = x.ToString()
    }).ToList();
var comingSoon = _db.Books.
    Any(x => x.PublishedOn > DateTime.Today); // Returns true if a book in the list is not yet published
if (comingSoon) // Adds a "coming soon" filter for all the future books
{
    result.Insert(0, new DropdownTuple
    {
        Value = BookListDtoFilter.AllBooksNotPublishedString,
        Text = BookListDtoFilter.AllBooksNotPublishedString
    });
}

return result;
````

The result of this code is a list of `Value/Text` pairs holding each year that books are published, plus a Coming Soon section for books yet to be published.



````c#
// The FilterBooksBy Query Object method
public static IQueryable<BookListDto> FilterBooksBy(this IQueryable<BookListDto> books, BooksFilterBy filterBy, string filterValue) // The method is given both the type of filter and the user-selected filter value.
{
    if (string.IsNullOrEmpty(filterValue)) // If the filter value isn’t set, returns IQueryable with no change
    {
        return books;
    }
    
    switch (filterBy)
    {
        case BooksFilterBy.NoFilter: // For no filter selected, returns IQueryable with no change
            return books;
        case BooksFilterBy.ByVotes:
            var filterVote = int.Parse(filterValue); // The filter by votes returns only books with an average vote above the filterVote value. If there are no reviews for a book, the ReviewsAverageVotes property will be null, and the test always returns false.
            return books.Where(x => x.ReviewsAverageVotes > filterVote);
        case BooksFilterBy.ByTags: // Selects any books with a Tag category that matches the filterValue
            return books.Where(x => x.TagStrings.Any(y => y == filterValue));
        case BooksFilterBy.ByPublicationYear:
            if (filterValue == AllBooksNotPublishedString) // If Coming Soon was picked, returns only books not yet published
            {
                return books.Where(x => x.PublishedOn > DateTime.UtcNow);
            }
            var filterYear = int.Parse(filterValue); // If we have a specific year, we filter on that. Note that we also remove future books (in case the user chose this year’s date).
            return books.Where(x => x.PublishedOn.Year == filterYear&& x.PublishedOn <= DateTime.UtcNow);
        default:
            throw new ArgumentOutOfRangeException(nameof(filterBy), filterBy, null);
    }
}
````



#### Other filtering options: Searching text for a specific string

We could’ve created loads of other types of filters/searches of books, and searching by title is an obvious one. But you want to make sure that the LINQ commands you use to search a string are executed in the database, because they’ll perform much better than loading all the data and filtering in software. EF Core converts the following C# code in a LINQ query to a database command: `==`, `Equal`, `StartsWith`, `EndsWith`, `Contains`, and `IndexOf`.

| String command | Example                                                      |
| -------------- | ------------------------------------------------------------ |
| StartsWith     | var books = context.Books.Where(p => p.Title.StartsWith("The")).ToList(); |
| EndsWith       | var books = context.Books.Where(p => p.Title.EndsWith("MAT.")).ToList(); |
| Contains       | var books = context.Books.Where(p => p.Title.Contains("cat")) |

The other important thing to know is that the case sensitivity of a string search executed by SQL commands depends on the type of database, and in some databases, the rule is called `collation`. A default SQL Server database default collation uses case-insensitive searches, so searching for `Cat` would find `cat` and `Cat`. Many SQL databases are `case-insensitive` by default, but Sqlite has a mix of `case-sensitive`/`case-insensitive`, and Cosmos DB is by default `case-sensitive`.



Typically, you configure the collation for the database or a specific column, but you can also define the collation in a query by using the `EF.Functions.Collate` method. The following code snippet sets an SQL Server collation, which means that this query will compare the string using the `Latin1_General_CS_AS` (`case-sensitive`) collation for this query:

````c#
context.Books.Where( x =>
	EF.Functions.Collate(x.Title, "Latin1_General_CS_AS") == "HELP" //This does not match "help"
````



Another string command is the SQL command `LIKE`, which you can access through the `EF.Function.Like` method. This command provides a simple pattern-matching approach using _ (underscore) to match any letter and % to match zero-to-many characters. The following code snippet would match "The Cat sat on the mat." and "The dog sat on the step." but not "The rabbit sat on the hutch." because rabbit isn’t three letters long:

````c#
var books = context.Books
    .Where(p => EF.Functions.Like(p.Title, "The ___ sat on the %."))
    .ToList();
````



#### Paging the books in the list

````c#
// A generic Page Query Object method
public static IQueryable<T> Page<T>(this IQueryable<T> query, int pageNumZeroStart, int pageSize)
{
    if (pageSize == 0)
    {
        throw new ArgumentOutOfRangeException(nameof(pageSize), "pageSize cannot be zero.");
    }

    if (pageNumZeroStart != 0)
    {
        query = query.Skip(pageNumZeroStart * pageSize); // Skips the correct number of pages
    }

    return query.Take(pageSize); // Takes the number for this page size
}
````



Paging works only if the data is ordered. Otherwise, SQL Server will throw an exception because relational databases don’t guarantee the order in which data is handed back; there’s no default row order in a relational database.



### Putting it all together: Combining Query Objects

The benefit of building a complex query in separate parts is that this approach makes writing and testing the overall query simpler, because you can test each part on its own.



````c#
// The ListBookService class providing a sorted, filtered, and paged list
public class ListBooksService
{
    private readonly EfCoreContext _context;

    public ListBooksService(EfCoreContext context)
    {
        _context = context;
    }

    public IQueryable<BookListDto> SortFilterPage(SortFilterPageOptions options)
    {
        var booksQuery = _context.Books // Starts by selecting the Books property in the Application’s DbContext
            .AsNoTracking() // Because this query is readonly, you add .AsNoTracking.
            .MapBookToDto() // Uses the Select Query Object, which picks out/calculates the data it needs
            .OrderBooksBy(options.OrderByOptions) // Adds the commands to order the data by using the given options
            .FilterBooksBy(options.FilterBy, options.FilterValue); // Adds the commands to filter the data

        options.SetupRestOfDto(booksQuery); // This stage sets up the number of pages and makes sure that PageNum is in the right range.

        return booksQuery.Page(options.PageNum-1, options.PageSize); // Applies the paging commands
    }
}
````



### Summary

* To access a database in any way via EF Core, you need to define an application DbContext.
* An EF Core query consists of three parts: the application’s DbContext property, a series of LINQ/EF Core commands, and a command to execute the query.
* Using EF Core, you can model three primary database relationships: `one-to-one`, `one-to-many`, and `many-to-many`, and others.
* The classes that EF Core maps to the database are referred to as *entity classes*. We use this term to highlight the fact that the class I’m referring to is mapped by EF Core to the database.
* If you load an entity class, it won’t load any of its relationships by default. Querying the `Book` entity class, for example, won’t load its relationship properties (`Reviews`, `AuthorsLink`, and `Promotion`); it leaves them as `null`.
* You can load related data that’s attached to an entity class in four ways: eager loading, explicit loading, select loading, and lazy loading.
* EF Core’s client vs. server evaluation feature allows the last stage of a query to contain commands, such as `string.Join`, that can’t be converted to SQL commands.
* We use the term *Query Object* to refer to an encapsulated query or a section of a query. These Query Objects are often built as .NET extension methods, which means that they can easily be chained together, similar to the way LINQ is written.
* Selecting, sorting, filtering, and paging are common query uses that can be encapsulated in a Query Object.
* If you write your LINQ queries carefully, you can move the aggregate calculations, such as `Count`, `Sum`, and `Average`, into the relational database, improving performance.





## 3. Changing The Database Content

* Creating a new row in a database table
* Updating existing rows in a database table for two types of applications
* Updating entities with one-to-one, one-to-many, and many-to-many relationships
* Deleting single entities, and entities with relationships, from a database



### EF Core’s entity State

Any entity class instance has a `State`, which can be accessed via the following EF Core command: `context.Entry(someEntityInstance).State`. The State tells EF Core what to do with this instance when `SaveChanges` is called. Here’s a list of the possible states and what happens if `SaveChanges` is called:

* `Added` - The entity needs to be created in the database. `SaveChanges` inserts it.
* `Unchanged` - The entity exists in the database and hasn’t been modified on the client. `SaveChanges` ignores it.
* `Modified` - The entity exists in the database and has been modified on the client. `SaveChanges` updates it.
* `Deleted` - The entity exists in the database but should be deleted. `SaveChanges` deletes it.
* `Detached` - The entity you provided isn’t tracked. `SaveChanges` doesn’t see it.

Normally, you don’t look at or alter the `State` directly. You use the various commands listed in this chapter to add, update, or delete entities. These commands make sure the `State` is set in a *tracked entity*. When `SaveChanges` is called, it looks at all the tracked entities and their `State` to decide what type of database changes it needs to apply to the database.

`Tracked` entities are entity instances that have been read in from the database by using a query that didn’t include the `AsNoTracking` method. Alternatively, after an entity instance has been used as a parameter to EF Core methods (such as `Add`, `Update`, or `Delete`), it becomes tracked.



### Creating new rows in a table

````c#
// An example of creating a single entity
var itemToAdd = new ExampleEntity
{
    MyMessage = "Hello World"
};
context.Add(itemToAdd); // Uses the Add method to add SingleEntity to the application’s DbContext. The DbContext determines the table to add it to, based on its parameter type.
context.SaveChanges(); // Calls the SaveChanges method from the application’s DbContext to update the database
````

Because you add the entity instance `itemToAdd` that wasn’t originally tracked, EF Core starts to track it and sets its `State` to Added. After `SaveChanges` is called, EF Core finds a tracked entity of type `ExampleEntity` with a `State` of `Added`, so it’s added as a new row in the database table associated with the `ExampleEntity` class.

EF Core creates the SQL command to update an SQL Server–based database.

````sql
-- SQL commands created to insert a new row into the SingleEntities table
SET NOCOUNT ON;
-- Inserts (creates) a new row into the ExampleEntities table
INSERT INTO ExampleEntities]
	([MyMessage]) VALUES (@p0);

-- Reads back the primary key in the newly created row
SELECT [ExampleEntityId] 
FROM [ExampleEntities]
WHERE @@ROWCOUNT = 1 AND
	[ExampleEntityId] = scope_identity();
````

The second SQL command produced by EF Core reads back the primary key of the row that was created by the database server. This command ensures that the original `ExampleEntity` instance is updated with the primary key so that the in-memory version of the entity is the same as the version in the database. Reading back the primary key is important, as you might update the entity later, and the update will need the primary key.



````c#
// Adding a Book entity class also adds any linked entity classes
var book = new Book
{
    Title = "Test Book",
    PublishedOn = DateTime.Today,
    Reviews = new List<Review>() // Creates a new collection of reviews
    {
        new Review // Adds one review with its content
        {
            NumStars = 5,
            Comment = "Great test book!",
            VoterName = "Mr U Test"
        }
    }
};

context.Add(book); // Uses the Add method to add the book to the application’s DbContext property, Books
context.SaveChanges(); // Calls the SaveChanges method from the application’s DbContext to update the database. It finds a new Book, which has a collection containing one new Review, and then adds both to the database.
````



![](./diagrams/svg/03_01_adding_entity_with_linked_entity.drawio.svg)



**WHAT HAPPENS AFTER THE SAVECHANGES RETURNS SUCCESSFULLY?**
When the `Add` and `SaveChanges` have finished successfully, a few things happen: the entity instances that have been inserted into the database are now tracked by EF Core, and their `State` is set to `Unchanged`. Because we are using a relational database, and because the two entity classes, `Book` and `Review`, have primary keys that are of type `int`, EF Core by default will expect the database to create the primary keys by using the SQL `IDENTITY` keyword. Therefore, the SQL commands created by EF Core read back the primary keys into the appropriate primary keys in the entity class instances to make sure that the entity classes match the database.

EF Core can detect any subsequent changes you make to the primary or foreign keys if you call `SaveChanges` again.



**Why you should call SaveChanges only once at the end of your changes**
You see that the `SaveChanges` method is called at the end of create, and you see the same pattern - the `SaveChanges` method is called at the end in the update and delete examples too. In fact, even for complex database change containing a mixture of creates, updates, and deletes, you should still call the `SaveChanges` method only once at the end. You do that because EF Core will save all your changes (creates, updates and deletes) and apply them to the database together, and if the database rejects any of your changes, all your changes are rejected (by means of a database feature called a transaction).

This pattern is called a `Unit Of Work` and means that your database changes can’t be half-applied to the database. If you created a new `Book` with a `BookAuthor` reference to an `Author` that wasn’t in the database, for example, you wouldn’t want the `Book` instance to be saved. Saving it might break the book display, which expects every `Book` to have at least one `Author`.



**EXAMPLE THAT HAS ONE INSTANCE ALREADY IN THE DATABASE**

````c#
// Adding a Book with an existing Author
var foundAuthor = context.Authors
    .SingleOrDefault(author => author.Name == "Mr. A"); // Reads in the Author with a check that the Author was found
if (foundAuthor == null)
{
    throw new Exception("Author not found");
}

var book = new Book
{
    Title = "Test Book",
    PublishedOn = DateTime.Today
};
book.AuthorsLink = new List<BookAuthor> // Adds an AuthorBook linking entry, but uses the Author that is already in the database
{
    new BookAuthor
    {
        Book = book,
        Author = foundAuthor
    }
};

context.Add(book); // Adds the new Book to the DbContext Books property and calls SaveChanges
context.SaveChanges();
````



The first four lines load an `Author` entity with some checks to make sure that it was found; this `Author` class instance is tracked, so EF Core knows that it is already in the database. You create a new `Book` entity and add a new `BookAuthor` linking entity, but instead of creating a new `Author` entity instance, you use the `Author` entity that you read in from the database. Because EF Core is tracking the `Author` instance and knows that it’s in the database, EF Core won’t try to add it again to the database when `SaveChanges` is called at the end.



### Updating database rows

Updating a database row is achieved in three stages:

1. Read the data (database row), possibly with some relationships.
2. Change one or more properties (database columns).
3. Write the changes back to the database (update the row).



````c#
// Updating Quantum Networking’s publication date
var book = context.Books
    .SingleOrDefault(p => p.Title == "Quantum Networking");
if (book == null)
{
throw new Exception("Book not found");
}

book.PublishedOn = new DateTime(2029, 1, 1);
context.SaveChanges(); // Calls SaveChanges, which includes running a method called DetectChanges. This method spots that the PublishedOn property has been changed.
````

When the `SaveChanges` method is called, it runs a method called `DetectChanges`, which compares the tracking snapshot against the entity class instance that it handed to the application when the query was originally executed. From this example, EF Core decides that only the `PublishedOn` property has been changed, and EF Core builds the SQL to update that property.

````sql
-- SQL generated by EF Core for the query and update
SELECT TOP(2) -- Reads up to two rows from the Books table. You asked for a single item, but this code makes sure that it fails if more than one row fits.
	[p].[BookId],
	[p].[Description],
	[p].[ImageUrl],
	[p].[Price],
	[p].[PublishedOn],
	[p].[Publisher],
	[p].[Title]
FROM [Books] AS [p]
WHERE [p].[Title] = N'Quantum Networking' -- Your LINQ Where method, which picks out the correct row by its title

SET NOCOUNT ON;
UPDATE [Books] -- SQL UPDATE command-in this case, on the Books table
	SET [PublishedOn] = @p0 -- Because EF Core’s DetectChanges method finds that only the PublishedOn property has changed, it can target that column in the table.
WHERE [BookId] = @p1; -- EF Core uses the primary key from the original book to uniquely select the row it wants to update.
SELECT @@ROWCOUNT; -- Sends back the number of rows that were inserted into this transaction. SaveChanges returns this integer, but normally, you can ignore it.
````



#### Handling disconnected updates in a web application

An update is a three-stage process, needing a read, an update, and a `SaveChanges` call to be executed by the same instance of the application’s DbContext. The problem is that for certain applications, such as websites and RESTful APIs, using the same instance of the application’s DbContext isn’t possible because in web applications, each HTTP request typically is a new request, with no data held over from the last HTTP request. In these types of applications, an update consists of two stages:

* The first stage is an initial read, done in one instance of the application’s DbContext.
* The second stage applies the update by using a new instance of the application’s DbContext.

In EF Core, this type of update is called a *disconnected* update because the first stage and the second stage use two different instances of the application’s DbContext. Here are the two main ways of handling disconnected updates:

* *You send only the data you need to update back from the first stage.* If you were updating the published date for a book, you would send back only the `BookId` and the `PublishedOn` properties. In the second stage, you use the primary key to reload the original entity with tracking and update the specific properties you want to change. In this example, the primary key is the `BookId`, and the property to update is the `PublishedOn` property of the `Book` entity. When you call `SaveChanges`, EF Core can work out which properties you’ve changed and update only those columns in the database.
* You send all the data needed to re-create the entity class back from the first stage. In the second stage, you rebuild the entity class, and maybe relationships, by using the data from the first stage and tell EF Core to update the whole entity. When you call `SaveChanges`, EF Core will know, because you told it, that it must update all the columns in the table row(s) affected with the substitute data that the first stage provided.



**DISCONNECTED UPDATE, WITH RELOAD**

For web applications, the approach of returning only a limited amount of data to the web server is a common way of handling EF Core updates. This approach makes the request faster, but a big reason for it is security. You wouldn’t want the `Price` of a `Book` to be returned, for example, as that information would allow hackers to alter the price of the book they want to buy. We prefer, is to use a special class that contains only properties that should be sent/received.



````c#
// ChangePubDateDto sends data to and receives it from the user
public class ChangePubDateDto
{
    public int BookId { get; set; } // Holds the primary key of the row you want to update, which makes finding the right row quick and accurate
    
    public string Title { get; set; } // You send over the title to show the user so that they can be sure they are altering the right book.
    
    [DataType(DataType.Date)] // The property you want to alter. You send out the current publication date and get back the changed publication date.
    public DateTime PublishedOn { get; set; }
}
````

````c#
// The ChangePubDateService class to handle the disconnected update
public class ChangePubDateService : IChangePubDateService
{
    private readonly EfCoreContext _context;
    
    public ChangePubDateService(EfCoreContext context)
    {
        _context = context;
    }

    public ChangePubDateDto GetOriginal(int id)
    {
        return _context.Books
            .Select(p => new ChangePubDateDto
            {
                BookId = p.BookId,
                Title = p.Title,
                PublishedOn = p.PublishedOn
            })
            .Single(k => k.BookId == id);
    }

    public Book UpdateBook(ChangePubDateDto dto)
    {
        var book = _context.Books.SingleOrDefault(
            x => x.BookId == dto.BookId);
        if (book == null)
        {
            throw new ArgumentException("Book not found");
        }

        book.PublishedOn = dto.PublishedOn;
        _context.SaveChanges();
        return book;
    }
}
````

The advantages of this reload-then-update approach is that it’s more secure (in our example, sending/returning the price of the book over HTTP would allow someone to alter it) and faster because of less data. The downside is that you have to write code to copy over the specific properties you want to update.



*The quickest way to read an entity class using its primary key(s)*
Two useful things about the `Find` method:

* The `Find` method checks the current application’s DbContext to see whether the required entity instance has already been loaded, which can save an access to the database. But if the entity isn’t in the application’s DbContext, the load will be slower because of this extra check.
* The `Find` method is simpler and quicker to type because it’s shorter than the `SingleOrDefault` version, such as `context.Find<Book>(key)` versus `context.SingleOrDefault(p => p.Bookid == key)`.

The upside of using the `SingleOrDefault` method is that you can add it to the end of a query with methods such as `Include`, which you can’t do with `Find`. Also `SingleOrDefault` is faster than `Find`.



**DISCONNECTED UPDATE, SENDING ALL THE DATA**

In some cases, all the data may be sent back, so there’s no reason to reload the original data. This can happen for simple entity classes, in some RESTful APIs, or process-to-process communication. A lot depends on how closely the given API format matches the database format and how much you trust the other system.



````c#
// Simulating an update/replace request from an external system
string json;
using (var context = new EfCoreContext(options)) // Simulates an external system returning a modified Author entity class as a JSON string
{
    var author = context.Books
        .Where(p => p.Title == "Quantum Networking")
        .Select(p => p.AuthorsLink.First().Author)
        .Single();
    author.Name = "Future Person 2";
    json = JsonConvert.SerializeObject(author);
}
using (var context = new EfCoreContext(options))
{
    var author = JsonConvert
        .DeserializeObject<Author>(json); // Simulates receiving a JSON string from an external system and decoding it into an Author class
    
    context.Authors.Update(author); // Update command, which replaces all the row data for the given primary key—in this case, AuthorId
    context.SaveChanges(); // Provides a link to the many-tomany linking table that links to the authors of this book
}
````



You call the EF Core Update command with the `Author` entity instance as a parameter, which marks as modified all the properties of the `Author` entity. When the `SaveChanges` command is called, it’ll update all the columns in the row that have the same primary key as the entity class.

The plus side of this approach is that the database update is quicker, because you don’t have the extra read of the original data. You also don’t have to write code to copy over the specific properties you want to update, which you did need to do in the previous approach.

The downsides are that more data can be transferred and that unless the API is carefully designed, it can be difficult to reconcile the data you receive with the data already in the database. Also, you’re trusting the external system to remember all the data correctly, especially the primary keys of your system.



### Handling relationships in updates

````c#
// The Book entity class, showing the relationships to update
public class Book // Book class contains the main book information.
{
    public int BookId { get; set; }
    //… other nonrelational properties removed for clarity
    //-----------------------------------------------
    //relationships

    public PriceOffer Promotion { get; set; } // Links to the optional PriceOffer
    public ICollection<Review> Reviews { get; set; } // Can be zero to many reviews of the book
    public ICollection<Tag> Tags { get; set; } // EF Core's automatic many-to-many relationship to the Tag entity class
    public ICollection<BookAuthor> AuthorsLink { get; set; } // Provides a link to the many-to-many linking table that links to the authors of this book
}
````



#### Principal and dependent relationships

The terms principal and dependent are used in EF to define parts of a relationship:

* *Principal entity* - Contains a primary key that the dependent relationship refer to via a foreign key
* *Dependent entity* - Contains the foreign key that refers to the principal entity’s primary key



**CAN THE DEPENDENT PART OF A RELATIONSHIP EXIST WITHOUT THE PRINCIPAL?**

The other aspect of a dependent relationship is whether it can exist on its own. If the principal relationship is deleted, is there a business case for the dependent relationship to still exist? In many cases, the dependent part of a relationship doesn’t make sense without the principal relationship. A book review has no meaning if the book it links to is deleted from the database, for example.

In a few cases, a dependent relationship should exist even if the principal part is deleted. Suppose that you want to have a log of all the changes that happen to a book in its lifetime. If you delete a book, you wouldn’t want that set of logs to be deleted too.

This task is handled in databases by handling the nullability of the foreign key. If the foreign key in the dependent relationship is non-nullable, the dependent relationship can’t exist without the principal. In the example Book App database, the `PriceOffer`, `Review`, and `BookAuthor` entities are all dependent on the principal, Book entity, so their foreign keys are of type int. If the book is deleted or the link to the book is removed, the dependent entities will be deleted.

But if you define a class for logging - let’s call it `BookLog` - you want this class to exist even if the book is deleted. To make this happen, you’d make its `BookId` foreign key of type `Nullable<int>`. Then, if you delete the book that the `BookLog` entity is linked to, you could configure that the `BookLog`’s `BookId` foreign key would be set to `null`.

What happens to the old relationships we remove depends on the nullability of the foreign key: if the foreign key is non-nullable, the dependent relationships are deleted, and if the foreign key is nullable, it’s set to `null`.



#### Updating one-to-one relationships

In our example Book App database, we have an optional, dependent relationship property called `Promotion` from the `Book` entity class to the `PriceOffer` entity class.

````c#
// PriceOffer entity class, showing the foreign key back to the Book entity
public class PriceOffer // PriceOffer, if present, is designed to override the normal price.
{
    public int PriceOfferId { get; set; }
    public decimal NewPrice { get; set; }
    public string PromotionalText { get; set; }
    
    //-----------------------------------------------
    //Relationships
    
    public int BookId { get; set; } // Foreign key back to the book it should be applied to
}
````



**CONNECTED STATE UPDATE**

The connected state update assumes that you’re using the same context for both the read and the update.



1. Load the `Book` entity with any existing `PriceOffer` relationship.
2. Set the relationship to the new `PriceOffer` entity you want to apply to this book.
3. Call `SaveChanges` to update the database.

````c#
// Adding a new promotional price to an existing book that doesn’t have one
var book = context.Books // Finds a book. In this example, the book doesn’t have an existing promotion, but it would also work if there were an existing promotion.
    .Include(p => p.Promotion) // Although the include isn’t needed because you’re loading something without a Promotion, using the include is good practice, as you should load any relationships if you’re going to change a relationship.
    .First(p => p.Promotion == null);

book.Promotion = new PriceOffer // Adds a new PriceOffer to this book
{
    NewPrice = book.Price / 2,
    PromotionalText = "Half price today!"
};
context.SaveChanges(); // The SaveChanges method calls DetectChanges, which finds that the Promotion property has changed, so it adds that entity to the PriceOffers table.
````

As you can see, the update of the relationship is like the basic update you made to change the book’s published date. In this case, EF Core has to do extra work because of the relationship. EF Core creates a new row in the `PriceOffers` table, which you can see in the SQL snippet that EF Core produces for the code:

````sql
INSERT INTO [PriceOffers]
	([BookId], [NewPrice], [PromotionalText])
	VALUES (@p0, @p1, @p2);
````

Now, what happens if there’s an existing promotion on the book (that is, the `Promotion` property in the `Book` entity class isn’t `null`)? That case is why the `Include(p => p.Promotion)` command in the query that loaded the `Book` entity class is so important. Because of that Include method, EF Core will know that an existing `PriceOffer` is assigned to this book and will delete it before adding the new version.

To be clear, in this case you must use some form of loading of the relationship - *eager*, *explicit*, *select*, or *lazy loading* of the relationship - so that EF Core knows about it before the update. If you don’t, and if there’s an existing relationship, EF Core will throw an exception on a duplicate foreign key `BookId`, which EF Core has placed a unique index on, and another row in the `PriceOffers` table will have the same value.



**DISCONNECTED STATE UPDATE**

In the disconnected state, the information to define which book to update and what to put in the `PriceOffer` entity class would be passed back from stage 1 to stage 2. That situation happened in the update of the book’s publication date, where the `BookId` and the `PublishedOn` values were fed back.

In the case of adding a promotion to a book, you need to pass in the `BookId`, which uniquely defines the book you want, plus the `NewPrice` and the `PromotionalText` values that make up the `PriceOffer` entity class. The next listing shows you the `ChangePriceOfferService` class, which contains the two methods to show the data to the user and update the promotion on the `Book` entity class when the user submits a request.

````c#
// ChangePriceOfferService class with a method to handle each stage
public class ChangePriceOfferService : IChangePriceOfferService
{
    private readonly EfCoreContext _context;

    public Book OrgBook { get; private set; }

    public ChangePriceOfferService(EfCoreContext context)
    {
        _context = context;
    }

    public PriceOffer GetOriginal(int id) // Gets a PriceOffer class to send to the user to update
    {
        OrgBook = _context.Books // Loads the book with any existing Promotion
            .Include(r => r.Promotion)
            .Single(k => k.BookId == id);
        
        return OrgBook?.Promotion // You return either the existing Promotion for editing or create a new one. The important point is to set the BookId, as you need to pass it through to the second stage.
            ?? new PriceOffer
        {
            BookId = id,
            NewPrice = OrgBook.Price
        };
    }

    public Book AddUpdatePriceOffer(PriceOffer promotion) // Handles the second part of the update, performing a selective add/update of the Promotion property of the selected book
    {
        var book = _context.Books // Loads the book with any existing promotion, which is important because otherwise, your new PriceOffer will clash and throw an error
            .Include(r => r.Promotion)
            .Single(k => k.BookId == promotion.BookId);
        
        if (book.Promotion == null) // Checks whether the code should create a new PriceOffer or update the existing PriceOffer
        {
            book.Promotion = promotion; // You need to add a new PriceOffer, so you assign the promotion to the relational link. EF Core will see it and add a new row in the PriceOffer table.
        }
        else
        {
            // You need to do an update, so you copy over only the parts that you want to change. EF Core will see this update and produce code to update only these two columns.
            book.Promotion.NewPrice = promotion.NewPrice;
            book.Promotion.PromotionalText = promotion.PromotionalText;
        }
        _context.SaveChanges(); // SaveChanges uses its DetectChanges method, which sees what changes—either adding a new PriceOffer or updating an existing one.
        return book; // Returns the updated book
    }
}
````

This code either updates an existing `PriceOffer` or adds a new `PriceOffer` if none exists. When `SaveChanges` is called, it can work out, via EF Core’s `DetectChanges` method, what type of update is needed and create the correct SQL to update the database.



**ALTERNATIVE WAY OF UPDATING THE RELATIONSHIP: CREATING A NEW ROW DIRECTLY**

We’ve approached this update as changing a relationship in the `Book` entity class, but you can also approach it as creating/deleting a row in the `PriceOffers` table. This listing finds the first `Book` in the database that doesn’t have a `Promotion` linked to it and then adds a new `PriceOffer` entity to that book.

````c#
// Creating a PriceOffer row to go with an existing book
var book = context.Books
    .First(p => p.Promotion == null); // You find the book that you want to add the new PriceOffer to, which must not be an existing PriceOffer.

context.Add( new PriceOffer
{
    BookId = book.BookId,
    NewPrice = book.Price / 2,
    PromotionalText = "Half price today!"
});
context.SaveChanges(); // SaveChanges adds the PriceOffer to the PriceOffers table.
````

You should note that previously, you didn’t have to set the `BookId` property in the `PriceOffer` entity class, because EF Core did that for you. But when you’re creating a relationship this way, you do need to set the foreign key. Having done so, if you load the `Book` entity class with its `Promotion` relationship after the previous create code, you’ll find that the `Book` has gained a `Promotion` relationship.

The advantage of creating the dependent entity class is that it saves you from needing to reload the principal entity class (in this case, `Book`) in a disconnected state. The downside is that EF Core doesn’t help you with the relationships. In this case, if there were an existing `PriceOffer` on the book and you added another, `SaveChanges` would fail because you’d have two `PriceOffer` rows with the same foreign key.

When EF Core can’t help you with the relationships, you need to use the create/delete approach with care. Sometimes, this approach can make handling a complex relationship easier, so it’s worth keeping in mind, but I prefer updating the principal entity class’s relationship in most one-to-one cases.



#### Updating one-to-many relationships

````c#
// The Review class, showing the foreign key back to the Book entity class
public class Review // Holds customer reviews with their ratings
{
    public int ReviewId { get; set; }
    public string VoterName { get; set; }
    public int NumStars { get; set; }
    public string Comment { get; set; }
    
    //-----------------------------------------
    //Relationships
    
    public int BookId { get; set; } // Foreign key holds the key of the book this review belongs to.
}
````

The one-to-many relationship in the Book App database is represented by `Book`’s `Reviews`; a user of the site can add a review to a book. There can be any number of reviews, from none to a lot. This listing shows the `Review`-dependent entity class, which links to the Books table via the foreign key called `BookId`.



**CONNECTED STATE UPDATE**

````c#
// Adding a review to a book in the connected state
var book = context.Books
    .Include(p => p.Reviews)
    .First(); // Finds the first book and loads it with any reviews it might have

book.Reviews.Add(new Review
{
    VoterName = "Unit Test",
    NumStars = 5,
    Comment = "Great book!"
});
context.SaveChanges(); // SaveChanges calls DetectChanges, which finds that the Reviews property has changed, and from there finds the new Review, which it adds to the Review table.
````

This code follows the same pattern as the one-to-one connected update: load the `Book` entity class and the `Reviews` relationship via the `Include` method. But in this case, you add the `Review` entity to the Book’s Reviews collection. Because you used the `Include` method, the `Reviews` property will be an empty collection if there are no reviews or a collection of the reviews linked to this book. In this example, the database already contains some `Book` entities, and we take the first.



**ALTERING/REPLACING ALL THE ONE-TO-MANY RELATIONSHIPS**

If the books had categories (say, Software Design, Software Languages, and so on), you might allow an admin user to change the categories. One way to implement this change would be to show the current categories in a multiselect list, allow the admin user to change them, and then replace *all* the categories on the book with the new selection.

EF Core makes replacing the whole collection easy. If you assign a new collection to a one-to-many relationship that has been loaded with tracking (such as by using the `Include` method), EF Core will replace the existing collection with the new collection. If the items in the collection can be linked to only the principal class (the dependent class has a non-nullable foreign key), by default, EF Core will delete the items that were in the collection that have been removed.

Next is an example of replacing the whole collection of existing book reviews with a new collection. The effect is to remove the original reviews and replace them with the one new review.

````c#
// Replacing a whole collection of reviews with another collection
var book = context.Books
    .Include(p => p.Reviews) // This include is important; it creates a collection with any existing reviews in it or an empty collection if there are no existing reviews.
    .Single(p => p.BookId == twoReviewBookId); // This book you’re loading has two reviews.

book.Reviews = new List<Review> // You replace the whole collection.
{
    new Review
    {
        VoterName = "Unit Test",
        NumStars = 5,
    }
};
context.SaveChanges(); // SaveChanges, via DetectChanges, knows that the old collection should be deleted and that the new collection should be written to the database.
````

Because you’re using test data in the example, you know that the book with the primary key `twoReviewBookId` has two reviews and that the book is the only one with reviews; hence, there are only two reviews in the whole database. After the `SaveChanges` method is called, the book has only one review, and the two old reviews have been deleted, so now the database has only one review in it.

Removing a single row is as simple as removing the entity from the list. EF Core will see the change and delete the row that’s linked to that entity. Similarly, if you add a new `Review` to the `Book`’s `Reviews` collection property, EF Core will see that change to that collection and add the new `Review` to the database.

The loading of the existing collection is important for these changes: if you don’t load them, EF Core can’t remove, update, or replace them. The old versions will still be in the database after the update because EF Core didn’t know about them at the time of the update. You haven’t replaced the existing two `Reviews` with your single `Review`. In fact, you now have three `Reviews` - the two that were originally in the database and your new one - which is not what you intended to do.



**DISCONNECTED-STATE UPDATE**

In the disconnected state, you create an empty `Review` entity class but fill in its foreign key, `BookId`, with the book the user wants to provide a review for. Then the user votes on the book, and you add that review to the book that they referred to.

````c#
// Adding a new review to a book in the example Book App
public class AddReviewService
{
    private readonly EfCoreContext _context;
    
    public string BookTitle { get; private set; }
    
    public AddReviewService(EfCoreContext context)
    {
        _context = context;
    }

    public Review GetBlankReview(int id) // Forms a review to be filled in by the user
    {
        BookTitle = _context.Books
            .Where(p => p.BookId == id)
            .Select(p => p.Title)
            .Single();
        return new Review
        {
            BookId = id
        };
    }

    public Book AddReviewToBook(Review review) // Updates the book with the new review
    {
        var book = _context.Books
            .Include(r => r.Reviews)
            .Single(k => k.BookId == review.BookId);
        book.Reviews.Add(review); // Adds the new review to the Reviews collection
        _context.SaveChanges(); // SaveChanges uses its DetectChanges method, which sees that the Book Review property has changed, and creates a new row in the Review table.
        return book; // Returns the updated book
    }
}
````

This code has a simpler first part than the previous disconnected-state examples because you’re adding a new review, so you don’t have to load the existing data for the user.



**ALTERNATIVE WAY OF UPDATING THE RELATIONSHIP: CREATING A NEW ROW DIRECTLY**

As with the `PriceOffer`, you can add a one-to-many relationship directly to the database. But again, you take on the role of managing the relationship. If you want to replace the entire reviews collection, for example, you’d have to delete all the rows that the reviews linked to the book in question before adding your new collection.

Adding a row directly to the database has some advantages, because loading all the one-to-many relationships might turn out to be a lot of data if you have lots of items and/or they’re big. Therefore, keep this approach in mind if you have performance issues.



#### Updating a many-to-many relationship

In EF Core, we talk about many-to-many relationships, but a relational database doesn’t directly implement many-to-many relationships. Instead, we’re dealing with two one-to-many relationships.



![](./diagrams/svg/03_02_updating_many_to_many_relationship.drawio.svg)



In EF Core, you have two ways to create many-to-many relationships between two entity classes:

* You link to a linking table in each entity—that is, you have an `ICollection<LeftRight>` property in your `Left` entity class. You need to create an entity class to act as the linking table (such as `LeftRight`), but that entity class lets you add extra data in the linking table so that you can sort/filter the many-to-many relationships.
* You link directly between the two entity classes you want to have a many-to-many relationship—that is, you have an `ICollection<Right>` property in your `Left` entity class. This link is much easier to code because EF Core handles the creation of the linking table, but then you can’t access the linking table in a normal Include method to sort/filter.



**UPDATING A MANY-TO-MANY RELATIONSHIP VIA A LINKING ENTITY CLASS**

In the `Book` entity class, you need a many-to-many link to the `Authors` of the book. But in a book, the order of the authors’ names matters. Therefore, you create a linking table with an `Order` (`byte`) property that allows you to display the `Author`'s `Name` properties in the correct order, which means that you

* Create an entity class called `BookAuthor`, which contains both the primary key of the `Book` entity class (`BookId`) and the primary key of the `Author` entity class (`AuthorId`). You also add an `Order` property, which contains a number setting the order in which the `Authors` should be displayed for this book. The `BookAuthor` linking entity class also contains two one-to-one relationships to the `Author` and the `Book`.
* You add a navigational property called `AuthorsLink` of type `ICollection<BookAuthor>` to your `Book` entity class.
* You also add a navigational property called `BooksLink` of type `ICollection<BookAuthor>` to your `Author` entity class.



![](./diagrams/svg/03_03_book_many_to_many_relationship.drawio.svg)



The `BookAuthor` entity class has two properties: `BookId` and `AuthorId`. These properties are foreign keys to the Books table and the Authors table, respectively. Together, they also form the primary key (known as a *composite key*, because it has more than one part) for the `BookAuthor` row. The composite key has the effect of ensuring that there’s only one link between the `Book` and the `Author`.

````c#
// Adding a new Author to the book Quantum Networking
var book = context.Books
    .Include(p => p.AuthorsLink)
    .Single(p => p.Title == "Quantum Networking");

var existingAuthor = context.Authors
    .Single(p => p.Name == "Martin Fowler");

book.AuthorsLink.Add(new BookAuthor // You add a new BookAuthor linking entity to the Book’s AuthorsLink collection.
{
    Book = book,
    Author = existingAuthor,
    Order = (byte) book.AuthorsLink.Count // You set the Order to the old count of AuthorsLink—in this case, 1 (because the first author has a value of 0).
});
context.SaveChanges(); // The SaveChanges will create a new row in the BookAuthor table.
````

The thing to understand is that the `BookAuthor` entity class is the *many* side of the relationship. This listing, which adds another author to one of the books, should look familiar because it’s similar to the one-to-many update methods I’ve already explained.

One thing to note is that when you load the Book’s `AuthorsLink`, you don’t need to load the corresponding `BooksLink` in the `Author` entity class. The reason is that when you update the `AuthorsLink` collection, EF Core knows that there is a link to the `Book`, and during the update, EF Core will fill in that link automatically. The next time someone loads the `Author` entity class and its `BooksLink` relationship, they’ll see a link to the *Quantum Networking* book in that collection.

Also be aware that deleting an `AuthorsLink` entry won’t delete the `Book` or `Author` entities they link to because that entry is the *one* end of a *one-to-many* relationship, which isn’t dependent on the `Book` or `Author`. In fact, the `Book` and `Author` entity classes are *principal entities*, with the `BookAuthor` classes being dependent on both of the principal entity classes.



**UPDATING A MANY-TO-MANY RELATIONSHIP WITH DIRECT ACCESS TO THE OTHER ENTITY**

EF Core added the ability to access another entity class directly in a many-to-many relationship. This ability makes it much easier to set up and use the many-to-many relationship, but you won’t be able to access the linking table in an `Include` method.

In the Book App, a book can have zero to many categories, such as Linux, Databases, and Microsoft .NET, to help a customer find the right book. These categories are held in a `Tag` entity (the `TagId` holds the category name) with a direct many-to-many relationship to a `Book`. This allows the `Book` to show its categories in the Book App’s book list display and also allows the Book App to provide a feature to filter the book list display by a category.



![](./diagrams/svg/03_04_book_many_to_many_relationship.drawio.svg)



This direct-access many-to-many feature makes adding/deleting links between the `Book` entity and the `Tag` entities simple.

````c#
// Adding a Tag to a Book via a direct many-to-many relationship
var book = context.Books
    .Include(p => p.Tags)
    .Single(p => p.Title == "Quantum Networking");

var existingTag = context.Tags
    .Single(p => p.TagId == "Editor's Choice");

book.Tags.Add(existingTag); // You add the Tag to the Books Tags collection.
context.SaveChanges(); // When SaveChanges is called, EF Core creates a new row in the hidden BookTags table.
````

You’ll see that it’s much easier to add a new entry to a direct many-to-many relationship. EF Core takes on the work of creating the necessary row in the `BooksTag` table. And if you removed an entry in the Tags collection, you would delete the corresponding row in the `BooksTag` table.



**ALTERNATIVE WAY OF UPDATING THE RELATIONSHIP: CREATING A NEW ROW DIRECTLY**

We’ll discuss another approach: creating the linking table row directly. The benefit of this approach is better performance when you have lots of entries in the collection. Rather than having to read in the collection, you can create a new entry in the linking table. You could create a `BookAuthor` entity class and fill in the `Book` and `Author` one-to-one relationships in that class, for example. Then you Add that new `BookAuthor` entity instance to the database and call `SaveChanges`. For the `AuthorsLink` collection, which is likely to be small, this technique is most likely not worth the extra effort, but for many-to-many relationships that contain lots of linking entries, it can significantly improve performance.



#### Advanced feature: Updating relationships via foreign keys

When you added a review to a book, for example, you loaded the `Book` entity with all its `Reviews`. That’s fine, but in a disconnected state, you have to load the `Book` and all its `Reviews` from the book’s primary key that came back from the browser/RESTful API. In many situations, you can cut out the loading of the entity classes and set the foreign keys instead.

The code assumes that the `ReviewId` of the `Review` the user wants to change and the new `BookId` that they want to attach the review to are returned in a variable called *dto*.

````c#
// Updating the foreign key to change a relationship
var reviewToChange = context
    .Find<Review>(dto.ReviewId);
reviewToChange.BookId = dto.NewBookId; // Changes the foreign key in the review to point to the book it should be linked to
context.SaveChanges(); // Calls SaveChanges, which finds the foreign key in the review changed, so it updates that column in the database
````

The benefit of this technique is that you don’t have to load the `Book` entity class or use an `Include` command to load all the `Reviews` associated with this book. In our example Book App, these entities aren’t too big, but in a real application, the principal and dependent entities could be quite large. (Some Amazon products have thousands of reviews, for example.) In disconnected systems, in which we often send only the primary keys over the disconnect, this approach can be useful for cutting down on database accesses and, hence, improving performance.



### Deleting entities

The final way to change the data in the database is to delete a row from a table.



#### Soft-delete approach: Using a global query filter to hide entities

One school of thought says that you shouldn’t delete anything from a database but use a status to hide it, known as a soft delete. EF Core provides a feature called global query filter that allows a soft delete to be implemented simply.

The thinking behind a soft delete is that in real-world applications, data doesn’t stop being data; it transforms into another state. In the case of our books example, a book may not still be on sale, but the fact that the book existed isn’t in doubt, so why delete it? Instead, you set a flag to say that the entity is to be hidden in all queries and relationship. To see how this process works, you’ll add the soft-delete feature to the list of `Book` entities. To do so, you need to do two things:

* Add a `boolean` property called `SoftDeleted` to the `Book` entity class. If that property is `true`, the `Book` entity instance is soft-deleted; it shouldn’t be found in a normal query.
* *Add a global query filter via EF Core’s fluent configuration commands.* The effect is to apply an extra `Where` filter to any access to the `Books` table.

````c#
public class Book
{
    //… other properties left out for clarity
    public bool SoftDeleted { get; set; }
}
````



````c#
// Adding a global query filter to the DbSet<Book>Books property
public class EfCoreContext : DbContext
{
    //… Other parts removed for clarity
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //… other configration parts removed for clarity

        modelBuilder.Entity<Book>()
            .HasQueryFilter(p => !p.SoftDeleted); // Adds a filter to all accesses to the Book entities. You can bypass this filter by using the IgnoreQueryFilters operator.
    }
}
````



To soft-delete a `Book` entity, you need to set the `SoftDeleted` property to true and call `SaveChanges`. Then any query on the `Book` entities will exclude the Book entities that have the `SoftDeleted` property set to `true`.

If you want to access all the entities that have a model-level filter, you add the `IgnoreQueryFilters` method to the query, such as `context.Books.IgnoreQueryFilters()`. This method bypasses any query filter on that entity.



#### Deleting a dependent-only entity with no relationships

````c#
// Removing (deleting) an entity from the database
var promotion = context.PriceOffers
    .First();

context.Remove(promotion); // Removes that PriceOffer from the application’s DbContext. The DbContext works out what to remove based on its parameter type.
context.SaveChanges(); // SaveChanges calls DetectChanges, which finds a tracked PriceOffer entity marked as deleted and then deletes it from the database.
````

Calling the `Remove` method sets the `State` of the entity provided as the parameter to `Deleted`. Then, when you call `SaveChanges`, EF Core finds the entity marked as `Deleted` and creates the correct database commands to delete the appropriate row from the table the entity referred to (in this case, a row in the PriceOffers table). The SQL command that EF Core produces for SQL Server is shown in the following snippet:

````sql
SET NOCOUNT ON;
DELETE FROM [PriceOffers]
WHERE [PriceOfferId] = @p0;
SELECT @@ROWCOUNT;
````



#### Deleting a principal entity that has relationships

*Referential integrity* is a relational database concept indicating that table relationships must always be consistent. Any foreign-key field must agree with the primary key referenced by the foreign key.

Relational databases need to keep referential integrity, so if you delete a row in a table that other rows are pointing to via a foreign key, something has to happen to stop referential integrity from being lost.

Following are three ways that you can set a database to keep referential integrity when you delete a principal entity with dependent entities:

* You can tell the database server to delete the dependent entities that rely on the principal entity, known as *cascade deletes*.
* You can tell the database server to set the foreign keys of the dependent entities to null, if the column allows that.
* If neither of those rules is set up, the database server will raise an error if you try to delete a principal entity with dependent entities.



#### Deleting a book with its dependent relationships

We’re going to delete a `Book` entity, which is a principal entity with three dependent relationships: `Promotion`, `Reviews`, and `AuthorsLink`. These three dependent entities can’t exist without the Book entity; a non-nullable foreign key links these dependent entities to a specific `Book` row.

By default, EF Core uses cascade deletes for dependent relationships with nonnullable foreign keys. Cascade deletes make deleting principal entities easier from the developer’s point of view, because the other two rules need extra code to handle deleting the dependent entities. But in many business applications, this approach may not be appropriate. This chapter uses the cascade delete approach because it’s EF Core’s default for non-nullable foreign keys.

With that caveat in mind, let’s see cascade delete in action by using the default cascade-delete setting to delete a Book that has relationships. This listing loads the `Promotion` (`PriceOffer` entity class), `Reviews`, `AuthorsLink`, and `Tags` relationships with the `Book` entity class before deleting that `Book`.

````c#
// Deleting a book that has three dependent entity classes
var book = context.Books
    .Include(p => p.Promotion)
    .Include(p => p.Reviews)
    .Include(p => p.AuthorsLink)
    .Include(p => p.Tags)
    .Single(p => p.Title == "Quantum Networking");

context.Books.Remove(book); // Deletes that book
context.SaveChanges(); // SaveChanges calls DetectChanges, which finds a tracked Book entity marked as deleted, deletes its dependent relationships, and then deletes the book.
````

EF Core deletes the `Book`, the `PriceOffer`, the two `Reviews`, the single `BookAuthor` link, and the single (hidden) `BookTag`.

You put in the four `Includes`, EF Core knew about the dependent entities and performed the delete. If you didn’t incorporate the `Includes` in your code, EF Core wouldn’t know about the dependent entities and couldn’t delete the three dependent entities. In that case, the problem of keeping referential integrity would fall to the database server, and its response would depend on how the DELETE ON part of the foreign-key constraint was set up. Databases created by EF Core for these entity classes would, by default, be set to use cascade deletes.

The `Author` and `Tag` linked to the `Book` aren’t deleted because they are not dependent entities of the `Book`; only the `BookAuthor` and `BookTag` linking entities are deleted. This arrangement makes sense because the `Author` and `Tag` might be used on other `Books`.

Sometimes, it’s useful to stop a principal entity from being deleted if a certain dependent entity is linked to it. In our example Book App, for example, if a customer orders a book, you want to keep that order information even if the book is no longer for sale. In this case, you change the EF Core’s on-delete action to `Restrict` and remove the `ON DELETE CASCADE` from the foreign-key constraint in the database so that an error will be raised if an attempt to delete the book is made.



### Summary

* Entity instances have a `State`, whose values can be `Added`, `Unchanged`, `Modified`, `Deleted`, or `Detached`. This `State` defines what happens to the entity when `SaveChanges` is called.
* If you `Add` an entity, its `State` is set to `Added`. When you call `SaveChanges`, that entity is written out to the database as a new row.
* You can update a property, or properties, in an entity class by loading the entity class as a tracked entity, changing the property/properties, and calling `SaveChanges`.
* Real-world applications use two types of update scenarios - connected and disconnected state - that affect the way you perform the update.
* EF Core has an Update method, which marks the whole of the entity class as updated. You can use this method when you want to update the entity class and have all the data already available to you.
* When you’re updating a relationship, you have two options, with different advantages and disadvantages:
  * You can load the existing relationship with the primary entity and update that relationship in the primary entity. EF Core will sort things out from there. This option is easier to use but can create performance issues when you’re dealing with large collections.
  * You can create, update, or delete the dependent entity. This approach is harder to get right but typically is faster because you don’t need to load any existing relationships.
* To delete an entity from the database, you use the `Remove` method, followed by the `SaveChanges` method.





## 4. Using EF Core in business logic

* Understanding business logic and its use of EF Core
* Looking at three types of business logic, from the easy to the complex
* Reviewing each type of business logic, with pros and cons
* Adding a step that validates the data before it’s written to the database
* Using transactions to daisy-chain code sequences



### Five guidelines for building business logic that uses EF Core

* *The business logic has first call on how the database structure is defined.* Because the problem you’re trying to solve is the heart of the problem, the logic should define the way the whole application is designed. Therefore, you try to make the database structure and the entity classes match your business logic data needs as much as you can.
* *The business logic should have no distractions.* Writing the business logic is difficult enough in itself, so you isolate it from all the other application layers other than the entity classes. When you write the business logic, you must think only about the business problem you’re trying to fix. You leave the task of adapting the data for presentation to the service layer in your application.
* *Business logic should think that it’s working on in-memory data.* You need to have some load and save parts, of course, but for the core of your business logic, treat the data (as much as is practical) as though it’s a normal, in-memory class or collection.
* *Isolate the database access code into a separate project.* This rule came out of writing an e-commerce application with complex pricing and delivery rules. You should use another project, a companion to the business logic, to hold all the database access code.
* *The business logic shouldn’t call EF Core’s `SaveChanges` directly.* You should have a class in the service layer (or a custom library) whose job it is to run the business logic. If there are no errors, this class calls `SaveChanges`. The main reason for this rule is to have control of whether to write out the data, but it has other benefits.



![](./diagrams/svg/04_01_business_logic.drawio.svg)



#### Guideline 1: Business logic has first call on defining the database structure

This guideline says that the design of the database should follow the business needs - in this case, represented by six business rules. Only three of these rules are relevant to the database design:

* An order must include at least one book (implying that there can be more).
* The price of the book must be copied to the order, because the price could change later.
* The order must remember the person who ordered the books.

These three rules dictates an `Order` entity class that has a collection of `LineItem` entity classes - a one-to many relationship. The `Order` entity class holds the information about the person placing the order, and each `LineItem` entity class holds a reference to the book order, how many, and at what price.



![](./diagrams/svg/04_02_guideline1_business_logic_has_first_call_on_defining.drawio.svg)



#### Guideline 2: Business logic should have no distractions

You are at the heart of the business logic code, and the code here will do most of the work. This code is going to be the hardest part of the implementation that you write, but you want to help yourself by cutting off any distractions. That way, you can stay focused on the problem.



**CHECKING FOR ERRORS AND FEEDING THEM BACK TO THE USER: VALIDATION**

You have two main approaches to handling the passing of errors back up to higher levels. One is to throw an exception when an error occurs, and the other is to pass back the errors to the caller via a status interface. Each option has its own advantages and disadvantages. This example uses the second approach: passing the errors back in some form of status class to the higher level to check.

`BizActionErrors` abstract class provides a common error-handling interface for all your business logic. The class contains a C# method called `AddError` that the business logic can call to add an error and an *immutable list* (a list that can’t be changed) called `Errors`, which holds all the validation errors found while running the business logic.

You’ll use a class called `ValidationResult` to store each error because it’s the standard way of returning errors with optional, additional information on the exact property the error was related to.

````c#
// Abstract base class providing error handling for your business logic
public abstract class BizActionErrors // Abstract class that provides error handling for business logic
{
    private readonly List<ValidationResult> _errors = new List<ValidationResult>(); // Holds the list of validation errors privately
    
    public IImmutableList<ValidationResult> Errors => _errors.ToImmutableList(); // Provides a public, immutable list of errors

    public bool HasErrors => _errors.Any(); // Creates a bool HasErrors to make checking for errors easier

    protected void AddError(string errorMessage, params string[] propertyNames) // Allows a simple error message, or an error message with properties linked to it, to be added to the errors list
    {
        _errors.Add( new ValidationResult(errorMessage, propertyNames)); // Validation result has an error message and a possibly empty list of properties it’s linked to
    }
}
````

Using this abstract class means that your business logic is easier to write and all your business logic has a consistent way of handling errors. The other advantage is that you can change the way errors are handled internally without having to change any of your business logic code.
Your business logic for handling an order does a lot of validation, which is typical for an order, because it often involves money. Other business logic may not do any validation, but the base class `BizActionErrors` will automatically return a `HasErrors` of false, which means that all business logic can be dealt with in the same way.



#### Guideline 3: Business logic should think that it’s working on in-memory data

Now you’ll start on the main class: `PlaceOrderAction`, which contains the pure business logic. This class relies on the companion class `PlaceOrderDbAccess` to present the data as an in-memory set (in this case, a dictionary) and to write the created order to the database. Although you’re not trying to hide the database from the pure business logic, you do want it to work as though the data is normal .NET classes.

`PlaceOrderAction` class, which inherits the abstract class `BizActionErrors` to handle returning error messages to the user. It also uses two methods that the companion `PlaceOrderDbAccess` class provides:

* `FindBooksByIdsWithPriceOffers` - Takes the list of `BookIds` and returns a dictionary with the `BookId` as the key and the `Book` entity class as the value and any associated `PriceOffers`
* `Add` - Adds the `Order` entity class with its `LineItem` collection to the database

````c#
// PlaceOrderAction class with build-a-new-order business logic
public class PlaceOrderAction :
	BizActionErrors, // The BizActionErrors class provides error handling for the business logic.
	IBizAction<PlaceOrderInDto,Order> // The IBizAction interface makes the business logic conform to a standard interface.
{
    private readonly IPlaceOrderDbAccess _dbAccess;

    public PlaceOrderAction(IPlaceOrderDbAccess dbAccess) // The PlaceOrderAction uses PlaceOrder-DbAccess class to handle database accesses.
    {
        _dbAccess = dbAccess;
    }

    public Order Action(PlaceOrderInDto dto) // This method is called by the BizRunner to execute this business logic.
    {
        // Some basic validation
        if (!dto.AcceptTAndCs)
        {
            AddError("You must accept the T&Cs to place an order.");
            return null;
        }

        if (!dto.LineItems.Any())
        {
            AddError("No items in your basket.");
            return null;
        }
        
        var booksDict = _dbAccess.FindBooksByIdsWithPriceOffers(dto.LineItems.Select(x => x.BookId)); // The PlaceOrderDbAccess class finds all the bought books, with optional PriceOffers.
        var order = new Order // Creates the Order, using FormLineItemsWithError Checking to create the LineItems
        {
            CustomerId = dto.UserId,
            LineItems = FormLineItemsWithErrorChecking(dto.LineItems, booksDict)
        };

        if (!HasErrors) // Adds the order to the database only if there are no errors
        {
            _dbAccess.Add(order);
        }

        return HasErrors ? null : order; // If there are errors, returns null; otherwise, returns the order
    }

    private List<LineItem> FormLineItemsWithErrorChecking(IEnumerable<OrderLineItem> lineItems, IDictionary<int,Book> booksDict) // This private method handles the creation of each LineItem for each book ordered.
    {
        var result = new List<LineItem>();
        var i = 1;

        foreach (var lineItem in lineItems) // Goes through each book type that the person ordered
        {
            if (!booksDict.ContainsKey(lineItem.BookId)) // Treats a missing book as a system error and throws an exception
            {
                throw new InvalidOperationException("An order failed because book, " + $"id = {lineItem.BookId} was missing.");
            }
            
            var book = booksDict[lineItem.BookId];
            var bookPrice = book.Promotion?.NewPrice ?? book.Price; // Calculates the price at the time of the order

            if (bookPrice <= 0) // More validation that checks whether the book can be sold
            {
                AddError($"Sorry, the book '{book.Title}' is not for sale.");
            }
            else
            {
                //Valid, so add to the order
                result.Add(new LineItem // Everything is OK, so create the LineItem entity class with the details.
                           {
                               BookPrice = bookPrice,
                               ChosenBook = book,
                               LineNum = (byte)(i++),
                               NumBooks = lineItem.NumBooks
                           });
            }
        }
        
        return result; // Returns all the LineItems for this order
    }
}
````

You’ll notice that you add another validation check to ensure that the book the user selected is still in the database. This check wasn’t in the business rules, but it could occur, especially if malicious inputs were provided. In this case, you make a distinction between errors that the user can correct, which are returned by the `Errors` property, and system errors (in this case, a missing book), for which you throw an exception that the system should log.

You may have seen at the top of the class that you apply an interface in the form of `IBizAction<PlaceOrderInDto,Order>`. This interface ensures that this business logic class conforms to a standard interface that you use across all your business logic.



#### Guideline 4: Isolate the database access code into a separate project

Put all the database access code that the business logic needs in a separate, companion class. This technique ensures that all the database accesses are in one place, making testing, refactoring, and performance tuning much easier.

You make sure that your pure business logic, class `PlaceOrderAction`, and business database access class `PlaceOrderDbAccess` are in separate projects. That approach allows you to exclude any EF Core libraries from the pure business logic project, ensuring that all database access is done via the companion class, `PlaceOrderDb-Access`.

`PlaceOrderDbAccess` class, which implements two methods to provide the database accesses that the pure business logic needs:

* The `FindBooksByIdsWithPriceOffers` method, which finds and loads each `Book` entity class, with any optional `PriceOffer`.
* The `Add` method, which adds the finished `Order` entity class to the application’s DbContext property, `Orders`, so that it can be saved to the database after EF Core’s `SaveChanges` method is called.

````c#
// PlaceOrderDbAccess, which handles all the database accesses
public class PlaceOrderDbAccess : IPlaceOrderDbAccess
{
    private readonly EfCoreContext _context;

    public PlaceOrderDbAccess(EfCoreContext context) // All the BizDbAccess need the application’s DbContext to access the database.
    {
        _context = context;
    }

    public IDictionary<int, Book> FindBooksByIdsWithPriceOffers // This method finds all the books that the user wants to buy.
        (IEnumerable<int> bookIds) // The BizLogic hands a collection of BookIds, which the checkout has provided.
    {
        return _context.Books
            .Where(x => bookIds.Contains(x.BookId)) // Finds a book for each Id, using the LINQ Contains method to find all the keys
            .Include(r => r.Promotion) // Includes any optional promotion, which the BizLogic needs for working out the price
            .ToDictionary(key => key.BookId); // Returns the result as a dictionary to make it easier for the BizLogic to look them up
    }

    public void Add(Order newOrder) // This method adds the new order to the DbContext’s Orders DbSet collection.
    {
        _context.Add(newOrder);
    }
}
````

The `PlaceOrderDbAccess` class implements an interface called `IPlaceOrderDbAccess`, which is how the `PlaceOrderAction` class accesses this class.



#### Guideline 5: Business logic shouldn’t call EF Core’s SaveChanges

The final rule says that the business logic doesn’t call EF Core’s `SaveChanges`, which would update the database directly. There are a few reasons for this rule:

* You consider the service layer to be the main orchestrator of database accesses: it’s in command of what gets written to the database.
* The service layer calls `SaveChanges` only if the business logic returns no errors.

Each `BizRunner` works by defining a generic interface that the business logic must implement. Your class in the `BizLogic` project runs an action that expects a single input parameter of type `PlaceOrderInDto` and returns an object of type `Order`. Therefore, the `PlaceOrderAction` class implements the interface as shown in the following listing, but with its input and output types (`IBizAction<PlaceOrderInDto,Order>`).

````c#
// The interface that allows the BizRunner to execute business logic
public interface IBizAction<in TIn, out TOut> // The BizAction uses the TIn and a TOut to define the input and output of the Action method.
{
    // Returns the error information from the business logic
    IImmutableList<ValidationResult> Errors { get; }
    bool HasErrors { get; }
    TOut Action(TIn dto); // The action that the BizRunner will call
}
````

When you have the business logic class implement this interface, the `BizRunner` knows how to run that code. This `BizRunner` variant is designed to work with business logic that has an input, provides an output, and writes to the database.

````c#
// The BizRunner that runs the business logic and returns a result or errors
public class RunnerWriteDb<TIn, TOut>
{
    private readonly IBizAction<TIn, TOut> _actionClass;
    private readonly EfCoreContext _context;

    // Error information from the business logic is passed back to the user of the BizRunner.
    public IImmutableList<ValidationResult> Errors => _actionClass.Errors;
    public bool HasErrors => _actionClass.HasErrors;

    // Handles business logic that conforms to the IBizAction<TIn, TOut> interface
    public RunnerWriteDb(
        IBizAction<TIn, TOut> actionClass,
        EfCoreContext context)
    {
        _context = context;
        _actionClass = actionClass;
    }

    public TOut RunAction(TIn dataIn) // Calls RunAction in your service layer or in your presentation layer if the data comes back in the right form
    {
        var result = _actionClass.Action(dataIn); // Runs the business logic you gave it
        if (!HasErrors) // If there are no errors, calls SaveChanges to execute any add, update, or delete methods
        {
            _context.SaveChanges();
        }
        return result; // Returns the result that the business logic returned
    }
}
````

The `BizRunner` pattern hides the business logic and presents a common interface/API that other classes can use. The caller of the `BizRunner` doesn’t need to worry about EF Core, because all the calls to EF Core are in the `BizDbAccess` code or in the BizRunner. That fact in itself is reason enough to use the `BizRunner` pattern, but as you’ll see later, this pattern allows you to create other forms of `BizRunner` that add extra features.



#### Putting it all together: Calling the order-processing business logic

If the business logic is successful, the code clears the basket cookie and returns the `Order` entity class key so that a confirmation page can be shown to the user. If the order fails, it doesn’t clear the basket cookie, and the checkout page is shown again, with the error messages, so that the user can correct any problems and retry.

````c#
// The PlaceOrderService class that calls the business logic
public class PlaceOrderService
{
    private readonly BasketCookie _basketCookie; // This class handles the basket cookie, which contains the user-selected books.

    private readonly RunnerWriteDb<PlaceOrderInDto, Order> _runner; // Defines the input, PlaceOrderInDto, and output, Order, of this business logic
    public IImmutableList<ValidationResult> Errors => _runner.Errors; // Holds any errors sent back from the business logic

    public PlaceOrderService(
        IRequestCookieCollection cookiesIn,
        IResponseCookies cookiesOut,
        EfCoreContext context) // The constructor takes in the cookie in/out data, plus the application’s DbContext.
    {
        _basketCookie = new BasketCookie(cookiesIn, cookiesOut); // Creates a BasketCookie using the cookie in/out data from ASP.NET Core
        _runner = new RunnerWriteDb<PlaceOrderInDto, Order>(new PlaceOrderAction(new PlaceOrderDbAccess(context)), context); // Creates the BizRunner, with the business logic, that is to be run
    }
    
    public int PlaceOrder(bool acceptTAndCs) // This method is the one to call when the user clicks the Purchase button.
    {
        var checkoutService = new CheckoutCookieService(_basketCookie.GetValue()); // Checkout- CookieService is a class that encodes/decodes the basket data.

        var order = _runner.RunAction(new PlaceOrderInDto(acceptTAndCs, checkoutService.UserId, checkoutService.LineItems)); // Runs the business logic with the data it needs from the basket cookie

        if (_runner.HasErrors) // If the business logic has errors, it returns immediately. The basket cookie is not cleared.
        {
            return 0;
        }
        
        // The order was placed successfully, so it clears the basket cookie.
        checkoutService.ClearAllLineItems();
        _basketCookie.AddOrUpdateCookie(checkoutService.EncodeForCookie());

        return order.OrderId; // Returns the OrderId, which allows ASP.NET to confirm the order details to the user
    }
}
````

In addition to running the business logic, this class acts as an Adapter pattern; it transforms the data from the basket cookie into a form that the business logic accepts, and on a successful completion, it extracts the `Order` entity class's primary key, `OrderId`, to send back to the ASP.NET Core presentation layer.

This Adapter-pattern role is typical of the code that calls the business logic because a mismatch often occurs between the presentation layer format and the business logic format. This mismatch can be small, as in this example, but you’re likely to need to do some form of adaptation in all but the simplest calls to your business logic. That situation is why my more-sophisticated EfCore.GenericBizRunner library has a built-in Adapter pattern feature.



![](./diagrams/svg/04_03_placing_an_order_in_the_book_app.drawio.svg)



From the click of the Purchase button, the ASP.NET Core action, `Place-Order`, in the `CheckoutController` is executed. This action creates a class called `PlaceOrderService` in the service layer, which holds most of the Adapter pattern logic. The caller provides that class with read/write access to the cookies, as the checkout data is held in an HTTP cookie on the user’s device.

`PlaceOrder` method extracts the checkout data from the HTTP cookie and creates a DTO in the form that the business logic needs. Then it calls the generic `BizRunner` to run the business logic that it needs to execute. When the `BizRunner` has returned from the business logic, two routes are possible:

* *The order was successfully placed (no errors).* In this case, the `PlaceOrder` method cleared the basket cookie and returned the `OrderId` of the placed order, so the ASP.NET Core code could show a confirmation page with a summary of the order.
* *The order was unsuccessful (errors present).* In this case, the `PlaceOrder` method returned immediately to the ASP.NET Core code, which detected errors, redisplayed the checkout page, and added the error messages so that the user could rectify the errors and try again.



#### The pros and cons of the complex business logic pattern

**ADVANTAGES OF THIS PATTERN**

This pattern follows the DDD approach, which is well respected and widely used. It keeps the business logic "pure" in that it doesn’t know about the database, which has been hidden via the `BizDbAccess` methods that provide a per-business logic repository. Also, the `BizDbAccess` class allows you to test your business logic without using a database, as your unit tests can provide a replacement class (known as a stub or mock) that can provide test data as required.



**DISADVANTAGES OF THIS PATTERN**

The key disadvantage is you have to write more code to separate the business logic from the database accesses, which takes more time and effort. If the business logic is simple, or if most of the code works on the database, the effort of creating a separate class to handle database accesses isn’t worthwhile.



### Simple business logic example: ChangePriceOfferService

For simple business logic, you are going to build business logic to handle the addition or removal of a price promotion for a book. This example has business rules, but as you will see, those rules are bound up with a lot of database accesses. The rules are

* If the `Book` has a `PriceOffer`, the code should delete the current `PriceOffer` (remove the price promotion).
* If the Book doesn’t have a `PriceOffer`, we add a new price promotion.
* If the code is adding a price promotion, the `PromotionalText` must not be null or empty.



The `ChangePriceOfferService` class contains two methods: a `GetOriginal` method, which is a simple CRUD command to load the `PriceOffer`, and an `AddRemovePriceOffer` method that handles the creation or removal of the `PriceOffer` class for a `Book`. The second method contains business logic.

````c#
// AddRemovePriceOffer method in ChangePriceOfferService
public ValidationResult AddRemovePriceOffer(PriceOffer promotion) // This method deletes a PriceOffer if present; otherwise, it adds a new PriceOffer.
{
    var book = _context.Books
        .Include(r => r.Promotion)
        .Single(k => k.BookId == promotion.BookId);

    if (book.Promotion != null) // If the book has an existing Promotion, removes that promotion
    {
        _context.Remove(book.promotion);
        _context.SaveChanges();
        return null; // Returns null, which means that the method finished successfully
    }

    if (string.IsNullOrEmpty(promotion.PromotionalText)) // Validation the check. The PromotionalText must contain some text.
    {
        return new ValidationResult(
            "This field cannot be empty",
            new []{ nameof(PriceOffer.PromotionalText)}); // Returns an error message, with the property name that was incorrect
    }

    book.Promotion = promotion; // Assigns the new PriceOffer to the selected book
    _context.SaveChanges(); // The SaveChanges method updates the database.
    return null; // The addition of a new price promotion was successful, so the method returns null.
}
````



#### The pros and cons of this business logic pattern

**ADVANTAGES OF THIS PATTERN**

This pattern has little or no set structure, so you can write the code in the simplest way to archive the required business goal. Normally, the code will be shorter than the complex business pattern, which has extra classes to isolate the business logic from the database.

The business logic is also self-contained, with all the code in one place. Unlike the complex business logic example, this business logic handles everything. It doesn’t need a `BizRunner` to execute it, for example, because the code calls `SaveChanges` itself, making it easier to alter, move, and test because it doesn’t rely on anything else.

Also, by putting the business logic classes in the service layer, we can group these simple business logic services in the same folder as the CRUD services related to this business feature. As a result, we can find all the basic code for a feature quickly, because the complex business code is in another project.



**DISADVANTAGES OF THIS PATTERN**

You don’t have the DDD-inspired approach of the complex business logic pattern to guide you, so the onus is on you to design the business logic in a sound way. Your experience will aid you in picking the best pattern to use and writing the correct code. Simplicity is the key here. If the code is easy to follow, you got it right; otherwise, the code is too complex and needs to follow the complex business logic pattern.



### Validation business logic example: Adding review to a book, with checks

* The `NumStars` property must be between 0 and 5.
* The `Comment` property should have some text in it.

````c#
// The improved CRUD code with business validation checks added
public IStatusGeneric AddReviewWithChecks(Review review) // This method adds a review to a book, with validation checks on the data.
{
    var status = new StatusGenericHandler(); // Creates a status class to hold any errors (The IStatusGeneric interface and StatusGenericHandler class come from a NuGet package called GenericServices.StatusGeneric.
    if (review.NumStars < 0 || review.NumStars > 5) // Adds an error to the status if the star rating is in the correct range
    {
        status.AddError("This must be between 0 and 5.", nameof(Review.NumStars));
    }

    if (string.IsNullOrWhiteSpace(review.Comment)) // This second check ensures that the user provided some sort of comment.
    {
        status.AddError("Please provide a comment with your review.", nameof(Review.Comment));
    }

    if (!status.IsValid) // If there are any errors, the method returns immediately with those errors.
    {
        return status;
    }

    var book = _context.Books
        .Include(r => r.Reviews)
        .Single(k => k.BookId == review.BookId);
    book.Reviews.Add(review);
    _context.SaveChanges();
    return status; // Returns the status, which will be valid if no errors were found
}
````

This method is a CRUD method with business validation added, which is typical of this type of business logic. In this case, you used `if-then` code to check the property, but you could use `DataAnnotations` instead. This type of validation is typically done in the frontend, but duplicating the validation of sensitive data in the backend code can make the application more robust.



#### The pros and cons of this business logic pattern

**ADVANTAGES OF THIS PATTERN**

These validation business logic classes to be the same as CRUD services with some extra checks in them.



**DISADVANTAGES OF THIS PATTERN**

The only disadvantage is that you need to do something with the status that the pattern returns, such as redisplaying the input form with an error message. But that’s the downside of providing extra validation rather than the validation business logic design.



### Adding extra features to your business logic handling

This pattern for handling business logic makes it easier to add extra features to your business logic handling. In this section, you’ll add two features:

* Entity class validation to `SaveChanges`
* Transactions that daisy-chain a series of business logic code

These features use EF Core commands that aren’t limited to business logic. Both features could be used in other areas, so you might want to keep them in mind when you’re working on your application.



#### Validating the data that you write to the database

The business logic contains lots of validation code, and it’s often useful to move this code into the entity classes as a validation check, especially if the error is related to a specific property in the entity class. This example is another case of breaking a complex set of rules into several parts.



The `LineItem` entity class has two types of validation added. The first type is a `[Range(min,max)]` attribute, known as Data Annotations, which is added to the `LineNum` property. The second validation method to apply is the `IValidatableObject` interface. This interface requires you to add a method called `IValidatableObject.Validate`, in which you can write your own validation rules and return errors if those rules are violated.

````c#
// Validation rules applied to the LineNum entity class
public class LineItem : IValidatableObject // The IValidatableObject interface adds a IValidatableObject.Validate method.
{
    public int LineItemId { get; set; }
    
    [Range(1,5, ErrorMessage = "This order is over the limit of 5 books.")] // Adds an error message if the LineNum property is not in range
    public byte LineNum { get; set; }

    public short NumBooks { get; set; }

    public decimal BookPrice { get; set; }

    // relationships

    public int OrderId { get; set; }
    public int BookId { get; set; }

    public Book ChosenBook { get; set; }

    IEnumerable<ValidationResult> IValidatableObject.Validate(ValidationContext validationContext) // The IValidatableObject interface requires this method to be created.
    {
        var currContext = validationContext.GetService(typeof(DbContext)); // Allows access to the current DbContext if necessary to get more information
        
        if (ChosenBook.Price < 0) // Moves the Price check out of the business logic into this validation
        {
            yield return new ValidationResult($"Sorry, the book '{ChosenBook.Title}' is not for sale.");
        }

        if (NumBooks > 100) // Extra validation rule: an order for more than 100 books needs to phone in an order.
        {
            yield return new ValidationResult("If you want to order a 100 or more books please phone us on 01234-5678-90", new[] { nameof(NumBooks) }); // Returns the name of the property with the error to provide a better error message
        }
    }
}
````

After adding the validation rule code to your `LineItem` entity class, you need to add a validation stage to EF Core’s `SaveChanges` method, called `SaveChangesWithValidation`. Although the obvious place to put this stage is inside the application’s `DbContext`, you’ll create an extension method instead. This method will allow `SaveChangesWithValidation` to be used on any `DbContext`, which means that you can copy this class and use it in your application.

````c#
// SaveChangesWithValidation added to the application’s DbContext
public static ImmutableList<ValidationResult> // SaveChangesWithValidation returns a list of ValidationResults.
    SaveChangesWithValidation(this DbContext context) // SaveChangesWithValidation is an extension method that takes the DbContext as its input.
{
    var result = context.ExecuteValidation(); // The ExecuteValidation is used in SaveChangesWithChecking/SaveChangesWithCheckingAsync.

    if (result.Any()) // If there are errors, return them immediately and don’t call SaveChanges.
    {
        return result;
    }

    context.SaveChanges(); // There aren’t any errors, so I am going to call SaveChanges.

    return result; // Returns the empty set of errors to signify that there are no errors
}
````

````c#
// SaveChangesWithValidation calls ExecuteValidation method
private static ImmutableList<ValidationResult> ExecuteValidation(this DbContext context)
{
    var result = new List<ValidationResult>();
    foreach (var entry in context.ChangeTracker.Entries() // Uses EF Core’s ChangeTracker to get access to all the entity classes it is tracking
             .Where(e => (e.State == EntityState.Added) || (e.State == EntityState.Modified))) // Filters the entities that will be added or updated in the database
    {
        var entity = entry.Entity;
        var valProvider = new ValidationDbContextServiceProvider(context); // Implements the IServiceProvider interface and passes the DbContext to the Validate method
        var valContext = new ValidationContext(entity, valProvider, null);
        var entityErrors = new List<ValidationResult>();
        if (!Validator.TryValidateObject(entity, valContext, entityErrors, true)) // The Validator.TryValidateObject is the method that validates each class.
        {
            result.AddRange(entityErrors); // Any errors are added to the list.
        }
    }
    return result.ToImmutableList(); // Returns the list of all the errors found (empty if there are no errors)
}
````

The main code is in the `ExecuteValidation` method, because you need to use it in sync and async versions of `SaveChangesWithValidation`. The call to `context.ChangeTracker.Entries` calls the DbContext’s `DetectChanges` to ensure that all the changes you’ve made are found before the validation is run. Then the code looks at all the entities that have been added or modified (updated) and validates them all.

The `ValidationDbContextServiceProvider` class is, which implements the `IServiceProvider` interface. This class is used when you create `ValidationContext`, so it is available in any entity classes that have the `IValidatableObject` interface, allowing the `Validate` method to access the current application’s DbContext if necessary. Having access to the current DbContext allows you to create better error messages by obtaining extra information from the database.

You design the `SaveChangesWithValidation` method to return the errors rather than throw an exception. You do this to fit in with the business logic, which returns errors as a list, not an exception. You can create a new `BizRunner` variant, `RunnerWriteDbWithValidation`, that uses `SaveChangesWithValidation` instead of the normal `SaveChanges` and returns errors from the business logic or any validation errors found when writing to the database.

````c#
// BizRunner variant RunnerWriteDbWithValidation
public class RunnerWriteDbWithValidation<TIn, TOut>
{
    private readonly IBizAction<TIn, TOut> _actionClass;
    private readonly EfCoreContext _context;

    public IImmutableList<ValidationResult> Errors { get; private set; }
    public bool HasErrors => Errors.Any(); // This version needs its own Errors/HasErrors properties, as errors come from two sources.

    public RunnerWriteDbWithValidation(IBizAction<TIn, TOut> actionClass, EfCoreContext context) // Handles business logic that conforms to the IBizAction<TIn, TOut> interface
    {
        _context = context;
        _actionClass = actionClass;
    }

    public TOut RunAction(TIn dataIn) // This method is called to execute the business logic and handle any errors.
    {
        var result = _actionClass.Action(dataIn); // Runs the business logic
        Errors = _actionClass.Errors; // Any errors from the business logic are assigned calls to the local errors list.
        if (!HasErrors) // If no errors, calls SaveChangesWithChecking
        {
            Errors = _context.SaveChangesWithValidation().ToImmutableList(); // Any validation errors are assigned to the Errors list.
        }
        return result; // Returns the result that the business logic returned
    }
}
````

The nice thing about this new variant of the `BizRunner` pattern is that it has exactly the same interface as the original, nonvalidating `BizRunner`. You can substitute `RunnerWriteDbWithValidation<TIn, TOut>` for the original `BizRunner` without needing to change the business logic or the way that the calling method executes the `BizRunner`.



#### Using transactions to daisy-chain a sequence of business logic code

Business logic can get complex. When it comes to designing and implementing a large or complex piece of business logic, you have three options:

* Option 1 - Write one big method that does everything.
* Option 2 - Write a few smaller methods, with one overarching method to run them in sequence.
* Option 3 - Write a few smaller methods, each of which updates the database, but combine them into one Unit Of Work.

Option 1 normally isn’t a good idea because the method will be so hard to understand and refactor. It also has problems if parts of the business logic are used elsewhere, because you could break the DRY (don’t repeat yourself) software principle.

Option 2 can work but can have problems if later stages rely on database items written by earlier stages, which could break the atomic unit rule: when there are multiple changes to the database, they all succeed, or they all fail.

This leaves option 3, which is possible because of a feature of EF Core (and most relational databases) called transactions. "Why you should call `SaveChanges` only once at the end of your changes" introduced the Unit Of Work and showed how `SaveChanges` saves all the changes inside a transaction to make sure that all the changes were saved or, if the database rejected any part of the change, that no changes were saved to the database.

In this case, you want to spread the Unit Of Work over several smaller methods; let’s call them `Biz1`, `Biz2`, and `Biz3`. You don’t have to change `Biz` methods; they still think that they are working on their own and will expect `SaveChanges` to be called when each `Biz` method finishes. But when you create an overarching transaction, all three `Biz` methods, with their `SaveChanges` call, will work as one Unit Of Work. As a result, a database rejection/error in `Biz3` will reject any database changes made by `Biz1`, `Biz2`, and `Biz3`.

This database rejection works because when you use EF Core to create an explicit relational database transaction, it has two effects:

* Any writes to the database are hidden from other database users until you call the transaction’s `Commit` method.
* If you decide that you don’t want the database writes (say, because the business logic has an error), you can discard all database writes done in the transaction by calling the transaction `RollBack` command.



![](./diagrams/images/04_01_transaction.png)



````c#
// RunnerTransact2WriteDb running two business logic stages in series
public class RunnerTransact2WriteDb<TIn, TPass, TOut> // The three types are input, class passed from Part1 to Part2, and output.
    where TOut : class // The BizRunner can return null if there are errors, so it has to be a class.
{
    // Defines the generic BizAction for the two business logic parts
    private readonly IBizAction<TIn, TPass> _actionPart1;
    private readonly IBizAction<TPass, TOut> _actionPart2;
    private readonly EfCoreContext _context;
    
    // Holds any error information returned by the business logic
    public IImmutableList<ValidationResult> Errors { get; private set; }
    public bool HasErrors => Errors.Any();

    public RunnerTransact2WriteDb(
        EfCoreContext context,
        IBizAction<TIn, TPass> actionPart1,
        IBizAction<TPass, TOut> actionPart2) // The constructor takes both business classes and the application DbContext.
    {
        _context = context;
        _actionPart1 = actionPart1;
        _actionPart2 = actionPart2;
    }

    public TOut RunAction(TIn dataIn)
    {
        using (var transaction = _context.Database.BeginTransaction()) // Starts the transaction within a using statement
        {
            var passResult = RunPart(_actionPart1, dataIn); // The private method, RunPart, runs the first business part.
            if (HasErrors) // If there are errors, returns null. (The rollback is handled by the dispose.)
            {
                return null;
            }
            var result = RunPart(_actionPart2, passResult); // If the first part of the business logic was successful, runs the second business logic

            if (!HasErrors) // If there are no errors, commits the transaction to the database
            {
                transaction.Commit();
            }
            return result; // Returns the result from the last business logic
        } // If commit is not called before the using end, RollBack undoes all the changes.
    }

    private TPartOut RunPart<TPartIn, TPartOut>(IBizAction<TPartIn, TPartOut> bizPart, TPartIn dataIn)
        where TPartOut : class // This private method handles running each part of the business logic.
    {
        // Runs the business logic and copies the business logic’s Errors
        var result = bizPart.Action(dataIn);
        Errors = bizPart.Errors;
        if (!HasErrors) // If the business logic was successful, calls SaveChanges
        {
            _context.SaveChanges();
        }
        return result; // Returns the result from the business logic it ran
    }
}
````

In `RunnerTransact2WriteDb` class, you execute each part of the business logic in turn, and at the end of each execution, you do one of the following:

* *No errors* - You call `SaveChanges` to save to the transaction any changes that business logic has run. That save is within a local transaction, so other methods accessing the database won’t see those changes yet. Then you call the next part of the business logic, if there is one.
* *Has errors* - You copy the errors found by the business logic that just finished to the BizRunner error list and exit the `BizRunner`. At that point, the code steps outside the using clause that holds the transaction, which causes disposal of the transaction. Because no transaction `Commit` has been called, the disposal will cause the transaction to execute its `RollBack` method, which discards the database writes to the transaction. Those writes are never written to the database.

If you’ve run all the business logic with no errors, you call the `Commit` command on the transaction. This command does an atomic update of the database to reflect all the changes to the database that are contained in the local transaction.



#### Using the RunnerTransact2WriteDb class

To test the `RunnerTransact2WriteDb` class, you’ll split the order-processing code you used earlier into two parts:

* *PlaceOrderPart1* - Creates the Order entity, with no `LineItems`
* *PlaceOrderPart2* - Adds the `LineItems` for each book bought to the `Order` entity that was created by the `PlaceOrderPart1` class

`PlaceOrderPart1` and `PlaceOrderPart2` are based on the `PlaceOrderAction` code you’ve already seen, so we don’t repeat the business code here. The code changes that are required for `PlaceOrderService` to change over to use the `RunnerTransact2WriteDb BizRunner`. The listing focuses on the part that creates and runs the two stages, `Part1` and `Part2`, with the unchanged parts of the code left out so you can see the changes easily.

````c#
// The PlaceOrderServiceTransact class showing the changed parts
public class PlaceOrderServiceTransact // This version of PlaceOrderService uses transactions to execute two business logic classes: PlaceOrderPart1 and PlaceOrderPart2.
{
    //… code removed as the same as in listing 4.5

    public PlaceOrderServiceTransact(
        IRequestCookieCollection cookiesIn,
        IResponseCookies cookiesOut,
        EfCoreContext context)
    {
        _checkoutCookie = new CheckoutCookie(cookiesIn, cookiesOut);
        _runner = new RunnerTransact2WriteDb // This BizRunner handles multiple business logic inside a transaction.
            <PlaceOrderInDto, Part1ToPart2Dto, Order>( // The BizRunner needs the input, the class passed from Part1 to Part2, and the output.
            context, // The BizRunner needs the application’s DbContext.
            new PlaceOrderPart1(new PlaceOrderDbAccess(context)), // Provides an instance of the first part of the business logic
            new PlaceOrderPart2(new PlaceOrderDbAccess(context))); // Provides an instance of the second part of the business logic
    }

    public int PlaceOrder(bool tsAndCsAccepted)
    {
        //… code removed as the same as in listing 4.6
    }
}
````

The important thing to note is that the business logic has no idea whether it’s running in a transaction. You can use a piece of business logic on its own or as part of a transaction. Only the caller of transaction-based business logic, which I call the `BizRunner`, needs to change. Using a transaction makes it easy to combine multiple business logic classes under one transaction without needing to change any of your business logic code.

The advantage of using transactions like this one is that you can split and/or reuse parts of your business logic while making these multiple business logic calls look to your application, especially its database, like one call. We’ve used this approach when we needed to create and then immediately update a complex, multipart entity. Because we needed the Update business logic for other cases, we used a transaction to call the Create business logic followed by the Update business logic, which saved me development effort and kept my code DRY.

The disadvantage of this approach is that it adds complexity to the database access, which can make debugging a little more difficult, or the use of database transactions could cause a performance issue. Also, be aware that if you use the `EnableRetryOnFailure` option to retry database accessed on errors, you need to handle possible multiple calls to your business logic.



### Summary

* The term *business logic* describes code written to implement real-world business rules. The business logic code can range from the simple to the complex.
* Depending on the complexity of your business logic, you need to choose an approach that balances how easy it is to solve the business problem against the time it takes you to develop and test your solution.
* Isolating the database access part of your business logic into another class/project can make the pure business logic simpler to write but take longer to develop.
* Putting all the business logic for a feature in one class is quick and easy but can make the code harder to understand and test.
* Creating a standardized interface for your business logic makes calling and running the business logic much simpler for the frontend.
* Sometimes, it’s easier to move some of the validation logic into the entity classes and run the checks when that data is being written to the database.
* For business logic that’s complex or being reused, it might be simpler to use a database transaction to allow a sequence of business logic parts to be run in sequence but, from the database point of view, look like one atomic unit.





## 5. Using EF Core in ASP.NET Core web applications

* Using EF Core in ASP.NET Core
* Using dependency injection in ASP.NET Core
* Accessing the database in ASP.NET Core MVC actions
* Using EF Core migrations to update a database
* Using async/await to improve scalability



### Understanding the architecture of the Book App

![](./diagrams/images/05_01_architecture.png)



### Understanding dependency injection

*Dependency injection* is a way to link together your application dynamically.

Using DI has lots of benefits, and here are the main ones:

* DI allows your application to link itself dynamically. The DI provider will work out what classes you need and create them in the right order. If one of your classes needs the application’s DbContext, for example, the DI can provide it.
* Using interfaces and DI together means that your application is more loosely coupled; you can replace a class with another class that matches the same interface. This technique is especially useful in unit testing: you can provide a replacement version of the service with another, simpler class that implements the interface (called *stubbing* or *mocking* in unit tests).
* Other, more advanced features exist, such as using DI to select which class to return based on certain settings. If you’re building an e-commerce application, in development mode, you might want to use a dummy credit card handler instead of the normal credit card system.



````c#
const string connection = "Data Source=(localdb)\\mssqllocaldb;Database=EfCoreInActionDb.Chapter02;Integrated Security=True;";
var optionsBuilder = new DbContextOptionsBuilder<EfCoreContext>();

optionsBuilder.UseSqlServer(connection);
var options = optionsBuilder.Options;

using (var context = new EfCoreContext(options))
{…
````

That code works but has a few problems. First, you’re going to have to repeat this code for each database access you make. Second, this code uses a fixed database access string, referred to as a *connection string*, which isn’t going to work when you want to deploy your site to a host, because the database location for the hosted database will be different from the database you use for development.



#### The lifetime of a service created by DI

![](./diagrams/images/05_02_dependency_injection_lifetime.png)



### Making the application’s DbContext available via DI

#### Providing information on the database’s location

The location and various database configuration settings are typically stored as a *connection string*.

* appsetting.json - Holds the settings that are common to development and production
* appsettings.Development.json - Holds the settings for the development build
* appsettings.Production.json - Holds the settings for the production build (when the web application is deployed to a host for users to access it)

````json
{
    "ConnectionStrings": {
        "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=EfCoreInActionDb;Trusted_Connection=True"
    },
    … other parts removed as not relevant to database access
}
````



#### Registering your application’s DbContext with the DI provider

The application’s DbContext for ASP.NET Core has a constructor that takes a `DbContextOptions<T>` parameter defining the database options. That way, the database connection string can change when you deploy your web application.

````c#
public class EfCoreContext : DbContext
{
    //… properties removed for clarity

    public EfCoreContext(
        DbContextOptions<EfCoreContext> options)
        : base(options) {}
    
    //… other code removed for clarity
}
````



````c#
// Registering your DbContext in ASP.NET Core’s Startup class
public void ConfigureServices(IServiceCollection services) // This method in the Startup class sets up services.
{
    services.AddControllersWithViews(); // Sets up a series of services to use with controllers and Views

    var connection = Configuration.GetConnectionString("DefaultConnection"); // You get the connection string from the appsettings.json file, which can be changed when you deploy.
    
    services.AddDbContext<EfCoreContext>(options => options.UseSqlServer(connection)); // Configures the application’s DbContext to use SQL Server and provide the connection

    //… other service registrations removed
}
````

When you use the type `EfCoreContext` in places where DI intercepts, the DI provider will create an instance of the application’s `DbContext`, using the `DbContextOptions<EfCoreContext>` options. Or if you ask for multiple instances in the same HTTP request, the DI provider will return the same instances.



#### Registering a DbContext Factory with the DI provider

Typically, you use the `AddDbContextFactory` method only with Blazor in the frontend or in applications where you cannot control the parallel access to the same application’s DbContext, which breaks the thread-safe rule. Many other applications, such as ASP.NET Core, manage parallel accesses for you, so you can obtain an instance of the application’s DbContext via DI.

````c#
// Registering a DbContext factory in ASP.NET Core’s Startup class
public void ConfigureServices(IServiceCollection services) // This method in the Startup class sets up services.
{
    services.AddControllersWithViews(); // Sets up a series of services to use with controllers and Views

    var connection = Configuration.GetConnectionString("DefaultConnection"); // You get the connection string from the appsettings.json file, which can be changed when you deploy.

    services.AddDbContextFactory<EfCoreContext>(options => options.UseSqlServer(connection)); // Configures the DbContext factory to use SQL Server and provide the connection

    //… other service registrations removed
}
````



### Calling your database access code from ASP.NET Core

![](./diagrams/images/05_03_db_access_from_app.png)



````c#
// The Index action in the HomeController displays the list of books
public class HomeController : Controller
{
    private readonly EfCoreContext _context;

    public HomeController(EfCoreContext context) // The application’s DbContext is provided by ASP.NET Core via DI.
    {
        _context = context;
    }

    public IActionResult Index // ASP.NET action, called when the home page is called up by the user
        (SortFilterPageOptions options) // The options parameter is filled with sort, filter, and page options via the URL.
    {
        var listService = new ListBooksService(_context); // ListBooksService is created by using the application’s DbContext from the private field _context.

        var bookList = listService
            .SortFilterPage(options) // The SortFilterPage method is called with the sort, filter, and page options provided.
            .ToList(); // The ToList() method executes the LINQ commands, causing EF Core to translate the LINQ into the appropriate SQL to access the database and return the result as a list.

        return View(new BookListCombinedDto(options, bookList)); // Sends the options (to fill in the controls at the top of the page) and the list of BookListDtos to display as an HTML table
    }
}
````



#### Improving registering your database access classes as services

The `NetCore.AutoRegisterDi` library scans one or more assembles; looks for standard public, nongeneric classes that have public interfaces; and registers them with NET Core’s DI provider. It has some simple filtering and some lifetime-setting capabilities, but not much more. But this simple piece of code gives you two benefits over manually registering your classes/interfaces with the DI provider:

* It saves you time because you don’t have to register every interface/class manually.
* More important, it automatically registers your interfaces/classes so that you don’t forget.

````c#
// Using NetCore.AutoRegisterDi to register classes as DI services
// You can get references to the assemblies by providing a class that is in that assembly.
var assembly1ToScan = Assembly.GetAssembly(typeof(ass1Class));
var assembly2ToScan = Assembly.GetAssembly(typeof(ass2Class));

service.RegisterAssemblyPublicNonGenericClasses(assembly1ToScan, assembly2ToScan) // This method takes zero to many assemblies to scan. If no assembly is provided, it will scan the calling assembly.
    .Where(c => c.Name.EndsWith("Service")) // This optional filter system allows you to filter the classes that you want to register.
    .AsPublicImplementedInterfaces(); // Registers all the classes that have public interfaces. By default, the services are registered as transient, but you can change that registration by adding a ServiceLifetime parameter or attributes.
````

````c#
// Extension method in ServiceLayer that handles all the DI service registering
public static class NetCoreDiSetupExtensions
{
    public static void RegisterServiceLayerDi // This class is in the ServiceLayer.
        (this IServiceCollection services) // The NetCore.AutoRegisterDi library understands NET Core DI, so you can access the IServiceCollection interface.
    {
        services.RegisterAssemblyPublicNonGenericClasses() // Calling the RegisterAssemblyPublicNonGenericClasses method without a parameter means that it scans the calling assembly.
            .AsPublicImplementedInterfaces(); // This method will register all the public classes with interfaces with a Transient lifetime.
    }
}
````

````c#
// Calling all your registration methods in the projects that need them
public void ConfigureServices(IServiceCollection services) // This method in the Startup class sets up services for ASP.NET Core.
{
    //… other registrations left out

    services.RegisterBizDbAccessDi();
    services.RegisterBizLogicDi();
    services.RegisterServiceLayerDi();
}
````



### Using EF Core’s migration feature to change the database’s structure

````c#
// ASP.NET Core Program class, including a method to migrate the database
public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build(); // This call runs the Startup.Configure method, which sets up the DI services you need to setup/migrate your database.
        await host.MigrateDatabaseAsync(); // Calls your extension method to migrate your database
        await host.RunAsync(); // At the end, you start the ASP.NET Core application.
    }
    //… other code not shown
}
````

````c#
// The MigrateDatabaseAsync extension method to migrate the database
public static async Task MigrateDatabaseAsync(this IHost webHost) // Creates an extension method that takes in IHost
{
    using (var scope = webHost.Services.CreateScope()) // Creates a scoped service provider. After the using block is left, all the services will be unavailable. This approach is the recommended way to obtain services outside an HTTP request.
    {
        // Creates an instance of the application’s DbContext that has a lifetime of only the outer using statement
        var services = scope.ServiceProvider;
        using (var context = services.GetRequiredService<EfCoreContext>())
        {
            try
            {
                await context.Database.MigrateAsync(); // Calls EF Core’s MigrateAsync command to apply any outstanding migrations at startup
                // You can add a method here to handle complex seeding of the database if required.
                //Put any complex database seeding here
            }
            catch (Exception ex) // If an exception occurs, you log the information so that you can diagnose it.
            {
                var logger = services.GetRequiredService<ILogger<Program>>();
                logger.LogError(ex, "An error occurred while migrating the database.");
                throw; // Rethrows the exception because you don’t want the application to carry on if a problem with migrating the database occurs
            }
        }
    }
}
````



In addition to migrating the database, you may want to add default data to the database at the same time, especially if it’s empty. This process, called seeding the database, covers adding initial data to the database or maybe updating data in an existing database.

The other option is to run some code when the migration has finished. This option is useful if you have dynamic data or complex updates that the migration seeding can’t handle.

````c#
// Example MigrateAndSeed extension method
public static async Task SeedDatabaseAsync(this EfCoreContext context) // Extension method that takes in the application’s DbContext
{
    if (context.Books.Any()) return; // If there are existing books, you return, as you don’t need to add any.
    
    context.Books.AddRange(EfTestData.CreateFourBooks()); // Database has no books, so you seed it; in this case, you add the default books.
    await context.SaveChangesAsync(); // SaveChangesAsync is called to update the database.
}
````



### Using async/await for better scalability

![](./diagrams/images/05_04_async.png)



````c#
// The async Index action method from the HomeController
public async Task<IActionResult> Index(SortFilterPageOptions options) // You make the Index action method async by using the async keyword, and the returned type has to be wrapped in a generic task.
{
    var listService = new ListBooksService(_context);
    
    public async Task<IActionResult> Index(SortFilterPageOptions options)
    {
        var listService = new ListBooksService(_context);
        
        var bookList = await listService // You must await the result of the ToListAsync method, which is an async command.
            .SortFilterPage(options)
            .ToListAsync(); // You can change SortFilterPage to async by replacing .ToList() with .ToListAsync().
        
        return View(new BookListCombinedDto(options, bookList));
    }
}
````



### Running parallel tasks: How to provide the DbContext

Parallel tasks are useful in various scenarios. Say you’re accessing multiple, external sources that you need to wait for before they return a result. By using multiple tasks running in parallel, you gain performance improvements. In another scenario, you might have a long-running task, such as processing order fulfillment in the background. You use parallel tasks to avoid blocking the normal flow and making your website look slow and unresponsive.



![](./diagrams/images/05_05_parallel.png)



If you want to run any code that uses EF Core in parallel, you can’t use the normal approach of getting the application’s DbContext because EF Core’s DbContext isn’t thread-safe; you can’t use the same instance in multiple tasks. EF Core will throw an exception if it finds that the same DbContext instance is used in two tasks.

In ASP.NET Core, the correct way to get a DbContext to run in the background is by using a DI scoped service. This scoped service allows you to create, via DI, a DbContext that’s unique to the task that you’re running. To do this, you need to do three things:

* Get an instance of the `IServiceScopeFactory` via constructor injection.
* Use the `IServiceScopeFactory` to a *scoped DI service*.
* Use the scoped DI service to obtain an instance of the application’s DbContext that is unique to this scope.

The following listing shows the method in your background task that uses the `IServiceScopeFactory` to obtain a unique instance of your application’s DbContext. This method counts the number of `Reviews` in the database and logs that number.



````c#
// The method inside the background service that accesses the database
private async Task DoWorkAsync(CancellationToken stoppingToken) // The IHostedService will call this method when the set period has elapsed.
{
    using (var scope = _scopeFactory.CreateScope()) // Uses the ScopeProviderFactory to create a new DI scoped provider
    {
        var context = scope.ServiceProvider.GetRequiredService<EfCoreContext>(); // Because of the scoped DI provider, the DbContext instance created will be different from all the other instances of the DbContext.
        var numReviews = await context.Set<Review>()
            .CountAsync(stoppingToken);
        _logger.LogInformation("Number of reviews: {numReviews}", numReviews);
    }
}
````

The important point of the code is that you provide `ServiceScopeFactory` to each task so that it can use DI to get a unique instance of the DbContext (and any other scoped services). In addition to solving the DbContext thread-safe issue, if you are running the method repeatedly, it’s best to have a new instance of the application’s DbContext so that data from the last run doesn’t affect your next run.



#### Running a background service in ASP.NET Core

````c#
// An ASP.NET Core background service that calls DoWorkAsync every hour
public class BackgroundServiceCountReviews : BackgroundService // Inheriting the BackgroundService class means that this class can run continuously in the background.
{
    private static TimeSpan _period = new TimeSpan(0,1,0,0); // Holds the delay between each call to the code to log the number of reviews

    // The IServiceScopeFactory injects the DI service that you use to create a new DI scope.
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<BackgroundServiceCountReviews> _logger;

    public BackgroundServiceCountReviews(
        IServiceScopeFactory scopeFactory,
        ILogger<BackgroundServiceCountReviews> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken) // The BackgroundService class has a ExecuteAsync method that you override to add your own code.
    {
        while (!stoppingToken.IsCancellationRequested) // This loop repeatably calls the DoWorkAsync method, with a delay until the next call is made.
        {
            await DoWorkAsync(stoppingToken);
            await Task.Delay(_period, stoppingToken);
        }
    }

    private async Task DoWorkAsync…
    //see listing 5.16
}
````

You need to register your background class with the NET DI provider, using the `AddHostedService` method. When the Book App starts, your background task will be run first, but when your background task gets to a place where it calls an async method and uses the `await` statement, control goes back to the ASP.NET Core code, which starts up the web application.



#### Other ways of obtaining a new instance of the application’s DbContext

Although DI is the recommended method to get the application’s DbContext, in some cases, such as a console application, DI may not be configured or available. In these cases, you have two other options that allow you to obtain an instance of the application’s DbContext:

* Move your configuration of the application’s DbContext by overriding the `OnConfiguring` method in the DbContext and placing the code to set up the DbContext there.
* Use the same constructor used for ASP.NET Core and manually inject the database options and connection string, as you do in unit tests.

The downside of the first option is it uses a fixed connection string, so it always accesses the same database, which could make deployment to another system difficult if the database name or options change. The second option - providing the database options manually - allows you to read in a connection string from the appsettings.json or a file inside your code.

Another issue to be aware of is that each call will give you a new instance of the application’s DbContext. At times you might want to have the same instance of the application’s DbContext to ensure that tracking changes works. You can work around this issue by designing your application so that one instance of the application’s DbContext is passed between all the code that needs to collaborate on database updates.



### Summary

* ASP.NET Core uses dependency injection (DI) to provide the application’s DbContext. With DI, you can dynamically link parts of your application by letting DI create class instances as required.
* The `ConfigureServices` method in ASP.NET Core’s `Startup` class is the place to configure and register your version of the application’s DbContext by using a connection string that you place in an ASP.NET Core application setting file.
* To get an instance of the application’s DbContext to use with your code via DI, you can use constructor injection. DI will look at the type of each of the constructor’s parameters and attempt to find a service for which it can provide an instance.
* Your database access code can be built as a service and registered with the DI. Then you can inject your services into the ASP.NET Core action methods via parameter injection: the DI will find a service that finds the type of an ASP.NET Core action method’s parameter that’s marked with the attribute `[FromServices]`.
* Deploying an ASP.NET Core application that uses a database requires you to define a database connection string that has the location and name of the database on the host.
* EF Core’s migration feature provides one way to change your database if your entity classes and/or the EF Core configuration change. The `Migrate` method has some limitations when used on cloud hosting sites that run multiple instances of your web application.
* Async/await tasking methods on database access code can make your website handle more simultaneous users, but performance could suffer, especially on simple database accesses.
* If you want to use parallel tasks, you need to provide a unique instance of the application’s DbContext by creating a new scoped DI provider.





## 6. Tips and techniques for reading and writing with EF Core

* Selecting the right approach to read data from the database
* Writing queries that perform well on the database side
* Avoiding problems when you use Query Filters and special LINQ commands
* Using AutoMapper to write Select queries more quickly
* Writing code to quickly copy and delete entities in the database



### Reading from the database

#### Understanding what AsNoTracking and its variant do

* `AsNoTracking` produces a quicker query time but doesn’t always represent the exact database relationships.
* `AsNoTrackingWithIdentityResolution` typically is quicker than a normal query but slower than the same query with `AsNoTracking`. The improvement is that the database relationships are represented correctly, with a entity class instance for each row in the database.

The `AsNoTracking` method doesn’t execute the feature called identity resolution that ensures that there is only one instance of an entity per row in the database. Not applying the identity resolution feature to the query means that you might get an extra instances of entity classes.



![](./diagrams/images/06_01_AsNoTracking_vs_AsNoTrackingWithIdentityResolution.png)



If you are using the relationships, such as to create a report of books which linked to other books by the same author, the `AsNoTracking` method might cause a problem. In a case like that one, you should use the `AsNoTrackingWithIdentityResolution` method.



#### Reading in hierarchical data efficiently



![](./diagrams/images/06_02_reading_in_hierarchical_data_efficiently.png)



You could use `.Include(x => x.WorksForMe).ThenInclude(x => x.WorksForMe)` and so on, but a single `.Include(x => x.WorksForMe)` is enough, as the relational fixup can work out the rest. The next listing provides an example in which you want a list of all the employees working in development, with their relationships. The LINQ in this query is translated into one SQL query.



````c#
// Loading all the employees working in development, with their relationships
var devDept = context.Employees // The database holds all the Employees.
    .Include(x => x.WorksForMe) // One Include is all you need; relational fixup will work out what is linked to what.
    .Where(x => x.WhatTheyDo.HasFlag(Roles.Development)) // Filters the employees down to ones who work in development
    .ToList();
````



#### Understanding how the Include method works

The simplest way to load an entity class with its relationships is to use the `Include` method, which is easy to use and normally produces an efficient database access. But it is worth knowing how the `Include` method works and what to watch out for. The `Include` method has a negative effect on performance for some complex queries.



````c#
var query = context.Books
    .Include(x => x.Reviews)
    .Include(x => x.AuthorsLink)
	    .ThenInclude(x => x.Author);
````

![](./diagrams/images/06_03_include_performance.png)



Performance problems occur if you have multiple collection relationships that you want to include in the query, and some of those relationships have a large number of entries in the collection. This figure shows that the number of rows read in via EF Core versions before 3.0 is calculated by adding the rows. But in EF Core 3.0 and later, the number of rows read is calculated by multiplying the rows. Suppose that you are loading 3 relationships, each of which has 100 rows. The 3.0 version of EF Core would read in 100+100+100 = 300 rows, but EF Core 3.0 and later would use 100 * 100 * 100 = 1 million rows.
Fortunately, EF Core 5 provides a method called `AsSplitQuery` that tells EF Core to read each `Include` separately, as in the following listing.

````c#
// Reading relationships separately and letting relational fixup join them up
var result = context.ManyTops
    .AsSplitQuery() // Causes each Include to be loaded separately, thus stopping the multiplication problem
    .Include(x => x.Collection1)
    .Include(x => x.Collection2)
    .Include(x => x.Collection3)
    .Single(x => x.Id == id)
````



If you find that a query that uses multiple `Includes` is slow, it could be because two or more included collections contain a lot of entries. In this case, add the `AsSplitQuery` method before your `Includes` to swap to the separate load of every included collection.



#### Making loading navigational collections fail-safe

For any navigational property that uses a collection, developers assign an empty collection to a collection navigational property, either in the constructor or via an assignment to the property:

````c#
// A entity class with navigational collections set to an empty collection
public class BookNotSafe
{
    public int Id { get; set; }
    public ICollection<ReviewNotSafe> Reviews { get; set; } // This navigational property called Reviews has many entries—that is, a one-to-many relationship.

    public BookNotSafe()
    {
        Reviews = new List<ReviewNotSafe>(); // The navigational property called Reviews is preloaded with an empty collection, making it easier to add ReviewNotSave to the navigational property when the primary entity, BookNotSafe, is created.
    }
}
````

Developers do this to make it easier to add entries to a navigational collection on a newly created instance of an entity class. The downside is that if you forget the Include to load a navigational property collection, you get an empty collection when the database might have data that should fill that collection.

You have another problem if you want to replace the whole collection. If you don’t have the Include, the old entries in the database aren’t removed, so you get a combination of new and old entities, which is the wrong answer. In the following code snippet, instead of replacing the two existing Reviews, the database ends up with three Reviews:

````c#
var book = context.Books
    //missing .Include(x => x.Reviews)
    .Single(p => p.BookId == twoReviewBookId);

book.Reviews = new List<Review>{ new Review{ NumStars = 1}};
context.SaveChanges();
````



Another good reason not to assign an empty collection to a collection is performance. If you need to use explicit loading of a collection, for example, and you know that it’s already loaded because it’s not null, you can skip doing the (redundant) explicit loading.

So we won’t preload any navigational properties with a collection. Instead of failing silently when we leave out the `Include` method, we get a `NullReferenceException` when the code accesses the navigational collection property. That result is much better than getting the wrong data.



#### Using Global Query Filters in real-world situations

**SOFT DELETE IN REAL-WORLD APPLICATIONS**

If you soft-delete a `Book`, for example, the `PriceOffer`, `Reviews`, and `AuthorLinks` are still there, which can cause problems if you don’t think things through. You built a background process that logged the number of `Reviews` in the database on every hour. If you soft-deleted a `Book` that had ten `Reviews`, you might expect the number of `Reviews` to go down, it wouldn’t. You need a way to handle this problem.

A pattern in Domain-Driven Design (DDD) called Root and Aggregates helps you in this situation. In this pattern, the `Book` entity class is the Root, and the `PriceOffer`, `Reviews`, and `AuthorLinks` are Aggregates. This pattern goes on to say you should access Aggregates only via the Root. This process works well with soft deletes because if the `Book` (Root) is soft-deleted, you can’t access its Aggregates. So the correct code for counting all the `Reviews`, taking the soft delete into account, is

````c#
var numReviews = context.Books.SelectMany(x => x.Reviews).Count();
````



Another way to solve the Root/Aggregate problem with soft deletes is to mimic the cascade delete behavior when setting soft deletes, which is quite complex to do.



The second thing to consider is that you shouldn’t apply soft deletes to a one-to-one relationship. You will have problems if you try to add a new one-to-one entity when an existing but soft-deleted entity is already there. If you had a soft-deleted `PriceOffer`, which has a one-to-one relationship with the Book, and tried to add another `PriceOffer` to the `Book`, you would get a database exception. A one-to-one relationship has a unique index on the foreign key `BookId`, and a (soft-deleted) `PriceOffer` was taking that slot.

The soft-delete feature is useful because users can mistakenly delete the wrong data. But being aware of the issues allows you to plan how to handle them in your applications. You can use the Root/Aggregate approach and don’t allow soft deletes of one-to-one dependent entities.



**USING QUERY FILTERS TO CREATE MULTITENANT SYSTEMS**

A multitenant system is one in which different users or groups of users have data that should be accessed only by certain users. Each tenant has a unique DataKey. A tenant might be an individual user or, more likely, a group of users. Software as a Service (SaaS) application that provides stock control for lots of retail companies.

When a user selects a book to buy in the Book App, a basket cookie is created to hold each book in the user’s basket, plus a `UserId`. This basket cookie is used if the user clicks the My Orders menu item to show only the Orders from this user. The following code takes the `UserId` from the basket cookie and uses a Query Filter to return only the Orders that the user created. Two main parts make this code work:

* A `UserIdService` gets the `UserId` from the basket cookie.
* The `IUserIdService` is injected via the application’s DbContext constructor and used to access the current user.



![](./diagrams/images/06_04_multitenant.png)



The following listing shows the `UserIdService` code, which relies on the `IHttpContextAccessor` to access the current HTTP request.

````c#
// UserIdService that extracts the UserId from the basket cookie
public class UserIdService : IUserIdService
{
    // The IHttpContextAccessor is a way to access the current HTTP context. To use it, you need to register it in the Startup class, using the command services.AddHttpContextAccessor().
    private readonly IHttpContextAccessor _httpAccessor;

    public UserIdService(IHttpContextAccessor httpAccessor)
    {
        _httpAccessor = httpAccessor;
    }

    public Guid GetUserId()
    {
        // In some cases, the HTTPContext could be null, such as a background task. In such a case, you provide an empty GUID.
        var httpContext = _httpAccessor.HttpContext;
        if (httpContext == null)
        {
            return Guid.Empty;
        }

        // Uses existing services to look for the basket cookie. If there is no cookie, the code returns an empty GUID.
        var cookie = new BasketCookie(httpContext.Request.Cookies);
        if (!cookie.Exists())
        {
            return Guid.Empty;
        }

        // If there is a basket cookie, creates the CheckoutCookieService, which extracts the UserId and returns it
        var service = new CheckoutCookieService(cookie.GetValue());
        return service.UserId;
    }
}
````

When you have a value to act as a DataKey, you need to provide it to the application’s DbContext. The typical way is via DI constructor injection; the injected service provides a way to get the DataKey. For our example, we are using the `UserId`, taken from the basket cookie, to serve as a DataKey. Then you use that UserId in a Query Filter applied to the `CustomerId` property in the `Order` entity class, which contains the `UserId` of the person who created the `Order`. Any query for `Order` entities will return only `Orders` created by the current user. The following listing shows how to inject the `UserIdService` service into the application’s DbContext and then use that `UserId` in a Query Filter.

````c#
// Book App’s DbContext with injection of UserId and Query Filter
public class EfCoreContext : DbContext
{
    private readonly Guid _userId; // This property holds the UserId used in the Query Filter on the Order entity class.

    public EfCoreContext(DbContextOptions<EfCoreContext> options, // Normal options for setting up the application’s DbContext
                         IUserIdService userIdService = null) // Sets the UserIdService. Note that this parameter is optional, which makes it much easier to use in unit tests that don’t use the Query Filter.
        : base(options)
    {
        _userId = userIdService?.GetUserId()
            ?? new ReplacementUserIdService().GetUserId(); // Sets the UserId. If the UserId is null, a simple replacement version provides the default Guid.Empty value.
    }

    public DbSet<Book> Books { get; set; }
    //… rest of DbSet<T> left out

    protected override void OnModelCreating(ModelBuilder modelBuilder) // The method where you configure EF Core and put your Query Filters
    {
        //… other configuration left out for clarity
        
        modelBuilder.Entity<Book>()
            .HasQueryFilter(p => !p.SoftDeleted); // Soft-delete Query Filter
        modelBuilder.Entity<Order>()
            .HasQueryFilter(x => x.CustomerName == _userId); // Order query filter, which matches the current UserId obtained from the cookie basket with the CustomerId in the Order entity class
    }
}
````

Every instance of the application’s DbContext gets the `UserId` of the current user, or an empty GUID if they never "bought" a book. Whereas the DbContext’s configuration is set up on first use and cached, the lambda Query Filter is linked to a live field called _userId. The query filter is fixed, but the _userId is dynamic and can change on every instance of the DbContext.

But it’s important that the Query Filter not be put in a separate configuration class, because the _userId would become fixed to the `UserId` provided on first use. You must put the lambda query somewhere that it can get the dynamic _userId variable. In this case, we place it in the `OnModelCreating` method inside the application’s DbContext, which is fine.

If you have an ASP.NET Core application that users log in to, you can use `IHttpContextAccessor` to access the current `ClaimPrincipal`. The `ClaimPrincipal` contains a list of `Claims` for the logged-in user, including their `UserId`, which is stored in a claim with the name defined by the system constant `ClaimTypes.NameIdentifier`. Or you could add a new `Claim` to the user on login to provide a DataKey that is used in the Query Filter.



#### Considering LINQ commands that need special attention

EF Core does a great job of mapping LINQ methods to SQL, the language of most relational databases. But three types of LINQ methods need special handling:

* Some LINQ commands need extra code to make them fit the way that the database works, such as the LINQ `Average`, `Sum`, `Max`, and other aggregate commands needed to handle a return of `null`. Just about the only aggregate that won’t return `null` is `Count`.
* Some LINQ commands can work with a database, but only within rigid boundaries because the database doesn’t support all the possibilities of the command. An example is the `GroupBy` LINQ command; the database can have only a simple key, and there are significant limitations on the `IGrouping` part.
* Some LINQ commands have a good match to a database feature, but with some limitations on what the database can return. Examples are `Join` and `GroupJoin`.

The problem is that if you get your LINQ slightly wrong, you will get the *could not be translated* exception. The message might not be too helpful in diagnosing the problem.



**AGGREGATES NEED A NULL (APART FROM COUNT)**

You are likely to use the LINQ aggregates `Max`, `Min`, `Sum`, `Average`, `Count`, and `CountLong`, so here are some pointers:

* The `Count` and `CountLong` methods work fine if you count something sensible in the database, such as a row or relational links such as the number of `Reviews` for a `Book`.
* The LINQ aggregates `Max`, `Min`, `Sum`, and `Average` need a nullable result, such as `context.Books.Max(x => (decimal?)x.Price)`. If the source (`Price` in this example) isn’t nullable, you must have cast to the nullable version of the source. Also, if you are using Sqlite for unit testing, remember that it doesn’t support decimal, so you would get an error even if you used the nullable version.
* You can’t use the LINQ Aggregate method directly on the database because it does a per-row calculation.



**GROUPBY LINQ COMMAND**

When `GroupBy` is used on an SQL database, the `Key` part needs to be a scalar value (or values) because that’s what the SQL `GROUP BY` supports. The `IGrouping` part can be a selection of data, including some LINQ commands. You need to follow a `GroupBy` command with an execute command such as `ToList`. Anything else seems to cause the could not be translated exception.



````c#
var something = await _context.SomeComplexEntity
    .GroupBy(x => new { x.ItemID, x.Item.Name })
    .Select(x => new
    {
        Id = x.Key.ItemID,
        Name = x.Key.Name,
        MaxPrice = x.Max(o => (decimal?)o.Price)
    })
    .ToListAsync();
````



#### Using AutoMapper to automate building Select queries

AutoMapper’s By Convention configuration, where it maps properties in the source—Book class, in this case—to the DTO properties by matching them by the type and name of each property. AutoMapper can automatically map some relationships.



````c#
// Handcoded version
var dto = context.Books
    .Select(p => new ChangePubDateDto
    {
        BookId = p.BookId,
        Title = p.Title,
        PublishedOn = p.PublishedOn
    })
    .Single(k => k.BookId == lastBook.BookId);

// AutoMapper version
var dto = context.Books
    .ProjectTo<ChangePubDateDtoAm>(config)
    .Single(x => x.BookId == lastBook.BookId);
````



Four by-convention configurations of using AutoMapper:

* *Same type and same name mapping* - Properties are mapped from the entity class to DTO properties by having the same type and same name.
* *Trimming properties* - By leaving out properties that are in the entity class from the DTO, the `Select` query won’t load those columns.
* *Flattening relationships* - The name in the DTO is a combination of the navigational property name and the property in the navigational property type. The `Book` entity reference of `Promotion.NewPrice`, for example, is mapped to the DTO's `PromotionNewPrice` property.
* *Nested DTOs* - This configuration allows you to map collections from the entity class to a DTO class, so you can copy specific properties from the entity class in a navigational collection property.



**FOR SIMPLE MAPPINGS, USE THE [AUTOMAP] ATTRIBUTE**

Using AutoMapper’s `ProjectTo` method is straightforward, but it relies on the configuration of AutoMapper, which is more complex. The `AutoMap` attribute, which allows by convention configuration of simple mappings.



````c#
[AutoMap(typeof(Book))]
public class ChangePubDateDtoAm
{
    public int BookId { get; set; }
    public string Title { get; set; }
    public DateTime PublishedOn { get; set; }
}
````



Classes mapped via `AutoMap` attribute use AutoMapper’s By Convention configuration, with a few parameters and attributes to allow some tweaking. By convention can do quite a lot, but certainly not all that you might need. For that, you need AutoMapper’s `Profile` class.



**COMPLEX MAPPINGS NEED A PROFILE CLASS**

When AutoMapper’s By Convention approach isn’t enough, you need to build an AutoMapper `Profile` class, which allows you to define the mapping for properties that aren’t covered by the By Convention approach. You have to create a `MappingConfiguration`. You have a few ways to do this, but typically, you use AutoMapper’s `Profile` class, which is easy to find and register. The following listing shows a class that inherits the `Profile` class and sets up the mappings that are too complex for AutoMapper to deduce.



````c#
// AutoMapper Profile class configuring special mappings for some properties
public class BookListDtoProfile : Profile // Your class must inherit the AutoMapper Profile class. You can have multiple classes that inherit Profile.
{
    public BookListDtoProfile()
    {
        CreateMap<Book, BookListDto>() // Sets up the mapping from the Book entity class to the BookListDto
            .ForMember(p => p.ActualPrice,
                       m => m.MapFrom(s => s.Promotion == null ? s.Price : s.Promotion.NewPrice)) // The Actual price depends on whether the Promotion has a PriceOffer.
            .ForMember(p => p.AuthorsOrdered,
                       m => m.MapFrom(s => string.Join(", ", s.AuthorsLink.Select(x => x.Author.Name)))) // Gets the list of Author names as a comma-delimited string
            .ForMember(p => p.ReviewsAverageVotes,
                       m => m.MapFrom(s => s.Reviews.Select(y => (double?)y.NumStars).Average())); // Contains the special code needed to make the Average method run in the database
    }
}
````

This code sets up three of the nine properties, with the other six properties using AutoMapper’s By Convention approach, which is why some of the names of the properties in the `ListBookDto` class are long. The DTO property called `PromotionPromotionalText`, for example, has that name because it maps by convention to the navigational property `Promotion` and then to the `PromotionalText` property in the `PriceOffer` entity class.

You can add lots of `CreateMap` calls in one `Profile`, or you can have multiple `Profiles`. `Profiles` can get complex, and managing them is the main pain point involved in using AutoMapper.



**REGISTER AUTOMAPPER CONFIGURATIONS**

The last stage is registering all the mapping with dependency injection. Fortunately, AutoMapper has a NuGet package called `AutoMapper.Extensions.Microsoft.DependencyInjection` containing the method `AddAutoMapper`, which scans the assemblies you provide and registers an `IMapper` interface as a service. You use the `IMapper` interface to inject the configuration for all your classes that have the `[AutoMap]` attribute and all the classes that inherit AutoMapper’s `Profile` class. In an ASP.NET Core application, the following code snippet would be added to the Configure method of the `Startup` class:

````c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllersWithViews();
    // … other code removed for clarity
    
    services.AddAutoMapper( MyAssemblyToScan1, MyAssemblyToScan2…);
}
````



#### Evaluating how EF Core creates an entity class when reading data in

Sometimes, it’s useful to have a constructor with parameters, because it makes it easier to create an instance or because you want to make sure that the class is created in the correct way.



````c#
// An entity class with a constructor that works with EF Core
public class ReviewGood
{
    // You can set your properties to have a private setter. EF Core can still set them.
    public int Id { get; private set; }
    public string VoterName { get; private set; }
    public int NumStars { get; set; }

    public ReviewGood // The constructor doesn’t need parameters for all the properties in the class. Also, the constructor can be any type of accessibility: public, private, and so on.
        (string voterName) // EF Core will look for a parameter with the same type and a name that matches the property (with matching of Pascal/camel case versions of the name).
    {
        VoterName = voterName; // The assignment should not include any changing of the data; otherwise, you won’t get the exact data that was in the database.
        NumStars = 2; // Any assignment to a property that doesn’t have a parameter is fine. EF Core will set that property after the constructor to the data read back from the database.
    }
}
````



**CONSTRUCTORS THAT CAN CAUSE YOU PROBLEMS WITH EF CORE**

The first type of constructor that EF Core can’t use is one with a parameter whose type or name doesn’t match. The following listing shows an example with a parameter called `starRating`, which assigns to the property called `NumStars`. If this constructor is the only one, EF Core will throw an exception the first time you use the application’s DbContext.

````c#
// Class with constructor that EF Core can’t use, causing an exception
public class ReviewBadCtor
{
    public int Id { get; set; }
    public string VoterName { get; set; }
    public int NumStars { get; set; }
    
    public ReviewBadCtor( // The only constructor in this class
        string voterName,
        int starRating) // This parameter’s name doesn’t match the name of any property in this class, so EF Core can’t use it to create an instance of the class when it is reading in data.
    {
        VoterName = voterName;
        NumStars = starRating;
    }
}
````



If your constructor doesn’t match EF Core’s By Convention pattern, you need to provide a constructor that EF Core can use. The standard solution is to add a private parameterless constructor, which EF Core can use to create the class instance and use its normal parameter/field setting.



**EF CORE CAN INJECT CERTAIN SERVICES VIA THE ENTITY CONSTRUCTOR**

EF Core can inject three types of services, the most useful of which injects a method to allow lazy loading of relationships. The other two uses are advanced features.

`Microsoft.EntityFrameworkCore.Proxies` NuGet package is the simplest way to configure lazy loading, but it has the drawback that all the navigational properties must be set up to use lazy loading - that is, every navigational property must have the keyword `virtual` added to its property definition.

If you want to limit what relationships use lazy loading, you can obtain a lazy loading service via an entity class’s constructor. Then you change the navigational properties to use this service in the property’s getter method. The following listing shows a `BookLazy` entity class that has two relationships: a `PriceOffer` relationship that doesn’t use lazy loading and a `Reviews` relationship that does.

````c#
// Showing how lazy loading works via an injected lazy loader method
public class BookLazy
{
    public BookLazy() { } // You need a public constructor so that you can create this book in your code.

    // This private constructor is used by EF Core to inject the LazyLoader.
    private BookLazy(ILazyLoader lazyLoader)
    {
        _lazyLoader = lazyLoader;
    }

    private readonly ILazyLoader _lazyLoader;

    public int Id { get; set; }

    public PriceOffer Promotion { get; set; } // A normal relational link that isn’t loaded via lazy loading

    private ICollection<LazyReview> _reviews; // The actual reviews are held in a backing field.
    public ICollection<LazyReview> Reviews // The list that you will access
    {
        get => _lazyLoader.Load(this, ref _reviews); // A read of the property will trigger a lazy loading of the data (if not already loaded).
        set => _reviews = value; // The set simply updates the backing field.
    }
}
````



Injecting the service via the `ILazyLoader` interface requires the NuGet package `Microsoft.EntityFrameworkCore.Abstractions` to be added to the project. If you are enforcing an architecture that doesn’t allow any external packages in it, you can add a parameter by using the type `Action<object, string>` in the entity’s constructor. EF Core will fill the parameter of type `Action<object, string>` with an action that takes the entity instance as its first parameter and the name of the field as the second parameter. When this action is invoked, it loads the relationship data into the named field in the given entity class instance.

The other two ways of injecting a service into the entity class via a constructor are as follows:

* Injecting the DbContext instance that the entity class is linked to is useful if you want to run database accesses inside your entity class. You shouldn’t use this technique unless you have a serious performance or business logic problem that can’t be solved any other way.
* The `IEntityType` for this entity class instance gives you access to the configuration, `State`, EF Core information about this entity, and so on associated with this entity type.



### Writing to the database with EF Core

#### Evaluating how EF Core writes entities/relationships to the database

````c#
// Adding a new Book entity with a new Review
var book = new Book //Creates a new Book
{
    Title = "Test",
    Reviews = new List<Review>()
};
book.Reviews.Add(new Review { NumStars = 1 }); // Adds a new Review to the Book’s Reviews navigational property
context.Add(book); // The Add method says that the entity instance should be added to the appropriate row, with any relationships added or updated.
context.SaveChanges(); // SaveChanges carries out the database update.
````



To add these two linked entities to the database, EF Core has to do the following:

* *Work out the order in which it should create these new rows* - In this case, it has to create a row in the Books table so that it has the primary key of the Book.
* *Copy any primary keys into the foreign key of any relationships* - In this case, it copies the Books row’s primary key, `BookId`, into the foreign key in the new Review row.
* *Copy back any new data created in the database so that the entity classes properly represent the database* - In this case, it must copy back the `BookId` and update the `BookId` property in both the `Book` and `Review` entity classes and the `ReviewId` for the `Review` entity class.



````sql
// The SQL commands to create the two rows, with return of primary keys
-- first database access
SET NOCOUNT ON;
INSERT INTO [Books] ([Description], [Title], ...)
VALUES (@p0, @p1, @p2, @p3, @p4, @p5, @p6);

SELECT [BookId] FROM [Books]
WHERE @@ROWCOUNT = 1 AND [BookId] = scope_identity();

-- second database access
SET NOCOUNT ON;
INSERT INTO [Review] ([BookId], [Comment], ...)
VALUES (@p7, @p8, @p9, @p10);

SELECT [ReviewId] FROM [Review]
WHERE @@ROWCOUNT = 1 AND [ReviewId] = scope_identity();
````



You don’t need to do multiple calls to the `SaveChanges` method if you have navigational properties that link the different entities. So if you think that you need to call `SaveChanges` twice, normally you haven’t set up the right navigational properties to handle that case. Calling `SaveChanges` multiple times to create an entity with relationships isn’t recommended because if the second `SaveChanges` fails for some reason, you have an incomplete set of data in your database, which could cause problems.



#### Evaluating how DbContext handles writing out entities/relationships

````c#
// Creating a new Book with a new many-to-many link to an existing Author
//STAGE1
var author = context.Authors.First();
var bookAuthor = new BookAuthor { Author = author }; // Creates a new BookAuthor linking row, ready to link to Book to the Author
var book = new Book // Creates a Book and fills in the AuthorsLink navigational property with a single entry, linking it to the existing Author
{
    Title = "Test Book",
    AuthorsLink = new List<BookAuthor> { bookAuthor }
};

//STAGE2
context.Add(book); // Calls the Add method, which tells EF Core that the Book needs to be added to the database

//STAGE3
context.SaveChanges(); // SaveChanges looks at all the tracked entities and works out how to update the database to achieve what you have asked it to do.
````

Each of the three figures shows the following data at the end of its stage:

* The State of each entity instance at each stage of the process (shown above each entity class)
* The primary and foreign keys with the current value in brackets. If a key is (0), it hasn’t been set yet.
* The navigational links are shown as connections from the navigational property to the appropriate entity class that it is linked to.
* Changes between each stage, shown by bold text or thicker lines for the navigational links.



The situation after Stage 1 has finished. This initial code sets up a new `Book` entity class (left) with a new `BookAuthor` entity class (middle) that links the Book to an existing `Author` entity class (right).



![](./diagrams/svg/06_01_state_stage_1.drawio.svg)



Here are the things that happen when the `Add` method is called in Stage 2.



![](./diagrams/svg/06_02_state_stage_2.drawio.svg)



The `Add` method sets the `State` of the entity provided as a parameter to Added - in this example, the `Book` entity. Then it looks at all entities linked to the entity provided as a parameter, either by navigational properties or by foreign-key values. For each linked entity, it does the following:

* If the entity is not tracked - that is, its current `State` is `Detached` - it sets its `State` to `Added`. In this example, that entity is `BookAuthor`. The `Author`’s `State` isn’t updated because that entity is tracked.
* It fills in any foreign keys for the correct primary keys. If the linked primary key isn’t yet available, it puts a unique negative number in the `CurrentValue` properties of the tracking data for the primary key and the foreign key.
* It fills in any navigational properties that aren’t currently set up by running a version of the relational fixup.



The final stage, Stage 3, is what happens when the `SaveChanges` method is called.



![](./diagrams/svg/06_03_state_stage_3.drawio.svg)



Any columns set or changed by the database are copied back into the entity class so that the entity matches the database. In this example, the `Book`’s `BookId` and the `BookAuthor`’s `BookId` were updated to have the key value created in the database. Also, now that all the entities involved in this database write match the database, their `States` are set to `Unchanged`.

When something doesn’t work correctly, or when you want to do something complex, such as logging entity class changes, this information is useful.



#### A quick way to copy data with relationships

Sometimes, you want to copy an entity class with all its relationships. One solution would be to clone each entity class and its relationships, but that’s hard work.

As an example, you are going to use your knowledge of EF Core to copy a user’s Book App `Order`, which has a collection of `LineItems`, which in turn links to `Books`. You want to copy the `Order` only with the `LineItems`, but you do not want to copy the `Books` that the `LineItems` links to; two copies of a `Book` would cause all sorts of problems.



````c#
// Creating an Order with two LineItems ready to be copied
var books = context.SeedDatabaseFourBooks(); // For this test, add four books to use as test data.
var order = new Order // Creates an Order with two LineItems to copy
{
    CustomerId = Guid.Empty, // Sets CustomerId to the default value so that the query filter reads the order back
    LineItems = new List<LineItem>
    {
        // Adds the first LineNum linked to the first book
        new LineItem
        {
            LineNum = 1, ChosenBook = books[0], NumBooks = 1
        },
        // Adds the second LineNum linked to the second book
        new LineItem
        {
            LineNum = 2, ChosenBook = books[1], NumBooks = 2
        },
    }
};
context.Add(order);
context.SaveChanges();
````

To copy that Order properly, you need to know three things:

* If you Add an entity that has linked entities that are not tracked - that is, with a `State` of `Detached` - they will be set to the `State` `Added`.
* EF Core can find linked entities via the navigational links.
* If you try to `Add` an entity class to the database, and the primary key is already in the database, you will get a database exception because the primary key must be unique.

When you know those three things, you can get EF Core to copy the `Order` with its `LineItems`, but not the `Books` that the `LineItems` link to. Here is the code that copies the `Order` and its `LineItems` but doesn’t copy the `Book` linked to the `LineItems`.

````c#
// Copying an Order with its LineItems
var order = context.Orders // This code is going to query the Orders table.
    .AsNoTracking() // AsNoTracking means that the entities are read-only; their State will be Detached.
    .Include(x => x.LineItems) // Include the LineItems, as you want to copy them too.
    // You do not add .ThenInclude(x => x.ChosenBook) to the query. If you did, the query would copy the Book entities, which is not what you want.
    .Single(x => x.OrderId == id); // Takes the Order that you want to copy

// Resets the primary keys (Order and LineItem) to their default value, telling the database to generate new primary keys
order.OrderId = default;
order.LineItems.First().LineItemId = default;
order.LineItems.Last().LineItemId = default;
context.Add(order);
context.SaveChanges();
````

Note that you haven’t reset the foreign keys because you are relying on the fact that the navigational properties override any foreign key values.



#### A quick way to delete an entity

````c#
// Deleting an entity from the database by setting its primary key
var book = new Book // Creates the entity class that you want to delete (in this case, a Book)
{
    BookId = bookId // Sets the primary key of the entity instance
};
context.Remove(book); // The call to Remove tells EF Core that you want this entity/row to be deleted.
context.SaveChanges();
````

In a disconnected situation, such as some form of web application, the command to delete returns only the type and primary key value(s), making the delete code simpler and quicker. Some minor things are different from the read/remove approach to relationships:

* If there is no row for the primary key you gave, EF Core throws a `DbUpdateConcurrencyException`, saying that nothing was deleted.
* The database is in command of which other linked entities are deleted; EF Core has no say in that.



### Summary

* When reading in entity classes as tracked entities, EF Core uses a process called relational fixup that sets up all the navigational properties to any other tracked entities.
* The normal tracking query uses identity resolution, producing the best representation of the database structure with one entity class instance for each unique primary key.
* The `AsNoTracking` query is quicker than a normal tracking query because it doesn’t use identity resolution, but it can create duplicate entity classes with the same data.
* If your query loads multiple collections of relationships by using the Include method, it creates one big database query, which can be slow in some circumstances.
* If your query is missing an `Include` method, you will get the wrong result, but there is a way to set up your navigational collections so that your code will fail instead of returning incorrect data.
* Using Global Query Filters to implement a soft-delete feature works well, but watch how you handle relationships that rely on the soft-deleted entity.
* Select queries are efficient from the database side but can take more lines of code to write. The AutoMapper library can automate the building of Select queries.
* EF Core creates an entity class when reading in data. It does this via the default parameterless constructor or any other constructors you write if you follow the normal pattern.
* When EF Core creates an entity in the database, it reads back any data generated by the database, such as a primary key provided by the database, so that it can update the entity class instance to match the database.





## 7. Configuring nonrelational properties

### Three ways of configuring EF Core



![](./diagrams/images/07_01_init_dbcontext.png)



This list summarizes the three approaches to configuring EF Core:

* *By Convention* - When you follow simple rules on property types and names, EF Core will autoconfigure many of the software and database features. The By Convention approach is quick and easy, but it can’t handle every eventuality.
* *Data Annotations* - A range of .NET attributes known as *Data Annotations* can be added to entity classes and/or properties to provide extra configuration information.
* *Fluent API* - EF Core has a method called `OnModelCreating` that’s run when the EF context is first used. You can override this method and add commands, known as the *Fluent API*, to provide extra information to EF Core in its modeling stage. The Fluent API is the most comprehensive form of configuration information, and some features are available only via that API.



### A worked example of configuring EF Core

For anything beyond a Hello World version of using EF Core, you’re likely to need some form of Data Annotations or Fluent API configuration.



![](./diagrams/images/07_02_configuring_ef_core.png)



Here are a few EF Core configurations:

* `[Required]` *attribute* - This attribute tells EF Core that the `Title` column can’t be SQL NULL, which means that the database will return an error if you try to insert/update a book with a `null` `Title` property.
* `[MaxLength(256)]` *attribute* - This attribute tells EF Core that the number of characters stored in the database should 256 rather than defaulting to the database’s maximum size (2 GB in SQL Server). Having fixed-length strings of the right type, 2-byte Unicode or 1-byte ASCII, makes the database access slightly more efficient and allows an SQL index to be applied to these fixed-size columns.
* `HasColumnType("date")` *Fluent API* - By making the `PublishedOn` column hold only the date (which is all you need) rather than the default `datetime2`, you reduce the column size from 8 bytes to 3 bytes, which makes searching and sorting on the `PublishedOn` column faster.
* `IsUnicode(false)` *Fluent API* - The `ImageUrl` property contains only 8-bit ASCII characters, so you tell EF Core so, which means that the string will be stored that way. So if the `ImageUrl` property has a `[MaxLength(512)]` attribute, the `IsUnicode(false)method` would reduce the size of the `ImageUrl` column from 1024 bytes (Unicode takes 2 bytes per character) to 512 bytes (ASCII takes 1 byte per character).



````c#
// The Book entity class with added Data Annotations
public class Book
{
    public int BookId { get; set; }

    [Required]
    [MaxLength(256)]
    public string Title { get; set; }
    public string Description { get; set; }
    public DateTime PublishedOn { get; set; }
    [MaxLength(64)]
    public string Publisher { get; set; }
    public decimal Price { get; set; }

    [MaxLength(512)]
    public string ImageUrl { get; set; }
    public bool SoftDeleted { get; set; }

    //-----------------------------------------------
    //relationships
    
    public PriceOffer Promotion { get; set; }
    public IList<Review> Reviews { get; set; }
    public IList<BookAuthor> AuthorsLink { get; set; }
}
````



### Configuring by convention

*By Convention* is the default configuration, which can be overridden by the other two approaches, Data Annotations and the Fluent API. The By Convention approach relies on the developer to use the By Convention naming standards and type mappings, which allow EF Core to find and configure entity classes and their relationships, as well as define much of the database model.



#### Conventions for entity classes

Classes that EF Core maps to the database are called *entity classes*. Entity classes are normal .NET classes, sometimes referred to as POCOs (plain old CLR objects). EF Core requires entity classes to have the following features:

* The class must be of public access: the keyword `public` should be before the class.
* The class can’t be a `static` class, as EF Core must be able to create a new instance of the class.
* The class must have a constructor that EF Core can use. The default, parameterless constructor works, and other constructors with parameters can work.



#### Conventions for parameters in an entity class

By convention, EF Core will look for `public` properties in an entity class that have a `public` getter and a setter of any access mode (`public`, `internal`, `protected`, or `private`). The typical, all-public property is



````c#
public int MyProp { get; set; }
````



Although the all-public property is the norm, in some places having a property with a more localized access setting (such as `public int MyProp { get; private set; }`) gives you more control of how it’s set. One example would be a method in the entity class that also does some checks before setting the property.

EF Core can handle read-only properties - properties with only a getter, such as `public int MyProp { get; }`. But in that case, the By Convention approach won’t work; you need to use Fluent API to tell EF Core that those properties are mapped to the database.



#### Conventions for name, type, and size

Here are the rules for the name, type, and size of a relational column:

* The name of the property is used as the name of the column in the table.
* The .NET type is translated by the database provider to the corresponding SQL type. Many basic .NET types have a one-to-one mapping to a corresponding database type. These basic .NET types are mostly .NET *primitive* types (`int`, `bool`, and so on), with some special cases (such as `string`, `DateTime`, and `Guid`).
* The size is defined by the .NET type; for instance, the 32-bit `int` type is stored in the corresponding SQL’s 32-bit INT type. `String` and `byte[]` types take on a size of max, which will be different for each database type.



#### By convention, the nullability of a property is based on .NET type

In relational databases, `NULL` represents missing or unknown data. Whether a column can be `NULL` is defined by the .NET type:

* If the type is `string`, the column can be `NULL`, because a string can be null.
* Primitive types (such as `int`) or struct types (such as `DateTime`) are non-null by default.
* Primitive or struct types can be made nullable by using either the `?` suffix (such as `int?`) or the generic `Nullable<T>` (such as `Nullable<int>`). In these cases, the column can be `NULL`.



![](./diagrams/images/07_03_by_convention_nullability.png)



#### An EF Core naming convention identifies primary keys

The other rule is about defining the database table’s primary key. The EF Core conventions for designating a primary key are as follows:

* EF Core expects one primary-key property. (The By Convention approach doesn’t handle keys made up of multiple properties/columns, called *composite keys*.)
* The property is called `Id` or `<class name>id` (such as `BookId`).
* The type of the property defines what assigns a unique value to the key.



![](./diagrams/images/07_04_by_convention_primary_key.png)



Although you have the option of using the short name, `Id`, for a primary key, recommend that you use the longer name: `<class name>` followed by `Id` (`BookId`, for example).



### Configuring via Data Annotations

*Data Annotations* are a specific type of .NET attribute used for validation and database features. These attributes can be applied to an entity class or property and provide configuration information to EF Core. The Data Annotation attributes that are relevant to EF Core configuration come from two namespaces.



#### Using annotations from System.ComponentModel.DataAnnotations

The attributes in the `System.ComponentModel.DataAnnotations` namespace are used mainly for data validation at the frontend, such as ASP.NET, but EF Core uses some of them for creating the mapping model. Attributes such as `[Required]` and `[MaxLength]` are the main ones, with many of the other Data Annotations having no effect on EF Core.



![](./diagrams/images/07_05_data_annotations.png)



#### Using annotations from System.ComponentModel.DataAnnotations.Schema

The attributes in the `System.ComponentModel.DataAnnotations.Schema` namespace are more specific to database configuration. This namespace was added in NET Framework 4.5, well before EF Core was written, but EF Core uses its attributes, such as `[Table]`, `[Column]`, and so on, to set the table name and column name/type, as described.



### Configuring via the Fluent API

The third approach to configuring EF Core, called the *Fluent API*, is a set of methods that works on the `ModelBuilder` class that’s available in the `OnModelCreating` method inside your application’s DbContext. The Fluent API works by extension methods that can be chained together, as LINQ commands are chained together, to set a configuration setting. The Fluent API provides the most comprehensive list of configuration commands, with many configurations available only via that API.

As your application grows, putting all Fluent API commands in the `OnModelCreating` method makes finding a specific Fluent API hard work. The solution is to move the Fluent API for an entity class into a separate configuration class that’s then called from the `OnModelCreating` method.

EF Core provides a method to facilitate this process in the shape of the `IEntityTypeConfiguration<T>` interface. Your new application DbContext, `EfCoreContext`, where you move the Fluent API setup of the various classes into separate configuration classes. The benefit of this approach is that the Fluent API for an entity class is all in one place, not mixed with Fluent API commands for other entity classes.



````c#
// Application’s DbContext for database with relationships
public class EfCoreContext : DbContext // UserId of the user who has bought some books
{
    public EfCoreContext(DbContextOptions<EfCoreContext> options)
        : base(options) // Creates the DbContext, using the options set up when you registered the DbContext
    { }

    // The entity classes that your code will access
    public DbSet<Book> Books { get; set; }
    public DbSet<Author> Authors { get; set; }
    public DbSet<PriceOffer> PriceOffers { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder) // The method in which your Fluent API commands run
    {
        // Run each of the separate configurations for each entity class that needs configuration.
        modelBuilder.ApplyConfiguration(new BookConfig());
        modelBuilder.ApplyConfiguration(new BookAuthorConfig());
        modelBuilder.ApplyConfiguration(new PriceOfferConfig());
        modelBuilder.ApplyConfiguration(new LineItemConfig());
    }
}
````

````c#
// BookConfig extension class configures Book entity class
internal class BookConfig : IEntityTypeConfiguration<Book>
{
    public void Configure(EntityTypeBuilder<Book> entity)
    {
        entity.Property(p => p.PublishedOn)
            .HasColumnType("date"); // Convention-based mapping for .NET DateTime is SQL datetime2. This command changes the SQL column type to date, which holds only the date, not the time.

        entity.Property(p => p.Price)
            .HasPrecision(9,2); // The precision of (9,2) sets a max price of 9,999,999.99 (9 digits, 2 after decimal point), which takes up the smallest size in the database.

        entity.Property(x => x.ImageUrl)
            .IsUnicode(false); // The conventionbased mapping for .NET string is SQL nvarchar (16 bit Unicode). This command changes the SQL column type to varchar (8-bit ASCII).

        entity.HasIndex(x => x.PublishedOn); // Adds an index to the PublishedOn property because you sort and filter on this property
    }
}
````



`OnModelCreating` is called when the application first accesses the application’s DbContext. At that stage, EF Core configures itself by using all three approaches: By Convention, Data Annotations, and any Fluent API you’ve added in the `OnModelCreating` method.



### Excluding properties and classes from the database

At times, you’ll want to exclude data in your entity classes from being in the database. You might want to have local data for a calculation used during the lifetime of the class instance, for example, but you don’t want it saved to the database. You can exclude a class or a property in two ways: via Data Annotations or via the Fluent API.



#### Excluding a class or property via Data Annotations

EF Core will exclude a property or a class that has a `[NotMapped]` data attribute applied to it.



````c#
// Excluding three properties, two by using [NotMapped]
public class MyEntityClass
{
    public int MyEntityClassId { get; set; }

    public string NormalProp{ get; set; } // Included: A normal public property, with public getter and setter

    [NotMapped] // Excluded: Placing a [NotMapped] attribute tells EF Core to not map this property to a column in the database.
    public string LocalString { get; set; }
    
    public ExcludeClass LocalClass { get; set; } // Excluded: This class won’t be included in the database because the class definition has a [NotMapped] attribute on it.
}

[NotMapped] // Excluded: This class will be excluded because the class definition has a [NotMapped] attribute on it.
public class ExcludeClass
{
    public int LocalInt { get; set; }
}
````



#### Excluding a class or property via the Fluent API

In addition, you can exclude properties and classes by using the Fluent API configuration command Ignore.



````c#
// Excluding a property and a class by using the Fluent API
public class ExcludeDbContext : DbContext
{
    public DbSet<MyEntityClass> MyEntities { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<MyEntityClass>()
            .Ignore(b => b.LocalString); // The Ignore method is used to exclude the LocalString property in the entity class, MyEntityClass, from being added to the database.

        modelBuilder.Ignore<ExcludeClass>(); // A different Ignore method can exclude a class such that if you have a property in an entity class of the Ignored type, that property isn’t added to the database.
    }
}
````



EF Core will ignore read-only properties - that is, a property with only a getter (such as `public int MyProp { get; }`).



### Setting database column type, size, and nullability

The convention-based modeling uses default values for the SQL type, size/precision, and nullability based on the .NET type. A common requirement is to set one or more of these attributes manually, either because you’re using an existing database or because you have performance or business reasons to do so.



Set not null (Default is nullable.)

​	Data Annotations

* ````c#
  [Required]
  public string MyProp { get; set; }
  ````

​	Fluent API

* ````c#
  modelBuilder.Entity<MyClass>()
      .Property(p => p.MyProp)
      .IsRequired();
  ````

Set size (string) (Default is MAX length.)

​	Data Annotations

* ````c#
  [MaxLength(123)]
  public string MyProp { get; set; }

​	Fluent API

* ````c#
  modelBuilder.Entity<MyClass>()
      .Property(p => p.MyProp)
      .HasMaxLength(123);
  ````

Set SQL type/size (Each type has a default precision and size.)

​	Data Annotations

* ````c#
  [Column(TypeName = "date")]
  public DateTime PublishedOn { get; set; }
  ````

  Fluent API

* ````c#
  modelBuilder.Entity<MyClass>(
      .Property(p => p.PublishedOn)
      .HasColumnType("date");



Some specific SQL types have their own Fluent API commands, which are shown in the following list.

* `IsUnicode(false)` - Sets the SQL type to `varchar(nnn)` (1-byte character, known as ASCII) rather than the default of `nvarchar(nnn)` (2-byte character, known as Unicode).
* `HasPrecision(precision, scale)` - Sets the number of digits (`precision` parameter) and how many of the digits are after the decimal point (`scale` parameter). This Fluent command is new in EF Core 5. The default setting of the SQL `decimal` is (18,2).
* `HasCollation("collation name")` - Another EF Core 5 feature that allows you to define the collation on a property - that is, the sorting rules, case, and accent sensitivity properties of `char` and `string` types.

Recommend using the `IsUnicode(false)` method to tell EF Core that a string property contains only single-byte ASCII-format characters, because using the `IsUnicode` method allows you to set the string size separately.



### Value conversions: Changing data to/from the database

EF Core’s value conversions feature allows you to change data when reading and writing a property to the database. Typical uses are

* Saving `Enum` type properties as a string (instead of a number) so that it’s easier to understand when you’re looking at the data in the database

* Fixing the problem of `DateTime` losing its UTC (Coordinated Universal Time) setting when read back from the database
* (Advanced) Encrypting a property written to the database and decrypting on reading back

The value conversions have two parts:

* Code that transforms the data as it is written out to the database
* Code that transforms the database column back to the original type when read back



The first example of value conversions deals with a limitation of the SQL database in storing `DateTime` types, in that it doesn’t save the `DateTimeKind` part of the `DateTime` struct that tells us whether the `DateTime` is local time or UTC. This situation can cause problems. If you send that `DateTime` to your frontend using JSON, for example, the `DateTime` won’t contain the Z suffix character that tells JavaScript that the time is UTC, so your frontend code may display the wrong time. The following listing shows how to configure a property to have a value conversion that sets the `DateTimeKind` on the return from the database.

````c#
// Configuring a DateTime property to replace the lost DateTimeKind setting
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    var utcConverter = new ValueConverter<DateTime, DateTime>( // Creates a ValueConverter from DateTime to DateTime
        toDb => toDb, // Saves the DateTime to the database in the normal way (such as no conversion)
        fromDb => DateTime.SpecifyKind(fromDb, DateTimeKind.Utc)); // On reading from the database, you add the UTC setting to the DateTime.

    modelBuilder.Entity<ValueConversionExample>()
        .Property(e => e.DateTimeUtcUtcOnReturn) // Selects the property you want to configure
        .HasConversion(utcConverter); // Adds the utcConverter to that property

    //… other configurations left out
}
````

In this case, you had to create your own value converter, but about 20 built-in value converters are available. In fact, one value converter is so popular that it has a predefined Fluent API method or an attribute - a conversion to store an `Enum` as a string in the database.

`Enum`s are normally stored in the database as numbers, which is an efficient format, but it does make things harder if you need to delve into the database to work out what happened. So some developers like to save `Enum`s in the database as a string. You can configure a conversion of an `Enum` type to a string by using the `HasConversion<string>()` command, as in the following code snippet:

````c#
modelBuilder.Entity<ValueConversionExample>()
    .Property(e => e.Stage)
    .HasConversion<string>();
````



Following are some rules and limitations on using value conversions:

* A `null` value will never be passed to a value converter. You need to write a value converter to handle only the non-null value, as your converter will be called only if the value isn’t a `null`.
* Watch out for queries that contain sorting on a converted value. If you converted your `Enum`s to a string, for example, the sorting will sort by the `Enum` name, not by the `Enum` value.
* The converter can only map a single property to a single column in the database.
* You can create some complex value converters, such as serializing a list of `int`s to a JSON string. At this point, EF Core cannot compare the `List<int>` property with the JSON in the database, so it won’t update the database. To solve this problem, you need to add what is called a *value comparer*.



### The different ways of configuring the primary key

You need to configure the primary key explicitly in two situations:

* When the key name doesn’t fit the By Convention naming rules
* When the primary key is made up of more than one property/column, called a *composite key*

A many-to-many relationship-linking table is an example of where the By Convention approach doesn’t work. You can use two alternative approaches to define primary keys.



#### Configuring a primary key via Data Annotations

The `[Key]` attribute allows you to designate one property as the primary key in a class. Use this annotation when you don’t use the By Convention primary key name. This code is simple and clearly marks the primary key.



````c#
// Defining a property as the primary key bu using the [Key] annotation
private class SomeEntity
{
    [Key] // [Key] attribute tells EF Core that the property is a primary key.
    public int NonStandardKeyName { get; set; }

    public string MyString { get; set; }
}
````



Note that the `[Key]` attribute can’t be used for composite keys. In earlier versions of EF Core, you could define composite keys by using `[Key]` and `[Column]` attributes, but that feature has been removed.



#### Configuring a primary key via the Fluent API

You can also configure a primary key via the Fluent API, which is useful for primary keys that don’t fit the By Convention patterns. The first primary key is a single primary key with a nonstandard name in the `SomeEntity` entity class, and the second is a composite primary key, consisting of two columns, in the BookAuthor linking table.



````c#
// Using the Fluent API to configure primary keys on two entity classes
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<SomeEntity>()
        .HasKey(x => x.NonStandardKeyName); // Defines a normal, single-column primary key. Use HasKey when your key name doesn’t match the By Convention defaults.

    modelBuilder.Entity<BookAuthor>()
        .HasKey(x => new {x.BookId, x.AuthorId}); // Uses an anonymous object to define two (or more) properties to form a composite key. The order in which the properties appear in the anonymous object defines their order.

    //… other configuration settings removed
}
````



There is no By Convention version for composite keys, so you must use the Fluent API’s `HasKey` method.



#### Configuring an entity as read-only

In some advanced situations, your entity class might not have a primary key. Here are three examples:

* *You want to define an entity class as read-only.* If an entity class hasn’t got a primary key, then EF Core will treat it as read-only.
* *You want to map an entity class to a read-only SQL View.* SQL Views are SQL queries that work like SQL tables.
* *You want to map an entity class to an SQL query by using the ToSqlQuery Fluent API command.* The `ToSqlQuery` method allows you to define an SQL command string that will be executed when you read in that entity class.

To set an entity class explicitly as read-only, you can use the fluent API `HasNoKey()` command or apply the attribute `[Keyless]` to the entity class. And if your entity class doesn’t have a primary key, you must mark it as read-only, using either of the two approaches. Any attempt to change the database via an entity class with no primary key will fail with an exception. EF Core does this because it can’t execute the update without a key, which is one way you can define an entity class as read-only. The other way to mark an entity as read-only is to map an entity to an SQL View by using the fluent API method `ToView("ViewNameString")` command, as shown in the following code snippet:

````c#
modelBuilder.Entity<MyEntityClass>()
    .ToView("MyView");
````

EF Core will throw an exception if you try to change the database via an entity class that is mapped to a View. If you want to map an entity class to an updatable view - an SQL View that can be updated - you should use the `ToTable` command instead.



### Adding indexes to database columns

Relational databases have a feature called an *index*, which provides quicker searching and sorting of rows based on the column, or columns, in the index. In addition, an index may have a constraint, which ensures that each entry in the index is unique. A primary key is given a unique index, for example, to ensure that the primary key is different for each row in the table.

An index will speed quick searching and sorting, and if you add the unique constraint, the database will ensure that the column value in each row will be different.



Add index, Fluent

````c#
modelBuilder.Entity<MyClass>()
    .HasIndex(p => p.MyProp);
````

Add index, Attribute

````c#
[Index(nameof(MyProp))]
public class MyClass …
````

Add index, multiple columns

````c#
modelBuilder.Entity<Person>()
    .HasIndex(p => new {p.First, p.Surname});
````

Add index, multiple columns, Attribute

````c#
[Index(nameof(First), nameof(Surname)]
public class MyClass …
````

Add unique index, Fluent

````c#
modelBuilder.Entity<MyClass>()
    .HasIndex(p => p.BookISBN)
    .IsUnique();
````

Add unique index, Attribute

````c#
[Index(nameof(MyProp), IsUnique = true)]
public class MyClass …
````

Add named index, Fluent

````c#
modelBuilder.Entity<MyClass>()
    .HasIndex(p => p.MyProp)
    .HasDatabaseName("Index_MyProp");
````



Some databases allow you to specify a filtered or partial index to ignore certain situations by using a `WHERE` clause. You could set a unique filtered index that ignored any soft-deleted items, for example. To set up a filtered index, you use the `HasFilter` Fluent API method containing an SQL expression to define whether the index should be updated with the value. The following code snippet gives an example of enforcing that the property MyProp will contain a unique value unless the SoftDeleted column of the table is `true`:

````c#
modelBuilder.Entity<MyClass>()
    .HasIndex(p => p.MyProp)
    .IsUnique()
    .HasFilter(“NOT SoftDeleted");
````



When you’re using the SQL Server provider, EF adds an `IS NOT NULL` filter for all nullable columns that are part of a unique index. You can override this convention by providing `null` to the `HasFilter` parameter - that is `HasFilter(null)`.



### Configuring the naming on the database side

#### Configuring table names

By convention, the name of a table is set by the name of the `DbSet<T>` property in the application’s DbContext, or if no `DbSet<T>` property is defined, the table uses the class name. In the application’s DbContext of our Book App, for example, you defined a `DbSet<Book>` Books property, so the database table name is set to Books. Conversely, you haven’t defined a `DbSet<T>` property for the `Review` entity class in the application’s DbContext, so its table name used the class name and is, therefore, Review.

If your database has specific table names that don’t fit the By Convention naming rules - for example, if the table name can’t be converted to a valid .NET variable name because it has a space in it - you can use either Data Annotations or the Fluent API to set the table name specifically.



Data Annotations

````c#
[Table("XXX")]
public class Book … etc.
````

Fluent API

````c#
modelBuilder.Entity<Book>()
    .ToTable("XXX");
````



#### Configuring the schema name and schema groupings

Some databases, such as SQL Server, allow you to group your tables by using what is called a schema name. You could have two tables with the same name but different schema names: a table called Books with a schema name Display, for example, would be different from a table called Books with a schema name Order.

By convention, the schema name is set by the database provider because some databases, such as SQLite and MySQL, don’t support schemas. In the case of SQL Server, which does support schemas, the default schema name is *dbo*, which is the SQL Server default name. You can change the default schema name only via the Fluent API, using the following snippet in the `OnModelCreating` method of your application’s DbContext:

````c#
modelBuilder.HasDefaultSchema("NewSchemaName");
````



You use this approach if your database is split into logical groups such as sales, production, accounts, and so on, and a table needs to be specifically assigned to a schema.

Data Annotations

````c#
[Table("SpecialOrder", Schema = "sales")]
class MyClass … etc.
````

Fluent API

````c#
modelBuilder.Entity<MyClass>()
    .ToTable("SpecialOrder", schema: "sales");
````



#### Configuring the database column names in a table

By convention, the column in a table has the same name as the property name. If your database has a name that can’t be represented as a valid .NET variable name or doesn’t fit the software use, you can set the column names by using Data Annotations or the Fluent API.

Data Annotations

````c#
[Column("SpecialCol")]
public int BookId { get; set; }
````

Fluent API

````c#
modelBuilder.Entity<MyClass>()
    .Property(b => b.BookId)
    .HasColumnName("SpecialCol");
````



### Configuring Global Query Filters

Many applications, such as ASP.NET Core, have security features that control what views and controls the user can access. EF Core has a similar security feature called *Global Query Filters* (shortened to *Query Filters*). You can use Query Filters to build a multitenant application. This type of application holds data for different users in one database, but each user can see only the data they are allowed to access. Another use is to implement a soft-delete feature; instead of deleting data in the database, you might use a Query Filter to make the soft-deleted row disappear, but the data will still be there if you need to undelete it later.



### Applying Fluent API commands based on the database provider type

The EF Core database providers provide a way to detect what database provider is being used when an instance of an application DbContext is created. This approach is useful for situations such as using, say, an SQLite database for your unit tests, but the production database is on an SQL Server, and you want to change some things to make your unit tests work.

SQLite, for example, doesn’t fully support a few NET types, such as decimal, so if you try to sort on a decimal property in an SQLite database, you’ll get an exception saying that you won’t get the right result from an SQLite database. One way to get around this issue is to convert the decimal type to a double type when using SQLite; it won’t be accurate, but it might be OK for a controlled set of unit tests.

Each database provider provides an extension method to return `true` if the database matches that provider. The SQL Server database provider, for example, has a method called `IsSqlServer();` the SQLite database provider has a method called `IsSqlite();` and so on. Another approach is to use the `ActiveProvider` property in the `ModelBuilder` class, which returns a string that is the NuGet package name of the database provider, such as "`Microsoft.EntityFrameworkCore.SqlServer`".



````c#
// Using database-provider commands to set a column name
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    //… put your normal configration here
    if (Database.IsSqlite()) // The IsSqlite will return true if the database provided in the options is SQLite.
    {
        // You set the two decimal values to double so that a unit test that sorts on these values doesn’t throw an exception.
        modelBuilder.Entity<Book>()
            .Property(e => e.Price)
            .HasConversion<double>();
        modelBuilder.Entity<PriceOffer>()
            .Property(e => e.NewPrice)
            .HasConversion<double>();
    }
}
````



### Shadow properties: Hiding column data inside EF Core

*Shadow properties* allow you to access database columns without having them appear in the entity class as a property. Shadow properties allow you to "hide" data that you consider not to be part of the normal use of the entity class. This is all about good software practice: you let upper layers access only the data they need, and you hide anything that those layers don’t need to know about. Let me give you two examples that show when you might use shadow properties:

* A common need is to track by whom and when data was changed, maybe for auditing purposes or to understand customer behavior. The tracking data you receive is separate from the primary use of the class, so you may decide to implement that data by using shadow properties, which can be picked up outside the entity class.
* When you’re setting up relationships in which you don’t define the foreign-key properties in your entity class, EF Core must add those properties to make the relationship work, and it does this via shadow properties.



#### Configuring shadow properties

There’s a By Convention approach to configuring shadow properties, but because it relates only to relationships. The other method is to use the Fluent API. You can introduce a new property by using the Fluent API method `Property<T>`. Because you’re setting up a shadow property, there won’t be a property of that name in the entity class, so you need to use the Fluent API’s `Property<T>` method, which takes a .NET Type and the name of the shadow property. The following listing shows the setup of a shadow property called `UpdatedOn` that’s of type `DateTime`.

````c#
// Creating the UpdatedOn shadow property by using the Fluent API
public class Chapter06DbContext : DbContext
{
    …

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<MyEntityClass>()
            .Property<DateTime>("UpdatedOn"); // Uses the Property<T> method to define the shadow property type
        …
    }
}
````



Under By Convention, the name of the table column the shadow property is mapped to is the same as the name of the shadow property. You can override this setting by adding the `HasColumnName` method on to the end of the `property` method.

If a property of that name already exists in the entity class, the configuration will use that property instead of creating a shadow property.



#### Accessing shadow properties

Because the shadow properties don’t map to a class property, you need to access them directly via EF Core. For this purpose, you have to use the EF Core command `Entry(myEntity).Property("MyPropertyName").CurrentValue`, which is a read/write property.

````c#
// Using Entry(inst).Property(name) to set the shadow property
var entity = new SomeEntityClass(); // Creates an entity class ...
context.Add(entity); // ... and adds it to the context, so it’s now tracked
context.Entry(entity) // Gets the EntityEntry from the tracked entity data
    .Property("UpdatedOn").CurrentValue // Uses the Property method to get the shadow property with read/write access
    	= DateTime.Now; // Sets that property to the value you want
context.SaveChanges(); // Calls SaveChanges to save the MyEntityClass instance, with its normal and shadow property values, to the database
````



If you want to read a shadow property in an entity that has been loaded, use the `context.Entry(entityInstance).Property("PropertyName").CurrentValue` command. But you must read the entity as a tracked entity; you should read the entity without the `AsNoTracking` method being used in the query. The `Entry(<entityInstance>).Property` method uses the tracked entity data inside EF Core to hold the value, as it’s not held in the entity class instance.

In LINQ queries, you use another technique to access a shadow property: the `EF.Property` command. You could sort by the `UpdatedOn` shadow property, for example, by using the following query snippet, with the `EF.Property` method in bold:

````c#
context.MyEntities
    .OrderBy(b => EF.Property<DateTime>(b, "UpdatedOn"))
    .ToList();
````



### Backing fields: Controlling access to data in an entity class

Columns in a database table are normally mapped to an entity class property with normal getters and setters - `public int MyProp { get ; set; }`. But you can also map a private field to your database. This feature is called a *backing field*, and it gives you more control of the way that database data is read or set by the software.

Like shadow properties, backing fields hide data, but they do the hiding in another way. For shadow properties, the data is hidden inside EF Core’s data, but backing fields hide the data inside the entity class, so it’s easier for the entity class to access the backing field inside the class. Here are some examples of situations in which you might use backing fields:

* *Hiding sensitive data* - Hiding a person’s date of birth in a private field and making their age in years available to the rest of the software.
* *Catching changes* - Detecting an update of a property by storing the data in a private field and adding code in the setter to detect the update of a property.
* *Creating Domain-Driven Design (DDD) entity classes* - Creating DDD entity classes in which all the entity classes’ properties need to be read-only. Backing fields allow you to lock down navigational collection properties.



#### Creating a simple backing field accessed by a read/write property

The following code snippet shows you a string property called `MyProperty`, in which the string data is stored in a private field. This form of backing field doesn’t do anything particularly different from using a normal property, but this example shows the concept of a property linked to a private field:

````c#
public class MyClass
{
    private string _myProperty;
    public string MyProperty
    {
        get { return _myProperty; }
        set { _myProperty = value; }
    }
}
````

EF Core’s By Convention configuration will find the type of backing field and configure it as a backing field, and by default, EF Core will read/write the database data to this private field.



#### Creating a read-only column

Creating a read-only column is the most obvious use, although it can also be implemented via a private setting property. If you have a column in the database that you need to read but don’t want the software to write, a backing field is a great solution. In this case, you can create a private field and use a public property, with a getter only, to retrieve the value.



````c#
public class MyClass
{
    private string _readOnlyCol;
    public string ReadOnlyCol => _readOnlyCol;
}
````



Something must set the column property, such as setting a default value in the database column or through some sort of internal database method.



#### Concealing a person’s date of birth: Hiding data inside a class

Hiding a person’s date of birth is a possible use of backing fields. In this case, you deem for security reasons that a person’s date of birth can be set, but only their age can be read from the entity class. The following listing shows how to do this in the `Person` class by using a private `_dateOfBirth` field and then providing a method to set it and a property to calculate the person’s age.

````c#
// Using a backing field to hide sensitive data from normal access
public class Person
{
    private DateTime _dateOfBirth; // The private backing field, which can’t be accessed directly via normal .NET software

    public void SetDateOfBirth(DateTime dateOfBirth) // Allows the backing field to be set
    {
        _dateOfBirth = dateOfBirth;
    }

    public int AgeYears => Years(_dateOfBirth, DateTime.Today); // You can access the person’s age but not their exact date of birth.

    private static int Years(DateTime start, DateTime end)
    {
        return (end.Year - start.Year - 1) +
            (((end.Month > start.Month) ||
              ((end.Month == start.Month)
               && (end.Day >= start.Day)))
             ? 1 : 0);
    }
}
````



From the class point of view, the `_dateOfBirth` field is hidden, but you can still access the table column via various EF Core commands in the same way that you accessed the shadow properties: by using the `EF.Property<DateTime>(entity, "_dateOfBirth")` method.

The backing field, `_dateOfBirth`, isn’t totally secure from the developer, but that’s not the aim. The idea is to remove the date-of-birth data from the normal properties so that it doesn’t get displayed unintentionally in any user-visible view.



#### Configuring backing fields

Having seen backing fields in action, you can configure them By Convention, via Fluent API, and now in EF Core 5 via Data Annotations. The By Convention approach works well but relies on the class to have a property that matches a field by type and a naming convention. If a field doesn’t match the property name/type or doesn’t have a matching property, you need to configure your backing fields with Data Annotations or by using the Fluent API.



**CONFIGURING BACKING FIELDS BY CONVENTION**

If your backing field is linked to a valid property, the field can be configured by convention. The rules for By Convention configuration state that the private field must have one of the following names that match a property in the same class:

* `_<property name>` (for example, `_MyProperty`)
* `_<camel-cased property name >` (for example, `_myProperty`)
* `m_<property name>` (for example, `m_MyProperty`)
* `m_<camel-cased property name>` (for example, `m_myProperty`)



**CONFIGURING BACKING FIELDS VIA DATA ANNOTATIONS**

New in EF Core 5 is the `BackingField` attribute, which allows you to link a property to a private field in the entity class. This attribute is useful if you aren’t using the By Convention backing field naming style, as in this example:

````c#
private string _fieldName;
[BackingField(nameof(_fieldName))]
public string PropertyName
{
    get { return _fieldName; }
}

public void SetPropertyNameValue(string someString)
{
    _fieldName = someString;
}
````



**CONFIGURING BACKING FIELDS VIA THE FLUENT API**

You have several ways of configuring backing fields via the Fluent API. We’ll start with the simplest and work up to the more complex. Each example shows you the `OnModelCreating` method inside the application’s DbContext, with only the field part being configured:

* *Setting the name of the backing field* - If your backing field name doesn’t follow EF Core’s conventions, you need to specify the field name via the Fluent API. Here’s an example:

````c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Person>()
        .Property(b => b.MyProperty)
        .HasField("_differentName");
    …
}
````

* *Supplying only the field name* - In this case, if there’s a property with the correct name, by convention EF Core will refer to the property, and the property name will be used for the database column. Here’s an example:

````c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Person>()
        .Property("_dateOfBirth")
        .HasColumnName("DateOfBirth");
    …
}
````

If no property getter or setter is found, the field will still be mapped to the column, using its name, which in this example is `_dateOfBirth`, but that’s most likely not the name you want for the column. So you add the `HasColumnName` Fluent API method to get a better column name. The downside is that you’d still need to refer to the data in a query by its field name (in this case, `_dateOfBirth`), which isn’t too friendly or obvious.



**ADVANCED: CONFIGURING HOW DATA IS READ/WRITTEN TO THE BACKING FIELD**

The default database access mode for backing fields is for EF Core to read and write to the field. This mode works in nearly all cases, but if you want to change the database access mode, you can do so via the Fluent API `UsePropertyAccessMode` method. The following code snippet tells EF Core to try to use the property for read/write, but if the property is missing a setter, EF Core will fill in the field on a database read:

````c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Person>()
        .Property(b => b.MyProperty)
        .HasField("_differentName")
        .UsePropertyAccessMode(PropertyAccessMode.PreferProperty);
    …
}
````



### Recommendations for using EF Core’s configuration

Here are suggested approaches to use for each part of EF Core configuration:

* Start by using the By Convention approach wherever possible, because it’s quick and easy.
* Use the validation attributes - `MaxLength`, `Required`, and so on - from the Data Annotations approach, as they’re useful for validation.
* For everything else, use the Fluent API approach, because it has the most comprehensive set of commands. But consider writing code to automate common settings, such as applying the `DateTime` "UTC fix" to all `DateTime` properties whose `Name` ends with "`Utc`".



#### Use By Convention configuration first

Always start with that approach. The By Convention approach is quick and easy. You’ll see in chapter 8 that most relationships can be set up purely by using the By Convention naming rules, which can save you a lot of time. Learning what By Convention can configure will dramatically reduce the amount of configuration code you need to write.



#### Use validation Data Annotations wherever possible

Although you can do things such as limit the size of a string property with either Data Annotations or the Fluent API, recommend using Data Annotations for the following reasons:

* *Frontend validation can use them.* Although EF Core doesn’t validate the entity class before saving it to the database, other parts of the system may use Data Annotations for validation. ASP.NET Core uses Data Annotations to validate input, for example, so if you input directly into an entity class, the validation attributes will be useful. Or if you use separate ASP.NET ViewModel or DTO classes, you can cut and paste the properties with their validation attributes.
* *You may want to add validation to EF Core’s `SaveChanges`.* Using data validation to move checks out of your business logic can make your business logic simpler.
* *Data Annotations make great comments.* Attributes, which include Data Annotations, are compile-time constants; they’re easy to see and easy to understand.



#### Use the Fluent API for anything else

The Fluent API practice is more a preference than a rule, though, so make your own decision.



#### Automate adding Fluent API commands by class/property signatures

One useful feature of the Fluent API commands allows you to write code to find and configure certain configurations based on the class/property type, name, and so on.

Automating finding/adding configurations relies on a type called `IMutableModel`, which you can access in the `OnModelCreating` method. This type gives you access to all the classes mapped by EF Core to the database, and each `IMutableEntityType` allows you to access the properties. Most configuration options can be applied via methods in these two interfaces, but a few, such as Query Filters, need a bit more work. To start, you will build the code that will iterate through each entity class and its properties, and add one configuration. This iteration approach defines the way to automate configurations, and in later examples, you will add extra commands to do more configurations.

The following example adds a value converter to a `DateTime` that applies the UTC fix. But in the following listing, the UTC fix value converter is applied to every property that is a `DateTime` with a `Name` that ends with "`Utc`".

````c#
// Applying value converter to any DateTime property ending in "Utc"
protected override void OnModelCreating(ModelBuilder modelBuilder) // The Fluent API commands are applied in the OnModelCreating method.
{
    var utcConverter = new ValueConverter<DateTime, DateTime>(
        toDb => toDb,
        fromDb => DateTime.SpecifyKind(fromDb, DateTimeKind.Utc)); // Defines a value converter to set the UTC setting to the returned DateTime

    foreach (var entityType in modelBuilder.Model.GetEntityTypes()) // Loops through all the classes that EF Core has currently found mapped to the database
    {
        foreach (var entityProperty in entityType.GetProperties()) // Loops through all the properties in an entity class that are mapped to the database
        {
            // Adds the UTC value converter to properties of type DateTime and Name ending in "Utc"
            if (entityProperty.ClrType == typeof(DateTime)
                && entityProperty.Name.EndsWith("Utc"))
            {
                entityProperty.SetValueConverter(utcConverter);
            }
            //… other examples left out for clarity
        }
    }
    //… rest of configration code left out
}
````



Above code showed the setup of only one `Type/Named` property, but normally, you would have lots of Fluent API settings. In this example, you are going to do the following:

1. Add the UTC fix value converter to properties of type `DateTime` whose `Name`s end with "`Utc`".
2. Set the decimal precision/scale where the property’s Name contains "`Price`".
3. Set any string properties whose `Name` ends in "`Url`" to be stored as ASCII - that is, `varchar(nnn)`.

The following code snippet shows the code inside the `OnModelCreating` method in the Book App DbContext to add these three configuration settings:

````c#
foreach (var entityType in modelBuilder.Model.GetEntityTypes())
{
    foreach (var entityProperty in entityType.GetProperties())
    {
        if (entityProperty.ClrType == typeof(DateTime)
            && entityProperty.Name.EndsWith("Utc"))
        {
            entityProperty.SetValueConverter(utcConverter);
        }

        if (entityProperty.ClrType == typeof(decimal)
            && entityProperty.Name.Contains("Price"))
        {
            entityProperty.SetPrecision(9);
            entityProperty.SetScale(2);
        }

        if (entityProperty.ClrType == typeof(string)
            && entityProperty.Name.EndsWith("Url"))
        {
            entityProperty.SetIsUnicode(false);
        }
    }
}
````



A few Fluent APIs configurations need class-specific code, however. The Query Filters, for example, need a query that accesses entity classes. For this case, you need to add an interface to the entity class you want to add a Query Filter to and create the correct filter query dynamically.

The following listing shows the extension class, called `SoftDeleteQueryExtensions`, with its `MyQueryFilterTypes` `enum`.

````c#
// The enum/class to use to set up Query Filters on every compatible class
public enum MyQueryFilterTypes { SoftDelete, UserId } // Defines the different type of LINQ query to put in the Query Filter

public static class SoftDeleteQueryExtensions
{
    public static void AddSoftDeleteQueryFilter( // Call this method to set up the query filter.
        this IMutableEntityType entityData, // First parameter comes from EF Core and allows you to add a query filter
        MyQueryFilterTypes queryFilterType, // Second parameter allows you to pick which type of query filter to add
        IUserId userIdProvider = null) // Third optional property holds a copy of the current DbContext instance so that the UserId will be the current one
    {
        // Creates the correctly typed method to create the Where LINQ expression to use in the Query Filter
        var methodName = $"Get{queryFilterType}Filter";
        var methodToCall = typeof(SoftDeleteQueryExtensions)
            .GetMethod(methodName,
                       BindingFlags.NonPublic | BindingFlags.Static)
            .MakeGenericMethod(entityData.ClrType);
        var filter = methodToCall
            .Invoke(null, new object[] { userIdProvider });
        entityData.SetQueryFilter((LambdaExpression)filter); // Uses the filter returned by the created type method in the SetQueryFilter method
        if (queryFilterType == MyQueryFilterTypes.SoftDelete) // Adds an index on the SoftDeleted property for better performance
            entityData.AddIndex(entityData.FindProperty(
                nameof(ISoftDelete.SoftDeleted)));
        if (queryFilterType == MyQueryFilterTypes.UserId) // Adds an index on the UserId property for better performance
            entityData.AddIndex(entityData.FindProperty(
                nameof(IUserId.UserId)));
    }

    // Creates a query that is true only if the _userId matches the UserID in the entity class
    private static LambdaExpression GetUserIdFilter<TEntity>(IUserId userIdProvider) where TEntity : class, IUserId
    {
        Expression<Func<TEntity, bool>> filter =
            x => x.UserId == userIdProvider.UserId;
        return filter;
    }

    // Creates a query that is true only if the SoftDeleted property is false
    private static LambdaExpression GetSoftDeleteFilter<TEntity>(IUserId userIdProvider) where TEntity : class, ISoftDelete
    {
        Expression<Func<TEntity, bool>> filter =
            x => !x.SoftDeleted;
        return filter;
    }
}
````

Because every query of an entity that has a Query Filter will contain a filter on that property, the code automatically adds an index on every property that is used in a Query Filter. That technique improves performance on that entity.

````c#
// Adding code to the DbContext to automate setting up Query Filters
public class EfCoreContext : DbContext, IUserId // Adding the IUserId to the DbContext means that we can pass the DbContext to the UserId query filter.
{
    public Guid UserId { get; private set; } // Holds the UserId, which is used in the Query Filter that uses the IUserId interface

    // Sets up the UserId. If the userIdService is null, or if it returns null for the UserId, we set a replacement UserId.
    public EfCoreContext(DbContextOptions<EfCoreContext> options, IUserIdService userIdService = null) : base(options)
    {
        UserId = userIdService?.GetUserId()
            ?? new ReplacementUserIdService().GetUserId();
    }

    //DbSets removed for clarity

    protected override void OnModelCreating(ModelBuilder modelBuilder) // The automate code goes in the OnModelCreating method.
    {
        //other configration code removed for clarity

        foreach (var entityType in modelBuilder.Model.GetEntityTypes()) // Loops through all the classes that EF Core has currently found mapped to the database
        {
            //other property code removed for clarity

            if (typeof(ISoftDelete).IsAssignableFrom(entityType.ClrType)) // If the class inherits the ISoftDelete interface, it needs the SoftDelete Query Filter.
            {
                entityType.AddSoftDeleteQueryFilter(MyQueryFilterTypes.SoftDelete); // Adds a Query Filter to this class, with a query suitable for SoftDelete
            }
                     
            if (typeof(IUserId).IsAssignableFrom(entityType.ClrType)) // If the class inherits the IUserId interface, it needs the IUserId Query Filter.
            {
                entityType.AddSoftDeleteQueryFilter(MyQueryFilterTypes.UserId, this); // Adds the UserId Query Filter to this class. Passing ‘this’ allows access to the current UserId.
            }
        }
    }
}
````

Here are some recommendations and limitations that you should know about if you are going to use this approach:

* If you run the automatic Fluent API code before your handcoded configurations, your handcoded configurations will override any of the automatic Fluent API settings. But be aware that if there is an entity class that is registered only via manually written Fluent API, that entity class won’t be seen by the automatic Fluent API code.
* The configuration commands must apply the same configurations every time because the EF Core configures the application’s DbContext only once - on first use - and then works from a cache version.



### Summary

* The first time you create the application’s DbContext, EF Core configures itself by using a combination of three approaches: By Convention, Data Annotations, and the Fluent API.
* Value converters allow you to transform the software type/value when writing and reading back from the database.
* Two EF Core features, shadow properties and backing fields, allow you to hide data from higher levels of your code and/or control access to data in an entity class. Use the By Convention approach to set up as much as you can, because it’s simple and quick to code.
* When the By Convention approach doesn’t fit your needs, Data Annotations and/or EF Core’s Fluent API can provide extra commands to configure both the way EF Core maps the entity classes to the database and the way EF Core will handle that data.
* In addition to writing configuration code manually, you can also add code to configure entity classes and/or properties automatically based on the class/ properties signature.





## 8. Configuring relationships

* Configuring relationships with By Convention
* Configuring relationships with Data Annotations
* Configuring relationships with the Fluent API
* Mapping entities to database tables in five other ways



### Defining some relationship terms



![](./diagrams/images/08_01_some_keys.png)



* *Principal key* - A new term, taken from EF Core’s documentation, that refers to either the primary key, or the new alternate key, which has a unique value per row and isn’t the primary key
* *Principal entity* - The entity that contains the principal-key properties, which the dependent relationship refer to via a foreign key(s)
* *Dependent entity* - The entity that contains the foreign-key properties that refer to the principal entity
* *Principal key* - The entity has a principal key, also known as the primary key, which is unique for each entity stored in the database
* *Navigational property* - A term taken from EF Core’s documentation that refers to the property containing a single entity class, or a collection of entity classes, that EF Core uses to link entity classes
* *Foreign key* - Holds the principal key value(s) of the database row it’s linked to (or could be null)
* *Required relationship* - A relationship in which the foreign key is non-nullable (and principal entity must exist)
* *Optional relationship* - A relationship in which the foreign key is nullable (and principal entity can be missing)
* *Composite keys* - A principal key and a foreign key can consist of more than one property/column.



### What navigational properties do you need?

The configuring of relationships between entity classes should be guided by the business needs of your project. You could add navigational properties at both ends of a relationship, but that suggests that every navigational property is useful, and some navigational properties aren’t. It is good practice to provide only navigational properties that make sense from the business or software design point of view.

In our Book App, for example, the `Book` entity class has many `Review` entity classes, and each `Review` class is linked, via a foreign key, to one `Book`. Therefore, you could have a navigational property of type `ICollection<Review>` in the `Book` class and a navigational property of type `Book` in the `Review` class. In that case, you’d have a fully defined relationship: a relationship with navigational properties at both ends.

But do you need a fully defined relationship?

* Does the `Book` entity class need to know about the `Review` entity classes? Yes, because we want to calculate the average review score.
* Does the `Review` entity class need to know about the `Book` entity class? No, because in this example application, we don’t do anything with that relationship.

Our solution, therefore, is to have only the `ICollection<Review>` navigational property in the `Book` class.
You should add a navigational property only when it makes sense from a business point of view or when you need a navigational property to create (EF Core’s Add) an entity class with a relationship. Minimizing navigational properties will help make the entity classes easier to understand.



### Configuring relationships

EF Core has three ways to configure relationships. Here are the three approaches for configuring properties, but focused on relationships:

* *By Convention* - EF Core finds and configures relationships by looking for references to classes that have a primary key in them.
* *Data Annotations* - These annotations can be used to mark foreign keys and relationship references.
* *Fluent API* - This API provides the richest set of commands to configure any relationship fully.

The By Convention approach can autoconfigure many relationships for you if you follow its naming standards. The Fluent API allows you to define every part of a relationship manually, which can be useful if you have a relationship that falls outside the By Convention approach.



### Configuring relationships By Convention

The By Convention approach is a real time-saver when it comes to configuring relationships.

The rules are straightforward, but the ways that the property name, type, and nullability work together to define a relationship take a bit of time to absorb.



#### What makes a class an entity class?

1. EF Core scans the application’s DbContext, looking for any public `DbSet<T>` properties. It assumes that the classes, `T`, in the `DbSet<T>` properties are entity classes.
2. EF Core also looks at every public property in the classes found in step 1 and looks at properties that could be navigational properties. The properties whose type contains a class that isn’t defined as being scalar properties (`string` is a class, but it’s defined as a scalar property) are assumed to be navigational properties. These properties may appear as a single link (such as `public PriceOffer Promotion( get; set; }`) or a type that implements the `IEnumerable<T>` interface(such as `public ICollection<Review> Reviews { get; set; }`).
3. EF Core checks whether each of these entity classes has a primary key. If the class doesn’t have a primary key and hasn’t been configured as not having a key, or if the class isn’t excluded, EF Core will throw an exception.



#### An example of an entity class with navigational properties

````c#
// The Book entity class, with the relationships at the bottom
public class Book
{
    public int BookId { get; set; }
    //other scalar properties removed as not relevant…

    public PriceOffer Promotion { get; set; } // Links to a PriceOffer, which is one-to-zeroor- one relationship

    public ICollection<Tag> Tags { get; set; } // Links directly to a list of Tag entities, using EF Core 5’s automatic many-to-many relationship

    public ICollection<BookAuthor> AuthorsLink { get; set; } // Links to one side of the many-to-many relationship of authors via a linking table

    public ICollection<Review> Reviews { get; set; } // Links to any reviews for this book: one-to-many relationship
}
````



If two navigational properties exist between the two entity classes, the relationship is known as fully defined, and EF Core can work out By Convention whether it’s a one-to-one or a one-to-many relationship. If only one navigational property exists, EF Core can’t be sure, so it assumes a one-to-many relationship.

Certain one-to-one relationships may need configuration via the Fluent API if you have only one navigational property or if you want to change the default By Convention setting, such as when you’re deleting an entity class with a relationship.



#### How EF Core finds foreign keys By Convention

A foreign key must match the principal key in type and in name, but to handle a few scenarios, foreign-key name matching has three options. The figure shows all three options for a foreign-key name using the entity class `Review` that references the primary key, `BookId`, in the entity class `Book`.



![](./diagrams/images/08_02_foreign_key.png)



````c#
// A hierarchical relationship with an option-3 foreign key
public class Employee
{
    public int EmployeeId { get; set; }

    public string Name { get; set; }

    //------------------------------
    //Relationships

    public int? ManagerEmployeeId { get; set; } // Foreign key uses the <NavigationalPropertyName><PrimaryKeyName> pattern
    public Employee Manager { get; set; }
}
````

The entity class called `Employee` has a navigational property called `Manager` that links to the employee’s manager, who is an employee as well. You can’t use a foreign key of `EmployeeId` (option 1), because it’s already used for the primary key. Therefore, you use option 3 and call the foreign key `ManagerEmployeeId` by using the navigational property name at the start.



#### Nullability of foreign keys: Required or optional dependent relationships

The nullability of the foreign key defines whether the relationship is required (nonnullable foreign key) or optional (nullable foreign key). A *required relationship* ensures that relationships exist by ensuring that the foreign key is linked to a valid principal key.

An *optional* relationship allows there to be no link between the principal entity and the dependent entity by having the foreign-key value(s) set to `null`. The `Manager` navigational property in the `Employee` entity class, is an example of an optional relationship, as someone at the top of the business hierarchy won’t have a boss.

The required or optional status of the relationship also affects what happens to dependent entities when the principal entity is deleted. The default setting of the `OnDelete` action for each relationship type is as follows:

* For a *required relationship*, EF Core sets the `OnDelete` action to `Cascade`. If the principal entity is deleted, the dependent entity will be deleted too.
* For a *optional relationship*, EF Core sets the `OnDelete` action to `ClientSetNull`. If the dependent entity is being tracked, the foreign key will be set to `null` when the principal entity is deleted. But if the dependent entity isn’t being tracked, the database constraint delete setting takes over, and the `ClientSetNull` setting sets the database rules as though the `Restrict` setting were in place. The result is that the delete fails at the database level, and an exception is thrown.



#### Foreign keys: What happens if you leave them out?

If EF Core finds a relationship via a navigational property or through a relationship you configured via the Fluent API, it needs a foreign key to set up the relationship in the relational database. Including foreign keys in your entity classes is good practice, giving you better control of the nullability of the foreign key. Also, access to foreign keys can be useful when you’re handling relationships in a disconnected update.

But if you do leave out a foreign key (on purpose or by accident), EF Core configuration will add a foreign key as a shadow property. *Shadow properties* are hidden properties that can be accessed only via specific EF Core commands. Having foreign keys added automatically as shadow properties can be useful. One of my clients, for example, had a general `Note` entity class that was added to a `Notes` collection in many entities.

A one-to-many relationship in which the `Note` entity class is used in a collection navigational property in two entity classes: `Customer` and `Job`. Note that the primary-key names of the `Customer` and `Job` entity classes use different By Convention naming approaches to show how the shadow properties are named.



![](./diagrams/images/08_03_shadow_properties.png)



If the entity class that gains a shadow property foreign key has a navigational link to the other end of the relationship, the name of that shadow property would be `<navigation property name><principal key property name>`. If the `Note` entity has a navigational link back to the `Customer` entity called `LinkBack`, the shadow property foreign key’s name would be `LinkBackId`.

If you want to add a foreign key as a shadow property, you can do that via the Fluent API `HasForeignKey`, but with the name of the shadow property name provided via a string. Be careful not to use the name of an existing property, as that will not add a shadow property but will use the existing property.



#### When does By Convention configuration not work?

If you’re going to use the By Convention configuration approach, you need to know when it’s not going to work so that you can use other means to configure your relationship. Here’s my list of scenarios that won’t work, with the most common listed first:

* You have composite foreign keys.
* You want to create a one-to-one relationship without navigational links going both ways.
* You want to override the default delete-behavior setting.
* You have two navigational properties going to the same class.
* You want to define a specific database constraint.



### Configuring relationships by using Data Annotations

Only two Data Annotations relate to relationships, as most of the navigational configuration is done via the Fluent API: the `ForeignKey` and `InverseProperty` annotations.



#### The ForeignKey Data Annotation

````c#
// Using the ForeignKey data annotation to set the foreign-key name
public class Employee
{
    public int EmployeeId { get; set; }
    public string Name { get; set; }

    public int? ManagerId { get; set; }
    [ForeignKey(nameof(ManagerId))] // Defines which property is the foreign key for the Manager navigational property
    public Employee Manager { get; set; }
}
````

The `ForeignKey` data annotation takes one parameter, which is a string. This string should hold the name of the foreign-key property. If the foreign key is a composite key (has more than one property), it should be comma-delimited - as in `[ForeignKey("Property1, Property2")]`.



#### The InverseProperty Data Annotation

The `InverseProperty` Data Annotation is a rather specialized Data Annotation for use when you have two navigational properties going to the same class. At that point, EF Core can’t work out which foreign keys relate to which navigational property. This situation is best shown in code. The following listing shows an example `Person` entity class with two lists: one for books owned by the librarian and one for `Books` out on loan to a specific person.

````c#
// LibraryBook entity class with two relationships to Person class
public class LibraryBook
{
    public int LibraryBookId { get; set; }
    public string Title { get; set; }

    public int LibrarianPersonId { get; set; }
    public Person Librarian { get; set; }

    public int? OnLoanToPersonId { get; set; }
    public Person OnLoanTo { get; set; }
}
````

The `Librarian` and the borrower of the book (`OnLoanTo` navigational property) are both represented by the `Person` entity class. The `Librarian` navigational property and the `OnLoanTo` navigational property both link to the same class, and EF Core can’t set up the navigational linking without help. The `InverseProperty` Data Annotation shown in the following listing provides the information to EF Core when it’s configuring the navigational links.

````c#
// The Person entity class, which uses the InverseProperty annotation
public class Person
{
    public int PersonId { get; set; }
    public string Name { get; set; }

    [InverseProperty("Librarian")] // Links LibrarianBooks to the Librarian navigational property in the LibraryBook class
    public ICollection<LibraryBook> LibrarianBooks { get; set; }

    [InverseProperty("OnLoanTo")] // Links the BooksBorrowedByMe list to the OnLoanTo navigational property in the LibraryBook class
    public ICollection<LibraryBook> BooksBorrowedByMe { get; set; }
}
````

This code is one of those configuration options that you rarely use, but if you have this situation, you must either use it or define the relationship with the Fluent API. Otherwise, EF Core will throw an exception when it starts, as it can’t work out how to configure the relationships.



### Fluent API relationship configuration commands

All Fluent API relationship configuration commands follow this pattern.



![](./diagrams/images/08_04_fluent_api.png)



#### Creating a one-to-one relationship

One-to-one relationships can get a little complicated because there are three ways to build them in a relational database.



![](./diagrams/svg/08_01_one_to_one_relationship.drawio.svg)



Option 1 is the standard approach to building one-to-one relationships, because it allows you to define that the one-to-one dependent entity is required (must be present). In our example, an exception will be thrown if you try to save an `Attendee` entity instance without a unique `Ticket` attached to it.

With the option-1 one-to-one arrangement, you can make the dependent entity optional by making the foreign key nullable. `WithOne` method has a parameter that picks out the `Attendee` navigational property in the `Ticket` entity class that links back to the `Attendee` entity class. Because the `Attendee` class is the dependent part of the relationship, if you delete the `Attendee` entity, the linked `Ticket` won’t be deleted, because the `Ticket` is the principal entity in the relationship. The downside of option 1 in this example is that it allows one `Ticket` to be used for multiple `Attendees`, which doesn’t match the business rules I stated at the start. Finally, this option allows you to replace `Ticket` with another `Ticket` instance by assigning a new `Ticket` to the Attendee’s `Ticket` navigational property.




![](./diagrams/svg/08_02_one_to_one_relationship.drawio.svg)




Options 2 and 3 turn the principal/dependent relationship around, with the Attendee becoming the principal entity in the relationship. This situation swaps the required/optional nature of the relationship. Now the Attendee can exist without the Ticket, but the Ticket can’t exist without the Attendee. Options 2 and 3 do enforce the assignment of a Ticket to only one Attendee, but replacing Ticket with another Ticket instance requires you to delete the old ticket first.



![](./diagrams/svg/08_03_one_to_one_relationship.drawio.svg)



Option 2 and 3 are useful because they form optional one-to-one relationships, often referred to as *one-to-zero-or-one* relationships. Option 3 is a more efficient way to define option 2, with the primary key and the foreign key combined. Another, even better version uses an `Owned` type because it is automatically loaded from the same table, which is safer and more efficient.



#### Creating a one-to-many relationship

One-to-many relationships are simpler, because there’s one format: the many entities contain the foreign-key value. You can define most one-to-many relationships with the By Convention approach simply by giving the foreign key in the many entities a name that follows the By Convention approach.



![](./diagrams/svg/08_04_one_to_many_relationship.drawio.svg)



You can use any generic type for a collection that implements the `IEnumerable<T>` interface, such as `IList<T>`, `Collection<T>`, `HashSet<T>`, `List<T>`, and so on. `IEnumerable<T>` on its own is a special case, as you can’t add to that collection.

For performance reasons, you should use `HashSet<T>` for navigational collections, because it improves certain parts of EF Core’s query and update processes. But `HashSet` doesn’t guarantee the order of entries, which could cause problems if you add sorting to your Includes. If you might sort your Include methods, as `ICollection` preserves the order in which entries are added. You don’t use sort in `Include`s so that you can use `HashSet<T>` for better performance.

Second, although you typically define a collection navigational property with a getter and a setter (such as `public ICollection<Review> Reviews { get; set; }`), doing so isn’t necessary. You can provide a getter only if you initialize the backing field with an empty collection. The following is also valid:

````c#
public ICollection<Review> Reviews { get; } = new List<Review>();
````



Although initializing the collection might make things easier in this case, we don’t recommend initializing a navigational collection property.



#### Creating a many-to-many relationship

* *Your linking table contains information that you want to access when reading in the data on the other side of the many-to-many relationship.* An example is the `Book` to `Author` many-to-many relationship, in which the linking table contains the order in which the `Author` `Name`s should be shown.
* *You directly access the other side of the many-to-many relationship.* An example is the `Book` to `Tag` many-to-many relationship, in which you can directly access the `Tags` collection in the `Book` entity class without ever needing to access the linking table.



**CONFIGURING A MANY-TO-MANY RELATIONSHIP USING A LINKING ENTITY CLASS**

You start with the many-to-many relationship in which you access the other end of the relationship via the linking table. This relationship takes more work but allows you to add extra data to the linking table, which you can sort/filter on.



![](./diagrams/svg/08_05_many_to_many_relationship.drawio.svg)



In the `Book`/`Author` example, the By Convention configuration can find and link all the scalar and navigational properties so that the only configuration required is setting up the primary key.



````c#
// Configuring a many-to-many relationship via two one-to-many relationships
public static void Configure(this EntityTypeBuilder<BookAuthor> entity)
{
    entity.HasKey(p => new { p.BookId, p.AuthorId }); // Uses the names of the Book and Author primary keys to form its own composite key

    //-----------------------------
    //Relationships

    entity.HasOne(p => p.Book)
        .WithMany(p => p.AuthorsLink)
        .HasForeignKey(p => p.BookId); // Configures the one-to-many relationship from the Book to BookAuthor entity class

    entity.HasOne(p => p.Author)
        .WithMany(p => p.BooksLink)
        .HasForeignKey(p => p.AuthorId); // Configures the one-to-many relationship from the Author to the BookAuthor entity class
}
````



**CONFIGURING A MANY-TO-MANY RELATIONSHIP WITH DIRECT ACCESS TO THE OTHER ENTITY**

The By Convention configuration works well for a direct many-to-many relationship. If the entity classes at the two ends are valid, the By Convention configuration will set up the relationships and keys for you. By Convention will also create the linking entity for you by using a property bag.



![](./diagrams/svg/08_06_many_to_many_relationship.drawio.svg)



But if you want to add your own linking table and configuration, you can do that via Fluent API configuration. The entity class for the linking table is similar to the `BookAuthor` linked entity class. The difference is that the `Author` key/relationship is replaced by the `Tag` key/relationship. The following listing shows the `Book` configuration class setting up the `BookTag` entity class to link the two parts.

````c#
// Configuring direct many-to-many relationships using Fluent API
public void Configure(EntityTypeBuilder<Book> entity)
{
    //… other configrations left out for clarity

    entity.HasMany(x => x.Tags)
        .WithMany(x => x.Books) // The HasMany/WithMany sets up a direct many-to-many relationship.
        .UsingEntity<BookTag>( // The UsingEntity<T> method allows you to define an entity class for the linking table.
        bookTag => bookTag.HasOne(x => x.Tag)
        	.WithMany().HasForeignKey(x => x.TagId), // Defined Tag side of the many-to-many relationship
        bookTag => bookTag.HasOne(x => x.Book)
        	.WithMany().HasForeignKey(x => x.BookId)); // Defined Book side of the many-to-many relationship
}
````



### Controlling updates to collection navigational properties

Sometimes, you need to control access to collection navigational properties. Although you can control access to a one-to-one navigational by making the setter private, that approach doesn’t work for a collection, as most collection types allow you to add or remove entries.

Storing the collection of linked entities classes in a field allows you to intercept any attempt to update a collection. Here are some business/software design reasons why this feature is useful:

* Triggering some business logic on a change, such as calling a method if a collection contains more than ten entries.
* Building a local cached value for performance reasons, such as holding a cached `ReviewsAverageVotes` property whenever a `Review` is added to or removed from your `Book` entity class.
* Applying a DDD to your entity classes. Any change to data should be done via a method.

For the example of controlling collection navigational properties, you are going to add a cached `ReviewsAverageVotes` property to the `Book` class. This property will hold the average of the votes in all the `Reviews` linked to this `Book`. To do so, you need to

* Add a backing field called `_reviews` to hold the `Reviews` collection and change the property to return a read-only copy of the collection held in the `_reviews` backing field.
* Add a read-only property called `ReviewsAverageVotes` to hold the cached average votes from the `Reviews` linked to this `Book`.
* Add methods to Add `Reviews` to and `Remove Reviews` from the `_reviews` backing field. Each method recalculates the average votes, using the current list of `Reviews`.

The following listing shows the updated `Book` class showing the code related to the `Reviews` and the cached `ReviewsAverageVotes` property.

````c#
// Book class with a read-only Reviews collection navigational property
public class Book
{
    private readonly ICollection<Review> _reviews = new List<Review>(); // You add a backing field, which is a list. By default, EF Core will read and write to this field.

    public int BookId { get; set; }
    public string Title { get; set; }
    //… other properties/relationships left out

    public double? ReviewsAverageVotes { get; private set; } // Holds a precalculated average of the reviews and is read-only

    // A read-only collection so that no one can change the collection
    public IReadOnlyCollection<Review> Reviews => _reviews.ToList(); // Returns a copy of the reviews in the _reviews backing field

    public void AddReview(Review review) // Adds a method to allow a new Review to be added to the _reviews collection
    {
        _reviews.Add(review); // Adds the new review to the backing field _reviews and updates the database on the call to SaveChanges
        ReviewsAverageVotes = _reviews.Average(x => x.NumStars); // Recalculates the average votes for the book
    }

    public void RemoveReview(Review review) // Adds a method to remove a review from the _reviews collection
    {
        _reviews.Remove(review); // Removes the review from the list and updates the database on the call to SaveChanges
        ReviewsAverageVotes = _reviews.Any()
            ? _reviews.Average(x => x.NumStars) // If there are any reviews, recalculates the average votes for the book
            : (double?)null; // If there are no reviews, sets the value to null
    }
}
````



You didn’t have to configure the backing field because you were using By Convention naming, and by default, EF Core reads and writes data to the `_reviews` field.

This example shows how to make your collection navigational properties read-only, but it’s not perfect because concurrent updates could make the `ReviewsAverageVotes` cache property out of date.



### Additional methods available in Fluent API relationships

* `OnDelete` - Changes the delete action of a dependent entity
* `IsRequired` - Defines the nullability of the foreign key
* `HasPrincipalKey` - Uses an alternate unique key
* `HasConstraintName` - Sets the foreign-key constraint name and `MetaData` access to the relationship data



#### OnDelete: Changing the delete action of a dependent entity

The `OnDelete` Fluent API method allows you to alter what EF Core does when a deletion that affects a dependent entity occurs.

You can add the `OnDelete` method to the end of a Fluent API relationship configuration. The code added to stop a `Book` entity from being deleted if it was referred to in a customer order, via the `LineItem` entity class.

````c#
// Changing the default OnDelete action on a dependent entity
public static void Configure(this EntityTypeBuilder<LineItem> entity)
{
    entity.HasOne(p => p.ChosenBook)
        .WithMany()
        .OnDelete(DeleteBehavior.Restrict);
}
````

This code causes an exception to be thrown if someone tries to delete a `Book` entity that a `LineItem`’s foreign key links to that `Book`. You do this because you want a customer’s order to not be changed.



The table explains the possible `DeleteBehavior` settings:

| Name          | Effect of the delete behavior on the dependent entity        | Default for            |
| ------------- | ------------------------------------------------------------ | ---------------------- |
| Restrict      | The delete operation isn’t applied to dependent entities. The dependent entities remain unchanged, which may cause the delete to fail, either in EF Core or in the relational database. |                        |
| SetNull       | The dependent entity isn’t deleted, but its foreign-key property is set to null. If any of the dependent entity foreign-key properties isn’t nullable, an exception is thrown when SaveChanges is called. |                        |
| ClientSetNull | If EF Core is tracking the dependent entity, its foreign key is set to null, and the dependent entity isn’t deleted. But if EF Core isn’t tracking the dependent entity, the database rules apply. In a database created by EF Core, this DeleteBehavior will set the SQL DELETE constraint to NO ACTION, which causes the delete to fail with an exception. | Optional relationships |
| Cascade       | The dependent entity is deleted.                             | Required relationships |
| ClientCascade | For entities being tracked by the DbContext, dependent entities will be deleted when the related principal is deleted. But if EF Core isn’t tracking the dependent entity, the database rules apply. In a database created by EF Core, this will be set to Restrict, which causes the delete to fail with an exception. |                        |

Two delete behaviors whose names start with `Client` are `ClientSetNull` and `ClientCascade`. These two delete behaviors move some of the handling of deletion actions from the database to the client - that is, the EF Core code. These two settings have been added to prevent the problems you can get in some databases, such as SQL Server, when your entities have navigational links that loop back to themselves. In these cases, you would get an error from the database server when you try to create your database, which can be hard to diagnose and fix.

In both cases, these commands execute code inside EF Core that does the same job that the database would do with the `SetNull` and `Cascade` delete behaviors, respectively. But EF Core can apply these changes only if you have loaded all the relevant dependent entities linked to the principal entity that you are going to delete. If you don’t, the database applies its delete rules, which normally will throw an exception.

The `ClientSetNull` delete setting is the default for optional relationships, and EF Core will set the foreign key of the loaded dependent entity class to `null`. If you use EF Core to create/migrate the database, EF Core sets the database delete rules to `ON DELETE NO ACTION` (SQL Server). The database server won’t throw an exception if your entities have a circular loop (referred to as possible cyclic delete paths by SQL Server). The `SetNull` delete setting would set the database delete rules to `ON DELETE SET NULL` (SQL Server), which would cause the database server to throw a `possible cyclic delete paths` exception.

The `ClientCascade` delete setting does the same thing for the database’s cascade-delete feature, in that it will delete any loaded dependent entity class(es). Again, if you use EF Core to create/migrate the database, EF Core sets the database delete rules to `ON DELETE NO ACTION` (SQL Server). The `Cascade` delete setting would set the database delete rules to `ON DELETE CASCADE` (SQL Server), which would cause the database server to throw a `possible cyclic delete paths` exception.

The entity in this listing is loaded with an optional dependent entity, which (by default) has the default delete behavior of `ClientSetNull`. But the same code would work for the `ClientCascade` as long as you load the correct dependent entity or entities.

````c#
// Deleting a principal entity with an optional dependent entity
var entity = context.DeletePrincipals // Reads in the principal entity
    .Include(p => p.DependentDefault) // Includes the dependent entity that has the default delete behavior of ClientSetNull
    .Single(p => p.DeletePrincipalId == 1);

context.Remove(entity); // Sets the principal entity for deletion
context.SaveChanges(); // Calls SaveChanges, which sets its foreign key to null
````

Note that if you don’t include the `Include` method or another way of loading the optional dependent entity, `SaveChanges` will throw a `DbUpdateException` because the database server will have reported a foreign-key constraint violation. One way to align EF Core’s approach to an optional relationship with the database server’s approach is to set the delete behavior to `SetNull` instead of the default `ClientSetNull`, making the foreign-key constraint in the database `ON DELETE SET NULL` (SQL Server) and putting the database in charge of setting the foreign key to `null`. Whether or not you load the optional dependent entity, the outcome of the called `SaveChanges` will be the same: the foreign key on the optional dependent entity will be set to `null`.

But be aware that some database servers may return an error on database creation if you have a delete-behavior setting of `SetNull` or `Cascade` and the servers detect a possible circular relationship, such as hierarchical data. That’s why EF Core has the `ClientSetNull` and `ClientCascade` delete behaviors.

If you’re managing the database creation/migration outside EF Core, it’s important to ensure that the relational database foreign-key constraint is in line with EF Core’s `OnDelete` setting. Otherwise, you’ll get inconsistent behavior, depending on whether the dependent entity is being tracked.



#### IsRequired: Defining the nullability of the foreign key

In a relationship, the same command sets the nullability of the foreign key, defines whether the relationship is required or optional.

The `IsRequired` method is most useful in shadow properties because EF Core makes shadow properties nullable by default, and the `IsRequired` method can change them to non-nullable.



````c#
// The Attendee entity class showing all its relationships
public class Attendee
{
    public int AttendeeId { get; set; }
    public string Name { get; set; }

    public int TicketId { get; set; } // Foreign key for the one-to-one relationship, Ticket
    public Ticket Ticket { get; set; } // One-to-one navigational property that accesses the Ticket entity

    public MyOptionalTrack Optional { get; set; } // One-to-one navigational property using a shadow property for the foreign key. By default, the foreign key is nullable, so the relationship is optional.
    public MyRequiredTrack Required { get; set; } // One-to-one navigational property using a shadow property for the foreign key. You use Fluent API commands to say that the foreign key isn’t nullable, so the relationship is required.
}
````

The `Optional` navigational property, which uses a shadow property for its foreign key, is configured by convention, which means that the shadow property is left as a nullable value. Therefore, it’s optional, and if the `Attendee` entity is deleted, the `MyOptionalTrack` entity isn’t deleted.



You use the `IsRequired` method to make the `Required` one-to-one navigational property as required. Each `Attendee` entity must have a `MyRequiredTrack` entity assigned to the `Required` property.

````c#
// The Fluent API configuration of the Attendee entity class
public void Configure(EntityTypeBuilder<Attendee> entity)
{
    entity.HasOne(attendee => attendee.Ticket) // Sets up the one-to-one navigational relationship, Ticket, which has a foreign key defined in the Attendee class
        .WithOne(attendee => attendee.Attendee)
        .HasForeignKey<Attendee>(attendee => attendee.TicketId) // Specifies the property that’s the foreign key. You need to provide the class type, as the foreign key could be in the principal or dependent entity class.
        .IsRequired();

    entity.HasOne(attendee => attendee.Required) // Sets up the one-to-one navigational relationship, Required, which doesn’t have a foreign key defined
        .WithOne(attendee => attendee.Attend)
        .HasForeignKey<Attendee>("MyShadowFk") // Uses the HasForeignKey<T> method, which takes a string because it’s a shadow property and can be referred to only via a name. Note that you use your own name.
        .IsRequired(); // Uses IsRequired to say the foreign key should not be nullable
}
````

You could’ve left out the configuration of the `Ticket` navigational property, as it would be configured correctly under the By Convention rules, but you leave it in so that you can compare it with the configuration of the `Required` navigational property, which uses a shadow property for its foreign key. The configuration of the `Required` navigational property is necessary because the `IsRequired` method changes the shadow foreign-key property from nullable to non-nullable, which in turn makes the relationship required.



**TYPE AND NAMING CONVENTIONS FOR SHADOW PROPERTY FOREIGN KEYS**

The `<T>` class tells EF Core where to place the shadow foreign-key property, which can be either end of the relationship for one-to-one relationships or the many entity class of a one-to-many relationship.

The string parameter of the `HasForeignKey<T>(string)` method allows you to define the shadow foreign-key property name. You can use any name; you don’t need to stick with the By Convention name. But you need to be careful not to use a name of any existing property in the entity class you’re targeting, because that approach can lead to strange behaviors. (There’s no warning if you do select an existing property, as you might be trying to define a nonshadow foreign key.)



#### HasPrincipalKey: Using an alternate unique key

The following listing creates a `Person` entity class, which uses a normal `int` primary key, but you’ll use the `UserId` as an alternate key when linking to the person’s contact information.

````c#
// Person class, with Name taken from ASP.NET authorization
public class Person
{
    public int PersonId { get; set; }

    public string Name { get; set; }

    public Guid UserId { get; set; } // Holds the person’s unique Id

    public ContactInfo ContactInfo { get; set; } // Navigational property linking to the ContactInfo
}
````

````c#
// ContactInfo class with EmailAddress as a foreign key
public class ContactInfo
{
    public int ContactInfoId { get; set; }

    public string MobileNumber { get; set; }
    public string LandlineNumber { get; set; }

    public Guid UserIdentifier { get; set; } // The UserIdentifier is used as a foreign key for the Person entity to link to this contact info.
}
````

The Fluent API configuration commands, which use the alternate key in the `Person` entity class as a foreign key in the `ContactInfo` entity class.

Here are a few notes on alternate keys:

* You can have composite alternate keys, which are made up of two or more properties. You handle them in the same way that you do composite keys: by using an anonymous Type, such as `HasPrincipalKey<MyClass>(c => new {c.Part1, c.Part2}`).
* Unique keys and alternate keys are different, and you should choose the correct one for your business case. Here are some of the differences:
  * Unique keys ensure that each entry is unique; they can’t be used in a foreign key.
  * Unique keys can be null, but alternate keys can’t.
  * Unique key values can be updated, but alternate keys can’t.
* You can define a property as a standalone alternate key by using the Fluent API command `modelBuilder.Entity<Car>().HasAlternateKey(c => c.LicensePlate)`, but you don’t need to do that, because using the `HasPrincipalKey` method to set up a relationship automatically registers the property as an alternate key.



![](./diagrams/svg/08_07_has_principal_key.drawio.svg)



#### Less-used options in Fluent API relationships

**HASCONSTRAINTNAME: SETTING THE FOREIGN-KEY CONSTRAINT NAME**

The method `HasConstraintName` allows you to set the name of the foreign-key constraint, which can be useful if you want to catch the exception on foreign-key errors and use the constraint name to form a more user-friendly error message.



**METADATA: ACCESS TO THE RELATIONSHIP INFORMATION**

The `MetaData` property provides access to the relationship data, some of which is read/write. Much of what the `MetaData` property exposes can be accessed via specific commands, such as `IsRequired`, but if you need something out of the ordinary, look through the various methods/properties supported by the `MetaData` property.



### Alternative ways of mapping entities to database tables

Sometimes, it’s useful to not have a one-to-one mapping from an entity class to a database table. Instead of having a relationship between two classes, you might want to combine both classes into one table. This approach allows you to load only part of the table when you use one of the entities, which will improve the query’s performance.

Five alternative ways to map classes to the database, each with advantages in certain situations:

* *Owned types* - Allows a class to be merged into the entity class’s table and is useful for using normal classes to group data.
* *Table per hierarchy (TPH)* - Allows a set of inherited classes to be saved in one table, such as classes called `Dog`, `Cat`, and `Rabbit` that inherit from the `Animal` class.
* *Table per type (TPT)* - Maps each class to a different table. This approach works like TPH except that each class is mapped to a separate table.
* *Table splitting* - Allows multiple entity classes to be mapped to the same table and is useful when some columns in a table are read more often than all the table columns.
* *Property bags* - Allows you to create an entity class via a `Dictionary`, which gives you the option to create the mapping on startup. Property bags also use two other features: mapping the same type to multiple tables and using an indexer in your entity classes.



#### Owned types: Adding a normal class into an entity class

EF Core has *owned types*, which allow you to define a class that holds a common grouping of data, such as an address or audit data, that you want to use in multiple places in your database. The owned type class doesn’t have its own primary key, so it doesn’t have an identity of its own; it relies on the entity class that "owns" it for its identity. In DDD terms, owned types are known as *value objects*.

Here are two ways of using owned types:

* The owned type data is held in the same table that the entity class is mapped to.
* The owned type data is held in a separate table from the entity class.



**OWNED TYPE DATA IS HELD IN THE SAME TABLE AS THE ENTITY CLASS**

As an example of an owned type, you’ll create an entity class called `OrderInfo` that needs two addresses: `BillingAddress` and `DeliveryAddress`. These addresses are provided by the `Address` class, shown in the following listing. You can mark an `Address` class as an owned type by adding the attribute `[Owned]` to the class. An owned type has no primary key, as shown at the bottom of the listing.

````c#
// The Address owned type, followed by the OrderInfo entity class
public class OrderInfo // The entity class OrderInfo, with a primary key and two addresses
{
    public int OrderInfoId { get; set; }
    public string OrderNumber { get; set; }

    // Two distinct Address classes. The data for each Address class will be included in the table that the OrderInfo is mapped to.
    public Address BillingAddress { get; set; }
    public Address DeliveryAddress { get; set; }
}

[Owned] // The attribute [Owned] tells EF Core that it is an owned type.
public class Address // An owned type has no primary key.
{
    public string NumberAndStreet { get; set; }
    public string City { get; set; }
    public string ZipPostCode { get; set; }
    [Required]
    [MaxLength(2)]
    public string CountryCodeIso2 { get; set; }
}
````

Because you added the attribute `[Owned]` to the `Address` class, and because you are using the owned type within the same table, you don’t need use the Fluent API to configure the owned type. This approach saves you time, especially if your owned type is used in many places, because you don’t have to write the Fluent API configuration. But if you don’t want to use the `[Owned]` attribute, the next listing shows you the Fluent API to tell EF Core that the `BillingAddress` and the `DeliveryAddress` properties in the `OrderInfo` entity class are owned types, not relationships.

````c#
// The Fluent API to configure the owned types within OrderInfo
public class SplitOwnDbContext: DbContext
{
    public DbSet<OrderInfo> Orders { get; set; }
    //… other code removed for clarity

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBulder.Entity<OrderInfo>() // Selects the owner of the owned type
            .OwnsOne(p => p.BillingAddress); // Uses the OwnsOne method to tell EF Core that property BillingAddress is an owned type and that the data should be added to the columns in the table that the OrderInfo maps to
        modelBulder.Entity<OrderInfo>()
            .OwnsOne(p => p.DeliveryAddress); // Repeats the process for the second property, DeliveryAddress
    }
}
````

The result is a table containing the two scalar properties in the `OrderInfo` entity class, followed by two sets of Address class properties, one prefixed by `BillingAddress_` and another prefixed by `DeliveryAddress_`. Because an owned type property can be `null`, all the properties are held in the database as nullable columns. The `CountryCodeIso2` property, for example, is marked as `[Required]`, so it should be nonnullable, but to allow for a `null` property value for the `BillingAddress` or `DeliveryAddress`, it is stored in a nullable column. EF Core does this to tell whether an instance of the owned type should be created when the entity containing an owned type is read in.

The fact that the owned type property can be `null` means that owned types within an entity class are a good fit for what DDD calls a *value object*. A value object has no key, and two value objects with the same properties are considered to be equal. The fact that they can be `null` allows for an "empty" value object.



````sql
-- The SQL CREATE TABLE command showing the column names
CREATE TABLE [Orders] (
    [OrderInfoId] int NOT NULL IDENTITY,
    [OrderNumber] nvarchar(max) NULL,
    [BillingAddress_City] nvarchar(max) NULL,
    [BillingAddress_NumberAndStreet] nvarchar(max) NULL,
    [BillingAddress_ZipPostCode] nvarchar(max) NULL,
    [BillingAddress_CountryCodeIso2] [nvarchar](2) NULL
    [DeliveryAddress_City] nvarchar(max) NULL,
    [DeliveryAddress_NumberAndStreet] nvarchar(max) NULL,
    [DeliveryAddress_ZipPostCode] nvarchar(max) NULL,
    [DeliveryAddress_CountryCodeIso2] [nvarchar](2) NULL, -- Property has a [Required] attribute but is stored as a nullable value to handle the billing/delivery address being null.
    CONSTRAINT [PK_Orders] PRIMARY KEY ([OrderInfoId])
);
````

By default, every property or field in an owned type is stored in a nullable column, even if they are non-nullable. EF Core does this to allow you to not assign an instance to an owned type, at which point all the columns that the owned type uses are set to NULL. And if an entity with an owned type is read in, and all the columns for an owned type are NULL, the owned type property is set to `null`.

But EF Core allow you to say that an owned type is required - that is, must always be present. To do so, you add the Fluent API `IsRequired` method to the `OrderInfo`’s `DeliveryAddress` navigational property mapped to the owned type (see the next listing). In addition, this feature allows the individual nullability of columns to follow normal rules.

````c#
// The Fluent API to configure the owned types within OrderInfo
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBulder.Entity<OrderInfo>()
        .OwnsOne(p => p.BillingAddress);
    modelBulder.Entity<OrderInfo>()
        .OwnsOne(p => p.DeliveryAddress);

    modelBulder.Entity<OrderInfo>() // Selects the DeliveryAddress navigational property
        .Navigation(p => p.DeliveryAddress)
        .IsRequired(); // Applying the IsRequired method means that the DeliveryAddress must not be null.
}
````

Using owned types can help you organize your database by turning common groups of data into owned types, making it easier to handle common data groups, such as Address and so on, in your code. Here are some final points on owned types held in an entity class:

* The owned type navigation properties, such as `BillingAddress`, are automatically created and filled with data when you read the entity. There’s no need for an `Include` method or any other form of relationship loading.
* Owned types have better performance and are automatically loaded, which means that they would be better implementations of the zero-or-one `PriceOffer` used in the Book App.
* Owned types can be nested. You could create a `CustomerContact` owned type, which in turn contains an `Address` owned type, for example. If you used the `CustomerContact` owned type in another entity class - let’s call it `SuperOrder` - all the `CustomerContact` properties and the `Address` properties would be added to the `SuperOrder`’s table.



**OWNED TYPE DATA IS HELD IN A SEPARATE TABLE FROM THE ENTITY CLASS**

The other way that EF Core can save the data inside an owned type is in a separate table rather than the entity class. In this example, you’ll create a `User` entity class that has a property called `HomeAddress` of type `Address`. In this case, you add a `ToTable` method after the `OwnsOne` method in your configuration code.



````c#
// Configuring the owned table data to be stored in a separate table
public class SplitOwnDbContext: DbContext
{
    public DbSet<OrderInfo> Orders { get; set; }
    //… other code removed for clarity

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBulder.Entity<User>()
            .OwnsOne(p => p.HomeAddress)
            .ToTable("Addresses"); // Adding ToTable to OwnsOne tells EF Core to store the owned type, Address, in a separate table, with a primary key equal to the primary key of the User entity that was saved to the database.
    }
}
````

EF Core sets up a one-to-one relationship in which the primary key is also the foreign key, and the `OnDelete` state is set to `Cascade` so that the owned type entry of the primary entity, `User`, is deleted. Therefore, the database has two tables: Users and Addresses.

````sql
-- The Users and Addresses tables in the database
CREATE TABLE [Users] (
    [UserId] int NOT NULL IDENTITY,
    [Name] nvarchar(max) NULL,
    CONSTRAINT [PK_Orders] PRIMARY KEY ([UserId])
);
CREATE TABLE [Addresses] (
    [UserId] int NOT NULL IDENTITY,
    [City] nvarchar(max) NULL,
    [CountryCodeIso2] nvarchar(2) NOT NULL, -- Notice that non-nullable properties, or nullable properties with the Required setting, are now stored in non-nullable columns.
    [NumberAndStreet] nvarchar(max) NULL,
    [ZipPostCode] nvarchar(max) NULL,
    CONSTRAINT [PK_Orders] PRIMARY KEY ([UserId]),
    CONSTRAINT "FK_Addresses_Users_UserId" FOREIGN KEY ("UserId")
    	REFERENCES "Users" ("UserId") ON DELETE CASCADE
);
````

This use of owned types differs from the first use, in which the data is stored in the entity class table, because you can save a `User` entity instance without an address. But the same rules apply on querying: the `HomeAddress` property will be read in on a query of the `User` entity without the need for an `Include` method.

The Addresses table used to hold the `HomeAddress` data is hidden; you can’t access it via EF Core. This situation could be a good thing or a bad thing, depending on your business needs. But if you want to access the `Address` part, you can implement the same feature by using two entity classes with a one-to-many relationship between them.



#### Table per hierarchy (TPH): Placing inherited classes into one table

Table per hierarchy (TPH) stores all the classes that inherit from one another in a single database table. If you want to save a payment in a shop, for example, that payment could be cash (`PaymentCash`) or credit card (`PaymentCard`). Each option contains the amount (say, $10), but the credit card option has extra information, such as an online-transaction receipt. In this case, TPH uses a single table to store all the versions of the inherited classes and return the correct entity type, `PaymentCash` or `PaymentCard`, depending on what was saved.

TPH can be configured By Convention, which will combine all the versions of the inherited classes into one table. This approach has the benefit of keeping common data in one table, but accessing that data is a little cumbersome because each inherited type has its own `DbSet<T>` property. But when you add the Fluent API, all the inherited classes can be accessed via one `DbSet<T>` property, which in our example makes the `PaymentCash / PaymentCard` example much more useful.

The first example uses multiple `DbSet<T>`s, one for each class, and is configured By Convention. The second example uses one `DbSet<T>` mapped to the base class and shows the TPH Fluent API commands.



**CONFIGURING TPH BY CONVENTION**

````c#
// The two classes: PaymentCash and PaymentCard
public class PaymentCash
{
    [Key]
    public int PaymentId { get; set; }
    public decimal Amount { get; set; }
}

//PaymentCredit – inherits from PaymentCash
public class PaymentCard : PaymentCash
{
    public string ReceiptCode { get; set; }
}
````

````c#
// The updated application’s DbContext with the two DbSet<T> properties
public class Chapter08DbContext : DbContext
{
    //… other DbSet<T> properties removed
    //Table-per-hierarchy
    public DbSet<PaymentCash> CashPayments { get; set; }
    public DbSet<PaymentCard> CreditPayments { get; set; }

    public Chapter08DbContext(DbContextOptions<Chapter08DbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //no configuration needed for PaymentCash or PaymentCard
    }
}
````

````sql
-- The SQL produced by EF Core to build the CashPayment table
CREATE TABLE [CashPayments] (
    [PaymentId] int NOT NULL IDENTITY,
    [Amount] decimal(18, 2) NOT NULL,
    [Discriminator] nvarchar(max) NOT NULL, -- The Discriminator column holds the name of the class, which EF Core uses to define what sort of data is saved. When set by convention, this column holds the name of the class as a string.
    [ReceiptCode] nvarchar(max), -- The ReceiptCode column is used only if it’s a PaymentCredit.
    CONSTRAINT [PK_CashPayments]
    	PRIMARY KEY ([PaymentId])
);
````

EF Core has added a Discriminator column, which it uses when returning data to create the correct type of class: `PaymentCash` or `PaymentCard`, based on what was saved. Also, the `ReceiptCode` column is filled/read only if the class type is `PaymentCard`.
Any scalar properties not in the TPH base class are mapped to nullable columns because those properties are used by only one version of the TPH’s classes. If you have lots of classes in your TPH classes, it’s worth seeing whether you can combine similar typed properties to the same column. In the `Product` TPH classes, for example, you might have a `Product` type "`Sealant`" that needs a double `MaxTemp` and another `Product` type, "`Ballast`", that needs a double `WeightKgs`. You could map both properties to the same column by using this code snippet:

````c#
public class Chapter08DbContext : DbContext
{
    //… other part left out
    Protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Sealant>()
            .Property(b => b.MaxTemp)
            .HasColumnName("DoubleValueCol");

        modelBuilder.Entity<Ballast>()
            .Property(b => b.WeightKgs)
            .HasColumnName("DoubleValueCol");
    }
}
````



**USING THE FLUENT API TO IMPROVE OUR TPH EXAMPLE**

Although the By Convention approach reduces the number of tables in the database, you have two separate `DbSet<T>` properties, and you need to use the right one to find the payment that was used. Also, you don’t have a common `Payment` class that you can use in any other entity classes. But by doing a bit of rearranging and adding some Fluent API configuration, you can make this solution much more useful.
You create a common base class by having an abstract class called `Payment` that the `PaymentCash` and `PaymentCard` inherit from. This approach allows you to use the `Payment` class in another entity class called `SoldIt`.



![](./diagrams/images/08_05_tph_fluent_api.png)



This approach is much more useful because now you can place a Payment abstract class in the `SoldIt` entity class and get the amount and type of payment, whether it’s cash or a card. The `PType` property tells you the type (the `PType` property is of type `PTypes`, which is an enum with values `Cash` or `Card`), and if you need the `Receipt` property in the `PaymentCard`, you can cast the `Payment` class to the type `PaymentCard`.

In addition to creating the entity classes you need to change the application’s DbContext and add some Fluent API configuration to tell EF Core about your TPH classes, as they no longer fit the By Convention approach.



````c#
// Changed application’s DbContext with Fluent API configuration added
public class Chapter08DbContext : DbContext
{
    //… other DbSet<T> properties removed
    public DbSet<Payment> Payments { get; set; } // Defines the property through which you can access all the payments, both PaymentCash and PaymentCard

    public DbSet<SoldIt> SoldThings { get; set; } // List of sold items, with a required link to Payment

    public Chapter08DbContext(DbContextOptions<Chapter08DbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //… other configurations removed
        modelBuilder.Entity<Payment>()
            .HasDiscriminator(b => b.PType) // The HasDiscriminator method identifies the entity as a TPH and then selects the property PType as the discriminator for the different types. In this case, it’s an enum, which you set to be bytes in size.
            .HasValue<PaymentCash>(PTypes.Cash) // Sets the discriminator value for the PaymentCash type
            .HasValue<PaymentCard>(PTypes.Card); // Sets the discriminator value for the PaymentCard type
    }
}
````



**ACCESSING TPH ENTITIES**

The creation of TPH entities is straightforward. You create an instance of the specific type you need.



````c#
var sold = new SoldIt()
{
    WhatSold = "A hat",
    Payment = new PaymentCash {Amount = 12}
};
context.Add(sold);
context.SaveChanges();
````

EF Core saves the correct version of data for that type and sets the discriminator so that it knows the TPH class type of the instance. When you read back the `SoldIt` entity you just saved, with an `Include` to load the `Payment` navigational property, the type of the loaded `Payment` instance will be the correct type (`PaymentCash` or `PaymentCard`), depending on what was used when you wrote it to the database. Also, in this example the `Payment`’s property `PType`, which you set as the discriminator, tells you the type of payment: `Cash` or `Card`.

When you query TPH data, the EF Core `OfType<T>` method allows you to filter the data to find a specific class. The query `context.Payments.OfType<PaymentCard>()` would return only the payments that used a card, for example. You can also filter TPH classes in `Include`s.



#### Table per Type (TPT): Each class has its own table

The EF Core release added the table per type (TPT) option, which allows each entity class inherited from a base class to have its own table. TPT is the opposite of the table per hierarchy (TPH) approach. TPT is a good solution if each class in the inherited hierarchy has lots of different information; TPH is better when each inherited class has a large common part and only a small amount of per-class data.
As an example, you will build a TPT solution for two types of containers: shipping containers used on bulk carrier ships and plastic containers such as bottles, jars, and boxes. Both types of containers have an overall height, length, and depth, but otherwise, they are different. The following listing shows the three entity classes, with the base Container abstract class and then the `ShippingContainer` and `PlasticContainer`.

````c#
// The three classes used in the TPT example
public abstract class Container // The Container class is marked as abstract because it won’t be created.
{
    [Key]
    public int ContainerId { get; set; } // Becomes the primary key for each TPT table

    // Common part of each container is the overall height, width, and depth
    public int HeightMm { get; set; }
    public int WidthMm { get; set; }
    public int DepthMm { get; set; }
}

// The class inherits the Container class.
public class ShippingContainer : Container
{
    // These properties are unique to a shipping container.
    public int ThicknessMm { get; set; }
    public string DoorType { get; set; }
    public int StackingMax { get; set; }
    public bool Refrigerated { get; set; }
}

// The class inherits the Container class.
public class PlasticContainer : Container
{
    // These properties are unique to a plastic container.
    public int CapacityMl { get; set; }
    public Shapes Shape { get; set; }
    public string ColorARGB { get; set; }
}
````

Next, you need to configure your application’s DbContext, which has two parts: (a) adding a `DbSet<Container>` property, which you will use to access all the containers, and (b) setting the other container types, `ShippingContainer` and `PlasticContainer`, to map to their own tables. The following listing shows these two parts.

````c#
// The updates to the application’s DbContext to set up the TPT containers
public class Chapter08DbContext : DbContext
{
    public Chapter08DbContext(DbContextOptions<Chapter08DbContext> options)
        : base(options) { }

    //… other DbSet<T> removed for clarity
    public DbSet<Container> Containers { get; set; } // This single DbSet is used to access all the different containers.

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //… other configrations removed for clarity

        // These Fluent API methods map each container to a different table.
        modelBuilder.Entity<ShippingContainer>()
            .ToTable(nameof(ShippingContainer));
        modelBuilder.Entity<PlasticContainer>()
            .ToTable(nameof(PlasticContainer));
    }
}
````

The result of the update to the application’s DbContext is three tables:

* A `Containers` table, via the DbSet, that contains the common data for each entry
* A `ShippingContainer` table containing the `Container` and `ShippingContainer` properties
* A `PlasticContainer` table containing the `Container` and `PlasticContainer` properties

You add a `ShippingContainer` and `PlasticContainer` in the normal way: by using the `context.Add` method. But the magic comes when you query the `DbSet<Container> Containers` in the application’s DbContext, because it returns all the containers using the correct class type, `ShippingContainer` or `PlasticContainer`, for each entity returned.

You have a few options for loading one type of the TPT classes. Here are three approaches, with the most efficient at the end:

* *Read all query* - `context.Containers.ToList()`
  This option reads in all the TPT types, and each entry in the list will be of the correct type (`ShippingContainer` or `PlasticContainer`) for the type it returns.
  This option is useful only if you want to list a summary of all the containers.
* `OfType` query - `context.Containers.OfType<ShippingContainer>().ToList()`
  This option reads in only the entries that are of the type `ShippingContainer`.
* `Set` query - `context.Set<ShippingContainer>().ToList()`
  This option returns only the `ShippingContainer` type (just like the `OfType` query), but the SQL is slightly more efficient than the `OfType` query.



#### Table splitting: Mapping multiple entity classes to the same table

The next feature, called table splitting, allows you to map multiple entities to the same table. This feature is useful if you have a large amount of data to store for one entity, but your normal queries to this entity need only a few columns. Table splitting is like building a `Select` query into an entity class; the query will be quicker because you’re loading only a subsection of the whole entity’s data. It can also make updates quicker by splitting the table across two or more classes.



![](./diagrams/images/08_06_table_splitting.png)



````c#
// Configuring a table split between BookSummary and BookDetail
public class SplitOwnDbContext : DbContext
{
    //… other code removed

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Defines the two books as having a relationship in the same way that you’d set up a one-to-one relationship
        modelBuilder.Entity<BookSummary>()
            .HasOne(e => e.Details)
            .WithOne()
            .HasForeignKey<BookDetail>(e => e.BookDetailId); // In this case, the HasForeignKey method must reference the primary key in the BookDetail entity.

        // You must map both entity classes to the Books table to trigger the table splitting.
        modelBuilder.Entity<BookSummary>()
            .ToTable("Books");
        modelBuilder.Entity<BookDetail>()
            .ToTable("Books");
    }
}
````

After you’ve configured the two entities as a table split, you can query the `BookSummary` entity on its own and get the summary parts. To get the `BookDetail`s part, you can either query the `BookSummary` entity and load the `Details` relationship property at the same time (say, with an `Include` method) or read only the `BookDetails` part straight from the database.

- You can update an individual entity class in a table split individually; you don’t have to load all the entities involved in a table split to do an update.
- You’ve seen a table split to two entity classes, but you can table-split any number of entity classes.
- If you have concurrency tokens, they must be in all the entity classes mapped to the same table to make sure that the concurrent token values are not out of data when only one of the entity classes mapped to the table is updated.



#### Property bag: Using a dictionary as an entity class

EF Core added a feature called a *property bag* that uses a `Dictionary<string, object>` type to map to the database. A property bag is used to implement the direct many-to-many relationship feature, where the linking table had to be created at configuration time. It is useful only in specific areas, such as creating a property-bag entity in a table whose structure is defined by external data.

A property bag uses two features. The first feature is *shared entity types*, where the same type can be mapped to multiple tables. The second feature uses a C# indexer property in an entity class to access data, such as `public object this[string key] …`.

As an example, you map a property bag to a table whose name and columns are defined by external data rather than by the structure of a class. For this example, the table is defined in a `TableSpec` class, which is assumed to have been read in on startup, maybe from an appsettings.json file. The following listing shows the application’s DbContext with the necessary code to configure and access a table via a property-bag entity.

````c#
// Using a property-bag Dictionary to define a table on startup
public class PropertyBagsDbContext : DbContext
{
    // You pass in a class containing the specification of the table and properties.
    private readonly TableSpec _tableSpec;

    public PropertyBagsDbContext(
        DbContextOptions<PropertyBagsDbContext> options,
        TableSpec tableSpec)
        : base(options)
    {
        _tableSpec = tableSpec;
}

    public DbSet<Dictionary<string, object>> MyTable
        => Set<Dictionary<string, object>>(_tableSpec.Name); // The DbSet called MyTable links to the SharedType entity built in OnModelCreating.

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Defines a SharedType entity type, which allows the same type to be mapped to multiple tables
        modelBuilder.SharedTypeEntity
            <Dictionary<string, object>>(
            	_tableSpec.Name, b => // You give this shared entity type a name so that you can refer to it.
		{
                foreach (var prop in _tableSpec.Properties) // Adds each property in turn from the tableSpec
                {
                    var propConfig = b.IndexerProperty(prop.PropType, prop.Name); // Adds an index property to find the primary key based on its name
                    if (prop.AddRequired) // Sets the property to not being null (needed only on nullable types such as string)
                    {
                        propConfig.IsRequired();
                    }
                }
        }).Model.AddAnnotation("Table", _tableSpec.Name); // Now you map to the table you want to access.
    }
}
````

The data in the `TableSpec` class must be the same every time because EF Core caches the configuration. The property-bag entity’s configuration is fixed for the whole time the application is running. To access the property-bag entity, you use the `MyTable` property shown in the next listing. This listing shows adding a new entry via a dictionary and then reading it back, including accessing the property bag’s properties in a LINQ query.

````c#
// Adding and querying a property bag
var propBag = new Dictionary<string, object> // The property bag is of type Dictionary<string, object>.
{
    ["Title"] = "My book",
    ["Price"] = 123.0
};
context.MyTable.Add(propBag); // For a shared type, such as a property bag, you must provide the DbSet to Add to.
context.SaveChanges(); // The property-bag entry is saved in the normal way.

var readInPropBag = context.MyTable // To read back, you use the DbSet mapped to the property-bag entity.
    .Single(x => (int)x["Id"] == 1); // To refer to a property/column, you need to use an indexer. You may need to cast the object to the right type.

var title = readInPropBag["Title"]; // You access the result by using normal dictionary access methods.
````

This is a specific example in which a property bag is a good solution, but you can configure a property bag manually.

Here is some more information on the property bag:

* A property bag’s property names follow By Convention naming. The primary key is "`Id`", for example. But you can override this setting with Fluent API commands as usual.
* You can have multiple property bags. The `SharedTypeEntity` Fluent API method allows you to map the same type to different tables.
* A property bag can have relationships to other classes or property bags. You use the `HasOne`/`HasMany` Fluent API methods, but you can’t define navigational properties in a property bag.
* You don’t have to set every property in the dictionary when you add a property-bag entity. Any properties/columns not set will be set to the type’s default value.



### Summary

* If you follow the By Convention naming rules for foreign keys, EF Core can find and configure most normal relationships.
* Two Data Annotations provide a solution to a couple of specific issues related to foreign keys with names that don’t fit the By Convention naming rules.
* The Fluent API is the most comprehensive way to configure relationships. Some features, such as setting the action on deletion of the dependent entity, are available only via the Fluent API.
* You can automate some of the configuration of your entity classes by adding code that is run in the DbContext’s `OnModelCreating` method.
* EF Core enables you to control updates to navigational properties, including stopping, adding, or removing entries in collection navigational properties.
* EF Core provides many ways to map entity classes to a database table. The main ones are owned types, table per hierarchy, table per type, table splitting, and property bags.





## 9. Handling database migrations

* Different ways to create commands to update a database’s structure
* Three starting points from which you create database structure changes
* How to detect and fix database structure changes that would lose data
* How the characteristics of your application affect the way you apply a database change



The structure of the database is called the database schema; it consists of the tables, columns, constraints, and so on that make up a database. Creating and updating a database schema can seem to be simple but The problem is that EF Core’s Migrate method hides a whole series of database migration issues that aren’t immediately obvious.

Renaming a property in an entity class, for example, by default causes that property’s database column to be deleted, along with any data it had!



### Understanding the complexities of changing your application’s database

#### A view of what databases need updating

![](./diagrams/images/09_01_db_usage_types.png)



### Part 1: Introducing the three approaches to creating a migration

![](./diagrams/images/09_02_migrations.png)



### Creating a migration by using EF Core’s add migration command

![](./diagrams/images/09_03_migration_command.png)



A summary of the good, the bad, and the limitations of a standard migration created by the add migration command:

* Good parts
  * Builds a correct migration automatically
  * Handles seeding of the database
  * Doesn’t require knowledge of SQL
  * Includes a remove migration feature
* Bad parts
  * Only works if your code is the source of truth
* Limitations
  * Standard EF Core migrations cannot handle breaking changes.
  * Standard EF Core migrations are database-specific
* Tips
  * Watch out for error messages when you run the add migration command. If EF Core detects a change that could lose data, it outputs an error message but still creates the migration files. You must alter the migration script; otherwise, you will lose data.
* Verdict
  * This approach is an easy way to handle migrations, and it works well in many cases. Consider this approach first if your application code is driving the database design.



#### Requirements before running any EF Core migration command

There are two versions of the EF Core migration tools: the `dotnet-ef` command-line interface (CLI) tools and Visual Studio’s Package Manager Console (PMC) version.

The following command will install the `dotnet-ef` tools globally so that you can use them in any directory:

````powershell
dotnet tool install --global dotnet-ef
````



To use Visual Studio’s PMC feature, you must include the NuGet package `Microsoft.EntityFrameworkCore.Tools` in your main application, and the correct EF Core database provider NuGet package, such as `Microsoft.EntityFrameworkCore.SqlServer`, in the project that holds the application’s DbContext you want to migrate.

These tools must be able to create an instance of the DbContext you want to migrate. If your startup project is an ASP.NET Core web host or .NET Core generic host, the tools can use it to get an instance of a DbContext set up in the startup class.

If you aren’t using ASP.NET Core, you can add a class that implements the `IDesignTimeDbContextFactory<TContext>` interface. This class must be in the same project as the DbContext you want to migrate.

````c#
// A class that provides an instance of the DbContext to the migration tools
public class DesignTimeContextFactory // EF Core tools use this class to obtain an instance of the DbContext.
    : IDesignTimeDbContextFactory<EfCoreContext> // This interface defines a way that the EF Core tools find and create this class.
{
    private const string connectionString = "Server=(localdb)\\mssqllocaldb;Database=...";

    public EfCoreContext CreateDbContext(string[] args) // The interface requires this method, which returns a valid instance of the DbContext.
    {
        var optionsBuilder = new DbContextOptionsBuilder<EfCoreContext>();
        optionsBuilder.UseSqlServer(connectionString);

        return new EfCoreContext(optionsBuilder.Options); // Returns the DbContext for the EF Core tools to use
    }
}
````



#### Seeding your database via an EF Core migration

You add seed data via Fluent API configuration, using the HasData method.

````c#
// An example of setting up seed data via the HasData Fluent API method
protected override void OnModelCreating(ModelBuilder modelBuilder) // Seeding is configured via the Fluent API.
{
    // Adds two default projects. Note that you must provide the primary key.
    modelBuilder.Entity<Project>().HasData(
        new { ProjectId = 1, ProjectName = "Project1"},
        new { ProjectId = 2, ProjectName = "Project2"});

    // Each Project and a ProjectManager. Note that you set the foreign key of the project they are on.
    modelBuilder.Entity<User>().HasData(
        new { UserId = 1, Name = "Jill", ProjectId = 1 },
        new { UserId = 2, Name = "Jack", ProjectId = 2 });

    modelBuilder.Entity<User>()
        .OwnsOne(x => x.Address).HasData( // // The User class has an owned type that holds the User’s address.
        	new {UserId = 1, Street = "Street1", City = "city1"},
        	new {UserId = 2, Street = "Street2", City = "city2"}); // Provide the user’s addresses. Note that you use the UserId to define which user you are adding data to.
}
````



#### Handling EF Core migrations with multiple developers

You will know if you have a migration merge conflict because your source control system will show a conflict in the migration snapshot file, which has the name `<DbContextClassName>ModelSnapShot`. If this conflict happens, here’s the recommended way to fix it:

1. Abort the source control merge that contained a migration change that conflicted with your migration.

2. Remove the migration you created by using either of the following commands
   (Note: Keep the entity classes and configuration changes; you will need them later):

   a. CLI - `dotnet ef migrations remove`
   b. PMC - `Remove-Migration`

3. Merge the incoming migration you abandoned in step 1. A merge conflict should no longer appear in the migration snapshot file.

4. Use the `add migration` command to re-create your migration.

That migration conflict resolution process works in most cases, but it can get complex. Recommendation for projects in which migration conflicts can happen are

* Merge the main/production branch into your local version before you create a migration.
* Have only one migration in a source control merge into your main/production branch, because undoing two migrations is hard work.
* Tell your development team members if you think that your migration might affect their work.



#### Using a custom migration table to allow multiple DbContexts to one database

EF Core creates a table called `__EFMigrationsHistory`, if you apply an EF Core migration to a database. You can change the name via an option method called `MigrationsHistoryTable`.

There aren’t many reasons for changing the migration history table, but sharing a database across multiple EF Core DbContexts is one of them.

* Saving money by combining databases - You are building an application with individual user accounts that needs an accounts database. Your application’s DbContext also needs a database. By using a custom migration table on your application’s DbContext would allow both contexts to use the same database.
* Using a separate DbContext for each business group.

````c#
// Changing the name of the migration history table for your DbContext
services.AddDbContext<EfCoreContext>( // Registers your application’s DbContext as a service in ASP.NET Core
    options => options.UseSqlServer(connection,
        dbOptions => // The second parameter allows you to configure at the database provider level.
	        dbOptions.MigrationsHistoryTable("NewHistoryName"))); // The MigrationsHistoryTable method allows you to change the migration table name and optionally the table’s schema.
````

````c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Book>()
        .ToTable("Books",
                 t => t.ExcludeFromMigrations());
}
````



### Editing an EF Core migration to handle complex situations

EF Core can’t handle every possible database migration, such as a data-loss breaking change. Let’s look at the types of migrations that a standard migration can’t handle without help:

* *Data-loss breaking changes*, such as moving columns from one table to a new table
* *Adding SQL features that EF Core doesn’t create*, such as adding user-defined functions, SQL stored procedures, views, and so on
* *Altering a migration to work for multiple database types*, such as handling both SQL Server and PostgreSQL

You can fix these problems by editing the standard migration class created via the `add migration` command. To do this, you need to edit the migration class whose filename ends with the migration name and has a type of .cs, such as …_InitialMigration.cs.

A summary of the good, the bad, and the limitations of a migration created by the add migration command edited by you to handle situations that the standard migration can’t handle on its own:

* Good parts
  * You start with most of the migration build via the `add migration` command.
  * You can customize the migration.
  * You can add SQL extra features, such as stored procedures.
* Bad parts
  * You need to know more about the database structure.
  * Some edits require SQL skills.
* Limitations
  * Your edits aren’t checked by EF Core, so you could get a mismatch between the updated database and your entity classes and application’s DbContext.
* Tips
  * Same as for standard migrations
* Verdict
  * This approach is great for small alterations, but making big changes can be hard work, as you are often mixing C# commands with SQL. If you expect to be editing lots of your migrations to add SQL features, you should consider an SQL script approach as an alternative.



#### Adding and removing MigrationBuilder methods inside the migration class

If you change the name of a property in an entity class, which causes a data-loss breaking change. This problem can be fixed by removing two commands and replacing them with `MigrationBuilder`’s `RenameColumn` method inside the migration class.

The standard migration sees this operation as being the removal of the `CustomerId` property and the addition of a new property called `UserId`, which would cause any existing values in the `CustomerId` column to be lost. To fix this problem, make the following changes:

* Remove the `AddColumn` command that adds the new `UserId` column.
* Remove the `DropColumn` command that removes the existing `CustomerId` column.
* Add a `RenameColumn` command to rename the `CustomerId` column to `UserId`.

````c#
// The updated migration class with old commands replaced
public partial class MyExample : Migration // Migration class created by the add migration command that has been edited
{
    protected override void Up(MigrationBuilder migrationBuilder) // There are two methods in a migration class. Up applied the migration, and another method called Down removed this migration.
    {
        // The command to add the new UserId column should not run, so you comment it out.
        //migrationBuilder.AddColumn<Guid>(
        //    name: "UserId",
        //    table: "Orders",
        //    type: "uniqueidentifier",
        //    nullable: false,
        //    defaultValue:
        //    new Guid("00000000-0000-0000-0000-000000000000"));

        // The command to remove the existing CustomerId column should not run, so you comment it out.
        //migrationBuilder.DropColumn(
        //    name: "CustomerId",
        //    table: "Orders");

        // The correct approach is to rename the CustomerId column to UserId.
        migrationBuilder.RenameColumn(
            name: "CustomerId",
            table: "Orders",
            newName: "UserId");
        
        //… rest of the migration code left out
    }
}
````



#### Adding SQL commands to a migration

There can be two main reasons for adding SQL commands to a migration: to handle a data-loss breaking change and to add or alter parts of the SQL database that EF Core doesn’t control, such as adding views or SQL stored procedures.



![](./diagrams/images/09_04_sql_to_migration.png)



First, you change your `User` entity class to remove the address and link to the new `Address` entity class to the DbContext. Then you create a new migration by using the `add migration` command, which will warn you that it may result in the loss of data. At this point, you are ready to edit the migration.

The second step is adding a series of SQL commands, using the `MigrationBuilder` method `Sql`, such as `migrationBuilder.Sql("ALTER TABLE…")`. The following listing shows you the SQL commands without the `migrationBuilder.Sql` so that they are easier to see.

````sql
// The SQL Server commands to copy over the addresses to a new table
ALTER TABLE [Addresses]
	ADD [UserId] [int] NOT NULL -- Adds a temporary column to allow the correct foreign key to be set in the Users table

INSERT INTO [Addresses] ([UserId],[Street],[City])
	SELECT [UserId],[Street],[City] FROM [Users] -- Copies over all the address data, with the User’s primary key, to the addresses table

UPDATE [Users] SET [AddressId] = ( -- Sets the foreign key in the Users table back to the Addresses table
	SELECT [AddressId] -- Uses the temporary UserId column to make sure that the right foreign keys are set up
		FROM [Addresses]
		WHERE [Addresses].[UserId] = [Users].[Userid])

ALTER TABLE [Addresses]
	DROP COLUMN [UserId] -- Removes the temporary UserId column in the Addresses table, as it’s not needed anymore
````

You add these SQL commands to the migration by using the `migrationBuilder.Sql` method for each SQL command, placing them after the Addresses table is created but before the foreign key is set up. Also, the `MigrationBuilder` methods that drop (remove) the address properties from the Users table must be moved to after the SQL code has run; otherwise, the data will have gone before your SQL can copy that data over.



#### Adding your own custom migration commands

If you often add certain types of SQL commands to a migration, you can build some templating code to make your edits easier to write:

* Create extension methods that take the `MigrationBuilder` class in and build commands with `MigrationBuilder`’s Sql method. These extension methods tend to be database-specific.
* A more complex but more versatile approach is to extend the `MigrationBuilder` class to add your own commands. This approach allows you to access methods to build commands that work for many database providers.



````c#
// Extension method to add/alter an SQL view in an EF Core migration
public static class AddViewExtensions // An extension method must be in a static class.
{
    public static void AddViewViaSql<TView>( // The method needs the class that is mapped to the view so that it can get its properties.
        this MigrationBuilder migrationBuilder, // The MigrationBuilder provides access to the migration methods-in this case, the Sql method.
        string viewName,
        string tableName, // The method needs the name to use for the view and the name of the table it is selecting from.
        string whereSql) // Views have a Where clause that filters the results returned.
        where TView : class // Ensures that the TView type is a class
    {
            // This method throws an exception if the database isn’t Server because it uses an SQL Server view format.
            if (!migrationBuilder.IsSqlServer())
            {
                throw new NotImplementedException("warning…");
            }

            // Gets the names of the properties in the class mapped to the view and uses them as column names
            var selectNamesString = string.Join(", ", typeof(TView).GetProperties().Select(x => x.Name));

            // Creates the SQL command to create/update a view
            var viewSql =
                $"CREATE OR ALTER VIEW {viewName} AS " +
                $"SELECT {selectNamesString} FROM {tableName} " +
                $"WHERE {whereSql}";

            migrationBuilder.Sql(viewSql); // Uses MigrationBuilder’s method to apply the created SQL to the database
    }
}
````

You would use this technique in a migration by adding it to the `Up` method (and a `DROP VIEW` command in the `Down` method to remove it).

````c#
migrationBuilder.AddViewViaSql<MyView>(
    "EntityFilterView", "Entities",
    "MyDateTime >= '2020-1-1'");
````

The resulting SQL looks like this snippet:

````sql
CREATE OR ALTER VIEW EntityFilterView AS
SELECT MyString, MyDateTime
FROM Entities
WHERE MyDateTime >= '2020-1-1'
````



#### Altering a migration to work for multiple database types

If you need to support migrations for two or more types of databases, the recommended way is to build separate migrations for each database provider.



````c#
// Two DbContexts that have the same entity classes and configuration
public class MySqlServerDbContext : DbContext // Inherits the normal DbContext class
{
    public DbSet<Book> Books { get; set; }
    // … other DbSets left out

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //… your Fluent API code goes here
    }
}

public class MySqliteDbContext : MySqlServerDbContext // The MySqliteDbContext inherits the Sql Server DbContext class instead of the normal DbContext.
{
    // The MySqliteDbContext inherits the DbSet properties and Fluent APIs from the Sql Server DbContext.
}
````

The next step is creating a way for the migration tools to access each DbContext with the database provider defined. The cleanest way is to create an `IDesignTimeDbContextFactory<TContext>` class. Alternatively, you can override the `OnConfiguring` method in each DbContext to define the database provider.

````c#
services.AddDbContext<MySqlServerDbContext>(
    options => options.UseSqlServer(connection,
		x => x.MigrationsAssembly("Database.SqlServer")));
````



### Using SQL scripts to build migrations

Change scripts contain SQL commands that update the schema of your database. This approach to handling database schema updates is more traditional and gives you much better control of the database features and the schema update. Tools can generate these migration scripts for you by comparing databases.

As with the migrations that EF Core can create, your aim is to create a migration that will alter the schema of your database to match the EF Core’s internal model of the database.

* Using SQL database comparison tools to produce migration from the current database schema to the desired database schema
* Handcoding a change script to migrate the database

Although option 1 should produce an exact match to EF Core’s internal model of the database, option 2 relies on the developer to write the correct SQL to match what EF Core needs. If the developer makes a mistake (which I can testify is easy to do), your application may fail with an exception; worse, it may silently lose data.



#### Using SQL database comparison tools to produce migration

A summary of the good, the bad, and the limitations of using an SQL database comparison tool to build SQL change scripts to migrate a database:

* Good parts
  * Tools build the correct SQL migration script for you.
* Bad parts
  * You need some understanding of databases and SQL.
  * SQL comparison tools often output every setting under the sun to make sure that they get everything right, which makes the SQL code output hard to understand.
  * Not all SQL comparison tools produce a migration remove script.
* Limitations
  * Tools do not handle breaking changes, so they need human input.
* Tips
  * You can use this approach only for complex/large migrations, and I strip out any extra settings to make the code easier to work with.
* Verdict
  * This approach is useful and especially good for people who aren’t comfortable with the SQL language. It’s also useful for people who have written their own SQL migration code and want to check that their code is correct.



![](./diagrams/images/09_05_sql_database_comparison.png)



#### Handcoding SQL change scripts to migrate the database

Another approach is to create the SQL commands needed for a migration yourself. This option is attractive to developers who want to define the database in ways that EF Core can’t. You can use this approach to set more-rigorous CHECK constraints on columns, add stored procedures or user-defined functions, and so on via SQL scripts.

The job of creating an SQL change script is made easier by the migration `scriptdbcontext` command, which outputs the SQL commands that EF Core would use to create a new database (equivalent to calling the `context.Database.EnsureCreated` method).

A summary of the good, the bad, and the limitations of handcoding the SQL change scripts to migrate a database:

* Good parts
  * You have total control of the database structure, including parts that EF Core won’t add, such as user-defined functions and column constraints.
* Bad parts
  * You must understand SQL commands such as `CREATE TABLE`.
  * You must work out what the changes are yourself.
  * There’s no automatic migration remove script.
  * This approach is not guaranteed to produce a correct migration.
* Limitations
  * None
* Tips
  * You can use the `Script-DbContext` migration command to get the actual SQL that EF Core would output and then look for the differences in the SQL from the previous database schema, which makes writing the SQL migrations much easier.
* Verdict
  * This approach is for someone who knows SQL and wants complete control of the database. It certainly makes you think about the best settings for your database, which can improve performance.



````sql
// Part of SQL generated by EnsureCreated when creating a database
-- other tables left out
CREATE TABLE [Review] ( -- Creates the Review table, with all its columns and constraints
    [ReviewId] int NOT NULL IDENTITY, -- Says that the database will provide a unique value when a new row is created
    [VoterName] nvarchar(100) NULL,
    [NumStars] int NOT NULL,
    [Comment] nvarchar(max) NULL,
    [BookId] int NOT NULL,
    CONSTRAINT [PK_Review] PRIMARY KEY ([ReviewId]), -- Says that the ReviewId column is the primary key
    -- Says that the BookId column is a foreign key link to the Books table and that if the Books row is deleted, the linked Review row will be deleted too
    CONSTRAINT [FK_Review_Books_BookId]
    	FOREIGN KEY ([BookId])
    	REFERENCES [Books] ([BookId]) ON DELETE CASCADE

-- other SQL indexes left out
CREATE INDEX [IX_Review_BookId] ON [Review] ([BookId]); -- Says that there should be an index of the BookId foreign key to improve performance
````



As with EF Core’s migrations, you create a series of SQL change scripts that need to be applied to your database in order. To aid this process, you should name your scripts with something that defines the order, such as a number or a sortable date.

Here are example SQL script names that I used for a client project:

````
Script001 - Create DatabaseRegions.sql
Script002 - Create Tenant table.sql
Script003 - TenantAddress table.sql
Script004 - AccountingCalenders table.sql
````



### Using EF Core’s reverse-engineering tool

In some cases, you already have a database that you want to access via EF Core code. For this purpose, you need to apply the opposite of migrations and allow EF Core to produce your entity classes and application’s DbContext by using your existing database as the template. This process is known as reverse engineering a database. This approach says that the database is the source of truth. You use EF Core’s reverse-engineering tool, also known as scaffolding, to re-create the entity classes and the application’s DbContext with all the required configurations.

A summary of the good, the bad, and the limitations of reverse-engineering a database as a way to access an existing database or continually update your entity classes and application DbContext to match a changed database:

* Good parts
  * The tool builds the EF Core code/classes from an existing database.
  * The tool allows you to make the database the source of truth, and your EF Core code and classes are created and updated as the database schema changes.
* Bad parts
  * Your entity classes can’t be edited easily, such as to change the way that the collections navigational properties are implemented.
  * The tool always adds navigational links at both ends of the relationship.
* Limitations
  * None
* Tips
  * When you are going to repeatedly reverse engineer a database, recommend using the Visual Studio EF Core Power Tools extension, as it remembers the setting from the last time you used the reverse engineering feature.
* Verdict
  * If you have an existing database that you need to access via EF Core, reverse engineering is going to save you a lot of time.



![](./diagrams/images/09_06_reverse_engineering_tool.png)



Let’s look at how to run the reverse-engineering tool. You have two options:

* Run EF Core’s reverse-engineering tool via a command line.
* Use the EF Core Power Tools Visual Studio extension.



#### Running EF Core’s reverse-engineering command

You can reverse engineer a database from a command line (CLI tools) or Visual Studio’s PMC window. CLI and PMC have different names and parameters. The following list shows the `scaffold` command to reverse engineer the BookApp database. Commands are run in the directory of the BookApp ASP.NET Core project and that the database connection string is in the appsettings.json file in that project:

* *CLI* - `dotnet ef dbcontext scaffold name=DefaultConnection Microsoft.EntityFrameworkCore.SqlServer`
* *PMC* - `Scaffold-DbContext -Connection name=DefaultConnection -Provider Microsoft.EntityFrameworkCore.SqlServe`r



#### Installing and running EF Core Power Tools reverse-engineering command

First, you need to install the EF Core Power Tools Visual Studio extension. After you have installed the extension, right-click a project in Visual Studio’s Solution Explorer. You should see a command called EF Core Power Tools, with a Reverse Engineering subcommand.



#### Updating your entity classes and DbContext when the database changes

One way to handle database changes is to migrate your database and then run the reverse-engineering tool to re-create your entity classes and application’s DbContext. That way, you know that the database schema and EF Core’s model are in step. Using EF Core’s reverse-engineering tool directly works, but you must remember all the settings for each run.



### Part 2: Applying your migrations to a database

Here is a list of the techniques you will be evaluating:

* Calling EF Core’s `Database.Migrate` method from your main application
* Executing EF Core’s Database.Migrate method from a standalone application designed only to migrate the database
* Applying an EF Core migration via an SQL change script and applying it to a database
* Applying SQL change scripts by using a migration tool



Affects how you migrate your database is the environment you are working in - specifically, the characteristics of the application that accesses the database being migrated, with special focus on your production system. The first characteristic is whether you are running multiple instances of the application, such as multiple instances of an ASP.NET Core. This characteristic is important because all the ways of applying a migration to a database rely on only one application’s trying to change the database’s schema. Having multiple instances running, therefore, rules out some of the simpler migration update techniques, such as running a migration when the application starts because all the multiple instances will try to run at the same time.

The second characteristic is whether the migration is applied while the current application is running. This situation happens if you have applications that need to be up all the time, such as email systems and sites that people want to access at any time.

Every migration applied to a database of a continuous-service application must not be an application-breaking change; the migrated database must still work with the currently running application code. If you add a non-nullable column with no default SQL value, for example, when the old application creates a new row, the database will reject it, as the old application didn’t provide a value to fill in the new column. This application-breaking change must be split into a series of nonbreaking changes.



#### Calling EF Core’s Database.Migrate method from your main application

You add some code that calls `context.Database.Migrate` before the main application starts. This approach is by far the easiest way to apply a migration, but it has a big limitation: you should not run multiple instances of the `Migrate` method at the same time. If your application has multiple instances running at the same time - the many app characteristic - you cannot use this approach.

A summary of the good, the bad, and the limitations of calling EF Core’s `Database.Migrate` method from your main application:

* Good parts
  * This approach is relatively easy to implement.
  * It ensures that the database is up to date before your application runs.
* Bad parts
  * You must not run two or more `Migrate` methods in parallel.
  * There is a small period when your application isn’t responding.
  * If the migration has an error, your application won’t be available.
  * It can be hard to diagnose startup errors.
* Limitations
  * This approach does not work if multiple instances of the application are running.
* Tips
  * For ASP.NET Core applications, recommend applying the migration in your CI/CD pipeline, even if you expect to run only one instance of the web app, because your app won’t be deployed if the migration fails, and you will be ready to scale out if you need to.
* Verdict
  * If you can guarantee that only one instance of your application is starting up at any one time, this approach is a simple solution to migrating your database. Unfortunately, that situation isn’t typical for websites and local applications.



**FINDING WHAT MIGRATIONS THE DATABASE.MIGRATE METHOD WILL APPLY TO THE DATABASE**

When you use the `context.Database.Migrate` method to migrate a database, you may want to run some C# code if a certain migration is applied. You can use this technique to fill in a new property/column added in a certain migration. You can find out what migrations are going to be applied to the database by calling the `GetPendingMigrations` method before you call the `Migrate` method and the method called `GetAppliedMigrations` to get the migrations that have been applied to the database. Both methods return a set of strings of the filenames that hold the migration.



````c#
// Detecting what migrations have been applied to the database
context.Database.Migrate(); // You call the migration method to apply any missing migrations to the database.
if (context.CheckIfMigrationWasApplied(nameof(InitialMigration))) // You use the extension method to find whether the InitialMigration was added to the database.
{
    // Code that must run after the Initial-Migration has run
    //... run your C# code for this specific migration
}

//Extension method to detect a specific migration was applied
public static bool CheckIfMigrationWasApplied( // Simple extension method to detect a specific migration from the class name
this DbContext context, string className)
{
    return context.Database.GetAppliedMigrations() // The GetAppliedMigrations method returns a filename for each migration applied to the database.
        .Any(x => x.EndsWith(className)); // All the filenames end with the class name, so we return true if any filename ends with className.
}
````



#### Executing EF Core’s Database.Migrate method from a standalone application

Instead of running the migration as part of your startup code, you can create a standalone application to apply a migration to your databases. You could add a console application project to your solution, for example, using your application’s DbContext to call the `context.Database.Migrate` method when it’s run, possibly taking the database connection string as a parameter. Another option is calling the CLI command `dotnet ef database update`, which in EF Core can take a connection string. This approach can be applied when the application is running or when it is stopped. Assumes that the application is stopped.

A summary of the good, the bad, and the limitations of executing EF Core’s `Database.Migrate` method from a standalone application:

* Good parts
  * If the migration fails, you get good feedback from the migration.
  * This approach overcomes the problem that the `Migrate` method isn’t thread safe.
* Bad parts
  * Your application is down while the migration is applied.
* Limitations
  * None
* Verdict
  * This option is a good one if you have multiple instances of your application. In your CI/CD pipeline, for example, you could stop the current applications, run one of EF Core’s `Migrate` commands (such as `dotnet ef database update`), and then upload and start your new application.



#### Applying an EF Core’s migration via an SQL change script

In some cases, you want to use EF Core’s migrations, but you want to check the migrations or apply them via SQL change scripts. You can get EF Core to create SQL change scripts, but watch out for a few things if you take this approach. The default SQL change script produced by EF Core, for example, contains only the script to update the database, with no check of whether a migration has already been applied. The reason is that developers normally apply SQL change scripts via some sort of deployment system that handles the job of working out what migrations need to be applied to the database being migrated.

A summary of the good, the bad, and the limitations of applying an EF Core’s migration via an SQL change scripts:

* Good parts
  * EF Core will build your migrations for you and then give you the migration as SQL.
  * The SQL scripts generated by EF Core update the migration history table.
* Bad parts
  * You need an application to apply the migrations to your databases.
* Limitations
  * None
* Tips
  * Be aware that the individual migrations don’t check whether the migration has been applied to the database. This approach assumes that some other application is keeping track of the migrations.
  * If you need a migration that checks whether it has already been applied to the database, you need to add the idempotent parameter to the command.
* Verdict
  * If you want to check/sign off a migration or use a more comprehensive app/database deployment system, such as Octopus Deploy or a RedGate product, this approach is the way to go.

The basic command to turn the latest migration into an SQL script is

* *CLI* - `dotnet ef migrations script`
* *PMC* - `Script-Migration`

These two commands output the SQL for the last migration with no check of whether that migration has been applied to the database. But when you add the `idempotent` parameter to these commands, the SQL code that they produce contains checks of the migration history table and applies only migrations that haven’t been applied to the database.

The SQL script created by the `Script-Migration` command has applied a migration within an SQL transaction. The whole of the migration will be applied to the database unless there is an error, in which case none of the migration will be applied.



#### Applying SQL change scripts by using a migration tool

If you have gone for the SQL-change-scripts approach, it’s likely that you already know how you will apply these change scripts to the database. You will need to use a migration tool such as DbUp (open source) or free or commercial tools such as RedGate’s flyaway. Typically, these migration tools have their own version of EF Core migration history table. (DbUp calls this table SchemaVersions.)

How you implement the migration depends on the migration tool you use. DbUp, for example, is a NuGet package, so you can use it the same way as EF Core’s `Migrate` method: call it on startup or as a separate application in your CI/CD pipeline, and so on. Other migration tools may not be callable from NET Core but use some form of command line or deployment pipeline integration.

A summary of the good, the bad, and the limitations of applying SQL change scripts by using a migration tool:

* Good parts
  * The tool works in all situations.
  * It works well with deployment systems.
* Bad parts
  * You must manage the scripts yourself and make sure that their names define the order in which they will be applied.
* Limitations
  * None
* Tips
  * When used this approach, you can see whether a migrated test database matched EF Core’s internal model.
* Verdict
  * SQL change scripts and DbUp works well. With some of the improvements in EF Core, might be tempted back to using EF Core migrations.



### Migrating a database while the application is running

Let’s compare the two types of applications: one that can be stopped for a migration or software update and one that must continue to provide a service while it’s being updated.

We will discuss how to migrate a database on a continuous-service application. There are two situations:

* The migration doesn’t contain any changes that would cause the currently running application (referred to as the original app) to fail.
* The migration contains changes that would cause the original app to fail (application-breaking changes).



![](./diagrams/images/09_07_migration_and_running_applications.png)



#### Handling a migration that doesn’t contain an application-breaking change

* If you’re adding a new scalar property to an existing table, the old application won’t set it. That’s OK, because SQL will give it a default value. But what default do you want the property to have? You can control that setting by setting an SQL default value for the column or make it nullable. That way, the existing application running in production won’t fail if you create a new row.
* If you’re adding a new foreign-key column to an existing table, you need to make that foreign key nullable and have the correct cascade-delete settings. That approach allows the old application to add a new row to that table without the foreign-key constraint’s reporting an error.

Some of these issues, such as making a column nullable when it would normally be non-nullable, might require a second migration to change the nullability of the database columns when your new application is in place. This situation leads to the multiplestep migration approach for dealing with application breaking changes.



#### Handling application-breaking changes when you can’t stop the app

Applying an application breaking migration to a continuous-service application is one of the most complicated migrations there is. As an example, you are going to consider to handle a database migration that moves columns from an Users table to a new Addresses table. In the original migration, this "move columns" issue was done by one migration, but it worked only because the original application was stopped, and after the migration finished, the new application ran.

For a continuous-service application, the move-columns task must be broken into a series of stages so that each migration doesn’t break the two applications that are running at the same time. As a result, we end up with three migrations:

* *ADD* - The first migration is applied while App1 is currently running and adds new database features that the new interim application (App2) needs to run.

* *COPY* - The second migration is applied after App1 has stopped and before App3, the target application, has started. This migration copies the data in its final format.

* *SUBTRACT* - The last migration is a clean-up, which runs only when App2 has stopped and App3 has taken over. At this point, it can remove the old tables and columns that are now redundant.

The `ADD` and then `SUBTRACT` migrations, with maybe a `COPY` in the middle, represent the common approach to applying breaking changes to continuous-service applications. At no time should the database be incorrect for two applications that are running.



![](./diagrams/images/09_08_migration_ci_cd.png)



Here is a detailed breakdown of these stages:

* *Stage 1* - This stage is the starting point, with the original application, App1, running.
* *Stage 2* - This stage is the most complex one. It does the following:
  * Runs a migration that creates a new Addresses table and links it to the current user.
  * Adds an SQL View that returns a User with their address from either the old Users’ Street/City columns or from the new Address table.
  * The interim application, App2, uses the SQL View to read the User, but if it needs to add or update a User’s address, it will use the new Address table.
* *Stage 3* - App1 is stopped, so there is no possibility that new addresses will be added to the Users table. At this point, the second migration runs and copies any address data in the Users table to the new Addresses table.
* *Stage 4* - At this point, the target application, App3, can be run; it gets a User’s address only from the new Addresses table.
* *Stage 5* - App2 is stopped, so nothing is accessing the address part of the old User’s table. This stage is when the last migration runs, cleaning up the database by removing the Street and City columns from the Users table, deleting the SQL View needed by App2, and fixing the User/Address relationship as required.



### Summary

* The easiest way to create a migration is via EF Core’s migration feature, but if you have a migration that removes or moves columns, you need to hand-edit before the migration will work.
* You can build SQL change scripts by using a database comparison tool or by hand. This approach gives you complete control of the database. But you need to check that your SQL change scripts create a database that matches EF Core’s internal model of the database.
* If you have an existing database, you can use EF Core’s scaffold command or the more visual EF Core Power Tools Visual Studio extension to create the entity classes and the application’s DbContext with all its configurations.
* Updating a production database is a serious undertaking, especially if data could be lost in the process. How you apply migration to a production system depends on the type of migration and certain characteristics of your application.
* There are several ways to apply a migration to a database. The simplest approach has significant limitations, but the complex approaches can handle all migration requirements.
* Applying migration to a database while the application is running requires extra work, especially if the migration changes the database schema to the point that the current application will fail.





## 10. Configuring advanced features and handling concurrency conflicts

* Using an SQL user-defined function in EF Core queries
* Configuring columns to have default values or computed values
* Configuring SQL column properties on databases not created by EF Core
* Handling concurrency conflicts



### DbFunction: Using user-defined functions (UDFs) with EF Core

SQL has a feature called UDFs that allows you to write SQL code that will be run in the database server. UDFs are useful because you can move a calculation from your software into the database, which can be more efficient because it can access the database directly. UDFs can return a single result, which is referred to as *scalar-valued function*, and one that can return multiple data in a result, known as a *table-valued function*. UDFs differ from SQL *stored procedures* (StoredProc) in that UDFs can only query a database, whereas a StoredProc can change the database.

Configuration:

1. Define a method that has the correct name, input parameters, and output type that matches the definition of your UDF. This method acts as a reference to your UDF.
2. Declare the method in the application’s DbContext or (optionally) in a separate class if it’s a scalar UDF.
3. Add the EF Core configuration commands to map your static UDF reference method to a call to your UDF code in the database.

Database setup:

4. Manually add your UDF code to the database by using some form of SQL command.

Use:

5. Now you can use the static UDF reference in a query. EF Core will convert that method to a call to your UDF code in the database.



#### Configuring a scalar-valued UDF

The configuration for a scalar-valued UDF consists of defining a method to represent your UDF and then registering that method with EF Core at configuration time. You’re going to produce a UDF called `AverageVotes` that works out the average review votes for a book. `AverageVotes` takes in the primary key of the book you want to calculate for and returns a nullable double value - `null` if no reviews exist or the average value of the review votes if there are reviews.

You can define the UDF representation as a static or nonstatic method. Nonstatic definitions need to be defined in your application’s DBContext; the static version can be placed in a separate class.



![](./diagrams/images/10_01_scalar_valued_udf.png)



The UDF representation method is used to define the signature of the UDF in the database: it will never be called as a NET method.

You can register your static UDF representation method with EF Core by using either of the following:

* `DbFunction` attribute
* Fluent API

You can use the `DbFunction` attribute if you place the method representing the UDF inside your application’s DbContext.

````c#
// Using a DbFunction attribute with a static method inside DbContext
public class MyEfCoreContext : DbContext
{
    public DbSet<Book> Books { get; set; }
    //… other code removed for clarity

    public Chapter08EfCoreContext(
        DbContextOptions<MyEfCoreContext> options)
        : base(options) {}

    [DbFunction] // The DbFunction attribute defines the method as being a representation of your UDF.
    public static double? AverageVotes(int id) // The return value, the method name, and the number, type, and order of the method parameters must match your UDF code.
    {
        return null; // The method is never called, but you need the right type for the code to compile.
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // If you use the DbFunction attribute, you don’t need any Fluent API to register the static method.
        //… no Fluent API needed
    }
}
````



The other approach is to use the Fluent API to register the method as a UDF representation. The advantage of this approach is that you can place the method in any class, which makes sense if you have a lot of UDFs.



````c#
// Registering your static method representing your UDF using Fluent API
protected override void OnModelCreating(ModelBuilder modelBuilder) // Fluent API is placed inside the OnModelCreating method inside your application’s DbContext.
{
    //… other configuration removed for clarity

    modelBuilder.HasDbFunction( // HasDbFunction will register your method as the way to access your UDF.
        () => MyUdfMethods.AverageVotes(default(int))) // Adds a call to your static method representation of your UDF code
        .HasSchema("dbo"); // You can add options. Here, you add HasSchema (not needed in this case); other options include HasName.
}
````



#### Configuring a table-valued UDF

Allow you to return multiple values in the same way that querying a table returns multiple values. The difference from querying a normal table is that the table-valued UDF can execute SQL code inside the database, using the parameters you provide to the UDF.

The table UDF example returns three values: the `Book`’s `Title`, the number of `Reviews`, and the average `Review Votes` for the `Book`. This example needs a class to be defined that will accept the three values coming back from the table-valued UDF, as shown in the following code snippet:

````c#
public class TableFunctionOutput
{
    public string Title { get; set; }
    public int ReviewsCount { get; set; }
    public double? AverageVotes { get; set; }
}
````



Unlike a scalar UDF, a table UDF can be defined in only one way - within your application’s DbContext - because it needs access to a method inside the `DbContext` class called `FromExpression`. What you are doing is defining the name and signature of the table-valued UDF: the name, the return type, and the parameters’ type all must match your UTF.



````c#
// Defining a table-valued UDF within your application’s DbContext
public class MyEfCoreContext : DbContext
{
    public DbSet<Book> Books { get; set; }
    //… other code removed for clarity

    public Chapter10EfCoreContext(
        DbContextOptions<MyEfCoreContext> options)
        : base(options) {}

    public IQueryable<TableFunctionOutput> GetBookTitleAndReviewsFiltered(int minReviews) // The return value, the method name, and the parameters type must match your UDF code.
    {
        return FromExpression(() => // The FromExpression will provide the IQueryable result.
                              GetBookTitleAndReviewsFiltered(minReviews)); // You place the signature of the method within the FromExpression parameter.
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<TableFunctionOutput>()
            .HasNoKey(); // You must configure the TableFunctionOutput class as not having a primary key.
        modelBuilder.HasDbFunction(() => GetBookTitleAndReviewsFiltered(default(int))); // You register your UDF method by using the Fluent API.
        //… other configurations left out
    }
}
````

EF Core will replace the inner method call with a call to your UDF when you use it in a query.



#### Adding your UDF code to the database

Before you can use the UDF you’ve configured, you need to get your UDF code into the database. A UDF normally is a set of SQL commands that run on the database, so you need to add your UDF code to the database manually before you call the UDF.

The first way is by adding a UDF by using EF Core’s migration feature. To do this, you use the `migrationBuilder.Sql` method.

Another approach is to add a UDF by using EF Core’s `ExecuteSqlRaw` or `ExecuteSqlInterpolated` method. This approach is more applicable to unit testing than to production use where you aren’t using migrations to create your database, in which case you must add the UDFs manually.



````c#
// Adding your UDF to the database via the ExecuteSqlRaw method
public const string UdfAverageVotes = nameof(MyUdfMethods.AverageVotes); // Captures the name of the static method that represents your UDF and uses it as the name of the UDF you add to the database

context.Database.ExecuteSqlRaw( // Uses EF Core’s ExecuteSqlRaw method to add the UDF to the database
    $"CREATE FUNCTION {UdfAverageVotes} (@bookId int)" + // The SQL code that follows adds a UDF to an SQL server database.
    @" RETURNS float
    AS
    BEGIN
    DECLARE @result AS float
    SELECT @result = AVG(CAST([NumStars] AS float))
    	FROM dbo.Review AS r
		WHERE @bookId = r.BookId
	RETURN @result
	END");
````



#### Using a registered UDF in your database queries

You can use this method as a return variable or as part of the query filter or sorting. The following listing has a query that includes a call to a scalar-values UDF that returns information about a book, including the average review votes.

````c#
// Using a scalar-valued UDF in a EF Core query
var bookAndVotes = context.Books.Select(x => new Dto // A normal EF Core query on the Books table
{
    BookId = x.BookId,
    Title = x.Title,
    AveVotes = MyUdfMethods.AverageVotes(x.BookId) // Calls your scalar valued UDF by using its representing method
}).ToList();
````

````sql
SELECT [b].[BookId], [b].[Title],
[dbo].AverageVotes([b].[BookId]) AS [AveVotes]
FROM [Books] AS [b]
````



A table-valued UDF requires a class to return the multiple values. The following code snippet shows a call to our `GetBookTitleAndReviewsFiltered` table-valued UDF:

````c#
var result = context.GetBookTitleAndReviewsFiltered(4)
    .ToList()
````



Scalar and table UDFs can also be used in any part of an EF Core query, as return values or for sorting or filtering. Here’s another example, in which your scalar-valued UDF returns only books whose average review is 2.5 or better:

````c#
var books = context.Books
    .Where(x => MyUdfMethods.AverageVotes(x.BookId) >= 2.5)
    .ToList();
````



### Computed column: A dynamically calculated column value

The main reason for using computed columns is to move some of the calculation - such as some string concatenations - into the database to improve performance. Another good use of computed columns is to return a useful value based on other columns in the row. An SQL computed column containing `[TotalPrice] AS (NumBook * BookPrice)`, for example, would return the total price for that order.

A *computed column* is a column in a table whose value is calculated by using other columns in the same row and/or an SQL built-in function. You can also call systems or UDFs with columns as parameters.

There are two versions of SQL *computed columns*:

* One that does the calculation every time the column is read. This type is a called *dynamic computed column*.
* One that does the calculation only when the entity is updated. This type is a called *persisted computed column* or *stored generated column*. Not all databases support persisted computed columns.

As an example of both types of SQL computed columns, you’ll use a dynamic computed column to get only the year of the person’s birth from a backing field that holds the date of birth.

The second example of SQL computed columns is a persisted computed column that fixes the problem of not using lambda properties in entity classes. In that example, you had a `FullName` property, which was formed by combining the `FirstName` and `LastName` properties, but you couldn’t use a lambda property, as EF Core can’t filter/order on a lambda property. When you use a persisted computed column, however, the computed column is updated whenever the row is updated, and you can use the `FullName` column in any filter, order, search, and similar operation. You declare the properties in the normal way in the class, as shown in the following listing, but because the computed columns are read-only, you make the setter private.

````c#
// Person entity class with two computed column properties
public class Person
{
    public int PersonId { get; set; }
    public int YearOfBirth { get; private set; }

    [MaxLength(50)]
    public string FirstName { get; set; }
    [MaxLength(50)]
    public string LastName { get; set; }
    [MaxLength(101)]
    public string FullName { get; private set; }
    
    //other properties/methods left out…
}
````

Then you need to configure the two computed columns and the index. The only way to configure columns is to use the Fluent API. This listing shows the various configurations for the `Person` entity class.

````c#
// Configuring two computed columns, one persistent, and an index
public class PersonConfig : IEntityTypeConfiguration<Person>
{
    public void Configure(EntityTypeBuilder<Person> entity)
    {
        entity.Property<DateTime>("_dateOfBirth")
            .HasColumnName("DateOfBirth"); // Configures the backing field, with the column name DateOfBirth

        entity.Property(p => p.YearOfBirth)
            .HasComputedColumnSql(
            	"DatePart(yyyy, [DateOfBirth])");

        entity.Property(p => p.FullName)
            .HasComputedColumnSql(
            	"[FirstName] + ' ' + [LastName]",
            	stored:true); // Makes this computed column a persisted computed column
        
        entity.HasIndex(x => x.FullName); // Adds an index to the FullName column because you want to filter/sort on that column
    }
}
````



![](./diagrams/images/10_02_computed_column.png)



The dynamic computed column is recalculated on each read: for simple calculations, the compute time will be minimal, but if you call a UDF that accesses the database, the time taken to read the data from the database can increase. Using a persisted computed column overcomes this problem. Both types of computed columns can have an index in some database types, but each database type has limitations and restrictions. SQL Server doesn’t allow an index on computed columns whose value came from a date function, for example.



### Setting a default value for a database column

When you first create a .NET type, it has a default value: `0` for an `int`, `null` for a `string`, and so on. Sometimes, it’s useful to set a different default value for a property.



````c#
public string Answer { get; set; } = "not given";
````



But with EF Core, you have two other ways to set a default value. First, you can configure EF Core to set up a default value within the database by using the `HasDefaultValue` Fluent API method. This method changes the SQL code used to create the table in the database and adds an SQL `DEFAULT` command containing your default value for that column if no value is provided. Generally, this approach is useful if rows are added to your database via raw SQL commands, as raw SQL often relies on the SQL `DEFAULT` command for columns that the SQL `INSERT` doesn’t provide values for.



The second approach is to create your own code that will create a default value for a column if no value is provided. This approach requires you to write a class that inherits the `ValueGenerator` class, which will calculate a default value. Then you have to configure the property or properties to use your `ValueGenerator` class via the `Configure` Fluent API method. This approach is useful when you have a common format for certain type of values, such as creating a unique string for a user’s order of books.

Before exploring each approach, let’s define a few things that EF Core’s default value-setting methods have in common:

* Defaults can be applied to properties, backing fields, and shadow properties. We’ll use the generic term *column* to cover all three types, because they all end up being applied to a column in the database.
* Default values (`int`, `string`, `DateTime`, `GUID`, and so on) apply only to scalar (nonrelational) columns.
* EF Core will provide a default value only if the property contains the CLR default value appropriate to its type. If a property of type `int` has the value `0`, for example, it’s a candidate for some form of provided default value, but if the property’s value isn’t `0`, that nonzero value will be used.
* EF Core’s default value methods work at the entity-instance level, not the class level. The defaults won’t be applied until you’ve called `SaveChanges` or (in the case of the value generator) when you use the `Add` command to add the entity.

Default values happen only on new rows added to the database, not to updates. You can configure EF Core to add a default value in three ways:

* Using the `HasDefaultValue` method to add a constant value for a column
* Using the `HasDefaultValueSql` method to add an SQL command for a column
* Using the `HasValueGenerator` method to assign a value generator to a property



#### Using the HasDefaultValue method to add a constant value for a column

The first approach tells EF Core to add the SQL DEFAULT command to a column when it creates a database migration, providing a simple constant to be set on a column if a new row is created and the property mapped to that column has a default value. You can add the SQL `DEFAULT` command to a column only via a Fluent API method called `HasDefaultValue`. The following code sets a default date of 1 January 2000 to the column `DateOfBirth` in the SQL table called People.

````c#
// Configuring a property to have a default value set inside the SQL database
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<DefaultTest>()
        .Property("DateOfBirth")
        .HasDefaultValue(new DateTime(2000,1,1));
    //… other configurations left out
}
````

If the SQL code that EF Core produces is asked to create/migrate an SQL Server database, it looks like the following SQL snippet, with the default constraint in bold:

````sql
CREATE TABLE [Defaults] (
    [Id] int NOT NULL IDENTITY,
    -- other columns left out
    [DateOfBirth] datetime2 NOT NULL
    	DEFAULT '2000-01-01T00:00:00.000',
    CONSTRAINT [PK_Defaults] PRIMARY KEY ([Id])
);
````



If the column in a new entity has the CLR default value, EF Core doesn’t provide a value for that column in the SQL `INSERT`, which means that the database server will apply the default constraint of the column definition to provide a value to insert into the new row.

If you are working with a database not created by EF Core, you still need to register the configuration because EF Core must not set that column if the value in the related property contains the CLR default value for that type.



#### Using the HasDefaultValueSql method to add an SQL command for a column

In some situations, it’s useful to get the time when a row is added to the database. In such a case, instead of providing a constant in the SQL `DEFAULT` command, you can provide an SQL function that will provide a dynamic value when the row is added to the database. SQL Server, for example, has two functions - `getdate` and `getutcdate` - that provide the current local datatime and the UTC datatime, respectively.



````c#
protected override void
OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<DefaultTest>()
        .Property(x => x.CreatedOn)
        .HasDefaultValueSql("getutcdate()");
    …
}
````



If you want to use this column to track when the row was added, you need to make sure that the .NET property isn’t set by code (remains at the default value). You do this by using a property with a private setter.



````c#
public DateTime CreatedOn { get; private set; }
````



#### Using the HasValueGenerator method to assign a value generator to a property

The third approach to adding a default value is executed inside your EF Core code. EF Core allows the class that inherits from the class `ValueGenerator` or `ValueGenerator<T>` to be configured as a value generator for a property or backing field.

* The entity’s `State` is set to Added; the entity is deemed to be a new entity to be added to the database.
* The property hasn’t already been set; its value is at the .NET type’s default value.



````c#
// A value generator that produces a unique string for the OrderId
public class OrderIdValueGenerator : ValueGenerator<string> // The value generator needs to inherit from EF Core’s ValueGenerator<T>.
{
    public override bool GeneratesTemporaryValues => false; // Set this to false if you want your value to be written to the database.

    public override string Next(EntityEntry entry) // This method is called when you Add the entity to the DbContext.
    {
        var name = entry // The parameter gives you access to the entity that the value generator is creating a value for. You can access its properties.
            .Property(nameof(DefaultTest.Name)) // Selects the property called "Name" and gets its current value
            .CurrentValue;
        var ticks = DateTime.UtcNow.ToString("s"); // Provides the date in sortable format
        var guidString = Guid.NewGuid().ToString(); // Provides a unique string
        var orderId = $"{name}-{ticks}-{guidString}"; // The orderId combines these three parts to create a unique orderId containing useful info.
        return orderId; // The method must return a value of the Type you have defined at T in the inherited ValueGenerator<T>.
    }
}
````

The following code configures the use of a value generator:

````c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<DefaultTest>()
        .Property(p => p.OrderId)
        .HasValueGenerator((p, e) => new OrderIdValueGenerator());
    …
}
````



The value generator’s `Next` method is called when you `Add` the entity via `context.Add(newEntity)` but before the data is written to the database. Any database-provided values, such as the primary key using SQL `IDENTITY`, won’t be set when the `Next` method is called.

The value generator is a specialized feature with limited applications.



### Sequences: Providing numbers in a strict order

Sequences in a database enable you to produce numbers in strict order with no gaps, such as 1,2,3,4. Key values created by the SQL `IDENTITY` command aren’t guaranteed to be in sequence; they might be like this: 1,2,10,11. Sequences are useful when you want a guaranteed known sequence, such as for an order number for purchases.

The way that sequences are implemented differs among database servers, but in general, a sequence is assigned not to a specific table or column, but to a schema. Every time a column wants a value from the sequence, it asks for that value. EF Core can set up a sequence and then, by using the `HasDefaultValueSql` method, set the value of a column to the next value in the sequence.



````c#
// The DbContext with the Fluent API configuration and the Order class
class MyContext : DbContext
{
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasSequence<int>("OrderNumbers", "shared") // Creates an SQL sequence OrderNumber in the schema "shared." If no schema is provided, it uses the default schema.
            .StartsAt(1000)
            .IncrementsBy(5); // (Optional) Allows you to control the sequence’s start and increments. The default is to start at 1 and increment by 1.

        modelBuilder.Entity<Order>()
            .Property(o => o.OrderNo)
            .HasDefaultValueSql("NEXT VALUE FOR shared.OrderNumbers"); // A column can access the sequence number via a default constraint. Each time the NEXT VALUE command is called, the sequence is incremented.
    }
}

public class Order
{
    public int OrderId { get; set; }
    public int OrderNo { get; set; }
}
````



### Marking database-generated properties

When working with an existing database, you may need to tell EF Core about specific columns that are handled differently from what EF Core expects. If your existing database has a computed column that you didn’t set up by using EF Core’s Fluent API, EF Core needs to be told that the column is computed so that it handles the column properly.

You don’t need any of the features in this section if you use EF Core to do the following:

* Create or migrate the database via EF Core.
* Reverse-engineer your database. (EF Core reads your database schema and generates your entity classes and application DbContext.)

You might use these features if you want to use EF Core with an existing database without reverse engineering. In that case, you need to tell EF Core about columns that don’t conform to its normal conventions. The following sections teach you how to mark three different types of columns:

* Columns that change on inserting a new row or updating a row
* Columns that change on inserting a new row
* "Normal" columns - that is, columns that are changed only by EF Core



#### Marking a column that’s generated on an addition or update

EF Core needs to know whether a column’s value is generated by the database, such as a computed column, if for no other reason than it’s read-only. EF Core can’t "guess" that the database sets a column’s value, so you need to mark it as such. You can use Data Annotations or the Fluent API.

The Data Annotation for an add-or-update column is shown in the following code snippet. Here, EF Core is using the existing `DatabaseGeneratedOption.Computed` setting. The setting is called `Computed` because that’s the most likely reason for a column to be changed on add or update:

````c#
public class PersonWithAddUpdateAttibutes
{
    …
    [DatabaseGenerated(DatabaseGeneratedOption.Computed)]
    public int YearOfBirth { get; set; }
}
````

This code snippet uses the Fluent API to set the add-or-update setting for the column:

````c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Person>()
        .Property(p => p.YearOfBirth)
        .ValueGeneratedOnAddOrUpdate();
    …
}
````



#### Marking a column’s value as set on insert of a new row

You can tell EF Core that a column in the database will receive a value via the database whenever a new row is inserted to the database.

* Via an SQL `DEFAULT` command, which provides a default value if no value is given in the `INSERT` command.
* By means of some form of key generation, of which SQL’s `IDENTITY` command is the primary method. In these cases, the database creates a unique value to place in the column when a new row is inserted.

If a column has the SQL `DEFAULT` command on it, it will set the value if EF Core creates a new row and no value was provided with a value. In that case, EF Core must read back the value that the SQL `DEFAULT` command set for the column; otherwise, the data inside your entity class will not match the database.

The other situation in which EF Core needs to read back the value of a column is for a primary-key column when the database provides the key value, because EF Core won’t know that the key was generated by SQL’s `IDENTITY` command.

````c#
public class MyClass
{
    public int MyClassId { get; set;}
    …
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int SecondaryKey { get; set;}
}
````

The second example does the same thing but uses the Fluent API. For this example, you have a column with a default constraint:

````c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Person>()
        .Property("DateOfBirth")
        .ValueGeneratedOnAdd();
    …
}
````



#### Marking a column/property as "normal"

All scalar properties that aren’t keys, don’t have an SQL default value, and aren’t computed columns are normal - that is, only you set the value of the property. In rare cases, you may want to set a property to be normal, and EF Core provides ways to do that. The one case in which this approach might be useful is for a primary key that uses a GUID; in that case, your software supplies the value.



````c#
public class MyClass
{
    [DatabaseGenerated(DatabaseGeneratedOption.None)]
    public Guid MyClassId { get; set;}
    …
}
````

You can also use the following Fluent API configuration:

````c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<MyClass>()
        .Property("MyClassId")
        .ValueGeneratedNever();
    …
}
````



### Handling simultaneous updates: Concurrency conflicts

By default, EF Core uses an Optimistic Concurrency pattern.



![](./diagrams/images/10_03_optimistic_concurrency.png)



#### Why do concurrency conflicts matter?

In some cases, concurrent conflicts do matter. In financial transactions, for example, you can imagine that the purity and auditing of data are going to be important, so you might want to guard against concurrency changes.

When concurrent conflicts are issues, and you can’t design around them, EF Core provides two ways of detecting a concurrent update and, when the update is detected, a way of getting at all the relevant data so you can implement code to fix the issue.



#### EF Core’s concurrency conflict–handling features

EF Core’s concurrency conflict-handling features can detect a concurrency update in two ways, activated by adding one of the following to an entity class:

* A *concurrency token* to mark a specific property/column in your entity class as one to check for a concurrency conflict
* A *timestamp* (also known as a rowversion), which marks a whole entity class/row as one to check for a concurrency conflict

In both cases, when `SaveChanges` is called, EF Core produces database server code to check for updates of any entities that contain concurrency tokens or timestamps. If that code detects that the concurrency tokens or timestamps have changed since it read the entity, it throws a `DbUpdateConcurrencyException` exception. At that point, you can use EF Core’s features to inspect the differing versions of the data and apply your custom code to decide which of the concurrent updates wins.



**DETECTING A CONCURRENT CHANGE VIA CONCURRENCY TOKEN**

The concurrency-token approach allows you to configure one or more properties as concurrency tokens. This approach tells EF Core to check whether the current database value is the same as the value found when the tracked entity was loaded as part of the SQL `UPDATE` command sent to the database. That way, the update will fail if the loaded value and the current database value are different.



![](./diagrams/images/10_04_concurrency_token.png)



````c#
// The ConcurrencyBook entity class, with a PublishedOn property
public class ConcurrencyBook
{
    public int ConcurrencyBookId { get; set; }
    public string Title { get; set; }

    [ConcurrencyCheck] // Tells EF Core that the PublishedOn property is a concurrency token, which means that EF Core will check whether it has changed when you update it
    public DateTime PublishedOn { get; set; }

    public ConcurrencyAuthor Author { get; set; }
}
````

Alternatively, you can define a concurrency token via the Fluent API:

````c#
// Setting a property as a concurrency token by using the Fluent API
protected override void OnModelCreating(ModelBuilder modelBuilder) // The OnModelCreating method is where you place the configuration of the concurrency detection.
{
    modelBuilder.Entity<ConcurrencyBook>()
        .Property(p => p.PublishedOn)
        .IsConcurrencyToken(); // Defines the PublishedOn property as a concurrency token, which means that EF Core checks whether it has changed when writing out an update

    //… other configuration removed
}
````



````c#
// Simulating a concurrent update of the PublishedOn column
var firstBook = context.Books.First(); // Loads the first book in the database as a tracked entity

context.Database.ExecuteSqlRaw(
    "UPDATE dbo.Books SET PublishedOn = GETDATE()" +
    " WHERE ConcurrencyBookId = @p0",
    firstBook.ConcurrencyBookId); // Simulates another thread/application, changing the PublishedOn column of the same book
firstBook.Title = Guid.NewGuid().ToString(); // Changes the title in the book to cause EF Core to update the book
context.SaveChanges(); // This SaveChanges throws a DbUpdateConcurrencyException.
````

The important thing to note is that only the property marked as a concurrency token is checked. If your SQL-simulated update changed, say, the Title property, which isn’t marked as a concurrency token, no exception would be thrown.

You can see this effect in the SQL that EF Core produces to update the Title in the next listing. The SQL WHERE clause contains not only the primary key of the book to update, but also the PublishedOn column.

````sql
-- SQL code to update Book where PublishedOn is a concurrency token
SET NOCOUNT ON;
UPDATE [Books] SET [Title] = @p0
WHERE [ConcurrencyBookId] = @p1
	AND [PublishedOn] = @p2; -- The test fails if the PublishedOn column has changed, which stops the update.
SELECT @@ROWCOUNT; -- Returns the number of rows updated by this SQL command
````

When EF Core runs this SQL command, the `WHERE` clause finds a valid row to update only if the PublishedOn column hasn’t changed from the value EF Core read in from the database. Then EF Core checks the number of rows that have been updated by the SQL command. If the number of rows updated is zero, EF Core raises `DbUpdateConcurrencyException` to say that a concurrency conflict exists; it can catch a concurrency conflict caused by another task by changing the PublishedOn column or deleting the row when this task does an update.

The good thing about using a concurrency token is that it works on any database because it uses basic commands.



**DETECTING A CONCURRENT CHANGE VIA TIMESTAMP**

The second way to check for concurrency conflicts is to use what EF Core calls a timestamp. A timestamp works differently from a concurrency token, as it uses a unique value provided by the database server that changes whenever a row is inserted or updated. The whole entity, rather than specific properties or columns, is protected against concurrency changes.
When a row with a property/column marked as a timestamp is inserted or updated, the database server produces a new, unique value for that column, which has the effect of detecting an update to an entity/row whenever `SaveChanges` is called.

The timestamp database type is database-type-specific.




![](./diagrams/images/10_05_concurrent_change_via_timestamp.png)



The following listing adds a `ChangeCheck` property, which watches for any updates to the whole entity, to an entity class called `ConcurrencyAuthor`. In this case, the `ChangeCheck` property has a `Timestamp` Data Annotation, which tells EF Core to mark it as a special column that the database will update with a unique value.

In the case of SQL Server, the database provider will set the column as an SQL Server `rowversion`; other databases have different approaches to implementing the `TimeStamp` column.

````c#
// The ConcurrencyAuthor class, with the ChangeCheck property
public class ConcurrencyAuthor
{
    public int ConcurrencyAuthorId { get; set; }
    public string Name { get; set; }
    [Timestamp] // Marks the ChangeCheck property as a timestamp, causing the database server to mark it as an SQL ROWVERSION. EF Core checks this property when updating to see whether it has changed.
    public byte[] ChangeCheck { get; set; }
}
````

Alternatively, you can use the Fluent API to configure a timestamp, as shown in the following listing:

````c#
// Configuring a timestamp by using the Fluent API
protected override void OnModelCreating(ModelBuilder modelBuilder) // OnModelCreating is where you place the configuration of the concurrency detection.
{
    modelBuilder.Entity<ConcurrencyAuthor>()
        .Property(p => p.ChangeCheck)
        .IsRowVersion(); // Defines an extra property called ChangeCheck that will be changed every time the row is created/updated. EF Core checks whether this property has changed when it does an update.
}
````

Both configurations create a column in a table that the database server will change automatically whenever there’s an `INSERT` or `UPDATE` to that table. For SQL Server database, the column type is set to `ROWVERSION`, as shown in the following listing. Other database servers can use different approaches, but they all provide a new, unique value on an `INSERT` or `UPDATE`.

````sql
-- The SQL to create the Authors table, with a timestamp column
CREATE TABLE [dbo].[Authors] (
    [ConcurrencyAuthorId] INT IDENTITY (1, 1),
    [ChangeCheck] TIMESTAMP NULL, -- If the table is created by EF Core, sets the column type to TIMESTAMP if your property is of type byte[]. This column’s value will be updated on each INSERT or UPDATE.
    [Name] NVARCHAR (MAX) NULL
);
````



You simulate a concurrent change, which consists of three steps:

1. You use EF Core to read in the Authors row that you want to update.
2. You use an SQL command to update the Authors table, simulating another task updating the same Author that you read in. EF Core doesn’t know anything about this change because raw SQL bypasses EF Core’s tracking snapshot feature.
3. In the last two lines, you update the Author’s name and call `SaveChanges`, which causes a `DbUpdateConcurrencyException` to be thrown because EF Core found that the ChangeCheck column has changed from step 1.

````c#
// Simulating a concurrent update of the ConcurrentAuthor entity
var firstAuthor = context.Authors.First(); // Loads the first author in the database as a tracked entity
context.Database.ExecuteSqlRaw(
    "UPDATE dbo.Authors SET Name = @p0" +
    " WHERE ConcurrencyAuthorId = @p1",
    firstAuthor.Name,
    firstAuthor.ConcurrencyAuthorId); // Simulates another thread/application updating the entity. Nothing is changed except the timestamp.
firstAuthor.Name = "Concurrency Name"; // Changes something in the author to cause EF Core to do an update to the book
context.SaveChanges(); // Throws DbUpdateConcurrencyException
````

This code is like the case in which you used a concurrency token. The difference is that the timestamp detects an update of the row via the unique value in the property/ column called `ChangeCheck`. You can see this difference in the following listing, which shows the SQL that EF Core produces to update the row with the check on the timestamp property, `ChangeCheck`.

````sql
-- The SQL code to update the author’s name, with ChangeCheck check
SET NOCOUNT ON;
UPDATE [Authors] SET [Name] = @p0
WHERE [ConcurrencyAuthorId] = @p1
	AND [ChangeCheck] = @p2; -- Checks that the ChangeCheck column hasn’t been changed since you read in the book entity
SELECT [ChangeCheck] -- Because the update will change the ChangeCheck column, EF Core needs to read it back so that its in-memory copy is correct.
FROM [Authors]
WHERE @@ROWCOUNT = 1 -- Checks whether one row was updated in the last command. If not, the ChangeCheck value won’t be returned, and EF Core will know that a concurrent change has taken place.
	AND [ConcurrencyAuthorId] = @p1;
````

The `UPDATE` part checks whether the `ChangeCheck` column is the same value as the copy it found when it first read the entity, and if so, it executes the update. The second part returns the new `ChangeCheck` column that the database server created after the current update, but only if the `UPDATE` was executed. If no value is returned for the `ChangeCheck` property, EF Core knows that a concurrency conflict has happened and throws a `DbUpdateConcurrencyException`.
The concurrency-token approach provides specific protection of the property/properties you place it on and is triggered only if a property marked as a concurrency token is changed. The timestamp approach catches any update to that entity.



#### Handling a DbUpdateConcurrencyException

````c#
// The method you call to save changes that trap concurrency conflicts
public static string BookSaveChangesWithChecks(ConcurrencyDbContext context) // Called after the Book entity has been updated in some way
{
    string error = null;
    try
    {
        context.SaveChanges(); // Calls SaveChanges within a try...catch so that you can catch DbUpdateConcurrencyException if it occurs
    }
    catch (DbUpdateConcurrencyException ex) // Catches DbUpdateConcurrencyException and puts in your code to handle it
    {
        var entry = ex.Entries.Single(); // In this case, you know that only one Book will be updated. In other cases, you might need to handle multiple entities.
        error = HandleBookConcurrency(context, entry); // Calls the HandleBookConcurrency method, which returns null if the error was handled or an error message if it wasn’t
        if (error == null)
        {
            context.SaveChanges(); // If the conflict was handled, you need to call SaveChanges to update the Book.
        }
    }
    return error; // Returns the error message or null if there’s no error
}
````



![](./diagrams/images/10_06_handling_db_update_concurrency_exception.png)



````c#
// Handling a concurrent update on the book
private static string HandleBookConcurrency(DbContext context, EntityEntry entry) // Takes in the application’s DbContext and the ChangeTracking entry from the exception’s Entities property
{
    var book = entry.Entity as ConcurrencyBook;
    if (book == null) // Handles only ConcurrencyBook, so throws an exception if the entry isn’t of type Book
    {
        throw new NotSupportedException("Don't know how to handle concurrency conflicts for " + entry.Metadata.Name);
    }

    var whatTheDatabaseHasNow = // You want to get the data that someone else wrote into the database after your read.
        context.Set<ConcurrencyBook>().AsNoTracking() // Entity must be read as NoTracking; otherwise, it’ll interfere with the same entity you’re trying to write.
        	.SingleOrDefault(p => p.ConcurrencyBookId == book.ConcurrencyBookId);
    if (whatTheDatabaseHasNow == null) // Concurrency conflict method doesn’t handle the case where the book was deleted, so it returns a userfriendly error message.
    {
        return "Unable to save changes.The book was deleted by another user.";
    }

    var otherUserData = context.Entry(whatTheDatabaseHasNow); // You get the EntityEntry<T> version of the entity, which has all the tracking information.

    foreach (var property in entry.Metadata.GetProperties()) // You go through all the properties in the book entity to reset the Original values so that the exception doesn’t happen again.
    {
        var theOriginalValue = entry
            .Property(property.Name).OriginalValue; // Holds the version of the property at the time you did the tracked read of the book
        var otherUserValue = otherUserData
            .Property(property.Name).CurrentValue; // Holds the version of the property as written to the database by someone else
        var whatIWantedItToBe = entry
            .Property(property.Name).CurrentValue; // Holds the version of the property that you wanted to set it to in your update

        // TODO: Logic to decide which value should be written to database
        if (property.Name == nameof(ConcurrencyBook.PublishedOn)) // Business logic to handle PublishedOn: sets to your value or the other person’s value, or throws an exception
        {
            entry.Property(property.Name).CurrentValue = //… your code to pick which PublishedOn to use
        }

        entry.Property(property.Name).OriginalValue =
            otherUserData.Property(property.Name)
            .CurrentValue; // Here, you set the OriginalValue to the value that someone else set it to. This code works for concurrency tokens or a timestamp.
    }
    return null; // You return null to say that you handled this concurrency issue.
}
````



#### The disconnected concurrent update issue



![](./diagrams/images/10_07_disconnected_concurrent_update_issue.png)



If a concurrency conflict exists, the user is shown a new screen with an error message indicating what happened. Then the user is invited to accept the current state or apply the update, knowing that this update will override the last user’s update.



![](./diagrams/images/10_08_disconnected_concurrent_update_issue.png)



````c#
// Entity class used to hold an employee’s salary with concurrency check
public class Employee
{
    public int EmployeeId { get; set; }

    public string Name { get; set; }

    [ConcurrencyCheck]
    public int Salary { get; set; } // Salary property set as a concurrency token by the ConcurrencyCheck attribute

    public void UpdateSalary(DbContext context, int orgSalary, int newSalary) // Updates the Salary in a disconnected state
    {
        Salary = newSalary; // Sets the Salary to the new value
        context.Entry(this).Property(p => p.Salary)
            .OriginalValue = orgSalary; // Sets the OriginalValue, which holds the data read from the database, to the original value that was shown to the user in the first part of the update
    }
}
````

After applying the `UpdateSalary` method to the Employee entity of the person whose salary you want to change, you call `SaveChanges` within a `try…catch` block to update the Employee. If `SaveChanges` raises `DbUpdateConcurrencyException`, the job of the `DiagnoseSalaryConflict` method shown in the following listing isn’t to fix the conflict, but to create an appropriate error message so that the user can decide what to do.

````c#
// Returns different errors for update or delete concurrency conflicts
private string DiagnoseSalaryConflict(ConcurrencyDbContext context, EntityEntry entry) // Called if a DbUpdateConcurrencyException occurs. Its job isn’t to fix the problem but to form an error message and provide options for fixing the problem.
{
    var employee = entry.Entity as Employee;
    if (employee == null) // If the entity that failed wasn’t an Employee, you throw an exception, as this code can’t handle that.
    {
        throw new NotSupportedException("Don't know how to handle concurrency conflicts for " +
                                        entry.Metadata.Name);
    }

    var databaseEntity = // You want to get the data that someone else wrote into the database after your read.
        context.Employees.AsNoTracking() // Must be read as NoTracking; otherwise, it’ll interfere with the same entity you’re trying to write.
        .SingleOrDefault(p => p.EmployeeId == employee.EmployeeId);

    if (databaseEntity == null) // Checks for a delete conflict: the employee was deleted because the user attempted to update it.
    {
        return
            $"The Employee {employee.Name} was deleted by another user. " +
            $"Click Add button to add back with salary of {employee.Salary}" +
            " or Cancel to leave deleted."; // Error message to display to the user, with two choices about how to carry on
    }

    return
        $"The Employee {employee.Name}'s salary was set to " +
        $"{databaseEntity.Salary} by another user. " +
        $"Click Update to use your new salary of {employee.Salary}" +
        $" or Cancel to leave the salary at {databaseEntity.Salary}."; // Otherwise, the error must be an update conflict, so you return a different error message with the two choices for this case.
}
````

The update conflict can be handled by using the same `UpdateSalary` method used for the normal update, but now the `orgSalary` parameter is the salary value as read back when the `DbUpdateConcurrencyException` was raised. The `FixDeleteSalary` method is used when the concurrent user deletes the `Employee` and the current user wants to add the Employee back with their new salary value.

````c#
// Two methods to handle update and delete conflicts
public class Employee
{
    public int EmployeeId { get; set; }

    public string Name { get; set; }

    [ConcurrencyCheck] // Set as a concurrency token by the ConcurrencyCheck attribute
    public int Salary { get; set; } // The same method used to update the Salary can be used for the Update conflict, but this time, it’s given the original value that was found when the DbUpdateConcurrencyException occurred.

    public void UpdateSalary(DbContext context, int orgSalary, int newSalary)
    {
        Salary = newSalary;
        context.Entry(this).Property(p => p.Salary)
            .OriginalValue = orgSalary; // Sets the OriginalValue, which is now the value that the database contained when the DbUpdateConcurrencyException occurred
    }

    public static void FixDeletedSalary(DbContext context, Employee employee) // Handles the Delete concurrency conflict
    {
        employee.EmployeeId = 0; // The key must be at the CLR default value for an Add to work.
        context.Add(employee); // Adds the Employee because it was deleted from the database and therefore must be added back
    }
}
````



### Summary

* Using SQL user-defined functions (UDFs) with EF Core to move calculations into the database can improve query performance.
* Configuring a column as an SQL computed column allows you to return a computed value based on the other properties in the row.
* EF Core provides two ways to set a default value for a property/column in an entity; these techniques go beyond what setting a default value via .NET could achieve.
* EF Core’s `HasSequence` method allows a known, predictable sequence provided by the database server to be applied to a column in a table.
* When the database is created/migrated outside EF Core, you need to configure columns that behave differently from the norm, such as telling EF Core that a key is generated in the database.
* EF Core provides concurrency tokens and timestamps to detect concurrency conflicts.
* When a concurrency conflict is detected, EF Core throws `DbUpdateConcurrencyException` and then allows you to implement code to handle the conflict.





## 11. Going deeper into the DbContext

* Seeing how your application’s DbContext detects changes in tracked entities
* Using the change tracking method in your DbContext to build an audit trail
* Using raw SQL commands via the DbContext’s `Database` property
* Finding the entities to database mapping using DbContext’s `Model` property
* Using EF Core’s database connection resiliency



### Overview of the DbContext class’s properties

* ChangeTracker - Provides access to EF Core’s change tracking code.
* ContextId - A unique identifier for the instance of the DbContext. Its main role is to be a correlation ID for logging and debugging so that you can see what reads and writes were done from the same instance of the application’s DbContext.
* Database - Provides access to three main groups of features:
  - Transaction control
  - Database creation/migration
  - Raw SQL commands
* Model - Provides access to the database model that EF Core uses when connecting to or creating a database.



### Understanding how EF Core tracks changes

EF Core uses a property called `State` that’s attached to all tracked entities. The `State` property holds the information about what you want to happen to that entity when you call the application’s DbContext method, `SaveChanges`.

The following list lists possible values of the State property, which is accessed via the EF command `context.Entry(myEntity).State`:

* `Added` - The entity doesn’t yet exist in the database. `SaveChanges` will insert it.
* `Unchanged` - The entity exists in the database and hasn’t been modified on the client. `SaveChanges` will ignore it.
* `Modified` - The entity exists in the database and has been modified on the client. `SaveChanges` will update it.
* `Deleted` - The entity exists in the database but should be deleted. `SaveChanges` will delete it.
* `Detached` - The entity you provided isn’t tracked. `SaveChanges` doesn’t see it.



When you have an entity in the `Modified` state, another per-property `boolean` flag, `IsModified`, comes into play. This flag identifies which of the properties, both scalar and navigational, have changed in the entity. This `IsModified` property for a scalar property is accessed via

````c#
context.Entry(entity).Property("PropertyName").IsModified;
````

and the `IsModified` property for navigational properties is accessed via

````c#
context.Entry(entity).Navigation("PropertyName").IsModified;
````



### Looking at commands that change an entity’s State

All the EF Core commands/actions that can change a tracked entity’s `State`, showing an example of each command/action and the final tracking `State` of the entity:

| Command/Action                   | Example                              | Final State   |
| -------------------------------- | ------------------------------------ | ------------- |
| `Add`/`AddRange`                 | `context.Add(entity);`               | `Added`       |
| `Remove`/`RemoveRange`           | `context.Remove(entity);`            | `Deleted`     |
| Changing a property              | `entity.MyString = "hello";`         | `Modified`    |
| `Update`/`UpdateRange`           | `context.Update(entity);`            | `Modified`    |
| `Attach`/`AttachRange`           | `context.Attach(entity);`            | `Unchanged`   |
| Setting `State` directly         | `context.Entry(entity).State = …`    | Given `State` |
| Setting `State` via `TrackGraph` | `context.ChangeTracker.TrackGraph(…` | Given `State` |



#### The Add command: Inserting a new row into the database

The `Add`/`AddRange` methods are used to create a new entity in the database by setting the
given entity’s `State` to `Added`. To summarize:

* The entity’s `State` is set to `Added`.
* The `Add` method looks at all the entities linked to the added entity.
  - If a relationship isn’t currently tracked, it is tracked, and its `State` is set to `Added`.
  - If a relationship is tracked, its current `State` is used unless there was a requirement to alter/set a foreign key, in which case its `State` is set to `Modified`.

Also, the `AddAsync`/`AddRangeAsync` methods are available for entities that use a value generator to set a property. If the value generator has a `NextAsync` method, you must use the `AddAsync`/`AddRangeAsync` methods when that entity is added.



#### The Remove method: Deleting a row from the database

The `Remove`/`RemoveRange` methods delete the entity from the database by setting the given entity’s `State` to `Deleted`. If the removed entity has any relationships that are loaded/tracked, the value of the `State` for each relationship entities will be one of the following:

* `State == Deleted` - Typical for a required dependent relationship, such as a `Review` entity class linked to a `Book` entity class
* `State == Modified` - Typical for an optional dependent relationship in which the foreign key is nullable. In this case, the optional relationship is not deleted, but the foreign key that links to the entity that was deleted is set to `null`.
* `State == Unchanged` - Result of deleting a dependent entity class that is linked to a principal class. Nothing changes in the principal class keys/foreign keys when a dependent entity class is deleted.

If the `OnDelete` behavior is set to `Cascade`, which is the default for a required dependent relationship, it will delete any required dependent relationships of the deleted entity class.



#### Modifying an entity class by changing the data in that entity class

* For EF Core to detect a change, the entity must be tracked. Entities are tracked if you read them in without an `AsNoTracking` method in the query or when you call a `Add`, `Update`, `Remove`, or `Attach` method with an entity class as a parameter.
* When you call `SaveChanges`/`SaveChangesAsync`, by default, EF Code executes a method called `ChangeTracker.DetectChanges`, which compares the current entity’s data with the entity’s tracking snapshot. If any properties, backing fields, or shadow properties are different, the entity’s `State` is set to `Modified`, and the properties, backing fields, or shadow properties are set to `IsModified`.



![](./diagrams/images/11_01_modifying_entity_class.png)



#### Modifying an entity class by calling the Update method

The `ChangeTracker.DetectChanges` method won’t work because there is no tracking snapshot to compare. In cases like this one, you can use the `Update` and `UpdateRange` methods.

The Update method tells EF Core to update all the properties/columns in this entity by setting the given entity’s `State` to `Modified` and sets the `IsModified` property to `true` on all nonrelational properties, including any foreign keys. As a result, the row in the database will have all its columns updated.

If the entity type using the `Update` call has loaded relationships, the `Update` method will recursively look at each related entity class and set its `State`. The rules for setting the `State` on a related entity class depend on whether the relationship entity’s primary key is generated by the database and is set (its value isn’t the default value for the key’s .NET type):

* Database-generated key, not the default value - In this case, EF Core will assume that the relationship entity is already in the database and will set the `State` to `Modified` if a foreign key needs to be set; otherwise, the `State` will be `Unchanged`.
* Not database-generated key, or the key is the default value - In this case, EF Core will assume that the relationship entity is new and will set its `State` to `Added`.



#### The Attach method: Start tracking an existing untracked entity class

The `Attach` and `AttachRange` methods are useful if you have an entity class with existing valid data and want it to be tracked. After you attach the entity, it’s tracked, and EF Core assumes that its content matches the current database state. This behavior works well for reconstituting entities with relationships that have been serialized and then deserialized to an entity, but only if the entities are written back to the same database, as the primary and foreign keys need to match.

When you Attach an entity, it becomes a normal tracked entity, without the cost of loading it from the database. The `Attach` method does this by setting the entity’s `State` to `Unchanged`. As with the `Update` method, what happens to the relationships of the updated entity depends on whether the relationship entity’s primary key is generated by the database and is set (its value isn’t the default value for the key’s .NET type):

* *Database-generated key, and key has a default value* - EF Core will assume that the relationship entity is already in the database and will set the `State` to `Added`.
* *Not a database-generated key, or the key is the not default value* - EF Core will assume that the relationship entity is new and will set its `State` to `Unchanged`.



#### Setting the State of an entity directly

Another way to set the `State` of an entity is to set it manually to whatever state you want. This direct setting of an entity’s `State` is useful when an entity has many relationships, and you need to specifically decide which state you want each relationship to have.

Because the entity’s State is read/write, you can set it. In the following code snippet, the `myEntity` instance’s `State` is set to `Added`:

````c#
context.Entry(myEntity).State = EntityState.Added;
````

You can also set the `IsModified` flag on the property in an entity. The following code snippet sets the `MyString` property’s `IsModified` flag to true, which sets the entity’s `State` to `Modified`:

````c#
var entity = new MyEntity();
context.Entry(entity).Property("MyString").IsModified = true;
````



#### TrackGraph: Handling disconnected updates with relationships

The `TrackGraph` method is useful if you have an untracked entity with relationships, and you need to set the correct `State` for each entity. The `TrackGraph` method will traverse all the relational links in the entity, calling an action you supplied on each entity it finds. This method is useful if you have a group of linked entities coming from a disconnected situation (say, via some form of serialization), and you want to change only part of the data you’ve loaded.



````c#
// Using TrackGraph to set each entity’s State and IsModified flags
var book = … untracked book with all relationships // Expects an untracked book with its relationships
context.ChangeTracker.TrackGraph(book, e => // Calls ChangeTracker.TrackGraph, which takes an entity instance and an Action, which, in this case, you define via a lambda. The Action method is called once on each entity in the graph of entities.
{
    e.Entry.State = EntityState.Unchanged; // If the method sets the state to any value other than Detached, the entity will become tracked by EF Core.

    if (e.Entry.Entity is Author) // Here, you want to set only the Name property of the Author entity to Modified, so you check whether the entity is of type Author.
    {
        e.Entry.Property("Name").IsModified = true; // Sets the IsModified flag on the Name property; also sets the State of the entity to Modified
    }
});
context.SaveChanges(); // Calls SaveChanges, which finds that only the Name property of the Author entity has been marked as changed; creates the optimal SQL to update the Name column in the Authors table
````

`TrackGraph` traverses the entity provided as its first parameter and any entities that are reachable by traversing its navigation properties. The traversal is recursive, so the navigation properties of any discovered entities will also be scanned. The `Action` method you provide as the second parameter is called for each discovered untracked (`State == Detached`) entity and can set the `State` that each entity should be tracked in. If the visited entity’s `State` isn’t set, the entity remains in the `State` of `Detached` (that is, the entity isn’t being tracked by EF Core). Also, `TrackGraph` will ignore any entities it visits that are currently being tracked.

Although you could still use the `Update` command for this purpose, doing so would be inefficient because the command would update every table and column in the book’s relationships instead of only the authors’ names. EF Core’s `ChangeTracker.TrackGraph` method provides a better approach.

Using `TrackGraph` allows you to target the specific entity and property you want to set the `State` to a new value; in this case, you set the property called `Name` to `IsModified` in any `Author` entity class in the relationships of the `Book` entity.

The result of running this code is that only the `Author` entity instance’s `State` is set to `Modified`, whereas the `State` of all the other entity types is set to `Unchanged`. In addition, the `IsModified` flag is set only on the `Author` entity class’s `Name` property. In this example, the difference between using an `Updated` method and using the `TrackGraph` code reduces the number of database updates: the `Updated` method would produce 20 column updates (19 of them needlessly), whereas the `TrackGraph` code would change only one column.



### SaveChanges and its use of ChangeTracker.DetectChanges

#### How SaveChanges finds all the State changes

Whereas states such as `Added` and `Deleted` are set by the EF Core commands, the "change a property" approach to updates relies on code to compare each entity class with its tracking snapshot. To do so, `SaveChanges` calls a method called `DetectChanges` that is accessed via the `ChangeTracker` property. If you have a lot of entities with lots of data, the process can become slow.



![](./diagrams/images/11_02_savechanges_finds_all_the_state_changes.png)



#### What to do if ChangeTracker.DetectChanges is taking too long

In some applications, you may have a large number of tracked entities loaded. The problem is if you have a large amount of tracked entity instances and/or your entities have a lot of data in them. In that case, a call to `SaveChanges`/`SaveChangesAsync` can become slow. If you are saving a lot of data, the slowness is most likely caused by the database accesses. But if you are saving only a small amount of data, any slowdown is likely due to the time the `ChangeTracker.DetectChanges` takes to compare each entity class instance with its matching tracking snapshot.

These features work by detecting individual updates to the data in your entity classes, cutting out any comparisons of data that hasn’t been changed.

You have four ways to replace the `ChangeTracker.DetectChanges`; each approach has different features and different levels of effort to implement.

A comparison of the four approaches you can use to stop the `ChangeTracker.DetectChanges` method from looking at an entity, thus saving time:

`INotifyPropertyChanged`

* Pros
  * Can change only the entities that are slow
  * Handles concurrency exceptions
* Cons
  * Need to edit every property

`INotifyPropertyChanged` and `INotifyPropertyChanging`

* Pros
  * Can change only the entities that are slow
  * No tracking snapshot, so uses less memory
* Cons
  * Need to edit every property

Proxy change tracking `INotifyPropertyChanged`

* Pros
  * Easy to code; add virtual to every property
  * Handles concurrency exceptions
* Cons
  * Must change all entity types to use proxy

Proxy change tracking `INotifyPropertyChanged` and `INotifyPropertyChanging`

* Pros
  * Easy to code; add virtual to every property
  * No tracking snapshot, so uses less memory
* Cons
  * Must change all entity types to use proxy
  * Have to create a new entity class via the `CreateProxy<T>` method

Overall, the proxy change tracking feature is easier to code but requires you to change all your entity classes to use proxy change tracking. But if you find a `SaveChanges` performance issue in an existing application, changing all your entity classes might be too much work. For this reason, We can focus on the first approach, `INotifyPropertyChanged`, which is easy to add to a few entity classes that have a problem, and the last approach, proxy changed/changing tracking, which is easier but requires you to use it across the whole application.



**FIRST APPROACH: INOTIFYPROPERTYCHANGED**

EF Core supports the `INotifyPropertyChanged` interface on an entity class to detect whether any property has changed. This interface notifies EF Core that a property has changed, but you have to raise a `PropertyChanged` event, which means the `ChangeTracker.DetectChanges` method isn’t used.

To use the `INotifyPropertyChanged` interface you need to create a `NotificationEntity` helper class. This class provides a `SetWithNotify` method that you call when any property in your entity class changes.



````c#
// NotificationEntity helper class that NotifyEntity inherits
public class NotificationEntity : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected void SetWithNotify<T>(T value, ref T field, [CallerMemberName] string propertyName = "") // Automatically gets the propertyName by using System.Runtime.CompilerServices
    {
        if (!Object.Equals(field, value)) // Only if the field and the value are different do you set the field and raise the event.
        {
            field = value; // Sets the field to the new value
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName)); // Invokes the PropertyChanged event, but using ?. to stop the method from failing when the new entity is created and the PropertyChangedEventHandler hasn’t been filled in by EF Core with the name of the property
        }
    }
}
````

You must call the `SetWithNotify` method whenever a noncollection property changes. For collections, you have to use a `ObservableCollection` to raise an event when a navigational collection property is changed.

````c#
// NotifyEntity using NotificationEntity class for events
public class NotifyEntity : NotificationEntity
{
    // Each noncollection property must have a backing field.
    private int _id;
    private string _myString;
    private NotifyOne _oneToOne;

    public int Id
    {
        get => _id;
        set => SetWithNotify(value, ref _id);
    }

    public string MyString
    {
        get => _myString;
        set => SetWithNotify(value, ref _myString);
    }

    public NotifyOne OneToOne
    {
        get => _oneToOne;
        set => SetWithNotify(value, ref _oneToOne);
    }

    public ObservableCollection<NotifyMany> Collection { get; } // Any collection navigational property must be an Observable collection, so you need to predefine that Observable collection.
    = new ObservableCollection<NotifyMany>(); // You can use any Observable collection, but for performance reasons, EF Core prefers ObservableHashSet<T>.
}
````



After you’ve defined your entity class to use the `INotifyPropertyChanged` interface, you must configure the tracking strategy for this entity class to `ChangedNotifications`. This configuration tells EF Core not to detect changes via `ChangeTracker.DetectChanges` because it will be notified of any changes via `INotifyPropertyChanged` events. To configure `INotifyPropertyChanged` events for one entity class, you use the Fluent API command.

````c#
// Setting the tracking strategy for one entity to ChangedNotifications
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder
        .Entity<NotifyEntity>()
        .HasChangeTrackingStrategy(
        	ChangeTrackingStrategy.ChangedNotifications);
}
````



**APPROACHES 2 AND 3**

* The `NotificationEntity` class must create change and changing events.
* You use a different `ChangeTrackingStrategy` setting, such as `ChangingAndChangedNotifications`.



**LAST APPROACH: PROXY CHANGE TRACKING**

The last approach uses proxy change tracking via the `INotifyPropertyChanged` and `INotifyPropertyChanging` events introduced. These change-tracking events are added to the lazy-loading proxy approach with the virtual properties. To use this approach, you need to do five things:

* Change all your entity classes to have virtual properties.
* Use an `Observable` collection type for navigational collection properties.
* Change your code that creates a new instance of an entity class to use the `CreateProxy<TEntity>` method.
* Add the NuGet library `Microsoft.EntityFrameworkCore.Proxies`.
* Add the method `UseChangeTrackingProxies` when building the application’s DbContext options.



````c#
// An example entity class set up to use proxy change tracking
public class ProxyMyEntity
{
    // All properties must be virtual.
    public virtual int Id { get; set; }
    public virtual string MyString { get; set; }
    public virtual ProxyOptional ProxyOptional { get; set; }

    public virtual ObservableCollection<ProxyMany> Many { get; set; } = new ObservableCollection<ProxyMany>(); // For navigational collection properties, you need to use an Observable collection type.
}
````

If you read in an entity class via a query, the proxy change tracking will add its extra code to create the `INotifyPropertyChanged` and `INotifyPropertyChanging` events when a property is changed. But if you want to create a new entity class, you can’t use the normal new command, such as `new Book()`. Instead, you must use the `CreateProxy<TEntity>` method.

````c#
var entity = context.CreateProxy<ProxyMyEntity>();
entity.MyString = "hello";
context.Add(entity);
context.SaveChanges();
````

You must use the `CreateProxy<TEntity>` method (first line of the preceding code snippet); otherwise, EF Core won’t be able to detect the changing event.

The final part is making sure that the `Microsoft.EntityFrameworkCore.Proxies` NuGet package is loaded and then updating your DbContext configuration to include the `UseChangeTrackingProxies` method, as shown in the following code snippet:

````c#
var optionsBuilder = new DbContextOptionsBuilder<EfCoreContext>();
optionsBuilder
    .UseChangeTrackingProxies()
    .UseSqlServer(connection);
var options = optionsBuilder.Options;

using (var context = new EfCoreContext(options))
````



#### Using the entities’ State within the SaveChanges method

So far, you’ve learned how to set the `State` of an entity and heard about how `ChangeTracker` can be used to find out what has changed. Now you are going to use the `State` data within the `SaveChanges`/`SaveChangesAsync` to do some interesting things. Here are some of the possible uses of detecting what’s about to be changed in the database:

* Automatically adding extra information to an entity - for instance, adding the time when an entity was added or updated
* Writing a history audit trail to the database each time a specific entity type is changed
* Add security checks to see whether the current user is allowed to update that particular entity type

The basic approach is to override the `SaveChanges`/`SaveChangesAsync` methods inside your application’s DbContext and execute a method before the base `SaveChanges`/`SaveChangesAsync` is called. We check the `States` before the base `SaveChanges` is called because a) the `State` of every tracked entity will have a value of `Unchanged` once `SaveChanges` is called and b) you want to add/alter some of the entities before they are written to the database. What you do with the `State` information is up to you.



````c#
// The ICreatedUpdated interface defining four properties and a method
public interface ICreatedUpdated // Add this interface to any entity class where you want to log when/who it was created or updated.
{
    DateTime WhenCreatedUtc { get; } // Holds the datetime when the entity was first added to the database
    Guid CreatedBy { get; } // Holds the UserId who created the entity
    DateTime LastUpdatedUtc { get; } // Holds the datetime when the entity was last updated
    Guid LastUpdatedBy { get; } // Holds the UserId who last updated the entity
    
    void LogChange(EntityState state, Guid userId = default); // Called when the entity’s state is Added or Modified State. Its job is to update the properties based on the state.
}
````

The `LogChange` method, which you’ll call in your modified `SaveChanges` method, sets the various properties in the entity class.

````c#
// Automatically setting who and when a entity was updated
public class CreatedUpdatedInfo : ICreatedUpdated // Entity class inherits ICreatedUpdated, which means any addition/update of the entity is logged.
{
    // These properties have private setters so that only the LogChange method can change them.
    public DateTime WhenCreatedUtc { get; private set; }
    public Guid CreatedBy { get; private set; }
    public DateTime LastUpdatedUtc { get; private set; }
    public Guid LastUpdatedBy { get; private set; }

    public void LogChange(EntityEntry entry, Guid userId = default) // Its job is to update the created and updated properties. It is passed the UserId if available.
    {
        if (entry.State != EntityState.Added && entry.State != EntityState.Modified) // This method only handles Added or Modified States.
        {
            return;
        }

        var timeNow = DateTime.UtcNow; // Obtains the current time so that an add and update time will be the same on create
        LastUpdatedUtc = timeNow;
        LastUpdatedBy = userId; // It always sets the LastUpdatedUtc and LastUpdatedBy.
        if (entry.State == EntityState.Added) // If it’s an add, then you update the WhenCreatedUtc and the CreatedBy properties.
        {
            WhenCreatedUtc = timeNow;
            CreatedBy = userId;
        }
        else
        {
            // For performance reasons you turned off DetectChanges, so you must manually mark the properties as modified.
            entry.Property(
                nameof(ICreatedUpdated.LastUpdatedUtc))
                .IsModified = true;
            entry.Property(
                nameof(ICreatedUpdated.LastUpdatedBy))
                .IsModified = true;
        }
    }
}
````

The next step is to override all versions of the `SaveChanges` method inside your application’s DbContext and then precede the call to the base `SaveChanges` with a call to your `AddUpdateChecks` method. This method looks for entities with a `State` of `Added` or `Modified` and inherits the `ICreatedUpdated` interface. If the method finds an entity (or entities) that fits that criteria, it calls the entity’s `LogChange` method to set the two properties to the correct values.

Notice too that the code ensures the `ChangeTracker.DetectChanges` method is only called once, because, as you have seen, that method can take some time.

````c#
// Your DbContext looks for added or modified ICreatedUpdated entities
private void AddUpdateChecks() // This private method will be called from SaveChanges and SaveChangesAsync.
{
    ChangeTracker.DetectChanges(); // It calls DetectChanges to make sure all the updates have been found.
    foreach (var entity in ChangeTracker.Entries()
             .Where(e =>
                    e.State == EntityState.Added ||
                    e.State == EntityState.Modified)) // It loops through all the tracked entities that have a State of Added or Modified.
    {
        var tracked = entity.Entity as ICreatedUpdated; // If the Added/Modified entity has the ICreatedUpdated, then the tracked isn’t null.
        tracked?.LogChange(entity); // So we call the LogChange command. In this example we don’t have the UserId available.
    }
}

public override int SaveChanges(bool acceptAllChangesOnSuccess) // You override SaveChanges (and SaveChangesAsync—not shown).
{
    AddUpdateChecks(); // You call the AddUpdateChecks, which contains a call to ChangeTracker.DetectChanges().
    try
    {
        ChangeTracker.AutoDetectChangesEnabled = false; // Because DetectChanges has been called we tell SaveChanges not to call it again (for performance reasons).
        return base.SaveChanges(acceptAllChangesOnSuccess); // You call the base.SaveChanges that you overrode
    }
    finally
    {
        ChangeTracker.AutoDetectChangesEnabled = true; // Finally to turn the AutoDetectChangesEnabled back on
    }
}
````

This is only one example of using `ChangeTracker` to take actions based on the `State` of tracked entities, but it establishes the general approach. The possibilities are endless.



#### Catching entity class’s State changes via events

EF Core added two events: `ChangeTracker.Tracked`, which is triggered when an entity is first tracked, and `ChangeTracker.StateChanged`, which is triggered when the `State` of an already tracked entity is changed. This feature provides a similar effect to calling `ChangeTracker.Entries()`, but by producing an event when something changes. The `ChangeTracker` events are useful for features such as logging changes or triggering actions when a specific entity type’s `State` changes.

The `Tracked` event, which is simpler, is triggered when an entity class is first tracked and tells you whether it came from a query via its `FromQuery` property. That event could occur when you execute an EF Core query (without the `AsNoTracking` method) or start tracking an entity class via an `Add` or `Attach` method.

````c#
// Example of a ChangeTracker.Tracked event and what it contains
var logs = new List<EntityTrackedEventArgs>(); // Holds a log of any tracked events
context.ChangeTracker.Tracked += delegate(
    object sender, EntityTrackedEventArgs args) // You register your event handler to the ChangeTracker.Tracked event.
{
    logs.Add(args); // This event handler simply logs the EntityTrackedEventArgs.
};

//ATTEMPT
var entity = new MyEntity { MyString = "Test" }; // Creates an entity class
context.Add(entity); // Adds that entity class to context

//VERIFY
logs.Count.ShouldEqual(1);
logs.Single().FromQuery.ShouldBeFalse(); // The entity wasn’t tracking during a query.
logs.Single().Entry.Entity.ShouldEqual(entity); // You can access the entity that triggered the event.
logs.Single().Entry.State
    .ShouldEqual(EntityState.Added); // You can also get the current State of that entity.
````

For a `Tracked` event, you get the `FromQuery` flag, which is `true` if the query was tracked during a query. The `Entry` property gives you information about the entity.

One thing to note in this example is that the `context.Add(entity)` method triggers an `Tracked` event but doesn’t trigger a `StateChanges` event. If you want to detect a newly added entity class, you can do so only via the `Tracked` event.

The `StateChanges` event is similar but contains different information. The event contains the entity’s `State` before `SaveChanges` was called in the property called `OldState` and the entity’s `State` after `SaveChanges` was called in the property called `NewState`.

````c#
// Example of a ChangeTracker.StateChanges event and what it contains
var logs = new List<EntityStateChangedEventArgs>(); // Holds a log of any StateChanged event
context.ChangeTracker.StateChanged += delegate (object sender, EntityStateChangedEventArgs args) // You register your event handler to the ChangeTracker.StateChanged event.
{
    logs.Add(args); // This event handler simply logs the EntityTrackedEventArgs.
};

//ATTEMPT
var entity = new MyEntity { MyString = "Test" }; // Creates an entity class
context.Add(entity); // Adds that entity class to context
context.SaveChanges(); // SaveChanges will change the State to Unchanged after the database update.

//VERIFY
logs.Count.ShouldEqual(1); // There is one event.
logs.Single().OldState.ShouldEqual(EntityState.Added); // The State before the change was Added
logs.Single().NewState.ShouldEqual(EntityState.Unchanged); // The State after the change is Unchanged
logs.Single().Entry.Entity.ShouldEqual(entity); // You get access to the entity data via the Entry property.
````

Now that you have seen the two `ChangeTracker` events, let’s use them for logging changes to some other form of storage.

````c#
// Class holding the code to turn ChangeTracker events into logs
public class ChangeTrackerEventHandler // This class is used in your DbContext to log changes.
{
    private readonly ILogger _logger;
    
    public ChangeTrackerEventHandler(DbContext context, ILogger logger)
    {
        _logger = logger; // You will log to ILogger.
        context.ChangeTracker.Tracked += TrackedHandler; // Adds a Tracked event handler
        context.ChangeTracker.StateChanged += StateChangeHandler; // Adds a StateChanged event handler
    }

    private void TrackedHandler(object sender, EntityTrackedEventArgs args) // Handles Tracked events
    {
        if (args.FromQuery) // We do not want to log entities that are read in.
        {
            return;
        }

        var message = $"Entity: {NameAndPk(args.Entry)}. " + $"Was {args.Entry.State}";
        _logger.LogInformation(message); // Forms a useful message on Add or Attach
    }

    // The StateChanged event handler logs any changes.
    private void StateChangeHandler(object sender, EntityStateChangedEventArgs args)
    {
        var message = $"Entity: {NameAndPk(args.Entry)}. " + $"Was {args.OldState} and went to {args.NewState}";
        _logger.LogInformation(message);
    }
}
````

````c#
// Adding the ChangeTrackerEventHandler to your application DbContext
public class Chapter11DbContext : DbContext // Your application DbContext that you want to log changes from
{
    private ChangeTrackerEventHandler _trackerEventHandler; // You need an instance of the event handler class while the DbContext exists.
    public Chapter11DbContext(DbContextOptions<Chapter11DbContext> options, ILogger logger = null) : base(options)
    {
        if (logger != null) // If an ILogger is available, you register the handlers.
        {
            _trackerEventHandler = new ChangeTrackerEventHandler(this, logger); // Creates the event handler class, which registers the event handlers
        }
    }
    //… rest of code left out
}
````

Logged messages are rather simple, but you could easily expand these messages to detail what properties have been modified, include the `UserId` of the user who changed things, and so on.

````c#
// Example output of ChangeTrackerEventHandler event logging
Entity: MyEntity {Id: -2147482647}. Was Added // Code that triggered that event: context.Add(new MyEntity)
Entity: MyEntity {Id: 1}. Was Added and went to Unchanged // Code that triggered that event: context.SaveChanges()
Entity: MyEntity {Id: 1}. Was Unchanged and went to Modified // Code that triggered that event: entity.MyString = "New string" + DetectChanges
Entity: MyEntity {Id: 1}. Was Modified and went to Unchanged // Code that triggered that event: context.SaveChanges()
````



#### Triggering events when SaveChanges/SaveChangesAsync is called

EF Core introduced `SavingChanges`, `SavedChanges`, and `SaveChangesFailed` events, which are called before the data is saved to the database, after the data has been successfully saved to the database, and if the save to the database failed, respectively. These events allow you to tap into what is happening in the `SaveChanges` and `SaveChangesAsync` methods. You could use these events to log what was written to the database or alert someone if there was a certain exception inside `SaveChanges` or `SaveChangesAsync`.

To use these events, you need to subscribe to the `SavingChanges` and `SavedChanges` events.

````c#
// How to subscribe to the SavingChanges/SavedChanges events
context.SavingChanges += // This event will trigger when SaveChanges is called but before it updates the database.
    delegate(object dbContext, 
             SavingChangesEventArgs args) // The SavingChangesEventArgs contains the SaveChanges Boolean parameter acceptAllChangesOnSuccess.
{
    var trackedEntities =
        ((DbContext)dbContext) // The first parameter is the instance of the DbContext, but you need to cast the object to use it.
        .ChangeTracker.Entries();
    
    //… your code goes here
};

context.SavedChanges += // This event will trigger when SaveChanges successfully updates the database.
    delegate(object dbContext,
             SavedChangesEventArgs args) // The SavedChangesEventArgs contains the count of entities that were saved to the database.
{
    //… your code goes here
};

context.SaveChangesFailed += // This event will trigger when SaveChanges has an exception during an update to the database.
    delegate(object dbContext,
             SaveChangesFailedEventArgs args) // The SavingChangesEventArgs contains the exception that happened during the update to the database.
{
    //… your code goes here
};
````

To use these events, you need to know a few things about them:

* Like all C# events, the subscription to these events lasts only as long as the instance of the DbContext exists.
* The events are triggered by both the `SaveChanges` and `SaveChangesAsync` methods.
* The `SavingChanges` event is called before the `ChangeTracker.DetectChanges` method is called, so if you want to implement the code to update entities by using their `State`, you need to call the `ChangeTracker.DetectChanges` method first. This approach isn’t a good idea, however, because `DetectChanges` would be called twice, which could cause a performance issue.



#### EF Core interceptors

EF Core introduced interceptors that enable you intercept, modify, and/or suppress EF Core operations, including low-level database operations, such as executing a command, as well as higher-level operations, such as calls to `SaveChanges`. These interceptors have some powerful features, such as altering commands being sent to the database.



### Using SQL commands in an EF Core application

EF Core’s SQL commands are designed to detect SQL injection attacks. EF Core provides two types of SQL commands:

* Methods ending in Raw, such as `FromSqlRaw`. In these commands, you provide separate parameters, and those parameters are checked.
* Methods ending in `Interpolated`, such as `FromSqlInterpolated`. The string parameter provided to these methods used string interpolation with the parameters in the string, such as `$"SELECT * FROM Books WHERE BookId = {myKey}"`. EF Core can check each parameter within the interpolated string type.

If you build an interpolated string outside the command - such as `var badSQL = $"SELECT … WHERE BookId = {myKey}"` - and then use it in a command like `FromSqlRaw(badSQL)`, EF Core can’t check SQL injection attacks. You should use `FromSqlRaw` with parameters or `FromSqlInterpolated` with parameters embedded in a string interpolation.

The groups of SQL commands that are c* overed are

* `FromSqlRaw`/`FromSqlInterpolated` sync/async methods, which allow you to use a raw SQL command in an EF Core query
* `ExecuteSqlRaw`/`ExecuteSqlInterpolated` sync/async methods, which execute a nonquery command
* `AsSqlQuery` Fluent API method, which maps an entity class to an SQL query
* `Reload`/`ReloadAsync` command, used to refresh an EF Core-loaded entity that has been changed by an `ExecuteSql…` method
* EF Core’s `GetDbConnection` method, which provides low-level database access libraries to access the database directly



#### FromSqlRaw/FromSqlInterpolated: Using SQL in an EF Core query

The `FromSqlRaw`/`FromSqlInterpolated` methods allow you to add raw SQL commands to a standard EF Core query, including commands that you wouldn’t be able to call from EF Core, such as stored procedures.



````c#
// Using a FromSqlInterpolated method to call an SQL stored procedure
int filterBy = 5;
var books = context.Books // You start the query in the normal way, with the DbSet<T> you want to read.
    .FromSqlInterpolated( // The FromSqlInterpolated method allows you to insert an SQL command.
    	$"EXECUTE dbo.FilterOnReviewRank @RankFilter = {filterBy}") // Uses C#6’s string interpolation feature to provide the parameter
    .IgnoreQueryFilters() // You need to remove any query filters; otherwise, the SQL won’t be valid.
    .ToList();
````



There are a few rules about an SQL query:

* The SQL query must return data for all properties of the entity type.
* The column names in the result set must match the column names that properties are mapped to.
* The SQL query can't contain related data, but you can add the Include method to load related navigational properties.

You can add other EF Core commands after the SQL command, such as `Include`, `Where`, and `OrderBy`.



````c#
// Example of adding extra EF Core commands to the end of an SQL query
double minStars = 4;
var books = context.Books
    .FromSqlRaw(
    	"SELECT * FROM Books b WHERE " +
    		"(SELECT AVG(CAST([NumStars] AS float)) " + // The SQL calculates the average votes and uses it in an SQL WHERE.
    		"FROM dbo.Review AS r " +
    		"WHERE b.BookId = r.BookId) >= {0}", minStars) // In this case, you use the normal sql parameter check and substitution method - {0}, {1}, {2}, and so on.
    .Include(r => r.Reviews) // The Include method works with the FromSql because you are not executing a store procedure.
    .AsNoTracking() // You can add other EF Core commands after the SQL command.
    .ToList();
````



If you’re using model-level query filters, the SQL you can write has limitations. `ORDER BY` won’t work, for example. The way around this problem is to apply the `IgnoreQueryFilters` method after the `Sql` command and re-create the model-level query filter in your SQL code.



#### ExecuteSqlRaw/ExecuteSqlInterpolated: Executing a nonquery command

In addition to putting raw SQL commands in a query, you can execute nonquery SQL commands via EF Core’s `ExecuteSqlRaw`/`ExecuteSqlInterpolated` methods. Typical commands are SQL `UPDATE` and `DELETE`, but any nonquery SQL command can be called.



````c#
// The ExecuteSqlCommand method executing an SQL UPDATE
var rowsAffected = context.Database // The ExecuteSqlRaw is in the context.Database property.
    .ExecuteSqlRaw( // The ExecuteSqlRaw will execute the SQL and return an integer, which in this case is the number of rows updated.
    	"UPDATE Books " + // The SQL command is a string, with places for the parameters to be inserted.
    	"SET Description = {0} " +
    	"WHERE BookId = {1}",
    	uniqueString, bookId); // Provides two parameters referred to in the command
````

The `ExecuteSqlRaw` method returns an integer, which is useful for checking that the command was executed in the way you expected.



#### AsSqlQuery Fluent API method: Mapping entity classes to queries

This feature allows you to hide your SQL code inside the application’s DbContext’s configuration, and developers can use this `DbSet<T>` property in queries as though it were a normal entity class mapped to an entity.



As an example, you will create an entity class called `BookSqlQuery` that returns three values for a `Book` entity class: `BookId`, `Title`, and the average votes for this `Book` in a property called `AverageVotes`.

````c#
// The BookSqlQuery class to map to an SQL query
public class BookSqlQuery
{
    [Key]
    public int BookId { get; set; }

    public string Title { get; set; }
    
    public double? AverageVotes { get; set; }
}
````

````c#
// Configuring the BookSqlQuery entity class to an SQL query
public class BookDbContext : DbContext
{
    //… other DbSets removed for clarity

    public DbSet<BookSqlQuery> BookSqlQueries { get; set; } // You add a DbSet<T> for the BookSqlQuery entity class to make querying easy.

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        //… other configrations removed for clarity

        modelBuilder.Entity<BookSqlQuery>().ToSqlQuery( // The ToSqlQuery method maps the entity class to an SQL query.
            @"SELECT BookId
            	,Title
            	,(SELECT AVG(CAST([r0].[NumStars] AS float))
            FROM Review AS r0
            WHERE t.BookId = r0.BookId) AS AverageVotes
            FROM Books AS t");
    }
}
````

You can add LINQ commands, such as `Where` and `OrderBy`, in the normal way, but the returned data follows the same rules as the `FromSqlRaw` and `FromSqlInterpolated` methods.



#### Reload: Used after ExecuteSql commands

If you have an entity loading (tracked), and you use an `ExecuteSqlRaw`/`ExecuteSqlInterpolated` method to change the data on the database, your tracked entity is out of date. That situation could cause you a problem later, because EF Core doesn’t know that the values have been changed. To fix this problem, EF Core has a method called `Reload`/`ReloadAsync`, which updates your entity by rereading the database.



````c#
// Using the Reload method to refresh the content of an existing entity
var entity = context.Books. // Loads a Book entity in the normal way
    Single(x => x.Title == "Quantum Networking");
var uniqueString = Guid.NewGuid().ToString();

// Uses ExecuteSqlRaw to change the Description column of that same Book entity
context.Database.ExecuteSqlRaw(
    "UPDATE Books " +
    "SET Description = {0} " +
    "WHERE BookId = {1}",
    uniqueString, entity.BookId);

context.Entry(entity).Reload(); // When calling the Reload method, EF Core rereads that entity to make sure that the local copy is up to date.
````



#### GetDbConnection: Running your own SQL commands

A low-level database libraries require a lot more code to be written but provide more-direct access to the database, so you can do almost anything you need to do.



````c#
// Obtaining a DbConnection from EF Core to run a Dapper SQL query
var connection = context.Database.GetDbConnection(); // Gets a DbConnection to the database, which the micro-ORM called Dapper can use
string query = "SELECT b.BookId, b.Title, " +
    "(SELECT AVG(CAST([NumStars] AS float)) " +
    "FROM dbo.Review AS r " +
    "WHERE b.BookId = r.BookId) AS AverageVotes " +
    "FROM Books b " +
    "WHERE b.BookId = @bookId"; // Creates the SQL query you want to execute

var bookDto = connection
    .Query<RawSqlDto>(query, new // Calls Dapper’s Query method with the type of the returned data
	{
        bookId = 4 // Provides parameters to Dapper to be added to the SQL command
    })
    .Single();
````



### Accessing information about the entity classes and database tables

EF Core provides two sources of information, one that emphasizes the entity classes and one that focuses more on the database:

* `context.Entry(entity).Metadata` - Has more than 20 properties and methods that provide information on the primary key, foreign key, and navigational properties
* `context.Model` - Has a set of properties and methods that provides a similar set of data to the `Metadata` property, but focuses more on the database tables, columns, constraints, indexes, and so on

Here are some examples of how you might use this information to automate certain services:

* Recursively visiting an entity class and its relationships so that you can apply some sort of action in each entity class, such as resetting its primary-key values
* Obtaining the settings on an entity class, such as its delete behavior
* Finding the table name and column names used by an entity class so that you can build raw SQL with the correct table and column names



#### Using context.Entry(entity).Metadata to reset primary keys

````c#
// Creating an Order with two LineItems ready to be copied
var books = context.SeedDatabaseFourBooks(); // For this test, add four books to use as test data.
var order = new Order // Create an Order with two LinItems that you want to copy.
{
    CustomerId = Guid.Empty, // Set CustomerId to the default value so that the query filter lets you read the order back.
    LineItems = new List<LineItem>
    {
        new LineItem
        {
            LineNum = 1, ChosenBook = books[0], NumBooks = 1
        },
        new LineItem
        {
            LineNum = 2, ChosenBook = books[1], NumBooks = 2
        },
    }
};
context.Add(order);
context.SaveChanges();
````

````c#
// Using metadata to visit each entity and reset its primary key
public class PkResetter
{
    private readonly DbContext _context;
    private readonly HashSet<object> _stopCircularLook; // Used to stop circular recursive steps

    public PkResetter(DbContext context)
    {
        _context = context;
        _stopCircularLook = new HashSet<object>();
    }

    public void ResetPksEntityAndRelationships(object entityToReset) // This method will recursively look at all the linked entities and reset their primary keys.
    {
        if (_stopCircularLook.Contains(entityToReset)) // If the method has already looked at this entity, the method exits.
        {
            return;
        }

        _stopCircularLook.Add(entityToReset); // Remembers that this entity has been visited by this method

        // Deals with an entity that isn’t known by your configuration
        var entry = _context.Entry(entityToReset);
        if (entry == null)
        {
            return;
        }

        var primaryKey = entry.Metadata.FindPrimaryKey(); // Gets the primary-key information for this entity
        if (primaryKey != null) // Resets every property used in the primary key to its default value
        {
            foreach (var primaryKeyProperty in primaryKey.Properties)
            {
                primaryKeyProperty.PropertyInfo
                    .SetValue(entityToReset, GetDefaultValue(primaryKeyProperty.PropertyInfo.PropertyType));
            }
        }

        foreach (var navigation in entry.Metadata.GetNavigations()) // Gets all the navigational properties for this entity
        {
            var navProp = navigation.PropertyInfo; // Gets a property that contains the navigation property

            var navValue = navProp.GetValue(entityToReset); // Gets the navigation property value
            if (navValue == null)
            {
                continue;
            }

            if (navigation.IsCollection) // If the navigation property is collection, visits every entity
            {
                foreach (var item in (IEnumerable)navValue) // Recursively visits each entity in the collection
                {
                    ResetPksEntityAndRelationships(item);
                }
            }
            else
            {
                ResetPksEntityAndRelationships(navValue); // If a singleton, visits that entity
        }
    }
}
````

* *Find the entity’s primary key* - `entry.Metadata.FindPrimaryKey()`
* *Get the primary key’s properties* - `primaryKeyProperty.PropertyInfo`
* *Find the entity’s navigational relationships* - `Metadata.GetNavigations()`
* *Get a navigational relationship’s property* - `navigation.PropertyInfo`
* *Checking whether the navigational property is a collection* - `navigation.IsCollection`



#### Using context.Model to get database information

The `context.Model` property gives you access to the `Model` of the database that EF Core builds on first use of an application’s DbContext. The `Model` contains some data similar to `context.Entry(entity).Metadata`, but it also has specific information of the database schema. Therefore, if you want to do anything with the database side, `context.Model` is the right information source to use.

If you deleted a group of dependent entities via EF Core, you would typically read in all the entities to delete, and EF Core would delete each entity with a separate SQL command. The method in the following listing produces a single SQL command that deletes all the dependent entities in one SQL command without the need to read them in. This process, therefore, is much quicker than EF Core, especially on large collections.

````c#
// Using context.Model to build a quicker dependent delete
public string BuildDeleteEntitySql<TEntity>(DbContext context, string foreignKeyName) where TEntity : class // This method provides a quick way to delete all the entities linked to a principal entity.
{
    var entityType = context.Model.FindEntityType(typeof(TEntity)); // Gets the Model information for the given type, or null if the type isn’t mapped to the database
    var fkProperty = entityType?.GetForeignKeys()
        .SingleOrDefault(x => x.Properties.Count == 1
                         && x.Properties.Single().Name == foreignKeyName)
        ?.Properties.Single(); // Looks for a foreign key with a single property with the given name

    if (fkProperty == null) // If any of those things doesn’t work, the code throws an exception.
    {
        throw new ArgumentException($"Something wrong!");
    }

    var fullTableName = entityType.GetSchema() == null
        ? entityType.GetTableName()
        : $"{entityType.GetSchema()}.{entityType.GetTableName()}"; // Forms the full table name, with a schema if required

    return $"DELETE FROM {fullTableName} " +
        $"WHERE {fkProperty.GetColumnName()}" // Forms the main part of the SQL code
        + " = {0}"; // Adds a parameter that the ExecuteSqlRaw can check
}
````

Having found the right entity/table and checked that the foreign key name matches, you can build the SQL. As the listing shows, you have access to the table’s name and schema, plus the column name of the foreign key. The following code snippet shows the output of the `BuildDeleteEntitySql` method with a `Review` entity class for the `TEntity` and a foreign-key name of `BookId`:

````c#
DELETE FROM Review WHERE BookId = {0}
````

The SQL command is applied to the database by calling the `ExecuteSqlRaw` method, with the SQL string as the first parameter and the foreign-key value as the second parameter.



### Dynamically changing the DbContext’s connection string

It provides a method called `SetConnectionString` that allows you to change the connection string at any time so that you can change the database you are accessing at any time.



![](./diagrams/images/11_03_dynamically_changing_connectioon_string.png)



EF Core made one other important change: the connection string can be `null` when you first create the application’s DbContext. The connection string can be `null` until you need to access the database. This feature is useful because on startup, there would be no tenant information, so the connection string would be `null`.



### Handling database connection problems

With relational database servers, a database access can fail because the connection times out or certain transient errors occur. EF Core has an execution strategy feature that allows you to define what should happen when a timeout occurs, how many timeouts are allowed, and so on.



````c#
// Setting up a DbContext with the standard SQL execution strategy
var connection = @"Server=(localdb)\mssqllocaldb;Database=…";
var optionsBuilder =
    new DbContextOptionsBuilder<EfCoreContext>();

optionsBuilder.UseSqlServer(connection,
	option => option.EnableRetryOnFailure());
var options = optionsBuilder.Options;

using (var context = new EfCoreContext(options))
{
	… normal code to use the context
````

Normal EF Core queries or `SaveChanges` calls will automatically be retried without your doing anything. Each query and each call to `SaveChanges` is retried as a unit if a transient failure occurs. But database transactions need a little more work.



#### Handling database transactions with EF Core’s execution strategy

The execution strategy works by rolling back the whole transaction if a transient failure occurs and then replaying each operation in the transaction; each query and each call to `SaveChanges` is retried as a unit. For all the operations in the transaction to be retried, the execution strategy must be in control of the transaction code.


````c#
// Writing transactions when you’ve configured an execution strategy
var connection = @"Server=(localdb)\mssqllocaldb;Database=…";
var optionsBuilder =
    new DbContextOptionsBuilder<EfCoreContext>();

optionsBuilder.UseSqlServer(connection,
	option => option.EnableRetryOnFailure()); // Configures the database to use the SQL execution strategy, so you have to handle transactions differently

using (var context = new Chapter09DbContext(options))
{
    var strategy = context.Database
        .CreateExecutionStrategy(); // Creates an IExecutionStrategy instance, which uses the execution strategy you configured the DbContext with
    strategy.Execute(() => // The important thing is to make the whole transaction code into an Action method it can call.
	{
        try
        {
            using (var transaction = context
                   .Database.BeginTransaction()) // The rest of the transaction setup and running your code are the same.
            {
                context.Add(new MyEntity());
                context.SaveChanges();
                context.Add(new MyEntity());
                context.SaveChanges();
                transaction.Commit();
            }
        }
        catch (Exception e)
        {
            //Error handling to go here
            throw;
        }
    });
}
````



#### Altering or writing your own execution strategy

If there’s an existing execution strategy for your database provider (such as SQL Server), you can change some options, such as the number of retries or the SQL errors to be retried.

If you want to write your own execution strategy, you need to implement a class that inherits the interface `IExecutionStrategy`.


````c#
// Configuring your own execution strategy into your DbContext
var connection = this.GetUniqueDatabaseConnectionString();
var optionsBuilder =
    new DbContextOptionsBuilder<Chapter09DbContext>();

optionsBuilder.UseSqlServer(connection,
	options => options.ExecutionStrategy(
		p => new MyExecutionStrategy()));

using (var context = new Chapter09DbContext(optionsBuilder.Options))
{
    … etc.
````



### Summary

* You can use EF Core’s entity `State` property, with a little help from a perproperty `IsModified` flag, to define what will happen to the data when you call `SaveChanges`.
* You can affect the `State` of an entity and its relationships in several ways. You can use the `DbContext`’s methods `Add`, `Remove`, `Update`, `Attach`, and `TrackGraph`; set the `State` directly; and track modifications.
* The `DbContext`’s `ChangeTracker` property provides several ways to detect the `State` of all the entities that have changed. These techniques are useful for marking entities with the date when an entity was created or last updated, or logging every `State` change for any of the tracked entities.
* The `Database` property has methods that allow you to use raw SQL command strings in your database accesses.
* You can access information about the entities and their relationships via the `Entry(entity).Metadata` and the database structure via the `Model` property.
* EF Core contains a system that allows you to provide a retry capability. This system can improve reliability by retrying accesses if there are connection or transient errors in your database.





## 12. Using entity events to solve business problems

* Understanding the types of events that work well with EF Core
* Using domain events to trigger extra business rules
* Using integration events to synchronize two parts of your application
* Implementing an Event Runner and then improving it



### Using events to solve business problems

#### Example of using domain events

The client’s company sells bespoke constructions in the United States, and every project starts with a quote to send to the client. The construction could be anywhere in the United States, and the state where the work is done defines the sales tax. As a result, the sales tax had to be recalculated when any of the following things happened:

* *A new quote was created.* By default, a new quote doesn’t have a location, so the business rule was to give it the highest sales tax until the location was specified.
* *The job location was set or changed.* The sales tax had to be recalculated, and it was the sales team’s job to select a location from a list of known locations.
* *A location’s address changed.* All the quotes linked to that location had to be recalculated to make sure that the sales tax was correct.

A change in the `Location` entity class created a domain event to trigger an event handler that recalculated the sales tax for a quote (or quotes). Each domain event needed a slightly different piece of business logic, plus a common service to calculate the tax. An example of what might happen if the address of a location changes:



![](./diagrams/images/12_01_example_of_using_domain_events.png)



#### Example of integration events

Precalculate the data you need to show to the user and store it in another database used only for displaying data to the user. This approach improves read performance and scalability.

The normal SQL commands for the Book App, for example, calculate the average star rating of a book by dynamically calculating the average across all the `Book`’s `Reviews`. That technique works fine for a small number of `Book`s and `Review`s, but with large numbers, sorting by average review ratings can be slow. You will use a Query Responsibility Segregation (CQRS) database pattern to store the precalculated data in a separate, read-side database.



![](./diagrams/images/12_02_example_of_integration_events.png)



### Defining where domain events and integration events are useful

DDD talks a lot about a bounded context, which represents a defined part of software where particular terms, definitions, and rules apply in a consistent way. A bounded context is about applying the Separation of Concerns (SoC) principle at the macro level. You can categorize the two event types as follows:

* The sales-tax example is referred to as a domain event because it is working exclusively within a single bounded context.
* The CQRS example is referred to as an integration event because it crosses from one bounded context to another.



### Where might you use events with EF Core?

The answer is best provided by some examples:

* Setting or changing an `Address` triggers a recalculation of the sales-tax code of a `Quote`.
* Creating an `Order` triggers a check on reordering `Stock`.
* Updating a `Book` triggers an update of that `Book`’s `Projection` on another database.
* Receiving a `Payment` that pays off the debt triggers the closing of the `Account`.
* Sending a `Message` to an `external service`.



#### Pro: Follows the SoC design principle

The event systems already described provide a way to run separate business rules on a change in an entity class. In the location-change/sales-tax example, the two entities are linked in a nonobvious way; changing the location of a job causes a recalculation of the sales tax for any linked quotes. When you apply the SoC principle, these two business rules should be separated.

You could create some business logic to handle both business rules, but doing so would complicate a simple update of properties in an address. By triggering an event if the `State`/`County` properties are changed, you can keep the simple address update and let the event handle the second part.



#### Pro: Makes database updates robust

The design of the code that handles domain events is such that the original change that triggers the event and the changes applied to entity classes via the called event handler are saved in the same transaction.



![](./diagrams/images/12_03_makes_database_updates_robust.png)



If the integration event fails, the database update will be rolled back, ensuring that the local database and the external service and different database are in step.



#### Con: Makes your application more complex

One of the downsides of using events is that your code is going to be more complicated. You will create your events, add the events to your entity classes, and write your event handlers, which requires more code than building services for your business logic.

But the trade-off of events that need more code is that the two business logic parts are decoupled. Changes to the address become a simple update, for example, while the event makes sure that the tax code is recalculated. This decoupling reduces the business complexity that the developer has to deal with.



#### Con: Makes following the flow of the code more difficult

It can be hard to understand code that you didn’t write or wrote a while back. You can do the same thing when you use events, but that technique does add one more level of indirection before you get to the code. For the sales-tax-change example, you would need to click the `LocationChangedEvent` class to find the `LocationChangedEventHandler` that has the business code you’re looking for - only one more step, but a step you don’t need if you don’t use events.



### Implementing a domain event system with EF Core

Two `Quote`s are linked to location, so their `SalesTax` property should be updated to the correct sales tax at that location.
To implement this domain event system, add the following code to your application:

1. You create some domain events classes to be triggered.
2. Add code to the entity classes to hold the domain events.
3. Alter the code in the entity class to detect a change on which you want to trigger an event.
4. Create some event handlers that are matched to the events. These event handlers may alter the calling entity class or access the database or business logic to execute the business rules it is designed to handle.
5. Build an Event Runner that finds and runs the correct event handler that matches each found event.
6. Add the Event Runner to the DbContext, and override the `SaveChanges` (and `SaveChangesAsync`) methods in your application’s DbContext.
7. When the Event Runner has finished, run the base `SaveChanges`, which updates the database with the original changes and any further changes applied by the event handlers.
8. Register the Event Runner and all the event handlers.



![](./diagrams/images/12_04_implementing_domain_event_system.png)



#### Create some domain events classes to be triggered

There are two parts to creating an event. First, an event must have an interface that allows the Event Runner to refer to it. This interface can be empty, representing an event.

Each application event contains data that is specific to the business needs. The following listing shows the `LocationChangedEvent` class, which needs only the `Location` entity class.

````c#
// The LocationChangedEvent class, with data that the event handler needs
public class LocationChangedEvent : IDomainEvent // The event class must inherit the IDomainEvent. The Event Runner uses the IDomainEvent to represent every domain event.
{
    public LocationChangedEvent(Location location)
    {
        Location = location;
    }

    public Location Location { get; } // The event handler needs Location to do the Quote updates.
}
````

Each event should send over the data that the event handler needs to do its job. Then it is the event handler’s job to run some business logic, using the data provided by the event.



#### Add code to the entity classes to hold the domain events

The entity class must hold a series of events. These events aren’t written to the database but are there for the Event Runner to read via a method.



````c#
// The class that entity classes inherit to create events
public class AddEventsToEntity : IEntityEvents // The IEntityEvents defines the GetEventsThenClear method for the Event Runner.
{
    private readonly List<IDomainEvent> _domainEvents = new List<IDomainEvent>(); // The list of IDomainEvent events is stored in a field.

    // The AddEvent is used to add new events to the _domainEvents list.
    public void AddEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    // This method is called by the Event Runner to get the events and then clear the list.
    public ICollection<IDomainEvent> GetEventsThenClear()
    {
        var eventsCopy = _domainEvents.ToList();
        _domainEvents.Clear();
        return eventsCopy;
    }
}
````

The entity class can call the `AddEvent` method, and the Event Runner can get the domain events via the `GetEventsThenClear` method. Getting the domain events also clears the events in the entity class, because these messages will cause an event handler to be executed, and you want the event handler to run only once per domain event. Domain events are messages passed to the Event Runner via the entity classes, and you want a message to be used only once.



#### Alter the entity class to detect a change to trigger an event on

An event is normally something being changed or something reaching a certain level. EF Core allows you to use backing fields, which make it easy to capture changes to scalar properties.



````c#
// The Location entity class creates a domain event if the State is changed
public class Location : AddEventsToEntity // This entity class inherits the AddEventsToEntity to gain the ability to use events.
{
    // These normal properties don’t generate events when they are changed.
    public int LocationId { get; set; }
    public string Name { get; set; }

    private string _state; // The backing field contains the real value of the data.

    public string State // The setter is changed to send a LocationChangedEvent if the State value changes.
    {
        get => _state;
        set // This code will add a LocationChangedEvent to the entity class if the State value changes.
        {
            if (value != _state)
            {
                AddEvent(new LocationChangedEvent(this));
            }
            _state = value;
        }
    }
}
````



Collection navigational properties are a little harder to check for changes, but DDD-styled entity classes make this check much simpler.



#### Create event handlers that are matched to the domain events

Event handlers are key to using events in your application. Each event handler contains some business logic that needs to be run when the specific event is found.



````c#
// The event handler updates the sales tax on Quotes linked to this Location
public class LocationChangedEventHandler : // This class must be registered as a service via DI.
IEventHandler<LocationChangedEvent> // Every event handler must have the interface IEventHandler<T>, where T is the event class type.
{
    // This specific event handler needs two classes registered with DI.
    private readonly DomainEventsDbContext _context;
    private readonly ICalcSalesTaxService _taxLookupService;

    // The Event Runner will use DI to get an instance of this class and will fill in the constructor parameters.
    public LocationChangedEventHandler(DomainEventsDbContext context, ICalcSalesTaxService taxLookupService)
    {
        _context = context;
        _taxLookupService = taxLookupService;
    }

    public void HandleEvent(LocationChangedEvent domainEvent) // The method from the IEventHandler<T> that the Event Runner will execute
    {
        var salesTaxPercent = _taxLookupService
            .GetSalesTax(domainEvent.Location.State); // Uses another service to calculate the right sales tax

        // Sets the SalesTax on every Quote that is linked to this Location
        foreach (var quote in _context.Quotes.Where(
            x => x.WhereInstall == domainEvent.Location))
        {
            quote.SalesTaxPercent = salesTaxPercent;
        }
    }
}
````

The key point here is that the event handler is registered as a service so that the Event Runner can get an instance of the event handler class via dependency injection (DI). The event handler class has the same access to DI services that normal business logic does. In this case, the `LocationChangedEventHandler` injects the application’s DbContext and the `ICalcSalesTaxService` service.



#### Build an Event Runner that finds and runs the correct event handler

The Event Runner is the heart of the event system: its job is to match each event to an event handler and then invoke the event handler’s method, providing the event as a parameter. This process uses NET Core’s `ServiceProvider` to get an instance of the event handler, which allows the event handlers to access other services.



![](./diagrams/images/12_05_event_runner.png)



````c#
// The Event Runner that is called from inside the overridden SaveChanges
public class EventRunner : IEventRunner // The Event Runner needs an interface so that you can register it with the DI.
{
    // The Event Runner needs the ServiceProvider to get an instance of the event handlers.
    private readonly IServiceProvider _serviceProvider;

    public EventRunner(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void RunEvents(DbContext context)
    {
        var allEvents = context
            .ChangeTracker.Entries<IEntityEvents>()
            .SelectMany(x => x.Entity.GetEventsThenClear()); // Reads in all the events and clears the entity events to stop duplicate events

        foreach (var domainEvent in allEvents) // Loops through each event found
        {
            // Gets the interface type of the matching event handler
            var domainEventType = domainEvent.GetType();
            var eventHandleType = typeof(IEventHandler<>)
                .MakeGenericType(domainEventType);

            // Uses the DI provider to create an instance of the event handler and returns an error if one is not found
            var eventHandler = _serviceProvider.GetService(eventHandleType);
            if (eventHandler == null)
            {
                throw new InvalidOperationException($"Could not find an event handler");
            }

            // Creates the EventHandlerRunner that you need to run the event handler
            var handlerRunnerType = typeof(EventHandlerRunner<>)
                .MakeGenericType(domainEventType);
            var handlerRunner = ((EventHandlerRunner)
				Activator.CreateInstance(
                    handlerRunnerType, eventHandler));

            handlerRunner.HandleEvent(domainEvent); // Uses the EventHandlerRunner to run the event handler
        }
    }
}
````

The following listing shows the `EventHandlerRunner` and `EventHandlerRunner<T>` classes. You need these two classes because the definition of an event handler is generic, so you can’t call it directly. You get around this problem by creating a class that takes the generic event handler in its constructor and has a nongeneric method (the abstract class called `EventHandlerRunner`) that you can call.

````c#
// The EventHandlerRunner class that runs the generic-typed event handler
// By defining a nongeneric method, you can run the generic event handler.
internal abstract class EventHandlerRunner
{
    public abstract void HandleEvent(IDomainEvent domainEvent);
}

internal class EventHandlerRunner<T> : EventHandlerRunner // Uses the EventHandlerRunner<T> to define the type of the EventHandlerRunner
    where T : IDomainEvent
{
    // The EventHandlerRunner class is created with an instance of the event handler to run.
    private readonly IEventHandler<T> _handler;
    
    public EventHandlerRunner(IEventHandler<T> handler)
    {
        _handler = handler;
    }

    // Method that overrides the abstract class's HandleEvent method
    public override void HandleEvent(IDomainEvent domainEvent)
    {
        _handler.HandleEvent((T)domainEvent);
    }
}
````



#### Override SaveChanges and insert the Event Runner before SaveChanges is called

Next, you override `SaveChanges` and `SaveChangesAsync` so that the Event Runner is run before the base `SaveChanges` and `SaveChangesAsync` run. Any changes the event handlers make to entities are saved with the original changes that caused the events. This point is really important: both the changes made to entities by your nonevent code are saved with any changes made by your event handler code. If a problem occurs with the data being saved to the database (a concurrency exception was thrown, for example), neither of the changes would be written to the database, so the two types of entity changes - nonevent code changes and event-handler code changes - won’t become CQRS out of step.



````c#
// Your application’s DbContext with SaveChanges overridden
public class DomainEventsDbContext : DbContext
{
    private readonly IEventRunner _eventRunner; // Holds the Event Runner that is injected by DI via the class's constructor

    // The constructor now has a second parameter DI fills in with the Event Runner.
    public DomainEventsDbContext(DbContextOptions<DomainEventsDbContext> options, IEventRunner eventRunner = null)
        : base(options)
    {
        _eventRunner = eventRunner;
    }

    //… DbSet<T> left out

    public override int SaveChanges(bool acceptAllChangesOnSuccess) // You override SaveChanges so that you can run the Event Runner before the real SaveChanges.
    {
        _eventRunner?.RunEvents(this); // Runs the Event Runner
        return base.SaveChanges(acceptAllChangesOnSuccess); // Runs the base.SaveChanges
    }

    //… overridden SaveChangesAsync left out
}
````



#### Register the Event Runner and all the event handlers

The last part is registering the Event Runner and the event handlers with the DI provider. The Event Runner relies on the DI to provide an instance of your event handlers, using their interfaces; also, your application’s DbContext needs the Event Runner injected by DI into the `IEventRunner` parameter of its constructor. When Event Runner and the event handlers are registered, along with any services that the event handlers need (such as the sales tax calculator service), the Event Runner will work.



````c#
// Manually registering the Event Runner and event handlers In ASP.NET Core
public void ConfigureServices(IServiceCollection services) // You register interfaces/classes with the NET dependency injection provider—in this case, in a ASP.NET Core app.
{
    //… other registrations left out
    services.AddTransient<IEventRunner, EventRunner>(); // Registers the Event Runner, which will be injected into your application’s DbContext

    // Registers all your event handlers
    services.AddTransient<IEventHandler<LocationChangedEvent>, LocationChangedEventHandler>();
    services.AddTransient<IEventHandler<QuoteLocationChangedEvent>, QuoteLocationChangedEventHandler>();

    services.AddTransient<ICalcSalesTaxService, CalcSalesTaxService>(); // You need to register any services that your event handlers will use.
}
````

Although manual registration works, a better way is to automate finding and registering the event handlers.

````c#
services.RegisterEventRunnerAndHandlers(
    Assembly.GetAssembly(
        typeof(LocationChangedEventHandler)));
````



````c#
// Automatically registering the Event Runner and your event handlers
public static void RegisterEventRunnerAndHandlers(
    this IServiceCollection services, // The method needs the NET Core’s service collection to add to.
    params Assembly[] assembliesToScan) // You provide one or more assemblies to scan.
{
    services.AddTransient<IEventRunner, EventRunner>(); // Registers the Event Runner
    
    // Calls a method to find and register event handler in an assembly
    foreach (var assembly in assembliesToScan)
    {
        services.RegisterEventHandlers(assembly);
    }
}

// Finds and registers all the classes that have the IEventHandler<T> interface
private static void RegisterEventHandlers(
    this IServiceCollection services,
    Assembly assembly)
{
    var allGenericClasses = assembly.GetExportedTypes()
        .Where(y => y.IsClass && !y.IsAbstract
               && !y.IsGenericType && !y.IsNested); // Finds all the classes that could be an event handler in the assembly
    var classesWithIHandle =
        from classType in allGenericClasses
        let interfaceType = classType.GetInterfaces()
	        .SingleOrDefault(y =>
				y.IsGenericType &&
                y.GetGenericTypeDefinition() ==
					typeof(IEventHandler<>))
        where interfaceType != null
        select (interfaceType, classType); // Finds all the classes that have the IEventHandler<T> interface, plus the interface type

    // Registers each class with its interface
    foreach (var tuple in classesWithIHandle)
    {
        services.AddTransient(
            tuple.interfaceType, tuple.classType);
    }
}
````



### Implementing an integration event system with EF Core

Integration events are simpler to implement than domain events but harder to design because they work across bounded contexts.

The core tries to update a CQRS read-side database only if the SQL update succeeded, and it commits the SQL update only if the CQRS read-side database was successful; that way, the two databases contain the same data. You can generalize this example into two parts, both of which must work for the action to be successful:

* Don’t send the integration event if the database update didn’t work.
* Don’t commit the database update unless the integration event worked.

Suppose that you are building a new service that sends customers their orders of Lego bricks by courier on the same day. You must be sure that your warehouse has the items in stock and has a courier that can deliver the order immediately.



![](./diagrams/images/12_06_implementing_integration_event_system.png)



You have two options for detecting and handling your integration event in your application’s DbContext:

* You inject the service directly into your application’s DbContext, which works out for itself whether a specific event has happened by detecting the `State` of the entities. A second part is called only if the first method says that it needs to be called.
* You could use an approach similar to the Event Runner that you used for domain events, but a different event type is run within a transaction after the base `SaveChanges` is called.

In most cases, you won’t have many integration events, so the first option is quicker; it bypasses the event system you added to the entity for the domain events and does its own detection of the event. This approach is simple and keeps all the code together, but it can become cumbersome if you have multiple events to detect and process.

The second option is an expansion of the Event Runner and domain events, which uses a similar creation of an integration event when something changes in the entity. In this specific case, the code will create an integration event when a new `Order` is created.

Both options require an event handler. What goes in the event handler is the business logic needed to communicate with the system/code and to understand its responses. The first option was used in the Lego example, where the event handler detected the event itself. You need to add two sections of code to implement this example:

* Build a service that communicates with the warehouse.
* Override `SaveChanges` (and `SaveChangesAsync`) to add code to create the integration event and its feedback.



#### Building a service that communicates with the warehouse

In the Lego example, the design suggests that the website where customers place orders is separate from the warehouse, which means some form of communication, maybe via some RESTful API. In this case, you would build a class that communicates with the correct warehouse and returns either a success or a series of errors.



````c#
// The Warehouse event handler that both detects and handles the event
public class WarehouseEventHandler : IWarehouseEventHandler
{
    private Order _order;

    public bool NeedsCallToWarehouse(DbContext context) // This method detects the event and returns true if there is an Order to send to the warehouse.
    {
        var newOrders = context.ChangeTracker
            .Entries<Order>()
            .Where(x => x.State == EntityState.Added)
            .Select(x => x.Entity)
            .ToList(); // Obtains all the newly created Orders

        if (newOrders.Count > 1) // The business logic handles only one Order per SaveChanges call.
        {
            throw new Exception("Can only process one Order at a time");
        }

        if (!newOrders.Any()) // If there isn’t a new Order, returns false
        {
            return false;
        }

        _order = newOrders.Single();
        return true; // If there is an Order, retains it and returns true
    }

    public List<string> AllocateOrderAndDispatch() // This method will communicate with the warehouse and returns any errors the warehouse sends back.
    {
        var errors = new List<string>();

        // Adds the code to communicate with the warehouse
        //... code to communicate with warehouse

        return errors; // Returns a list of errors. If the list is empty, the code was successful.
    }
}
````



#### Overriding SaveChanges to handle the integration event

You are using an integration event implementation that detects the event itself, rather than adding an event to the entity class, so the code inside the overridden `SaveChanges` and `SaveChangesAsync` is specific to the integration event.



````c#
// DbContext with overridden SaveChanges and Warehouse event handler
public class IntegrationEventDbContext : DbContext
{
    private readonly IWarehouseEventHandler _warehouseEventHandler; // Holds the instance of the code that will communicate with the external warehouse

    public IntegrationEventDbContext(
        DbContextOptions<IntegrationEventDbContext> options,
        IWarehouseEventHandler warehouseEventHandler) // Injects the warehouse event handler via DI
        : base(options)
    {
        _warehouseEventHandler = warehouseEventHandler;
    }

    public DbSet<Order> Orders { get; set; }
    public DbSet<Product> Products { get; set; }

    public override int SaveChanges(bool acceptAllChangesOnSuccess) // Overrides SaveChanges to include the warehouse event handler
    {
        if (!_warehouseEventHandler.NeedsCallToWarehouse(this)) // If the event handler doesn’t detect an event, it does a normal SaveChanges.
        {
            return base.SaveChanges(acceptAllChangesOnSuccess);
        }

        using(var transaction = Database.BeginTransaction()) // There is an integration event, so a transaction is opened.
        {
            var result = base.SaveChanges(acceptAllChangesOnSuccess); // Calls the base SaveChange to save the Order
            var errors = _warehouseEventHandler
                .AllocateOrderAndDispatch(); // Calls the warehouse event handler that communicates with the warehouse

            if (errors.Any()) // If the warehouse returned errors, throws an OutOfStockException
            {
                throw new OutOfStockException(string.Join('.', errors));
            }

            transaction.Commit(); // If there were no errors, the Order is committed to the database.
            return result; // Returns the result of the SaveChanges
        }
    }
    
    //… overridden SaveChangesAsync left out
}
````



### Improving the domain event and integration event implementations

#### Generalizing events: Running before, during, and after the call to SaveChanges

Another event type might be useful - one that runs when `SaveChanges` or `SaveChangesAsync` has finished successfully. You could send an email when you are sure that an Order has been checked and successfully added to the database. That example uses three event types, which we can call *Before* (domain events), *During* (integration events), and *After* events.



![](./diagrams/images/12_07_generalizing_events.png)



To implement the Before, During, and After event system, you must add two more Event Runners: one called within a transaction to handle the integration events, and one after `SaveChanges`/`SaveChangesAsync` has finished successfully. You also need three event-handler interfaces - Before, During, and After - so that the correct event handler is run at the same time.



#### Adding support for async event handlers

In many of today’s multiuser applications, async methods will improve scalability, so you need to have async versions of the event handlers. Adding an async method requires an extra event handler interface for an async event handler version. Also, the Event Runner code must be altered to find an async version of the event handler when the `SaveChangesAsync` is called.



````c#
// The original RunEvents method updated to run async event handlers
public async Task RunEventsAsync(DbContext context) // The RunEvent becomes an async method, and its name is changed to RunEventAsync.
{
    var allEvents = context
        .ChangeTracker.Entries<IEntityEvents>()
        .SelectMany(x => x.Entity.GetEventsThenClear());

    foreach (var domainEvent in allEvents)
    {
        var domainEventType = domainEvent.GetType();
        var eventHandleType = typeof(IEventHandlerAsync<>) // The code is now looking for a handle with an async type.
            .MakeGenericType(domainEventType);

        var eventHandler = _serviceProvider.GetService(eventHandleType);
        if (eventHandler == null)
        {
            throw new InvalidOperationException("Could not find an event handler");
        }

        var handlerRunnerType = typeof(EventHandlerRunnerAsync<>) // Needs a async EventHandlerRunner to run the event handler
            .MakeGenericType(domainEventType);
        var handlerRunner = ((EventHandlerRunnerAsync) // Is cast to a async method
			Activator.CreateInstance(
                handlerRunnerType, eventHandler));

        await handlerRunner.HandleEventAsync(domainEvent); // Allows the code to run the async event handler
    }
}
````



#### Handling multiple event handers for the same event

You might define more than one event handler for an event. Your `LocationChangedEvent`, for example, might have one event handler to recalculate the tax code and another event handler to update the company’s map of ongoing projects. In the current implementations of the Event Runners, the .NET Core DI method `GetService` would throw an exception because it can return only one service. The solution is simple. Use the .NET Core DI method `GetServices` method and then loop through each event handler found:

````c#
var eventHandlers = _serviceProvider.GetServices(eventHandleType);
if (!eventHandlers.Any())
{
    throw new InvalidOperationException("Could not find an event handler");
}

foreach(var eventHandler in eventHandlers)
{
    //… use code from listing 12.5 that runs a single event handler
````



#### Handling event sagas in which one event kicks off another event

One event could cause a new event to be created. The `LocationChangedEvent` event updated the `SalesTax`, which, in turn, caused a `QuotePriceChangeEvent`. These updates are referred to as *event sagas* because the business logic consists of a series of steps that must be executed in a certain order for the business rule to be completed.

Handling event sagas requires you to add a looping arrangement that looks for events being created by other events.



````c#
// Adding looping on events to the RunEvents method in the Event Runner
public void RunEvents(DbContext context)
{
    bool shouldRunAgain; // Controls whether the code should loop around again to see whether there are any new events
    int loopCount = 1; // Counts how many times the Event Runner loops around to check for more events
    do // This do/while code keeps looping while shouldRunAgain is true.
    {
        var allEvents = context
            .ChangeTracker.Entries<IEntityEvents>()
            .SelectMany(x => x.Entity.GetEventsThenClear());

        shouldRunAgain = false; // shouldRunAgain is set to false. If there are no events, it will exit the do/while loop.
        foreach (var domainEvent in allEvents)
        {
            shouldRunAgain = true; // There are events, so shouldRunAgain is set to true.

            var domainEventType = domainEvent.GetType();
            var eventHandleType = typeof(IEventHandler<>)
                .MakeGenericType(domainEventType);

            var eventHandler = _serviceProvider.GetService(eventHandleType);
            if (eventHandler == null)
            {
                throw new InvalidOperationException("Could not find an event handler");
            }

            var handlerRunnerType = typeof(EventHandlerRunner<>)
                .MakeGenericType(domainEventType);
            var handlerRunner = ((EventHandlerRunner)
				Activator.CreateInstance(
                    handlerRunnerType, eventHandler));

            handlerRunner.HandleEvent(domainEvent);
        }

        // This check catches an event handler that triggers a circular set of events.
        if (loopCount++ > 10)
        {
            throw new Exception("Looped to many times");
        }
    } while (shouldRunAgain); // Stops looping when there are no events to handle
}
````



### Summary

* A domain event class carries a message that is held inside an entity class. The domain event defines the type of event and carries event-specific data, such as what data has changed.
* Event handlers contain business logic that is specific to a domain event. Their job is to run the business logic, using the domain event data to guide what it does.
* The domain events version of the `SaveChanges` and `SaveChangesAsync` methods captures all the domain events in the tracked-entities classes and then runs matching event handlers.
* The integration events versions of the `SaveChanges` and `SaveChangesAsync` methods use a transaction to ensure that both the database and integration event handler succeed before the database is updated. This requirement allows you to synchronize two separate parts of your application.





## 13. Domain-Driven Design and other architectural approaches

* Three architectural approaches
* The differences between normal and DDD-styled entity classes
* Eight ways you can apply DDD to your entity classes
* Three ways to handle performance problems when using DDD



### The Book App’s evolving architecture



![](./diagrams/images/13_01_the_book_app_architecture.png)



#### Building a modular monolith to enforce the SoC principles



![](./diagrams/images/13_02_moduler.png)



#### Using DDD principles both architecturally and on the entity classes

DDD details many approaches for defining, building, and managing software applications.

* The main rule is that a DDD entity is in total control of the data in that entity: all the properties are made read-only, and there are constructors/methods to create/update the entities’ data. Giving the entity total control of its data makes your entity classes much more powerful; each entity class has clearly defined constructors/methods for the developer to use.
* DDD says that entities, which contain both data and domain (business) logic, should not know anything about how the entities are persisted to a database.
* DDD talks about bounded contexts, which separate your application into distinct parts. The idea is to create bounded contexts that are separate so that they are easier to understand, and then set up clearly defined communication between the bounded contexts.



### Altering the Book App entities to follow the DDD approach

Here are the steps in the process of changing your code to the DDD approach:

* Changing the properties in the `Book` entity to read-only
* Updating the `Book` entity properties via methods in the entity class
* Controlling how the `Book` entity is created
* Understanding the differences between an entity and a value object
* Minimizing the relationships between entity classes
* Grouping entity classes (DDD name: *aggregates*)
* Deciding when the business logic shouldn’t be run inside an entity
* Applying DDD’s bounded context to your application’s DbContext



#### Changing the properties in the Book entity to read-only

DDD says that the entity class is in charge of the data it contains; therefore, it must control how the data is created or changed. For the entity class to control its data, you make all the properties in the entity read-only. After that, a developer can set the data in the entity class only via the class’s constructor or via the methods in the entity class. The entity can ensure that it is always in a valid state.



![](./diagrams/images/13_03_non_ddd_vs_ddd.png)



````c#
// Making the Book entity class’s properties read-only
public class Book
{
    // Noncollection properties have their setter set to private.
    public int BookId { get; private set; }
    public string Title { get; private set; }
    //… other non-collection properties left out

    // A collection is stored in a backing field.
    private HashSet<Review> _reviews;
    // The property collection returns the appropriate backing fields as readonly collections.
    public IReadOnlyCollection<Review> Reviews => _reviews?.ToList();

    private HashSet<BookAuthor> _authorsLink;
    public IReadOnlyCollection<BookAuthor> AuthorsLink => _authorsLink?.ToList();
    //… other collection properties left out
}
````



If you are using AutoMapper, it will ignore the private access scope on your setter and update the property, which is not what you want to happen when using DDD. To stop this update, you need to add the `IgnoreAllPropertiesWithAnInaccessibleSetter` method after the call to AutoMapper’s `CreateMap<TSource,TDestination>` method.



#### Updating the Book entity properties via methods in the entity class

With all the properties converted to read-only, you need another way to update the data inside an entity. The answer is to add methods inside the entity class that can update the properties:

* You can use an entity like a black box. The access methods and constructors are its API: it’s up to the entity to make sure that the data inside the entity is always in a valid state.
* You can put your business rules in the access method. The method can return errors to users so that they can fix the problem and retry, or for a software problem, you can throw an exception.
* If there isn’t a method to update a specific property, you know that you’re not allowed to change that property.

Turning the rules into an access method means that no one can get them wrong. Also, the rules are in one place, so they’re easy to change if necessary. These access methods are some of DDD’s most powerful techniques.



````c#
// Example of a DDD access method that contains business logic/validation
public IStatusGeneric AddPromotion( // The AddPromotion returns a status. If that status has errors, the promotion is not applied.
    decimal actualPrice, string promotionalText) // The parameters came from the input.
{
    var status = new StatusGenericHandler(); // Creates a status that is successful unless errors are added to it
    if (string.IsNullOrWhiteSpace(promotionalText)) // Ensures that the promotionalText has some text in it
    {
        // The AddError method adds an error and returns immediately.
        return status.AddError("You must provide text to go with the promotion.", nameof(PromotionalText)); // The error contains a userfriendly message and the name of the property that has the error.
    }

    // If no errors occur, the ActualPrice and PromotionalText are updated.
    ActualPrice = actualPrice;
    PromotionalText = promotionalText;

    return status; // The status, which is successful, is returned.
}

public void RemovePromotion() // This removes an existing promotion. Because there are no possible errors it returns void.
{
    // Removes the promotion by resetting the ActualPrice and the PromotionalText
    ActualPrice = OrgPrice;
    PromotionalText = null;
}
````



#### Controlling how the Book entity is created

In line with the DDD approach, in which the entity controls the setting of data in it, you need to think about the creation of an entity. You need to provide at least one constructor or a static create factory method for a developer to use to create a new instance of the entity.



````c#
// The static create factory to create a valid Book or return the errors
private Book() { } // Creating a private constructor means that people can’t create the entity via a constructor.

public static IStatusGeneric<Book> CreateBook( // The static CreateBook method returns a status with a valid Book (if there are no errors).
    string title, DateTime publishedOn,
    decimal price,
    ICollection<Author> authors) // These parameters are all that are needed to create a valid Book.
{
    var status = new StatusGenericHandler<Book>(); // Creates a status that can return a result—in this case, a Book
    if (string.IsNullOrWhiteSpace(title)) // Adds an error. Note that it doesn’t return immediately so that other errors can be added.
    {
        status.AddError("The book title cannot be empty.");
    }

    var book = new Book
    {
        Title = title,
        PublishedOn = publishedOn,
        OrgPrice = price,
        ActualPrice = price,
    };

    if (authors == null) // The authors parameter, which is null, is considered to be a software error and throws an exception.
    {
        throw new ArgumentNullException(nameof(authors));
    }

    // Creates the BookAuthor class in the order in which the Authors have been provided
    byte order = 0;
    book._authorsLink = new HashSet<BookAuthor>(
        authors.Select(a =>
			new BookAuthor(book, a, order++)));

    // If there are no Authors, add an error.
    if (!book._authorsLink.Any())
    {
        status.AddError("You must have at least one Author for a book.");
    }
    
    return status.SetResult(book); // Sets the status's Result to the new Book instance. If there are errors, the value is null.
}
````

For simple entity classes, you can use a public constructor with specific parameters, but any entities that have business rules and return error messages should use a static factory in the entity class.



#### Understanding the differences between an entity and a value object

DDD talks about an entity (the `Book` entity being an example), but it also talks about a *value object*. The difference is what uniquely defines an instance of each.

* An entity isn’t defined by the data inside it. We expect that more than one person named John Smith has written a book, for example. Therefore, the Book App would need a different `Author` entity for each author named John Smith.
* A value object is defined by the data inside it. If we have an address to send an order to, and another address with the same road, city, state, zip code, and country, was created, the two instances of the address are said to be equal.

From an EF Core perspective, a DDD entity is an EF Core entity class, which is saved to the database with some form of primary key. The primary key ensures that the entity is unique in the database, and when EF Core returns a query including entity classes (and the query doesn’t include any form of the `AsNoTracking` method), it uses a single instance for each entity class that has the same primary key.

You can implement a value object by using EF Core’s owned type. The main form of an owned type is a class with no primary key; the data is added to the table it is included in.



#### Minimizing the relationships between entity classes

Added two-way relationships between entities mean you need to understand both entities when working on either entity, which makes the code harder to understand. Our recommendation is to minimize the relationships.

A `Book`, for example, has a navigational property of all the `Review`s for a `Book`, but the `Review` does not have a navigational property back to the `Book`. Understanding the `Book` entity requires some idea of what the `Review` entity does, but when dealing with the `Review` entity, we had to understand only what the `Review` entity does.



#### Grouping entity classes

The aggregates principle says that you should group entities that can be considered to be "one unit for the purpose of data changes". One of the entities in an aggregate is the root aggregate, and any changes in the other aggregates are made via this root aggregate.



![](./diagrams/images/13_04_grouping_entity_classes.png)



The aggregate rule simplifies the handling of entities classes because one root entity can handle multiple aggregates in its group. Also, the root entity can validate that the other, nonroot aggregates are set up correctly for the root aggregate, such as the Book create factory’s checking that there is at least one `Author` for a `Book` entity.



````c#
// The access methods that control the aggregate entity Review
public void AddReview(int numStars, string comment, string voterName) // Adds a new review with the given parameters
{
    if (_reviews == null) // This code relies on the _reviews field to be loaded, so it throws an exception if it isn’t.
    {
        throw new InvalidOperationException("Reviews collection not loaded");
    }
    
    _reviews.Add(new Review(
        numStars, comment, voterName)); // Creates a new Review, using its internal constructor
}

public void RemoveReview(int reviewId) // Removes a Review, using its primary key
{
    // Finds the specific Review to remove
    if (_reviews == null)
    {
        throw new InvalidOperationException("Reviews collection not loaded");
    }

    var localReview = _reviews.SingleOrDefault(x => x.ReviewId == reviewId);

    if (localReview == null) // Not finding the Review is considered to be a software error, so the code throws an exception.
    {
        throw new InvalidOperationException("The review wasn’t found");
    }

    _reviews.Remove(localReview); // The found Review is removed.
}
````

One additional change you make is marking the `Review` entity class’s constructor as `internal`. That change stops a developer from adding a `Review` by creating an instance outside the `Book` entity.



#### Deciding when the business logic shouldn’t be run inside an entity

DDD says that you should move as much of your business logic inside your entities, but the DDD aggregates rule says that the root aggregate should work only with other entities in the aggregate group. If you have business logic that includes more than one DDD aggregate group, you shouldn’t put (all) the business logic in an entity; you need to create some external class to implement the business logic.

An example of a situation that requires more than one aggregate group in the business logic is processing a user’s order for books. This business logic involves the Book entity, which is in the `Book`/`Review`/`BookAuthor` aggregate group, and the `Order`/`LineItem` aggregate group.



````c#
// PlaceOrderBizLogic class working across Book and Order entities
public async Task<IStatusGeneric<Order>> // This method returns a status with the created Order, which is null if there are no errors.
    CreateOrderAndSaveAsync(PlaceOrderInDto dto) // The PlaceOrderInDto contains a TandC bool, and a collection of BookIds and number of books.
{
    var status = new StatusGenericHandler<Order>(); // This status is used to gather errors, and if there are no errors, the code returns an Order.

    // Validate the user's input
    if (!dto.AcceptTAndCs)
    {
        return status.AddError("accept T&Cs…");
    }
    if (!dto.LineItems.Any())
    {
        return status.AddError("No items in your basket.");
    }

    // The _dbAccess contains the code to find each book.
    var booksDict = await _dbAccess
        .FindBooksByIdsAsync(dto.LineItems.Select(x => x.BookId));

    // This method creates a list of bookIds and numbers of books.
    var linesStatus = FormLineItemsWithErrorChecking(dto.LineItems, booksDict);
    // If any errors were found while checking each order line, returns the error status
    if (status.CombineStatuses(linesStatus).HasErrors)
    {
        return status;
    }

    // Calls the Order static factory. It is the Order's job to form the Order with LineItems.
    var orderStatus = Order.CreateOrder(dto.UserId, linesStatus.Result);

    if (status.CombineStatuses(orderStatus).HasErrors) // Again, any errors will abort the Order and return errors.
    {
        return status;
    }

    await _dbAccess.AddAndSave(orderStatus.Result); // The _dbAccess contains the code to add the Order and call SaveChangesAsync.

    return status.SetResult(orderStatus.Result); // Returns a successful status with the created Order entity
}
````

````c#
// This static factory creates an Order with the LineItems, with error checks
public static IStatusGeneric<Order> CreateOrder // This static factory creates the Order with lineItems.
    (Guid userId, // The Order uses the UserId to show orders only to the person who created it.
     IEnumerable<OrderBookDto> bookOrders) // The OrderBookDto lives in the Order domain and carries the info that the Order needs.
{
    var status = new StatusGenericHandler<Order>(); // Creates a status to return with an optional result of Order
    var order = new Order
    {
        UserId = userId,
        DateOrderedUtc = DateTime.UtcNow
    };

    byte lineNum = 1;
    order._lineItems = new HashSet<LineItem>(
        bookOrders
        .Select(x => new LineItem(x, lineNum++))); // Creates each of the LineItems in the same order in which the user added them

    if (!order._lineItems.Any()) // Double-checks that the Order is valid
    {
        status.AddError("No items in your basket.");
    }

    return status.SetResult(order); // Returns the status with the Order. If there are errors, the status sets the result to null.
}
````



#### Applying DDD’s bounded context to your application’s DbContext

Using an SQL View is an excellent solution in this case because it follows many of the DDD rules. First, the `BookView` contains only the data that the `Orders` side needs, so the developer isn’t distracted by irrelevant data. Second, when an entity class is configured as a View, EF Core marks that entity class as read-only, enforcing the DDD rule that only the `Books` entity should be able to change the data in the Books table. Another benefit is that a class mapped to an SQL View won’t add migration code to alter that table.



![](./diagrams/images/13_05_ddd_bounded_context.png)



### Using your DDD-styled entity classes in your application

The DDD approach is to keep the focus on the domain mode - that is, on the entities and their relationships. Conversely, it doesn’t want the database (DDD persistence) parts to distract the developer who is working on the domain design. The idea is that the entity and its relationships (navigational properties in EF Core terms) are all the developer needs to consider when solving domain issues.



#### Calling the AddPromotion access method via a repository pattern

Repositories are classes or components that encapsulate the logic required to access data sources. They centralize common data access functionality, providing better maintainability and decoupling the infrastructure or technology used to access databases from the domain model layer.



````c#
// A generic repository that handles some basic database commands
public class GenericRepository<TEntity> // The generic repository will work with any entity class.
    where TEntity : class
{
	protected readonly DbContext Context; // The repository needs the DbContext of the database.

    public GenericRepository(DbContext context)
    {
        Context = context;
    }

    public IQueryable<TEntity> GetEntities() // Returns an IQueryable query of the entity type
    {
        return Context.Set<TEntity>();
    }

    public async Task<TEntity> FindEntityAsync(int id) // This method finds and returns a entity with a integer primary key.
    {
        var entity = await Context.FindAsync<TEntity>(id); // Finds an entity via its single, integer primary key

        if (entity == null)
        {
            throw new Exception("Could not find entity"); // A rudimentary check that the entity was found
        }

        return entity;
    }

    public Task PersistDataAsync()
    {
        return Context.SaveChangesAsync(); // Calls SaveChanges to update the database
    }
}
````

````c#
// Handling the AddPromotion update by using a repository pattern
public class AdminController : Controller
{
    private readonly GenericRepository<Book> _repository; // The GenericRepository<Book> is injected into the Controller.

    public AdminController(
        GenericRepository<Book> repository)
    {
        _repository = repository;
    }

    public async Task<IActionResult> AddPromotion(int id)
    {
        var book = await _repository.FindEntityAsync(id);

        var dto = new AddPromotionDto // Copies over the parts of the Book you need to show the page
        {
            BookId = id,
            Title = book.Title,
            OrgPrice = book.OrgPrice
        };

        return View(dto);
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> AddPromotion(AddPromotionDto dto)
    {
        if (!ModelState.IsValid)
        {
            return View(dto);
        }

        var book = await _repository
            .FindEntityAsync(dto.BookId);
        var status = book.AddPromotion(dto.ActualPrice, dto.PromotionalText); // Calls the AddPromotion access method with the two properties from the dto

        if (!status.HasErrors)
        {
            await _repository.PersistDataAsync(); // The access method returned no errors, so you persist the data to the database.
            return View("BookUpdated", "Updated book…");
        }

        //Error state
        status.CopyErrorsToModelState(ModelState, dto);
        return View(dto);
    }
}
````



#### Calling the AddPromotion access method via a class-to-method-call library

Although calling DDD access methods by using a repository system works, this approach has some repetitious code, such as in the first stage, where you copy properties into a DTO/ViewModel to show to the user, and in the second stage, where returned data in the DTO is turned into an access method call. What would happen if you had a way to automate this process? (Nuget library: `EfCore.GenericServices`)



````c#
// Handling the AddPromotion update by using GenericServices
public class AdminController : Controller
{
    private readonly ICrudServicesAsync _service; // The ICrudServicesAsync service comes from GenericServices and is injected via the Controller's constructor.

    public AdminController(ICrudServicesAsync service)
    {
        _service = service;
    }

    public async Task<IActionResult> AddPromotion(int id)
    {
        var dto = await _service
            .ReadSingleAsync<AddPromotionDto>(id); // The ReadSingleAsync<T> reads into the DTO, using the given primary key.

        return View(dto);
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> AddPromotion(AddPromotionDto dto)
    {
        if (!ModelState.IsValid)
        {
            return View(dto);
        }

        await _service.UpdateAndSaveAsync(dto); // The UpdateAndSaveAsync method calls the access method, and if no errors occur, it saves the access method to the database.

        if (!_service.HasErrors)
        {
            return View("BookUpdated", service.Message);
        }

        //Error state
        _service.CopyErrorsToModelState(ModelState, dto);
        return View(dto);
    }
}
````



#### Adding a Review to the Book entity class via a repository pattern

When you’re updating navigational properties, you need to handle another step: preloading the navigational property.



````c#
// Add the LoadBookWithReviewsAsync method to the repository
public class BookRepository : GenericRepository<Book>
{
    public BookRepository(DbContext context)
        : base(context) { } // The GenericRepository needs the application's DbContext.

    public async Task<Book> LoadBookWithReviewsAsync(int bookId)
    {
        var book = await GetEntities() // Uses the GenericRepository's GetEntities to get a IQueryable<Book> query
            .Include(b => b.Reviews) // Makes sure that the Review collection is loaded with the book
            .SingleOrDefaultAsync(b => b.BookId == bookId);
        if (book == null) // A rudimentary check that the entity was found
        {
            throw new Exception("Could not find book");
        }
        return book; // Returns the book with the Reviews collection loaded
    }
}
````

This code snippet shows how you would call the `LoadBookWithReviewsAsync` method in ASP.NET Core's `POST` action method:

````c#
var book = await _repository
    .LoadBookWithReviewsAsync(dto.BookId);
book.AddReview(dto.NumStars, dto.Comment, dto.VoterName);
await _repository.PersistDataAsync();
````



#### Adding a Review to the Book entity class via a class-to-method-call library

For preloading navigational properties, the `GenericServices` library provides an `IncludeThen` attribute that you add to the DTO. This attribute allows you to define the name of navigational properties to `lnclude` or `ThenInclude`.



````c#
// The AddReviewDto class with an attribute to load the Reviews
[IncludeThen(nameof(Book.Reviews))] // The IncludeThen attribute includes the Book's Reviews navigational property.
public class AddReviewDto // The name of the DTO shows that it should call the AddReview access method.
    : ILinkToEntity<Book> // The entity that this DTO is linked to is the Book entity class.
{
    public int BookId { get; set; } // The primary key of the Book is filled in by the read and used by the update method.

    public string Title { get; set; } // The Title is read in on a read and used to confirm to the user what book they are adding a Review to.

    // These three properties are used as parameters in the AddReview access method.
    public string VoterName { get; set; }
    public int NumStars { get; set; }
    public string Comment { get; set; }
}  
````

After you add the `IncludeThen` attribute, any read of an entity will include the navigational properties. You use `GenericServices`' `ReadSingleAsync<T>` and `UpdateAndSaveAsync(dto)` methods the same way that you would access methods that do not have navigational properties to update.



### The downside of DDD entities: Too many access methods

When you are building a large application, the time it takes to write an access method grows if you have hundreds to write.

If the property has no business rules (other than validation attributes), the setter on that property could be made public.

As an example, if you look at the `Book` entity class, the `Title` property and the `Publisher` property have no business logic but should not be empty, so the setter of these two properties could be made public without having any effect on the business rules. Making the properties' setter public would save you from writing two more access methods and allow JSON Patch or AutoMapper to update these properties.



### Getting around performance issues in DDD-styled entities

Typically, the performance issues in an application involve queries, and DDD doesn’t affect them at all. But if you have database write performance issues, you might feel the need to bypass DDD. Instead of ditching DDD, you have three ways to keep using DDD with minimal bending of the rules.

As an example, we look at the performance of adding or removing a `Review`. So far, you have loaded all the reviews before running add/remove access methods. If you have only a few Reviews, you have no problem, but if your site is like Amazon, where products can have thousands of reviews, loading all of them to add one new `Review` is going to be too slow.



````c#
// The updated Review public constructor with optional foreign key
internal Review( // The Review constructor is internal, so only entity classes can create a Review.
    int numStars, string comment, string voterName, // Standard properties
    int bookId = 0) // A new, optional property is added for setting the Review foreign key.
{
    // Sets the standard properties
    NumStars = numStars;
    Comment = comment;
    VoterName = voterName;

    if (bookId != 0) // If a foreign-key parameter was provided, the BookId foreign key is set.
    {
        BookId = bookId;
    }
}
````

After you have changed the `Review` entity, you can use any of three options:

* Allow database code into your entity classes.
* Make the `Review` constructor public and write non-entity code to add a `Review`.
* Use domain events to ask an event handler to add a `Review` to the database.

The other option is to expose a navigational property linking the `Review` back to the Book entity. This option keeps the entity from knowing about foreign keys but breaks the DDD rule on minimizing relationships. Pick which rule you want to break.



#### Allow database code into your entity classes

One solution is for the `AddReview` access method to have access to the application’s DbContext. You can provide the application’s DbContext by adding an extra parameter to the `AddReview`/`RemoveReview` access methods or using EF Core’s service injection.



````c#
// Providing the application’s DbContext to the access methods
public void AddReview(
    int numStars, string comment, string voterName, // The access method takes the normal AddReview inputs ...
    DbContext context) // ... but a new parameter is added, which is EF Core DbContext.
{
    if (BookId == default) // This method works only on a Book that is already in the database.
    {
        throw new Exception("Book must be in db");
    }

    if (context == null) // This method works only if an DbContext instance is provided.
    {
        throw new ArgumentNullException(nameof(context), "You must provide a context");
    }
    
    var reviewToAdd = new Review(
        numStars, comment, voterName,
        BookId); // Creates the Review and sets the Review BookId foreign key

    context.Add(reviewToAdd); // Uses the DbContext Add method to mark the new Review to be added to the database
}

public void RemoveReview(
    int reviewId, // The access method takes the normal RemoveReview input of the ReviewId.
    DbContext context)
{
    if (BookId == default) // This method works only on a Book that is already in the database.
    {
        throw new Exception("Book must be in db");
    }

    if (context == null) // This method works only if an DbContext instance is provided.
    {
        throw new ArgumentNullException(nameof(context), "You must provide a context");
    }

    var reviewToDelete = context.Set<Review>()
        .SingleOrDefault(x => x.ReviewId == reviewId); // Reads in the review to delete

    if (reviewToDelete == null) // A rudimentary check that the review entity was found
    {
        throw new Exception("Not found");
    }

    if (reviewToDelete.BookId != BookId) // If not linked to this Book, throw an exception.
    {
        throw new Exception("Not linked to book");
    }

    context.Remove(reviewToDelete);
}
````



#### Make the Review constructor public and write nonentity code to add a Review

This solution removes the database features from the Book’s access methods and places them in another project (most likely BizLogic). The solution makes the `Book` entity cleaner, but it does require the `Review` constructor’s access modifier to be changed to public. The downside is that anyone can create a `Review` entity instance.

The code to add/remove a `Review` is the same, but now it is in its own class. This solution breaks the following DDD rules:

* The `Book` entity isn’t in charge of the `Review` entities linked to it.
* The `Review` has a public constructor, so any developer can create a `Review`.
* The `Review` entity knows about a database feature: the `BookId` foreign key.



#### Use domain events to ask an event handler to add a review to the database

The last solution is to use a domain event to send a request to event handlers that add or remove a `Review`.



![](./diagrams/images/13_06_domain_evvents.png)



### Three architectural approaches: Did they work?

#### A modular monolith approach that enforces SoC by using projects

Here are a few tips for using the modular monolith approach:

* Use a hierarchal naming rule for your projects. A name like `BookApp.Persistence.EfCoreSql.Books`, for example, makes it easier to find things.
* Don’t end a project name with the name of a class. Instead, use something like …`Books`, not …`Book`.
* You’re going to get project names wrong.
* If you change the name of a project in Visual Studio by selecting the project and typing the new name, you don’t change the folder name.



#### DDD principles, both architecturally and on the entity classes

Here is a list of DDD features that made the development of the Book App easier:

* Each entity class contained all the code needed to create or update that entity and any aggregate entities. If we needed to change anything, we knew where to look, and we knew that there wasn’t another version of this code elsewhere.
* The DDD access methods were especially useful when we used domain events.
* The DDD access methods were even more useful when we used integration events because we had to capture every possible change to the `Book` entity and its aggregates, which was easy to do by adding integration events to every access method and static create factory method in the `Book` entity. If ew couldn’t capture all changes in that way, we would have to detect changes by using the entities' `State`, and we know from experience that detecting changes is hard to implement.
* The DDD bounded context that allowed two different EF Core DbContexts, `BookDbContext` and `OrderDbContext`, also worked well. Migrating the two parts of the same database worked fine.



#### Clean architecture

The Book App consisted of five layers, starting at the center and working out:

* *Domain* - Holding entity classes
* *Persistence* - Holding the DbContexts and other database classes
* *Infrastructure* - Holding a mixture of projects, such as seeding the database and event handlers
* *ServiceLayers* - Holding code to adapt the lower layers to the frontend
* *UI* - Holding the ASP.NET Core application



![](./diagrams/images/13_07_clean_architecture.png)



### Summary

* The architecture you use to build an application should help you focus on the feature you are adding while keeping the code nicely segregated so that it’s easier to refactor.
* DDD provides lots of good recommendations on how to build an application.
* DDD-styled entities control how they are created and updated; it’s the job of an entity to ensure that the data inside it is valid.
* DDD has lots of rules to make sure that developers can put all their effort into the domain (business) needs that they have been asked to implement.
* DDD groups entities into aggregates and says that one entity in the group, known as the root aggregate, manages the data and relationships across the aggregate.
* Bounded context is a major DDD concept.
* To update a DDD entity, you call a method within the entity class.
* To create a new instance of an DDD entity, you use a constructor with specific parameters or a static create factory method that returns validation feedback.
* To update a DDD entity, first load the entity so that you can call the access method. You can do this via normal EF Core code, a repository, or the `EFCore.GenericServices` library.
* Updating collection relationships can be slow if there are lots of existing entries in the collection. You have three ways to improve performance in these cases.





## 14. EF Core performance tuning

* Deciding which performance issues to fix
* Employing techniques that find performance issues
* Using patterns that promote good performance
* Finding patterns that cause performance issues



### Part 1: Deciding which performance issues to fix

#### "Don’t performance-tune too early" doesn’t mean you stop thinking

The number-one goal is to get your application working properly first, but:

* Make sure that any software patterns you use don’t contain inherent performance problems. Otherwise, you’ll be building in inefficiencies from day one.
* Don’t write code that makes it hard to find and fix performance problems. If you mix your database access code with other code, such as frontend code, for example, performance changes can get messy and difficult to test.
* Don’t pick the wrong architecture. Nowadays, the scalability of web applications is easier to improve by running multiple instances of the web application. But if you have an application that needs high scalability, a Command and Query Responsibility Segregation (CQRS) architecture might help.



#### How do you decide what’s slow and needs performance tuning?

You need clear metrics:

* Define the feature. What’s the exact query/command that needs improving, and under what circumstances is it slow (number of concurrent users, for example)?
* Get timings. How long does the feature take now, and how fast does it need to be?
* Estimate the cost of the fix. How much is the improvement worth? When should you stop?
* Prove that it still works. Do you have a way to confirm that the feature is working properly before you start the performance tuning and that it still works after the performance change?



#### The cost of finding and fixing performance issues



![](./diagrams/images/14_01_cost_of_finding_and_fixing_performance_issues.png)



### Part 2: Techniques for diagnosing a performance issue



![](./diagrams/images/14_02_diagnosing_performance_issue.png)



#### Stage 1: Get a good overview, measuring the user’s experience



#### Stage 2: Find all the database code involved in the feature you’re tuning



#### Stage 3: Inspect the SQL code to find poor performance

**CAPTURING THE LOGGING OUTPUT**

````c#
// Capturing EF Core’s logging output in a unit test
var logs = new List<string>(); // Holds all the logs that EF Core outputs
var builder = new DbContextOptionsBuilder<BookDbContext>() // The DbContextOptionsBuilder<T> is the way to build the options needed to create a context.
    .UseSqlServer(connectionString) // Says you are using a SQL Server database and takes in a connection string
    .EnableSensitiveDataLogging() // By default, exceptions don't contain sensitive data. This code includes sensitive data.
    .LogTo(log => logs.Add(log), // The log string is captured and added to the log.
		LogLevel.Information); // Sets the log level. Information level contains the executed SQL.
using var context = new BookDbContext(builder.Options); // Creates the application's DbContext—in this case, the context holding the books data
//... your query goes here
````



**EXTRACTING THE SQL COMMANDS SENT TO THE DATABASE**

````sql
// An Information log showing the SQL command sent to the database
Executed DbCommand (4ms) -- Tells you how long the database took to return from this command
	[Parameters=[], -- If any external parameters are used in the command, their names will be listed here.
	CommandType='Text',
	CommandTimeout='30'] -- The timeout for the command. If the command takes more than that time, it's deemed to have failed.
SELECT [p].[BookId], [p].[Description], -- SQL command that was sent to the database
	[p].[ImageUrl], [p].[Price],
	[p].[PublishedOn], [p].[Publisher],
	[p].[Title],
	[p.Promotion].[PriceOfferId],
	[p.Promotion].[BookId],
	[p.Promotion].[NewPrice],
	[p.Promotion].[PromotionalText]
FROM [Books] AS [p]
LEFT JOIN [PriceOffers] AS [p.Promotion]
ON [p].[BookId] = [p.Promotion].[BookId]
ORDER BY [p].[BookId] DESC
````



### Part 3: Techniques for fixing performance issues

* Good EF Core patterns - "Apply always" patterns that you might like to adopt. They aren’t foolproof but give your application a good start.
* Poor database query patterns - EF Core code antipatterns, or patterns you shouldn’t adopt, because they tend to produce poor-performing SQL queries.
* Poor software patterns - EF Core code antipatterns that make your database write code run more slowly.
* Scalability patterns - Techniques that help your database handle lots of database accesses.



### Using good patterns makes your application perform well

#### Using Select loading to load only the columns you need

Creating a Select query with a DTO does take more effort than using eager loading with the Include method, but benefits exist beyond higher database access performance, such as reducing coupling between layers.



#### Using paging and/or filtering of searches to reduce the rows you load

You need to apply commands that will limit the amount of data returned to the user.

* *Paging* - You return a limited set of data to the user (say, 100 rows) and provide the user commands to step through the "pages" of data.
* *Filtering* - If you have a lot of data, a user will normally appreciate a search feature, which will return a subset of the data.



#### Warning: Lazy loading will affect database performance

Lazy loading is a technique that allows relationships to be loaded when read. The problem is that lazy loading has a detrimental effect on the performance of your database accesses, and after you’ve used lazy loading in your application, replacing it can require quite a bit of work. If you have too many lazy loads, your query is going to be slow.



#### Always adding the AsNoTracking method to read-only queries

If you’re reading in entity classes directly and aren’t going to update them, including the `AsNoTracking` method in your query is worthwhile.

````c#
// Using the AsNoTracking method to improve the performance of a query
var result = context.Books
    .Include(r => r.Reviews)
    .AsNoTracking() // Adding the AsNoTracking method tells EF Core not to create a tracking snapshot, which saves time and memory use.
    .ToList();
````

If you use a `Select` query in which the result maps to a DTO, and that DTO doesn’t contain any entity classes, you don’t need to add the `AsNoTracking` method. But if your DTO contains an entity class, adding the `AsNoTracking` method will help.



#### Using the async version of EF Core commands to improve scalability

Microsoft’s recommended practice for ASP.NET applications is to use async commands wherever possible.



#### Ensuring that your database access code is isolated/decoupled

* *Is in a clearly defined place (isolated).* Isolating each database access into its own method allows you to find the database code that’s affecting performance.
* *Contains only the database access code (decoupled).* My advice is to not mix your database access code with other parts of the application, such as the UI or API. That way, you can change your database access code without worrying about other, nondatabase issues.



### Performance antipatterns: Database queries

#### Antipattern: Not minimizing the number of calls to the database

If you’re reading an entity from the database with its related data, you have four ways of loading that data: select loading, eager loading, explicit loading, and lazy loading. Although all three techniques achieve the same result, their performance differs quite a lot. The main difference comes down to the number of separate database accesses they make; the more separate database accesses you do, the longer your database access will take.



| Type of query                     | #Database accesses | EF time (ms) / % |
| --------------------------------- | ------------------ | ---------------- |
| `Select` and eager loading        | 1                  | 1.95 / 100%      |
| Eager loading with `AsSplitQuery` | 4                  | 2.10 / 108%      |
| Explicit and lazy loading         | 6                  | 2.10 / 108%      |

With the improvements in EF Core, the differences between `Select`/eager, eager with `AsSplitQuery`, and explicit/lazy loading are smaller, but multiple accesses to the database still have a cost. So the rule is to try to create one LINQ query that gets all the data you need in one database access. `Select` queries are the best-performing if you need only specific properties; otherwise, eager loading, with its `Include` method, is better if you want the entity with its relationships to apply an update.



#### Antipattern: Missing indexes from a property that you want to search on

If you plan to search on a property that isn’t a key (EF Core adds an index automatically to primary, foreign, or alternate keys), adding an index to that property will improve search and sort performance.



#### Antipattern: Not using the fastest way to load a single entity

| Method                                            | Time    | Ratio to single |
| ------------------------------------------------- | ------- | --------------- |
| `context.Books.Single(x => x.BookId == id)`       | 175 us. | 100%            |
| `context.Books.First(x => x.BookId == id)`        | 190 us. | 109%            |
| `context.Find<Book>(id)` (entity not tracked)     | 610 us. | 350%            |
| `context.Find<Book>(id)` (entity already tracked) | 0.5 us. | 0.3%            |

`Single` (and `SingleOrDefault`) was fastest for a database access, and also better than using `First`, as `Single` will throw an exception if your `Where` clause returns more than one result. `Single` and `First` also allow you to use `Includes` in your query.

You should use the `Find` method if the entity is being tracked in the context, in which case `Find` will be super-fast. `Find` is fast because it scans the tracked entities first, and if it finds the required entity, it returns that entity without any access to the database. The downside of this scan is that `Find` is slower if the entity isn’t found in the context.



#### Antipattern: Allowing too much of a data query to be moved into the software side

````c#
// Two LINQ commands that would have different performance times
context.Books.Where(p => p.Price > 40).ToList(); // This query would perform well, as the Where part would be executed in the database.
context.Books.ToList().Where(p => p.Price > 40); // This query would perform badly, as all the books would be returned (which takes time), and then the Where part would be executed in software.
````



#### Antipattern: Not moving calculations into the database



#### Antipattern: Not replacing suboptimal SQL in a LINQ query

Sometimes, you know something about your data that allows you to come up with a piece of SQL code that’s better than EF Core. But at the same time, you don’t want to lose the ease of creating queries with EF Core and LINQ. You have several ways to add SQL calculations to the normal LINQ queries:

* *Add user-defined functions to your LINQ queries.* A scalar-valued user-defined function returns a single value that you can assign to a property
  in a query, whereas a table-valued UDF returns data as though it came from
  a table.
* *Create an SQL View in your database that has the SQL commands to compute values.*
  Map an entity class to that View and then apply LINQ queries
  to that mapped entity class. This approach gives you room to add some
  sophisticated SQL inside the View while using LINQ to access that data.
* *Use EF Core’s raw SQL methods `FromSqlRaw` and `FromSqlInterpolated`.* These
  methods allow you to use SQL to handle the first part of the query. You can follow
  with other LINQ commands, such as `sort` and `filter`.
* *Configure a property as a computed column.* Use this approach if that property calculation
  can be done with other properties/columns in the entity class and/or
  SQL commands.



#### Antipattern: Not precompiling frequently used queries

When you first use an EF Core query, it’s compiled and cached, so if you use it again, the compiled query can be found in the cache, which saves compiling the query again. But there’s a (small) cost to this cache lookup, which the EF Core method `EF.CompiledQuery` can bypass. If you have a query that you use a lot, it’s worth trying.

* You can use a compiled query only if the LINQ command isn’t built dynamically, with parts of the LINQ being added or removed.
* The query returns a single entity class - an `IEnumerable<T>` or an `IAsyncEnumerable<T>` - so you can’t chain query objects.

The `EF.CompiledQuery` method allows you to hold the compiled query in a static variable, which removes the cache lookup part.



### Performance antipatterns: Writes

#### Antipattern: Calling SaveChanges multiple times

If you have lots of information to add to the database, you have two options:

* *Add one entity and call `SaveChanges`.* If you’re saving 10 entities, call the Add method followed by a call to the `SaveChanges` method 10 times.
* *Add all the entity instances, and call `SaveChanges` at the end.* To save 10 entities, call `Add` 10 times (or, better, one call to `AddRange`) followed by one call to `SaveChanges` at the end.

Option 2 - calling `SaveChanges` only once - is a lot faster because EF Core will batch multiple data writes on database servers that allow this approach, such as SQL Server. As a result, this approach generates SQL code that’s more efficient at writing multiple items to the database.



A comparison of calling `SaveChanges` after adding each entity, and adding all the entities and then calling `SaveChanges` at the end. Calling `SaveChanges` at the end is about 15 times faster than calling `SaveChanges` after every Add.

* One at a time

````c#
for (int i = 0; i < 100; i++)
{
    context.Add(new MyEntity());
    context.SaveChanges();
}
// Total time = 160 ms
````

* All at once (batched in SQL Server)

````c#
for (int i = 0; i < 100; i++)
{
    context.Add(new MyEntity());
}
context.SaveChanges();
// Total time = 9 ms
````



#### Antipattern: Making DetectChanges work too hard

Time taken by the `SaveChanges` method, which contains the call to the `DetectChanges.Detect` method, to save one entity for different levels of tracked entities. Note that the tracked entities used in this table are small.

| Number of tracked entities | How long SaveChanges took | How much slower? |
| -------------------------- | ------------------------- | ---------------- |
| 0                          | 0.2 ms.                   | n/a              |
| 100                        | 0.6 ms.                   | 2 times slower   |
| 1000                       | 2.2 ms.                   | 11 times slower  |
| 10000                      | 20.0 ms.                  | 100 times slower |



#### Antipattern: Not using HashSet<T> for navigational collection properties

* When you’re loading collection navigational properties in a query - say, by using the `Include` method - `HashSet<T>` for collections is quicker than collection navigational properties using `ICollection<T>` / `IList<T>`. Adding an entity with 1000 entities in a collection navigational property, for example, took 30% longer with `ICollection<T>` than using `HashSet<T>` because it is easier to detect/find instances in a `HashSet<T>`.
* The more tracked entities of the same type found in the entity (and its relationships) that was added, the more time it takes to check them all. The performance hit is hard to measure but seems to be small. But if you have issues with an Add taking a long time, it’s worthwhile to check for a lot of tracked entities, which may be part of the slowness of your Add method call.
* The downside of using `HashSet<T>` is that it does not guarantee the order of the entries in the collection. So if you are using EF Core’s ability sort entries in an Include method, you can’t use `HashSet<T>`.



#### Antipattern: Using the Update method when you want to change only part of the entity

EF Core is great at detecting changes to individual properties in an entity class using the `DetectChanges.Detect` method.



#### Antipattern: Startup issue—Using one large DbContext



![](./diagrams/images/14_03_antipattern_using_one_large_dbcontext.png)



### Performance patterns: Scalability of database accesses

#### Using pooling to reduce the cost of a new application’s DbContext

If you’re building an ASP.NET Core application, EF Core provides a method called `AddDbContextPool<T>` that replaces the normal `AddDbContext<T>` method. The `AddDbContextPool<T>` method uses an internal pool of an application’s DbContext instances, which it can reuse. This method speeds your application’s response time when you have lots of short requests.

But be aware that you shouldn’t use it in some situations. When you’re passing in data based on the HTTP request, such as the logged-in user’s ID, you shouldn’t use DbContext pooling because it would use the wrong user ID in some instances of the application’s DbContext.



````c#
// Using AddDbContextPool to register the application’s DbContext
services.AddDbContextPool<EfCoreContext>( // You’re using an SQL Server database, but pooling works with any database provider.
    options => options.UseSqlServer(connection, // You register your application DbContext by using the AddDbContextPool<T>.
	b => b.MigrationsAssembly("DataLayer"))); // Because you’re using migrations in a layered architecture, you need to tell the database provider which assembly the migration code is in.
````



#### Adding scalability with little effect on overall speed

async/await releases a thread to allow other requests to be handled while the async part is waiting for the database to respond. Using an async method instead of the normal, synchronous method does add a small amount of overhead to each call but the slow queries need async, as it releases a thread for a long time. The fact that the fastest queries have the smallest sync/async difference says that using async won’t penalize the small queries. Overall, you have plenty to gain and little downside from using async/await.

Performance for a mixture of types of database access returning books, using sync and async versions. The database contains 1000 books.

| Type of database access                             | #DB trips | Sync     | Async    | Difference |
| --------------------------------------------------- | --------- | -------- | -------- | ---------- |
| Read book only, simple load                         | 1         | 0.7 ms.  | 0.8 ms.  | 112%       |
| Read book, eager-load relationships                 | 1         | 9.7 ms.  | 13.7 ms. | 140%       |
| Read book, eager-load relationships+sort and filter | 1         | 10.5 ms. | 14.5 ms. | 140%       |



#### Helping your database scalability by making your queries simple



#### Scaling up the database server



#### Picking the right architecture for applications that need high scalability



### Summary

* Don’t performance-tune too early; get your application to work properly first. But try to design your application so that if you need to performance-tune later, it’s easier to find and fix your database code.
* Performance tuning isn’t free, so you need to decide what performance issues are worth the development effort to fix.
* EF Core’s log output can help you identify database access code that has performance issues.
* Make sure that any standard patterns or techniques you use in writing your application perform well. Otherwise, you’ll bake in performance issues from day one.
* Avoid or fix any database performance antipatterns (database accesses that don’t perform well).
* If scalability is an issue, try simple improvements, but high scalability may need a fundamental rethinking of the application’s architecture.





## 15. Master class on performance-tuning database queries

* Understanding four different approaches to performance-tuning EF Core queries
* Comparing the different performance gains each approach provides
* Extracting the good practices from each approach to use in your applications
* Evaluating the skills and development effort needed to implement each approach
* Understanding what database scalability is and how to improve it



### Good LINQ approach: Using an EF Core Select query

````c#
// MapBookToDto method that selects what to show in the book display query
public static IQueryable<BookListDto> MapBookToDto(this IQueryable<Book> books)
{
    return books.Select(p => new BookListDto // Good practice: Load only the properties you need.
    {
        BookId = p.BookId,
        Title = p.Title,
        PublishedOn = p.PublishedOn,
        EstimatedDate = p.EstimatedDate,
        OrgPrice = p.OrgPrice, // Good practice: Use indexed properties to sort/filter on (in this case, the ActualPrice).
        ActualPrice = p.ActualPrice,
        PromotionText = p.PromotionalText,
        AuthorsOrdered = string.Join(", ", // Good practice: Don’t load the whole entity of each relationships, only the parts you need.
			p.AuthorsLink
				.OrderBy(q => q.Order)
                .Select(q => q.Author.Name)),
        TagStrings = p.Tags
            .Select(x => x.TagId).ToArray(),
        ReviewsCount = p.Reviews.Count(), // Good practice: The ReviewsCount and ReviewsAverageVotes are calculated in the database.
        ReviewsAverageVotes =
            p.Reviews.Select(y =>
				(double?)y.NumStars).Average(),
        ManningBookUrl = p.ManningBookUrl
    });
}
````



**LOADING ONLY THE PROPERTIES YOU NEED FOR THE QUERY**

You could have loaded the whole `Book` entity, but that would mean loading data you didn’t need. The Manning Publications book data contains large strings summarizing the book’s content, what technology it covers, and so on. The book display doesn’t need that data, however, and loading it would make the query slower, so you don’t load it.

In line with the recommendation that you don’t performance-tune too early, you might start with a simple query that reads in the entity classes, and performance-tune later. If a query is slow and you are loading the whole entity class, consider changing to the LINQ `Select` method and loading only the properties you need.



**DON’T LOAD WHOLE RELATIONSHIPS - ONLY THE PARTS YOU NEED**

Typically, you don’t need to load the relationship’s whole entity classes. If you need data from relationships, try to extract the specific parts from any relationships. An even better idea is to move calculations into the database if you can.



**IF POSSIBLE, MOVE CALCULATIONS INTO THE DATABASE**

If you want good performance, especially for sorting or filtering on values that need calculating, it’s much better for the calculation to be done inside the database. Calculating data inside the database has two benefits:

* The data used in the calculation never leaves the database, so less data needs to be sent back to the application.
* The calculated value can be used in a sort or filter, so you can execute the query in one command to the database.



**IF POSSIBLE, USE INDEXED PROPERTIES TO SORT/FILTER ON**

````c#
ActualPrice = book.Promotion == null
    ? book.Price
    : book.Promotion.NewPrice,
PromotionPromotionalText =
    book.Promotion == null
    ? null
    : book.Promotion.PromotionalText,
````

That code has two negative effects on sorting on price: the LINQ is converted to an SQL `JOIN` to find the optional `PriceOffers` row, which takes time, and you can’t add a SQL index to this calculation. Changing the code to not use the `PriceOffer` entity removes the SQL `JOIN`, and you can add an SQL `INDEX` to the `ActualPrice` column in the database, significantly improving the sort-on-price feature.

So if you need to query some data, especially if you’re sorting or filtering on that data, try to precompute the data in your code. Or use a persisted computed column if the property is calculated based on other properties/columns in the same entity class, such as `[TotalPrice] AS (NumBook * BookPrice)`. That way, you will get a significant improvement in any sort or filter because of the SQL index on that column.



### LINQ+UDFs approach: Adding some SQL to your LINQ code

````sql
CREATE FUNCTION AuthorsStringUdf (@bookId int)
RETURNS NVARCHAR(4000)
AS
BEGIN
DECLARE @Names AS NVARCHAR(4000)
SELECT @Names = COALESCE(@Names + ', ', '') + a.Name
FROM Authors AS a, Books AS b, BookAuthor AS ba
WHERE ba.BookId = @bookId
	AND ba.AuthorId = a.AuthorId
	AND ba.BookId = b.BookId
ORDER BY ba.[Order]
RETURN @Names
END
````

````c#
// MapBookUdfsToDto using UDFs to concatenate Name/Tag names
public static IQueryable<UdfsBookListDto> MapBookUdfsToDto(this IQueryable<Book> books) // Updated MapBookToDto method, now called MapBookUdfsToDto
{
    return books.Select(p => new UdfsBookListDto
	{
        BookId = p.BookId,
        Title = p.Title,
        PublishedOn = p.PublishedOn,
        EstimatedDate = p.EstimatedDate,
        OrgPrice = p.OrgPrice,
        ActualPrice = p.ActualPrice,
        PromotionText = p.PromotionalText,
        AuthorsOrdered = UdfDefinitions // The AuthorsOrdered and TagsString are set to the strings from the UDFs.
            .AuthorsStringUdf(p.BookId),
        TagsString = UdfDefinitions
            .TagsStringUdf(p.BookId),
        ReviewsCount = p.Reviews.Count(),
        ReviewsAverageVotes =
            p.Reviews.Select(y =>
				(double?)y.NumStars).Average(),
        ManningBookUrl = p.ManningBookUrl
    });
}
````

When you change the `MapBookToDto` extension method to use the `AuthorsStringUdf` and the `TagsStringUdf` UDFs, each book returns only one row, and there is no `ORDER BY` other than the default ordering on `BookId`, descending. This change has a small effect on a nonsorted display of 100 books (improving it by a few milliseconds), but the big effect is on the sort by average votes, which comes down from 840 ms in the Good LINQ approach to 620 ms in the LINQ+SQL approach - an improvement of about 25%.



### SQL+Dapper: Creating your own SQL



![](./diagrams/images/15_01_sql_dapper.png)




![](./diagrams/images/15_02_sql_dapper.png)



### LINQ+caching approach: Precalculating costly query parts

Precalculating the parts of the query that take a long time to calculate and storing them in extra properties/columns in entity class. This technique is known as *caching* or *denormalization*.

Caching has the biggest effect on the sort-by-votes query, making it about 14 times faster than the Good LINQ approach and 8 times faster than the Dapper approach.

But when you’re thinking about using caching, you also need to think about how often the cached value is updated and how long it takes to update the cache. If the data that is cached is updated a lot, the cost of updating the cache may move the performance problem from running the query to updating entities.

Adding a caching system isn’t trivial to implement. Here are the steps:

1. Add a way to detect changes that affect the cached values.
2. Add code to update the cached values.
3. Add the cache properties to the Book entity and provide concurrency code to handle simultaneous updates of the cached values.
4. Build the book display query to use the cached values.



#### Adding a way to detect changes that affect the cached values

One positive feature of the domain events approach is that the change that triggers an update of a cached value is saved in the same transaction that saves the cached value. As a result, both changes are applied to the database, or if anything fails, none of the updates are applied to the database. That approach prevents the real data and cached data from getting out of step (known as a dirty cache).



To speed the development, you are going to use my `EfCore.GenericEventRunner` library.

````c#
// BookDbContext updated to use GenericEventRunner
public class BookDbContext // The BookDbContext handles the Books side of the data.
    : DbContextWithEvents<BookDbContext> // Instead of inheriting EF Core’s DbContext, you inherit the class from GenericEventRunner.
{
    public BookDbContext(
        DbContextOptions<BookDbContext> options,
        IEventsRunner eventRunner = null) // DI will provide GenericEventRunner’s EventRunner. If null, no events are used (useful for unit tests).
        : base(options, eventRunner) // The constructor of the DbContextWithEvents class needs the EventRunner.
    { }

    //… rest of BookDbContext is normal, so left out
}
````

The next stage is adding the events to the `Book`'s `AddReview` and `RemoveReview` access methods.

````c#
// The Book entity with the AddReview and RemoveReview methods
public class Book : EntityEventsBase, // Adding the EntityEventsBase will provide the methods to send an event.
	ISoftDelete
{
    //… other code left out for clarity
    public void AddReview(int numStars, string comment, string voterName) // The AddReview is the only way to add a Review to this Book.
    {
        if (_reviews == null)
        {
            throw new InvalidOperationException("The Reviews collection must be loaded");
        }

        _reviews.Add(new Review(numStars, comment, voterName));
        
        AddEvent(new BookReviewAddedEvent(numStars, // Adds a BookReviewAddedEvent domain event with the NumStars of the new Review
			UpdateReviewCachedValues)); // Provides the event handler a secure way to update the Review cached values
    }
    
    public void RemoveReview(int reviewId) // The RemoveReview method is the only way to remove a Review from this Book.
    {
        if (_reviews == null)
        {
            throw new InvalidOperationException("The Reviews collection must be loaded");
        }

        var localReview = _reviews.SingleOrDefault(
            x => x.ReviewId == reviewId);
        if (localReview == null)
        {
            throw new InvalidOperationException("The review was not found.");
        }
        
        _reviews.Remove(localReview);

        AddEvent(new BookReviewRemovedEvent(localReview, // Adds a BookReviewAddedEvent domain event with the review that has been deleted
			UpdateReviewCachedValues)); // The RemoveReview method is the only way to remove a Review from this Book.
    }

    private void UpdateReviewCachedValues(int reviewsCount, double reviewsAverageVotes) // This private method can be used by the event handlers to update the cached values.
    {
        ReviewsCount = reviewsCount;
        ReviewsAverageVotes = reviewsAverageVotes;
    }
}
````

To catch a change of an `Author`'s `Name`, we will use a non-DDD approach and intercept the setting of a property. This approach uses EF Core’s backing-field feature so that we can detect a change in the `Author`'s `Name`.

````c#
// Author entity sending an event when the Name property is changed
public class Author : EntityEventsBase // Adding the EntityEventsBase will provide the methods to send an event.
{
    private string _name; // The backing field for the Name property, which EF Core will read/write

    public string Name
    {
        get => _name;
        set // You make the setting public and override the setter to add the event test/send.
        {
            // If the Name has changed, and it’s not a new Author, sends a domain event
            if (value != _name && AuthorId != default)
            {
                AddEvent(new AuthorNameUpdatedEvent());
            }
            _name = value;
        }
    }
    
    //… other code left out for clarity
}
````



#### Adding code to update the cached values

You will create some event handlers to update the cached values when the appropriate domain event comes in. These event handlers will be called before `SaveChanges`/`SaveChangesAsync`, so the changes that triggered the events and the subsequent changes applied by the event handlers will be saved in the same transaction. Two styles of updating the cached values within the event handlers:

* The fast delta updates, which work with numeric changes to cached values. When the `AddReview` event is received, for example, the event handler will increment the `ReviewsCount` cache property. This option is fast, but it needs careful coding to make sure that it produces the correct result in every situation.
* The more-normal recalculate updates, in which you run a query to recalculate the cached value. This option is used to update the `AuthorsOrdered` cache property.



**UPDATING THE REVIEWS CACHED VALUES USING THE DELTA UPDATE STYLE**

Adding, updating, or removing `Reviews` causes specific events, which in turn cause an event handle linked to each event type to run.



![](./diagrams/images/15_03_updating_reviews_cached_values_using_delta_update_style.png)



````c#
// Linking ReviewAddedHandler class to the BookReviewAddedEvent
public class ReviewAddedHandler : IBeforeSaveEventHandler<BookReviewAddedEvent> // Tells the Event Runner that this event should be called when it finds a BookReviewAddedEvent
{
    public IStatusGeneric Handle(object callingEntity, BookReviewAddedEvent domainEvent) // The Event Runner provides the instance of the calling entity and the event.
    {
        var book = (Domain.Books.Book) callingEntity; // Casts the object back to its actual type of Book to make access easier

        // The first part of this calculation works out how many stars before adding the new stars.
        var totalStars = Math.Round(
            book.ReviewsAverageVotes * book.ReviewsCount) +
            domainEvent.NumStars; // Adds the star rating from the new Review
        var numReviews = book.ReviewsCount + 1; // A simple add of 1 gets the new number of Reviews.

        domainEvent.UpdateReviewCachedValues( // The entity class provides a method to update the cached values.
            numReviews, // The first parameter is the number of reviews.
            totalStars / numReviews); // The second parameter provides the new average of the NumStars.
        
        return null; // Returning null is a quick way to say that the event handler is always successful.
    }
}
````

This event handler doesn’t access the database and therefore is quick, so the overhead of updating the `ReviewsCount` and `ReviewsAverageVotes` cached values is small.



**UPDATING THE BOOK’S AUTHORS' NAME CACHED VALUE BY RECALCULATION**



![](./diagrams/images/15_04_updating_books_authors_name_cached_value_by_recalculation.png)



The following listing shows the `AuthorNameUpdatedHandler` that the `GenericEventRunner` calls when it finds the domain event that was created when an `Author`'s `Name` property was changed. This event handler loops through all the `Book`s that have that `Author` and recalculates each `Book`'s `AuthorsOrdered` cache value.

````c#
// The event handler that manages a change of an Author’s Name property
public class AuthorNameUpdatedHandler : IBeforeSaveEventHandler<AuthorNameUpdatedEvent> // Tells the Event Runner that this event should be called when it finds a AuthorNameUpdatedEvent
{
    // The event handler needs to access the database.
    private readonly BookDbContext _context;
    
    public AuthorNameUpdatedHandler(BookDbContext context)
    {
        _context = context;
    }

    // The Event Runner provides the instance of the calling entity and the event.
    public IStatusGeneric Handle(object callingEntity, AuthorNameUpdatedEvent domainEvent)
    {
        var changedAuthor = (Author) callingEntity; // Casts the object back to its actual type of Author to make access easier

        // Loops through all the books that contain the Author that has changed
        foreach (var book in _context.Set<BookAuthor>()
			.Where(x => x.AuthorId == changedAuthor.AuthorId)
            .Select(x => x.Book))
        {
            // Gets the Authors, in the correct order, linked to this Book
            var allAuthorsInOrder = _context.Books
                .Single(x => x.BookId == book.BookId)
                .AuthorsLink.OrderBy(y => y.Order)
                .Select(y => y.Author).ToList();

            // Creates a comma-delimited string with the names from the Authors in the Boo
            var newAuthorsOrdered =
                string.Join(", ",
					allAuthorsInOrder.Select(x =>
						x.AuthorId == changedAuthor.AuthorId // Returns the list of author names, but replaces the changed Author's Name with the name provided in the callingEntity parameter
						? changedAuthor.Name
						: x.Name));

            book.ResetAuthorsOrdered(newAuthorsOrdered); // Updates each Book's AuthorsOrdered property
        }
        
        return null; // Returning null is a quick way to say that the event handler is always successful.
    }
}
````

As you can see, the `Author`'s `Name` event handler is much more complex and accesses the database multiple times, which is much slower than the `AddReview`/`RemoveReview` event handler. Therefore, you need to decide whether caching this value will provide an overall performance gain. In this case, the likelihood of updating an `Author`'s `Name` is small, so on balance, it is worthwhile to cache the list of author names for a book.



#### Adding cache properties to the Book entity with concurrency handling

The best way to handle the simultaneous-updates problem is to configure the three cache values as concurrency tokens. Two simultaneous updates of a cache value will throw a `DbUpdateConcurrencyException`, which then calls a concurrency handler written to correct the cache values to the right values.



![](./diagrams/images/15_05_adding_cache_properties_to_book_entity_with_concurrency_handling.png)



**CODE TO CAPTURE ANY EXCEPTION THROWN BY SAVECHANGES/SAVECHANGESASYNC**

To capture `DbUpdateConcurrencyException`, you need to add a C# `try`/`catch` around the call to the `SaveChanges`/`SaveChangesAsync` methods. This addition allows you to call an exception handler to try to fix the problem that caused the exception or rethrow the exception if it can’t fix the problem. If your exception handler managed to fix the exception, you call `SaveChanges`/`SaveChangesAsync` again to update the database with your fix.

In this specific case, you need to consider another issue: while you were fixing the first concurrency update, another concurrency update could have happened. Sure, this scenario is rather unlikely, but you must handle it; otherwise, the second call to `SaveChanges`/`SaveChangesAsync` would fail. For this reason, you need a C# `do`/`while` outer loop to keep retrying the call to the `SaveChanges`/`SaveChangesAsync` method until it is successful or an exception that can’t be fixed occurs.



````c#
// A simplified version of the GenericEventRunner’s SaveChanges call
private IStatusGeneric<int> // The returned value is a status, with int returned from SaveChanges.
    CallSaveChangesWithExceptionHandler(DbContext context, Func<int> callBaseSaveChanges) // The base SaveChanges is provided to be called.
{
    var status = new StatusGenericHandler<int>(); // The status that will be returned
    
    do // The call to the SaveChanges is done within a do/while.
    {
        try
        {
            int numUpdated = callBaseSaveChanges(); // Calls the base SaveChanges
            // If no exception occurs, sets the status result and breaks out of the do/while
            status.SetResult(numUpdated);
            break;
        }
        catch (Exception e) // The catch caches any exceptions that SaveChanges throws.
        {
            IStatusGeneric handlerStatus = … YOUR EXCEPTION HANDLER GOES HERE; // Your exception handler is called here, and it returns null or a status.
            if (handlerStatus == null) // If the exception handler returns null, it rethrows the original exception...
            {
                throw;
            }
            status.CombineStatuses(handlerStatus); // ...otherwise, any errors from your exception handler are added to the main status.
        }
    } while (status.IsValid); // If the exception handler was successful, it loops back to try calling SaveChanges again.

    return status;
}
````



**TOP-LEVEL CONCURRENCY HANDLER THAT FINDS THE BOOK(S) THAT CAUSED THE EXCEPTION**

Handling a concurrency issue involves several common parts, so you build a top-level concurrency handler to manage those parts.



````c#
// The top-level concurrency handler containing the common exception code
public static IStatusGeneric HandleCacheValuesConcurrency(this Exception ex, DbContext context) // This extension method handles the Reviews and Author cached values concurrency issues.
{
    var dbUpdateEx = ex as DbUpdateConcurrencyException; // Casts the exception to a DbUpdateConcurrencyException
    if (dbUpdateEx == null) // If the exception isn’t a Db.UpdateConcurrencyException, we return null to say that we can't handle that exception
    {
        return null;
    }

    foreach (var entry in dbUpdateEx.Entries) // Should be only one entity, but we handle many entities in case of bulk loading
    {
        if (!(entry.Entity is Book bookBeingWrittenOut)) // Casts the entity to a Book. If it isn't a Book, we return null to say the method can't handle it.
        {
            return null;
        }

        // Reads a nontracked version of the Book from the database. (Note the IgnoreQueryFilters, because it might have been soft-deleted.)
        var bookThatCausedConcurrency = context.Set<Book>()
            .IgnoreQueryFilters()
            .AsNoTracking()
            .SingleOrDefault(p => p.BookId
				== bookBeingWrittenOut.BookId);

        // If no book was deleted, marks the current book as detached so it won't be updated
        if (bookThatCausedConcurrency == null)
        {
            entry.State = EntityState.Detached;
            continue;
        }

        var handler = new FixConcurrencyMethods(entry, context); // Creates the class containing the Reviews and AuthorsOrdered cached values

        handler.CheckFixReviewCacheValues(bookThatCausedConcurrency, bookBeingWrittenOut); // Fixes any concurrency issues with the Reviews cached values

        handler.CheckFixAuthorOrdered(bookThatCausedConcurrency, bookBeingWrittenOut); // Fixes any concurrency issues with the AuthorsOrdered cached value
    }
    
    return new StatusGenericHandler(); // Returns a valid status to say that the concurrency issue was fixed
}
````



**CONCURRENCY HANDLER FOR A PROBLEM WITH THE REVIEW’S CACHED VALUES**

The `CheckFixReviewCacheValues` concurrency handler method deals only with the Review cached values. Its job is to combine the `Review` cached values in the entity that is being written out and the `Review` cached values that have been added to the database.

This method uses the same delta update style used in the `Review` cached values event handler.



````c#
// The code to fix a concurrent update of the Review cached values
public void CheckFixReviewCacheValues( // This method handles concurrency errors in the Reviews cached values.
    Book bookThatCausedConcurrency, // This parameter is the Book from the database that caused the concurrency issue.
    Book bookBeingWrittenOut) // This parameter is the Book you were trying to update.
{
    // Holds the count and votes in the database before the events changed them
    var previousCount = (int)_entry
        .Property(nameof(Book.ReviewsCount))
        .OriginalValue;
    var previousAverageVotes = (double)_entry
        .Property(nameof(Book.ReviewsAverageVotes))
        .OriginalValue;

    // If the previous count and votes match the current database, there is no Review concurrency issue, so the method returns.
    if (previousCount == bookThatCausedConcurrency.ReviewsCount
        && previousAverageVotes == bookThatCausedConcurrency.ReviewsAverageVotes)
    {
        return;
    }

    // Works out the stars before the new update is applied
    var previousTotalStars = Math.Round(previousAverageVotes * previousCount);

    // Gets the change that the event was trying to make to the cached values
    var countChange = bookBeingWrittenOut.ReviewsCount - previousCount;
    var starsChange = Math.Round(bookBeingWrittenOut.ReviewsAverageVotes * bookBeingWrittenOut.ReviewsCount) - previousTotalStars;

    // Works out the combined change from the current book and the other updates done to the database
    var newCount = bookThatCausedConcurrency.ReviewsCount + countChange;
    var newTotalStars = Math.Round(bookThatCausedConcurrency.ReviewsAverageVotes * bookThatCausedConcurrency.ReviewsCount) + starsChange;

    // Sets the Reviews cached values with the recalculated values
    _entry.Property(nameof(Book.ReviewsCount))
        .CurrentValue = newCount;
    _entry.Property(nameof(Book.ReviewsAverageVotes))
        .CurrentValue = newCount == 0
        ? 0 : newTotalStars / newCount;

    // Sets the OriginalValues for the Review cached values to the current database
    _entry.Property(nameof(Book.ReviewsCount))
        .OriginalValue = bookThatCausedConcurrency
        .ReviewsCount;
    _entry.Property(nameof(Book.ReviewsAverageVotes))
        .OriginalValue = bookThatCausedConcurrency.ReviewsAverageVotes;
}
````



**CONCURRENCY HANDLER FOR A PROBLEM WITH THE AUTHORSSTRING CACHED VALUE**

The `CheckFixAuthorsOrdered` concurrency handler method has the same format as the `CheckFixReviewCacheValues` method, but it deals with the `AuthorsOrdered` cached value. Its job is to combine the `AuthorsOrdered` cached value in the entity that is being written out and the `AuthorsOrdered` cached value that has been added to the database.



````c#
// The code to fix a concurrent update of the AuthorsOrdered cached value
public void CheckFixAuthorsOrdered( // This method handles concurrency errors in the AuthorsOrdered cached value.
    Book bookThatCausedConcurrency, // This parameter is the Book from the database that caused the concurrency issue.
    Book bookBeingWrittenOut) // This parameter is the Book you were trying to update.
{
    // Gets the previous AuthorsOrdered string before the event updated it
    var previousAuthorsOrdered = (string)_entry
        .Property(nameof(Book.AuthorsOrdered))
        .OriginalValue;

    // If the previous AuthorsOrdered match the current database AuthorsOrdered, there is no AuthorsOrdered concurrency issue, so the method returns.
    if (previousAuthorsOrdered == bookThatCausedConcurrency.AuthorsOrdered)
    {
        return;
    }

    // Gets the AuthorIds for each Author linked to this Book in the correct order
    var allAuthorsIdsInOrder = _context.Set<Book>()
        .IgnoreQueryFilters()
        .Where(x => x.BookId == bookBeingWrittenOut.BookId)
        .Select(x => x.AuthorsLink
			.OrderBy(y => y.Order)
            .Select(y => y.AuthorId)).ToList()
        .Single();

    // Gets the Name of each Author, using the Find method
    var namesInOrder = allAuthorsIdsInOrder
        .Select(x => _context.Find<Author>(x).Name);

    // Creates a comma-delimited list of authors
    var newAuthorsOrdered = string.Join(", ", namesInOrder);

    // From this, you can set the AuthorsOrdered cached value with the combined values.
    _entry.Property(nameof(Book.AuthorsOrdered))
        .CurrentValue = newAuthorsOrdered;

    // Sets the OriginalValues for the AuthorsOrdered cached value to the current databas
    _entry.Property(nameof(Book.AuthorsOrdered))
        .OriginalValue = bookThatCausedConcurrency.AuthorsOrdered;
}
````

The important part to point out is that you must read in the `Author` entity classes by using the `Find` method because the `Author` that created the update to the `AuthorsOrdered` cached value hasn’t yet been written to the database. `Find` is the only query method that will first inspect the current application’s DbContext for tracked entities to find the entity you want. The `Find` will load the tracked entity with that `AuthorId` instead of loading the version in the database that hasn’t been updated yet.



#### Adding a checking/healing system to your event system



![](./diagrams/images/15_06_adding_checking_healing_system_to_event_system.png)



### Comparing the four performance approaches with development effort



![](./diagrams/images/15_07_comparing_four_performance_approaches_with_development_effort.png)



### Improving database scalability

One basic fact about database scalability is that the quicker you make the database accesses, the more concurrent accesses the database can handle. Reducing the number of round trips to the database also reduces the load on the database. Fortunately the default query type has loaded any collections within one database access. Also, lazy loading might feel like a great time-saver, but it adds all those individual database accesses back in, and both scalability and performance suffer.

But some large applications will have high concurrent database accesses, and you need a way out of this situation. The first, and easiest, approach is to pay for a more powerful database. If that solution isn’t going to cut it, here are some ideas to consider:

* *Split your data over multiple databases*: Sharding your data If your data is segregated in some way (if you have a financial application that many small businesses use, for example), you could spread each business’s data over a different database - that is, one database for each business. This approach is called sharding.
* *Split your database reads from your writes*: CQRS architecture Command and Query Responsibility Segregation (CQRS) architecture splits the database reads from the database writes. This approach allows you to optimize your reads and possibly use a separate database, or multiple read-only databases, on the CQRS read side.
* *Mix NoSQL and SQL databases*: Polyglot persistence The cached SQL approach makes the Book entity look like a complete definition of a book that a JSON structure would hold. With a CQRS architecture, you could have used a relational database to handle any writes, but on any write, you could build a JSON version of the book and write it to a read-side NoSQL database or multiple databases. This approach, which might provide higher read performance, is one form of polyglot persistence.



### Summary

* If you build your LINQ queries carefully and take advantage of all its features, EF Core will reward you by producing excellent SQL code.
* You can use EF Core’s `DbFunction` feature to inject a piece of SQL code held in an SQL UDF into a LINQ query. This feature allows you to tweak part of an EF Core query that’s run on the database server.
* If a database query is slow, check the SQL code that EF Core is producing. You can obtain the SQL code by looking at the `Information` logged messages that EF Core produces.
* If you feel that you can produce better SQL for a query than EF Core is producing, you can use several methods to call SQL from EF Core, or use Dapper to execute your SQL query directly.
* If all other performance-tuning approaches don’t provide the performance you need, consider altering the database structure, including adding properties to hold cached values. But be warned: you need to be careful.
* In addition to improving the time that a query takes, consider the scalability of your application—that is, supporting lots of simultaneous users. In many applications, such as ASP.NET Core, using async EF Core commands can improve scalability.





## 16. Cosmos DB, CQRS, and other database types

### Building a Command and Query Responsibility Segregation (CQRS) system



![](./diagrams/images/16_01_building_cqrs_system.png)



Because the CQRS architecture separates read and write operations, using one database for read operations and another for write operations is a logical step. The write side holds the data in a relational form, with no duplication of data (normalization) and the read side holds the data in a form that is appropriate for the user interface.

This design creates good performance gains for reads but a performance cost on writes, making the two-database CQRS architecture appropriate when your business application has more reads of the data than writes.



### The design of a two-database CQRS architecture application

The fundamental issue in building any CQRS system is making sure that any changes to the data change the associated projection in the read-side CQRS database.

Another approach is using integration events triggered by the DDD access methods. Here are some benefits of this approach:

* *More robust* - Using integration events ensures that the SQL database is updated only when the Cosmos DB database has successfully updated its database. Applying both database updates within a transaction reduces the possibility that the Cosmos DB database will get out of step with the SQL write side.
* *More obvious* - You trigger integration events inside the DDD methods that change the data. Each event tells the event handler whether it’s an `Add`, `Update`, or `Delete` (soft delete, in this case) of a `Book`. Then it’s easy to write the event handler to `Add`, `Update`, or `Delete` a `Book` projection in the Cosmos DB.
* *Simpler* - As already stated, sending integration events is much simpler than making detected changes via the `State` of the tracked entities.



![](./diagrams/images/16_02_design_of_two_database_cqrs_architecture_application.png)



#### Creating an event to trigger when the SQL Book entity changes

````c#
// The BookChangedEvent sending Add, Update, and Delete changes
public enum BookChangeTypes { Added, Updated, Deleted } // The three types of changes that need mapping to the Cosmos DB database

[RemoveDuplicateEvents] // This attribute causes the GenericEventRunner to remove duplicate events from the same Book instance.
public class BookChangedEvent : IEntityEvent // When an event is created, you must say what type of change the Book has gone through.
{
    // Used by the event handler to work out whether to add, update, or delete the CosmosBook
    public BookChangedEvent(BookChangeTypes bookChangeType)
    {
        BookChangeType = bookChangeType;
    }

    public BookChangeTypes BookChangeType { get; } // Holds the type of change for the event handler to use
}
````



#### Adding events to the Book entity send integration events

````c#
// Adding a BookUpdate to a Book’s AddPromotion method
public IStatusGeneric AddPromotion(decimal actualPrice, string promotionalText)
{
    var status = new StatusGenericHandler();
    if (string.IsNullOrWhiteSpace(promotionalText))
    {
        return status.AddError("You must provide text to go with the promotion.", nameof(PromotionalText));
    }

    ActualPrice = actualPrice;
    PromotionalText = promotionalText;
    
    if (status.IsValid) // You don’t want to trigger unnecessary updates, so you trigger only if the change was valid.
    {
        // Adds a BookChangedEvent event with the Update setting as a During (integration) event
        AddEvent(new BookChangedEvent(BookChangeTypes.Updated), EventToSend.DuringSave);
    }
    
    return status;
}
````



For the delete event, you are using a soft delete, so you capture a change to the `SoftDeleted` property via its access method. The options are

* If the `SoftDeleted` value isn’t changed, no event is sent.
* If the `SoftDeleted` value is changed to `true`, a `Deleted` event is sent.
* If the `SoftDeleted` value is changed to `false`, an `Added` event is sent.

````c#
// A change of SoftDeleted that triggers an AddBook or DeleteBook event
public void AlterSoftDelete(bool softDeleted)
{
    if (SoftDeleted != softDeleted) // You don’t trigger unnecessary updates, so you trigger only if there was a change to the SoftDeleted property.
    {
        // The type of event to send depends on the new SoftDelete setting.
        var eventType = softDeleted
            ? BookChangeTypes.Deleted
            : BookChangeTypes.Added;

        // Adds the BookChangedEvent event as a During (integration) event
        AddEvent(new BookChangedEvent(eventType), EventToSend.DuringSave);
    }
    SoftDeleted = softDeleted;
}
````



#### Using the EfCore.GenericEventRunner to override your BookDbContext

The Cached SQL approach and this CQRS approach can coexist, with each part having no knowledge of the other - another example of applying the SoC principle.



#### Creating the Cosmos entity classes and DbContext

Because adding, updating, and deleting entries in the Cosmos database use similar code, you build a service that contains three methods, one for each type of update. Putting all the update code in a service makes the event handler simple.



````c#
// An example Cosmos event handler that handles an Add event
public class BookChangeHandlerAsync : IDuringSaveEventHandlerAsync<BookChangedEvent> // Defines the class as a During (integration) event for the BookChanged event
{
    private readonly IBookToCosmosBookService _service; // This service provides the code to Add, Update, and Delete a CosmosBook.

    public BookChangeHandlerAsync(IBookToCosmosBookService service)
    {
        _service = service;
    }

    public async Task<IStatusGeneric> HandleAsync(object callingEntity, BookChangedEvent domainEvent, Guid uniqueKey) // The event handler uses async, as Cosmos DB uses async.
    {
        var bookId = ((Book)callingEntity).BookId; // Extracts the BookId from the calling entity, which is a Book
        switch (domainEvent.BookChangeType) // The BookChangeType can be added, updated, or deleted.
        {
            case BookChangeTypes.Added:
                await _service.AddCosmosBookAsync(bookId); // Calls the Add part of the service with the BookId of the SQL Book
                break;
            case BookChangeTypes.Updated:
                await _service.UpdateCosmosBookAsync(bookId); // Calls the Update part of the service with the BookId of the SQL Book
                break;
            case BookChangeTypes.Deleted:
                await _service.DeleteCosmosBookAsync(bookId); // Calls the Delete part of the service with the BookId of the SQL Book
                break;
            default:
                throw new ArgumentOutOfRangeException();
        }

        return null; // Retuning null tells the GenericEventRunner that this method is always successful.
    }
}  
````



````c#
// Creating a projection of the SQL Book and adding it to the Cosmos database
public async Task UpdateCosmosBookAsync(int bookId) // This method is called by the BookUpdated event handler with the BookId of the SQL book.
{
    // The Book App can be run without access to Cosmos DB, in which case it exits immediately.
    if (CosmosNotConfigured)
    {
        return;
    }

    var cosmosBook = await MapBookToCosmosBookAsync(bookId); // This method uses a Select method on a CosmosBook entity class.

    if (cosmosBook != null) // If the CosmosBook is successfully filled, the Cosmos update code is executed.
    {
        // Updates the CosmosBook to the cosmosContext and then calls a method to save it to the database
        _cosmosContext.Update(cosmosBook);
        await CosmosSaveChangesWithChecksAsync(WhatDoing.Updating, bookId);
    }
    else
    {
        // If the SQL book wasn’t found, we ensure that the Cosmos database version was removed.
        await DeleteCosmosBookAsync(bookId);
    }
}
````



````c#
// Part of the handling of SaveChanges exceptions with Cosmos DB
private async Task CosmosSaveChangesWithChecksAsync( // Calls SaveChanges and handles certain states
    WhatDoing whatDoing, int bookId) // The whatDoing parameter tells the code whether this is an Add, Update, or Delete.
{
    try
    {
        await _cosmosContext.SaveChangesAsync();
    }
    catch (CosmosException e) // Catches any CosmosExceptions
    {
        if (e.StatusCode == HttpStatusCode.NotFound && whatDoing == WhatDoing.Updating) // Catches an attempt to update a CosmosBook that wasn’t there
       {
            // You need to remove the attempted update; otherwise, EF Core will throw an exception.
            var updateVersion = _cosmosContext
                .Find<CosmosBook>(bookId);
            _cosmosContext.Entry(updateVersion)
                .State = EntityState.Detached;

            await AddCosmosBookAsync(bookId); // Turns the Update into an Add
        }
        else if (e.StatusCode == HttpStatusCode.NotFound && whatDoing == WhatDoing.Deleting) // Catches the state where the CosmosBook was already deleted...
        {
            //Do nothing as already deleted
        }
        else // ...otherwise, not an exception state you can handle, so rethrow the exception
        {
            throw;
        }
    }
    catch (DbUpdateException e) // If you try to add a new CosmosBook that’s already there, you get a DbUpdateException.
    {
        var cosmosException = e.InnerException as CosmosException; // The inner exception contains the CosmosException.
        if (cosmosException?.StatusCode == HttpStatusCode.Conflict && whatDoing == WhatDoing.Adding) // Catches an Add where there is already a CosmosBook with the same key
        {
            //… rest of code left out as nothing new there
        }
    }
}
````



### Understanding the structure and data of a Cosmos DB account

#### How the CosmosClass is stored in Cosmos DB

````json
// The CosmosBook data stored as JSON in Cosmos DB
{
    // The standard properties from the CosmosBook class
    "BookId": 214,
    "ActualPrice": 59.99,
    "AuthorsOrdered": "Jon P Smith",
    "EstimatedDate": true,
    "OrgPrice": 59.99,
    "PromotionalText": null,
    "PublishedOn": "2021-05-15T05:00:00+01:00",
    "ReviewsAverageVotes": 5,
    "ReviewsCount": 1,
    "Title": "Entity Framework Core in Action, Second Edition",

    // Holds the collection of Tags, which are configured as an owned type
    "Tags": [
        {
            "TagId": "Databases"
        },
        {
            "TagId": "Microsoft & .NET"
        }
    ],

    // These two properties are added to overcome some limitations in the EF Core Cosmos provider.
    "YearPublished": 2021,
    "TagsString": "| Databases | Microsoft & .NET |",

    "Discriminator": "CosmosBook", // EF Core adds the discriminator to differentiate this class from other classes saved In the same Cosmos container.

    "id": "CosmosBook|214", // The id is the database’s primary key and must be unique. This id is set by EF Core, using the EF Core designated primary key and the discriminator.

    // Cosmos-specific properties
    "_rid": "QmRlAMizcQmwAg…",
    "_self": "dbs/QmRlAA==/colls/QmRlAMizcQk=…",
    "_etag": "\"1e01b788-0000-1100-0000-5facfa2f0000\"",
    "_ts": 1605171759,
    "_attachments": "attachments/"
}
````

* The `id` key/value is the unique key used to define this data. EF Core fills the unique key with a value - by default, a combination of the `Discriminator` value and the value from the property(s) that you told EF Core is the primary key of this entity class.
* The `_etag` key/value can be used with the `UseETagConcurrency` Fluent API method to provide a concurrency token covering any change in the data.
* The `_ts` key/value contains the time of the last `Add`/`Update` in Unix format and is useful for finding when an entry last changed. The `_ts` value can be converted to C# `DateTime` format by using the `UnixDateTimeConverter` class.
* The `_rid` and `_self` key/value are unique identifiers used internally for navigation and resources.
* The `_attachments` key/value is depreciated and is there only for old systems.



### Displaying books via Cosmos DB

#### Cosmos DB differences from relational databases

**THE COSMOS DB PROVIDES ONLY ASYNC METHODS**

Because Cosmos DB uses HTTP to access databases, all the methods in the Cosmos DB .NET SDK use async/await, and there are no sync versions. EF Core does provide access to Cosmos DB via EF Core’s sync methods, such as `ToList` and `SaveChanges`, but these methods currently use the `Task`’s `Wait` method, which can have deadlock problems.



**COSMOS DIFFERENCE: THERE ARE NO DATABASE-CREATED PRIMARY KEYS**

In Cosmos and many other NoSQL databases, by default, the key for an item (item is Cosmos’s name for each JSON entry) must be generated by the software before you add an item. The key for an item must be unique, and Cosmos will reject (with the HTTP code Conflict) a new item if its key was already used. Also, after you have added an item with a key, you can’t change the key.



**COMPLEX QUERIES MAY NEED BREAKING UP**

In the filter-by-year option in the book display, the `FilterDropdownService` finds all the years when books were published. This task requires a series of steps:

1. Filter out any books that haven’t yet been published.
2. Extract the `Year` part of the `Book`'s `PublishedOn` `DateTime` property.
3. Apply the LINQ `Distinct` command to obtain the years for all the published books.
4. Order the years.

This complex query works in SQL, but Cosmos DB can’t handle it.

The strength of a Cosmos DB database is its scalability and availability, not its ability to handle complex queries.



**SKIP IS SLOW AND EXPENSIVE**

In the Book App, we used paging to allow the user to move through the books display. This type of query uses the LINQ `Skip` and `Take` methods to provide paging. The query `context.Books.Skip(100).Take(10)`, for example, would return the 101st to 111th books in a sequence. Cosmos DB can do this too, but the `Skip` part gets slower as the skip value gets bigger (another difference from relational databases) and is expensive too.



**BY DEFAULT, ALL PROPERTIES ARE INDEXED**

Cosmos DB’s default setup is to index all the key/values, included nested key values. You can change the Cosmos DB indexing policy, but "index all" is a good starting point.



#### Cosmos DB/EF Core difference: Migrating a Cosmos database

At some point, you are going to change or add properties to an entity class mapped to a Cosmos DB database. You must be careful, though; otherwise, you could break some of your existing Cosmos DB queries. This example shows what can go wrong and how to fix it:

1. You have a `CosmosBook` entity class, and you have written data to a Cosmos DB database.
2. You decide that you need an additional property called `NewProperty` of type `int` (but it could be any non-nullable type).
3. You read back old data that was added before the `NewProperty` property was added to the `CosmosBook` entity class.
4. At this point, you get an exception saying something like `object must have a value`.

Cosmos DB doesn’t mind your having different data in every item, but EF Core does. EF Core expects a `NewProperty` of type int, and it’s not there. The way around this problem is to make sure that any new properties are nullable; then reading the old data will return a `null` value for the new properties. If you want the new property to be non-nullable, start with a nullable version and then update every item in the database with a non-null value for the new property. After that, you can change the new property back to a non-nullable type, and because there is a value for that property in every item, all your queries will work.

Another point is that you can’t use the `Migrate` command to create a new Cosmos DB database, because EF Core doesn’t support migrations for a Cosmos DB database. You need to use the `EnsureCreatedAsync` method instead. The `EnsureCreatedAsync` method is normally used for unit testing, but it’s the recommended way to create a database (Cosmos DB container) when working with Cosmos DB.



#### EF Core Cosmos DB database provider limitations

* Counting the number of books in Cosmos DB is slow!
* Many database functions are not implemented.
* EF Core cannot do subqueries on a Cosmos DB database.
* There are no relationships or `Includes`.



### Was using Cosmos DB worth the effort? Yes!

#### How difficult would it be to use this two-database CQRS design in your application?

* *Detecting changes to an SQL `Book`* - This part was made easy by the use of DDD classes, as I could add an event to each access method in the `Book` entity class. If you aren’t using DDD classes, you would need to detect changes to entities during `SaveChangesAsync`.
* *Writing to the Cosmos DB database* - That part was fairly easy, with some straightforward `Add`, `Update`, and `Delete` methods.
* *Querying the Cosmos DB database* - This part took the most time, mainly because there are limitations in EF Core and in Cosmos DB.



### Summary

* A NoSQL database is designed to be high-performance in terms of speed, scalability, and availability. It achieves this performance by dropping relational-database features such as strongly linked relationships between tables.
* A CQRS architecture separates the read operations from the write operations, which allows you to improve the read side’s performance by storing the data in a form that matches the query, known as a projection.
* The Book App has been augmented by the ability to store a projection of the SQL Book on the read side of the CQRS architecture, which uses a Cosmos DB database. This approach improves performance, especially with lots of entries.
* The design used to implement the SQL/Cosmos DB CQRS architecture uses an integration event.
* The Cosmos DB database works differently from relational databases, and the process of adding this database to the Book App exposes many of these differences.
* The EF Core Cosmos DB database provider has many limitations, which are discussed and overcome in this chapter. But it is still possible to implement a useful app with Cosmos DB.
* The updated Book App shows that the Cosmos DB database can provide superior read performance over a similarly priced SQL Server database.
* The SQL/Cosmos DB CQRS design is suitable for adding to an existing application where read-side performance needs a boost, but it does add a time cost to every addition or update of data.
* Relational databases are more like one another than they are like NoSQL databases, due to the standardization of the SQL language. But you need to make some changes and checks if you change from one type of relational database to another.





## 17. Unit testing EF Core applications

* Simulating a database for unit testing
* Using the database type as your production app for unit testing
* Using an SQLite in-memory database for unit testing
* Solving the problem of one database access breaking another part of your test
* Capturing logging information while unit testing



### An introduction to the unit test setup

#### The test environment: xUnit unit test library

The static `Assert` methods are built into XUnit; the fluent validation style has to be added as an extra step.

| Type                         | Example Code                      |
| ---------------------------- | --------------------------------- |
| Fluent validation style      | `books.Count().ShouldEqual(2);`   |
| Static `Assert` method style | `Assert.Equal(2, books.Count());` |



````c#
// A simple example xUnit unit test method
[Fact] // The [Fact] attribute tells the unit test runner that this method is an xUnit unit test that should be run.
public void DemoTest() // The method must be public. It should return void or, if you're running async methods, a Task.
{
    //SETUP
    const int someValue = 1; // Typically, you put code here that sets up the data and/or environment for the unit test.

    //ATTEMPT
    var result = someValue * 2; // This line is where you run the code you want to test.

    //VERIFY
    result.ShouldEqual(2); // Here is where you put the test(s) to check that the result of your test is correct.
}
````



### Getting your application’s DbContext ready for unit testing

Before you can unit test your application’s DbContext with a database, you need to ensure that you can alter the database connection string. Otherwise, you can’t provide a different database(s) for unit testing.



#### The application’s DbContext options are provided via its constructor

If the options are provided via the application’s DbContext constructor, you don’t need any changes to the application’s DbContext to work with the unit test. You already have total control of the options given to the application’s DbContext constructor; you can change the database connection string, the type of database provider it uses, and so on.



````c#
// An application DbContext that uses a constructor for option setting
public class EfCoreContext : DbContext
{
    public EfCoreContext(
        DbContextOptions<EfCoreContext> options)
        : base(options) {}

    public DbSet<Book> Books { get; set; }
    public DbSet<Author> Authors { get; set; }

    //… rest of the class left out
}
````

For this type of application’s DbContext, the unit test can create the options variable and provide that value as a parameter in the application’s DbContext constructor.

````c#
// Creating a DbContext by providing the options via a constructor
const string connectionString // Holds the connection string for the SQL Server database
    = "Server= … content removed as too long to show";
var builder = new
    DbContextOptionsBuilder<EfCoreContext>(); // You need to create the DbContextOptionsBuilder<T> class to build the options.
builder.UseSqlServer(connectionString); // Defines that you want to use the SQL Server database provider
var options = builder.Options; // Builds the final DbContextOptions<EfCoreContext> options that the application’s DbContext needs
using (var context = new EfCoreContext(options)) // Allows you to create an instance for your unit tests
{
    //… unit test starts here
````



#### Setting an application’s DbContext options via OnConfiguring

If the database options are set in the `OnConfiguring` method inside the application’s DbContext, you must modify your application’s DbContext before you can use it in unit testing.



````c#
// A DbContext that uses the OnConfiguring method to set options
public class DbContextOnConfiguring : DbContext
{
    private const string connectionString
        = "Server=(localdb)\\... shortened to fit";

    protected override void OnConfiguring
        DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer(connectionString);
        base.OnConfiguring(optionsBuilder);
    }
    // … other code removed
}
````



The next listing shows Microsoft’s recommended way to change a DbContext that uses the `OnConfiguring` method to set up the options.

````c#
// An altered DbContext allowing the connection string to be set by the unit test
public class DbContextOnConfiguring : DbContext
{
    private const string ConnectionString
        = "Server=(localdb)\\ … shortened to fit";

    protected override void OnConfiguring(
        DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured) // Changes the OnConfigured method to run its normal setup code only if the options aren't already configured
        {
            optionsBuilder
                .UseSqlServer(ConnectionString);
        }
    }

    // Adds the same constructor-based options settings that the ASP.NET Core version has, which allows you to set any options you want
    public DbContextOnConfiguring(
        DbContextOptions<DbContextOnConfiguring> options)
        : base(options) { }

    public DbContextOnConfiguring() { } // Adds a public, parameterless constructor so that this DbContext works normally with the application
    // … other code removed
}
````

````c#
// A unit test providing a different connection string to the DbContext
const string connectionString // Holds the connection string for the database to be used for the unit test
    = "Server=(localdb)\\... shortened to fit";
var builder = new
    DbContextOptionsBuilder<DbContextOnConfiguring>();
builder.UseSqlServer(connectionString);
var options = builder.Options;
using (var context = new DbContextOnConfiguring(options)) // Provides the options to the DbContext via a new, oneparameter constructor
{
    //… unit test starts here
````



### Three ways to simulate the database when testing EF Core applications



![](./diagrams/images/17_01_three_ways_to_simulate_database_when_testing_ef_core_applications.png)



### Choosing between a production-type database and an SQLite in-memory database

You should consider using an SQLite in-memory database because it is easier for unit testing, creating a new database every time. As a result:

* The database schema is always up to date.
* The database is empty, which is a good starting point for a unit test.
* Running your unit tests in parallel works because each database is held locally in each test.
* Your unit tests will run successfully in the Test part of a DevOps pipeline without any other settings.
* Your unit tests are faster.

The downside is that the SQLite database doesn’t support and/or match some SQL commands in your production database, so your unit tests will fail or, in a few cases, give you incorrect results. If this possibility worries you, you should ignore SQLite and use the same database type as your production database for unit testing.

* *Wrong answer* - The feature might work but give you the wrong answer (which, in unit testing, is the worst result). You must be careful to run the test with a production-type database or make sure that you understand the limitations and work around them.
* *Might break* - The feature might work correctly in your unit test code, but in some cases, it might throw an exception. You can test this feature with SQLite, but you might have to change to a production-type database if a unit test fails.
* *Will break* - The feature is likely to fail when the database is set up (but might work if the SQL is basic). This result rules out using an SQLite in-memory database.



The SQL features that EF Core can control but that aren’t going to work with SQLite, because SQLite doesn’t support the feature or because SQLite uses a different format from SQL Server, MySQL, and so on

| SQL feature                   | SQLite support?                          | Breaks?      |
| ----------------------------- | ---------------------------------------- | ------------ |
| String compare and collations | Works but provides different results     | Wrong answer |
| Different schemas             | Not supported; ignores config            | Wrong answer |
| SQL column default value      | C# constants work; SQL is likely to fail | Might break  |
| SQL computed columns          | SQL is different; likely to fail         | Will break   |
| Any raw SQL                   | SQL is different; very likely to fail    | Will break   |
| SQL sequences                 | Not supported exception                  | Will break   |

Also, the following C# types aren’t natively supported by SQLite, so they could produce the wrong value:

* Decimal
* UInt64
* DateTimeOffset
* TimeSpan

EF Core will throw an exception if you sort/filter on a property that is of type Decimal while running on SQLite, for example. If you still want to unit test with SQLite, you can add a value converter to convert the `Decimal` to a `double`, but that approach might not return the exact `Decimal` value you saved to the database.



### Using a production-type database in your unit tests

#### Providing a connection string to the database to use for the unit test

To access any database, you need a connection string. You could define a connection string as a constant and use that, but as you’ll see, that approach isn’t as flexible as you’d want. Therefore, in this section you’ll mimic what ASP.NET Core does by adding to your test project a simple appsettings.json file that holds the connection string. Then you’ll use some of the .NET configuration packages to access the connection string in your application. The appsettings.json file looks something like this:

````json
{
    "ConnectionStrings": {
        "UnitTestConnection": "Server=(localdb)\\mssqllocaldb;Database=... etc"
    }
}
````

````c#
// GetConfiguration method allowing access to the appsettings.json file
public static IConfigurationRoot GetConfiguration() // Returns IConfigurationRoot, from which you can use methods such as GetConnectionString("ConnectionName") to access the configuration information
{
    var callingProjectPath =
        TestData.GetCallingAssemblyTopLevelDir(); // In the TestSupport library, a method returns the absolute path of the calling assembly's top-level directory (the assembly that you're running your tests in).
    var builder = new ConfigurationBuilder()
        .SetBasePath(callingProjectPath)
        .AddJsonFile("appsettings.json", optional: true); // Uses ASP.NET Core's ConfigurationBuilder to read that appsettings.json file. It's optional, so no error is thrown if the configuration file doesn't exist.
    return builder.Build(); // Calls the Build method, which returns the IConfigurationRoot type
}
````

You can use the `GetConfigration` method to access the connection string and then use this code to create an application's DbContext:

````c#
var config = AppSettings.GetConfiguration();
config.GetConnectionString("UnitTestConnection");
var builder = new DbContextOptionsBuilder<EfCoreContext>();
builder.UseSqlServer(connectionString);
using var context = new EfCoreContext(builder.Options);
// … rest of unit test left out
````

That code solves the problem of getting a connection string, but you still have the problem of having different databases for each test class because by default, xUnit runs unit tests in parallel.



#### Providing a database per test class to allow xUnit to run tests in parallel

Because xUnit can run each class of unit tests in parallel, using one database for all your tests wouldn’t work. Good unit tests need a known starting point and should return a known result, which rules out using one database, as different tests will simultaneously change the database.

One common solution is to have separately named databases for each unit test class or possibly each unit test method. The EfCore.TestSupport library contains methods that produce an SQL Server `DbContextOptions<T>` result in which the database name is unique to a test class or method. The first method creates a database with a name unique to this class, and the second one produces a database with a name that’s unique to that class and method.

The result of using either of these classes is that each test class or method has its own uniquely named database. So when unit tests are run in parallel, each test class has its own database to test against.



![](./diagrams/images/17_02_providing_database_per_test_class_to_allow_xunit_to_run_tests_in_parallel.png)



````c#
// CreateUniqueClassOptions extension method with a helper
public static DbContextOptions<T>
    CreateUniqueClassOptions<T>( // Returns options for an SQL Server database with a name starting with the database name in the original connection string in the appsettings.json file, but with the name of the class of the instance provided in the first parameter
    	this object callingClass, // It's expected that the object instance provided will be this—the class in which the unit test is running.
    	Action<DbContextOptionsBuilder<T>> builder = null) // This parameter allows you to add more option methods while building the options.
    where T : DbContext
{
    // Calls a private method shared between this method and the CreateUniqueMethodOptions options
    return CreateOptionWithDatabaseName<T>(callingClass, builder);
}

private static DbContextOptions<T>
    CreateOptionWithDatabaseName<T>( // Builds the SQL Server part of the options, with the correct database name
    	object callingClass,
    	Action<DbContextOptionsBuilder<T>> extraOptions,
    	string callingMember = null) // These parameters are passed from CreateUniqueClassOptions. For CreateUniqueClassOptions, the calling method is left as null.
    where T : DbContext
{
    var connectionString = callingClass
        .GetUniqueDatabaseConnectionString(callingMember); // Returns the  connection string from the appsetting.json file, but with the database name modified to use the callingClass's type name as a suffix
    var builder = new DbContextOptionsBuilder<T>(); // Sets up OptionsBuilder and creates an SQL Server database provider with the connection string
    builder.UseSqlServer(connectionString);
    builder.ApplyOtherOptionSettings(); // Calls a general method used on all your option builders, enabling sensitive logging and better error messages
    extraOptions?.Invoke(builder); // Applies any extra option methods that the caller provided
    return builder.Options; // Returns the DbContextOptions<T> to configure the application's DbContext
}
````

xUnit’s parallel-running feature has some other constraints. The use of static variables (static constants are fine) to carry information causes problems, for example, as different tests may set a static variable to different values in parallel. Nowadays, we don’t use statics much because dependency injection fills that gap. But if you use static variables in your code, you should turn off parallel running in xUnit so that you run unit tests serially.



#### Making sure that the database’s schema is up to date and the database is empty

You need a way to make sure that the database has an up-to-date schema and, at the same time, provide an empty database as a starting point for a unit test.



````c#
// The foolproof way to create a database that’s up to date and empty
[Fact]
public void TestExampleSqlDatabaseOk()
{
    //SETUP
    var options = this
        .CreateUniqueClassOptions<EfCoreContext>();
    using (var context = new EfCoreContext(options))
    {
        context.Database.EnsureDeleted(); // Deletes the current database (if present)
        context.Database.EnsureCreated(); // Creates a new database, using the configuration inside your application's DbContext
        //… rest of test removed
````



`EnsureClean` method removes the current schema of the database by deleting all the SQL indexes, constraints, tables, sequences, UDFs, and so on in the database. Then, by default, it calls the `EnsureCreated` method to return a database that has the correct schema and is empty of data.

````c#
// Using the EnsureClean method to update the database’s schema
[Fact]
public void TestExampleSqlServerEnsureClean()
{
    //SETUP
    var options = this.
        CreateUniqueClassOptions<BookDbContext>();

    using var context = new BookDbContext(options);

    context.Database.EnsureClean(); // Wipes the data and schema in the database and then calls EnsureCreated to set up the correct schema

    //… rest of test removed
}
````

`EnsureClean` approach is faster, maybe twice as fast as the `EnsureDeleted`/`EnsureCreated` version, which could make a big difference in how long your unit tests take to run. It’s also better when your database server doesn’t allow you to delete or create new databases but does allow you to read/write a database, such as when your test databases are on an SQL server on which you don’t have admin privileges.



The final approach works by applying changes to the database only within a transaction. It works because when the transaction is disposed, if you haven’t called the `transaction.Commit` method, it rolls back all the changes made in a database while the transaction is active. As a result, each unit test starts with the same data every time.

This approach is useful if you have an example database, maybe copied from the production database (with personal data anonymized, of course), that you want to test against, but you don’t want the example database to be changed.



````c#
// Using a transaction to roll back any database changes made in the test
[Fact]
public void TestUsingTransactionToRollBackChanges()
{
    //SETUP
    var builder = new
        DbContextOptionsBuilder<BookDbContext>();
    builder.UseSqlServer(_connectionString);
    using var context = new BookDbContext(builder.Options);

    using var transaction = context.Database.BeginTransaction(); // The transaction is held in a user var variable, which means that it will be disposed when the current block ends.

    //ATTEMPT
    var newBooks = BookTestData
        .CreateDummyBooks(10);
    context.AddRange(newBooks);
    context.SaveChanges();

    //VERIFY
    context.Books.Count().ShouldEqual(4+10);
} // When the unit test method ends, the transaction will be disposed and will roll back the changes made in the unit test. In this case, four books were already in the database.
````



#### Mimicking the database setup that EF Core migration would deliver

If you use a UDF in your code, for example, how do you get that SQL into your unit test database? You have three solutions:

* For simple SQL, such as a UDF, you can execute a script file after the `EnsureCreated` method.
* If you’ve added your SQL to the EF Core migration files, you should call context.Database.Migrate instead of `….EnsureCreated`.
* If you’re using script-based migrations, instead of calling `EnsureCreated`, you should execute the scripts to build the database.



````sql
-- An example SQL script file with GO at the end of each SQL command
-- Removes existing version of the UDF you want to add. If you don’t do this, the create function will fail.
IF OBJECT_ID('dbo.AuthorsStringUdf') IS NOT NULL
DROP FUNCTION dbo.AuthorsStringUdf
GO

-- ExecuteScriptFileInTransaction looks for a line starting with GO to split out each SQL command to send to the database.
CREATE FUNCTION AuthorsStringUdf (@bookId int) -- Adds a user-defined function to the database
RETURNS NVARCHAR(4000)
-- … SQL commands removed to make the example shorter
RETURN @Names
END
GO
````

````c#
// An example of applying an SQL script to a unit test database
[Fact]
public void TestApplyScriptExampleOk()
{
    var options = this
        .CreateUniqueClassOptions<EfCoreContext>();
    var filepath = TestData.GetFilePath("AddUserDefinedFunctions.sql"); // Gets the file path of the SQL script file via your TestData's GetFilePath method
    using (var context = new EfCoreContext(options))
    {
        context.Database.EnsureDeleted();
        context.Database.EnsureCreated();
        context.ExecuteScriptFileInTransaction(filepath); // Applies your script to the database by using the ExecuteScriptFileInTransaction method

        //… the rest of the unit test left out
    }
}
````



### Using an SQLite in-memory database for unit testing

SQLite has a useful option for creating an in-memory database. This option allows a unit test to create a new database in-memory, which means that it’s isolated from any other database. This approach solves all the problems of running parallel tests, having an up-to-date schema, and ensuring that the database is empty, and it’s fast.

To make an SQLite database in-memory, you need to set DataSource to "`:memory:`".

````c#
// Creating SQLlite in-memory database DbContextOptions<T> options
public static DbContextOptionsDisposable<T> CreateOptions<T> // A class containing the SQLite in-memory options, which is also disposable
    (Action<DbContextOptionsBuilder<T>> builder = null) // This parameter allows you to add more option methods while building the options.
    where T : DbContext
{
    // Gets the DbContextOptions<T> and returns a disposable version
    return new DbContextOptionsDisposable<T>
        (SetupConnectionAndBuilderOptions<T>(builder)
         	.Options);
}

private static DbContextOptionsBuilder<T>
    SetupConnectionAndBuilderOptions<T> // This method builds the SQLite in-memory options.
    (Action<DbContextOptionsBuilder<T>> applyExtraOption) // Contains any extra option methods the user provided
    where T : DbContext
{
    // Creates an SQLite connection string with the DataSource set to ":memory:"
    var connectionStringBuilder =
        new SqliteConnectionStringBuilder
    		{ DataSource = ":memory:" };
    var connectionString = connectionStringBuilder.ToString(); // Turns the SQLiteConnectionStringBuilder into a connection string
    var connection = new SqliteConnection(connectionString); // Forms an SQLite connection by using the connection string
    connection.Open(); // You must open the SQLite connection. If you don’t, the in-memory database won't work.

    // create in-memory context
    var builder = new DbContextOptionsBuilder<T>();
    builder.UseSqlite(connection); // Builds a DbContextOptions<T> with the SQLite database provider and the open connection
    builder.ApplyOtherOptionSettings(); // Calls a general method used on all your option builders, enabling sensitive logging and better error messages
    applyExtraOption?.Invoke(builder); // Adds any extra options the user added

    return builder; // Returns the DbContextOptions<T> to use in the creation of your application's DbContext
}
````

Then you can use the `SQLiteInMemory.CreateOptions` method in one of your unit tests, as shown in the next listing. You should note that in this case, you need to call only the `EnsureCreated` method, because no database currently exists.

````c#
// Using an SQLite in-memory database in an xUnit unit test
[Fact]
public void TestSQLiteOk()
{
    //SETUP
    var options = SQLiteInMemory
        .CreateOptions<EfCoreContext>(); // The SQLiteInMemory.CreateOptions provides the options for an in-memory database. The options are also IDisposable.

    using var context = new BookDbContext(options); // Uses that option to create your application's DbContext

    context.Database.EnsureCreated(); // You call context.Database.EnsureCreated to create the database.

    //ATTEMPT
    context.SeedDatabaseFourBooks(); // Runs a test method you've written that adds four test books to the database

    //VERIFY
    context.Books.Count().ShouldEqual(4); // Checks that your SeedDatabaseFourBooks worked and adds four books to the database
}
````



### Stubbing or mocking an EF Core database

* *Stubbing* a database means creating some code that replaces the current database. Stubbing works well when you are using a repository pattern.
* *Mocking* usually requires a mocking library such as Moq, which you use to take control of the class you are mocking. This task is basically impossible for EF Core; the closest library to mocking EF Core is EF Core’s in-memory database provider.

The stub provides a lot more control of the database access, and you can more easily simulate various error conditions, but it does take longer to write the mocking and unit tests.



````c#
// A unit test providing a stub instance to the BizLogic
[Fact]
public void ExampleOfStubbingOk()
{
    //SETUP
    var lineItems = new List<OrderLineItem>
    {
        new OrderLineItem {BookId = 1, NumBooks = 4}
    };
    var userId = Guid.NewGuid();
    var input = new PlaceOrderInDto(true, userId, lineItems.ToImmutableList());

    var stubDbA = new StubPlaceOrderDbAccess(); // Creates an instance of the mock database access code. This instance has numerous controls, but in this case, you use the default settings.
    var service = new PlaceOrderAction(stubDbA); // Creates your PlaceOrderAction instance, providing it a mock of the database access code

    //ATTEMPT
    service.Action(input); // Runs the PlaceOrderAction's method called Action, which takes in the input data and outputs an order

    //VERIFY
    service.Errors.Any().ShouldEqual(false); // Checks that the order placement completed successfully
    mockDbA.AddedOrder.CustomerId
        .ShouldEqual(userId); // Your mock database access code has captured the order that the PlaceOrderAction's method "wrote" to the database, so you can check whether it was formed properly.
}
````

````c#
// The stub database access code used for unit testing
public class StubPlaceOrderDbAccess
    : IPlaceOrderDbAccess // Mock MockPlaceOrderDbAccess implements the IPlaceOrderDbAccess, which allows it to replace the normal PlaceOrderDbAccess class.
{
    public ImmutableList<Book> DummyBooks { get; private set; } // Holds the dummy books that the mock uses, which can be useful if the test wants to compare the output with the dummy database

    public Order AddedOrder { get; private set; } // Will contain the Order built by the PlaceOrderAction's method

    public StubPlaceOrderDbAccess( // In this case, you set up the mock via its constructor.
        bool createLastInFuture = false, // Allows you to check that a book that hasn't been published yet won't be accepted in an order
        int? promoPriceFirstBook = null) // Allows you to add a PriceOffer to the first book so you can check that the correct price is recorded on the order
    {
        var numBooks = createLastInFuture
            ? DateTime.UtcNow.Year - EfTestData.DummyBookStartDate.Year + 2
            : 10; // Works out how to create enough books so that the last one isn't published yet
        var books = EfTestData.CreateDummyBooks(numBooks, createLastInFuture); // Creates a method to create dummy books for your test
        if (promotionPriceForFirstBook != null) // Adds a PriceOffer to the first book, if required
        {
            books.First().Promotion = new PriceOffer
            {
                NewPrice = (int) promoPriceFirstBook,
                PromotionalText = "Unit Test"
            };
        }
        DummyBooks = books.ToImmutableList();
    }

    public IDictionary<int, Book> FindBooksByIdsWithPriceOffers(IEnumerable<int> bookIds) // Called to get the books that the input selected; uses the DummyBooks generated in the constructor
    {
        return DummyBooks.AsQueryable()
            .Where(x => bookIds.Contains(x.BookId))
            .ToDictionary(key => key.BookId); // Similar code to the original, but in this case reads from the DummyBooks, not the database
    }

    // Called by the PlaceOrderAction’s method to write the Order to the database. In this case, you capture the Order so that the unit test can inspect it.
    public void Add(Order newOrder)
    {
        AddedOrder = newOrder;
    }
}
````



### Seeding a database with test data to test your code correctly

Often, a unit test needs certain data in the database before you can run a test. To test the code that handles orders for books, for example, you need some `Book`s in the database before you run the test. In cases like this one, you would add some code in the Setup stage of the unit test to add those books before you test the order code in the Verify stage.

Here are some tips on seeding a unit test database:

* It’s OK at the start to write the setup code in the unit test, but as soon as you find yourself copying that setup code, it’s time to turn that code into a method.
* Keep your test-data setup methods up to date, refactoring them as you come across different scenarios.
* Consider storing complex test data in a JSON file. I created a method to serialize data from a production system to a JSON file and have another method that will deserialize that data and write it to the database.
* The `EnsureCreated` method will also seed the database with data configured via the `HasData` configuration.



### Solving the problem of one database access breaking another stage of your test

Every tracked database query (that is, a query without the `AsNoTracking` method in it) will try to reuse the instances of any the entities already tracking by the unit test's DbContext. The effect is that any tracked query can affect any tracked query after it, so it can affect the Attempt and Verify parts of your unit test.

An example is the best way to understand this concept. Suppose that you want to test your code for adding a new `Review` to a `Book`, and you wrote the code shown in the following snippet:

````c#
var book = context.Books
    .OrderBy(x => x.BookId).Last();
book.Reviews.Add( new Review{NumStars = 5});
context.SaveChanges();
````

But there’s a problem with this code: it has a bug. The code should have `Include(b => b.Reviews)` added to the first line to ensure that the current Reviews are loaded first.

````c#
// An INCORRECT simulation of a disconnected state, with the wrong result
[Fact]
public void INCORRECTtestOfDisconnectedState()
{
    //SETUP
    var options = SqliteInMemory
        .CreateOptions<EfCoreContext>();
    using var context = new EfCoreContext(options);

    // Sets up the test database with test data consisting of four books
    context.Database.EnsureCreated();
    context.SeedDatabaseFourBooks();

    //ATTEMPT
    var book = context.Books
        .OrderBy(x => x.BookId).Last(); // Reads in the last book from your test set, which you know has two reviews
    book.Reviews.Add(new Review { NumStars = 5 }); // Adds another Review to the book, which shouldn't work but does because the seed data is still being tracked by the DbContext instance
    context.SaveChanges(); // Saves the Review to the database

    //VERIFY
    //THIS IS INCORRECT!!!!!
    context.Books
        .OrderBy(x => x.BookId).Last()
        .Reviews.Count.ShouldEqual(3); // Checks that you have three Reviews, which works, but the unit test should have failed with an exception
}
````

In fact, this unit test has two errors because of tracked entities:

* *Attempt stage* - Should have failed because the `Reviews` navigational property was `null`, but works because of relational fixup from the Setup stage
* *Verify stage* - Should fail if a `context.SaveChanges` call was left out, but works because of relational fixup from the Attempt stage



#### Test code using ChangeTracker.Clear in a disconnected state

````c#
// Using ChangeTracker.Clear to make the unit test work properly
[Fact]
public void UsingChangeTrackerClear()
{
    //SETUP
    var options = SqliteInMemory
        .CreateOptions<EfCoreContext>();
    using var context = new EfCoreContext(options);

    // Sets up the test database with test data consisting of four books
    context.Database.EnsureCreated();
    context.SeedDatabaseFourBooks();

    context.ChangeTracker.Clear(); // Calls ChangeTracker.Clear to stop tracking all entities

    //ATTEMPT
    var book = context.Books
        .OrderBy(x => x.BookId).Last(); // Reads in the last book from your test set, which you know has two reviews
    book.Reviews.Add(new Review { NumStars = 5 }); // When you try to add the new Review, EF Core throws a NullReferenceException because the Book's Review collection isn't loaded and therefore is null.
    context.SaveChanges(); // Saves the Review to the database

    //VERIFY
    context.ChangeTracker.Clear(); // Calls ChangeTracker.Clear to stop tracking all entities
    context.Books.Include(b => b.Reviews)
        .OrderBy(x => x.BookId).Last()
        .Reviews.Count.ShouldEqual(3); // Reloads the book with its Reviews to check whether there are three Reviews
}
````



#### Test code by using multiple DbContext instances in a disconnected state

````c#
// Three separate DbContext instances that make the test work properly
[Fact]
public void UsingThreeInstancesOfTheDbcontext()
{
    //SETUP
    var options = SqliteInMemory
        .CreateOptions<EfCoreContext>(); // Creates the in-memory SQLite options in the same way as the preceding example
    options.StopNextDispose();
    using (var context = new EfCoreContext(options)) // Creates the first instance of the application's DbContext
    {
        context.Database.EnsureCreated();
        context.SeedDatabaseFourBooks(); // Sets up the test database with test data consisting of four books, but this time in a separate DbContext instance
    }
    options.StopNextDispose();
    using (var context = new EfCoreContext(options))
    {
        //ATTEMPT
        var book = context.Books
            .Include(x => x.Reviews)
            .OrderBy(x => x.BookId).Last(); // Reads in the last book from your test set, which you know has two Reviews
        book.Reviews.Add(new Review { NumStars = 5 }); // When you try to add the new Review, EF Core throws a NullReferenceException because the Book's Review collection isn't loaded and therefore is null.

        context.SaveChanges(); // Calls SaveChanges to update the database
    }
    using (var context = new EfCoreContext(options))
    {
        //VERIFY
        context.Books.Include(b => b.Reviews)
            .OrderBy(x => x.BookId).Last()
            .Reviews.Count.ShouldEqual(3); // Reloads the Book with its Reviews to check whether there are three Reviews
    }
}
````



### Capturing the database commands sent to a database

Sometimes, it’s helpful to see what EF Core is doing when it accesses a real database, and EF Core provides a couple of ways to do that. Inspecting the EF Core logging from your running application is one way, but it can be hard to find the exact log among all the other logs. Another, more focused approach is to write unit tests that test specific parts of your EF Core queries by capturing SQL commands that EF Core would use to query the database.

* The `LogTo` option extension, which makes it easy to filter and capture EF Core logging
* The `ToQueryString` method, which shows the SQL generated from a LINQ query



#### Using the LogTo option extension to filter and capture EF Core logging

````c#
// Outputting logs from an xUnit test by using the LogTo method
public class TestLogTo // The class holding your unit tests of LogTo
{
    private readonly ITestOutputHelper _output; // An xUnit interface that allows output to the unit test runner

    // xUnit will inject the ITestOutputHelper via the class’s constructor.
    public TestLogTo(ITestOutputHelper output)
    {
        _output = output;
    }

    [Fact]
    public void TestLogToDemoToConsole() // This method contains a test of LogTo.
    {
        //SETUP
        var connectionString = this.GetUniqueDatabaseConnectionString(); // Provides a database connection where the database name is unique to this class
        var builder = new DbContextOptionsBuilder<BookDbContext>()
            .UseSqlServer(connectionString)
            .EnableSensitiveDataLogging() // It is good to turn on EnableSensitiveData Logging in your unit tests.
            .LogTo(_output.WriteLine); // Adds the simplest form of the LogTo method, which calls an Action<string> method
        
        using var context = new BookDbContext(builder.Options);
        // … rest of unit test left out
    }
}
````



The default has the following format:

* LINE1: `<loglevel(4 chars)> <DateTime.Now> <EventId> <Category>`
* LINE2: `<the log message>`

The following code snippet shows one of the logs in this format:

* LINE1: `warn: 10/12/2020 11:59:38.658 CoreEventId.SensitiveDataLoggingEnabledWarning[10400] (Microsoft.EntityFrameworkCore.Infrastructure)`
* LINE2: `Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.`

As well as outputting the logs, the LogTo method can filter by the following types:

* `LogLevel`, such as `LogLevel.Information` or `LogLevel.Warning`
* `EventIds`, which define a specific log output, such as `CoreEventId.ContextInitialized` and `RelationalEventId.CommandExecuted`
* Category names, which EF Core defines for commands in groups, such as `DbLoggerCategory.Database.Command.Name`
* Functions that take in the `EventId` and the `LogLevel` and return `true` for the logs you want to be output

````c#
// The LogToOptions class with all the settings for the LogTo method
public class LogToOptions
{
    // If false, your Action<string> method isn't called; defaults to true
    public bool ShowLog { get; set; } = true;

    // Only logs at or higher than the LogLevel property will be output; defaults to LogLevel.Information
    public LogLevel LogLevel { get; set; } = LogLevel.Information;

    // If not null, returns only logs with a Category name in this array; defaults to null
    public string[] OnlyShowTheseCategories { get; set; }

    // If not null, returns only logs with an EventId in this array; defaults to null
    public EventId[] OnlyShowTheseEvents { get; set; }

    // If not null, this function is called, and logs only where this function returns true are returned; defaults to null
    public Func<EventId, LogLevel, bool> FilterFunction { get; set; }

    // Controls the format of the EF Core log. The default setting does not prefix the log with extra information, such as LogLevel, DateTime, and so on.
    public DbContextLoggerOptions LoggerOptions { get; set; } = DbContextLoggerOptions.None;
}
````

````c#
// Turning off log output until the //SETUP stage of the unit test is finished
[Fact]
public void TestEfCoreLoggingCheckSqlOutputShowLog()
{
    //SETUP
    // In this case, you want to change the default LogToOptions to set the ShowLog to false.
    var logToOptions = new LogToOptions
    {
        ShowLog = false
    };
    // This method sets up the SQLite in-memory options and adds LogTo to those options.
    var options = SqliteInMemory
        .CreateOptionsWithLogTo
        <BookDbContext>(
        	_output.WriteLine, // The parameter is your Action<string> method and must be provided.
        	logToOptions); // The second parameter is optional, but in this case, you want to provide the logToOptions to control the output.

    // This setup and seed section doesn't produce any output because the ShowLog property is false.
    using var context = new BookDbContext(options);
    context.Database.EnsureCreated();
    context.SeedDatabaseFourBooks();

    //ATTEMPT
    logToOptions.ShowLog = true; // Turns on the logging output by setting the ShowLog property to true
    var book = context.Books.Count(); // This query produces one log output, which will be sent to the xUnit runner's window.

    //VERIFY
} 
````

The result is that instead of wading through the logs from creating the database and seeding the database, you see only one log output in the xUnit runner’s window, as shown in the following code snippet:

````sql
Executed DbCommand (0ms) [Parameters=[],
	CommandType='Text', CommandTimeout='30']
SELECT COUNT(*)
FROM "Books" AS "b"
WHERE NOT ("b"."SoftDeleted")
````



#### Using the ToQueryString method to show the SQL generated from a LINQ query

The logging output is great and contains lots of useful information, but if you simply want to see what your query looks like, you have a much simpler way. If you have built a database query that returns an `IQueryable` result, you can use the `ToQueryString` method. The following listing incorporates the output of the `ToQueryString` method in the test.

````c#
// A unit test containing the ToQueryString method
[Fact]
public void TestToQueryStringOnLinqQuery()
{
    //SETUP
    var options = SqliteInMemory.CreateOptions<BookDbContext>();
    using var context = new BookDbContext(options);
    context.Database.EnsureCreated();
    context.SeedDatabaseFourBooks();

    //ATTEMPT
    var query = context.Books.Select(x => x.BookId); // You provide the LINQ query without an execution part.
    var bookIds = query.ToArray(); // Then you run the LINQ query by adding ToArray on the end.

    //VERIFY
    _output.WriteLine(query.ToQueryString()); // Outputs the SQL for your LINQ query
    query.ToQueryString().ShouldEqual(
        "SELECT \"b\".\"BookId\"\r\n" +
        "FROM \"Books\" AS \"b\"\r\n" +
        "WHERE NOT (\"b\".\"SoftDeleted\")"); // Tests whether the SQL is what you expected
    bookIds.ShouldEqual(new []{1,2,3,4}); // Tests the output of the query
}
````



### Summary

* Unit testing is a way to test a unit of your code - a small piece of code that can be logically isolated in your application.
* Unit testing is a great way to catch bugs when you develop your code and, more important, when you or someone else refactors your code.
* We recommend using xUnit because it is widely used (EF Core uses xUnit and has ~70,000 tests), well supported, and fast.
* An application’s DbContext designed to work with an ASP.NET Core application is ready for unit testing, but any application’s DbContext that uses the `OnConfiguring` method to set options needs to be modified to allow unit testing.
* There are three main ways to simulate a database when unit testing, each with its own trade-offs:
  * *Using the same type of database as your production database* - This approach is the safest, but you need to deal with out-of-date database schemas and managing databases to allow parallel running of unit test classes.
  * *Using an SQLite in-memory database* - This approach is the fastest and easiest, but it doesn’t mimic every SQL feature of your production database.
  * *Stubbing the database* - When you have a repository pattern for accessing the database, such as in business logic, stubbing that repository gives you fast and comprehensive control of the data for unit testing, but it typically needs more test code to be written.
* Many unit tests need the test database to contain some data to be used in the test, so it’s worth spending time to design a suite of test methods that will create test data to use in your unit tests.
* Your unit tests might say that the code under test is correct when it’s not. This situation can happen if one section of your unit test is picking up tracked instances from a previous stage of the test. You have two ways to ensure that this problem doesn’t happen: use separate DbContext instances or use `ChangeChanger.Clear`.
* EF Core has added two methods that make capturing the SQL produced from your code much easier: the `LogTo` option to capture logging output and the `ToQueryString` method to convert LINQ queries to database commands.





## Appendix A. A brief introduction to LINQ

Language Integrated Query (LINQ)



### An introduction to the LINQ language

#### The two ways you can write LINQ queries

LINQ has two syntaxes for writing LINQ queries: the method syntax and the *query* syntax.



````c#
// Your first look at the LINQ language using the method/lambda syntax
int[] nums = new[] {1, 5, 4, 2, 3, 0}; // Creates an array of integers from 0 to 5, but in a random order

int[] result = nums // Applies LINQ commands and returns a new array of integers
    .Where(x => x > 3) // Filters out all the integers 3 and below
    .OrderBy(x => x) // Orders the numbers
    .ToArray(); // Turns the query back into an array. The result is an array of ints { 4, 5 }.
````

The *lambda* name comes from lambda syntax, introduced in C# 3. The lambda syntax allows you to write a method without all the standard method definition syntax. The `x => x > 3` part inside the `Where` method is equivalent to the following method:

````c#
private bool AnonymousFunc(int x)
{
    return x > 3;
}
````



````c#
// Your first look at the LINQ language using the query syntax
int[] nums = new[] { 1, 5, 4, 2, 3, 0}; // Creates an array of integers from 0 to 5, but in random order

IOrderedEnumerable<int> result = // The result returned here is an IOrderedEnumerable<int>.
    from num in nums // The query syntax starts with a from <item> in <collection>.
    where num > 3 // Filters out all the integers 3 and below
    orderby num // Orders the numbers
    select num; // Applies a select to choose what you want. The result is an IOrderedEnumerable<int> containing { 4, 5 }.
````



The query syntax has a feature specifically to handle the concept of precalculating values: the `let` keyword. This keyword allows you to calculate a value once and then use that value multiple times in the query, making the query more efficient.

````c#
// Using the let keyword in a LINQ query syntax
int[] nums = new[] { 1, 5, 4, 2, 3, 0 }; // Creates an array of integers from 0 to 5, but in random order

string [] numLookop = new[] {"zero","one","two","three","four","five"}; // A lookup to convert a number to its word format

IEnumerable<int> result = // The result returned here is an IEnumerable<int>.
    from num in nums // The query syntax starts with a from <item> in <collection>.
    let numString = numLookop[num] // The let syntax allows you to calculate a value once and use it multiple times in the query.
    where numString.Length > 3 // Filters out all the numbers indicating that the word is shorter than three letters
    orderby numString // Orders the number by the word form
    select num; // Applies a select to choose what you want. The result is an IEnumerable<int> containing { 5,4,3,0 }.
````

````c#
// Using the LINQ Select operator to hold a calculated value
int[] nums = new[] { 1, 5, 4, 2, 3, 0 }; // Creates an array of integers from 0 to 5, but in random order
string[] numLookop = new[] {"zero","one","two","three","four","five"}; // A lookup to convert a number to its word format

IEnumerable<int> result = nums // The result returned here is an IEnumerable<int>.
    .Select(num => new
    {
        num,
        numString = numLookop[num]
    }) // Uses an anonymous type to hold the original integer value and your numString word lookup
    .Where(r => r.numString.Length > 3) // Filters out all the numbers indicating that the word is shorter than three letters
    .OrderBy(r => r.numString) // Orders the number by the word form
    .Select(r => r.num); // Applies another Select to choose what you want. The result is an IEnumerable<int> containing { 5,4,3,0 }.
````



#### The data operations you can do with LINQ

The LINQ feature has many methods, referred to as operators.

Examples of LINQ operators, grouped by purpose

| Group          | Examples (not all operators shown)        |
| -------------- | ----------------------------------------- |
| Sorting        | `OrderBy`, `OrderByDescending`, `Reverse` |
| Filtering      | `Where`                                   |
| Select element | `First`, `FirstOrDefault`                 |
| Projection     | `Select`                                  |
| Aggregation    | `Max`, `Min`, `Sum`, `Count`, `Average`   |
| Partition      | `Skip`, `Take`                            |
| Boolean tests  | `Any`, `All`, `Contains`                  |



````c#
// A Review class and a ReviewsList variable containing two Reviews
class Review
{
    public string VoterName { get; set; }
    public int NumStars { get; set; }
    public string Comment { get; set; }
}

List<Review> ReviewsList = new List<Review>
{
    new Review
    {
        VoterName = "Jack",
        NumStars = 5,
        Comment = "A great book!"
    },
    new Review
    {
        VoterName = "Jill",
        NumStars = 1,
        Comment = "I hated it!"
    }
}; 
````



LINQ group

* Projection

  * Code using LINQ operators

    ````c#
    string[] result = ReviewsList
        .Select(p => p.VoterName)
        .ToArray();
    ````

  * Result value

    ````c#
    string[]{"Jack", "Jill"}
    ````

* Aggregation

  * Code using LINQ operators

    ````c#
    double result = ReviewsList
        .Average(p => p.NumStars);

  * Result value

    ````c#
    3 (average of 5 and 1)

* Select element

  * Code using LINQ operators

    ````c#
    string result = ReviewsList
        .First().VoterName;

  * Result value

    ````c#
    "Jack" (first voter)

* Boolean test

  * Code using LINQ operators

    ````c#
    bool result = ReviewsList
        .Any(p => p.NumStars == 1);

  * Result value

    ````c#
    true (Jill voted 1)



### Introduction to IQueryable<T> type, and why it’s useful

Another important part of LINQ is the generic interface `IQueryable<T>`. LINQ is rather special, in that whatever set of LINQ operators you provide isn’t executed straightaway but is held in a type called `IQueryable<T>`, awaiting a final command to execute it. This `IQueryable<T>` form has two benefits:

* You can split a complex LINQ query into separate parts by using the `IQueryable<T>` type.
* Instead of executing the `IQueryable<T>`'s internal form, EF Core can translate it into database access commands.



#### Splitting up a complex LINQ query by using the IQueryable<T> type

````c#
// Your method encapsulates part of your LINQ code via IQueryable<int>
public static class LinqHelpers // Extension method needs to be defined in a static class
{
    public static IQueryable<int> MyOrder( // Static method Order returns an IQueryable<int> so other extension methods can chain on
        this IQueryable<int> queryable, // Extension method’s first parameter is of IQueryable and starts with the this keyword
        bool ascending) // Provides a second parameter that allows you to change the order of the sorting
    {
        return ascending // Uses the Boolean parameter ascending to control whether you add the OrderBy or OrderByDescending LINQ operator to the IQueryable result
            ? queryable
            	.OrderBy(num => num) // Ascending parameter is true, so you add the OrderBy LINQ operator to the IQueryable input
            : queryable
                .OrderByDescending(num => num); // Ascending parameter is false, so you add the OrderByDescending LINQ operator to the IQueryable input
    }
}
````

````c#
// Using the MyOrder IQueryable<int> method in LINQ code
var numsQ = new[] { 1, 5, 4, 2, 3 }
	.AsQueryable(); // Turns an array of integers into a queryable object

var result = numsQ
    .MyOrder(true) // Calls the MyOrder IQueryable<int> method, with true, giving you an ascending sort of the data
    .Where(x => x > 3) // Filters out all the numbers 3 and below
    .ToArray(); // Executes the IQueryable and turns the result into an array. The result is an array of ints { 4, 5 }.
````

Extension methods, such as the MyOrder example, provide two useful features:

* *They make your LINQ code dynamic.* By changing the parameter into the MyOrder
  method, you can change the sort order of the final LINQ query. If you didn’t
  have that parameter, you’d need two LINQ queries - one using `OrderBy` and one using `OrderByDescending` - and then you’d have to pick which one you
  wanted to run by using an `if` statement. That approach isn’t good software practice,
  as you’d be needlessly repeating some LINQ code, such as the `Where` part.
* *They allow you to split complex queries into a series of separate extension methods that you
  can chain.* This approach makes it easier to build, test, and understand complex
  queries.

````c#
// The book list query with select, order, filter, and page Query Objects
public IQueryable<BookListDto> SortFilterPage
    (SortFilterPageOptions options)
{
    var booksQuery = _context.Books
        .AsNoTracking()
        .MapBookToDto()
        .OrderBooksBy(options.OrderByOptions)
        .FilterBooksBy(options.FilterBy,
                       options.FilterValue);

    options.SetupRestOfDto(booksQuery);

    return booksQuery.Page(options.PageNum - 1, options.PageSize);
}
````

The book list query uses both features: it allows you to change the sorting, filtering, and paging of the book list dynamically, and it hides some of the complex code behind an aptly named method that tells you what it’s doing.



#### How EF Core translates IQueryable<T> into database code

EF Core translates your LINQ code into database code that can run on the database server. It can do this because the `IQueryable<T>` type holds all the LINQ code as an expression tree, which EF Core can translate into database access code.

EF Core provides many extra extension methods to extend the LINQ operators available to you. EF Core methods add to the LINQ expression tree, such as `Include`, `ThenInclude`, and so on. Other EF methods provide async versions of the LINQ methods, such as `ToListAsync` and `LastAsync`.



### Querying an EF Core database by using LINQ



![](./diagrams/images/A1.quering_ef_core_database_by_using_linq.png)



![](./diagrams/images/A2.example_database_access.png)



These three parts of an EF Core database query are as follows:

* *Application's DbContext property access* - In your application’s DbContext, you define a property by using a `DbSet<T>` type. This type returns an `IQueryable<T>` data source to which you can add LINQ operators to create a database query.
* *LINQ operators and/or EF Core LINQ methods* - Your database LINQ query code goes here.
* *The execute command* - Commands such as `ToList` and `First` trigger EF Core to translate the LINQ commands into database access commands that are run on the database server.
