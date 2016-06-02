Monads, Functors, and Applicative Functors oh my! Have you ever wondered what any of these concepts are and why they are useful? Let's dive into some explanations and then we can see how useful they really can be in your daily programming.
  
# Swift's Optional Type

So how does Swift define Optional? Opening up the generated Swift header we see the following:

```swift
/// A type that can represent either a `Wrapped` value or `nil`, the absence
/// of a value.
public enum Optional<Wrapped> : _Reflectable, NilLiteralConvertible {
    case None
    case Some(Wrapped)
    /// Construct a `nil` instance.
    public init()
    /// Construct a non-`nil` instance that stores `some`.
    public init(_ some: Wrapped)
    /// If `self == nil`, returns `nil`.  Otherwise, returns `f(self!)`.
    @warn_unused_result
    public func map<U>(@noescape f: (Wrapped) throws -> U) rethrows -> U?
    /// Returns `nil` if `self` is `nil`, `f(self!)` otherwise.
    @warn_unused_result
    public func flatMap<U>(@noescape f: (Wrapped) throws -> U?) rethrows -> U?
    /// Create an instance initialized with `nil`.
    public init(nilLiteral: ())
}
```

Reading the first comment in the source code gives us a clear definition of what an Optional really is. 

> A type that can represent either a Wrapped value or nil, the absence of a value.

In other words, an Optional is a container that wraps the concept of having a value or not.

So why is this useful? Because Swift is a strongly typed language, it makes us code for both cases of an Optional (having a value .Some or no value at all .None). When we declare variables with let or var and make them Optional, we are telling the compiler that our variable may contain a value or not. Any subsequent access to our variable will require us to check that it contains a value before performing actions on the wrapped value.

A simple example:

```swift
var string: String? = "Yay! I contain a value."
var emptyString: String?

// We unwrap our Optional before we print it out
if let string = string {
  print(string)
}

// Try to unwrap our emptyString variable
if let emptyString = emptyString {
  // we won't make it here
  print(emptyString)
} else {
  // here we can handle the case where the Optional
  // doesn't contain a value
  print("Your string was empty")
}
```
---

# What the Functor?

So what is a functor? It sounds complicated and only adding to it's complex nature, it's roots are in Mathematics (Category Theory to be a bit more specific). But let's try and define it in terms a programmer understands instead of mathematical terms.

> A functor is any type that defines how map applies to it. 

It just so happens that Optional defines this very method. Let's look again at the declaration for Optional, but cut down on the noise a bit.

```swift
public enum Optional<T> {
    // ...

    public func map<U>(f: (T) -> U) -> U?

    // ...
}
```

If we break down Optional's implementation of map, it's a function that accepts another function as it's only argument. This other function, we'll call it f, accepts an object of the type T and returns an object of the type U. And finally, map returns U? which is an Optional of the type U.

A more concrete example of map being used looks something like this:

```swift
let optionalWithValue: Int? = 42
let mappedResultWithValue = optionalWithValue { "\($0 * 2)" } // mappedResultWithValue is initialized to "84"
```

In this case, if we fill in the generic blanks, T is Int and U is String. f accepts an Int and returns a String. And finally, map returns a String?. In our case it's value is "84".

The more interesting example of map in use is when the wrapped value is nil.

```swift
let optionalWithoutValue: Int?
let mappedResultWithoutValue = optionalWithoutValue { "\($) * 2)" } // mappedResultWithoutValue is initialized to nil
```

When our Optional doesn't contain a value, we skip over calling f with our missing value and just return .None. So if I were guessing how Apple implemented map on the Optional class, I'd figure it would look similar to this:

```swift
func map<U>(@noescape f: (Wrapped) throws -> U) rethrows -> U? {
  switch self {
  case .Some(let x): return try f(x)
  case .None: return .None
  }
}
```

So Swift's Optional is a functor because it meets the requirement of defining the map method.

--- 

# Applicative Functor

Armed with the knowledge of what a Functor is, next up is Applicative Functor. Let's start with a definition in programmer terms.

> An applicative functor is any type that defines apply.

Looking back at the declaration for Optional, we don't find any methods defining apply. Hmm, by default Optionals aren't applicative functors. But wait! We can add an extension to this built in type and make it work. 

```swift
extension Optional {
  func apply<U>(f: (Wrapped -> U)?) -> U? {
    return f.flatMap { self.map($0) }
  }
}
```

Looking at the implementation of apply kind of melts your brain, so let's break it down and explain it in simpler terms. First we should notice a very important detail buried in the middle of the function. f's type is (Wrapped -> U)?. f is an Optional! This little interesting tidbit is the key difference between map and apply. So looking into the implementation we see that we are calling flatMap on f. So if f is not nil we are going to call self.map($0) where $0 is referring to f. So putting it all together, we pass in an Optional function to apply. If the function exists, we call map on our Optional and pass in our function. So if either our function is nil or our Optionals value is nil, we return nil. If both are not nil, we return the result of calling our function with the value wrapped in the Optional. 

So now that we've added this method to Swift's Optional, it's now an applicative functor!

---

# Monads Ahoy

Last but not least, we come to Monads. Let's again start with a definition in programmer terms.

> A monad is any type that defines flatMap.

Looking at the declaration code for Optional and right away flatMap is screaming in our face. So Swift's Optional type is a Monad without any further modification! Let's make an educated guess how Swift defines flatMap on Optional.

```swift
func flatMap<U>(@noescape f: (Wrapped) throws -> U?) rethrows -> U? {
  switch self {
  case .Some(let x): return try f(x)
  case .None: return .None
}
```

Comparing flatMap and map on Optional you might be asking yourself what's the difference? Really there is only one specific difference and it's that f in map cannot return another Optional whereas f in flatMap can return an Optional.

---

# Why It Matters

So now that we have a good understanding of what each of these abstract concepts mean, it kind of begs the question, what are they good for?

### Mapping out Optionals

Formatting NSDate? can be a bit tricky. Because it's an Optional we are forced into using if/let syntax or heaven forbid force unwrapping the Optional.

```swift
import Foundation

var date1: NSDate?
var formatted1: NSString?

// some time after date1 *may* have been set or initialized

if let date1 = date1 {
  formatted1 = NSDateFormatter().stringFromDate(date1)
}
```

Here's functionally equivalent code using `map`:

```swift
var date2: NSDate?

// some time after date2 *may* have been set or initialized

var formatted2 = date2.map(NSDateFormatter().stringFromDate)
```

To be clear, we aren't aiming for code golf. But switching over from the `if/let` to using `map` makes our intent more clear in my opinion. If the date isn't nil, let's format it as a string. If it is nil, our formatted date string is nil. 

Another wonderful usage of `map` is while we are interpolating values in a string:

```swift
var numberOfFriends: Int?

// some time after numberOfFriends *may* 
// have been set or initialized

var friendsStatus = numberOfFriends.map { 
  "You have \($0) friends."
} ?? "No friends yet." 
```

This allows us to choose two completely different strings based on whether our `numberOfFriends` variable has a value or not.

One last example of a great usage for `map`. What if we want to read the contents of a url?

```swift
var webURL: NSURL? = NSURL(string: "http://rouxbe.com/")

var webContents = webURL.map {
  NSData(contentsOfURL: $0)
}.map {
  NSString(data: $0, encoding: NSUTF8StringEncoding)
}

webContents.map { print($0) }
```

If I was a bit more mischievous, I could leave the example like that and go on with the post. But the most astute reader who really gets these concepts would point out the mistake in the code sample above. In case you didn't catch it, that's ok. The problem with our code above is that map accepts a closure/function that does not return an `Optional`. `NSData`'s initializer that accepts an `NSURL` is a failable initializer. Since it can return `nil`, our closure can return `nil`. Xcode will complain inside our second call to `map` that `$0` has not been unwrapped. So what do we do? 

### FlatMap to the Rescue
```swift
var webURL: NSURL? = NSURL(string: "http://rouxbe.com/")

var webContents = webURL.flatMap {
  NSData(contentsOfURL: $0)
}.flatMap {
  NSString(data: $0, encoding: NSUTF8StringEncoding)
}

webContents.map { print($0) }
```

### How do we really apply

Seeing map and flatMap in action should get you started on potential places in code where you can use these concepts or refactor existing logic. But where does apply fit in? When do you ever have an Optional function hanging around that you may want to call with an Optional argument?

Let's dive in with an example. Suppose you have a function with an optional completion handler as one of it's arguments.

```swift
typealias ParseResponseHandler = (responseData: NSData?) -> [String: AnyObject]

func getJsonFromUrl1(urlString: String, 
    parseResponseHandler: ParseResponseHandler?) -> [String: AnyObject]? {
  let url = NSURL(string: urlString)
  
  if let url = url, responseData = NSData(contentsOfURL:url), 
      parseResponseHandler = parseResponseHandler {
    return parseResponseHandler(responseData: responseData)
  } else {
    return nil
  }
}
```

Notice we have to use if/let syntax to unwrap a slew of optionals before we call the handler with the data. We can simplify this a bit!

```swift
func getJsonFromUrl2(urlString: String, 
    parseResponseHandler: ParseResponseHandler?) -> [String: AnyObject]? {
  let url = NSURL(string: urlString)

  return url.flatMap {
    NSData(contentsOfURL: $0)
  }.apply(parseResponseHandler)
}
```

# Wrapping Up

Hopefully by now you can see how useful functors, applicative functors, and monads can really be. Apple's decision to make Optionals work as functors and monads really allows writing confident code a cinch. And just to keep your brain thinking about this crazy topic, Swift's Array type is also a functor and a monad!
