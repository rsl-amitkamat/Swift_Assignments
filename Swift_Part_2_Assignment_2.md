# Swift Part 2 Assignment

## 1. static vs class

Both keywords make a type method (or type property), which belongs to the type itself and is called on the type, like `Shape.describe()`, and not on an instance.

The difference is about overriding. A `static` method cannot be overridden by a subclass. In a class it is the same as marking it `final class`. A `class` method can be overridden by a subclass. Also, `class` can only be used inside classes, because only classes have subclasses, while `static` works in structs, enums and classes. For properties, `static` can be a stored or computed type property, while `class` can only be a computed one.

```swift
class Shape {
    static func category() -> String { "Shape" }          // cannot be overridden
    class func describe() -> String { "A generic shape" } // can be overridden
}

class Circle: Shape {
    // override static func category() -> String { "Circle" }  // error
    override class func describe() -> String { "A circle" }    // fine
}

print(Shape.category())    // Shape
print(Circle.describe())   // A circle

struct Utils {
    static func greet() -> String { "Hello" }   // class is not allowed in a struct
}
```

## 2. Menu enum

`Size` is its own enum with a string value for each case, so it prints as "Small", "Medium" or "Large". `Menu` uses associated values for the burger (with a default of 1 patty) and for the fries. The print function goes through the ordered items with a switch, and prints "Coke" right after "Meal".

```swift
enum Size: String {
    case small = "Small"
    case medium = "Medium"
    case large = "Large"
}

enum Menu {
    case burger(patties: Int = 1)
    case fries(Size)
    case meal
    case coke
}

func printOrder(_ items: [Menu]) {
    for item in items {
        switch item {
        case .burger(let patties):
            print("Burger with \(patties) patties")
        case .fries(let size):
            print("\(size.rawValue) fries")
        case .meal:
            print("Meal")
            print("Coke")
        case .coke:
            print("Coke")
        }
    }
}

let orderedItems: [Menu] = [.meal, .fries(.large), .burger()]
printOrder(orderedItems)
```

Output:

```
Meal
Coke
Large fries
Burger with 1 patties
```

## 3. Person struct

The initializer takes the middle name as an optional with a default of nil, so a person can be created with or without one. The extension adds `fullName`, which joins the first, middle and last name, and skips the middle name if there isn't one.

```swift
struct Person {
    var firstName: String
    var middleName: String?
    var lastName: String
    var age: Int

    init(firstName: String, middleName: String? = nil, lastName: String, age: Int) {
        self.firstName = firstName
        self.middleName = middleName
        self.lastName = lastName
        self.age = age
    }
}

extension Person {
    var fullName: String {
        [firstName, middleName, lastName]
            .compactMap { $0 }
            .filter { !$0.isEmpty }
            .joined(separator: " ")
    }
}

let p1 = Person(firstName: "Asha", middleName: "Kumar", lastName: "Rao", age: 28)
let p2 = Person(firstName: "Ravi", lastName: "Shah", age: 31)

print(p1.fullName)   // Asha Kumar Rao
print(p2.fullName)   // Ravi Shah
```

## 4. Stack struct

My first version looked like this:

```swift
struct Stack {
    var items: [Int] = []

    func push(_ value: Int) {
        items.insert(value, at: 0)
    }

    func pop() -> Int? {
        if items.isEmpty { return nil }
        return items.removeFirst()
    }
}
```

It doesn't compile. Both functions give the error **"Cannot use mutating member on immutable value: 'self' is immutable"**.

The reason is that a struct is a value type. By default, the methods of a struct are not allowed to change the struct's own properties, because `self` is treated as a constant inside them. Both `push` and `pop` change `items`, so the compiler stops them. (In a class this works without any change, since classes are reference types.)

The fix is to mark both methods `mutating`, which tells Swift they are allowed to change the struct. Because of that, the stack must also be created with `var` and not `let`, otherwise calling them would give the same kind of error.

```swift
struct Stack {
    var items: [Int] = []

    mutating func push(_ value: Int) {
        items.insert(value, at: 0)
    }

    mutating func pop() -> Int? {
        if items.isEmpty { return nil }
        return items.removeFirst()
    }
}

var stack = Stack()
stack.push(1)
stack.push(2)
stack.push(3)
print(stack.items)   // [3, 2, 1]

print(stack.pop() as Any)   // Optional(3)
print(stack.pop() as Any)   // Optional(2)
print(stack.pop() as Any)   // Optional(1)
print(stack.pop() as Any)   // nil, the stack is empty
```