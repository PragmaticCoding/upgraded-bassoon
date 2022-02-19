## Optional Isn’t About Avoiding Null

# The Introduction of Optional

When `Optional` came out there was a lot of noise about how it was supposed to replace `Null` and help do away with `NullPointerExceptions`.  So people started using `Optional` instead of `Null` in places where it really wasn’t appropriate, didn’t make sense, and certainly didn't make code any cleaner or easy to understand.  

You’d see things like this:

``` java
if (Optional.ofNull(variable).isPresent()) then {
           someCode();
}
```
Which, of course, isn’t any clearer than just checking if variable is `Null`.  Its' possible this might have led to a backlash against `Optional`, and it seems to me that it’s not used today as much as it should be.  So let’s look at what `Optional` really does.

# The Purpose of Optional

To better understand how `Optional` should be used, we'll use an example that could be a real world programming situation:

## An Example

Imagine that we have a system with customer accounts and invoices.  Somewhere, there’s a persistence layer that will look up a customer account and instantiate a domain object for the account.  In most systems I’ve seen, it’ll have a signature something like this:

``` java
public CustAccount getAccount(String accountNumber)
```

In this system, if we have an invoice and it has an account number referenced on it, then that account has to exist on file.  If not, then we have data corruption and that’s a real problem.  But if someone enters an account number into a screen to do a lookup, then they may have just typed in an invalid account, and that’s not necessarily a problem.

These two different situations need to be resolved differently, and the situation is all about the context of the code that calls the `getAccount()` method.  Usually, systems are written so that `getAccount()` just returns `Null` if the account isn’t on file, and it’s up to the calling code to remember to check for `Null`, and decide what it means if the answer is `Null`.

But that’s not really how you are supposed to use `Null`, and Java has always had an alternative that seems to be rarely used, and that’s `Checked Exceptions`.  

## Checked Exceptions

`Checked Exceptions` are a way of declaring to any calling routines that your method might legitimately not be able to return a result, and forcing them to deal with that possibility.  Of course, many programmers just put an empty catch clause (or put a `stackdump` in it), or throw the exception all the way up to the top where it’s mishandled.

So, in order to set this up properly, we really need two methods in our persistence layer, one for the case where the account needs to be on file, and one where it might not be there.  Something like this:

``` java
public CustAccount getAccountIfPresent(String accountNumber) throws MissingAccountException

public CustAccount getAccount(String accountNumber)
```

`MissingAccountException` is a checked exception.  If `getAccount()` cannot find the account on file, then it throws some sort of `RuntimeException`, maybe something called `DataCorruptionException`.  Neither method is capable of returning a `Null` value.

The real problem with `getAccountIfPresent()` is that it’s cumbersome and forces awkward code structures around the try-catch blocks and scoping of variables.  So almost nobody ever uses this.  They just pass `Null` when the account can’t be found.

The problem with that, though, is that there's no way tell a consumer of `getAccount()` that `Null` is a reasonable return value or what it means without checking the code itself to see how the result is determined.  And that's a problem.  

## Optional to the Rescue

This is where `Optional` comes in.  With `Optional` you can do away with the checked exceptions and lock the answer in a wrapper that requires the calling method to deal with the possibility that the answer might be missing when you take it out.  So the method signature would change to this:

``` java
public Optional<CustAccount> getAccountIfPresent(String accountNumber)
```

With this, there’s only one way to get that `CustAccount` out of the `Optional` without handling its possible absence  That’s the `Optional.get()` method, which should only be used inside an if statement that has already checked `Optional.isPresent()` first.  Until Java 9, if you had `Optional` in a stream, you needed to use something like this:

``` java
stream().filter(Optional::isPresent).map(Optional::get)....
```

But now the Optional class has it’s own stream() method, which does the filter and get in one step.  So you can do this:

``` java
stream().flatMap(Optional::stream)...
```
All of which means that there's no longer any place that the Java language forces you to use `Optional.get()` any more.

Brian Goetz has said in a famous StackOverflow response:

> (Public service announcement: NEVER call Optional.get unless you can prove it will never be null; instead use one of the safe methods like orElse or ifPresent. In retrospect, we should have called get something like getOrElseThrowNoSuchElementException or something that made it far clearer that this was a highly dangerous method that undermined the whole purpose of Optional in the first place. Lesson learned. (UPDATE: Java 10 has Optional.orElseThrow(), which is semantically equivalent to get(), but whose name is more appropriate.))

Many programmers first experience with `Optional` is when they start using the `Streams` facility, and encounter it with the `.findAny()`, or `findFirst()` methods.  These methods return `Optional` because it's entirely possible that your stream, especially after filtering, is empty and there may legitimately be no stream elements returned by these methods.  

# Learning From Kotlin

Kotlin handles this concept in a much more natural manner and it's instructive to take a look at it to understand Optional better.

## Most Variables Cannot be Null

In Kotlin, the default for a data type is that it cannot be null.  When you declare a variable, it needs to be initialized right away.  If it's a field, it needs to be initialized in the constructor or in the `init{}` block that runs right after the constructor.

This forces you to think about how your fields and variables are created and used.  Unlike Java the default instantiation isn't Null, and if you want to allow Null you have to put in effort to implement it.

Think about it.  Nobody really wants `Null`.  It's a pain, and the reason that you have to deal with it can often be because somebody else didn't take the time to deal with it properly in some method you rely on.  

## Nullable Variables

Kotlin allows you to declare a variable as "Nullable".  You do it like this, putting a "?" after the type name:

``` kotlin
var myNumber : Int?
```
No big deal right?  Think again.  `myNumber` isn't an `Int`, it's a Nullable `Int` or an `Int?`.  You can't use it like an `Int`.  You can't do:

``` kotlin
myNumber = myNumber + 2;
```
and you can't do this, either:
``` kotlin
if (myNumber < 20) {}
```
Because those are operations that work on `Int`, which `myNumber` is not.

## Nullable Methods

Methods (or "functions" in Kotlin) can return Nullable types.  So you could declare something like this:

``` kotlin
fun myFunction():Int? {

}
```
But then you have the same constraints that a variable of type `Int?` would have.  It's not an `Int` and cannot be treated as one.

## Null Safe Operations

Kotlin supplies the "Safe Call Operator" - `?`.  If you want to do something with a nullable type, you put that `?` after the variable name.  Then the method call will only happen if the variable isn't Null.  Something like this:

``` kotlin
 val myString : String? = myNumber?.toString()
```
If `myNumber` is Null, the `toString()` won't be called and the result will still be `Null`.

## The "Elvis" Operator

Here's something that you cannot do:

``` kotlin
val myString: String = myNumber?.toString()
```

Why not?

Because `myNumber?.toString()` is not a `String`, it's a `String?`.  You cannot assign it to a `String` you cannot perform `String` operations on it (at least not without the Safe Call Operator).  Essentially, `myNumber` is an `Int` locked away in a safe, and you cannot take it out without telling the compiler what to do if it's `Null`.

Enter the "Elvis Operator":

``` kotlin
val myString: String = myNumber?.toString()?: "I'm empty"
```
If `myNumber` is not `Null`, then `myString` will have whatever `myNumber` translates to as a `String`.  If it is `Null` then it will be assigned "I'm empty".

## The Dangerous Alternative

The last way to get a value out of a nullable type is the `!!` operator.  The Kotlin documentation describes it this way:

>The third option is for NPE-lovers: the not-null assertion operator (!!) converts any value to a non-null type and throws an exception if the value is null.

You can think of it as the "Look ma! No hands!" operator.  Or the "Please sir, may I have another NullPointerExeception" operator.  You can see how the Kotlin documentation clearly discourages its use.  Event the choice of `!!` means it just jumps out of the code at you.

For the sake of completeness, this is how you would use it:

``` kotlin
val myString : String = myNumber!!.toString()
```
This will throw an NPE if `myNumber` is `Null`. It's really the only way to get an NPE in Kotlin.

## How This Relates to Optional

I think you can see Kotlin nullable types relate to `Optional` in Java.  The Safe Call Operator is fairly close to `Optional.map()`, the Elvis Operator is similar to `Optional.orElse()` and the `!!` operator corresponds to `Optional.get()`.

The big difference between Kotlin and Java in this respect is that in Java the use of `Optional` is optional.  It's mandatory in Kotlin.  Sure, you could thumb you nose at the "man" and say, "I'm just gonna make all my variables nullable".  But you'd pretty quickly see that your code had become an unreadable nightmare of Safe Call Operators and convoluted logic.

You couldn't do this:

``` kotlin
var index : Int? = 0
.
.
var counter: Int? = index++
```
because the "++" operator doesn't work in Int?.  So you'd have to do this:

``` kotlin
var index : Int? = 0
.
.
var counter: Int? = index?.let{it++}
```

Eventually you'll decide not to make all of your variables nullable, and then you'll have to decide which ones should be nullable.  Ultimately you'll end up having to decide why a variable should be nullable, and what it means when it is nullable.  At that point, it's not really about nullable any more, and it's more about the contexts where data and answers might not be available.

# Conclusion

Understanding this purpose of `Optional` makes it much easier to use it appropriately.  Ideally, you shouldn't ever be creating an `Optional` unless it's the return value of a method.  `Optional` shouldn't be a field in a class, and `Optional` shouldn't be a parameter passed to a method.  Further, there needs to be logic somewhere that unpacks the `Optional` and handles the situations where it's empty in a contextually appropriate manner.

The motivation for `Optional` isn't to avoid `Null`.  Yes, it does this, but that's a side-effect.  Really, `Optional` is about handling situations where data might be missing, for whatever reason.  It's about telling the consumers of a method that, "Hey!  There may not be a result here", and forcing them to deal with it without having to look into your code to understand what `Null` means, or even if you method might return it.
