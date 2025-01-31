---
sidebar_label: Customisation
title: Customisation
---

Smithy4s is opinionated in what the generated code look like, there are a few things that can be tweaked.

## Packed inputs

By default, Smithy4s generates methods the parameters of which map to the fields of the input structure of the corresponding operation.

For instance :

```kotlin
service PackedInputsService {
  version: "1.0.0"
  operations: [PackedInputOperation]
}

operation PackedInputOperation {
  input: PackedInput
}

structure PackedInput {
    @required
    a: String
    @required
    b: String
}
```

leads to something conceptually equivalent to :

```scala
trait PackedInputServiceGen[F[_]] {

  def packedInputOperation(a: String, b: String) : F[Unit]

}
```

It is however possible to annotate the service (or operation) definition with the `smithy4s.meta#packedInputs` trait, in order for the rendered method to contain a single parameter, typed with actual input case class of the operation.

For instance :

```scala
use smithy4s.meta#packedInputs

@packedInputs
service PackedInputsService {
  version: "1.0.0"
  operations: [PackedInputOperation]
}
```

will produce the following Scala code

```scala
trait PackedInputServiceGen[F[_]] {

  def packedInputOperation(input: PackedInput) : F[Unit]

}
```

## ADT Member Trait

The default behavior of Smithy4s when rendering unions that target structures is to render the structure
in a separate file from the union that targets it. This makes sense if the structure is used in other
contexts other than the union. However, it also causes an extra level of nesting within the union.
This is because the union will create another case class to contain your structure case class.

For example:

```kotlin
union OrderType {
  inStore: InStoreOrder
}

structure InStoreOrder {
    @required
    id: OrderNumber
    locationId: String
}
```

Would render the following scala code:

OrderType.scala:
```scala
sealed trait OrderType extends scala.Product with scala.Serializable
case class InStoreCase(inStore: InStoreOrder) extends OrderType
```

InStoreOrder.scala:
```scala
case class InStoreOrder(id: OrderNumber, locationId: Option[String] = None)
```

The sealed hierarchy `OrderType` has a member named `InStoreCase`. This is because
`InStoreOrder` is rendered in a separate file and `OrderType` is sealed.

However, adding the `adtMember` trait to the `InStoreOrder` structure changes this.

```kotlin
union OrderType {
  inStore: InStoreOrder
}

@adtMember(OrderType) // added the adtMember trait here
structure InStoreOrder {
    @required
    id: OrderNumber
    locationId: String
}
```

```scala
sealed trait OrderType extends scala.Product with scala.Serializable
case class InStoreOrder(id: OrderNumber, locationId: Option[String] = None) extends OrderType
```

The `IsStoreOrder` class has now been updated to be rendered directly as a member of the `OrderType`
sealed hierarchy.

*The `adtMember` trait can be applied to any structure as long as said structure is targeted by EXACTLY ONE union.*
This means it must be targeted by the union that is provided as parameter to the adtMember trait.
This constraint is fulfilled above because `OrderType` targets `InStoreOrder` and `InStoreOrder` is
annotated with `@adtMember(OrderType)`.
The structure annotated with `adtMember` (e.g. `InStoreOrder`) also must not be targeted by any other
structures or unions in the model. There is a validator that will make sure these requirements are met
whenever the `adtMember` trait is in use.

Note: The `adtMember` trait has NO impact on the serialization/deserialization behaviors of Smithy4s.
The only thing it changes is what the generated code looks like. This is accomplished by keeping the
rendered schemas equivalent, even if the case class is rendered in a different place.

## Specialized collection types

Smithy supports `list` and `set`, Smithy4s renders that to `List[A]` and `Set[A]` respectively. You can also use the `@uniqueItems` annotation on `list` which is equivalent to `set`.

Smithy4s has support for two specialized collection types: `Vector` and `IndexedSeq`. The following examples show how to use them:

```kotlin
use smithy4s.meta#indexedSeq
use smithy4s.meta#vector

@indexedSeq
list SomeIndexSeq {
  member: String
}

@vector
list SomeVector {
  member: String
}
```

Both annotations are only applicable on `list` shapes. You can't mix `@vector` with `@indexedSeq`, and neither one can be used with `@uniqueItems`.

## Refinements

Refinements provide a mechanism for using types that you control inside the code generated by smithy4s. Creating a refinement for use in your application starts with creating a custom smithy trait that represents the refinement.

```kotlin
namespace test

@trait(selector: "string")
structure emailFormat {}
```

This trait can now be used on `string` shapes to indicate that they must match an email format.

```kotlin
@emailFormat
string Email
```

Now we need to tell smithy4s that we want to represent shapes annotated with `@emailFormat` as a custom type that we define.

Given a custom email type such as:

```scala mdoc:silent
// Note, we recommend using a newtype library over a regular case class in most cases
// But this is shown to simplify the example
case class Email(value: String)
object Email {

  private def isValidEmail(value: String): Boolean = ???

  def apply(value: String): Either[String, Email] =
    if (isValidEmail(value)) Right(new Email(value))
    else Left("Email is not valid")
}
```

Next, we will need to provide a way for smithy4s to understand how to construct and deconstruct our `Email` type. We do this by defining an instance of a `RefinementProvider`. Note that the `RefinementProvider` we create MUST be `implicit`.

```scala mdoc:reset:invisible
// this is just here so the lower blocks will compile
import smithy4s.schema.Schema._
case class EmailFormat()
object EmailFormat extends smithy4s.ShapeTag.Companion[EmailFormat] {
  val id: smithy4s.ShapeId = smithy4s.ShapeId("smithy4s.example", "emailFormat")

  val hints : smithy4s.Hints = smithy4s.Hints.empty

    implicit val schema: smithy4s.Schema[EmailFormat] = constant(EmailFormat()).withId(id).addHints(hints)

}
```

```scala mdoc:silent
// package myapp.types
import smithy4s._

case class Email(value: String)
object Email {

  private def isValidEmail(value: String): Boolean = ???

  def apply(value: String): Either[String, Email] =
    if (isValidEmail(value)) Right(new Email(value))
    else Left("Email is not valid")

  // highlight-start
  implicit val provider = Refinement.drivenBy[EmailFormat](
    Email.apply, // Tells smithy4s how to create an Email (or get an error message) given a string
    (e: Email) => e.value // Tells smithy4s how to get a string from an Email
  )
  // highlight-end
}
```

:::info

The `EmailFormat` type passed as a type parameter to `Refinement.drivenBy` is the type that smithy4s generated from our `@emailFormat` trait we defined in our smithy file earlier.

:::

Now, we just have one thing left to do: tell smithy4s where to find our custom `Email` type. We do this using a trait called `smithy4s.meta#refinement`.

```kotlin
use smithy4s.meta#refinement

apply test#emailFormat @refinement(
  targetType: "myapp.types.Email"
)
```

Here we are applying the refinement trait to our `emailFormat` trait we defined earlier. We are providing the `targetType` which is our `Email` case class we defined.

Smithy4s will now be able to update how it does code generation to reference our custom `Email` type.

:::info

If the provider was not in the companion object of our `targetType`, we would need to provide the `providerImport` to the `refinement` trait
so that smithy4s would be able to find it. For example:

```kotlin
use smithy4s.meta#refinement

apply test#emailFormat @refinement(
  targetType: "myapp.types.Email",
  providerImport: "myapp.types.providers._"
)
```

Whether the provider is in the companion object or not, it must be `implicit`.

:::

## Unwrapping

By default, smithy4s will wrap all standalone primitive types in a Newtype. A standalone primitive type is one that is defined like the following:

```kotlin
string Email // standalone primitive

structure Test {
  email: Email
  other: String // not a standalone primitive
}
```

Given this example, smithy4s would generate something like the following:

```scala
final case class Test(email: Email, other: String)
```

This wrapping may be undesirable in some circumstances. As such, we've provided the `smithy4s.meta#unwrap` trait. This trait tells the smithy4s code generation to not wrap these types in a newtype when they are used.

```kotlin
use smithy4s.meta#unwrap

@unwrap
string Email

structure Test {
  email: Email
  other: String
}
```

This would now generate something like:

```scala
final case class Test(email: String, other: String)
```

This can be particularly useful when working with refinement types (see above for details on refinements). By default, any type that is `refined` will be generated inside of a newtype. If you don't want this, you can mark the type with the `unwrap` trait.

```kotlin
@trait(selector: "string")
structure emailFormat {}

@emailFormat
@unwrap
string Email
```

:::info

By default, smithy4s renders collection types as unwrapped EXCEPT when the collection has been refined. In this case, the collection will be rendered within a newtype by default. If you wish your refined collection be rendered unwrapped, you can accomplish this using the same `@unwrap` trait annotation on it.

:::

## Default rendering

Smithy4s allows you to customize how defaults on the fields of smithy structures are rendered inside of case classes. There are three options:

- `FULL`
- `OPTION_ONLY`
- `NONE`

The default is `FULL`.

This value is set using metadata which means that the setting will be applied to all the rendering done by smithy4s.

#### FULL

`FULL` means that default values are rendered for all field types. For example:

```kotlin
metadata smithy4sDefaultRenderMode = "FULL"

structure FullExample {
  one: Integer = 1
  two: String
  @required
  three: String
}
```

would render to something like:

```scala
case class FullExample(three: String, one: Int = 1, two: Option[String] = None)
```

Notice how the fields above are ordered. The reason for this is that fields are ordered as:

1. Required Fields
2. Fields with defaults
3. Optional Fields

#### OPTION_ONLY

```kotlin
metadata smithy4sDefaultRenderMode = "OPTION_ONLY"

structure OptionExample {
  one: Integer = 1
  two: String
  @required
  three: String
}
```

would render to something like:

```scala
case class FullExample(one: String, three: String, two: Option[String] = None)
```

Now `one` doesn't have a default rendered and as such it is placed first in the case class.

#### NONE

```kotlin
metadata smithy4sDefaultRenderMode = "NONE"

structure OptionExample {
  one: Integer = 1
  two: String
  @required
  three: String
}
```

would render to something like:

```scala
case class FullExample(one: String, two: Option[String], three: String)
```

Now none of the fields are rendered with defaults. As such, the order of the fields is the same as is defined in the smithy structure.

:::caution

The presence of the `smithy4sDefaultRenderMode` metadata does NOT change the way smithy4s codecs behave. As such, defaults will still be used when decoding
fields inside of clients and servers. This feature is purely for changing the generated code for your convenience.

:::
