---
title: Translation Reference
layout: docs
---

# Translation Reference

## Types
* For primitive types, J2ObjC has defined JNI-style typedefs.
* For typical class types, the package is camel-cased and prepended to the class name.
  * To rename the generated package prefix see [Package Prefixes](Package-Prefixes.html).
* For the base Object type and all type variables, "id" is used.
* A few core Java types are mapped to foundation types. (eg. String to NSString)
* For fields declared 'volatile', J2ObjC has more typedefs that use C11 _Atomic(...) types.
* For inner types the inner class name is appended to the outer name with an underscore.

|Java type|Objective-C type|Objective-C volatile type|
|---|---|---|
|boolean|jboolean|volatile_jboolean|
|char|jchar|volatile_jchar|
|byte|jbyte|volatile_jbyte|
|short|jshort|volatile_jshort|
|int|jint|volatile_jint|
|long|jlong|volatile_jlong|
|float|jfloat|volatile_jfloat|
|double|jdouble|volatile_jdouble|
|java.lang.Object|id|volatile_id|
|type variables|id|volatile_id|
|java.lang.String|NSString*|volatile_id|
|java.lang.Number|NSNumber*|volatile_id|
|java.lang.Cloneable|NSCopying*|volatile_id|
|foo.bar.Mumble|FooBarMumble*|volatile_id|
|foo.bar.Mumber$Inner|FooBarMumble_Inner*|volatile_id|

## Methods
Objective-C methods differ from Java methods in two important ways. They are syntactically
different, embedding the parameters in between components of the method selector. Objective-C
methods don't support overloading like Java does. These differences are resolved by embedding the
parameter types into the generated selector. This is necessary to prevent name collisions between
overloaded Java methods.

Java static methods are translated to Objective-C class methods.

Method names are generated as follows:

* Zero parameter methods are unchanged
* One or more parameter uses the following pattern:
  * `<java name>With<1st param keyword>:with<2nd param keyword>:with<3rd param keyword>:`
* Parameter keyword rules:
  * For primitive types, the keyword is the capitalized name of the Java primitive. (eg. "Char")
  * For non-primitive types, the keyword is the camel-cased fully qualified type name. (eg. "ComGoogleFoo")
  * For array types, "Array" is appended to the keyword of the element type.

##### Example Declarations
```
void foo();

- (void)foo;
```
```
static int foo();

+ (jint)foo;
```
```
String foo(int i);

- (NSString *)fooWithInt:(jint)i;
```
```
static java.util.List foo(String s, long[] l);

+ (id<JavaUtilList>)fooWithNSString:(NSString *)s
                      withLongArray:(IOSLongArray *)l;
```

## Fields

### Instance Fields (non-static)
Java instance variables become Objective-C instance variables. The name is the same with a trailing
underscore. Primitive fields declared "final" are a special case and are not translated to instance
variables.

* Fields can be accessed directly using "->" syntax.
* Primitive fields can be set directly.
  * Final primitives (constants) are translated like static constants. (see [Static Fields](#static-fields))
* Non-primitive fields must be set using the provided setter function:
  * `ClassName_set_fieldName_(instance, value)`

##### Example Java
```java
package com.google;
class Foo {
  public int myInt;
  public String myString;
}
```

##### Example Objective-C
```objc
Foo *foo = [[Foo alloc] init];

// Access a primitive field.
i = foo->myInt_;

// Set a primitive field.
foo->myInt_ = 5;

// Access a non-primitive field.
NSString *s = foo->myString_;

// Set a non-primitive field.
ComGoogleFoo_set_myString_(foo, @"bar");
```

### Static Fields
Static variables must be accessed using the provided getter and setter functions.
These accessor functions ensure that class initialization has occurred prior to accessing the variable.

* Access a static field
  * `ClassName_get_fieldName()`
* Assign a (non-final) static field
  * `ClassName_set_fieldName()`
* Get a pointer to a primitive static field
  * `ClassName_getRef_fieldName()`
  * Only available for non-final and non-volatile fields.

Final primitive fields (constants) are safe to access directly because their value does not depend on class initialization.

  * `ClassName_fieldName`

##### Example Java
```java
package com.google;
class Foo {
  public static final MY_FINAL_INT = 5;
  public static int myInt;
  public static String myString;
}
```

##### Example Objective-C
```objc
// Access a primitive constant field.
jint i = ComGoogleFoo_MY_FINAL_INT;   // No class initialization
i = ComGoogleFoo_get_MY_FINAL_INT();  // Class initialization

// Access a primitive field.
i = ComGoogleFoo_get_myInt();

// Set a primitive field.
ComGoogleFoo_set_myInt(5);

// Access a non-primitive field.
NSString *s = ComGoogleFoo_get_myString();

// Set a non-primitive field.
ComGoogleFoo_set_myString(@"bar");
```

## Enums
J2ObjC generates two types for each Java enum. An Objective-C class type is generated which provides
the full functionality of a Java enum. Additionally a C enum is generated using the Foundation
framework's NS_ENUM macro. The Objective-C class type is used by all generated API. The C enum is
useful as constant values for a switch statement, or as a storage type.

The generated enum types are named as follows:

* The Objective-C class is named using the same rule as a regular Java class. (see [Types](#Types))
* The C enum is named as a regular Java class with an added "_Enum" suffix.

Enum constants are accessed like static fields.

##### Example Java
```java
package com.google;
enum Color {
  RED, GREEN, BLUE
}
```

##### Example Objective-C Header
```objc
typedef NS_ENUM(NSUInteger, ComGoogleColor_Enum) {
  ComGoogleColor_Enum_RED = 0;
  ComGoogleColor_Enum_GREEN = 1;
  ComGoogleColor_Enum_BLUE = 2;
};

@interface ComGoogleColor : JavaLangEnum < NSCopying >
+ (IOSObjectArray *)values;
+ (ComGoogleColor *)valueOfWithNSString:(NSString *)name;
@end

inline ComGoogleColor *ComGoogleColor_get_RED();
inline ComGoogleColor *ComGoogleColor_get_GREEN();
inline ComGoogleColor *ComGoogleColor_get_BLUE();

// Provides conversion from ComGoogleColor_Enum values.
FOUNDATION_EXPORT ComGoogleColor *ComGoogleColor_fromOrdinal(NSUInteger ordinal);
```