# Python OOP — The Complete Deep-Understanding Course

**How to use this document:** Read one module at a time. Don't skip the "Practice" sections — do them before reading the next module. Every concept follows the same skeleton: **Problem → Concept → Tiny Example → Internal Working → Real World → Wrong vs Right → Comparisons → Memory Rule → Practice.**

---

# MODULE 1 — OOP FOUNDATIONS

## 1.1 What is OOP and why does it exist?

### The problem (without OOP)
Imagine you're building a system to manage bank accounts using only plain functions and variables:

```python
name1 = "Alice"
balance1 = 1000

name2 = "Bob"
balance2 = 500

def deposit(balance, amount):
    return balance + amount

balance1 = deposit(balance1, 200)
```

This works for 2 accounts. Now imagine 10,000 accounts. You'd need 10,000 pairs of variables, and every function needs to know which variables belong together. Nothing *enforces* that `name1` and `balance1` belong to the same account — they're just two separate variables that happen to share a naming pattern. Nothing stops you from writing `deposit(balance1, amount)` but accidentally using `name2` somewhere else.

**The core problem: data and the logic that operates on that data are disconnected.**

### The concept — Object-Oriented Programming
OOP's solution: bundle data (**attributes**) and the behavior that acts on that data (**methods**) into a single unit called an **object**. Instead of "a name variable and a balance variable that we hope stay in sync," you get one `Account` object that owns both, guaranteed.

- **What is it?** A programming style where you model your program as a collection of interacting objects, each owning its own data and behavior.
- **Why does it exist?** To keep related data and logic together, so code mirrors how we naturally think about real things (a car, an account, an employee) instead of scattered variables and functions.
- **What problem does it solve?** Disorganized data, duplicated logic, and fragile code where nothing enforces which data belongs together.
- **When to use it?** When your program models distinct "things" with their own state and behavior that interact with each other.
- **When NOT to use it?** For small scripts, simple data transformations, or one-off calculations — wrapping a single function in a class adds ceremony with no benefit.

## 1.2 Procedural programming vs OOP

| Aspect | Procedural | OOP |
|---|---|---|
| Organizing unit | Functions | Objects (data + functions together) |
| Data | Passed around between functions | Owned by objects |
| Reuse | Copy-paste or generic functions | Inheritance, composition |
| Data safety | Any function can touch any variable | Encapsulation restricts access |
| Mental model | "Steps to follow" | "Things that interact" |

Neither is "better" universally — procedural is fine for a script that reads a CSV and prints stats. OOP shines when you have many related, evolving pieces of state (accounts, orders, users) with rules about how they change.

## 1.3 Real-world objects and classes

Think about a **car**. Every real car has:
- **State** (attributes): color, current speed, fuel level, model
- **Behavior** (methods): accelerate, brake, refuel

A **class** is the blueprint — "Car" as a concept, describing what every car *will have*. An **object** is an actual car built from that blueprint — your neighbor's red Honda with 30% fuel.

**Memory rule:** *Class = blueprint. Object = the actual thing built from the blueprint.*

**Mental picture:** A cookie cutter (class) vs the cookies it stamps out (objects). One cutter, many cookies — each cookie can have different icing (different attribute values), but they all share the same shape (same methods, same structure).

## 1.4 Class, Object, Attributes, Methods — tiny example

```python
class Car:                          # the blueprint
    def __init__(self, color, fuel):
        self.color = color           # attribute
        self.fuel = fuel             # attribute

    def drive(self):                 # method
        self.fuel -= 10
        print(f"{self.color} car driving, fuel left: {self.fuel}")

car1 = Car("Red", 100)   # object 1
car2 = Car("Blue", 80)   # object 2

car1.drive()   # Red car driving, fuel left: 90
car2.drive()   # Blue car driving, fuel left: 70
```

**Line by line:**
- `class Car:` — declares the blueprint named `Car`.
- `def __init__(self, color, fuel):` — the special method that runs automatically when a new `Car` object is created; it sets up initial state.
- `self.color = color` — stores the value onto *this specific* object.
- `car1 = Car("Red", 100)` — this is where Python actually builds an object and runs `__init__` on it.
- `car1.drive()` — calls `drive`, but Python secretly passes `car1` in as `self`.

Notice: `car1.fuel` and `car2.fuel` change independently. That's the whole point — each object has its **own** copy of the attributes.

## 1.5 State and Behavior

- **State** = the current values of an object's attributes at any moment (`car1.fuel` is currently `90`).
- **Behavior** = what the object can *do*, defined by its methods (`drive()`).

An object is really just "state + the behavior that knows how to change that state correctly." This pairing is why OOP prevents the disconnect from Section 1.1 — the `fuel` attribute can only be changed through `drive()` (or whatever methods you define), not by some unrelated stray function.

## 1.6 Creating objects and understanding `self`

When you write `car1 = Car("Red", 100)`, three things happen internally:
1. Python creates a new, empty object in memory.
2. Python calls `Car.__init__(that_new_object, "Red", 100)`.
3. The variable `car1` is bound to that object.

**`self` is just a name for "the specific object this method is currently running on."** It's not magic syntax — it's the first parameter, and Python fills it in automatically when you call a method via an object.

```python
car1.drive()
# is literally equivalent to:
Car.drive(car1)
```

Try running that second line yourself — it works identically. This is the single most important internal fact to internalize about Python OOP: **methods are just functions stored on the class, and dot-calling passes the object in as the first argument.**

### Wrong vs Right
```python
# WRONG — forgetting self, or using a different name inconsistently
class Car:
    def drive():          # missing self
        print("driving")

car = Car()
car.drive()   # TypeError: drive() takes 0 positional arguments but 1 was given
```
Python still passes `car` in automatically — but `drive()` wasn't written to accept it, so it crashes. `self` isn't optional decoration; it's how the method receives the object it's operating on.

```python
# RIGHT
class Car:
    def drive(self):
        print("driving")
```

## 1.7 Object identity

Every object has a unique identity (its memory address, exposed via `id()`), separate from its value.

```python
a = Car("Red", 100)
b = Car("Red", 100)
print(a == b)        # False by default — different objects
print(a is b)        # False — different identities
print(a.color == b.color)  # True — same attribute values
```

`a` and `b` look identical but are **two separate objects** — changing `a.fuel` never affects `b.fuel`. `is` checks identity (same object in memory); `==` checks equality (by default, same as `is`, unless you customize it — more in Module 7).

**Memory rule:** *`is` asks "are you the same object?" `==` asks "do you have the same value?"*

---

## Module 1 — Comparison Table

| Concept A | Concept B | Key difference |
|---|---|---|
| Class | Object | Class = blueprint (one). Object = instance built from it (many). |
| `is` | `==` | `is` = same identity. `==` = same value (customizable). |
| Attribute | Method | Attribute = data stored on object. Method = function stored on class, acting on object. |

## Module 1 — Practice

**Recall (answer without scrolling up):**
1. In your own words, what is the difference between a class and an object?
2. What does `self` actually represent, mechanically?
3. Why does OOP exist — what specific problem does bundling data + behavior solve?

**Recognition:**
```python
p1 = Person("Sam")
p2 = Person("Sam")
print(p1 == p2)
```
What will this print, and why (assuming `Person` defines nothing special)?

**Coding challenge (Level 1–2):**
Create a class `Book` with attributes `title` and `pages_read` (starting at 0), and a method `read(self, pages)` that increases `pages_read` by that many pages and prints the new total. Create two `Book` objects and prove their `pages_read` values are independent.

Send me your answers when ready, or say "next" if you want me to keep building the full file through Module 2 before you attempt these — your call.

---

# MODULE 2 — CLASSES AND OBJECTS (DEEPER)

## 2.1 Instance attributes vs Class attributes

### The problem
Suppose every `Employee` should belong to the same company name, "TechCorp". If you set it as an instance attribute on every object, you're duplicating the same value hundreds of times, and if the company rebrands, you'd have to update every single object.

### The concept
- **Instance attribute**: belongs to one specific object, set (usually) inside `__init__` via `self.x = ...`. Each object has its own copy.
- **Class attribute**: belongs to the class itself, shared by all objects, unless an object overrides it locally.

```python
class Employee:
    company = "TechCorp"          # class attribute — shared by ALL employees

    def __init__(self, name, salary):
        self.name = name           # instance attribute — unique per object
        self.salary = salary       # instance attribute

e1 = Employee("Asha", 50000)
e2 = Employee("Ravi", 60000)

print(e1.company)   # TechCorp
print(e2.company)   # TechCorp

Employee.company = "TechCorp Global"   # change once...
print(e1.company)   # TechCorp Global  -- both see the update
print(e2.company)   # TechCorp Global
```

### Internal working
When you access `e1.company`, Python first looks in `e1.__dict__` (the object's own storage). It's not there, so Python falls back to `Employee.__dict__` (the class's storage) — and finds it there. This fallback chain is called **attribute lookup**, and it's central to how Python OOP works (we go deeper in Module 9).

A subtlety that trips people up:
```python
e1.company = "Solo Corp"     # this creates a NEW instance attribute on e1 only
print(e1.company)             # Solo Corp  (found on e1 now)
print(e2.company)             # TechCorp Global (unaffected — still uses class attribute)
```
Assigning `e1.company = ...` does **not** modify the class attribute — it shadows it with a new instance attribute, local to `e1`. Only `Employee.company = ...` changes the shared value.

**Memory rule:** *Class attribute = shared shelf everyone reads from. Instance attribute = your own private drawer. Writing through the object creates your own drawer instead of touching the shared shelf.*

### When to use each
- Instance attribute: data that differs per object (name, salary, balance).
- Class attribute: data or defaults shared by every object of that class (company name, a counter of how many objects exist, a constant like `interest_rate`).

## 2.2 The Constructor — `__init__`

`__init__` is not technically "the constructor" in the strictest sense (that's `__new__`, Module 11) — it's the **initializer**, called right after the object is created, to set up its starting state.

```python
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height
```

### Object creation lifecycle
1. `Rectangle(3, 4)` is called.
2. Python calls `__new__` to actually allocate the object (usually inherited from `object`, you rarely touch this).
3. Python calls `__init__(new_object, 3, 4)` to fill in its attributes.
4. The fully-initialized object is returned and bound to your variable.

### Wrong vs Right
```python
# WRONG — forgetting to store the parameter
class Rectangle:
    def __init__(self, width, height):
        width = width      # does nothing useful — local variable, discarded
        self.height = height

r = Rectangle(3, 4)
print(r.width)   # AttributeError: no attribute 'width'
```
```python
# RIGHT
class Rectangle:
    def __init__(self, width, height):
        self.width = width   # must attach to self to persist beyond __init__
        self.height = height
```

## 2.3 Instance methods, Class methods, Static methods

### The problem
Not every method needs the full object. Sometimes you want a method that:
(a) operates on one specific object's data → **instance method** (the normal case)
(b) operates on the class itself, e.g. to create objects in an alternate way, or touch class-level data → **class method**
(c) is just logically related to the class but needs no object *or* class data at all → **static method**

### The concept & tiny examples

```python
class Pizza:
    count = 0                       # class attribute: tracks how many pizzas made

    def __init__(self, toppings):
        self.toppings = toppings
        Pizza.count += 1

    def describe(self):                        # INSTANCE METHOD
        return f"Pizza with {self.toppings}"

    @classmethod
    def margherita(cls):                        # CLASS METHOD
        return cls(["cheese", "tomato"])         # alternate constructor

    @staticmethod
    def is_valid_topping(topping):               # STATIC METHOD
        return topping in ["cheese", "pepperoni", "tomato", "olive"]

p1 = Pizza(["cheese"])
p2 = Pizza.margherita()          # built via the class method
print(Pizza.count)                # 2
print(Pizza.is_valid_topping("olive"))  # True
```

- **Instance method**: takes `self`, can read/write that specific object's attributes.
- **Class method**: takes `cls` (the class itself, not an object), typically used for alternate constructors or class-wide operations. Called via `@classmethod`.
- **Static method**: takes neither `self` nor `cls` — it's just a regular function that happens to live inside the class because it's conceptually related. Called via `@staticmethod`.

### Comparison table

| | Instance method | Class method | Static method |
|---|---|---|---|
| First param | `self` (the object) | `cls` (the class) | none |
| Can access instance data | Yes | No | No |
| Can access/modify class data | Yes (via `self.__class__` or the class name) | Yes | No |
| Typical use | Normal behavior | Alternate constructors, class-wide logic | Utility/helper logic related to the class |

**Memory rule:** *Instance method needs "this object." Class method needs "the blueprint itself." Static method needs neither — it's just grouped here for organization.*

### Common mistake
Using `@staticmethod` when you actually needed `self` or `cls` — e.g. writing a "helper" that secretly needs to read an attribute, then hacking around it with global variables. If a method needs *any* object or class data, it should not be static.

## 2.4 How objects store data — `__dict__` preview

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
print(p.__dict__)   # {'x': 1, 'y': 2}
```
Every ordinary object keeps its instance attributes in a dictionary. `p.x` is really just sugar for looking up `'x'` in `p.__dict__` (with a fallback to the class if not found there). This single fact demystifies a huge amount of "magic" in Python OOP — we'll build on it heavily in Module 9.

---

## Module 2 — Practice

**Recall:**
1. If you assign `obj.class_attr = value` through an instance, does it change the class attribute for everyone, or only that object? Why?
2. When would you reach for `@classmethod` instead of `__init__` directly?
3. Give one real example where a `@staticmethod` makes sense inside a class.

**Debugging:**
```python
class Counter:
    total = 0
    def __init__(self):
        total += 1     # bug here

c1 = Counter()
```
What's wrong, and what's the fix?

**Design (Level 6):** You're modeling a `Student` class. `school_name` is the same for every student in the batch. `roll_number` and `name` differ per student. Decide which should be class attributes vs instance attributes, and write the class.

---

# MODULE 3 — ENCAPSULATION

## 3.1 The problem — unrestricted access

```python
class BankAccount:
    def __init__(self, balance):
        self.balance = balance

acc = BankAccount(1000)
acc.balance = -5000     # nothing stops this!
print(acc.balance)      # -5000, an impossible bank balance
```
Anyone, anywhere in the codebase, can directly set `acc.balance` to anything — negative numbers, strings, whatever. There is no rule enforcement. This is the exact "disconnect between data and the logic that should protect it" problem from Module 1, but at the level of a single attribute.

## 3.2 The concept — Encapsulation

**What is it?** Restricting direct access to an object's internal data, forcing changes to go through controlled methods that can validate/protect that data.

**Why does it exist?** So invariants (rules that must always hold, like "balance can't go negative") can't be silently broken from outside the class.

**How does Python implement it?** Python doesn't have true "private" like Java — it uses **naming conventions** the community and tooling respect:

| Prefix | Meaning | Enforcement |
|---|---|---|
| `name` | Public | None — freely accessible |
| `_name` | Protected (convention: "internal, but subclasses may use it") | None — just a signal to other devs |
| `__name` | Private (convention: "don't touch from outside") | Name mangling (see below) — makes accidental access harder, not impossible |

```python
class BankAccount:
    def __init__(self, balance):
        self.__balance = balance          # private attribute

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self.__balance += amount

    def get_balance(self):
        return self.__balance

acc = BankAccount(1000)
acc.deposit(500)
print(acc.get_balance())     # 1500
acc.deposit(-100)             # ValueError raised — invalid state prevented
```

### Internal working — Name mangling
`self.__balance` inside the class is secretly rewritten by Python to `self._BankAccount__balance`.

```python
print(acc._BankAccount__balance)   # 1500 -- still technically reachable!
print(acc.__balance)                # AttributeError -- this exact name doesn't exist
```
So `__name` is not unbreakable security — it's a strong deterrent against *accidental* external access and against name clashes in inheritance, not a security wall. **Memory rule:** *`__name` doesn't hide data, it renames it to make you type an ugly, obviously-wrong-looking name to reach it on purpose.*

## 3.3 Getters, Setters, and `@property`

Writing `get_balance()` / `set_balance()` methods everywhere works but is verbose and un-Pythonic. Python's idiomatic tool is `@property`, which lets you write code that *looks* like plain attribute access but secretly runs validation.

```python
class BankAccount:
    def __init__(self, balance):
        self.__balance = balance

    @property
    def balance(self):                 # GETTER
        return self.__balance

    @balance.setter
    def balance(self, value):          # SETTER
        if value < 0:
            raise ValueError("Balance can't be negative")
        self.__balance = value

    @balance.deleter
    def balance(self):                 # DELETER
        print("Closing account, clearing balance")
        del self.__balance

acc = BankAccount(1000)
print(acc.balance)      # calls the getter -- looks like plain attribute access
acc.balance = 2000       # calls the setter -- validated!
acc.balance = -50        # ValueError raised
del acc.balance           # calls the deleter
```

**Key insight:** From the *outside*, `acc.balance` looks exactly like accessing a plain attribute — no parentheses, no `get_`/`set_` calls. But internally, every read/write is quietly routed through your validation logic. This is Python's version of encapsulation: hide the mechanism, keep the interface simple.

### Wrong vs Right
```python
# WRONG — Java-style getters/setters (works, but not idiomatic Python)
acc.set_balance(acc.get_balance() + 100)
```
```python
# RIGHT — Pythonic property-based access
acc.balance += 100    # reads via getter, validates via setter, all invisibly
```

## 3.4 Why encapsulation matters — tying it back

Real-world analogy: a car's engine is encapsulated behind the accelerator pedal. You don't reach in and manually adjust fuel injection — you press the pedal, and internal mechanisms (which you don't need to understand) handle it safely. Encapsulation gives your classes the same "safe interface, protected internals" property.

**Common mistake:** Making everything private "just in case," even data that has no real invariant to protect. This adds boilerplate with no benefit. Only encapsulate what actually needs protecting or computing.

---

## Module 3 — Comparison Table

| Encapsulation vs Abstraction | |
|---|---|
| Encapsulation | **Hides data**, controls how it's accessed/modified (the "how"). |
| Abstraction | **Hides complexity**, exposes only what's necessary (the "what"). Covered fully in Module 6. |

Quick intuition: encapsulation is about *protecting state*; abstraction is about *simplifying interfaces*. They're often used together but solve different problems.

## Module 3 — Practice

**Recall:**
1. Does `__name` make an attribute truly private in Python? What actually happens to it?
2. Why would you use `@property` instead of a plain public attribute?

**Recognition:** What OOP concept is this code demonstrating, and why?
```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius

    @property
    def fahrenheit(self):
        return self._celsius * 9/5 + 32
```

**Coding challenge:** Build a `Person` class with a `age` property that raises `ValueError` if set to a negative number, and returns `"unknown"` when the value hasn't been set. Test that assigning `-5` fails correctly.

---

# MODULE 4 — INHERITANCE

## 4.1 The problem — duplicated classes

```python
class Dog:
    def __init__(self, name):
        self.name = name
    def eat(self):
        print(f"{self.name} is eating")
    def bark(self):
        print(f"{self.name} says Woof")

class Cat:
    def __init__(self, name):
        self.name = name           # duplicated
    def eat(self):
        print(f"{self.name} is eating")   # duplicated
    def meow(self):
        print(f"{self.name} says Meow")
```
`__init__` and `eat` are identical in both classes. If you need to fix a bug in `eat`, you must remember to fix it in every animal class. This duplication is exactly what inheritance eliminates.

## 4.2 The concept — Inheritance

**What is it?** A mechanism where a class (child/derived/subclass) automatically gets the attributes and methods of another class (parent/base/superclass), and can add or override its own.

**Why does it exist?** To share common structure/behavior once, in one place, and let specific classes only define what's *different* about them.

```python
class Animal:                         # parent / base class
    def __init__(self, name):
        self.name = name
    def eat(self):
        print(f"{self.name} is eating")

class Dog(Animal):                    # child / derived class
    def bark(self):
        print(f"{self.name} says Woof")

class Cat(Animal):
    def meow(self):
        print(f"{self.name} says Meow")

d = Dog("Rex")
d.eat()    # inherited from Animal: "Rex is eating"
d.bark()   # defined in Dog: "Rex says Woof"
```

**"is-a" relationship:** Inheritance should model "a Dog **is an** Animal." If that sentence doesn't sound true and natural, inheritance is probably the wrong tool (composition, Module 8, might fit better).

## 4.3 Method Overriding and `super()`

### The problem
Sometimes a child needs to *change* inherited behavior, not just add new behavior.

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def make_sound(self):
        print(f"{self.name} makes a generic sound")

class Dog(Animal):
    def make_sound(self):                    # OVERRIDING the parent's version
        print(f"{self.name} barks")

Dog("Rex").make_sound()   # "Rex barks" -- Dog's version wins
```
This is **method overriding**: the child redefines a method with the same name, and Python uses the child's version when called on a child object.

### `super()` — calling the parent's version anyway
Sometimes you want to *extend* the parent's behavior, not fully replace it:

```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

class Manager(Employee):
    def __init__(self, name, salary, team_size):
        super().__init__(name, salary)    # let Employee handle name/salary
        self.team_size = team_size        # Manager adds its own attribute

m = Manager("Sara", 90000, 5)
print(m.name, m.salary, m.team_size)   # Sara 90000 5
```
Without `super().__init__(...)`, you'd have to duplicate `self.name = name; self.salary = salary` inside `Manager` too — the same duplication problem inheritance was meant to solve.

**Memory rule:** *`super()` = "go ask my parent to do its part first, then I'll add mine."*

### Wrong vs Right
```python
# WRONG — forgetting to call super().__init__, parent's setup never runs
class Manager(Employee):
    def __init__(self, name, salary, team_size):
        self.team_size = team_size   # name and salary never get set!

m = Manager("Sara", 90000, 5)
print(m.name)   # AttributeError
```
```python
# RIGHT
class Manager(Employee):
    def __init__(self, name, salary, team_size):
        super().__init__(name, salary)
        self.team_size = team_size
```

## 4.4 Types of Inheritance

```python
class A: pass
class B(A): pass          # SINGLE inheritance: B <- A
class C(B): pass          # MULTILEVEL: C <- B <- A

class Base: pass
class Child1(Base): pass  # HIERARCHICAL: many children, one parent
class Child2(Base): pass

class X: pass
class Y: pass
class Z(X, Y): pass        # MULTIPLE inheritance: Z inherits from both X and Y

# HYBRID: any combination of the above patterns mixed together
```

## 4.5 Method Resolution Order (MRO) and the Diamond Problem

### The problem
```python
class A:
    def greet(self):
        print("Hello from A")

class B(A):
    def greet(self):
        print("Hello from B")

class C(A):
    def greet(self):
        print("Hello from C")

class D(B, C):    # D inherits from BOTH B and C, which both inherit from A
    pass

D().greet()   # which greet() runs -- B's or C's?
```
This is the classic **diamond problem**: `D` has two paths up to `A` (through `B` and through `C`), and both `B` and `C` override `greet`. Which one wins?

### The concept — MRO
Python resolves this deterministically using the **C3 linearization algorithm**, and you can see the exact order yourself:
```python
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
```
Python checks `D` first, then `B`, then `C`, then `A`, then `object`. Since `B` comes before `C`, `D().greet()` prints `"Hello from B"`. The rule in plain English: **left-to-right, depth-first, but a class never appears before its own subclasses** — MRO guarantees a consistent, predictable order rather than ambiguity.

**Memory rule:** *MRO = the exact queue Python follows when looking for a method, and `ClassName.__mro__` always shows you that queue directly — never guess, just check it.*

## 4.6 Real-world example

```python
class Vehicle:
    def __init__(self, brand):
        self.brand = brand
    def start_engine(self):
        print(f"{self.brand} engine starting")

class ElectricVehicle(Vehicle):
    def start_engine(self):
        print(f"{self.brand} silently powers on (electric)")

class Car(Vehicle):
    pass

class Tesla(ElectricVehicle):
    pass
```
`Tesla` is an `ElectricVehicle`, which is a `Vehicle` — multilevel inheritance modeling a genuine real-world "is-a" chain.

---

## Module 4 — Comparison Table

| Inheritance vs Composition | |
|---|---|
| Inheritance | "is-a" — reuse by extending a class |
| Composition | "has-a" — reuse by containing another object (Module 8) |

## Module 4 — Practice

**Recall:**
1. What does `super()` actually do, mechanically?
2. In your own words, describe the diamond problem and how Python resolves it.

**Output prediction:**
```python
class A:
    def show(self):
        print("A")
class B(A):
    def show(self):
        super().show()
        print("B")
B().show()
```
What prints, in what order, and why?

**Design:** Model `Shape` (base), with `Circle` and `Rectangle` as children, each overriding an `area()` method that `Shape` defines with a placeholder. What should `Shape.area()` do by default? (Hint: this question foreshadows Module 6 — Abstraction.)

---

# MODULE 5 — POLYMORPHISM

## 5.1 The problem

```python
def make_it_speak(animal):
    if isinstance(animal, Dog):
        animal.bark()
    elif isinstance(animal, Cat):
        animal.meow()
    elif isinstance(animal, Bird):
        animal.tweet()
    # ...every new animal type requires editing this function again
```
This function must be modified every time a new animal type is added — it's tightly coupled to every specific type, violating the idea that adding new behavior shouldn't require rewriting existing, working code (this connects directly to the Open/Closed Principle in Module 12).

## 5.2 The concept — Polymorphism

**What is it?** "Many forms" — the ability to call the *same* method name on different object types and have each respond in its own appropriate way, without the caller needing to know which exact type it's dealing with.

```python
class Dog:
    def make_sound(self):
        print("Woof")

class Cat:
    def make_sound(self):
        print("Meow")

class Bird:
    def make_sound(self):
        print("Tweet")

def make_it_speak(animal):
    animal.make_sound()          # works for ANY object with make_sound()

for a in [Dog(), Cat(), Bird()]:
    make_it_speak(a)
# Woof
# Meow
# Tweet
```
`make_it_speak` never checks types. It just trusts that whatever is passed in has a `make_sound()` method, and calls it. Adding a `Fish` class tomorrow requires **zero changes** to `make_it_speak`.

## 5.3 Duck Typing

**"If it walks like a duck and quacks like a duck, it's a duck."** Python doesn't check an object's declared type before calling a method — it just tries to call it. If the object has that method, it works, regardless of its class or inheritance hierarchy.

```python
class Duck:
    def make_sound(self):
        print("Quack")

class Robot:
    def make_sound(self):     # not related to Duck at all, no shared parent
        print("Beep-sound-imitation")

for thing in [Duck(), Robot()]:
    thing.make_sound()    # works for both -- Python doesn't care about the class
```
This is more flexible (and more "Pythonic") than languages that require explicit interfaces before this works.

## 5.4 Operator Overloading — polymorphism on operators

```python
class Money:
    def __init__(self, amount):
        self.amount = amount
    def __add__(self, other):              # overloads the + operator
        return Money(self.amount + other.amount)
    def __repr__(self):
        return f"Money({self.amount})"

m1 = Money(100)
m2 = Money(50)
print(m1 + m2)     # Money(150) -- '+' now means something sensible for Money
```
`+` behaves differently depending on the object type — that's `+` being polymorphic. (Full dunder method coverage is Module 7.)

## 5.5 Why Python doesn't support traditional method overloading

In Java, you can define multiple methods with the *same name* but different parameter lists (`add(int, int)` vs `add(double, double)`), and the compiler picks the right one. Python doesn't do this:

```python
class Calc:
    def add(self, a, b):
        return a + b
    def add(self, a, b, c):     # this just REPLACES the previous add — doesn't overload it
        return a + b + c

Calc().add(1, 2)   # TypeError: missing 1 required positional argument
```
Only the *last* defined `add` exists — Python classes are just dictionaries of names to functions, and defining `add` twice simply overwrites the first entry. The idiomatic Python workaround is default arguments or `*args`:
```python
class Calc:
    def add(self, a, b, c=0):
        return a + b + c
```
This is a classic tricky interview question: *"Why doesn't Python support method overloading?"* — Answer: because a class body is executed top-to-bottom like any code block, and a later `def` with the same name simply rebinds that name, overwriting the earlier function object.

---

## Module 5 — Practice

**Recall:** Explain duck typing in one sentence, without using the word "duck."

**Recognition:** What concept is at play?
```python
for shape in [Circle(2), Square(3), Triangle(3,4,5)]:
    print(shape.area())
```

**Coding challenge:** Write a `Shape` family (`Circle`, `Square`) each with their own `area()`, then write one function `total_area(shapes)` that sums areas across a mixed list — without any `isinstance` checks.

---

# MODULE 6 — ABSTRACTION

## 6.1 The problem

Continuing from Module 4's design question: if `Shape.area()` has some arbitrary default body (like `return 0`), nothing stops someone from creating a bare `Shape()` object directly and calling `.area()` on it, getting a meaningless `0` — even though "a generic Shape" isn't a real, usable thing. You want to **force** every concrete shape to provide its own `area()`, and **prevent** anyone from instantiating the vague, incomplete `Shape` on its own.

## 6.2 The concept — Abstraction & Abstract Base Classes

**What is it?** Defining a class that specifies *what* methods subclasses must implement, without providing (or without requiring) an implementation itself, and preventing that incomplete class from being instantiated directly.

**Why does it exist?** To enforce a contract: "every subclass of `Shape` MUST implement `area()`," catching the mistake at object-creation time rather than silently returning wrong data later.

```python
from abc import ABC, abstractmethod

class Shape(ABC):                     # inherits from ABC -- marks it abstract
    @abstractmethod
    def area(self):
        pass                            # no implementation -- subclasses MUST provide one

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self):
        return 3.14159 * self.radius ** 2

s = Shape()          # TypeError: Can't instantiate abstract class Shape with abstract method area
c = Circle(3)
print(c.area())       # 28.27...
```
If `Circle` forgot to implement `area()`, trying `Circle(3)` would *also* raise `TypeError` — Python actively checks this at instantiation time, not just as a convention.

### Abstract classes can still have concrete (implemented) methods
```python
class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

    def describe(self):                     # CONCRETE method — shared, implemented
        return f"This shape has an area of {self.area()}"
```
`describe()` works for every subclass automatically, and it can even call the not-yet-implemented `area()` — because by the time `describe()` actually runs on a real object, that object's concrete `area()` exists.

## 6.3 Abstract Base Class vs Interface vs Protocol

| | Abstract Base Class (`ABC`) | Protocol (`typing.Protocol`) |
|---|---|---|
| Relationship required | Must explicitly inherit from it | No inheritance needed — "structural typing": if it has the right methods, it qualifies |
| Enforcement | Enforced at instantiation (`TypeError` if incomplete) | Only enforced by type checkers (like mypy), not at runtime |
| Use case | You control the class hierarchy, want a real "is-a" contract | You want duck-typing with static-type-checking support |

```python
from typing import Protocol

class SoundMaker(Protocol):
    def make_sound(self) -> None: ...

def trigger(sm: SoundMaker):
    sm.make_sound()
# Any object with a make_sound() method satisfies this "interface" -- no inheritance needed
```

**Memory rule:** *ABC = "you must formally sign the contract (inherit) to prove you comply." Protocol = "if you happen to already do what's needed, that's good enough."*

### When abstraction should be used
Use it when you're designing a **family** of related classes and want to guarantee every member implements certain core behavior — e.g., every `PaymentMethod` must have `pay()`, every `Shape` must have `area()`. Don't use it for a single, standalone class with no planned subclasses — that's unnecessary ceremony.

---

## Module 6 — Practice

**Recall:** Why does trying to instantiate an abstract class raise an error, and at what point does that error happen?

**Debugging:**
```python
from abc import ABC, abstractmethod

class Payment(ABC):
    @abstractmethod
    def pay(self, amount): pass

class CreditCard(Payment):
    def charge(self, amount):     # bug: wrong method name
        print(f"Charging {amount}")

cc = CreditCard(100)
```
What error will this raise, and why?

**Design:** Design an abstract `Notifier` class with an abstract `send(message)` method, then implement `EmailNotifier` and `SMSNotifier` concretely.

---

# MODULE 7 — PYTHON SPECIAL (DUNDER) METHODS

## 7.1 The problem

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
print(p)                # <__main__.Point object at 0x7f...> -- useless
p2 = Point(1, 2)
print(p == p2)            # False -- even though they look "equal"
```
By default, Python objects print as unhelpful memory addresses, and `==` only checks identity. Dunder ("double underscore") methods let you plug your class into Python's built-in syntax and behavior (printing, comparison, arithmetic, iteration, etc.) so your objects behave the way you'd naturally expect.

## 7.2 The concept

Dunder methods are hooks. When you write `print(p)`, Python doesn't guess how to display `p` — it calls `p.__str__()` internally. When you write `p1 + p2`, Python calls `p1.__add__(p2)`. You're not calling these directly; **Python calls them for you** in response to normal syntax.

### `__str__` vs `__repr__`
```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __str__(self):
        return f"({self.x}, {self.y})"          # for humans -- print()
    def __repr__(self):
        return f"Point(x={self.x}, y={self.y})"  # for developers -- debugging, repr()

p = Point(1, 2)
print(p)        # (1, 2)               <- uses __str__
p                # Point(x=1, y=2)      <- uses __repr__ (in REPL/debugger)
```
**Memory rule:** *`__str__` = "readable for users." `__repr__` = "unambiguous for developers," ideally something you could paste back into Python to recreate the object.* If you only define one, define `__repr__` — Python falls back to it for `print()` too if `__str__` is missing.

### `__eq__`, `__ne__`, and ordering (`__lt__`, `__gt__`, `__le__`, `__ge__`)
```python
class Money:
    def __init__(self, amount):
        self.amount = amount
    def __eq__(self, other):
        return self.amount == other.amount
    def __lt__(self, other):
        return self.amount < other.amount

m1, m2 = Money(100), Money(50)
print(m1 == m2)   # False -- now compares VALUE, not identity
print(m1 < m2)     # False
print(sorted([m1, m2], key=lambda m: m.amount))   # sorting still works either way
```
Once `__eq__` and `__lt__` are defined, `sorted()`, `max()`, `min()`, and comparison operators all "just work" on your objects.

### `__len__`, `__contains__`, `__getitem__`, `__setitem__` — container-like behavior
```python
class Playlist:
    def __init__(self):
        self.songs = []
    def add(self, song):
        self.songs.append(song)
    def __len__(self):
        return len(self.songs)
    def __contains__(self, song):
        return song in self.songs
    def __getitem__(self, index):
        return self.songs[index]
    def __setitem__(self, index, value):
        self.songs[index] = value

pl = Playlist()
pl.add("Song A")
pl.add("Song B")
print(len(pl))            # 2               -- via __len__
print("Song A" in pl)     # True             -- via __contains__
print(pl[0])                # "Song A"        -- via __getitem__
pl[0] = "New Song"          # via __setitem__
```
Your custom object now behaves like a list in these respects — `len()`, `in`, and indexing all work naturally.

### `__iter__` and `__next__` — making objects loopable
```python
class Countdown:
    def __init__(self, start):
        self.current = start
    def __iter__(self):
        return self
    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

for num in Countdown(3):
    print(num)   # 3, 2, 1
```
`for x in obj` internally calls `iter(obj)` (which calls `__iter__`), then repeatedly calls `next()` (which calls `__next__`) until `StopIteration` is raised.

### `__call__` — making objects callable like functions
```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor
    def __call__(self, value):
        return value * self.factor

double = Multiplier(2)
print(double(5))     # 10 -- calling the OBJECT like a function
```
Use case: objects that need to remember configuration (`factor`) but be *used* like a plain function elsewhere (common in decorators, callbacks).

### `__enter__` and `__exit__` — context managers (`with` statement)
```python
class FileOpener:
    def __init__(self, filename):
        self.filename = filename
    def __enter__(self):
        self.file = open(self.filename, 'w')
        return self.file
    def __exit__(self, exc_type, exc_value, traceback):
        self.file.close()
        print("File closed automatically")

with FileOpener("test.txt") as f:
    f.write("hello")
# "File closed automatically" prints -- guaranteed cleanup, even if an error occurred inside the block
```
`__exit__` runs even if an exception happens inside the `with` block — this is how resource cleanup (files, DB connections, locks) is guaranteed.

### `__new__` (preview — full detail in Module 11)
`__new__` actually *creates* the object; `__init__` just configures it afterward. You rarely override `__new__` except for advanced cases like Singletons (Module 13) or immutable types.

## 7.3 When each should and should NOT be used

| Method | Use when | Avoid when |
|---|---|---|
| `__eq__` | Objects should compare by value | Identity comparison is actually what you want |
| `__add__`/operators | The operation genuinely maps to domain meaning (Money + Money) | Overloading purely for cleverness, hurting readability |
| `__len__`/`__getitem__` | Your class truly behaves like a collection | Forcing container-behavior onto something that isn't really a collection |
| `__call__` | Object represents a reusable, configured "action" | Just to be clever instead of a normal method |
| `__new__` | Controlling object creation itself (singletons, immutability) | Normal initialization — that's `__init__`'s job |

**Common mistake:** overloading `__eq__` but forgetting `__hash__` — by default, defining `__eq__` makes your objects unhashable (can't be used in sets/dict keys) unless you also define `__hash__`.

---

## Module 7 — Practice

**Recall:** What's the practical difference between `__str__` and `__repr__`, and which one should you always try to implement first?

**Output prediction:**
```python
class Box:
    def __init__(self, items):
        self.items = items
    def __len__(self):
        return len(self.items)

b = Box([1,2,3])
if b:
    print("Box has items")
```
Will this print? (Hint: think about how truthiness relates to `__len__` and `__bool__`.)

**Coding challenge:** Implement a `Vector` class with `__add__`, `__sub__`, `__eq__`, and `__repr__`, so `Vector(1,2) + Vector(3,4)` gives `Vector(4,6)`.

---

# MODULE 8 — COMPOSITION AND RELATIONSHIPS

## 8.1 The problem with over-using inheritance

```python
class Engine:
    def start(self):
        print("Engine starting")

class Car(Engine):     # "a Car IS AN Engine"?? that sentence is false!
    pass
```
A car doesn't "is-a" an engine — a car **has an** engine. Using inheritance here is a modeling mistake: it exposes all of `Engine`'s internals directly on `Car`, and locks `Car` into a rigid relationship it doesn't actually have.

## 8.2 The concept — Composition ("has-a")

**What is it?** Building complex objects by including other objects as attributes, rather than inheriting from them.

```python
class Engine:
    def start(self):
        print("Engine starting")

class Car:
    def __init__(self):
        self.engine = Engine()      # Car HAS AN Engine
    def start(self):
        self.engine.start()          # delegate to the contained object
        print("Car is ready to drive")

c = Car()
c.start()
# Engine starting
# Car is ready to drive
```
`Car` doesn't inherit `Engine`'s methods automatically — it explicitly *delegates* to its own `engine` object. This is more flexible: you could swap in an `ElectricEngine` object without touching `Car`'s class hierarchy at all.

## 8.3 Association, Aggregation, Composition — the relationship spectrum

| Relationship | Meaning | Lifetime coupling | Example |
|---|---|---|---|
| **Association** | Objects know about / use each other, loosely | Independent — no ownership | A `Teacher` and a `Student` (they interact but exist independently) |
| **Aggregation** | "Has-a," but the contained object can exist independently and be shared | Weak — child can outlive parent | A `Team` has `Player` objects, but a `Player` can exist/move to another team |
| **Composition** | "Has-a," and the contained object's lifetime is tied to the container | Strong — child dies with parent | A `House` has `Room` objects — a `Room` doesn't meaningfully exist without its `House` |

```python
# Aggregation example -- Player can exist independent of Team
class Player:
    def __init__(self, name):
        self.name = name

class Team:
    def __init__(self, name, players):
        self.name = name
        self.players = players    # players passed in from outside, can be reused elsewhere

# Composition example -- Room is created and owned entirely by House
class Room:
    def __init__(self, name):
        self.name = name

class House:
    def __init__(self):
        self.rooms = [Room("Living Room"), Room("Bedroom")]   # House creates & owns its Rooms
```

**Memory rule:** *Aggregation = "borrowed" — the contained object can live on its own. Composition = "owned" — the contained object is created and destroyed with its parent.*

## 8.4 Composition vs Inheritance — when to choose which

| | Inheritance | Composition |
|---|---|---|
| Relationship | "is-a" | "has-a" |
| Coupling | Tight — child depends on parent's internal implementation | Loose — container just calls the contained object's public interface |
| Flexibility | Fixed at class-definition time | Can swap components at runtime |
| Risk | Deep hierarchies get fragile ("fragile base class problem") | Slightly more boilerplate (explicit delegation) |

**Guideline (also Module 12's principle): "favor composition over inheritance."** Reach for inheritance only when the "is-a" relationship is genuinely true and stable; otherwise, compose.

### Wrong vs Right
```python
# WRONG -- inheriting just to reuse a method, no real "is-a" relationship
class Logger:
    def log(self, msg): print(f"LOG: {msg}")

class OrderProcessor(Logger):     # OrderProcessor "is-a" Logger?? No.
    def process(self):
        self.log("Processing order")
```
```python
# RIGHT -- compose instead
class OrderProcessor:
    def __init__(self):
        self.logger = Logger()     # OrderProcessor HAS-A Logger
    def process(self):
        self.logger.log("Processing order")
```

---

## Module 8 — Practice

**Recall:** What's the practical difference between aggregation and composition, in terms of object lifetime?

**Design:** Model a `Library` that contains `Book` objects. Is this aggregation or composition? Justify your answer — could a `Book` exist and be checked out to a `Person` independent of any one `Library`?

**Coding challenge:** Refactor a `Car IS-A Engine` design (bad) into `Car HAS-A Engine` (good), and add a `refuel()` delegate method.

---

# MODULE 9 — ADVANCED CLASS CONCEPTS

## 9.1 Class vs Instance Namespace, Attribute Lookup

Every class and every object has its own namespace (a dict-like storage). When you write `obj.attr`, Python follows a specific search order:

1. Look in `obj.__dict__` (the instance's own storage).
2. If not found, look in `type(obj).__dict__` (the class's storage).
3. If not found, continue up the MRO chain (parent classes).
4. If still not found, raise `AttributeError`.

```python
class Animal:
    kingdom = "Animalia"      # class attribute

class Dog(Animal):
    def __init__(self, name):
        self.name = name        # instance attribute

d = Dog("Rex")
print(d.__dict__)          # {'name': 'Rex'}          -- only instance data
print(Dog.__dict__.keys())  # class-level stuff (methods, etc, NOT 'kingdom')
print(d.kingdom)            # "Animalia" -- found by walking up to Animal's __dict__
```
**Method lookup** follows the exact same chain — this is *why* inheritance and MRO (Module 4) work at all: calling `d.some_method()` searches instance → class → parent classes, in MRO order, until it finds `some_method`.

## 9.2 `type()`, `isinstance()`, `issubclass()`

```python
print(type(d))                    # <class '__main__.Dog'>
print(isinstance(d, Dog))          # True
print(isinstance(d, Animal))       # True -- Dog IS-A Animal (inheritance!)
print(isinstance(d, str))          # False
print(issubclass(Dog, Animal))     # True -- checks the CLASS relationship, no object needed
print(issubclass(Dog, Dog))        # True -- a class is considered a subclass of itself
```
- `type(obj)` — gives the exact class of an object.
- `isinstance(obj, Cls)` — checks if an object is `Cls` OR any subclass of `Cls` (respects inheritance).
- `issubclass(ClsA, ClsB)` — checks the class relationship directly, no object needed.

**Common mistake:** using `type(obj) == Cls` instead of `isinstance(obj, Cls)` — this breaks polymorphism, because it fails for legitimate subclasses.
```python
type(d) == Animal      # False -- WRONG check, breaks for subclasses
isinstance(d, Animal)   # True  -- RIGHT check, respects inheritance
```

## 9.3 Dynamic attributes — `getattr`, `setattr`, `hasattr`, `delattr`

```python
class Config:
    pass

c = Config()
setattr(c, "debug", True)          # same as c.debug = True
print(getattr(c, "debug"))          # True -- same as c.debug
print(getattr(c, "missing", "N/A"))  # "N/A" -- default if attribute doesn't exist
print(hasattr(c, "debug"))          # True
delattr(c, "debug")                  # same as del c.debug
```
This is useful when attribute names are only known at runtime (e.g., loading settings dynamically from a file), rather than hardcoded in your source.

```python
# Real-world example: loading config from a dict dynamically
settings = {"debug": True, "verbose": False}
c = Config()
for key, value in settings.items():
    setattr(c, key, value)
```

---

## Module 9 — Practice

**Recall:** Explain the attribute lookup order Python follows when you write `obj.attr`.

**Debugging:**
```python
class Dog(Animal):
    pass

d = Dog()
if type(d) == Animal:
    print("It's an animal")
else:
    print("Not recognized")
```
This prints "Not recognized" even though `Dog` clearly is an `Animal`. Why, and what's the one-word fix?

**Coding challenge:** Write a function `describe(obj)` that uses `hasattr` to check if `obj` has a `make_sound` method, and calls it if so, otherwise prints "Silent object."

---

# MODULE 10 — ADVANCED PYTHON OOP

## 10.1 `__slots__`

### The problem
By default, every object carries a `__dict__` to allow arbitrary dynamic attributes — flexible, but memory-heavy if you create millions of small objects.

```python
class Point:
    __slots__ = ('x', 'y')     # explicitly declares the ONLY allowed attributes
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
p.z = 5     # AttributeError: 'Point' object has no attribute 'z'
```
`__slots__` tells Python to skip creating a `__dict__` for instances, storing `x` and `y` in a fixed, more memory-efficient structure instead. Trade-off: you lose the ability to dynamically add new attributes, and multiple inheritance with slots gets tricky. Use it when creating huge numbers of simple objects (performance-sensitive code).

## 10.2 Dataclasses

### The problem
```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __repr__(self):
        return f"Point(x={self.x}, y={self.y})"
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y
```
For simple "data holder" classes, writing `__init__`, `__repr__`, `__eq__` by hand every time is repetitive boilerplate.

### The concept
```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p1 = Point(1, 2)
p2 = Point(1, 2)
print(p1)          # Point(x=1, y=2)   -- __repr__ auto-generated
print(p1 == p2)      # True             -- __eq__ auto-generated
```
`@dataclass` auto-generates `__init__`, `__repr__`, and `__eq__` based on the declared fields — perfect for classes that are mostly about holding structured data.

### Frozen dataclasses (immutability)
```python
@dataclass(frozen=True)
class Point:
    x: int
    y: int

p = Point(1, 2)
p.x = 5     # dataclasses.FrozenInstanceError -- can't modify after creation
```
Use `frozen=True` when the object should represent an immutable value (like a coordinate, a config snapshot) — this prevents whole categories of bugs from accidental mutation.

## 10.3 Mixins

**What is it?** A small class designed *not* to be used standalone, but to be combined with other classes via multiple inheritance to "mix in" one specific piece of reusable behavior.

```python
class JSONSerializableMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class LoggerMixin:
    def log(self, msg):
        print(f"[LOG] {msg}")

class User(JSONSerializableMixin, LoggerMixin):
    def __init__(self, name):
        self.name = name

u = User("Sam")
print(u.to_json())    # {"name": "Sam"}
u.log("User created")   # [LOG] User created
```
`User` gains both behaviors "for free" through multiple inheritance, without either mixin needing to know anything about `User` specifically. Mixins should generally have no `__init__` of their own and shouldn't be instantiated alone.

## 10.4 Descriptors — `__get__`, `__set__`, `__delete__`

### The problem
`@property` is great for one attribute on one class, but if you need the *same* validation logic (e.g., "must be a positive number") reused across many different attributes or classes, copy-pasting `@property` blocks gets repetitive.

### The concept
A **descriptor** is an object that defines `__get__`/`__set__`/`__delete__`, and when placed as a class attribute, it intercepts attribute access for you — like a reusable, class-level version of `@property`.

```python
class PositiveNumber:                          # the descriptor
    def __set_name__(self, owner, name):
        self.name = "_" + name
    def __get__(self, obj, objtype=None):
        return getattr(obj, self.name)
    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name} must be positive")
        setattr(obj, self.name, value)

class Product:
    price = PositiveNumber()       # reused descriptor
    quantity = PositiveNumber()     # reused again, same validation logic

p = Product()
p.price = 100         # works
p.quantity = -5        # ValueError: _quantity must be positive
```
This is exactly how `@property` itself is implemented internally — descriptors are the underlying mechanism that powers properties, methods, and `@staticmethod`/`@classmethod`.

## 10.5 Metaclasses

### The problem
Sometimes you need to control **how classes themselves are created** — e.g., automatically registering every subclass, enforcing naming conventions on every class in a system, or auto-adding logging to every method.

### The concept
Just as a class is a blueprint for objects, a **metaclass** is a blueprint for classes. `type` is the default metaclass in Python — every class you write is actually an *instance* of `type`.

```python
print(type(Product))    # <class 'type'> -- Product is itself an "object" created by type!
```

```python
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"Creating class: {name}")
        return super().__new__(mcs, name, bases, namespace)

class MyClass(metaclass=Meta):
    pass
# prints: "Creating class: MyClass"   -- runs when the CLASS itself is defined, not when instantiated
```
Metaclasses are advanced and rarely needed in everyday code — frameworks like Django (for its ORM models) use them internally. **Rule of thumb:** if you're not sure you need a metaclass, you don't.

**Memory rule:** *Class creates objects. Metaclass creates classes. `type` is the metaclass behind every ordinary class.*

## 10.6 Protocols & Structural Typing (recap link to Module 6)
Covered in 6.3 — Protocols let you define "shape-based" interfaces without requiring inheritance, useful with static type checkers.

---

## Module 10 — Practice

**Recall:** What's the actual benefit of `__slots__`, and what do you give up to get it?

**Comparison:** In your own words, how is a descriptor different from a plain `@property`?

**Coding challenge:** Write a `@dataclass(frozen=True)` representing an immutable `Coordinates(lat, lon)`, and demonstrate that attempting to mutate it raises an error.

---

# MODULE 11 — OBJECT LIFECYCLE AND MEMORY (OOP-relevant parts)

## 11.1 `__new__` vs `__init__`

**The real object creation order:**
```python
class Demo:
    def __new__(cls, *args, **kwargs):
        print("1. __new__ creates the object")
        instance = super().__new__(cls)
        return instance
    def __init__(self, value):
        print("2. __init__ configures the object")
        self.value = value

d = Demo(10)
# 1. __new__ creates the object
# 2. __init__ configures the object
```
- `__new__` is a `@staticmethod` (implicitly) responsible for **creating** and returning a new object.
- `__init__` receives that already-created object as `self` and **configures** it.

**When you'd override `__new__`:** controlling whether an object gets created at all — e.g., the Singleton pattern (Module 13) uses `__new__` to return an *existing* instance instead of creating a new one.

| | `__new__` | `__init__` |
|---|---|---|
| Job | Creates the object | Initializes the already-created object |
| Returns | The new object (must return something) | Nothing (implicitly `None`) |
| When to override | Controlling creation itself (singletons, immutables) | Normal attribute setup (99% of the time) |

## 11.2 `__del__` — object destruction

```python
class Resource:
    def __del__(self):
        print("Resource cleaned up")

r = Resource()
del r     # "Resource cleaned up" (usually -- timing isn't 100% guaranteed)
```
`__del__` runs when an object is garbage-collected. **Common mistake:** relying on `__del__` for critical cleanup (like closing files) — its exact timing isn't guaranteed. Prefer `__enter__`/`__exit__` (Module 7) with `with` statements for deterministic cleanup.

## 11.3 References, Mutable vs Immutable

```python
a = [1, 2, 3]
b = a            # b is a REFERENCE to the same list, not a copy
b.append(4)
print(a)          # [1, 2, 3, 4] -- a changed too! same object.
```
This matters constantly in OOP: passing an object into a method doesn't copy it — the method receives a reference to the *same* object, so mutations inside the method are visible outside it too.

```python
class Cart:
    def add_discount(self, items):
        items.append("discount applied")   # mutates the caller's list directly!

my_items = ["apple", "banana"]
Cart().add_discount(my_items)
print(my_items)   # ['apple', 'banana', 'discount applied'] -- surprised? this is why it matters
```
- **Mutable** objects (lists, dicts, most custom objects) can be changed in place.
- **Immutable** objects (strings, tuples, frozen dataclasses, ints) cannot — any "change" actually creates a new object.

## 11.4 Shallow vs Deep Copy

```python
import copy

class Order:
    def __init__(self, items):
        self.items = items

o1 = Order(["apple", "banana"])
o2 = copy.copy(o1)              # SHALLOW copy -- new Order object, but SAME items list inside
o2.items.append("cherry")
print(o1.items)                  # ['apple', 'banana', 'cherry'] -- o1 affected too!

o3 = copy.deepcopy(o1)          # DEEP copy -- recursively copies nested objects too
o3.items.append("mango")
print(o1.items)                  # unaffected -- fully independent copy
```
**Memory rule:** *Shallow copy = new box, same contents inside. Deep copy = new box AND new copies of everything inside it.*

## 11.5 Garbage Collection (basics)
Python automatically frees objects once nothing references them anymore (reference counting, plus a cycle detector for circular references). As an OOP developer, the main practical takeaway: you rarely need to manually manage memory, but understanding references (11.3) prevents subtle aliasing bugs, which are far more common than actual memory leaks.

---

## Module 11 — Practice

**Recall:** Why is relying on `__del__` for critical cleanup considered risky?

**Output prediction:**
```python
import copy
class Box:
    def __init__(self, items):
        self.items = items

b1 = Box([1,2,3])
b2 = copy.copy(b1)
b2.items.append(4)
print(b1.items)
```

**Coding challenge:** Write a method that safely returns a copy of an internal list attribute (not the original reference), so external code can't accidentally mutate your object's internal state.

---

# MODULE 12 — OOP DESIGN PRINCIPLES

## 12.1 DRY — Don't Repeat Yourself
Every piece of knowledge/logic should exist in exactly one place. If the same validation logic appears in three classes, a bug fix requires three edits, and they'll eventually drift out of sync. This is the founding motivation behind inheritance, composition, and mixins — all are DRY tools.

## 12.2 SOLID Principles

### S — Single Responsibility Principle (SRP)
**A class should have one, and only one, reason to change.**
```python
# WRONG -- one class doing THREE unrelated jobs
class Report:
    def generate(self): ...
    def save_to_file(self, path): ...
    def send_email(self, address): ...
```
If the email logic changes, or the file format changes, or the report content changes, this *one* class is touched for all three unrelated reasons — high risk of breaking unrelated functionality.
```python
# RIGHT -- split by responsibility
class Report:
    def generate(self): ...

class ReportSaver:
    def save_to_file(self, report, path): ...

class ReportEmailer:
    def send_email(self, report, address): ...
```

### O — Open/Closed Principle (OCP)
**Classes should be open for extension, but closed for modification.** You should be able to add new behavior without editing existing, tested code.
```python
# WRONG -- adding a new discount type means editing this function again and again
def calculate_discount(customer_type, amount):
    if customer_type == "regular":
        return amount * 0.95
    elif customer_type == "vip":
        return amount * 0.80
    # every new type = another elif, and risk of breaking existing ones
```
```python
# RIGHT -- polymorphism lets you EXTEND via new classes, no existing code touched
class DiscountStrategy(ABC):
    @abstractmethod
    def apply(self, amount): pass

class RegularDiscount(DiscountStrategy):
    def apply(self, amount): return amount * 0.95

class VIPDiscount(DiscountStrategy):
    def apply(self, amount): return amount * 0.80

# adding "StudentDiscount" later = a new class, zero changes to existing ones
```

### L — Liskov Substitution Principle (LSP)
**A subclass should be substitutable for its parent class without breaking the program.** The classic violation:
```python
class Bird:
    def fly(self):
        print("Flying")

class Penguin(Bird):          # Penguin IS-A Bird, but...
    def fly(self):
        raise Exception("Penguins can't fly!")   # breaks the contract Bird promised

def make_it_fly(bird: Bird):
    bird.fly()      # this function trusted Bird's contract — Penguin breaks it silently
```
Anywhere `Bird` is expected, substituting `Penguin` should work seamlessly — here it crashes instead. The fix is usually to rethink the hierarchy (e.g., separate `FlyingBird` from `Bird`), not to force the inheritance.

### I — Interface Segregation Principle (ISP)
**Don't force a class to implement methods it doesn't need.** Prefer several small, focused interfaces over one large, "fat" one.
```python
# WRONG -- one bloated interface
class Worker(ABC):
    @abstractmethod
    def work(self): pass
    @abstractmethod
    def eat(self): pass       # a Robot worker doesn't eat!

class Robot(Worker):
    def work(self): print("Working")
    def eat(self): pass       # forced to implement something meaningless
```
```python
# RIGHT -- split into focused interfaces
class Workable(ABC):
    @abstractmethod
    def work(self): pass

class Eatable(ABC):
    @abstractmethod
    def eat(self): pass

class Robot(Workable):          # only implements what actually applies
    def work(self): print("Working")

class Human(Workable, Eatable):
    def work(self): print("Working")
    def eat(self): print("Eating")
```

### D — Dependency Inversion Principle (DIP)
**High-level modules shouldn't depend on low-level details — both should depend on abstractions.**
```python
# WRONG -- OrderService is tightly coupled to a specific email implementation
class EmailSender:
    def send(self, message): print(f"Email: {message}")

class OrderService:
    def __init__(self):
        self.sender = EmailSender()      # hardcoded dependency
    def notify(self, message):
        self.sender.send(message)
```
```python
# RIGHT -- depend on an abstraction, inject the concrete implementation
class NotificationSender(ABC):
    @abstractmethod
    def send(self, message): pass

class EmailSender(NotificationSender):
    def send(self, message): print(f"Email: {message}")

class SMSSender(NotificationSender):
    def send(self, message): print(f"SMS: {message}")

class OrderService:
    def __init__(self, sender: NotificationSender):    # DEPENDENCY INJECTION
        self.sender = sender
    def notify(self, message):
        self.sender.send(message)

OrderService(EmailSender()).notify("Order shipped")
OrderService(SMSSender()).notify("Order shipped")     # swap implementations freely
```

## 12.3 Coupling and Cohesion

- **Coupling** — how much one class depends on the internal details of another. **Low coupling is good** — classes should interact through clean interfaces, not by reaching into each other's internals.
- **Cohesion** — how focused a single class's responsibilities are. **High cohesion is good** — everything in a class should relate to one clear purpose (ties directly to SRP).

**Memory rule:** *Aim for classes that are tightly focused inside (high cohesion) and loosely connected outside (low coupling).*

## 12.4 Favor Composition Over Inheritance
Already covered deeply in Module 8 — restated here as a formal principle because it's one of the most important high-level design heuristics in professional OOP. Default to composition; reach for inheritance only when "is-a" is unambiguous and stable over time.

---

## Module 12 — Practice

**Recall:** Name all five SOLID principles and give a one-sentence definition for each, from memory.

**Recognition:** Which SOLID principle is being violated?
```python
class ReportGenerator:
    def generate_pdf(self): ...
    def generate_excel(self): ...
    def upload_to_s3(self): ...
    def send_slack_notification(self): ...
```

**Design:** Refactor the class above to comply with SRP, splitting responsibilities into focused classes.

---

# MODULE 13 — DESIGN PATTERNS

Each pattern follows: **Problem → Bad approach → Better approach (the pattern) → Pros/Cons → When NOT to use.**

## 13.1 Singleton
**Problem:** You need exactly one instance of a class across the whole program (e.g., one shared configuration object, one database connection pool).
```python
# Bad -- nothing stops multiple instances, each with possibly different state
config1 = Config()
config2 = Config()     # oops, now two different "single sources of truth"
```
**Pattern:**
```python
class Singleton:
    _instance = None
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

class Config(Singleton):
    def __init__(self):
        self.settings = {}

c1 = Config()
c2 = Config()
print(c1 is c2)   # True -- same object every time
```
**Pros:** guarantees one shared instance, global controlled access. **Cons:** acts like hidden global state, makes testing harder (state persists across tests), can hide dependencies. **When NOT to use:** when you just need "shared configuration" — often dependency injection is cleaner and more testable.

## 13.2 Factory & Factory Method
**Problem:** Object creation logic is complex or depends on conditions, and scattering `if/elif` creation logic across your codebase duplicates that decision everywhere.
```python
# Bad -- creation logic scattered and duplicated across the codebase
if payment_type == "credit_card":
    payment = CreditCardPayment()
elif payment_type == "paypal":
    payment = PaypalPayment()
```
**Pattern (Factory Method — centralizes the decision):**
```python
class PaymentFactory:
    @staticmethod
    def create(payment_type):
        if payment_type == "credit_card":
            return CreditCardPayment()
        elif payment_type == "paypal":
            return PaypalPayment()
        raise ValueError("Unknown payment type")

payment = PaymentFactory.create("paypal")
```
**Pros:** creation logic lives in one place; callers don't need to know concrete classes. **Cons:** adds an extra layer/class for what might be simple construction. **When NOT to use:** when object creation is trivial (`ClassName()` with no branching) — a factory adds needless indirection.

## 13.3 Abstract Factory
**Problem:** You need to create *families* of related objects that must work together (e.g., a UI toolkit needing matching `Button` + `Checkbox` for either "Dark" or "Light" theme).
```python
class DarkButton: ...
class DarkCheckbox: ...
class LightButton: ...
class LightCheckbox: ...

class UIFactory(ABC):
    @abstractmethod
    def create_button(self): pass
    @abstractmethod
    def create_checkbox(self): pass

class DarkThemeFactory(UIFactory):
    def create_button(self): return DarkButton()
    def create_checkbox(self): return DarkCheckbox()

class LightThemeFactory(UIFactory):
    def create_button(self): return LightButton()
    def create_checkbox(self): return LightCheckbox()

def build_ui(factory: UIFactory):
    return factory.create_button(), factory.create_checkbox()
```
**Pros:** guarantees consistent, matching families of objects. **Cons:** more classes and abstraction layers than a simple factory. **When NOT to use:** when you only ever create one kind of object, not families of related ones.

## 13.4 Builder
**Problem:** An object has many optional parameters, and a giant constructor becomes unreadable.
```python
# Bad -- unreadable, error-prone constructor call
burger = Burger(True, False, True, False, True, None, "large")
```
**Pattern:**
```python
class BurgerBuilder:
    def __init__(self):
        self.cheese = False
        self.lettuce = False
        self.size = "medium"
    def add_cheese(self):
        self.cheese = True
        return self
    def add_lettuce(self):
        self.lettuce = True
        return self
    def set_size(self, size):
        self.size = size
        return self
    def build(self):
        return Burger(self.cheese, self.lettuce, self.size)

burger = BurgerBuilder().add_cheese().add_lettuce().set_size("large").build()
```
**Pros:** readable, self-documenting construction; handles optional parameters cleanly. **Cons:** more code than a plain constructor for simple objects. **When NOT to use:** objects with 2-3 simple, always-required parameters — a builder there is overkill.

## 13.5 Adapter
**Problem:** You have an existing class with an incompatible interface, and you can't (or shouldn't) modify it, but your code expects a different interface.
```python
class OldPrinter:
    def print_old(self, text):
        print(f"[OLD]: {text}")

class NewPrinterInterface(ABC):
    @abstractmethod
    def print(self, text): pass

class PrinterAdapter(NewPrinterInterface):
    def __init__(self, old_printer):
        self.old_printer = old_printer
    def print(self, text):
        self.old_printer.print_old(text)     # translates the call

printer = PrinterAdapter(OldPrinter())
printer.print("Hello")     # [OLD]: Hello -- works through the new interface
```
**Pros:** lets incompatible interfaces work together without modifying either side. **Cons:** adds an indirection layer. **When NOT to use:** if you can simply modify the original class's interface directly, do that instead — no need for an adapter.

## 13.6 Decorator (the pattern, not the `@` syntax)
**Problem:** You want to add behavior to an object dynamically, without modifying its class or creating an explosion of subclasses for every combination of features.
```python
class Coffee:
    def cost(self):
        return 5

class MilkDecorator:
    def __init__(self, coffee):
        self.coffee = coffee
    def cost(self):
        return self.coffee.cost() + 2

class SugarDecorator:
    def __init__(self, coffee):
        self.coffee = coffee
    def cost(self):
        return self.coffee.cost() + 1

order = SugarDecorator(MilkDecorator(Coffee()))
print(order.cost())   # 5 + 2 + 1 = 8
```
**Pros:** flexible, composable add-ons without subclass explosion (`CoffeeWithMilkAndSugar`, `CoffeeWithMilk`, etc.). **Cons:** many small wrapper objects can be harder to trace/debug. **When NOT to use:** when there are only 1-2 fixed variations — plain subclassing is simpler.

## 13.7 Strategy
**Problem:** You have multiple interchangeable algorithms for the same task, and hardcoding the choice via `if/elif` makes the class rigid (recall the OCP violation in 12.2).
```python
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data): pass

class QuickSort(SortStrategy):
    def sort(self, data): return sorted(data)     # simplified

class Sorter:
    def __init__(self, strategy: SortStrategy):
        self.strategy = strategy
    def sort(self, data):
        return self.strategy.sort(data)

sorter = Sorter(QuickSort())
sorter.sort([3,1,2])
```
**Pros:** swap algorithms at runtime, easy to add new strategies (an OCP win). **Cons:** more classes for simple cases. **When NOT to use:** if there's genuinely only one algorithm and no foreseeable need for alternatives.

## 13.8 Observer
**Problem:** Multiple objects need to be notified automatically when another object's state changes, without tight coupling between them.
```python
class Subject:
    def __init__(self):
        self.observers = []
    def subscribe(self, observer):
        self.observers.append(observer)
    def notify(self, event):
        for obs in self.observers:
            obs.update(event)

class EmailObserver:
    def update(self, event):
        print(f"Emailing about: {event}")

class SMSObserver:
    def update(self, event):
        print(f"Texting about: {event}")

subject = Subject()
subject.subscribe(EmailObserver())
subject.subscribe(SMSObserver())
subject.notify("Order shipped")
# Emailing about: Order shipped
# Texting about: Order shipped
```
**Pros:** loose coupling — the subject doesn't know or care what the observers do. **Cons:** can create hidden, hard-to-trace chains of updates if overused. **When NOT to use:** simple one-to-one relationships — a direct method call is clearer.

## 13.9 Command
**Problem:** You want to represent an "action" as an object (so it can be queued, logged, or undone), rather than as an immediate, direct function call.
```python
class Command(ABC):
    @abstractmethod
    def execute(self): pass

class LightOnCommand(Command):
    def __init__(self, light):
        self.light = light
    def execute(self):
        self.light.turn_on()

class RemoteControl:
    def __init__(self):
        self.history = []
    def press(self, command: Command):
        command.execute()
        self.history.append(command)     # can support undo/replay later
```
**Pros:** decouples "what to do" from "when/how it's triggered"; supports undo, queuing, logging. **Cons:** adds a class per action — overkill for simple, immediate operations. **When NOT to use:** when you just need a direct function call with no need for queuing, undo, or deferred execution.

## 13.10 Repository
**Problem:** Business logic gets tangled with data-access code (SQL queries, file reads) scattered everywhere, making it hard to test or swap storage.
```python
class UserRepository:
    def __init__(self, db_connection):
        self.db = db_connection
    def get_by_id(self, user_id):
        return self.db.query(f"SELECT * FROM users WHERE id={user_id}")
    def save(self, user):
        self.db.execute("INSERT INTO users ...")

class UserService:
    def __init__(self, repository: UserRepository):
        self.repository = repository        # business logic doesn't know about SQL at all
    def get_user_profile(self, user_id):
        return self.repository.get_by_id(user_id)
```
**Pros:** isolates persistence details, makes testing easy (swap in a fake repository), single place to change storage technology. **Cons:** extra abstraction layer for very small apps. **When NOT to use:** tiny scripts with no real persistence complexity.

## 13.11 Dependency Injection
**Problem:** Already shown in DIP (12.2) — classes that create their own dependencies internally are hard to test and inflexible.
**Pattern:** Pass dependencies *in* from outside (via constructor, as in `OrderService(sender)` from 12.2) rather than creating them inside the class. **Pros:** testable (inject fakes/mocks), flexible (swap implementations). **Cons:** requires the caller to assemble dependencies, which can get complex in large systems (often solved with a "DI container" in bigger frameworks). **When NOT to use:** trivial scripts with no need for swappable behavior or testing isolation.

---

## Module 13 — Practice

**Recall:** Pick any 3 patterns above and explain, from memory, the specific problem each one solves.

**Recognition:** Which pattern is this?
```python
class Shape(ABC):
    @abstractmethod
    def draw(self): pass

class ShapeFactory:
    @staticmethod
    def create(shape_type):
        if shape_type == "circle": return Circle()
        if shape_type == "square": return Square()
```

**Design:** You're building a notification system that must support Email, SMS, and Push notifications, chosen at runtime, with the ability to add new notification types later without touching existing code. Which pattern(s) fit, and why?

---

# MODULE 14 — PROFESSIONAL OOP

## 14.1 Responsibility Assignment
Before writing any class, ask: *"What is this class's ONE job?"* If you can't state it in a single, non-"and"-containing sentence, it likely violates SRP (12.2). Professional design starts from responsibilities, not from data structures — identify *who does what* before deciding *what attributes exist where*.

## 14.2 Dependency Management
Depend on abstractions (interfaces/ABCs/Protocols), not concrete classes, wherever a component might need to change or be swapped (recall DIP, 12.2). Keep dependencies flowing in one clear direction — avoid two classes depending directly on each other's concrete internals (circular coupling).

## 14.3 Extensibility, Maintainability, Testability

- **Extensibility** — can you add new behavior without editing existing, tested code? (OCP, Strategy/Factory patterns)
- **Maintainability** — can another developer (or you, in six months) understand and safely change this code? Favored by SRP, high cohesion, clear naming.
- **Testability** — can you test a class in isolation, without spinning up a database or network call? Favored by dependency injection — inject a fake/mock dependency in tests.

```python
# Testable design -- can inject a FakeSender for testing, no real email sent
class OrderService:
    def __init__(self, sender: NotificationSender):
        self.sender = sender

class FakeSender(NotificationSender):
    def __init__(self):
        self.sent_messages = []
    def send(self, message):
        self.sent_messages.append(message)     # just records, doesn't actually send

# In a test:
fake = FakeSender()
service = OrderService(fake)
service.notify("test")
assert "test" in fake.sent_messages
```

## 14.4 Loose Coupling, High Cohesion — recap and reinforcement
Restating Module 12.3 as the professional mantra you should run through your head while designing every class: *does this class do one focused thing (cohesion), and does it talk to others only through clean, minimal interfaces (coupling)?*

## 14.5 Refactoring Poor OOP — Code Smells and Anti-Patterns

| Code Smell | What it looks like | Fix |
|---|---|---|
| **God Object** | One class does everything (data + business logic + I/O + validation) | Split by responsibility (SRP) |
| **Feature Envy** | A method uses another object's data more than its own | Move the method to the class whose data it actually uses |
| **Shotgun Surgery** | One conceptual change requires editing many unrelated classes | Consolidate the duplicated logic into one place (DRY) |
| **Deep Inheritance Chains** | 5+ levels of inheritance, hard to trace behavior | Favor composition; flatten the hierarchy |
| **Primitive Obsession** | Using raw strings/dicts everywhere instead of proper objects (e.g., passing `"pending"` strings instead of an `OrderStatus` type) | Introduce small, focused value objects/enums |
| **Anemic Domain Model** | Classes are just data bags; all logic lives in separate "service" functions | Move behavior that belongs to the data back into the class itself |

## 14.6 Over-Engineering
Applying every pattern and principle to a 20-line script is itself a mistake. **Professional judgment = knowing when simplicity beats "correct-looking" architecture.** A single function is sometimes genuinely better than a `Factory` + `Strategy` + `Repository` for something trivial.

## 14.7 When NOT to Use OOP
- Simple, one-shot scripts (data transformation, a quick calculation) — plain functions are clearer.
- Purely functional transformations with no meaningful internal state (map/filter/reduce-style pipelines).
- When the "objects" you'd create have no real behavior — if a class would just be a bag of data with getters, consider a `dataclass`, `namedtuple`, or plain dict instead.

**Memory rule:** *OOP is a tool for managing complexity through structure — don't manufacture complexity just to have somewhere to apply the tool.*

---

## Module 14 — Practice

**Recall:** Name three code smells and, for each, the design principle that fixes it.

**Design:** You inherit a `God Object` called `OrderManager` that validates orders, calculates prices, sends emails, and saves to a database, all in one class. Sketch how you'd split it (which classes, what each owns).

---

# MODULE 15 — REAL-WORLD PROJECTS (Worked Example: Project 1)

The full progression (Bank Account → Library → Employee Management → Shopping Cart → E-commerce Orders → Parking Lot → Food Delivery → Ride Booking → Payment System → Inventory Management) is meant to be done **interactively** — requirements first, then you design, then review together — because that back-and-forth is what actually builds the "see a problem, think in objects" instinct this whole course is for. A fully pre-written solution below would skip that step, so here's how Project 1 works as a template for the rest:

## Project 1 — Bank Account System

### Requirements
- Support creating an account with an owner name and starting balance.
- Support deposit and withdrawal, with validation (no negative deposits, no overdraft).
- Keep a transaction history.
- Support multiple account types (Savings earns interest; Checking has no interest but allows overdraft up to a limit).

### Your task before reading further
Think through and write down:
1. What classes do you need?
2. What attributes does each class own?
3. What methods does each class need?
4. Where does "Savings vs Checking" difference fit — inheritance? A flag? Composition?

### A reference design (compare against your own — don't just copy it)
```python
from abc import ABC, abstractmethod
from datetime import datetime

class Account(ABC):
    def __init__(self, owner, balance=0):
        self.owner = owner
        self._balance = balance
        self.transactions = []

    @property
    def balance(self):
        return self._balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self._balance += amount
        self.transactions.append(("deposit", amount, datetime.now()))

    @abstractmethod
    def withdraw(self, amount):
        pass          # each account type has different withdrawal rules

class SavingsAccount(Account):
    def __init__(self, owner, balance=0, interest_rate=0.02):
        super().__init__(owner, balance)
        self.interest_rate = interest_rate

    def withdraw(self, amount):
        if amount > self._balance:
            raise ValueError("Insufficient funds")
        self._balance -= amount
        self.transactions.append(("withdraw", amount, datetime.now()))

    def apply_interest(self):
        self._balance += self._balance * self.interest_rate

class CheckingAccount(Account):
    def __init__(self, owner, balance=0, overdraft_limit=500):
        super().__init__(owner, balance)
        self.overdraft_limit = overdraft_limit

    def withdraw(self, amount):
        if amount > self._balance + self.overdraft_limit:
            raise ValueError("Exceeds overdraft limit")
        self._balance -= amount
        self.transactions.append(("withdraw", amount, datetime.now()))
```

### Why this design (the reasoning you should be able to reproduce yourself)
- `Account` is abstract (Module 6) because a bare, untyped account isn't a real, usable thing — every real account is specifically a Savings or Checking account.
- `withdraw` is abstract because the *rule* for withdrawal genuinely differs by type (overdraft vs no-overdraft) — this is a legitimate use of inheritance + polymorphism (Modules 4–5), not composition, because "is-a" truly holds: a `SavingsAccount` **is an** `Account`.
- `_balance` is protected (Module 3) with a read-only `balance` property — external code can read it but must go through `deposit`/`withdraw` to change it, preserving the "no impossible states" guarantee from Module 3's opening problem.
- `transactions` uses composition — an `Account` **has a** list of transaction records; a transaction record isn't a variant of `Account`, so inheritance would be wrong here.

### Now apply this template yourself
For each remaining project (Library, Employee Management, Shopping Cart, E-commerce, Parking Lot, Food Delivery, Ride Booking, Payment, Inventory):
1. Write the requirements down in your own words.
2. List candidate classes and, for each, ask: *is-a or has-a relationships to what?*
3. Decide where abstraction is warranted (a real family of interchangeable types) vs where it's overkill (a single, standalone class).
4. Check your design against SOLID (Module 12) before writing code.
5. Identify which design pattern (Module 13), if any, naturally fits — don't force one in.

---

# COURSE COMPLETE — FINAL SELF-CHECK

You've genuinely mastered this material only when you can, without looking back at this document:
- Explain every module's core concept in your own words.
- Look at unfamiliar OOP code and name the concepts/patterns in use.
- Write a class family from a plain-English requirement, unaided.
- Justify every design decision ("why this class, why this attribute, why inheritance vs composition here").
- Spot a code smell and name which principle it violates.
- Answer the interview-style questions embedded in each module's practice section from memory.

Revisit the **Practice** sections at the end of each module regularly — spaced repetition is what turns "I read this once" into "I actually remember this."
