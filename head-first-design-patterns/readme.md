# Notes of Head First Design Patterns (2nd Edition)

> Someone has already solved your problems.

## Strategy Pattern (SimUDuck App)

SimUDuck is a duck pond game. The game can show a large variety of duck species swimming and making quacking sounds. The game has built with standard OO techniques and created one Duck superclass from which all other duck types inherit.



![](./diagrams/svg/01_01_duck_superclass.drawio.svg)



The executives decided that flying ducks is just what the simulator needs. The designer thinks to add a fly() method in the Duck class and then all ducks will inherit it. But not all subclasses of Duck should fly.

The designer could always just override the fly() method in rubber duck, like he has with the quack() method. But then what happens when he adds wooden decoy ducks to the program? They aren't supposed to fly or quack.

The designer realized that inheritance probably wasn’t the answer, because the executives now want to update the product every six months. The spec will be possibly override fly() and quack() for every new Duck subclass that’s ever added to the program, forever.



![](./diagrams/svg/01_02_duck_interface.drawio.svg)



We know that not all of the subclasses should have flying or quacking behavior, so inheritance isn’t the right answer. But while having the subclasses implement Flyable and/or Quackable solves part of the problem, it completely destroys code reuse for those behaviors, so it just creates a different maintenance nightmare. Whenever you need to modify a behavior, you are often forced to track down and change it in all the different subclasses where that behavior is defined, probably introducing new bugs along the way.

> Identify the aspects of your application that vary and separate them from what stays the same.

fly() and quack() are the parts of the Duck class that vary across ducks. To separate these behaviors from the Duck class, we’ll pull both methods out of the Duck class and create a new set of classes to represent each behavior.

One of the secrets to creating maintainable OO systems is thinking about how they might change in the future.



### Designing Behaviors

> Program to an interface, not an implementation.

The Duck behaviors will live in a separate class, a class that implements a particular behavior interface. That way, the Duck classes won’t need to know any of the implementation details for their own behaviors.

Here’s the key: A Duck will now delegate its flying and quacking behaviors, instead of using quacking and flying methods defined in the Duck class (or subclass).




![](./diagrams/svg/01_03_duck_behaviours.drawio.svg)



````c#
public abstract class Duck
{
    IFlyBehavior flyBehavior;
    IQuackBehavior quackBehavior;
    
    public Duck() {}
    
    public abstract void Display();
    
    public void PerformFly()
    {
        flyBehavior.Fly();
    }
    
    public void PerformQuack()
    {
        quackBehavior.Quack();
    }
    
    public void Swim()
    {
        Console.WriteLine("All ducks float, even decoys!");
    }
    
    public void SetFlyBehavior(IFlyBehavior fb)
    {
        flyBehavior = fb;
    }
    
    public void SetQuackBehavior(IQuackBehavior qb)
    {
        quackBehavior = qb;
    }
}
````

````c#
public interface IFlyBehavior
{
    void Fly();
}

public class FlyWithWings : IFlyBehavior
{
    public void Fly()
    {
        Console.WriteLine("I'm flying!!");
    }
}

public class FlyNoWay : IFlyBehavior
{
    public void Fly()
    {
        Console.WriteLine("I can't fly");
    }
}

public class FlyRocketPowered : IFlyBehavior
{
    public void Fly()
    {
        Console.WriteLine("I'm flying with a rocket!");
    }
}
````

````c#
public interface IQuackBehavior
{
    void Quack();
}

public class Quack : IQuackBehavior
{
    public void Quack()
    {
        Console.WriteLine("Quack");
    }
}

public class MuteQuack : IQuackBehavior
{
    public void Quack()
    {
        Console.WriteLine("<< Silence >>");
    }
}

public class Squeak : IQuackBehavior
{
    public void Quack()
    {
        Console.WriteLine("Squeak");
    }
}
````

````c#
public class MallardDuck : Duck
{
    public MallardDuck()
    {
        quackBehavior = new Quack();
        flyBehavior = new FlyWithWings();
    }
    
    public void Display()
    {
        Console.WriteLine("I'm a real Mallard duck");
    }
}

public class ModelDuck : Duck
{
    public ModelDuck()
    {
        flyBehavior = new FlyNoWay();
        quackBehavior = new Quack();
    }
    
    public void Display()
    {
        Console.WriteLine("I'm a model duck");
    }
}
````

````c#
public class MiniDuckSimulator
{
    static void Main(string[] args)
    {
        Duck mallard = new MallardDuck();
        mallard.PerformQuack();
        mallard.PerformFly();

        Duck model = new ModelDuck();
        model.PerformFly();
        model.SetFlyBehavior(new FlyRocketPowered());
        model.PerformFly();
    }
}
````



When you put two classes together like this you’re using composition. Instead of inheriting their behavior, the ducks get their behavior by being composed with the right behavior object.

> Favor composition over inheritance.



* Knowing the OO basics does not make you a good OO designer.
* Good OO designs are reusable, extensible, and maintainable.
* Patterns show you how to build systems with good OO design qualities.
* Patterns are proven object-oriented experience.
* Patterns don’t give you code, they give you general solutions to design problems. You apply them to your specific application.
* Patterns aren’t invented, they are discovered.
* Most patterns and principles address issues of change in software.
* Most patterns allow some part of a system to vary independently of all other parts.
* We often try to take what varies in a system and encapsulate it.
* Patterns provide a shared language that can maximize the value of your communication with other developers.



OO Basic

* Abstraction

  Data abstraction is the process of hiding certain details and showing only essential information to the user. Abstraction can be achieved with either abstract classes or interfaces.

  ````c#
  // Abstract class
  abstract class Animal
  {
      // Abstract method (does not have a body)
      public abstract void AnimalSound();
      // Regular method
      public void Sleep()
      {
          Console.WriteLine("Zzz");
      }
  }
  
  // Derived class (inherit from Animal)
  class Cat : Animal
  {
      public override void AnimalSound()
      {
          // The body of animalSound() is provided here
          Console.WriteLine("The cat says: meow meow");
      }
  }
  ````

* Encapsulation

  Encapsulation is the concept of wrapping data into a single unit. It collects data members and member functions into a single unit called class. The purpose of encapsulation is to prevent alteration of data from outside. This data can only be accessed by getter functions of the class.

  A fully encapsulated class has getter and setter functions that are used to read and write data. This class does not allow data access directly.

  ````c#
  namespace AccessSpecifiers
  {
      class Student
      {
          public string Id { get; set; }
          public string Name { get; set; }
          public string Email { get; set; }  
      }
  }
  ````

* Polymorphism

  Polymorphism means "many forms", and it occurs when we have many classes that are related to each other by inheritance.

  ````c#
  class Animal  // Base class (parent)
  {
      public virtual void AnimalSound()
      {
          Console.WriteLine("The animal makes a sound");
      }
  }
  
  class Cat : Animal  // Derived class (child)
  {
      public override void AnimalSound()
      {
          Console.WriteLine("The cat says: meow meow");
      }
  }
  
  class Dog : Animal  // Derived class (child)
  {
      public override void AnimalSound()
      {
          Console.WriteLine("The dog says: bow wow");
      }
  }
  ````

* Inheritance

  Inherit fields and methods from one class to another. The "inheritance concept" into two categories:

  * Derived Class (child) - the class that inherits from another class
  * Base Class (parent) - the class being inherited from

  ````c#
  class Vehicle  // base class (parent)
  {
      public string brand = "Ford";  // Vehicle field
      public void Honk()             // Vehicle method
      {
          Console.WriteLine("Tuut, tuut!");
      }
  }
  
  class Car : Vehicle  // derived class (child)
  {
      public string modelName = "Mustang";  // Car field
  }
  ````

OO Principle

* Encapsulate what varies.
* Favor composition over inheritance.
* Program to interfaces, not implementations.

OO Patterns

* Strategy - defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.



### Exercises

There are classes for game characters along with classes for weapon behaviors the characters can use in the game. Each character can make use of one weapon at a time, but can change weapons at any time during the game.



![](./diagrams/svg/01_04_design_puzzle_game.drawio.svg)





## Observer Pattern (Weather Monitoring App)

The system has three components: the weather station (the physical device that acquires the actual weather data), the WeatherData object (that tracks the data coming from the Weather Station and updates the displays), and the display that shows users the current weather conditions.




![](./diagrams/svg/02_01_weather_monitoring_app.drawio.svg)



Misguided implementation of the Weather Station:

````c#
public class WeatherData
{
    // instance variable declarations
    public void MeasurementsChanged()
    {
        float temp = GetTemperature();
        float humidity = GetHumidity();
        float pressure = GetPressure();
        
        currentConditionsDisplay.Update(temp, humidity, pressure);
        statisticsDisplay.Update(temp, humidity, pressure);
        forecastDisplay.Update(temp, humidity, pressure);
    }
    
    // other WeatherData methods here
}
````



### Observer Pattern

1. A newspaper publisher begins publishing newspapers.
2. You subscribe to a particular publisher, and every time there’s a new edition it gets delivered to you.
3. You unsubscribe when you don’t want newspapers anymore, and they stop being delivered.
4. People, hotels, airlines, and other businesses constantly subscribe and unsubscribe to the newspaper.

The Observer Pattern defines a one-to-many dependency between objects so that when one object changes state, all of its dependents are notified and updated automatically.




![](./diagrams/svg/02_02_observer_pattern.drawio.svg)



#### Loose Coupling

> Strive for loosely coupled designs between objects that interact.

Loosely coupled designs allow us to build flexible systems because they minimize the interdependency between objects.



### Designing the Weather Station




![](./diagrams/svg/02_03_weather_monitoring_observer_pattern.drawio.svg)



````c#
public interface ISubject
{
    void RegisterObserver(Observer o);
    void RemoveObserver(Observer o);
    void NotifyObservers();
}

public interface IObserver
{
    void Update(float temperature, float humidity, float pressure);
}

public interface IDisplayElement
{
    void Display();
}
````

````c#
public class WeatherData : ISubject
{
    private List<Observer> observers;
    private float temperature;
    private float humidity;
    private float pressure;
    
    public WeatherData()
    {
        observers = new List<Observer>();
    }
    
    public void RegisterObserver(Observer o)
    {
        observers.Add(o);
    }
    
    public void RemoveObserver(Observer o)
    {
        observer.Remove(o);
    }
    
    public void NotifyObservers()
    {
        foreach(Observer observer in observers)
        {
            observer.Update(temp, humidity, pressure);
        }
    }
    
    public void MeasurementsChanged()
    {
        NotifyObservers();
    }
    
    public void SetMeasurements(float temperature, float humidity, float pressure)
    {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        MeasurementsChanged();
    }
    
    // other WeatherData methods here
}
````

````c#
public class CurrentConditionsDisplay : IObserver, IDisplayElement
{
    private float temperature;
    private float humadity;
    private WeatherData weatherData;
    
    public CurrentConditionsDisplay(WeatherData weatherData)
    {
        this.weatherData = weatherData;
        weatherData.RegisterObserver(this);
    }
    
    public void Update(float temperature, float humidity, float pressure)
    {
        this.temperature = temperature;
        this.humidity = humidity;
        Display();
    }
    
    public void Display()
    {
        Console.WriteLine("Current conditions: " + temperature + "C degrees and " + humidity + "% humidity");
    }
}
````

````c#
public class WeatherStation
{
    static void Main(string[] args)
    {
        WeatherData weatherData = new WeatherData();
        
        CurrentConditionsDisplay currentDisplay = new CurrentConditionsDisplay(weatherData);
        StatisticsDisplay statisticsDisplay = new StatisticsDisplay(weatherData);
        ForecastDisplay forecastDisplay = new ForecastDisplay(weatherData);
        
        weatherData.setMeasurements(80, 65, 30.4f);
        weatherData.setMeasurements(82, 70, 29.2f);
        weatherData.setMeasurements(78, 90, 29.2f);
    }
}
````



* The Observer Pattern defines a one-to-many relationship between objects.
* Subjects update Observers using a common interface.
* Observers of any concrete type can participate in the pattern as long as they implement the Observer interface.
* Observers are loosely coupled in that the Subject knows nothing about them, other than that they implement the Observer interface.
* You can push or pull data from the Subject when using the pattern (pull is considered more "correct").
* The Observer Pattern is related to the Publish/Subscribe Pattern, which is for more complex situations with multiple Subjects and/or multiple message types.



OO Principles

* Strive for loosely coupled designs between objects that interact.

OO Patterns

* Observer - defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.





## Decorator Pattern (Coffee Shop)

The coffee shop has grown so quickly, they are scrambling to update their ordering systems to match their beverage offerings.

When they first went into business they designed their classes like this...

In addition to coffee, you can also ask for several condiments like steamed milk, soy, and mocha, and have it all topped off with whipped
milk. The coffee shop charges for each condiment.



![](./diagrams/svg/03_01_coffee_shop_first_class_design.drawio.svg)



First improvement is below. What requirements or other factors might change that will impact this design?

- Price changes for condiments will force us to alter existing code.
- New condiments will force us to add new methods and alter the cost method in the superclass.
- We may have new beverages. For some of these beverages (iced tea?), the condiments may not be appropriate, yet the Tea subclass will still inherit methods like hasWhip().
- What if a customer wants a double mocha?



![](./diagrams/svg/03_02_coffee_shop_class_design_improvement.drawio.svg)



### The Open-Closed Principle

> Classes should be open for extension, but closed for modification.

* Come on in; we’re *open*. Feel free to extend our classes with any new behavior you like. If your needs or requirements change, just go ahead and make your own extensions.
* Sorry, we’re *closed*. We spent a lot of time getting this code correct and bug free, so we can’t let you alter the existing code. It must remain closed to modification. If you don’t like it, you can speak to the manager.

Allow classes to be easily extended to incorporate new behavior without modifying existing code.



### Decorator Pattern

We have seen that representing beverage and condiments with inheritance has not worked out very well. We get class explosions and rigid designs, or we add functionality to the base class that isn’t appropriate for some of the subclasses.

We will start with a beverage and decorate it with the condiments at runtime. For example, if the customer wants a Dark Roast with Mocha and Whip, then we will:

1. Start with a DarkRoast object.
2. Decorate it with a Mocha object.
3. Decorate it with a Whip object.
4. Call the cost() method and rely on delegation to add up the condiment costs.

Think of decorator objects as *wrappers*.



![](./diagrams/svg/03_03_coffee_shop_object_wrapping.drawio.svg)



* Decorators have the same supertype as the objects they decorate.
* You can use one or more decorators to wrap an object.
* Given that the decorator has the same supertype as the object it decorates, we can pass around a decorated object in place of the original (wrapped) object.
* The decorator adds its own behavior before and/or after delegating to the object it decorates to do the rest of the job.
* Objects can be decorated at any time, so we can decorate objects dynamically at runtime with as many decorators as we like.

Decorator Pattern attaches additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.



![](./diagrams/svg/03_04_decorator_pattern.drawio.svg)



### Designing Beverages



![](./diagrams/svg/03_05_coffees_decorator_pattern.drawio.svg)



````c#
public abstract class Beverage
{
    private protected string description = "Unknown Beverage";
    
    public virtual string GetDescription()
    {
        return description;
    }
    
    public abstract decimal Cost();
}

public abstract class CondimentDecorator : Beverage
{
    private protected Beverage beverage;
    
    public abstract override string GetDescription();
}
````

````C#
public class Espresso : Beverage
{
    public Espresso()
    {
        description = "Espresso";
    }
    
    public override decimal Cost()
    {
        return 1.99M;
    }
}

public class HouseBlend : Beverage
{
    public HouseBlend()
    {
        description = "House Blend Coffee";
    }
    
    public override decimal Cost()
    {
        return .89M;
    }
}
````

````c#
public class Mocha : CondimentDecorator
{
    public Mocha(Beverage beverage)
    {
        this.beverage = beverage;
    }
    
    public override string GetDescription()
    {
        return beverage.GetDescription() + ", Mocha";
    }
    
    public override decimal Cost()
    {
        return beverage.Cost() + .20M;
    }
}

public class Whip : CondimentDecorator
{
    public Mocha(Beverage beverage)
    {
        this.beverage = beverage;
    }
    
    public override string GetDescription()
    {
        return beverage.GetDescription() + ", Whip";
    }
    
    public override decimal Cost()
    {
        return beverage.Cost() + .10M;
    }
}
````

````c#
public class CoffeeShop
{
    static void Main(string[] args)
    {
        Beverage beverage = new Espresso();
        beverage = new Mocha(beverage);
        beverage = new Mocha(beverage);
        beverage = new Whip(beverage);
        
        Console.Write($"{beverage.GetDescription()} ${beverage.Cost()}"); // Espresso, Mocha, Mocha, Whip $2,49
    }
}
````



* Inheritance is one form of extension, but not necessarily the best way to achieve flexibility in our designs.
* In our designs we should allow behavior to be extended without
  the need to modify existing code.
* Composition and delegation can often be used to add new
  behaviors at runtime.
* The Decorator Pattern provides an alternative to subclassing for
  extending behavior.
* The Decorator Pattern involves a set of decorator classes that are used to wrap concrete components.
* Decorator classes mirror the type of the components they decorate.
* Decorators change the behavior of their components by adding new functionality before and/or after (or even in place of) method calls to the component.
* You can wrap a component with any number of decorators.
* Decorators are typically transparent to the client of the component, unless the client is relying on the component’s concrete type.
* Decorators can result in many small objects in our design, and overuse can be complex.



OO Principles

* Classes should be open for extension but closed for modification.

OO Patterns

* Decorator - Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.





## Factory Pattern (Pizza Store)

By coding to an interface, you know you can insulate yourself from many of the changes that might happen to a system down the road. If your code is written to an interface, then it will work with any new classes implementing that interface through polymorphism. However, when you have code that makes use of lots of concrete classes, you’re looking for trouble because that code may have to be changed as new concrete classes are added. So your code will not be “closed for modification.” To extend your code with new concrete types, you’ll have to reopen it.

So what can you do? Our first principle deals with change and guides us to identify the aspects that vary and separate them from what stays the same.



![](./diagrams/svg/04_01_pizzastore_simple_factory_pattern.drawio.svg)



````c#
private Pizza OrderPizza(string type)
{
    Pizza pizza;
    
    // As the pizza selection changes over time, you’ll have to modify this code over and over.
    if (type == "cheese")
    {
        pizza = new CheesePizza();
    }
    else if (type == "italian")
    {
        pizza = new ItalianPizza();
    }
    else if (type == "clam")
    {
        pizza = new ClamPizza();
    }
    else if (type == "veggie")
    {
        pizza = new VeggiePizza();
    }
    
    // This is what we expect to stay the same.
    // For the most part, preparing, cooking, and packaging a pizza has remained the same for years and years.
    // So, we don’t expect this code to change, just the pizzas it operates on.
    pizza.Prepare();
    pizza.Bake();
    pizza.Cut();
    pizza.Box();

    return pizza;
}
````

````c#
public class SimpleStaticPizzaFactory
{
    // Defining a simple factory as a static method is a common technique and is often called a static factory.
    // Why use a static method? Because you don’t need to instantiate an object to make use of the create method.
    // But it also has the disadvantage that you can’t subclass and change the behavior of the create method.
    public static Pizza CreatePizza(string type)
    {
        Pizza pizza = null;
        
        if (type == "cheese")
        {
            pizza = new CheesePizza();
        }
        else if (type == "italian")
        {
            pizza = new ItalianPizza();
        }
        else if (type == "clam")
        {
            pizza = new ClamPizza();
        }
        else if (type == "veggie")
        {
            pizza = new VeggiePizza();
        }

        return pizza;
    }
}
````

````c#
public class PizzaStore
{
    private readonly SimplePizzaFactory _factory;
    
    public PizzaStore(SimplePizzaFactory factory)
    {
        _factory = factory;
    }
    
    public Pizza OrderPizza(string type)
    {
        Pizza pizza = _factory.CreatePizza(type);
    
        pizza.Prepare();
        pizza.Bake();
        pizza.Cut();
        pizza.Box();
        
        return pizza;
    }
}
````



### Franchising The Pizza Store

Each franchise might want to offer different styles of pizzas (New York, Chicago, California, etc.), depending on where the franchise store is located.

Starting to employ their own home-grown procedures for the rest of the process: they’d bake things a little differently, they’d forget to cut the pizza, and they’d use third-party boxes.
You see that what you’d really like to do is create a framework that ties the store and the pizza creation together, yet still allows things to remain flexible.



![](./diagrams/svg/04_02_franchising_pizzastore_factory_method_pattern.drawio.svg)



````c#
public abstract class Pizza
{
    public string Name;
    public string Dough;
    public string Sauce;
    public List<string> Toppings = new();

    public void Prepare()
    {
        Console.WriteLine("Preparing " + Name);
        Console.WriteLine("Tossing dough...");
        Console.WriteLine("Adding sauce...");
        Console.WriteLine("Adding toppings:");
        foreach(string topping in Toppings)
        {
            Console.WriteLine("	" + topping);
        }
    }

    public virtual void Bake()
    {
        Console.WriteLine("Bake for 25 minutes at 350.");
    }

    public virtual void Cut()
    {
        Console.WriteLine("Cutting the pizza into diagonal slices.");
    }

    public virtual void Box()
    {
        Console.WriteLine("Place pizza in official PizzaStore box.");
    }

    public string GetName()
    {
        return Name;
    }
}

public class NYStyleCheesePizza : Pizza
{
    public NYStyleCheesePizza()
    {
        Name = "NY Style Sauce and Cheese Pizza";
        Dough = "Thin Crust Dough";
        Sauce = "Marinara Sauce";
        
        Toppings.Add("Grated Reggiano Cheese");
    }
}

public class ChicagoStyleCheesePizza : Pizza
{
    public ChicagoStyleCheesePizza()
    {
        Name = "Chicago Style Deep Dish Cheese Pizza";
        Dough = "Extra Thick Crust Dough";
        Sauce = "Plum Tomato Sauce";
        
        Toppings.Add("Shredded Mozzarella Cheese");
    }
    
    public override void Cut()
    {
        Console.WriteLine("Cutting the pizza into square slices.");
    }
}
````

````c#
public abstract class PizzaStore
{
    public Pizza OrderPizza(string type)
    {
        Pizza pizza = CreatePizza(type);
        
        pizza.Prepare();
        pizza.Bake();
        pizza.Cut();
        pizza.Box();
        
        return pizza;
    }
    
    protected abstract Pizza CreatePizza(string type);
}

public class NYPizzaStore : PizzaStore
{
    protected override Pizza CreatePizza(string type)
    {
        if (type == "cheese")
        {
            return new NYStyleCheesePizza();
        }
        else if (type == "veggie")
        {
            return new NYStyleVeggiePizza();
        }
        else if (type == "clam")
        {
            return new NYStyleClamPizza();
        }
        else
        {
            return null;
        }
    }
}
````

````c#
public class PizzaTest
{
    static void Main(string[] args)
    {
        PizzaStore nyStore = new NYPizzaStore();
        PizzaStore chicagoStore = new ChicagoStore();
        
        Pizza pizza = nyStore.OrderPizza("cheese");
        Console.WriteLine("Ethan ordered a " + pizza.GetName());
        
        Pizza pizza = chicagoStore.OrderPizza("cheese");
        Console.WriteLine("Joel ordered a " + pizza.GetName());
    }
}
````



#### Factory Method Pattern

The Factory Method Pattern defines an interface for creating an object, but lets subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.



![](./diagrams/svg/04_03_factory_method_pattern.drawio.svg)



The Factory Method Pattern gives us a way to encapsulate the instantiations of concrete types. Looking at the class diagram, you can see that the abstract Creator class gives you an interface with a method for creating objects, also known as the "factory method". Any other methods implemented in the abstract Creator are written to operate on products produced by the factory method. Only subclasses actually implement the factory method and create products.
You will often hear developers say, "the Factory Method pattern lets subclasses decide which class to instantiate". Because the Creator class is written without knowledge of the actual products that will be created, we say "decide" not because the pattern allows subclasses themselves to decide, because the decision actually comes down to which subclass is used to create the product.

