# Test-driving value object

In this stage, we skip further ahead in time. Johnny and Benjamin just extracted a value object type for a train ID and will work further to specify it.

## Initial value object

**Johnny**: Oh, you're back. The refactoring is over -- we've got a nice value object type extracted from the current code. Here's the source code of the `TrainId` class:

```csharp
public class TrainId
{
 private readonly string _value;

 public static TrainId From(string trainIdAsString)
 {
  return new TrainId(trainIdAsString);
 }

 private TrainId(string value)
 {
  _value = value;
 }
 
 public override bool Equals(object obj)
 {
  return _value == ((TrainId) obj)._value;
 }

 public override string ToString()
 {
  return _value;
 }
}
```

**Benjamin**: Wait, but we don't have any Specification for this class yet. Where did all of this implementation come from?

**Johnny**: That's because while you were drinking your tea, I extracted this type from an existing implementation that was already a response to false Statements.

**Benjamin**: I see. So we didn't mock the `TrainId` class in other Statements, right?

**Johnny**: No. This is a general rule -- we don't mock value objects. They don't represent abstract, polymorphic behaviors. For the same reasons we don't create an interface and mocks for `string` or `int`, we don't do that for `TrainId`.

## Value semantics

**Benjamin**: So, given the existing implementation, is there anything left for us to write?

**Johnny**: Yes. I decided that this `TrainId` should be a value object and my design principles for value objects demand it to hold some more guarantees than ones resulting from a mere refactoring. Also, don't forget that the comparison of train ids needs to be case-insensitive. This is something we've not specified anywhere.

**Benjamin**: You mentioned "more guarantees". Do you mean equality?

**Johnny**: Yes, C# as a language expects equality to follow certain rules. I want this class to implement them. Also, I want it to implement `GetHashCode` properly and make sure instances of this class are immutable. Last but not least, I'd like the factory method `From` to throw an exception when null or empty string are passed. Neither of these inputs should lead to creating a valid ID.

**Benjamin**: Sounds like a lot of Statements to write.

**Johnny**: Not really. Add the validation and lowercase comparison to TODO list. In the meantime -- look -- I downloaded a library that will help me with the immutability and equality part. This way, all I need to write is:

```csharp
[Fact] public void
ShouldHaveValueSemantics()
{
 var trainIdString = Any.String();
 var otherTrainIdString = Any.OtherThan(trainIdString);
 Assert.HasValueSemantics<TrainId>(
   new Func<TrainId>[]
   {
    () => TrainId.From(trainIdString)
   },
   new Func<TrainId>[]
   {
    () => TrainId.From(otherTrainIdString);
   },
 );
}
```

This assertion accepts two arrays:

- the first array contains factory functions that create objects that should be equal to each other. For now, we only have a single example, because I didn't touch the lowercase vs uppercase issue. But when I do, the array will contain more entries to stress that ids created from the same string with different letter casing should be considered equal.

After evaluating this Statement, I get the following output:

```txt
- TrainId must be sealed, or derivatives will be able to override GetHashCode() with mutable code.

- a.GetHashCode() and b.GetHashCode() should return same values for equal objects.

- a.Equals(null) should return false, but instead threw System.NullReferenceException: Object reference not set to an instance of an object.

- '==' and '!=' operators are not implemented
```

**Benjamin**: Very clever. So these are the rules that our `TrainId` doesn't follow yet. Are you going to implement them one by one?

**Johnny**: Hehe, no. The implementation would be so dull that I'd either use my IDE to generate the necessary implementation or, again, use a library. Lately, I prefer the latter. So let me just download a library called [Value](https://www.nuget.org/packages/Value/) and use it on our `TrainId`.

First off, the `TrainId` needs to inherit from `ValueType` generic class like this:

```csharp
public class TrainId : ValueType<TrainId>
```

This inheritance requires us to implement the following method:

```csharp
protected override IEnumerable<object> GetAllAttributesToBeUsedForEquality
{
 throw new NotImplementedException();
}
```

**Benjamin**: Weird, what does that do?

**Johnny**: It's how the library automates equality implementation. We just need to return an array of values we want to be compared between two instances. As our equality is based solely on the `_value` field, we need to return just that:

```csharp
protected override IEnumerable<object> GetAllAttributesToBeUsedForEquality
{
 yield return _value;
}
```

We also need to remove the existing Equals() method.

**Benjamin**: Great, now the only reason for the assertion to fail is:

```txt
- TrainId must be sealed, or derivatives will be able to override GetHashCode() with mutable code.
```

I am impressed that the library took care of the equality methods, equality operators, and `GetHashCode()`.

**Johnny**: Nice, huh? Ok, let's end this part and add the `sealed` keyword. The complete source code of the class looks like this:

```csharp
public sealed class TrainId : ValueType<TrainId>
{
 private readonly string _value;

 public static TrainId From(string trainIdAsString)
 {
  return new TrainId(trainIdAsString);
 }

 private TrainId(string value)
 {
  _value = value;
 }

 protected override IEnumerable<object> GetAllAttributesToBeUsedForEquality
 {
  yield return _value;
 }

 public override string ToString()
 {
  return _value;
 }
}
```

**Benjamin**: And the Statement is true.

## Case-insensitive comparison

**Johnny**: What's left on our TODO list?

**Benjamin**: Two items:

* Comparison of train ids should be case-insensitive.
* `null` and empty string should not be allowed as valid train ids.

**Johnny**: Let's do the case-insensitive comparison, it should be relatively straightforward.

**Benjamin**: Ok, you mentioned that we would need to expand the Statement you wrote, by adding another "equal value factory" to it. Let me try. What do you think about this?

```csharp
[Fact] public void
ShouldHaveValueSemantics()
{
 var trainIdString = Any.String();
 var otherTrainIdString = Any.OtherThan(trainIdString);
 Assert.HasValueSemantics<TrainId>(
   new Func<TrainId>[]
   {
    () => TrainId.From(trainIdString.ToUpper())
    () => TrainId.From(trainIdString.ToLower())
   },
   new Func<TrainId>[]
   {
    () => TrainId.From(otherTrainIdString);
   },
 );
}
```

How about that? From what you explained to me, I understand that by adding a second factory to the first array, I say that both instances should be treated as equal - the one with lowercase string and the one with uppercase string.

**Johnny**: Exactly. Now, let's make the Statement true. Fortunately, we can do this by changing the `GetAllAttributesToBeUsedForEquality` method from:

```csharp
protected override IEnumerable<object> GetAllAttributesToBeUsedForEquality
{
 yield return _value;
}
```

to:

```csharp
protected override IEnumerable<object> GetAllAttributesToBeUsedForEquality
{
 yield return _value.ToLower();
}
```

Aaand done! The assertion checked `Equals`, equality operators and `GetHashCode()` and everything seems to be working. We can move on to the next item on our TODO list.

## Input validation

**Johnny**: Let's take care of the `From` method - it should disallow null input -- we expect an exception when we pass a `null` inside.

For now, the method looks like this:

```csharp
public static TrainId From(string trainIdAsString)
{
 return new TrainId(trainIdAsString);
}
```

**Benjamin**: OK, let me write a Statement about the expected behavior:

```csharp
[Fact] public void
ShouldThrowWhenCreatedWithANullInput()
{
  Assert.Throws<ArgumentNullException>(() => TrainId.From(null));
}
```

That was easy, huh?

Johnny: Thanks. The Statement is currently false because it expects an exception but nothing is thrown. Let's make it true by implementing the `null` check.

```csharp
public static TrainId From(string trainIdAsString)
{
 if(trainIdAsString == null)
 {
  throw new ArgumentNullException(nameof(trainIdAsString));
 }
 return new TrainId(trainIdAsString);
}
```

TODO!!!!!!!!!!!!!!!!


Do we specify the `ToString()`? Not necessarily... There is already a Statement that will turn false.




Other libraries - mutability detector and equalsverifier

TODO value library (Value)
TODO sealed


Driven by design principles, already covered by an existing test, because we don't mock value objects.
