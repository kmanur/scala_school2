# Pattern Matching and For-Expressions

Control flow on steroids, transforming collections with style.

# `match` expressions and literal patterns

Old and busted branching:

    def monthName(n: Int): String =
      if (n == 1) "January"
      else if (n == 2) "February"
      else if (n == 3) "March"
      // ...
      else "Unknown"

New hotness branching:

    def monthName(n: Int): String =
      n match {
        case 1 => "January"
        case 2 => "February"
        case 3 => "March"
        // ...
      }

Wait, this looks suspiciously like `switch-case` from C. But notice that there's no `break` statement needed, to prevent falling through to subsequent cases (sorry, no [Duff's device](http://en.wikipedia.org/wiki/Duff's_device) in Scala). If you want multiple cases to correspond to a single output, there's a very convenient syntax:

    def daysInMonth(n: Int): Int =
      n match {
        case 1 | 3 | 5 | 7 | 8 | 10 | 12 => 31
        case 4 | 6 | 9 | 11 => 30
        case 2 => 28
      }

The values appearing on the left hand side of the `=>` are called "literal patterns."

> #### Exercise: type inference
> What if the expressions on the right hand side of the `=>` are of different types?

# Variable and typed patterns

Pattern matching is more powerful than `switch-case`, because it can introduce new `val` bindings. First, a simple example: what happens if you pass an unmatched value to `monthName` or `daysInMonth`?

    monthName(13)
    daysInMonth(13)

> #### Exercise: fix it
> Use a variable or wildcard to provide a "default" case.

You can also combine this with type ascriptions to provide safe down-casting:

    def whatIs(a: Any): String =
      a match {
        case n: Int =>
          val evenOrOdd = if (n % 2 == 0) "even" else "odd"
          "the %s number %d".format(evenOrOdd, n)
        case s: String =>
          val englishOrNot = if (s forall { '\u0020' to '\u007F' contains _ }) "probably english" else ""
          "the %s string %s".format(englishOrNot, s)
        case _ =>
          "something else"
      }

This can be written differently with _pattern guards_ (which sort of look like `if-else` expressions, but aren't):

    def whatIs(a: Any): String =
      a match {
        case n: Int if n % 2 == 0 =>
          "the even number " + n
        case n: Int =>
          "the odd number " + n
        case s: String if s forall { '\u0020' to '\u007F' contains _ } =>
          "the probably english string " + s
        case s: String =>
          "the string " + s
        case _ =>
          "something else"
      }

Notice that the first pattern to match, in order from top to bottom, wins.

# Extractor patterns

Pattern matching is most compelling when used to _extract_ features from various data structures. For example:

    def whatIs(a: Any): String =
      a match {
        case (x, y, z) =>
          "a Tuple3 containing %s, %s and %s".format(x, y, z)
        case List(x, y, z, _*) =>
          "a List containing three or more elements, starting with: %s, %s and %s".format(x, y, z)
        case List((x1, y1), (x2, y2)) =>
          "a List containing exactly two Tuple2s: %s -> %s, %s -> %s".format(x1, y1, x2, y2)
        case _ =>
          "something else"
      }

> #### Exercise: moar!
> This example composes two different kinds of patterns. Identify them, and expand the example to include literal and typed patterns as well.

# Exception handling

This code probably behaves badly:

    import util.Random._
    val s = if (nextBoolean()) "42" else "i like pretty flowers"
    val n = s.toInt

An _exception_ unwinds the call stack, from the frame in which it's thrown to the nearest enclosing `catch` block. When you anticipate the possibility of one or more kinds of exceptions (for example, because you're validating possibly malicious user input, or because you're trying to connect to a possibly unavailable server), you can pattern match against those exceptions:

    def defensiveToInt(s: String): Int =
      try {
        s.toInt
      } catch {
        case _: NumberFormatException => 0
      }

Note that, just like all our other control flow so far, `try-catch` is an expression that produces a value and runtime and has a type at compile time. The static type is the LUB of the types of the `try` block and all of the `case` clauses.

> #### Exercise: verbose mode
> Print the stack trace when an exception is caught, and catch more kinds of exceptions (e.g. `NullPointerException`). Note: just as in Java, `Throwable` is the mother of all exception types.

---

> #### Combo Bonus: `finally`
> It's often important to close resources, such as input/output streams, database connections and sockets. What does this look like, and how does it affect the static type of the corresponding `try-catch`?

---

> #### Mega Combo Bonus: `Option`
> If `defensiveToInt` produces the value `0`, we don't know whether it was because the input was the string `"0"` or a non-number. Scala has a data type called `Option` which disambiguates; use it.

# `for`-loops

We've all written this code in some language at least once:

    for (i <- 0 until 10)
      println("I'M AWESOME")

Fantastic. Now let's say you're trying to do something slightly more useful:

    import collection.mutable.Buffer

    val quietWords = List("let's", "transform", "some", "collections")
    val noisyWords = Buffer.empty[String]
    for (i <- 0 until quietWords.size)
      noisyWords += quietWords(i).toUpperCase

There are a few truly awful things about this code:

* Explicitly indexing into the list is prone to bugs; you're likely to have an off-by-one error or accidentally use the wrong index variable in the wrong collection.
* The time complexity of this algorithm is O(n<sup>2</sup>), because indexing into a `List` is a linear-time operation, not constant-time (note: if we used a more appropriate collection type, such as `Vector`, this wouldn't be an issue).
* It's not nearly DRY enough: `noisyWords` appears two times and `quietWords` appears three times.

So we can do better:

    val noisyWords = Buffer.empty[String]
    for (word <- quietWords)
      noisyWords += word.toUpperCase

No more explicit indexing, and we're now linear time...

# `for`-expressions

The mutable `Buffer`-based approach is still dissatisfying. We've been doing very well treating control flow structures as value-producing expressions so far, so why not here?

    val noisyWords = for (word <- quietWords) yield
      word.toUpperCase

Aha, immutable and DRY. The `for-yield` expression indicates _transformation_: it directly produces a new collection, where each element is transformed from a corresponding element in the original collection. The behavior varies depending on the type of the original collection, but in this case where you start with a `List[String]`, the `for-yield` expression produces a `List[String]` in the order you'd expect. In many other programming languages, this is referred to as "list comprehensions."

> #### Note for Java refugees
> The `yield` keyword has nothing to do with Java's `Thread.yield()` method. In fact, since `yield` is a reserved keyword in Scala, if you want to call that method, you have to write ``Thread.`yield`()`` instead. Surrounding the keyword in backticks forces it to be parsed as an identifier.

One nice thing about `for`-loops and `for`-expressions is that they can contain multiple _generators_, producing a similar effect to nested loops:

    val salutations = for (hello <- List("hello", "greetings"); world <- List("world", "interwebs")) yield
      "%s %s!".format(hello, world)

This syntax starts to get a bit clunky, the more generators you have, so Scala provides an alternative syntax:

    val salutations = for {
      hello <- List("hello", "greetings")
      world <- List("world", "interwebs")
    } yield "%s %s!".format(hello, world)

This is a bit confusing-looking at first, since the braces look like they're delineating a block of statements. It's not difficult to get used to, though, and it's the recommended style when you have multiple generators.

You can also directly assign `val`s inside the `for { ... }`:

    val salutations = for {
      hello <- List("hello", "greetings")
      world <- List("world", "interwebs")
      salutation = "%s %s!".format(hello, world)
    } yield salutation

... which isn't terribly useful by itself, but becomes very handy when you need to _filter_ some of the results:

    val salutations = for {
      hello <- List("hello", "greetings")
      world <- List("world", "interwebs")
      salutation = "%s %s!".format(hello, world)
      if salutation.length < 20 // tl;dr
    } yield salutation


# Patterns run amok!

TODO