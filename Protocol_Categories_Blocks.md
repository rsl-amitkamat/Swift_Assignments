# Classes, Objects and Properties Assignment

Answers and code

Each class is written as a `.h` and `.m` pair.

## 1 & 2. Classes and initializers

### Person

`Person.h`

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;
@property (nonatomic, assign) NSInteger age;

- (instancetype)initWithFirstName:(NSString *)firstName
                         lastName:(NSString *)lastName
                              age:(NSInteger)age;

- (void)clearLastName;

@end
```

`Person.m`

```objc
#import "Person.h"

@implementation Person

- (instancetype)initWithFirstName:(NSString *)firstName
                         lastName:(NSString *)lastName
                              age:(NSInteger)age {
    self = [super init];
    if (self) {
        _firstName = [firstName copy];
        _lastName = [lastName copy];
        _age = age;
    }
    return self;
}

- (void)clearLastName {
    self.lastName = nil;
    NSLog(@"Last name: %@", self.lastName);                        // (null)
    NSLog(@"Length: %lu", (unsigned long)[self.lastName length]);  // 0
}

@end
```

### Employee

Employee inherits from Person, so it already has the first name, last name and age. Its initializer calls the Person initializer first and then sets the things that belong to Employee. Full name is worked out from the first and last name, so it is read-only and has no stored value of its own. Organization is read-only in the header, and I make it writable only inside the `.m` file.

`Employee.h`

```objc
#import "Person.h"

@class Organization;

@interface Employee : Person

@property (nonatomic, copy, readonly) NSString *fullName;
@property (nonatomic, weak, readonly) Organization *organization;
@property (nonatomic, copy) NSString *employeeID;

- (instancetype)initWithFirstName:(NSString *)firstName
                         lastName:(NSString *)lastName
                              age:(NSInteger)age
                       employeeID:(NSString *)employeeID;

- (void)updateOrganization:(Organization *)organization;

@end
```

`Employee.m`

```objc
#import "Employee.h"
#import "Organization.h"

@interface Employee ()
@property (nonatomic, weak, readwrite) Organization *organization;
@end

@implementation Employee

- (instancetype)initWithFirstName:(NSString *)firstName
                         lastName:(NSString *)lastName
                              age:(NSInteger)age
                       employeeID:(NSString *)employeeID {
    self = [super initWithFirstName:firstName lastName:lastName age:age];
    if (self) {
        _employeeID = [employeeID copy];
    }
    return self;
}

- (NSString *)fullName {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}

- (void)updateOrganization:(Organization *)newOrganization {
    Organization *oldOrganization = self.organization;
    if (oldOrganization == newOrganization) {
        return;
    }
    [oldOrganization removeEmployee:self];
    self.organization = newOrganization;
    [newOrganization addEmployee:self];
}

@end
```

### Organization

`Organization.h`

```objc
#import <Foundation/Foundation.h>

@class Employee;

@interface Organization : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *organizationID;
@property (nonatomic, strong, readonly) NSArray<Employee *> *employees;

- (instancetype)initWithName:(NSString *)name
              organizationID:(NSString *)organizationID;

- (void)addEmployee:(Employee *)employee;
- (void)removeEmployee:(Employee *)employee;

@end
```

`Organization.m`

```objc
#import "Organization.h"
#import "Employee.h"

@implementation Organization {
    NSMutableArray<Employee *> *_employees;
}

- (instancetype)initWithName:(NSString *)name
              organizationID:(NSString *)organizationID {
    self = [super init];
    if (self) {
        _name = [name copy];
        _organizationID = [organizationID copy];
        _employees = [NSMutableArray array];
    }
    return self;
}

- (NSArray<Employee *> *)employees {
    return [_employees copy];
}

- (void)addEmployee:(Employee *)employee {
    if (employee == nil || [_employees containsObject:employee]) {
        return;
    }
    [_employees addObject:employee];
    [employee updateOrganization:self];
}

- (void)removeEmployee:(Employee *)employee {
    [_employees removeObject:employee];
}

@end
```

## 3. Adding organizations and employees

A small manager class keeps the list of organizations. It can add a new organization, and add an employee to an organization that already exists (looked up by its ID).

`OrganizationManager.h`

```objc
#import <Foundation/Foundation.h>
#import "Organization.h"
#import "Employee.h"

@interface OrganizationManager : NSObject

@property (nonatomic, strong, readonly) NSArray<Organization *> *organizations;

- (Organization *)addOrganizationWithName:(NSString *)name
                           organizationID:(NSString *)organizationID;

- (BOOL)addEmployee:(Employee *)employee
toOrganizationWithID:(NSString *)organizationID;

@end
```

`OrganizationManager.m`

```objc
#import "OrganizationManager.h"

@implementation OrganizationManager {
    NSMutableArray<Organization *> *_organizations;
}

- (instancetype)init {
    self = [super init];
    if (self) {
        _organizations = [NSMutableArray array];
    }
    return self;
}

- (NSArray<Organization *> *)organizations {
    return [_organizations copy];
}

- (Organization *)addOrganizationWithName:(NSString *)name
                           organizationID:(NSString *)organizationID {
    Organization *organization = [[Organization alloc] initWithName:name
                                                     organizationID:organizationID];
    [_organizations addObject:organization];
    return organization;
}

- (BOOL)addEmployee:(Employee *)employee
toOrganizationWithID:(NSString *)organizationID {
    for (Organization *organization in _organizations) {
        if ([organization.organizationID isEqualToString:organizationID]) {
            [organization addEmployee:employee];
            return YES;
        }
    }
    return NO;
}

@end
```

Using it:

```objc
OrganizationManager *manager = [[OrganizationManager alloc] init];
[manager addOrganizationWithName:@"Raja Software" organizationID:@"ORG1"];

Employee *emp = [[Employee alloc] initWithFirstName:@"Asha"
                                           lastName:@"Rao"
                                                age:28
                                         employeeID:@"E101"];
[manager addEmployee:emp toOrganizationWithID:@"ORG1"];
```

## 4. Setting a property to nil

In `Person`, the method `clearLastName` sets `lastName` to nil and then tries to use it:

```objc
- (void)clearLastName {
    self.lastName = nil;
    NSLog(@"Last name: %@", self.lastName);                        // (null)
    NSLog(@"Length: %lu", (unsigned long)[self.lastName length]);  // 0
}
```

The value of the property came out as `(null)`. Calling a method on it, like `length`, also does not crash. In Objective-C, sending a message to nil is allowed and simply does nothing, and it returns 0 (or nil / NO) depending on the return type.

## 5. Why ivars inside getters and setters, not the dot operator

The dot operator on `self` is not direct access to the variable. `self.firstName = value` is the same as calling `[self setFirstName:value]`, and `self.firstName` on its own is `[self firstName]`. So if I use the dot operator inside the setter, the setter calls itself, which calls itself again, and so on forever until the app crashes with a stack overflow. The same happens inside a getter that returns `self.firstName`. An ivar (`_firstName`) reads or writes the stored value directly without going through any method, so there is no loop.

```objc
- (void)setFirstName:(NSString *)firstName {
    _firstName = [firstName copy];     // fine, writes the ivar directly
    // self.firstName = firstName;     // calls setFirstName: again, forever
}

- (NSString *)firstName {
    return _firstName;                 // fine
    // return self.firstName;          // calls firstName again, forever
}
```

## 6. Updating an employee's organization

The organization property is declared `readonly` in `Employee.h`, so code outside the class can read it but cannot assign to it. Inside `Employee.m`, a class extension redeclares it as `readwrite`, so only the class itself can change it. The one public way to change it is `updateOrganization:`, which removes the employee from the old organization, sets the new one, and adds the employee to the new organization's list.

```objc
// Employee.h
@property (nonatomic, weak, readonly) Organization *organization;
- (void)updateOrganization:(Organization *)organization;

// Employee.m
@interface Employee ()
@property (nonatomic, weak, readwrite) Organization *organization;
@end

- (void)updateOrganization:(Organization *)newOrganization {
    Organization *oldOrganization = self.organization;
    if (oldOrganization == newOrganization) {
        return;
    }
    [oldOrganization removeEmployee:self];
    self.organization = newOrganization;
    [newOrganization addEmployee:self];
}
```

```objc
[emp updateOrganization:otherOrg];   // allowed
emp.organization = otherOrg;         // compile error, it is readonly
```

## 7. atomic vs nonatomic

These attributes decide how a property's getter and setter behave when several threads use it.

`atomic` is the default. The getter and setter are wrapped in a lock, so a thread always reads a complete value and never one that is half written by another thread. This costs some speed, and it only protects that single read or write. It does not make the whole object thread-safe, for example when you read a value and then write it back as two separate steps.

`nonatomic` skips the lock, so it is faster, but there is no protection if two threads access the property at the same time. Most iOS code uses `nonatomic` for properties, since the UI and most objects are used from one thread anyway.

```objc
@property (atomic, copy) NSString *title;      // same as leaving it out
@property (nonatomic, copy) NSString *name;    // faster, no locking
```