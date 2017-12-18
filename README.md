# Unit Testing in C#
[//]: # (robkeim: Great stuff here! I don't know where I'd put it but one thing that would be helpful to know for people learning about unit tests is that you should test both positive and negative behavior. It's common for people when starting out to only test the cases they expect to work, and not the cases that you know won't work.)

## Purpose
The purpose of this best practices document is to offer suggested best practices when writing unit tests in C#.

## What Makes a Good Unit Test
- **Fast**. It is not uncommon for mature projects to have thousands of unit tests. Unit tests should take very little time to run. Milliseconds.
- **Isolated**. Unit tests are standalone and can be run in isolation and have no dependencies on any outside factors such as a file system or database.
- **Repeatable**. Running a unit test should be consistent with its results, that is, it always returns the same result if you do not change anything in between runs.
- **Self-Checking**. The test should be able to automatically detect if it passed or failed without any human interaction.
- **Timely**. A unit test and the code being tested should not take a disproportionally long time to write.
[//]: # (robkeim: I read this in two different ways: 1. A unit test shouldn't take a disproportionally long time to write compared to the code. If that's what you're going for I'd provide some guideline on "disproportionally"... maybe 1.5x the cost of writing the code? 2. Writing a block of work - code and test - shouldn't take a "long" time. In that case as well, I'd provide another guideline. If your work unit takes longer than 2 days it should be broken into smaller, more managable sub-components.)

## Lets Speak the Same Language
The term *mock* is unfortunately very misused when talking about testing. The following defines the most common types of *fakes* when writing unit tests:

*Fake* - A fake is a generic term which can be used to describe either a Stub or a Mock object. Whether it is a Stub or a Mock depends on the context in which it's used. So in other words, a Fake can be a Stub or a Mock.

*Mock* - A mock object is a fake object in the system that decides whether or not a unit test has passed or failed. A Mock starts out as a Fake until it is asserted against.

*Stub* - A stub is a controllable replacement for an existing dependency (or collaborator) in the system. By using a stub, you can test your code without dealing with the dependency directly. By default, a fake starts out as a stub.

Consider the following code snippet:

```csharp
var mockOrder = new MockOrder();
var purchase = new Purchase(mockOrder);

purchase.ValidateOrders();

Assert.True(purchase.CanBeShipped);
```

This would be an example of Mock being used improperly. In this case, it is a stub. We're just passing in the Order as a means to be able to instantiate `Purchase` (the system under test). The name `MockOrder` is also very misleading because again, the order is not a mock.

A better approach would be

```csharp
var fakeOrder = new FakeOrder();
var purchase = new Purchase(fakeOrder);

purchase.ValidateOrders();

Assert.True(purchase.CanBeShipped);
```

By renaming the class to `FakeOrder`, we've made the class a lot more generic, the class can be used as a mock or a stub. Whichever is better for the test case. In the above example, `FakeOrder` is used as a stub. We're not using the `FakeOrder` in any shape or form during the assert. We just passed it into the `Purchase` class to satisfy the requirements of the constructor.

To use it as a Mock, we could do something like this

```csharp
var fakeOrder = new FakeOrder();
var purchase = new Purchase(fakeOrder);

purchase.ValidateOrders();

Assert.True(fakeOrder.Validated);
```

In this case, we are checking a property on the Fake (asserting against it), so in the above code snippet the `fakeOrder` is a Mock.

**It's important to get this terminology correct. If you call your stubs "mocks", other developers are going to make false assumptions about your intent.**

The main thing to remember about mocks versus stubs is that mocks are just like stubs, but you assert against the mock object, whereas you do not assert against a stub. Which means that only mocks can break your tests, not stubs.

## Best Practices
1. [Arranging Your Tests](#arranging-your-tests)
1. [Naming Your Tests](#naming-your-tests)
1. [Avoid Magic Strings](#avoid-magic-strings)
1. [No Logic In Tests](#no-logic-in-tests)
1. [Prefer Helper Methods to Setup and Teardown](#prefer-helper-methods-to-setup-and-teardown)
1. [Avoid Multiple Asserts](#avoid-multiple-asserts)
1. [Write Minimally Passing Tests](#write-minimally-passing-tests)

### Arranging Your Tests
The AAA (Arrange, Act, Assert) pattern is a typical pattern when unit testing, and consists of three main actions:
- *Arrange* your objects, creating and setting them up as necessary.
- *Act* on an object.
- *Assert* that something is as expected.

#### Why?
- Clearly separates what is being tested from the *arrange* and *assert* steps.
- Less chance to intermix assertions with "Act" code.

#### Bad:
```csharp
public void IsValidWord_WhenInputIsNull_ReturnsFalse()
{
    // Arrange
    var glossary = new Glossary();

    // Assert
    Assert.False(glossary.IsValidWord(null));
}
```

#### Better:
```csharp
public void IsValidWord_WhenInputIsNull_ReturnsFalse()
{
    var numberValidator = new NumberValidator();
    
    var result = numberValidator.IsValidWord(null);

    Assert.False(result);
}
```
[//]: # (robkeim: Is there a reason you removed the Arrange, Act, Assert comments in the better example? It makes it seem like you're suggesting it's better not to have those, which you totally might be but I'm personally a fan of leaving them in. You also don't seem to use them consistently throughout the examples so maybe you prefer not having them in a general case and added only here?)
### Naming Your Tests
The name of your test should consist of three parts
- The name of the method being tested.
- The scenario under which it's being tested.
- The expected behavior when the scenario is invoked.

#### Why?
- Naming standards are important because they explicitly express intent of the test.

#### Bad:
```csharp
public void Test_Invalid()
{
    var glossary = new Glossary();

    var result = glossary.IsValidWord(null);

    Assert.False(result);
}
```

#### Better:
```csharp
public void IsValidWord_WhenInputIsNull_ReturnsFalse()
{
    var glossary = new Glossary();

    var result = glossary.IsValidWord(null);

    Assert.False(result);
}
```

### Avoid Magic Strings
Naming variables in unit tests is as important, if not more important, than naming variables in production code. Unit tests should not contain magic strings.

#### Why?
- Prevents the need for the reader to inspect the production code to figure out what makes the value special.
- Explicitly shows what you're trying to *prove* rather than trying to *accomplish*.

#### Bad:
```csharp
public void ParseWord_InputIsNumber_ReturnsInvalidInputErrorCode()
{
    var glossary = new Glossary();

    var result = glossary.ParseWord("1");

    Assert.Equal(-1, result);
}
```
[//]: # (robkeim: Looks like you started dropping the When in the second part here in the examples. Should when be removed/added everywhere for consistency?)

#### Better:
```csharp
public void ParseWord_InputIsNumber_ReturnsInvalidInputErrorCode()
{
    var glossary = new Glossary();
    const int INVALID_INPUT = -1;

    var result = glossary.ParseWord("1");

    Assert.Equal(INVALID_INPUT, result);
}
```

### No Logic in Tests
When writing your unit tests avoid manual string concatenation and logical conditions such as `if`, `while`, `for`, `switch`, etc.

#### Why?
- Less chance to introduce a bug inside of your tests.
- Focus on the end result, rather than implementation details.

#### Bad:
```csharp
public void ConcatenateWords_TwoWords_ReturnsStringWithCommaBetween()
{
    var glossary = new Glossary();
    var firstWord = "a";
    var secondWord = "b";

    var result = glossary.ConcatenateWords(firstWord, secondWord);

    Assert.Equals(string.Format("{0},{1}", firstWord, secondWord), result);
}
```
[//]: # (Where's your string interpolation at? We're almost in 2017 :)

#### Better:
```csharp
public void ConcatenateWords_TwoWords_ReturnsStringWithCommaBetween()
{
    var glossary = new Glossary();

    var result = glossary.ConcatenateWords("a", "b");

    Assert.Equals("a,b", result);
}
```

### Prefer Helper Methods to Setup and Teardown
If you require a similar object or state for your tests, prefer a helper method than leveraging Setup and Teardown attributes if they exist.

#### Why?
- Less confusion when reading the tests since all of the code is visible from within each test.
- Less chance of setting up too much or too little for the given test.
- Less chance of sharing state between tests which creates unwanted dependencies between them.

#### Bad:
```csharp
Glossary glossary;

[SetUp]
public void Initialize()
{
    glossary = new Glossary();
}

// more tests..

public void ParseWord_NullValue_ThrowsArgumentException()
{
    Assert.Throws<ArgumentException>(() => glossary.ParseWord(null));
}

public void ParseWord_EmptyString_ReturnsEmptyString()
{
    Assert.Empty(glossary.ParseWord(""));
}
```

#### Better:
```csharp
public void ParseWord_NullValue_ThrowsArgumentException()
{
    var glossary = CreateDefaultGlossary();

    var result = () => glossary.ParseWord(null);

    Assert.Throws<ArgumentException>(result);
}

public void ParseWord_EmptyString_ReturnsEmptyString()
{
    var glossary = CreateDefaultGlossary();

    var result = glossary.ParseWord("");

    Assert.Empty(result);
}

// more tests..

public Glossary CreateDefaultGlossary()
{
    return new Glossary();
}
```

### Avoid Multiple Asserts
When writing your tests, try to only include one Assert per test. Common approaches to using only one assert include:
- Create a separate test for each assert.
- Use parameterized tests.

#### Why?
- If one Assert fails, the subsequent Asserts will not be evaluated.
- Ensures you are not asserting multiple cases in your tests.

#### Bad:
```csharp
public void IsValidWord_WhenInputIsNullOrEmpty_ReturnsFalse()
{
    var glossary = new Glossary();

    Assert.False(glossary.IsValidWord(null)); // potential NullException
    Assert.False(glossary.IsValidWord("")); // this never gets executed if the above test fails
}
```

#### Better:
```csharp
[InlineData(null)]
[InlineData("")]
public void IsValidWord_WhenInputIsNullOrEmpty_ReturnsFalse(string input)
{
    var glossary = new Glossary();
    
    var result = glossary.IsValidWord(input);

    Assert.False(result);
}
```

*Note: A common exception to this best practice is when you are validating the state of an object.*

### Write Minimally Passing Tests
The input to be used in a unit test should be the simplest possible in order to pass the test.

#### Why?
- Tests become more resilient to future changes in the codebase.
- Closer to testing behavior over implementation.

#### Bad:
```csharp
[InlineData("aardvark")]
[InlineData("elephant")]
[InlineData("iguana")]
[InlineData("orangutan")]
[InlineData("unicorn")]
public void WordStartsWithVowel_InputStartsWithVowel_ReturnsTrue(string input)
{
    var glossary = new Glossary();

    var result = glossary.WordStartsWithVowel(input);

    Assert.True(result);
}
```

#### Better:
```csharp
[InlineData("a")]
[InlineData("e")]
[InlineData("i")]
[InlineData("o")]
[InlineData("u")]
public void WordStartsWithVowel_InputStartsWithVowel_ReturnsTrue(string input)
{
    var glossary = new Glossary();

    var result = glossary.WordStartsWithVowel(input);

    Assert.True(result);
}
```
