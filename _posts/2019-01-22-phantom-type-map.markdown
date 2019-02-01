---
title:  "Phantom Type Safe Map"
date:   2019-01-22 14:39:47
categories: [scala]
tags: [scala, java, data structures, phantom types]
---

![A Phantom Map?](/images/phantom-map.png)

In this post I'll take a look at the type-safe map pattern, and how it is
used in real world open source libraries. Then I'll dive into how we can 
add phantom types to actually track at compile time which values are present
in the map.

## The Type-Safe Map

I've ran into a "type-safe `Map`" pattern in Java that looks something like this:

```java
abstract class Key<V> {}

public class TypesafeMap {
  private Map<Class<?>, Object> underlying =
    new HashMap<Class<?>, Object>();

  public <V> void set(Class<? extends Key<V>> key, V value) {
    underlying.put(key, value);
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
abstract class Key[V] {}

case class TypesafeMap(underlying: Map[Key[_], Any] = Map()) {
  def get[V](key: Key[V]): Option[V] = {
    underlying.get(key).map(_.asInstanceOf[V])
  }

  def updated[V](key: Key[V], value: V): TypesafeMap = {
    copy(underlying + (key -> value))
  }
}
```

## But Why?

A type-safe `Map` can be used as a sort of anonymous class where we can collect any set of sparse attributes.

For example, this snippet

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

In the `Person` example this seems pretty useless, but if the class had dozens or hundreds
of possible fields, where most of them are `None`, you could imagine that this could be
quite useful.

A practical example of this might be a library for NLP text annotations. 
The annotations could apply to documents, sentences, tokens, etc, and
new modules and annotations could be added without modifying the core library.

In fact, Stanford's CoreNLP library uses [a `TypesafeMap` class] [TypesafeMap] for
exactly this, and defines [hundreds of `CoreAnnotation`s] [CoreAnnotations] which 
are really just keys. 

Grepping for `implements CoreAnnotation`, we can get a sense of how this is used:

```java
src/edu/stanford/nlp/ling/SegmenterCoreAnnotations.java
10:  public static class CharactersAnnotation implements CoreAnnotation<List<CoreLabel>> {
17:  public static class XMLCharAnnotation implements CoreAnnotation<String> {

src/edu/stanford/nlp/ling/CoreAnnotations.java
47:  public static class TextAnnotation implements CoreAnnotation<String> {
60:  public static class LemmaAnnotation implements CoreAnnotation<String> {
72:  public static class PartOfSpeechAnnotation implements CoreAnnotation<String> {
85:  public static class NamedEntityTagAnnotation implements CoreAnnotation<String> {
95:  public static class NamedEntityTagProbsAnnotation implements CoreAnnotation<Map<String,Double>> {
105:  public static class CoarseNamedEntityTagAnnotation implements CoreAnnotation<String> {
115:  public static class FineGrainedNamedEntityTagAnnotation implements CoreAnnotation<String> {
// ...

src/edu/stanford/nlp/sentiment/SentimentCoreAnnotations.java
22:  public static class SentimentAnnotatedTree implements CoreAnnotation<Tree> {
34:  public static class SentimentClass implements CoreAnnotation<String> {

src/edu/stanford/nlp/ie/machinereading/structure/MachineReadingAnnotations.java
23:  public static class EntityMentionsAnnotation implements CoreAnnotation<List<EntityMention>> {
34:  public static class RelationMentionsAnnotation implements CoreAnnotation<List<RelationMention>> {
47:  public static class AllRelationMentionsAnnotation implements CoreAnnotation<List<RelationMention>> {
58:  public static class EventMentionsAnnotation implements CoreAnnotation<List<EventMention>> {
77:  public static class DocumentDirectoryAnnotation implements CoreAnnotation<String> {

// ... Many many more ...
```

In total, they define over 300 `CoreAnnotation`s in thirty different files. This would be pretty
hard to accomplish with standarad data classes. 

Another real world example that we happen to use at Foursquare can be found
in Twitter's Finagle library. Finagle uses [a type-safe map called `Stack.Params`] [StackParams]
to configure http clients and servers. In this case, they represent `Param` as a type class
instance that acts as a key and also provides a default value.

Again, grepping for `extends Stack.Param` gives a similar pattern with dozens of fields
across a number of files:

```scala
finagle-thrift/src/main/scala/com/twitter/finagle/Thrift.scala
98:    implicit object ClientId extends Stack.Param[ClientId] {
103:    implicit object ProtocolFactory extends Stack.Param[ProtocolFactory] {
115:    implicit object Framed extends Stack.Param[Framed] {
125:    implicit object AttemptTTwitterUpgrade extends Stack.Param[AttemptTTwitterUpgrade] {
311:      implicit object MaxReusableBufferSize extends Stack.Param[MaxReusableBufferSize] {

finagle-http/src/main/scala/com/twitter/finagle/Http.scala
68:    implicit object HttpImpl extends Stack.Param[HttpImpl] {
83:    implicit object MaxChunkSize extends Stack.Param[MaxChunkSize] {
91:    implicit object MaxHeaderSize extends Stack.Param[MaxHeaderSize] {
99:    implicit object MaxInitialLineSize extends Stack.Param[MaxInitialLineSize] {
107:    implicit object MaxRequestSize extends Stack.Param[MaxRequestSize] {
115:    implicit object MaxResponseSize extends Stack.Param[MaxResponseSize] {
120:    implicit object Streaming extends Stack.Param[Streaming] {
125:    implicit object Decompression extends Stack.Param[Decompression] {
130:    implicit object CompressionLevel extends Stack.Param[CompressionLevel] {

finagle-core/src/main/scala/com/twitter/finagle/loadbalancer/LoadBalancerFactory.scala
29:  implicit object EnableProbation extends Stack.Param[EnableProbation] {

finagle-core/src/main/scala/com/twitter/finagle/service/Retries.scala
64:  object Budget extends Stack.Param[Budget] {

finagle-mysql/src/main/scala/com/twitter/finagle/mysql/Handshake.scala
38:  implicit object Credentials extends Stack.Param[Credentials] {
47:  implicit object Database extends Stack.Param[Database] {
56:  implicit object Charset extends Stack.Param[Charset] {

finagle-netty4/src/main/scala/com/twitter/finagle/netty4/package.scala
68:    private[netty4] implicit object Allocator extends Stack.Param[Allocator] {
85:    implicit object WorkerPool extends Stack.Param[WorkerPool] {

finagle-mux/src/main/scala/com/twitter/finagle/mux/lease/exp/Lessor.scala
61:  implicit object Param extends Stack.Param[Param] {
 ```

## Making it Safer

While these maps are "type safe" in that the key can guarantee the type of the value,
there is no way to tell at compile time which values are actually present in the `Map`.

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
def tokenize[T <: TypesafeMap](text: T): T with HasTokens = {
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

// Private constructor to prevent adding elements outside of updated
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

object TypesafeMap {
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

You can see the working code and tests for this post [on GitHub here] [GitHub Repo].
 


  [TypesafeMap]: https://github.com/stanfordnlp/CoreNLP/blob/master/src/edu/stanford/nlp/util/TypesafeMap.java
  [CoreAnnotations]: https://github.com/stanfordnlp/CoreNLP/blob/master/src/edu/stanford/nlp/ling/CoreAnnotations.java
  [GitHub Repo]: https://github.com/iantabolt/phantom-type-safe-map
  [StackParams]: https://twitter.github.io/finagle/docs/com/twitter/finagle/Stack$$Params
