# ARC Assignment

## 1. Reference counting, ARC and memory management before ARC

Every object in memory keeps a count of how many other things currently own it. When something takes ownership of the object, the count goes up by 1 (a retain), and when it lets go, the count goes down by 1 (a release). As long as the count is above zero the object stays in memory. The moment it reaches zero, the object is removed from memory and its `dealloc` runs. This way memory is freed exactly when nobody needs the object anymore.

Before ARC, this was all done by hand (manual reference counting). The developer had to write `retain`, `release` and `autorelease` calls in the right places. Forgetting a release caused a memory leak, and releasing too many times left a pointer to an object that was already gone, which usually crashed the app. It was a lot of extra work and an easy place to make mistakes.

ARC (Automatic Reference Counting) came in with iOS 5 and Xcode 4.2. The training material says it was introduced from iOS 4 with improvements in iOS 5. Code written for ARC could target iOS 4 as well, but weak references needed iOS 5. With ARC, the compiler adds the retain and release calls for us while compiling, so we don't write them ourselves. It is not a garbage collector. There is no background process running while the app is working, so there is no pause or extra cost at runtime. Memory is still freed as soon as an object's count hits zero.

## 2. Strong and weak

A **strong** reference means "I own this object". Pointing a strong property or variable at an object increases its reference count by 1, and that object will not be freed as long as the strong reference exists. Object properties are strong by default, and the variable version is `__strong`.

A **weak** reference means "I'm only looking at this object, I don't own it". It does not increase the reference count, so it can't keep the object alive. The object stays in memory only as long as something else holds a strong reference to it. When the object is freed, the weak reference is automatically set to nil. The variable version is `__weak`.

```objc
@property (nonatomic, strong) Apartment *apartment;   // owns it
@property (nonatomic, weak) Person *owner;            // does not own it

__strong id a = someObject;
__weak id b = someObject;
```

## 3. What it means to declare a property weak, and the dangers

Declaring a property weak means the object does not take ownership of whatever it points to, so it has no say in when that object is freed. This is mostly used for delegates and for the "back pointer" to a parent, and to break retain cycles.

The dangers come from the fact that the object can disappear at any time:

- The property can suddenly become nil. If the last strong reference goes away, the weak property is set to nil automatically, and code that expects the object to still be there will quietly do nothing, because messages to nil are ignored.
- If an object is only referenced weakly from the start, it is freed immediately. For example, `__weak Person *p = [[Person alloc] init];` gives a nil `p` right away, since nothing owns the new object.
- In multi-step code, the object can become nil between two lines. If it needs to stay around, it is safer to copy it into a local strong variable first.

Weak is the safe option, because the pointer is set to nil. Its unsafe cousins (`assign` on an object property, or `__unsafe_unretained`) do not do that, and they leave a dangling pointer that can crash the app when used.

## 4. Retain cycle

A retain cycle (strong reference cycle) happens when two objects hold strong references to each other, directly or through a longer chain. Each object is keeping the other one's reference count above zero, so neither count can ever drop to zero, and neither object gets freed even after the rest of the app has stopped using them. This is a memory leak, and the memory use keeps growing every time that code runs. The usual fix is to make one side of the relationship weak, so that it does not own the other.

## 5. Retain cycle demo

A `Person` and an `Apartment` hold strong references to each other. Each class logs when it is created and when it is deallocated.

```objc
#import <Foundation/Foundation.h>

@class Apartment;

@interface Person : NSObject
@property (nonatomic, strong) Apartment *apartment;
@end

@implementation Person

- (instancetype)init {
    self = [super init];
    if (self) {
        NSLog(@"Person created");
    }
    return self;
}

- (void)dealloc {
    NSLog(@"Person deallocated");
}

@end

@interface Apartment : NSObject
@property (nonatomic, strong) Person *person;
@end

@implementation Apartment

- (instancetype)init {
    self = [super init];
    if (self) {
        NSLog(@"Apartment created");
    }
    return self;
}

- (void)dealloc {
    NSLog(@"Apartment deallocated");
}

@end

static void createReferenceCycle(void) {
    Person *person = [[Person alloc] init];
    Apartment *apartment = [[Apartment alloc] init];

    person.apartment = apartment;   // person -> apartment (strong)
    apartment.person = person;      // apartment -> person (strong)

    NSLog(@"Leaving createReferenceCycle");
}   // both local variables go out of scope here

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        createReferenceCycle();
        NSLog(@"Back in main, no dealloc messages above means both objects leaked");
    }
    return 0;
}
```

Output:

```
Person created
Apartment created
Leaving createReferenceCycle
Back in main, no dealloc messages above means both objects leaked
```

When the function ends, the two local variables go away, but each object is still owned by the other. Their reference counts never reach zero, so "Person deallocated" and "Apartment deallocated" are never printed. Changing either `apartment` in `Person` or `person` in `Apartment` to `weak` breaks the cycle, and both dealloc messages then appear.

## 6. Other property qualifiers, and what copy does

**retain:** The older, manual reference counting name for `strong`. It takes ownership of the object and increases its count. Under ARC, `strong` is used instead.

**assign:** Just stores the new value with no ownership and no reference counting. It is meant for non-object types like `int`, `BOOL`, `float` and `NSInteger`. On an object it leaves a dangling pointer if the object is freed.

**unsafe_unretained:** Like weak, it does not take ownership, but it is not set to nil when the object is freed. It still points at the old address, and using it can crash. It is only used for the few older classes that don't support weak references.

**copy:** Instead of keeping a reference to the object that was passed in, the setter makes its own copy of it and owns that copy. The property then holds a separate object, so changes made to the original afterwards do not affect it. This matters most for classes that have a mutable version, like `NSString`, `NSArray` and `NSDictionary`. If someone passes in an `NSMutableString` and the property is only `strong`, the property would be pointing at the same mutable string, and any later change to it would silently change the property. It is also used for blocks.

```objc
@property (nonatomic, copy) NSString *name;

NSMutableString *text = [NSMutableString stringWithString:@"Asha"];
person.name = text;
[text appendString:@" Rao"];

NSLog(@"%@", person.name);   // Asha (with strong it would print "Asha Rao")
```