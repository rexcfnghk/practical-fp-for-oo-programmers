- title : Practical FP for OO Programmers
- description : Practical FP for OO Programmers
- author : Rex Ng
- theme : serif
- transition : default

***

## Practical FP for OO Programmers

***

### What is functional programming?

- Uncle Bob defines functional programming as

> Functional programming is programming without assignment statements

[Clean Coder](https://8thlight.com/blog/uncle-bob/2012/12/22/FPBE1-Whats-it-all-about.html)

***

![Safety, Quality, Quantity](http://www.safetysign.com/images/source/large-images/D3948.png)

***

### What is the problem with assignment

Difficult to reason about

***

### Just to be clear

- Assignments, and thereby side effects are necessary for any useful programs
- FP tries to contain them in a disciplined manner

***

### A typical method in a C♯ interface

    [lang=cs]
    public class Course { }

    public class Student
    {
        public IEnumerable<Course> Courses 
            { get; set; }
    }

    public interface IStudentService
    {
        IEnumerable<Course> GetAllCourses(
            Student student);
    }

What can a concrete implementation of `IStudentService.GetAllCourses` do?

---

### It can

- Save to database
- Write to console, log
- Try catch exceptions
- **Update properties of `student`**


    [lang=cs]
    public IEnumerable<Course> GetAllCourses(
        Student student)
    {
        var courses = student.Courses;

        // Side-effect
        student.Courses = null;

        return courses;
    }

Not ‘safe’ to call

' Counterargument: how about removing the setter for `Courses`?
' Then you have to make sure `Course` does not have setters as well
' Deep vs shallow immutability

---

### Problems

1. To the caller, a call to `GetAllCourses` returns a modified `student`
2. Caller is now responsible for tracking changes to the `student` parameter
3. Calls to `GetAllCourses` cannot be reordered as it affects the state of the program flow

---

### Why

Asking for `Courses` changes the `student` input?

***

### How F♯ tackles this problem

- The assignment operator (`=` in C♯, `<-` in F♯) is seldom used
- F♯ has immutable data types (records, unions, lists, maps, sets, etc.)

---

### F♯ version of `GetAllCourses`

    // Alias to string
    type Course = string

    // Record type (think of it as immutable class)
    type Student = { Courses: seq<Course> }

    // getAllCourses: student: Student -> seq<Course>
    let getAllCourses student =
        // Compile error
        student.Courses <- null

        // Last expression in F♯ functions is the return value
        student.Courses

---

### How can I update `student.Courses` then?

---

### Answer

You don’t

    // Alias to string
    type Course = string

    // Record type (think of it as immutable class)
    type Student = { Courses: seq<Course> }

    // Original getAllCourses function
    let getAllCourses student =
        student.Courses

    // Original student
    let myStudent = 
        { Courses = ["CourseA"; "CourseB"] }

    // "Update" counterparty
    let updated = { myStudent with Courses = Seq.empty }

    let courses = getAllCourses updated

' Food for thought: If the `with` syntax copies a record, won't there be a lot of memory overhead?


***

### Command Query Separation

- From Wikipedia

> It states that every method should either be a command that performs an action, or a query that returns data to the caller, but not both. In other words, Asking a question should not change the answer. More formally, methods should return a value only if they are referentially transparent and hence possess no side effects

---

### Put it simply

`GetAllCourses` violates CQS because it is intended to be a query but it contains side effects

---
### Queries vs commands
- **Queries**
    - No side effects
    - Pure
        - Always returns the same result when given the same input parameters
    - Can be reordered by caller without affecting program flow
    - Always returns some value to the caller
- **Commands**
    - Side effects
    - Cannot be reordered by caller (because their side effects may be interdependent)
    - Represented by `void` methods in C♯

---
### Notice how the F♯ version already forces you to do CQS

- Command: *Updates* `student`
- Query: *Returns* the `courses` collection

***
### What is LINQ (-to-objects) really

- C♯ programmers should already be quite familiar with it
    - Similar to Java 8’s `Stream` API
- It is an *enhancement* to the `IEnumerable<T>` interface
- Functional programmers have it for ages and they have a big and scary term for it

> Sequence (aka IEnumerable, Stream) is a **monad**

---
### Big and scary explanation of monad

> All told, a monad in X is just a monoid in the category of endofunctors of X, with product × replaced by composition of endofunctors and unit set by the identity endofunctor.

---
### Overly simplified explanation of monad

> [Fluent APIs](https://www.wikiwand.com/en/Fluent_interface) which obey certain laws

---
### Common LINQ operations

- `Enumerable.Select` (aka `fmap`, `map`, `<$>`)
- `Enumerable.Where` (aka `filter`)

The type `IEnumerable<T>` (aka `Stream` in Java) is called a **functor** in FP, because it has a `fmap` function defined.

Here is the `Functor` typeclass in Haskell (analagous to C♯’s/Java’s `interface`)

    [lang=haskell]
    class Functor f where
        fmap :: (a -> b) -> f a -> f b

---
## Why higher order functions

---
### Imperative C♯ `Select`

    [lang=cs]
    var input = Enumerable.Range(1, 5).ToList();
    var inputLength = input.Count;
    var result = new List<int>(inputLength);
    for (int i = 0; i <= inputLength - 1; i++)
    {
        result.Add(input[i] * 2);
    }

    return result;

1. Unwrap a data structure (the `for` loop is extracting elements from the `List<int>`)
2. Perform some computation for each element (times two)
3. Wrap it back into the original data structure (`List<int>`)

---
### Imperative F♯ `Select`

    let input = ResizeArray [1..5]
    let length = input.Count
    let output = ResizeArray length
    for i = 0 to length - 1 do
        output.Add (input.[i] * 2)
    output

Same as above

---
### How can we update state without mutating?

First, let us redefine `list` in a functional way

---
### What is a (linked) `list`

    // Recursive data type
    type List<'a> =
        | Empty
        | Cons of head: 'a * tail: List<'a>

    // Empty list: []
    let empty = Empty

    // List with one element: [1]
    let list1 = Cons (1, Empty)

    // List with two elements: [2; 1]
    let list2 = Cons (2, Cons (1, Empty))

    // 'Add' a new item to list2 to build list3
    let list3 = Cons (3, list2) 

- No mutation methods
- Can only ‘update’ by building a new `list`

---
### Update state without assignments

    let input = [1 .. 5]
    
    // Non tail recursive
    let rec buildList list =
        match list with
        | [] -> []
        | h :: t -> h * 2 :: buildList t

    buildList input

--- 
### Tail recursion
    
    // Tail recursive
    let buildList list =
        let rec buildListImpl acc list =
            match list with
            | [] -> List.rev acc
            | h :: t -> buildListImpl (h * 2 :: acc) t
        buildListImpl [] list

    buildList input

There is a better way to avoid the expensive `List.rev` call in the end but requires the knowledge of continuation passing style (CPS)

---
### Less commonly known LINQ function

`Aggregate`, aka

- `Stream.reduce` (in Java)
- `reduce/fold/foldBack` (in F♯)
    
Fun fact: [Redux](https://github.com/reactjs/redux) a popular JavaScript framework, is actually `reduce` of the IO monad. Here we are dealing with the list monad instead.

---
### Imperative `Sum`

    [lang=cs]
    public static int Sum(this IEnumerable<int> elements)
    {
        var sum = 0;
        foreach (var e in elements)
        {
            sum += e;
        }

        return sum;
    }

---
### Notice how this involves four main steps

1. Initialise some kind of *state* (`var sum = 0`)
2. Unwrap a data structure (the `for` loop is extracting elements from the enumerable)
3. Update the *state* while looping over each element (`sum += e`)
4. Return the *state* (`return sum`)

Sounds familiar?

---
### `Aggregate` generalises this idea


    [lang=cs]
    public static TAccumulate Aggregate<TSource, TAccumulate>(
        this IEnumerable<TSource> source, 
        TAccumulate seed, 
        Func<TAccumulate, TSource, TAccumulate> func)

- aka `Seq.fold` in F♯

---
### Usage of `Aggregate`

    [lang=cs]
    var list = Enumerable.Range(1, 5).ToList();

    // Sum() is just a special case for Aggregate()
    var sum1 = list.Aggregate(0, (acc, curr) => acc + curr);
    var sum2 = list.Sum();

    Assert.Equals(sum1, sum2);

---
### Implementing `fold` and `sum` in F♯

    let rec fold folder acc list =
        match list with
        | [] -> acc
        | h :: t -> fold folder (folder acc h) t

    let sum list = fold (fun acc h -> acc + h) 0 list

Once you get hold of the concepts, it actually can be simplified to…

---
### Here is how they are typically defined

    let rec fold folder acc = function
        | [] -> acc
        | h :: t -> fold folder (folder acc h) t

    let sum = fold (+) 0

Powerful concepts in a few lines of code

---
### How powerful is `fold`?

    // Defined using List.fold
    let map f = List.fold (fun acc h -> f h :: acc) [] >> List.rev 

    // Defined using List.foldBack
    let map f list = List.foldBack (fun h acc -> f h :: acc) list []

***
### By the way

We have just gone through

- `map`
- `reduce`

which is the basis of [Google’s MapReduce](https://en.wikipedia.org/wiki/MapReduce) algorithm for big data processing :)

***
### Validation without exceptions

Suppose we want to validate `User`

    type User =
        { Name: string 
          Age: int }

---
### First attempt with exceptions

    [lang=cs]
    public void SaveUser(User user)
    {
        if (string.IsNullOrEmpty(user.Name))
        {
            throw new InvalidOperationException(
                "User name cannot be null or empty");
        }

        // Continue
    }

You are relying on a higher level to catch the exception

---
But very likely see calling code like

    [lang=cs]
    public void Foo(User user)
    {
        try
        {
            _userDatabase.SaveUser(user);
        }
        // or even worse: catch (Exception e)
        catch (InvalidOperationException e)
        {
            _logger.Error(e);
            // No rethrow?!
        }
    }

Is the exception really handled?

---
### If the exception is rethrown
![Who watches the watchmen](http://www.opinionatedbastard.ca/wp-content/uploads/2016/02/WhoWatchesTheWatchmen.jpg)

---
### If the exception is not rethrown

How is the caller of `Foo` aware that the save failed?

---
### Learn from the wise

![Do or do not, there is no try](http://static1.squarespace.com/static/54f26c32e4b09389aa381106/55358fdbe4b02eb715d0ff52/55b66544e4b038ee5edb329c/1438090911304/?format=1000w)

---
### Some help from the type system

    type Result<'a> =
        | Success of value: 'a
        | Error of errors: string list

    let either success error = function
        | Success x -> success x
        | Error e -> error e

    let bind f = either f Error

    let map f = bind (f >> Success)

I will not go too deep on how and why for this presentation

---
### Our `Result` type is a functor

Why?

---
### Some interesting custom operators

    let (>=>) f g = f >> bind g

    let (&&&) f g x =
        match f x, g x with
        | Error e1, Error e2 -> Error (e1 @ e2)
        | Error e, _ | _, Error e -> Error e
        | Success s1, Success s2 -> Success s2

---
### Our sample use cases of validators

    let validateName user =
        if String.length user.Name = 0
        then Error ["Name cannot be an empty string"]
        else Success user

    let validateAge user =
        if user.Age < 18
        then Error ["User must be older than 18 years old"]
        else Success user

    // This validator takes an extra duplicateChecker,
    // which can come from databases, files, in-memory
    let validateDuplicateName duplicateChecker user =
        if duplicateChecker user.Name
        then Error ["Duplicate user name"]
        else Success user

    // This will be used as our in-memory duplicateChecker
    let dummyDuplicateChecker = (=) "Foo"

---
    let sequentialValidatePipeline =
        validateName
        >=> validateAge
        >=> validateDuplicateName dummyDuplicateChecker


    let parallelValidatePipeline = 
        validateName 
        &&& validateAge
        &&& validateDuplicateName dummyDuplicateChecker

    let testUser = { Name = ""; Age = 11 }

    sequentialValidatePipeline testUser
    parallelValidatePipeline testUser

Notice how the pipeline is constructed with an expressive syntax

***

### Dependency injection

    // C# equivalent will be an ILogger interface
    type Logger = string -> unit

    // C# equivalent will be the generic delegate Action
    type SaveAction = unit -> unit

    let saveToDb (infoLogger: Logger) (saveAction: SaveAction) () =
        infoLogger "Saving to database"
        saveAction ()
        infoLogger "Save complete"

    // Create fake for unit testing
    let fakeSaveToDb = saveToDb ignore ignore

    // Real production code
    let efSaveToDb = saveToDb Log4NetLogger DbContext.Save

***

## Q & A

***

## Reference Links

- [F♯ For Fun and Profit](http://fsharpforfunandprofit.com/)
    - Especially the [Why use F♯ series](http://fsharpforfunandprofit.com/series/why-use-fsharp.html)
- [Mark Seemann’s blog](http://blog.ploeh.dk/)
    - One most recent blog entry: [Better domain models with F♯’s types](http://blog.ploeh.dk/2016/11/28/easy-domain-modelling-with-types/)

***

## Thank you

By the way, this presentation is also written using an F♯ library [FsReveal](https://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&cad=rja&uact=8&ved=0ahUKEwjXgeuv_9HQAhXHopQKHXO2Cv8QFgggMAE&url=https%3A%2F%2Ffsprojects.github.io%2FFsReveal%2F&usg=AFQjCNFC0ocv4cDfE2P-9lcnCJbkVctp_w&sig2=ncze0oWlYG-BpfFzDOsmIg). It integrates Markdown, C♯, and F♯ to the [reveal.js](https://github.com/hakimel/reveal.js/) web presentation framework.
