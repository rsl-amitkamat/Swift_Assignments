# Objective-C Training Assignment

Answers and code

For question 1, all the methods are written as instance methods of a small class called Assignment (a subclass of NSObject). I used NSNumber inside arrays and dictionaries since Objective-C collections can only hold objects, and NSInteger for plain integer arguments and return values. A short main.m at the end of that section shows how each one is called.

## 1. Methods

### a. Sum of two integers

```objc
- (NSInteger)sumOfA:(NSInteger)a andB:(NSInteger)b {
    return a + b;
}
```

### b. Sum of the even numbers in an array

```objc
- (NSInteger)sumOfEvenNumbersIn:(NSArray<NSNumber *> *)numbers {
    NSInteger sum = 0;
    for (NSNumber *number in numbers) {
        NSInteger value = [number integerValue];
        if (value % 2 == 0) {
            sum += value;
        }
    }
    return sum;
}
```

### c. Hello / World from 1 to 100

A switch can only match on a single value, so I boil the two divisibility checks down to one small number: 1 means divisible by 3, 2 means divisible by 5, and 3 means both. Numbers that are divisible by neither are just printed as they are.

```objc
- (void)printHelloWorld {
    for (NSInteger i = 1; i <= 100; i++) {
        NSInteger key = (i % 3 == 0 ? 1 : 0) + (i % 5 == 0 ? 2 : 0);

        switch (key) {
            case 1:
                NSLog(@"Hello");
                break;
            case 2:
                NSLog(@"World");
                break;
            case 3:
                NSLog(@"Hello World");
                break;
            default:
                NSLog(@"%ld", (long)i);
                break;
        }
    }
}
```

### d. Merge two arrays alternately

I loop up to the length of the longer array and add an element from each one whenever it still has something at that index. The inputs are only read, never changed.

```objc
- (NSArray *)mergeAlternatingFirst:(NSArray *)first second:(NSArray *)second {
    NSMutableArray *result = [NSMutableArray array];
    NSUInteger maxCount = MAX(first.count, second.count);

    for (NSUInteger i = 0; i < maxCount; i++) {
        if (i < first.count) {
            [result addObject:first[i]];
        }
        if (i < second.count) {
            [result addObject:second[i]];
        }
    }
    return [result copy];
}
```

[10, 50, 70, 42] and [78, 90] gives [10, 78, 50, 90, 70, 42].

### e. Remove duplicates

A set remembers what has already been seen, so each value is added to the result only the first time it shows up. This keeps the original order and leaves the input array alone.

```objc
- (NSArray *)removeDuplicates:(NSArray *)input {
    NSMutableArray *result = [NSMutableArray array];
    NSMutableSet *seen = [NSMutableSet set];

    for (id item in input) {
        if (![seen containsObject:item]) {
            [seen addObject:item];
            [result addObject:item];
        }
    }
    return [result copy];
}
```

### f. Circular shift of an array

Here the array itself has to change, so it takes an NSMutableArray. The last element is pulled off the end and put back at the front, which shifts everything else one place to the right.

```objc
- (void)rotateRight:(NSMutableArray *)characters {
    if (characters.count < 2) {
        return;
    }
    id last = [characters lastObject];
    [characters removeLastObject];
    [characters insertObject:last atIndex:0];
}
```

[a, b, c, d] becomes [d, a, b, c], and calling it again gives [c, d, a, b].

### g. Character count of a string

The counting method returns the dictionary. A second method walks through it and prints the sentence for each character. I sort the keys before printing so the output comes out in the same order every time, because dictionaries do not guarantee any order.

```objc
- (NSDictionary<NSString *, NSNumber *> *)characterCountsInString:(NSString *)string {
    NSMutableDictionary *counts = [NSMutableDictionary dictionary];

    for (NSUInteger i = 0; i < string.length; i++) {
        NSString *ch = [string substringWithRange:NSMakeRange(i, 1)];
        NSInteger current = [counts[ch] integerValue];   // nil -> 0
        counts[ch] = @(current + 1);
    }
    return [counts copy];
}

- (void)printCharacterCounts:(NSDictionary<NSString *, NSNumber *> *)counts {
    NSArray *keys = [counts.allKeys sortedArrayUsingSelector:@selector(compare:)];

    for (NSString *key in keys) {
        NSLog(@"Character %@ appears %@ times in the string", key, counts[key]);
    }
}
```

### h. Merge two dictionaries

I start with a copy of the second dictionary and then add everything from the first on top of it. Since addEntriesFromDictionary: overwrites existing keys, the first dictionary wins whenever a key appears in both.

```objc
- (NSDictionary<NSString *, NSNumber *> *)mergeDictionary:(NSDictionary *)first
                                           withDictionary:(NSDictionary *)second {
    NSMutableDictionary *merged = [second mutableCopy];
    [merged addEntriesFromDictionary:first];
    return [merged copy];
}
```


## 2. NSSet vs NSArray

An NSArray is an ordered list. Items stay in the position you put them, you can reach them by index, and the same value can appear as many times as you like. An NSSet is an unordered collection of unique objects. There is no index, adding something that is already in the set does nothing, and checking whether something is in the set is very fast because it uses hashing. So I would reach for an array when order or duplicates matter, and a set when I only care about "is it in there?" or want unique values.

Demo:

```objc
NSArray *numbers = @[@1, @2, @2, @3, @3, @3];
NSSet *unique = [NSSet setWithArray:numbers];

NSLog(@"Array count: %lu", (unsigned long)numbers.count);   // 6
NSLog(@"Set count: %lu", (unsigned long)unique.count);      // 3
NSLog(@"Contains 2? %@", [unique containsObject:@2] ? @"YES" : @"NO");

NSMutableSet *a = [NSMutableSet setWithObjects:@1, @2, @3, nil];
NSSet *b = [NSSet setWithObjects:@3, @4, nil];

[a addObject:@2];     // already there, so nothing changes
[a unionSet:b];       // a is now {1, 2, 3, 4}
[a intersectSet:b];   // a is now {3, 4}
NSLog(@"%@", a);
```

## 3. == vs isEqual:

With objects, `==` compares the pointers. It only answers "are these two variables pointing to the exact same object in memory?" The `isEqual:` method asks the object itself whether it is equal in value to another one. Each class decides what that means (NSString compares characters, NSNumber compares the number, NSArray compares its contents), and a class that overrides `isEqual:` should override `hash` too, so that equal objects give the same hash.

```objc
NSString *s1 = [NSString stringWithFormat:@"%@", @"hello"];
NSString *s2 = [NSString stringWithFormat:@"%@", @"hello"];

s1 == s2;                // usually NO, two different objects
[s1 isEqual:s2];         // YES, same content
[s1 isEqualToString:s2]; // YES, the faster string-specific version
```

There are still plenty of times when `==` is the right choice for objects. The most common is checking for nil (`if (obj == nil)`), and it is also how you check for the same instance, for example whether a delegate or a sender is a particular object you already hold. It works well for singletons and shared constants such as `[NSNull null]`, and for comparing Class or SEL values. It is also fine for plain C types like int, NSInteger and enums, which are values and not objects. What I would avoid is using `==` to compare strings or numbers by value, because it might happen to work for small values or string literals and then break in other cases.

## 4. Dynamic runtime typing and id

In Objective-C, a lot of decisions that other languages make at compile time are made while the program is running. Calling a method is really sending a message to an object, and the runtime looks at the actual class of that object at that moment to find which method to run. That is dynamic typing: what matters is what the object really is when the code runs, and not what the variable was declared as. It is why the same line of code can behave differently depending on the object it receives, and why you can send a message to an object without the compiler knowing its exact class. If the object does not have that method, the program does not fail to compile, it crashes at runtime with an "unrecognized selector" error. Methods like `respondsToSelector:`, `isKindOfClass:` and `isMemberOfClass:` let you check first.

The keyword `id` stands for "a pointer to any Objective-C object". It is the generic object type, so a variable of type id can hold an NSString, an NSArray or an instance of your own class, and the compiler will let you send it any message it knows about. Note that there is no `*` when you declare it, because id is already a pointer type.

```objc
id thing = @"hello";
NSLog(@"%lu", (unsigned long)[thing length]);   // works, it is an NSString

thing = @[@1, @2, @3];
NSLog(@"%lu", (unsigned long)[thing count]);    // works, now it is an NSArray

if ([thing respondsToSelector:@selector(length)]) {
    // safe to call length
}
```

## 5. NSObject

NSObject is the root class of almost every class in Objective-C, both Apple's and the ones we write. Since everything inherits from it, every object gets a common set of basic abilities: creating and initialising objects (`alloc` and `init`), memory management support, comparing objects (`isEqual:` and `hash`), printing a readable description (`description`), and asking questions about itself at runtime (`class`, `isKindOfClass:`, `respondsToSelector:`). It is basically the foundation that makes a class behave like a proper Objective-C object.