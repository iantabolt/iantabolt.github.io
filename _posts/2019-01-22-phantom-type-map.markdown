---
title:  "Phantom Type Safe Map"
date:   2018-01-22 14:39:47
categories: [scala]
tags: [scala]
comments: true
---

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
to have a library for NLP text annotations. We might have a `annotations: TypesafeMap` and define
`case object Tokens extends Key[List[String]]`, `case object PartOfSpeech extends Key[List[POSTag]]`, 
`case object AvgSentenceLength extends Key[Double]` etc, and this will leave it open to have any 
arbitrary number of annotations, and the flexibility for users of the library to add their own 
annotations. In fact, Stanford's CoreNLP library uses [a `TypesafeMap` class] [TypesafeMap] for
exactly this.  

## Making it Safer

The problem here is that we have no way of knowing at compile time which values are present in the `Map`.
So in our text annotation example, a part of speech annotator might want to require that the text has already
passed through a tokenizer.

This will probably manifest itself as an `assert` or an `Option`/`Either` return type:
```scala
def tokenize(text: TypesafeMap): TyepsafeMap = {
  text.updated(Tokens, text.split(" ").toList
}
def posTag(text: TypesafeMap): TypesafeMap = {
  text.get(Tokens).map(tokens => {
    text.updated(PosTag, doPosTagging(tokens)
  }).getOrElse(text)  
}
```

But really we want something like

```scala
def tokenize[Text <: TypesafeMap](text: Text): Text with HasTokens = {
  text.updated(Tokens, text.split(" ").toList)
}
def posTag[Text <: TypesafeMap with HasTokens](text: Text): Text with HasPosTag = {
  text.updated(PosTag, doPosTagging(text[Tokens]))
}
```

This would let us construct a pipeline of text annotators that build off each other in a type-safe way,
without worrying about runtime `None`s or errors from missing annotations.

Here is where phantom types come in!

## Phantom Types to the Rescue
A phantom type is a type parameter that doesn't correspond to any concrete value. It will let us 
achieve the following syntax:

```scala
def tokenize[Text <: TypesafeMap](text: Text): Text with HasTokens = {
  text.updated(Tokens, text.split(" ").toList)
}
def posTag[Text <: TypesafeMap with HasTokens](text: Text): Text with HasPosTag = {
  text.updated(PosTag, doPosTagging(text[Tokens]))
}
```



  [TypesafeMap]: https://github.com/stanfordnlp/CoreNLP/blob/master/src/edu/stanford/nlp/util/TypesafeMap.java