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



#### ONE-TO-ONE RELATIONSHIP: PRICEOFFER TO A BOOK

A book can have a promotional price applied to it with an optional row in the Price- Offer, which is an example of a one-to-one relationship. (Technically, the relationship is one-to-zero-or-one, but EF Core handles it the same way.)

To calculate the final price of the book, you need to check for a row in the PriceOffer table that’s linked to the Books via a foreign key. If such a row is found, the NewPrice supersedes the price for the original book, and the PromotionalText is shown onscreen, as in this example:

​	*$40 $30 Our summertime price special, for this week only!*



![](./diagrams/svg/02_01_one_to_one_price_offer_to_book.drawio.svg)



#### ONE-TO-MANY RELATIONSHIP: REVIEWS TO A BOOK

You want to allow customers to review a book; they can give a book a star rating and optionally leave a comment. Because a book may have no reviews or many (unlimited) reviews, you need to create a table to hold that data. In this example, you’ll call the table Review. The Books table has a one-to-many relationship to the Review table.

In the Summary display, you need to count the number of reviews and work out the average star rating to show a summary. Here’s a typical onscreen display you might produce from this one-to-many relationship:

​	*Votes 4.5 by 2 customers*



![](./diagrams/svg/02_02_one_to_many_reviews_to_book.drawio.svg)



#### MANY-TO-MANY RELATIONSHIP: MANUALLY CONFIGURED

Books can be written by one or more authors, and an author may write one or more books. The link between the Books and Authors tables is called a many-to-many relationship, which in this case needs a linking table to achieve this relationship.
In this case, you create your own linking table with an Order value in it because the names of the authors in a book must be displayed in a specific order.

A typical onscreen display from the many-to-many relationship would look like this:

​	*by Dino Esposito, Andrea Saltarello*



![](./diagrams/svg/02_03_many_to_many_authors_books.drawio.svg)



#### MANY-TO-MANY RELATIONSHIP: AUTOCONFIGURED BY EF CORE

Books can be tagged with different categories - such as Microsoft .NET, Linux, Web, and so on - to help the customer to find a book on the topic they are interested in. A category might be applied to multiple books, and a book might have one or more categories, so a many-to-many linking table is needed. But unlike in the previous BookAuthor linking table, the tags don’t have to be ordered, which makes the linking table simpler.

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





## 3. Changing the database content

* Creating a new row in a database table
* Updating existing rows in a database table for two types of applications
* Updating entities with one-to-one, one-to-many, and many-to-many relationships
* Deleting single entities, and entities with relationships, from a database



