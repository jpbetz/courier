Courier Data Binding Generator for Swift
========================================

Running the generator from the command line
-------------------------------------------

Download the latest jar from maven central:

http://repo1.maven.org/maven2/org/coursera/courier/courier-swift-generator/

Then run the generator using:
```
java -jar generator/build/libs/courier-swift-generator-<version>.jar targetPath resolverPath sourcePath1[:sourcePath2]+
```

For example, if you have `.pdsc` or `.courier` files in a `pegasus` directory, and wish to generate
`.swift` files into a `swift` directory, run:

```
java -jar generator/build/libs/courier-swift-generator-<version>.jar swift pegasus pegasus
```

Note that pegasus is used twice, once as the resolverPath (which is where the generator will search
for dependencies) and again for the source path that will be searched for all schemas to generate
data bindings for.

Getting started with Xcode
--------------------------

Add SwiftyJSON to your Xcode project (https://github.com/SwiftyJSON/SwiftyJSON#integration):

If you use Cocopods, update your Podfile to include SwiftyJSON, e.g.:

```
platform :osx, '10.10'
use_frameworks!

target 'PROJECT_NAME' do
  pod 'SwiftyJSON', :git => 'https://github.com/SwiftyJSON/SwiftyJSON.git'
end
```

Be sure to replace `PROJECT_NAME` with the name of your Xcode project.

And then run:

```
pod install
```

Running the generator from within Xcode
---------------------------------------

1. In the root directory of your xcode project, add a directory for pegasus schemas, e.g. `pegasus`.
2. Also add a directory for the generator jar, e.g. `bin`.
3. Download the jar from http://repo1.maven.org/maven2/org/coursera/courier/courier-swift-generator/
into the directory created for it, e.g. `bin`.
4. In Xcode, go to `Project` -> `Build Phases`
5. Add a Run Script Phase before the compile phase, rename it to something like "Courier Data Binding Generator".
6. Select a target directory to generate swift data bindings into and create it, e.g. `courier`.
7. Set the script to something like:

```
GENERATOR_JAR=$SRCROOT/bin/courier-swift-generator-0.14.0.jar
SCHEMA_ROOT=$SRCROOT/pegasus
TARGET_DIR=$SRCROOT/courier # or $DERIVED_FILE_DIR/courier if you prefer

java -jar $GENERATOR_JAR $TARGET_DIR $SCHEMA_ROOT $SCHEMA_ROOT
```

8. Run the Xcode `build` (command-B).
9. Add all the files generated into the target directory to your Xcode project sources

How code is generated
---------------------

**Records:**

* Records are generated as a struct.

E.g.:

```swift
struct Fortune: JSONSerializable {
    /**
    The fortune telling.
    */
    let telling: Telling
    let createdAt: DateTime

    static func readJSON(json: JSON) -> Fortune
    func writeJSON() -> JSON
}
```

**Enums:**

* Enums are represented as a enum.
* An `UNKNOWN$` symbol is also generated and represents unrecognized enums symbols.  This exists to
  make wire compatibility issues easier to manage.

E.g.:

```swift
enum MagicEightBallAnswer {
    case IT_IS_CERTAIN
    case ASK_AGAIN_LATER
    case OUTLOOK_NOT_SO_GOOD
    case UNKNOWN$(String)
}
```

**Arrays:**

* Arrays are represented as a swift array.

**Maps:**

* Maps are represented as a swift dictionary.

**Unions:**

* Unions are represented as a swift enum with a "wrapper" enum case generated for each union member.
* A `UNKNOWN$` member is also generated for each union to represent unrecognized union members. This exists to
  make wire compatibility issues easier to manage.
For example, given a union "AnswerFormat" with member types "TextEntry" and "MultipleChoice", the
Java class signatures will be:

```swift
enum AnswerFormat: JSONSerializable {
    case TextEntryMember(TextEntry)
    case MultipleChoiceMember(MultipleChoice)
    case UNKNOWN$([String, JSON])

    static func readJSON(json: JSON) -> AnswerFormat
    func writeJSON() -> JSON
}
```

Immutable Value Types
---------------------

All generate Swift bindings are immutable value types.  All fields are declared using "let" and
structs are generated instead of classes.

Equatable/Hashable
------------------

TODO: This feature is at risk due to http://stackoverflow.com/questions/33377761/swift-equality-operator-on-nested-arrays.
We may revisit the issue if it is urgently needed. In the meantime we'll continue to discuss the limitation with any
Swift experts we can to see if there is a workaround.  Worst case we can define == for various combinations of nested collections up to
some reasonable depth.

Equatable and Hashable would check for "structural" equality.

For example:

```swift

// Equatable:
Example(array: [Note("hello"), Note("bye!")]) == Example(array: [Note("hello"), Note("bye!")]) // -> true
Example(array: [Note("hello"), Note("bye!")]) == Example(array: [Note("hello"), Note(""XXXX")]) // -> false

// Hashable:
Example(array: [Note("hello"), Note("bye!")]).hashvalue == Example(array: [Note("hello"), Note("bye!")]).hashvalue // -> true
```

Projections and Optionality
---------------------------

When using REST frameworks like Naptime, it is common to send and receive partial data.  This
is very commonly used when a subset of fields of a resources are "projected".

Since even fields that are marked as required in a Pegasus schema may be absent when data is
projected, Courier's [Optionality](https://github.com/coursera/courier/blob/master/swift/generator/src/main/java/org/coursera/courier/swift/SwiftProperties.java#L31)
settings defaults to REQUIRED_FIELDS_MAY_BE_ABSENT.  This allows a single
generated Swift struct to be used for bindings to unprojected and projected data.

If this behaviour is not desired, one may set `Optionality` to `STRICT`.

See the [Optionality property docs](https://github.com/coursera/courier/blob/master/swift/generator/src/main/java/org/coursera/courier/swift/SwiftProperties.java#L8)
for details on how to set the `Optionality` property.

Namespaces
----------

TODO: implement

Pegasus types are namespaced.  E.g. "org.example.User" and "org.oauth.User" are considered distinct
types because, even though they have the same name, they are in separate namespaces.

To prevent name collisions, we generate swift bindings within structs that serve as namespaces, e.g.:

```
struct example {
  struct User {
    ...
  }
}

struct oauth {
  struct User {
    ...
  }
}
```

Example usage:

```
let user = example.User()
```

Insignificant namespace parts (such as the "org" in the above example) are ignored.

Custom Types and Coercers
-------------------------

Custom Types allow any Swift type to be bound to any pegasus primitive type.

For example, say a schema has been defined to represent a "date time" as a unix timestamp long:

```
namespace org.example

typeref DateTime = long
```

And we want to use `NSDate` in our Swift code to represent this type.

First, we would write a coercer with the `Coercer` protocol does implements necessary conversion
functions:

```swift
public struct NSDateCoercer: Coercer {
    public typealias CustomType = NSDate
    public typealias DirectType = Int

    public static func coerceInput(value: Int) throws -> NSDate {
        return NSDate(timeIntervalSince1970: Double(value) / 1000)
    }

    public static func coerceOutput(value: NSDate) -> Int {
        return Int(value.timeIntervalSince1970 * 1000)
    }
}
```

Then we register both the coercer and the `NSDate` class in the schema:

```
namespace org.example

@swift.class = "Foundation.NSDate"
@swift.coercerClass = "Coercers.NSDateCoercer"
typeref DateTime = long
```

Once this has been done, a schema using this type, e.g.:

```
namespace org.example

record WithDateTime {
  createdAt: DateTime
}
```

will be generated to Swift as:

```swift
public class WithDateTime: Serializable {
  let createdAt: NSDate

  // ...
}
```

And `readJSON` and `writeJSON` will call the conversion methods on `NSDateCoercer` coercer when reading
and writing JSON.

JSON Serialization
------------------

JSON serialization is supported using generated `read()` and `write()` methods.  These methods
are generated to use SwiftyJSON's `JSON` type.

For example, to read JSON:

```swift
let json = JSON("{ \"body\": \"Hello Pegasus!\"}")
let message = Message.readJSON(json)
message.writeJSON() // -> { "body": "Hello Pegasus!" }
```

NSCoding Serialization
----------------------

```swift
// serialize to NSData:
let message = Message(...)
let archived = NSKeyedArchiver.archivedDataWithRootObject(message.writeData())

// deserialize from NSData:
if let unarchived = NSKeyedUnarchiver.unarchiveObjectWithData(archived) as? [String: AnyObject] {
  let deserialized = Message.readData(unarchived)
  // do something
}
```


Binary Protocols
----------------

TODO: Implement PSON or one of the other wire format binary protocols used by the Pegasus ecosystem.


Runtime library
---------------

All generated swift bindings depend on a `CourierRuntime.swift` class. This class builds on
SwiftyJSON and Foundation classes to define minimal set of functions used by the generator to
produce clean source code.

Building from source
--------------------

See the main CONTRIBUTING document for details.

Building a Fat Jar
------------------

```sh
$ sbt
> project swift-generator
> assembly
```

This will build a standalone "fat jar". This is particularly convenient for use as a standalone
commandline application that can be called from Xcode build steps.

Testing
-------

1. Unit tests that are run by executing: `./gradlew test`.
2. Swift code is compiled and tested for correctness by the `testsuite` Xcode project.  To test, open
the project and run `Test` (command-U).

TODO
----

* [ ] Figure out the best way to distribute the 'fat jar'.
* [ ] Automate distribution of the Fat Jar, and generally make the distribution sane
* [ ] Publish Fat Jar to remote repos?
* [ ] Look into adding NSCoding support (see:
  http://swiftandpainless.com/nscoding-and-swift-structs/?utm_campaign=Swift+Developer+Weekly&utm_medium=email&utm_source=Swift_Developer_Weekly_27
  and https://github.com/nicklockwood/AutoCoding)
* [ ] Implement namespace handling strategy (details below)
* [ ] Generate scala style copy() methods?
* [ ] Dig deeper into Equatable, Hashable (how to support arrays and maps?  Deep check?)
* [ ] Add typed map key support (requires Equatable/Hashable)
* [ ] Make CourierRuntime.swift available as a Pod?
* [ ] Support recursively defined types (RecursivelyDefinedRecord.swift does not compile)
* [ ] typerefs (anything left to do?)
* [ ] custom types (what to do?)
