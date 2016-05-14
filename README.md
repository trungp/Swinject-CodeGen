# Swinject Code Generation

[![Build Status](https://travis-ci.org/Swinject/Swinject-CodeGen.svg?branch=master)](https://travis-ci.org/Swinject/Swinject-CodeGen)

## TLDR;

Swinject-CodeGen provides a method to get rid of duplicate use of class values and namestrings, by generating explicit functions for registering and resolving using Swinject.
Doing this, we also can generate typed tuples to use when resolving, thus allowing better documented and less error-prone code.

## Installation
### Cocoapods

Add

```
pod 'Swinject-CodeGen'
```

to your podfile.

### Carthage

Add

```
github "Swinject/Swinject-CodeGeneration"
```

to your Cartfile.

## Integration
1. Define your dependencies in a .csv or .yml file (see below and example file)
2. Add a call to generate the code as build script phase:

For Cocoapods:
```Shell
$PODS_ROOT/Swinject-CodeGen/bin/swinject_codegen -i baseInput.csv -o extensions/baseContainerExtension.swift
```

For Carthage:
```Shell
$SRCROOT/Carthage/Checkouts/Swinject-CodeGen/bin/swinject_codegen -i baseInput.csv -o extensions/baseContainerExtension.swift
```

3. Add the generated file (here: `extensions/baseContainerExtension.swift`) to xcode
4. Repeat if you need to support multiple targets/have multiple input files.

The code is then generated at every build run.

## The Issue

When using Swinject, lots of duplicate definitions appear, whenever we do a

```Swift
container.register(PersonType.self, name: "initializer") { r in
    InjectablePerson(pet: r.resolve(AnimalType.self)!)
}

let initializerInjection = container.resolve(PersonType.self, name:"initializer")!
```

the tuple (PersonType.self, name:"initializer") becomes very redundant across the code.

Furthermore, when using arguments, as done in

```Swift
container.register(AnimalType.self) { _, name in Horse(name: name) }
let horse1 = container.resolve(AnimalType.self, argument: "Spirit") as! Horse
```

the `argument: "Spirit"` part is not strictly typed when calling it.

We propose a solution to both these problems by using CodeGeneration

## Input Format
Input can be given as .csv or .yml

The call
```
./swinject_codegen -i example.csv -c
```

can be used to convert example.csv into example.csv.yml (also works for .yml).

### CSV

#### Basic Structure

Our basic csv structure is defined as follows:

```CSV
SourceClassName; TargetClassName; Identifier; Argument 1 ... 9
```

The example above would translate to

```CSV
PersonType; InjectablePerson; initializer
```

to generate both a `registerPersonType_initializer` and a `resolvePersonType_initializer` function.

See the examples below for more examples.

We decided to use `;` as delimiter instead of `,` to allow the use of tuples as types.

#### Additional Commands
The ruby parser allows using  `//` and `#` for comments.
Empty lines are ignored and can be used for grouping.

`#= <header>` can be used to specify additional lines, e.g. `#= import KeychainAccess`

#### Dictionaries and Arrays as Parameters
When using typed dictionaries or arrays as parameters, use `Array<Type>` instead of `[Type]` and `Dictionary<TypeA, TypeB>` instead of `[TypeA:TypeB]`:

```CSV
PersonType; InjectablePerson; initializer; additionalNames:Array<String>; family:Dictionary<String, String>;
```

### YML

Example for a .yml definition:
```yml
---
HEADERS:
  - import ADependency
DEFINITIONS:
- service: PersonType
  component: InjectablePerson
  name: initializer
- service: PersonType
  component: InjectablePerson
- service: PersonType
  component: PersonType
- service: AnotherPersonType
  component: AnotherPersonType
- service: PersonType
  component: InjectablePerson
  arguments:
  - argument_name: argument_name
    argument_type: argument_type
- service: PersonType
  component: InjectablePerson
  arguments:
  - argument_name: argument_name
    argument_type: argument_type
  - argument_name: argument_typewithoutspecificname
    argument_type: argument_typeWithoutSpecificName
  - argument_name: title
    argument_type: String
  - argument_name: string
    argument_type: String
- service: PersonType
  component: InjectablePerson
  name: initializer
  arguments:
  - argument_name: argument_name
    argument_type: argument_type
  - argument_name: argument_typewithoutspecificname
    argument_type: argument_typeWithoutSpecificName
  - argument_name: title
    argument_type: String
  - argument_name: string
    argument_type: String
```

## Generation Examples


### Example A: Same class as source and target

#### Input
```CSV
PersonType
```

#### Output
```Swift
// this code is autogenerated, do not modify!

import Swinject

extension Resolvable {

    func resolvePersonType() -> PersonType {
        return self.resolve(PersonType.self)!
    }
}

extension Container {

    func registerPersonType(registerClosure: (resolver: ResolverType) -> (PersonType)) -> ServiceEntry<PersonType> {
        return self.register(PersonType.self, factory: registerClosure)
    }
}
```

### Example B: Different source and target

#### Input
```CSV
PersonType; InjectablePerson
```

#### Output
```Swift
// this code is autogenerated, do not modify!

import Swinject

extension Resolvable {

    func resolveInjectablePerson() -> InjectablePerson {
        return self.resolve(PersonType.self) as! InjectablePerson
    }
}

extension Container {

    func registerInjectablePerson(registerClosure: (resolver: ResolverType) -> (InjectablePerson)) -> ServiceEntry<PersonType> {
        return self.register(PersonType.self, factory: registerClosure)
    }
}
```

### Example C: Different source and target class with name

#### Input
```CSV
PersonType; InjectablePerson; initializer
```

#### Output
```Swift
// this code is autogenerated, do not modify!

import Swinject

extension Resolvable {

    func resolveInjectablePerson_initializer() -> InjectablePerson {
        return self.resolve(PersonType.self, name: "initializer") as! InjectablePerson
    }
}

extension Container {

    func registerInjectablePerson_initializer(registerClosure: (resolver: ResolverType) -> (InjectablePerson)) -> ServiceEntry<PersonType> {
        return self.register(PersonType.self, name: "initializer", factory: registerClosure)
    }
}
```

### Example D: Different source and target with a single, explicitly named argument
#### Input
```CSV
PersonType; InjectablePerson; ; argumentName:ArgumentType
```

#### Output
```Swift
// this code is autogenerated, do not modify!

import Swinject

extension Resolvable {

    func resolveInjectablePerson(argumentName argumentName: ArgumentType) -> InjectablePerson {
        return self.resolve(PersonType.self, argument: argumentName) as! InjectablePerson
    }
}

extension Container {

    func registerInjectablePerson(registerClosure: (resolver: ResolverType, argumentName: ArgumentType) -> (InjectablePerson)) -> ServiceEntry<PersonType> {
        return self.register(PersonType.self, factory: registerClosure)
    }
}
```

### Example E: Different source and target with multiple arguments, both explicitly named and not
If no explicit name is given, the lowercase type is used as argumentname.

#### Input
```CSV
PersonType; InjectablePerson; ; argumentName:ArgumentType; ArgumentTypeWithoutSpecificName; title:String; String
```

#### Output
```Swift
// this code is autogenerated, do not modify!

import Swinject

extension Resolvable {

    func resolveInjectablePerson(argumentName argumentName: ArgumentType, argumenttypewithoutspecificname: ArgumentTypeWithoutSpecificName, title: String, string: String) -> InjectablePerson {
        return self.resolve(PersonType.self, arguments: (argumentName, argumenttypewithoutspecificname, title, string)) as! InjectablePerson
    }
}

extension Container {

    func registerInjectablePerson(registerClosure: (resolver: ResolverType, argumentName: ArgumentType, argumenttypewithoutspecificname: ArgumentTypeWithoutSpecificName, title: String, string: String) -> (InjectablePerson)) -> ServiceEntry<PersonType> {
        return self.register(PersonType.self, factory: registerClosure)
    }
}
```

### Example F:  Different source and target with name with multiple arguments, both explicitly named and not
#### Input
```CSV
PersonType; InjectablePerson; initializer; argumentName:ArgumentType; ArgumentTypeWithoutSpecificName; title:String; String
```

#### Output
```Swift
// this code is autogenerated, do not modify!

import Swinject

extension Resolvable {

    func resolveInjectablePerson_initializer(argumentName argumentName: ArgumentType, argumenttypewithoutspecificname: ArgumentTypeWithoutSpecificName, title: String, string: String) -> InjectablePerson {
        return self.resolve(PersonType.self, name: "initializer", arguments: (argumentName, argumenttypewithoutspecificname, title, string)) as! InjectablePerson
    }
}

extension Container {

    func registerInjectablePerson_initializer(registerClosure: (resolver: ResolverType, argumentName: ArgumentType, argumenttypewithoutspecificname: ArgumentTypeWithoutSpecificName, title: String, string: String) -> (InjectablePerson)) -> ServiceEntry<PersonType> {
        return self.register(PersonType.self, name: "initializer", factory: registerClosure)
    }
}
```

## Usage Examples

Using the examples given at the beginning, we can now instead of

```Swift
container.register(PersonType.self, name: "initializer") { r in
    InjectablePerson(pet: r.resolve(AnimalType.self)!)
}

let initializerInjection = container.resolve(PersonType.self, name:"initializer")!
```

write:

```Swift
container.registerPersonType_initializer { r in
    InjectablePerson(pet: r.resolve(AnimalType.self)!)
}

let initializerInjection = container.resolvePersonType_initializer()
```

Also

```Swift
container.register(AnimalType.self) { _, name in Horse(name: name) }
let horse1 = container.resolve(AnimalType.self, argument: "Spirit")
```

becomes

```Swift
container.registerAnimalType { (_, name:String) in
  Horse(name: name)
}
let horse1 = container.resolveAnimalType("Spirit")
```

## Migration
The script also generates migration.sh files (when using the -m switch), which use sed to go through the code and replace simple cases (i.e. no arguments) of resolve and register.
No automatic migration is available for cases with arguments, yet.
Simply call the .sh file from the root of the project and compare the results in a git-GUI.

## Results
We currently use the code generation in two medium-sized apps across tvOS and iOS.

We found our code to become much more convenient to read and write, due to reduced duplication and autocompletion.
We also have a much better overview the classes available through dependency injection.
Changing some definition immediately leads to information, where an error will occur.
We were able to replace all our occurences of `.resolve(` and `.register(` using the current implementation.

## Contributors
The original idea for combining CodeGeneration and Swinject came from [Daniel Dengler](https://github.com/ddengler), [David Kraus](https://github.com/davidkraus) and [Wolfgang Lutz](https://github.com/lutzifer).
