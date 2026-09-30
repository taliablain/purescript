# Applicative Validation 

# Chapter Goals 

In this chapter, we will meet an important new abstraction - the applicative functor, described by the Applicative type class. Don't worry if the name sounds confusing - we will motivate the concept with a practical example - validating form data. This technique allows us to convert code which usually involves a lot of boilerplate checking into a simple, declarative description of our form. 

We will also meet another type class, Traversable, which describes traversable functors, and see how this concept also arises very naturally from solutions to real-world problems. 

The example code for this chapter will be a continutation of the address book example from Chapter 3. This time, we will extend our address book data types and write functions to validate values for those types. The understanding is that these functions could be used, for example, in a web user interface, to display errors to the user as part of a data entry form. 

# Project Setup 

The source code for this chapter is defined in the files src/Data/AddressBook.purs and src/Data/AddressBook/validation.purs . 

The project has a number of dependencies, many of which we have seen before. There are two new dependencies: 

* control, which defines functions for abstracting control flow using type classes like Applicative 
* validation, which defines a functor for applicative validation, the subject of this chapter 

The Data.AddressBook module defines data types and Show instances for the types in our project and the Data.AddressBook.Validation module contains validation rules for those types. 

# Generealizing Function Application 

To explain the concept of an applicative functor, let's consider the type constructor Maybe that we met earlier. The source code for this module defines a function address that has the following type: 

    address :: String -> String -> String -> Address 

This function is used to construct a value of type Address from three strings: a street name, a city, and a state. 

We can apply this function easily and see the result in PSCi: 

    > import Data.AddressBook 

    > address "123 Fake St." "Faketown" "CA"
    { street: "123 Fake St.", city: "Faketown", state: "CA" }

However, suppose we did not necessarily have a street, city, or state, and wanted to use the Maybe type to indicate a missing value in each of the three cases.

In one case, we might have a missing city. If we try to apply our function directly, we will receive an error from the type checker:

    > import Data.Maybe 
    > address (Just "123 Fake St.") Nothing (Just "CA")

    Could not match type 
      Maybe String 
    with type
      String 

Of course, this is an expected type error - address takes strings as arguments, not values of type Maybe String. 

However, it is reasonale to expect that we should be able to "lift" the address function to work with optional values described by the Maybe type. In fact, we can, and the Control.Apply provides the function lift3 function which does exactly what we need:

    > import Control.Apply
    > lift3 address (Just "123 Fake St.") Nothing (Just "CA")

    Nothing 

In this case, the result is Nothing, because one of the arguments (the city) was missing. If we provide all three arguments using the Just constructor, then the result will contain a value as well:

    > lift3 address (Just "123 Fake St.") (Just "Faketown") (Just "CA")

    Just ({ street: "123 Fake St.", city: "Faketown", state: "CA" })

The name of the function lift3 indicates that it can be used to lift functions of 3 arguments. There are similar functions defined in Control.Apply for functions of other numbers of arguments.

# Lifting Arbitrary Functions 

So we can lift functions with small numbers of arguments by using lift2, lift3, etc. But how can we generalize this to arbitrary functions? 

It is instructive to look at the type of lift3: 

    > :type lift3
    forall (a :: Type) (b :: Type) (c :: Type) (d :: Type) (f :: Type -> Type). Apply f => (a -> b -> c -> d) -> f a -> f b -> f c -> f d

In the Maybe example above, the type constructor f is Maybe, so that lift3 is specialized into the following type:

    forall a b c d. (a -> b -> c -> d) -> Maybe a -> Maybe b -> Maybe c -> Maybe d

This type says that we can take any function with three arguments and lift it to give a new function whose argument and result types are wrapped with Maybe. 

Certainly, this is not possible for every type constructor f, so what is it about the Maybe type which allowed us to do this? well, in specializing the type above, we removed a type class constraint on f from the Apply type class. Apply is defined in the Prelude as follows: 

    class Functor f where
      map :: forall a b. (a -> b) -> f a -> f b

    class Functor f <= Apply f where
      apply :: forall a b. f (a -> b) -> f a -> f b

The Apply type class is a subclass of Functor, and defines an additional function apply. As <$> was defined as an alias for map, the Prelude module defines <*> as an alias for apply. As we'll see, these two operators are often used together. 

Note that this apply is different than the apply from Data.Function (infixed as $). Luckily, infix notation is almost always used for the latter, so you don't need to worry about name collisions. 

The type of apply looks a lot like the type of map. The difference between map and apply is that map takes a function as an argument, whereas the first argument to apply is wrapped in the type constrcutor f. We'll see how this is used soon, but first, let's see how to implement the Apply type class for the Maybe type:

    instance Functor Maybe where
      map f (Just a) = Just (f a)
      map f Nothing  = Nothing

    instance Apply Maybe where
      apply (Just f) (Just x) = Just (f x)
      apply _        _        = Nothing

This type class instance says that we can apply an optional function to an optional value, and the result is defined only if both are defined. 

Now we'll see hoe map and apply can be used together to lift functions of an arbitrary number of arguments. 

For functions of one argument, we can use map directly. 

For functions of two arguments, we have a curried function g with type a -> b -> c, say. This is equivalent to the type a -> (b -> c), so we can apply map to g to get a new function of type f a -> f (b -> c) for any type constructor f with a Functor instance. Partially applying this function to the first lifted argument (of type fa), we get a new wrapped function of type f (b -> c). If we also have an Apply instance for f, we can then use applu to apply the second lifted argument (of type f b) to get our final value of type f c. 

Putting all this together, we see that if we have values x :: f a and y :: f b, then the expression (g <$> x) <*> y has type f c (remember, this expression is equivalent to apply (map g x) y). The precedence rules defined in Prelude allow us to remove the parentheses: g <$> x <*> y. 

In general, we can use <$> on the first argument, and <*> for the remaining arguments, as illustrated here for lift3:

    lift3 :: forall a b c d f
           . Apply f
         => (a -> b -> c -> d)
         -> f a
         -> f b
         -> f c
         -> f d
    lift3 f x y z = f <$> x <*> y <*> z

As an example, we can try lifting the address function over Maybe, directly using the <$> and <*> functions:

    > address <$> Just "123 Fake St." <*> Just "Faketown" <*> Just "CA"
    Just ({ street: "123 Fake St.", city: "Faketown", state: "CA" })

    > address <$> Just "123 Fake St." <*> Nothing <*> Just "CA"
    Nothing

Try lifting some other functions of various numbers of arguments over Maybe in this way.

Alternatively, applicative do notation can be used for the same purpose in a way that looks similar to the familiar do notation. Here is lift3 using applicative do notation. Note ado is used instead of do, and in is used on the final line to denote the yielded value:

    lift3 :: forall a b c d f
       . Apply f
      => (a -> b -> c -> d)
      -> f a
      -> f b
      -> f c
      -> f d
    
    lift3 f x y z = ado
      a <- x
      b <- y
      c <- z
      in f a b c

#  The Applicative Type Class

There is a related type class called Applicative, defined as follows:

    class Apply f <= Applicative f where 
      pure :: forall a. a -> f a

Applicative is a subclass of Apply and defines the pure function. pure takes a value and returns a value whose type has been wrapped with the type constructor f. 

Here is the Applicative instance for Maybe:

    instance Applicative Maybe where
      pure x = Just x

If we think of applicative functors as functors that allow lifting of functions, then pure can be thought of as lifting functions of zero arguments. 

# Intuition for Applicative

Functions in purescript ar epure and do not support side-effects. Applicative functors allow us to work in larger "programming languages" which support some sort of side-effect encoded by the functor f.

As an example, the functor Maybe represents the side effect of possibly-missing values. Some other examples include Either err, which represents the side effect of possible errors of type err, and the arrow functor r ->, which represents the side-effect of reading from a global configuration. For now, we'll only consider the Maybe functor.

If the functor f represents this larger programming language with effects, then the Apply and Applicative instances allow us to lift values and function applications from our smaller programming language (Purescript) into the new language. 

pure lifts pure (side-effect free) values into the larger language; for functions, we can use map and apply as described above. 

This raises a question: if we can use Applicative to embed purescript functions and values into this new language, then how is the new language any larger? the answer depends on the functor f. if we can find expressions of type f a which cannot be expressed as pure x for some x, then that expression represents a term which only exists in the larger language. 

When f is Maybe, an example is the expression Nothing: we cannot write Nothing as pure x for any x. Therefore, we can think of Purescript as having been enlarged to include the new term Nothing, which represents a missing value.

# More Effects

Let's see some more examples of lifting functions over different Applicative functors. 

Here is a simple example function defined in PSCi, which joins three names to form a full name:

    > import Prelude
    > fullName first middle last = last <>  ", " <> first <> " " <> middle 
    > fullName "Philip" "A" "Freeman"
    Freeman, Phillip A

Suppose that this function forms the implementation of a (very simple!) web service with the three arguments provided as query parameters. We want to ensure that the user provided each of the three parameters, so we might use the Maybe type to indicate the presence or absence of a parameter. We can lift fullName over Maybe to create an implementation of the web service which checks for missing parameters:

    > import Data.Maybe

    > fullName <$> Just "Phillip" <*> Just "A" <*> Just "Freeman"
    Just ("Freeman, Phillip A")

    > fullName <$> Just "Phillip" <*> Nothing <*> Just "Freeman"

Or with applicative do:

    > import Data.Maybe

    > :paste…
    … ado
    …   f <- Just "Phillip"
    …   m <- Just "A"
    …   l <- Just "Freeman"
    …   in fullName f m l
    … ^D
    (Just "Freeman, Phillip A")

    … ado
    …   f <- Just "Phillip"
    …   m <- Nothing
    …   l <- Just "Freeman"
    …   in fullName f m l
    … ^D
    Nothing

Note that the lifted function returns Nothing if any of the arguments was Nothing.

This is good because now we can send an error response back from our web service if the parameters are invalid. However, it would be better if we could indicate which field was incorrect in the response. 

Instead of lifting over Maybe, we can lift over Either String, which allows us to return an error message. First, let's write an operator to convert optional inputs into computations which can signal an error using Either String: 

    > import Data.Either
    > :paste
    ... withError Nothing  err = Left err
    ... withError (Just a) _   = Right a
    ... ^D

Note: in the Either err applicative functor, the Left constructor indicates an error, and the Right constructor indicates success. 

Now we can lift over Either String, providing an an appropriate error message for each parameter:

    > :paste
    ... fullNameEither first middle last =
    ...   fullName <$> (first  `withError` "First name was missing")
    ...            <*> (middle `withError` "Middle name was missing")
    ...            <*> (last   `withError` "Last name was missing")
    ... ^D

    > :paste
    … fullNameEither first middle last = ado
    …  f <- first  `withError` "First name was missing"
    …  m <- middle `withError` "Middle name was missing"
    …  l <- last   `withError` "Last name was missing"
    …  in fullName f m l
    … ^D

    > :type fullNameEither
    Maybe String -> Maybe String -> Maybe String -> Either String String

Now our function takes three optional arguments using Maybe, and returns either a String error message or a string result. 

We can try out the function with different inputs:

    > fullNameEither (Just "Phillip")(Just "A")(Just "Freeman")
    (Right "Freeman, Phillip A")

    > fullNameEither (Just "Phillip") Nothing (Just "Freeman")
    (Ledt "Middle name was missing")

    > fullNameEither (Just "Phillip") (Just "A") Nothing 
    (Left "Last name was missing")

In this case, we see the error message corresponding to the first missing field or a successful result if every field was provided. However, if we are missing multiple inputs, we still see only the first error:

    > fullNameEither Nothing Nothing Nothing 
    (Left "First name was missing")

This might be good enough, but if we want to see a list of all missing fields in the error, then we need something more powerful than Either String. We will see a solution later in this chapter. 

# Combining Effects 

As an example of working with applicative functors abstractly, this section will show how to write a function that generically combines side-effects encoded by an applicative functor f. 

What does this mean? Well, suppose we have a list of wrapped arguments of type f a for some a. That is, suppose we have a list of type List (f a). Intuitively, this represents a list of computations with side-effects tracked by f, each with return type a. If we could run all of these computations in order, we would obtain a list of results of type List a. However, we would still have side-effects tracked by f. That is, we expect to be able to turn something of type List (f a) into something of type f (List a) by "combining" the effects inside the original list. 

For any fixed list size n, there is a function of n arguments that builds a list of size n out of those arguments. For example, if n is 3, the function is \x y z -> x : y : z : Nil. This function has type a -> a -> a -> List a. We can use the Applicative instance for List to lift this function over f, to get a function of type f a -> f a -> f a -> f (List a). But, since we can do this for any n, it makes sense that we should be able to perform the same listing for any list of arguments. 

That means that we should be able to write a function 

    combineList :: forall f a. Applicative f  => List (f a) -> f (List a)

This function will take a list of arguments, which possibly have side-effects, and return a single wrapped list, applying the side-effects of each. 

To write this function, we'll consider the length of the list of arguments. If the list is empty, then we do not need to perform any effects, and we can use pure to simply return an empty list:

    combineList Nil = pure Nil

In fact, this is the only thing we can do!

If the list is non-empty, then we have a head element, which is a wrapped argument of type f a, and a tail of type List (f a). We can recursively combine the effects in the tail, giving a result of type f (List a). We can then use <$> and <*> to lidt the Cons constructor over the head and new tail:

    combineList (Cons x xs) =

