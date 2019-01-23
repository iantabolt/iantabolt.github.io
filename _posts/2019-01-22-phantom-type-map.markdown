---
title:  "Phantom Type Safe Map"
date:   2019-01-22 14:39:47
categories: [scala]
tags: [scala, java, data structures, phantom types]
---

In this post I'll dive into how we can use phantom types to build a type-safe 
map that can hold arbitrary data while still providing compile-time safety.

## The Type-Safe Map

I've ran into a "type-safe `Map`" pattern in Java that looks something like this:

```java
interface Key<T> {}

public class TypesafeMap {
  private Map<Class<?>, Object> underlying =
    new HashMap<Class<?>, Object>();

  public <V> void set(Class<? extends Key<V>> key, V thing) {
    underlying.put(key, thing);
  }

  public <V> V get(Class<? extends Key<V>> key) {
    return (V) underlying.get(key);
  }
}
```

The map is really a `Map[Class[_], Any]` but the keys tell you what type the value is, 
so we can do "type-safe" `get` and `set` operations while hiding the underlying casting. 

An immutable version in Scala might look something like this:

```scala
trait Key[T]

case class TypesafeMap private (underlying: Map[Key[_], Any] = Map()) {
  def get[T](key: Key[T]): Option[T] = {
    underlying.get(key).map(_.asInstanceOf[T])
  }

  def updated[T](key: Key[T], value: T): TypesafeMap = {
    copy(underlying + (key -> value))
  }
}
```

## But Why?

A type-safe `Map` can be used as a sort of anonymous class where we can collect any set of attributes.

For example,

```scala
case class Person(
  name: Option[String] = None,
  age: Option[Int] = None,
  children: Option[List[Person]] = None,
  birthYear: Option[Int] = None,
  address: Option[String] = None
)

val person = Person(
  age = Some(40),
  address = Some("123 Main St")
)
```
can be expressed as
```scala
case object Name extends Key[String]
case object Age extends Key[Int]
case object Children extends Key[List[TypesafeMap]]
case object BirthYear extends Key[Int]
case object Address extends Key[String]

val person = TypesafeMap()
  .updated(Age, 40)
  .updated(Address, "123 Main St")  
```

Unlike a case class, this gives more flexibility to add arbitrarily more "fields" later on 
by extending `Key`.

In the `Person` example this seems pretty useless, but suppose we wanted
to have a library for NLP text annotations. We might want to annotate documents, sentences, 
tokens, etc, and we might want new modules and annotations to be added without 
modifying the core library.

Some keys might look like
```scala
case object Text extends Key[String]
case object Sentences extends Key[List[TypesafeMap]]
case object Tokens extends Key[List[TypesafeMap]]
case object Lemma extends Key[String]
case object PartOfSpeech extends Key[String]
case object NamedEntityTag extends Key[String]
case object NamedEntityTagProbs extends Key[Map[String, Double]]
case object CoarseNamedEntityTag extends Key[String]
case object FineGrainedNamedEntityTag extends Key[String]
case object StackedNamedEntityTag extends Key[String]
case object TrueCase extends Key[String]
case object TrueCaseText extends Key[String]
case object GenericTokens extends Key[List[TypesafeMap]]
case object Quotations extends Key[List[TypesafeMap]]
case object UnclosedQuotations extends Key[List[TypesafeMap]]
// ... hundreds more ...
```

In fact, Stanford's CoreNLP library uses [a `TypesafeMap` class] [TypesafeMap] for
exactly this, and defines [hundreds of `CoreAnnotation`s] [CoreAnnotations] which 
are really just keys.

## Making it Safer

The problem here is that we have no way of knowing at compile time which 
values are present in the `Map`.

So in our text annotation example, suppose we want to compose a pipeline of `Annotator`s,
or functions that add new values to our map, but depend on existing values.

This will probably manifest itself as an `assert` or an `Option`/`Either` return type:

```scala
def tokenize(text: TypesafeMap): TyepsafeMap = {
  text.updated(Tokens, text.split(" ").toList)
}
def posTag(text: TypesafeMap): TypesafeMap = {
  text.get(Tokens).map(tokens => {
    text.updated(PosTag, doPosTagging(tokens)
  }).getOrElse(text)
}

val text = TypesafeMap().updated(Text, "foo bar")
// This compiles but doesn't do anything,
// because Tokens are needed before posTag is called
val tagged = posTag(text)
```

But really we want something like

```scala
def tokenize[Text <: TypesafeMap](text: T): T with HasTokens = {
  text.updated(Tokens, text.split(" ").toList)
}
def posTag[T <: TypesafeMap with HasTokens](text: T): T with HasPosTag = {
  text.updated(PosTag, doPosTagging(text[Tokens]))
}
```

This would let us construct a pipeline of text annotators that build off each other in a type-safe way,
without worrying about runtime `None`s or errors from missing annotations.

Here is where phantom types come in!

## Phantom Types to the Rescue
A phantom type is a type parameter that doesn't correspond to any concrete
value. 

In our case we can add a phantom type parameter to `TypesafeMap` that
will keep track of what values are in our map so far. It will let us achieve
the following syntax:

```scala
case object Tokens extends Key[List[String]] 
case object PosTag extends Key[List[String]] 
case object Text extends Key[String]

def tokenize[T <: Text.Aware](
  text: TypesafeMap[T]
): TypesafeMap[T with Tokens.Aware] = {
  text.updated(Tokens, text(Text).split("""\s*\b\s*""").toList)
}

def posTag[T <: Tokens.Aware](
  text: TypesafeMap[T]
): TypesafeMap[T with PosTag.Aware] = {
  text.updated(PosTag, doPosTag(text(Tokens)))
}

val text = TypesafeMap().updated(Text, "Hello, world!")
posTag(text) // Does not compile
posTag(tokenize(text)) // Compiles!
```

where `TypesafeMap` and `Key` are now defined as

```scala
abstract class Key[V] {
  trait Aware
}

case class TypesafeMap[+T] private (underlying: Map[Key[_], Any]) {
  // Returns the value associated with this key. Will only compile
  // if the value exists.
  def apply[V](key: Key[V])(implicit has: T <:< key.Aware): V = {
    // Looks dangerous but will never throw an error!
    underlying(key).asInstanceOf[V]
  }

  // Stores the key/value and adds key.Aware to our phantom type
  def updated[V](key: Key[V], value: V): TypesafeMap[T with key.Aware] = {
    TypesafeMap[T with key.Aware](underlying + (key -> value))
  }
}

object TypeSafeMap {
  def apply(): TypesafeMap[Any] = TypesafeMap[Any](Map())
}
```

Notice that we can now use `apply[V](key: Key[V]): V` instead of 
`get[V](key: Key[V]): Option[V]`. 

But unlike a normal `Map`, `apply` will never throw a `NoSuchElementException`.
This is because the only way to add to the map is through `updated` which
appends `with key.Aware` to the phantom type parameter. 

Then the `apply` method only compiles if there is an implicit `T <:< key.Aware`, which 
asserts that there must be proof that `T` extends `key.Aware` (ie the value exists).


  [TypesafeMap]: https://github.com/stanfordnlp/CoreNLP/blob/master/src/edu/stanford/nlp/util/TypesafeMap.java
  [CoreAnnotations]: https://github.com/stanfordnlp/CoreNLP/blob/master/src/edu/stanford/nlp/ling/CoreAnnotations.java
