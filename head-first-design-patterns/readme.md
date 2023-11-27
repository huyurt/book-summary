# Notes of Head First Design Patterns (2nd Edition)

## 1. Welcome To Design Patterns

> Someone has already solved your problems.

### Strategy Pattern (SimUDuck App)

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



#### Designing Behaviors

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



#### Exercises

There are classes for game characters along with classes for weapon behaviors the characters can use in the game. Each character can make use of one weapon at a time, but can change weapons at any time during the game.



![](./diagrams/svg/01_04_design_puzzle_game.drawio.svg)





### Observer Pattern (Weather Monitoring App)

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



#### Meet the Observer Pattern

1. A newspaper publisher begins publishing newspapers.
2. You subscribe to a particular publisher, and every time there’s a new edition it gets delivered to you.
3. You unsubscribe when you don’t want newspapers anymore, and they stop being delivered.
4. People, hotels, airlines, and other businesses constantly subscribe and unsubscribe to the newspaper.

The Observer Pattern defines a one-to-many dependency between objects so that when one object changes state, all of its dependents are notified and updated automatically.




![](./diagrams/svg/02_02_observer_pattern.drawio.svg)



##### The Power of Loose Coupling

> Strive for loosely coupled designs between objects that interact.

Loosely coupled designs allow us to build flexible systems because they minimize the interdependency between objects.



#### Designing the Weather Station




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





### Decorator Pattern (Coffee Shop)

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



