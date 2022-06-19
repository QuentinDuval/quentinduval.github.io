---
layout: post
title: "Answering r/haskell: How to unit test code that uses polymorphic interfaces?"
description: ""
excerpt: "My answer to the question asked on r/haskell on how to test code that lives in a Monad class with polymorphic functions."
categories: [functional-programming]
tags: [Functional-Programming, Haskell]
---

This short post is an answer to the following question asked on [r/haskell](https://www.reddit.com/r/haskell/comments/7gfw3v/how_to_unit_test_code_that_uses_polymorphic/). The original question is about how to test code that lives in a Monad class with polymorphic functions.

I highly encourage you to read the post. Its different answers are full of technical gems which we are not going to explore here in this post. Instead, our goal will be to provide a simpler solution to the problem, by just playing and reworking some abstractions.

## The original problem

The OP came with the following need. She needs to model some notion of secure token management. The real implementation of the secure token management makes use of complex encryption and decryption algorithms, which she would like to abstract away from her unit tests.

### Monad type class

To abstract away the details of the encryption and decryption, and make it possible to test her code without having to deal with their real implementation, the OP introduced the following MTL-like type-class:

* `encryptToken` maps a polymorphic token to a string
* `decryptToken` maps back a string to a polymorphic token
* And a `Token` is something that can be serialized to and from JSON

```haskell
class Monad m => MonadToken m where
  encryptToken :: Token t => t -> m String
  decryptToken :: Token t => String -> m (Maybe t)

class FromJSON a where
  fromJson :: JSON -> a
 
class ToJSON a where
  toJson :: a -> JSON

class (FromJSON t, ToJSON t) => Token t
```

Let us see of example of use of this Monad. The following code encrypts a token to decrypt it immediately after and return the result of the operation (agreed, it is not really useful, except maybe for some property based testing needs):

```haskell
backAndForth :: (MonadToken m, Token t) => t -> m (Maybe t)
backAndForth t = do
  s <- encryptToken t
  decryptToken s
```

The whole approach described by the OP follows a typical [Hexagonal-like Architecture]({% link _posts/2017-07-06-hexagonal-architecture-free-monad.md %}), which is interesting for it decouples the code from the implementation of the services it relies upon (the encryption and decryption of tokens), allowing us to test it more easily.

### Testing polymorphic classes

The problem with the approach above, pointed by the OP, is that testing code that makes use of the `MonadToken` typeclass gets pretty complex due to the polymorphic functions.

A typical fake implementation would use a Map to associate tokens with their corresponding encrypted string (and vice-versa for the decryption). The problem is that building a map with polymorphic keys in Haskell is not an easy task.

There are ways to do this, and you can check some of the great answers available in [r/haskell](https://www.reddit.com/r/haskell/comments/7gfw3v/how_to_unit_test_code_that_uses_polymorphic/). They make use of advanced and pretty interesting features of Haskell (such as [Data.Typeable](https://hackage.haskell.org/package/base-4.10.0.0/docs/Data-Typeable.html) or [Data.Constraint](https://hackage.haskell.org/package/constraints)). We will instead explore a simpler solution.

## Another take at the problem

Let us try a simpler solution, that does not require using any advanced Haskell features, but instead relies on rethinking the design just a little bit.

### Looking at the types

Let us start from our use case. We are really interested in testing some code that makes use of the encryption and decryption of token, such as this code:

```haskell
backAndForth :: (MonadToken m, Token t) => t -> m (Maybe t)
backAndForth t = do
  s <- encryptToken t
  decryptToken s
```

This code makes use of the `encryptToken` and `decryptToken` polymorphic functions, whose type are given below (using `:type` in the REPL):

```haskell
encryptToken :: (Token t, MonadToken m) => t -> m String
decryptToken :: (Token t, MonadToken m) => String -> m (Maybe t)
```

Now, the fact that our code makes use of these function does not imply in any way that these functions must be part of the `MonadToken` interface. Instead, these functions could be based on lower-level function available in the `MonadToken` interface.

This is the option we will be exploring below.

### Another Monad interface

We can transform slightly our Monad interface by realizing that the only thing that we know form the token given to the `encryptToken` and `decryptToken` polymorphic functions is that:

* Our token must be serializable to JSON
* Our token must be de-serializable from JSON

So there is no loss in generality in transforming our interface `MonadToken` into the following `MonadCypher` interface, which deals with concrete JSONs instead of polymorphic tokens:

```haskell
class Monad m => MonadCypher m where
  encryptJson :: JSON -> m String
  decryptJson :: String -> m (Maybe JSON)
```

To keep our original code working, we can then build our `encryptToken` and `decryptToken` polymorphic function on top this `MonadCypher` type class:

```haskell
encryptToken :: (Token t, MonadCypher m) => t -> m String
encryptToken = encryptJson . toJson

decryptToken :: (Token t, MonadCypher m) => String -> m (Maybe t)
decryptToken str = (fmap . fmap) fromJson (decryptJson str)
```

In fact, we can simplify further these two functions. We can generalize them by realizing that `encryptToken` only needs the `ToJSON` constraint, and `decryptToken` only needs the `FromJSON` constraint. I linked the [implementation here](https://gist.github.com/deque-blog/fd2cc916721d49870a2100100872366e).

Now, let us see now how this design helps with testing.

### Testing with a Reader Monad

The key aspect in our new design is that our `MonadCypher` does not rely on polymorphic tokens anymore, but instead relies on concrete JSON values. It makes testing much easier.

To define a fake interface, we can start by defining a `Cyphers` data type that contains the necessary maps to associate JSON values to encrypted values, and vice-versa:

```haskell
data Cyphers = Cyphers {
  encryptMap :: M.Map JSON String,
  decryptMap :: M.Map String JSON
} deriving (Show, Eq, Ord)
```

From there, you know the drill. We simply wrap this data type inside a `FakeCypher` type, and implements the necessary Monad type class instances:

```haskell
— Wrapping a Reader Monad transformer (test environment)
newtype FakeCypher m a = FakeCypherT (ReaderT Cyphers m a)
  deriving (Functor, Applicative, Monad, MonadTrans)

— MonadCypher instance for our test environment
instance Monad m => MonadCypher (FakeCypher m) where
  encryptJson json = do
    cyphers <- FakeCypherT ask
    pure (encryptMap cyphers M.! json)
  decryptJson str = do
    cyphers <- FakeCypherT ask
    pure (M.lookup str (decryptMap cyphers))

— To run a `MonadCypher` computation in our test environment
runFakeCypher :: (Monad m) => FakeCypher m a -> Cyphers -> m a
runFakeCypher (FakeCypherT f) r = runReaderT f r
```

Here we are. We managed to make our code testable. And we also avoided to rely on advanced features such as [Data.Typeable](https://hackage.haskell.org/package/base-4.10.0.0/docs/Data-Typeable.html) or [Data.Constraint](https://hackage.haskell.org/package/constraints) to do so.

### Additional benefits & going further

In additional of being more easily testable, our `MonadCypher` interface also has additional benefits in comparison to the previous `MonadToken` interface.

A first advantage we get is that we are able to test more things. The faking of the encryption does not involve the faking of the serialization to JSON. A second advantage is that this design imposes less constraints on the implementation: encrypting does not require anymore the `FromJSON` constraint.

We could go a bit further and try to get to the essence of encryption and decryption, by relaxing some more constraints and being more generic. For instance, we could try to define our `MonadCypher` so that it is not defined in terms of JSON (and rather use [Data.ByteString](https://hackage.haskell.org/package/bytestring-0.10.8.2/docs/Data-ByteString.html) for instance).

## Conclusion

There is a great deal of freedom in the way we can define type-classes or interfaces. Following the precepts of programming against abstraction helps us make our test more testable, but some abstractions are easier to test against than other.

Depending on the host language, and as shown by the OP of the initial [r/haskell post](https://www.reddit.com/r/haskell/comments/7gfw3v/how_to_unit_test_code_that_uses_polymorphic/), the choice of the mean of abstraction (the notion being abstracted being the same) leads to different level of sophistication in order to implement it or fake it.

Ultimately though, we can often drastically reduce the complexity of a solution by reworking it even just slightly, typically to avoid falling into the dark corners of the language we use.
