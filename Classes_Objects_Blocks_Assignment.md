# Categories and Blocks Assignment

Answers and code

## 1. Limitations and problems with categories

- **No stored properties.** A category can declare a property, but it cannot add an instance variable to the class, so there is nothing to store the value in, and the compiler won't synthesize one. The property has to be backed by something else, like associated objects (`objc_setAssociatedObject`), which is more work and easy to get wrong.
- **Method name clashes and overriding.** Categories are not subclasses, so they should not be used to override existing methods. If a category defines a method with the same name as one that already exists in the class, or in another category, one of them silently replaces the other, and which one wins is not defined. When a category overrides a method that is declared only in the class being categorised, the original implementation is lost, and the change affects every object of that class.
- **It affects every instance.** A category method becomes part of the class everywhere, including subclasses and any framework code using that class. A careless change can alter behaviour in places the developer never meant to touch.
- **No access to the class's private variables.** A category in a separate file cannot see instance variables declared privately in the original `.m` file.
- **Static library issue.** If a category is in a static library, it may not get loaded unless the `-ObjC` linker flag is used, and the app then crashes with "unrecognized selector".
- **The category file has to be imported.** The methods are only usable in files that import the category's header, so forgetting the import means the compiler can't see them.
- **Overuse makes code hard to follow.** When behaviour is spread across many categories, it is harder to find where a method actually comes from. Putting business logic into categories, when a subclass would be the better fit, is not a good idea either. It is also a good idea to prefix method names on categories of system classes, so they don't clash with methods Apple adds later.

## 2. Category vs inheritance

A category adds methods to an existing class without creating a new class. Every instance of that class, and of its subclasses, including ones that already exist or are created by the system, gets the new methods straight away. At runtime there is no difference between a method added by a category and one the class had originally. The original source is not needed. But a category can't add stored properties, and it can't properly override a method and still call the original.

Inheritance creates a new subclass. The subclass can add its own instance variables and properties, override methods and call `super`, and be specialized for a particular purpose. The catch is that only instances of the subclass get the new behaviour, so the code has to create the subclass. Objects created by the system, such as string literals, will not be of that subclass, and subclassing some classes (like `NSString` or `NSArray`) is awkward.

So a category suits small helper methods on a class you can't or don't want to subclass (for example, a method on `NSString`), while inheritance suits cases where you need extra state or want to change how a method behaves.

## 3. Category vs extension

An extension (also called a class continuation) is written as `@interface ClassName ()` with empty brackets, and it has no name. A category has a name inside the brackets, like `@interface ClassName (Helpers)`.

The main differences:

- An extension can only be written for a class whose source code you own, since it has to be in that class's own `.m` file. A category can be added to any class, including Apple's classes like `NSString`.
- An extension can add instance variables and properties, which are usually private to the class. A category cannot add instance variables.
- The methods declared in an extension must be implemented in the class's main `@implementation` block, and the compiler warns if they are missing. Category methods go in their own `@implementation ClassName (Name)` block.
- An extension is mainly used to keep private properties and methods out of the public header. A category is mainly used to add or organize extra functionality.

```objc
// Extension: in Person.m, private to the class
@interface Person ()
@property (nonatomic, copy) NSString *secretNote;   // allowed
@end

// Category: can be on any class, no new ivars
@interface NSString (SampleCategory)
- (NSString *)reverseString;
@end

@implementation NSString (SampleCategory)
- (NSString *)reverseString {
    NSMutableString *reversedString = [NSMutableString string];
    NSInteger charIndex = [self length];
    while (charIndex > 0) {
        charIndex--;
        NSRange subStrRange = NSMakeRange(charIndex, 1);
        [reversedString appendString:[self substringWithRange:subStrRange]];
    }
    return [reversedString copy];
}
@end
```

## 4. Block that concatenates two strings

```objc
NSString *(^concatBlock)(NSString *, NSString *) = ^NSString *(NSString *first, NSString *second) {
    return [first stringByAppendingString:second];
};

NSLog(@"%@", concatBlock(@"Hello, ", @"World"));   // Hello, World
```

## 5. Addition and subtraction blocks

```objc
NSInteger (^additionBlock)(NSInteger, NSInteger) = ^NSInteger(NSInteger a, NSInteger b) {
    return a + b;
};

NSInteger (^subtractionBlock)(NSInteger, NSInteger) = ^NSInteger(NSInteger a, NSInteger b) {
    return a - b;
};

NSLog(@"%ld", (long)additionBlock(10, 5));       // 15
NSLog(@"%ld", (long)subtractionBlock(10, 5));    // 5
```

## 6. Chaining the blocks with a printing block

For this one, the addition block calls the subtraction block, and the subtraction block calls the printing block. A block captures the variables it uses at the moment it is created, so the blocks have to be declared in reverse order: the printing block first, then subtraction, then addition.

The addition block adds the two numbers and passes the sum, along with the second number, to the subtraction block. The subtraction block subtracts and passes both results to the printing block.

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void (^printingBlock)(NSInteger, NSInteger) = ^(NSInteger sum, NSInteger difference) {
            NSLog(@"Sum: %ld, Difference: %ld", (long)sum, (long)difference);
        };

        void (^subtractionBlock)(NSInteger, NSInteger) = ^(NSInteger a, NSInteger b) {
            NSInteger difference = a - b;
            printingBlock(a, difference);
        };

        void (^additionBlock)(NSInteger, NSInteger) = ^(NSInteger a, NSInteger b) {
            NSInteger sum = a + b;
            subtractionBlock(sum, b);
        };

        additionBlock(10, 5);
    }
    return 0;
}
```

Output:

```
Sum: 15, Difference: 10
```