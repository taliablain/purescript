## Type Classes

# Chapter Goals

This chapter will introduce a powerful form of abstraction enabled by PureScript's type system - type classes. 

This motivating example for this chapter will be a library for hashing data structures. We will see how the machinery of type classes allows
us to hash complex data structures without having to think directly about the structure of the data itself. 

We will also see a collection of standard type classes from PureScript's Prelude and standard libraries. PureScript code leans heavily on the power of type classes to express ideas concisely, so it will be beneficial to familiarize yourself with these classes. 

If you come from an object oriented background, please note that the word "class" means something very different in this context than what
you're used to. A type class serves a purpose more similar to an OO interface.

# Project setup

The source code for this chapter is in the file src/Data/Hashable.purs

The project has the following dependencies:

* maybe, which defines the Maybe data type, which represents optional values 
* tuples, which defines the Tuple data type, which represents pairs of values 
* either, which defines the Either data type, which represents disjoint unions 
* strings, which defines functions that operate on strings
* functions, which defines some helper functions for defining PureScript functions 

The module Data.Hashable imports several modules provided by these packages 

# Show me!

Our first example of a type class is provided by a function we've seen several times already: the show function, which takes a value and displays it as a string. show is defined by a type class in the Prelude module called Show, which is defined as follows:

        class Show a where 
          show :: a -> String

This code declares a new type class called Show, which is parameterized by the type variable a.

A type class instance contains implementations of the functions defined in a type class, specialized to a particular type. 

For example, here is the definition of the Show type class instance for Boolean values, taken from the Prelude:

        instance Show Boolean where 
          show true = "true"
          show false = "false"

This code declares a type class instance, we say that the Boolean type belongs to the Show type class.

If you're unhappy with the generated name (the generated JS code calls it showBoolean), you can give names to type class instances. For example:

        instance myShowBoolean :: Show Boolean where 
          show true = "true"
          show false = "false"

We can try out the Show type class in PSCi by showing a few values with different types:

        > import Prelude

        > show true
        "true"

        > show 1.0
        "1.0"

        > show "Hello World"
        "\"Hello World\""

These examples demonstrate how to show values of various primitive types, but we can also show values with more complicated types:

        > import Data.Tuple

        > show (Tuple 1 true)
        "(Tuple 1 true)"

        > import Data.Maybe

        > show (Just "testing")
        "(Just \"testing\")"

The output of show should be a string that you can paste back into the repl (or .purs file) to recrete the item being shown. Here we'll use logShow, which just calls show and then log, to render the string without quotes. Ignore the unit print - that will be covered in Chapter 8 when we examine Effect s, like log.

        > import Effect.Console

        > logShow (Tuple 1 true)
        (Tuple 1 true)
        unit

        > logShow (Just "testing")
        (Just "testing")
        unit

If we try to show a value of type Data.Either, we get an interesting error message:

        > import Data.Either
        > show (Left 10)

        The inferred type

            forall a. Show a => String

        has type variables which are not mentioned in the body of the type. Consider adding a type annotation.

The problem here is not that there is no Show instance for the type we intended to show, but rather that PSCi could not infer the type. This is indicated by the unknown type a in the inferred type. We can annotate the expression with a type using the :: operator, so that PSCi can choose the correct type class instance: 

        > show (Left 10 :: Either Int String)
        "(Left 10)"

some types do not have a Show instance defined at all. one example of this is the function type ->. if we try to show a function from Int to Int, we get an appropriate error message from the type checker:

        > import Prelude
        > show $ \n -> n + 1

        No type class instance was found for

        Data.Show.Show (Int -> Int)

Type class instances can be defined in one of two places: in the same module that the type class is defined, or in the same module that the type "belonging to" the type class is defined. an instance defined in any other spot is called an "orphan instance" and is not allowed by the PureScript compiler. Some of the exercises in this chapter will require you to copy the definition of a type into your MySolutions module so that you can define type class instances for that type. 

# Common Type Classes

In this section, we'll look at some standard type classes defined in the Prelude and standard libraries. These type classes
form the basis of many common patterns of abstraction in idiomatic PureScript code, so a basic understanding of their 
functions is highly recommended. 

# Eq

The Eq type class defines the eq function, which tests two values for equality. The == operator is actually an alias for 
eq. 

class Eq a where
  eq :: a -> a -> Boolean 

In either case, the two arguments must have the same type: it does not make sense to compare two values of different types
for equality. 
Try out the Eq type class in PSCi:

        > 1 == 2
        false 
        > "Test" == "Test"
        true 

# Ord

The Ord type class defines the compare function, which can be used to compare two values, for types that support ordering. 
The comparison operators < and > along with their non-strict companions <= and >= , can be defined in terms of compare.

Note: in the example below, the class signature contains <=. This usage of <= in this context indicates that Eq is a 
superclass of Ord and is not intended to represent the use of <= as a comparison operator. See tht section Superclasses 
below. 

        data Ordering = LT | EQ | GT
        class Eq a <= Ord a where 
          compare :: a -> a -> Ordering 

The compare function compares two values and returns an Ordering, which has three alternatives:

* LT - if the first argument is less than the second 
* EQ - if the first argument is equal to the second
* GT - if the first argument is greater than the second 

Again, we can try out the compare function in PSCi:

        > compare 1 2
        LT
        > compare "A" "Z"
        LT

# Field 

The Field type class identifies those types which support numeric operators such as addition, subtraction, multiplication,
and division. It is provided to abstract over those operators, so that they can be reused where appropriate. 

Note: Just like the Eq and Ord type classes, the Field type class has special support in the PureScript compiler, so that 
simple expressions such as 1 + 2 * 3 get translated into simple JavaScript, as opposed to function caalls which dispatch 
based on a type class implementation 

        class EuclideanRing a <= Field a 

The field type class is composed from several more general superclasses. This allows us to talk abstractly about types that 
support some but not all of the Field operations. For example, a type of natural numbers would be closed under addition 
and multiplication, but not necessarily under subtraction, so that type might have an instance of the Semiring class 
(which is. superclass of Num), but not an instance of Ring or Field. 

Superclasses will be explained later in this chapter, but the full numeric type class hierarchy (cheatsheet) is beyond the 
scope of this chapter. The interested reader is encouraged to read the documentation for the superclasses of Field in 
prelude. 

# Semigroups and Monoids 

The Semigroup type class identifies those types which support an append opertion to combine two values:

        class Semigroup a where 
          append :: a -> a -> a

Strings form a semigroup under regular string concatenation, and so do arrays. The prelude package provides several other standard instances. 

The <> concatenation operator, which we have already seen, is provided as an alias for append.

The Monoid type class (provided by the prelude package) extends the Semigroup type class with the concept of an empty value, called 
mempty:

        class Semigroup m <= Monoid m where 
          mempty :: m 

Again, strings and arrays are simple examples of monoids. 

A Monoid type class instance for a type describes how to accumulate a result with that type by starting with an "empty" value and 
combining new results. For example, we can write a function that concatenates an array of values in some monoid using a fold. In PSCi:

        > import Prelude
        > import Data.Monoid
        > import Data.Foldable 

        > foldl append mempty ["Hello", " ", "world"]
        "Hello world"
        > foldl append mempty [[1, 2, 3], [4, 5], [6]]
        [1,2,3,4,5,6]

The prelude package provides many examples of monoids and semigroups, which we will use in the rest of the book

# Foldable 

If the Monoid type class identifies those types which act as the result of a fold, then the Foldable type class identifies those type 
constructors which can be used as the source of a fold. 

The Foldable type class is provided in the foldable-traversable package, which also contains instances for some standard containers such 
as arrays and Maybe. 

The type signatures for the functions belonging to the Foldable class are a little more complicated than the ones we've seen so far:

        class Foldable f where
          foldr :: forall a b. (a -> b -> b) -> b -> f a -> b
          foldl :: forall a b. (b -> a -> b) -> b -> f a -> b
          foldMap :: forall a m. Monoid m => (a -> m) -> f a -> m

It is instructive to specialize to the case where f is the array type constructor. In this case, we can replace f a with Array a for 
any a, and we notice that the type of foldl and foldr become the types we saw when we first encountered folds over arrays.

What about foldMap? well, that becomes forall a m. Monoid m => (a -> m) -> Array a -> m. 
This type signature says that we can choose any type m for our result type, as long as that type is an instance of the Monoid type 
class. If we can provide a function that turns our array elements into values in that monoid, then we can accumulate over our 
array using the structure of the monoid and return a single value. 

Let's try out foldMap in PSCi:

        > import Data.Foldable 
        > foldMap show [1, 2, 3, 4, 5]
        "12345"

Here, we choose the monoid for strings, which concatenates strings together, and the show function, which renders an Int as a String. 
Then, passing in an array of integers, we see that the result of showing each integer have been concatenated into a single String. 

But arrays are not the only types that are foldable. foldable-traversable also defines Foldable instances for types like Maybe and 
Tuple, and other libraries like lists define Foldable instances for their own data types. Foldable captures the notion of an ordered
container. 

# Functor and Type Class Laws 

The Prelude also defines a collection of type classes that enable a functional style of programming with side-effects in PureScript:
Functor, Applicative, and Monad. We will cover these abstractions later in the book, but for now, let's look at the definition of
the Functor type class, which we have seen already in the form of the map function: 

        class Functor f where 
          map :: forall a b. (a -> b) -> f a -> f b

The map function (and its alias <$>) allows a function to be "lifted" over a data structure. The precise definition of the word 
"lifted" here depends on the data structure in question, but we have already seen its behaviour for some simple types: 

        > import Prelude 
        
        > map (\n -> n < 3) [1, 2, 3, 4, 5]
        [true, true, false, false, false]

        > import Data.Maybe 
        > import Data.String (length)

        > map length (Just "testing")
        (Just 7)

How can we understand the meaning of the map function, when it acts on many different structures, each in a different way?

Well, we can build an intuition that the map function applies the function it is given to each element of a container, and builds a new container from 
the results, with the same shape as the original. But how do we make this concept precise? 

type class instances for Functor are expected to adhere to a set of laws, called the functor laws: 

* map identity xs = xs
* map g (map f xs) = map (g <<< f) xs

The first law is the identity law. It states that lifting the identity function (the function which returns its arguments unchanged) over
a structure just returns the original structure. This makes sense since the identity function does not modify its input.

The second law is the composition law. It states that mapping one function over a structure and then mapping a second is the same 
as mapping the composition of the two functions over the structure. 

Whatever "lifting" means in the general sense, it should be true that any reasonable definition of lifting a function over a data 
structure should obey these rules. 

Many standard type classes come with their own set of similar laws. The laws given to a type class give structure to the functions 
of that type class and allow us to study its instances in generality. The interested reader can research the laws ascribed to the standard
type classes that we have seen already.

# Deriving Instances 

Rather than writing instances manually, you can let the compiler do most of the work for you. Take a look at this Type Class Deriving guide. 
That information will help you solve the following exercises. 

# Type Class Deriving 

The compiler can derive type class instances to spare you the tedium of writing boilerplate. There are a few ways to do this depending on the specific type and 
class being derived. 

# Classes with built-in compiler support

Some classes have special built-in compiler support, and their instances can be derived from all types. For example, if you'd like to be able to remove duplicates
from an array of an ADT using nub, you need an Eq and Ord instance. Rather than writing these manually, let the compiler do the work:

        import Data.Array (nub)

        data MyADT
        = Some
        | Arbitrary Int
        | Contents Number String

        derive instance eqMyADT :: Eq MyADT
        derive instance ordMyADT :: Ord MyADT

        nub [Some, Arbitrary 1, Some, Some] == [Some, Arbitrary 1]

# Derive from newtype 

If you would like your newtype to defer to the instance that the underlying type uses for a given class, then you can use newtype deriving via the derive 
newtype keywords. 

For example, let's say you want to add two Score values using the Semiring instance of the wrapped Int. 

        newtype Score = Score Int

        derive newtype instance semiringScore :: Semiring Score 

        tenPoints :: Score
        tenPoints = (Score 4) + (Score 6)

That derive line replaced all this code:

        -- No need to write this
        instance semiringScore :: Semiring Score where
        zero = Score 0
        add (Score a) (Score b) = Score (a + b)
        mul (Score a) (Score b) = Score (a * b)
        one = Score 1

Note that we can use either of these options to derive an Eq instance for a newtype, since Eq has built-in compiler support. They are equivalent in this case.

        derive instance eqScore :: Eq Score
        derive newtype instance eqScore :: Eq Score

# Deriving from Generic 

The compiler's built-in support for Generic unlocks convenient deriving for many other classes not listed above. 

For example, if we wanted to derive a Show instance for MyADT it might seem like we're out of luck: Show is not a class with built-in compiler support for deriving
and MyADT is not a newtype (so we can't use newtype deriving).

But we can use genericShow, which works with any type that has a Generic instance. And recall that the compiler has built-in support for deriving a Generic 
instance for any type (including the MyADT type). we put all those pieces together like so:

        import Data.Generic.Rep (class Generic)
        import Data.Show.Generic (genericShow)
        import Effect.Console (logShow)

        derive instance genericMyADT :: Generic MyADT _ 

        instance showMyADT :: Show MyADT where 
          show = genericShow 
        
        main = logShow [Some, Arbitrary 1, Contents 2.0 "Three"]
        -- prints:
        -- [Some,(Arbitrary 1),(Contents 2.0 "Three")]

The show type class is most often used for debugging data, so the output of most Show instance can be copy-pasted back into a PureScript source file to reconstruct the original data. The Show instance we created by deriving Generic and then using genericShow follows this convention. 

This is a good opportunity to emphasize how newtype deriving is different from instances derived by the compiler or through the Generic type class. In the examples below, notice how the instance derived through Generic includes the newtype constructor Score, but the newtype-derived instance simply reuses the underlying Show instance for Int and therefore does not include the constructor:

        import Effect.Console (logShow)

        newtype Score = Score Int 

        -- newtype deriving omits wrapper with show 
        derive newtype instance showScore :: Show Score 

        main = logShow (Score 5)
        -- Prints:
        -- 5

        import Data.Generic.Rep (class Generic)
        import Data.Show.Generic (genericShow)
        import Effect.Console (logShow)

        newtype Score = Score Int

        --generic deriving prints wrapper with show 
        derive instance genericScore :: Generic Score _
        instance showScore :: Show Score where 
          show = genericShow
        
        main = logShow (Score 5)
        -- Prints:
        -- (Score 5)


# Type Class Constraints 

Types of functions can be constrained by using type classes. Here is an example: suppose we want to write a function that tests if three values are equal,
by using an equality defined using an Eq type class instance. 

        threeAreEqual :: forall a. Eq a => a -> a -> a -> Boolean 
        threeAreEqual a1 a2 a3 = a1 == a2 && a2 == a3

The type declaration looks like an ordinary polymorphic type defined using forall. However, there is a type class constraint Eq a, separated from the rest 
of the type by a double arrow => . 

This type says that we can call threeAreEqual with any choice of type a, as long as there is an Eq instance available for a in one of the imported modules. 

Constrained types can contain several type class instances, and the types of the instances are not restricted to simple type variables. Here is another 
example which uses Ord and Show instances to compare two values:

        showCompare :: forall a . Ord a => Show a => a -> a -> String 
        showCompare a1 a2 | a1 < a2 =
          show a1 <> " is less than " <> show a2
        showCompare a1 a2 | a1 > a2 = 
          show a1 <> " is greater than " <> show a2
        showCompare a1 a2 = 
          show a1 <> " is equal to " <> show a2 

Note that multiple constraints can be specified by using the => symbol multiple times, just like we specify curried functions of multiple arguments. But remember
not to confuse the two symbols:

* a -> b denotes the type of functions from type a to type b, whereas 
* a => b applies the constraint a to the type b 

The PureScript compiler will try to infer constrained types when a type annotation is not provided. This can be useful if we want to use the most general type 
possible for a function. 

To see this, try using one of the standard type classes like Semiring in PSCi: 

        > import Prelude 
        > :type \x -> x + x
        forall (a :: Type). Semiring a => a -> a

Here, we might have annotated this function as Int -> Int or Number -> Number, but PSCi shows us that the most general type works for any Semiring, allowing us 
to use our function with both Ints and Number.

# Instance Dependencies 

Just as the implementation of functions can depend on type class instances using constrained types, so can the implementation of type class instances depend on
other type class instances. This provides a powerful form of program inference, in which the implementation of a program can be inferred using its types. 

For example, consider the Show type class. We can write a type class instance to show arrays of elements, as long as we have a way to show the elements themselves:

        instance Show a => Show (Array a) where 
         ... 

If a type class instance depends on multiple other instances, those instances should be grouped in parentheses and separated by commas on the left-hand side of 
the => symbol:

instance (Show a, Show b) => Show (Either a b) where
  ...

These two type class instances are provided in the prelude library.

When the program is compiled, the correct type class instance for Show is chosen based on the inferred type of the argument to show. The selected instance might 
depend on many such instance relationships, but this complexity is not exposed to the developer. 

# Multi-Parameter Type Classes 

It's not the case that a type class can only take a single type as an argument. This is the most common case, but a type class can be parameterized by zero or 
more type arguments.

Let's see an example of a type class with two type arguments:

        module Stream where 

        import Data.Array as Array
        import Data.Maybe as (Maybe)
        import Data.String.CodeUnits as String 

        class Stream stream element where 
          uncons :: stream -> Maybe { head :: element, tail :: stream }
        
        instance Stream (Array a) a where 
          uncons = Array.uncons 

        instance Stream String Char where 
          uncons = String.uncons 
        
The Stream module defines a class Stream which identifies types that look like streams of elements, where elements can be pulled from the front of the stream 
using the uncons function. Note that the Stream type class is parameterized not only by the type of the stream itself, but also by its elements. This allows us to
define type class instances for the same stream type but different element types. 

The module defines two type class instances: an instance for the arrays, where uncons removes the head element of the array using pattern matching, and an instance for String, which removes the first character from the string. 

We can write functions that work over arbitrary streams. For example, here is a function that accumulates a result in some Monoid based on the elements of a stream: 
        import Prelude 
        import Data.Monoid (class Monoid, mempty)

        foldStream :: forall l e m. Stream l e => Monoid m => (e -> m) -> l -> m
        foldStream f list = 
          case uncons list of 
            Nothing -> mempty
            Just cons -> f cons.head <> foldStream f cons.tail 
        
Try using foldStream in PSCi for different types of Stream and different types of Monoid.

# Functional Dependencies 

Multi-parameter type classes can be very useful but can easily lead to confusing types and even issues with type inference. As a simple example, consider writing a generic tail function on streams using the Stream class given above: 

        genericTail xs = map _.tail (uncons xs)

This gives a somewhat confusing error message:

        The inferred type 
          forall stream a. Stream stream a => stream -> Maybe stream 
        has type variables which are not mentioned in the body of the type. Consider adding a type annotation 

The problem is that the genericTail function does not use the element type mentioned in the definition of the Stream type class, so that type is left unsolved. 
Worse still, we cannot even use genericTail by applying it to a specific type of stream: 

        > map _.tail (uncons "testing")

        The inferred type 
          forall a. Stream String a => Maybe String
        has type variables which are not mentioned in the body of the type. Consider adding a type annotation.

Here we might expect the compiler to choose the streamString instance. After all, a string is a stream of Chars, and cannot be a stream of any other type of elements. 

The compiler cannot make that deduction automatically or commit to the streamString instance. However, we can help the compiler by adding a hint to the type class definiton: 

        class Stream stream element | stream -> element where 
          uncons :: stream -> Maybe { head :: element, tail :: stream }

Here, stream -> element is called a functional dependency. A functional dependency asserts a functional relationship between the type arguments of a multi-parameter type class. This functional dependency tells the compiler that there is a function from stream types to (unique) element types, so if the compiler knows the stream type, then it can commit to the element type. 

This hint is enough for the compiler to infer the correct type for our generic tail function above: 

        > :type genericTail 
        forall (stream :: Type) (element :: Type). Stream stream element => stream -> Maybe stream 
        > genericTail "testing"
        (Just "esting")

Functional dependencies can be useful when designing certain APIs using multi-parameter type classes. 

# Nullary Type Classes 

We can even define type classes with zero-type arguments! These correspond to compile-time assertions about our functions, allowing us to track the global properties of our code in the type system. An important example is the Partial class we saw earlier when discussing partial functions. Take for example, the functions head and tail defined in Data.Array.Partial that allow us to get the head or tail of an array without wrapping them in a Maybe, so they can fail if the array is empty:

        head :: forall a. Partial => Array a -> a
        tail :: forall a. Partial => Array a -> Array a 

Note that there is no instance defined for the Partial type class! Doing so would defeat its purpose: attempting to use the head function directly will result in a type error:

        > head [1, 2, 3]

        No type class instance was found for 

          Prim.Partial 

Instead, we can republish the Partial constraint for any functions making use of partial functions:

        secondElement :: forall a. Partial => Array a -> a
        secondElement xs = head (tail xs)

We've already seen the unsafePartial function, which allows us to treat a partial function as a regular function (unsafely). This function is defined in the Partial.Unsafe module.:

        unsafePartial :: forall a. (Partial => a) -> a

Note that the Partial constraint appears inside the parentheses on the left of the function arrow, but not in the outer forall. That is, unsafePartial is a function from partial values to regular values: 

        > unsafePartial head [1, 2, 3]
        1

        > unsafePartial secondElement [1, 2, 3]
        2

# Superclasses 

Just as we can express relationships between type class instances by making an instance dependent on another instance, we can express relationships between type classes themselves using so-called superclasses. 

We say that one type class is a superclass of another if every instance of the second class is required to be an instance of the first, and we indicate a superclass relationship in the class definition by using a backwards facing double arrow (<=).

We've already seen an example of superclass relationships: The Eq class is a superclass of Ord, and the Semigroup class is a superclass of Monoid. For every type class instance of the Ord class, there must be a corresponding Eq instance for the same type. This makes sense since, in many cases, when the compare function reports that two values are incomparable, we often want to use the Eq class to determine if they are equal. 

In general, it makes sense to define a superclass relationship when the laws for the subclass mention the superclass members. For example, for any pair of Ord and Eq instances, it is reasonable to assume that if two values are equal under the Eq instance, then the compare function should return EQ. In other words, a == b should be true exactly when compare a b evaluates to EQ. This relationship on the level of laws justifies the superclass relationship between Eq and Ord. 

Another reason to define a superclass relationship is when there is a clear "is-a" relationship between the two classes. That is, every member of the subclass is a member of the superclass as well. 

# A Type Class for Hashes 

In the last section of this chapter, we will use the lessons from the rest of the chapter to create a library for hashing data structures. Note that this library is for demonstration purposes only and is not intended to provide a robust hashing mechanism.

What properties might we expect of a hash function?

* a hash function should be deterministic and map equal values to equal hash codes 
* a hash function should distribute its results approximately uniformly over some set of hash codes 

The first property looks a lot like a law for a type class, whereas the second property is more along the lines of an informal contract and certainly would not be enforceable by PureScript's type system. However, this should provide the intuition for the following type class:

        newtype HashCode = HashCode Int

        instance Eq HashCode where 
          eq (HashCode a) (HashCode b) = a == b 

        hashcode :: Int -> HashCode 
        hashcode h = HashCode (h `mod` 65535)

        class Eq a <= Hashable a where 
          hash :: a -> HashCode 
        
with the associated law that a == b implies hash a == hash b.

We;ll spend the rest of this section building a library of instances and functions associated with the Hashable type class. We will need a way to combine hash codes in a deterministic way:

        combineHashes :: HashCode -> HashCode -> HashCode
        combineHashes (HashCode h1)(HashCode h2) = hashCode (73 * h1 + 51 * h2)

The combineHashes function will mix two hashcodes and redistribute the result over the interval 0 - 65535.

Let's write a function that uses the Hashable constraint to restrict the types of its inputs. One common task which requires a hashing function is to determine if two values hash to the same hash code. The hashEqual function provides such a capability: 

        hashEqual :: forall a. Hashable a => a -> a -> Boolean 
        hashEqual = eq `on` hash 

This function uses the on function from Data.Function to define hash-equality in terms of equality of hash codes, and should read like a declarative definition of hash-equality: two values are "hash-equal" if they are equal after each value passed through the hash function. 

Let's write some Hashable instances for some primitive types. Let's start with an instance of integers. Since a HashCode is really just a wrapped integer, this is simple - we can use the hashCode helper function:

        instance Hashable Int where 
          hash = hashCode 

We can also define a simple instance for Boolean values using pattern matching:

        instance Hashable Boolean where
          hash false = hashCode 0
          hash true = hashCode 1

With an instance for hashing integers, we can create an instance for hashing Char's by using the toCharCode function from Data.Char: 

        instance Hashable Char where 
          hash = hash <<< toCharCode 

To define an instance for arrays, we can map the hash function over the elements of the array (if the element type is also an instance of Hashable) and then perform a left fold over the resulting hashes using the combineHashes function:

        instance Hashable a => Hashable (Array a) where 
          hash = foldl combineHashes (hashCode 0) <<< map hash 

Notice how we build up instances using the simpler instances we have already written. Let's use our new Array instance to define an instance for String(s), by turning a String into an array of Char(s):

        instance Hashable String where 
          hash = hash <<< toCharArray 

How can we prove that these Hashable instances satisfy the type class law that we stated above? We need to make sure that equal values have equal hash codes. In case like Int, Char, String, and Boolean, this is simple because there are no values of those types that are equal in the sense of Eq but not equal identically. 

What about some more interesting types? To prove the type class law for the Array instance, we can use induction on the length of the array. The only array with a length zero is []. Any two non-empty arrays are equal only if they have equal head elements and equal tails, by the definition of Eq on arrays. By the inductive hypothesis, the tails have equal hashes, and we know that the head elements have equal hashes if the Hashable a instance must satisfy the law. Therefore, the two arrays have equal hashes, and so the Hashable (Array a) obeys the type class law as well. 

The source code for this chapter includes several other examples of Hashable instances, such as instances for the Maybe and Tuple types. 

# Conclusion 

In this chapter, we've been introduced to type classes, a type-oriented form of abstraction that enables powerful forms of code reuse.