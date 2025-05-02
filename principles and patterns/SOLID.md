# SOLID

**SOLID** is an acronym that represents five key principles of object-oriented design aimed at creating code that is **more understandable, flexible, and maintainable**. These principles were popularized by Robert C. Martin (Uncle Bob).

Here’s what SOLID stands for:

1. **S – Single Responsibility Principle (SRP):**
   A class should have only one reason to change, meaning it should only have **one job or responsibility**.

2. **O – Open/Closed Principle (OCP):**
   Software entities (classes, modules, functions) should be **open for extension** but **closed for modification**. You should be able to add new behavior without changing existing code.

3. **L – Liskov Substitution Principle (LSP):**
   Objects of a subclass should be **replaceable** with objects of the superclass without breaking the correctness of the program.

4. **I – Interface Segregation Principle (ISP):**
   Clients should not be forced to depend on interfaces they do not use. It’s better to have **many small, specific interfaces** than a large, general-purpose one.

5. **D – Dependency Inversion Principle (DIP):**
   High-level modules should not depend on low-level modules; **both should depend on abstractions**. Abstractions should not depend on details; **details should depend on abstractions**.

## Liskov Substitution Principle (LSP)
> If a program works with a parent class, it should also work if you replace the parent class with one of its child classes — **without breaking anything**.

### Example:

Imagine this:

```php
<?php

class Bird {
    public function fly() {
        echo "Flying\n";
    }
}

class Sparrow extends Bird {
    // Inherits fly(), which is fine
}

class Penguin extends Bird {
    // Penguins can't fly, but still inherit fly() — this is the problem!
}

function makeItFly(Bird $bird) {
    $bird->fly();  // This will call fly() even for penguins!
}

makeItFly(new Sparrow());  // Outputs: Flying
makeItFly(new Penguin());  // Outputs: Flying ❌ — Doesn't make sense!

```

**What's wrong?**
We expected all birds to fly, but penguins can't. So replacing `Bird` with `Penguin` **breaks the logic** — this **violates LSP**.

### Fix:

We should design classes so that **child classes truly behave like the parent**. Maybe create a better hierarchy like:

```php
<?php

class Bird {
    // Common bird properties, but no fly() here
}

class FlyingBird extends Bird {
    public function fly() {
        echo "Flying\n";
    }
}

class Sparrow extends FlyingBird {
    // Inherits fly(), and that’s valid
}

class Penguin extends Bird {
    // No fly() method — no problem!
}

function makeItFly(FlyingBird $bird) {
    $bird->fly();  // Only works with birds that can fly
}

makeItFly(new Sparrow());  // ✅ Works
// makeItFly(new Penguin()); ❌ Fatal error if uncommented — which is good!
```

Now we **don’t expect all birds to fly**, so no broken logic!

## Dependency Inversion Principle (DIP)
> **Don't depend on concrete things. Depend on abstract things (like interfaces).**

---

### Code example (bad – violates DIP):

```php
<?php

  class LightBulb {
    public function turnOn() {
      echo "Light is on\n";
    }
  }

class SwitchDevice {
  private $bulb;

  public function __construct() {
    $this->bulb = new LightBulb(); // Tightly coupled
  }

  public function toggle() {
    $this->bulb->turnOn();
  }
}

$switch = new SwitchDevice();
$switch->toggle();  // Output: Light is on

```

Now we **can't use** another device like a fan with the same switch — it only works with `LightBulb`.

---

### Code example (good – follows DIP):

```php
<?php

interface Switchable {
    public function turnOn();
}

class LightBulb implements Switchable {
    public function turnOn() {
        echo "Light is on\n";
    }
}

class Fan implements Switchable {
    public function turnOn() {
        echo "Fan is on\n";
    }
}

class SwitchDevice {
    private Switchable $device;

    public function __construct(Switchable $device) {
        $this->device = $device;
    }

    public function toggle() {
        $this->device->turnOn();
    }
}

// Now the switch can work with any Switchable device
$switch1 = new SwitchDevice(new LightBulb());
$switch1->toggle();  // Output: Light is on

$switch2 = new SwitchDevice(new Fan());
$switch2->toggle();  // Output: Fan is on
```

Now, `Switch` doesn’t care **what the device is** — as long as it follows the `Switchable` interface. That's **dependency inversion**!

---

### In short:

* Bad: Code depends on specific classes.
* Good: Code depends on **interfaces or abstract types**, making it flexible and testable.

