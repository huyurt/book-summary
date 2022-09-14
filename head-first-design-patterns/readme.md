# Notes of Head First Design Patterns (2nd Edition)

## 1. Welcome To Design Patterns

> Someone has already solved your problems.

### SimUDuck App

SimUDuck is a successful duck pond simulation game. The game can show a large variety of duck species swimming and making quacking sounds. The designers of the system used standard OO techniques and created one Duck superclass from which all other duck types inherit.

![](./diagrams/svg/01_duck_superclass.drawio.svg)



The executives decided that flying ducks is just what the simulator needs.

Joe thinks to add a fly() method in the Duck class and then all ducks will inherit it. But not all subclasses of Duck should fly.