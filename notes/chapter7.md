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

For functions of two arguments, we have a curried function g with type a -> b -> c, say. This is equivalent to the type a -> (b -> c), so we can apply map to g to get a new function of type f a -> f (b -> c) for any type constructor f with a Functor instance. Partially applying this function to the first lifted argument (of type fa), we get a new wrapped function of type f (b -> c)

