---
layout: post
title: "Intro to Test-Driven Development in Golang"
permalink: /resources/tutorials/fizzbuzz-go
---

To complete this tutorial, you will need to have the following installed:
- Go version >= 1.14
- Git
- A text editor
- A terminal or command prompt (we will be working mainly from the terminal, if
	you are not comfortable using yours, you may want to complete a [Command Line Crash Course](https://learnpythonthehardway.org/python3/appendixa.html)
	before you continue).

Think you are missing something? [Check/Install here](https://github.com/fouralarmfire/square-one/blob/master/tutorials/tdd-setup.md).

## Task:
Using [TDD](https://www.agilealliance.org/glossary/tdd), write a program which
will help you cheat at the drinking game fizzbuzz.

#### Rules:
- When the given number is divisible by 3, say fizz
- When the given number is divisible by 5, say buzz
- When the given number is divisible by 3 and 5, say fizzbuzz!
- When the given number does not fit any of the other rules, print the number

#### What we will be building:
```sh
$ go build .
# > creates executable binary/program `fizzbuzz`

$ ./fizzbuzz 3
fizz

$ ./fizzbuzz 5
buzz

$ ./fizzbuzz 15
fizzbuzz!

$ ./fizzbuzz 7
7 :(
```

## Part 1: Project setup

**Steps**:

1. Open Terminal (or iTerm or whatever else you like to get a command prompt) and create a new directory.
    Then change into that directory and initialise a new git repository _(New to Git? See [this guide](/resources/tutorials/git).)_:

	```sh
	cd ~
	mkdir -p workspace/fizzbuzz
	cd workspace/fizzbuzz
	git init
	git remote add origin <URL OF YOUR REPO ONLINE>
	echo "tdd exercise in go" > README.md
	git add README.md
	git commit -m "readme.md"
	git push -u origin master
	```

1. Initialize your Go project. This is how we store information about which version of Go we are using,
	and which dependencies our project requires. 'Dependencies' are
	packages (exported and publicly available) written by others which we can use to help build our code.
	We use `go mod` to manage them:

	```sh
	go mod init github.com/<YOUR USERNAME>/<YOUR PROJECT NAME>
	# eg: go mod init github.com/callisto13/fizzbuzz
	```

	You should end up with a file called `go.mod` in your repo.

1. Install `ginkgo` and `gomega`. [Ginkgo](http://onsi.github.io/ginkgo/) is the testing framework we are going to use to write
[unit tests](https://code.tutsplus.com/articles/the-beginners-guide-to-unit-testing-what-is-unit-testing--wp-25728) for our code.
Go does have a built-in testing framework, but it is not very user friendly and encourages a style of test-writing which
is hard to debug. **If testing is too hard then people wont do it, which is bad**.
Ginkgo and Gomega are Go packages so we need to install them using these two `go get` commands.

	```sh
	go get github.com/onsi/ginkgo/ginkgo
	go get github.com/onsi/gomega/...
	```

1. You should notice that your `go.mod` file has changed and you now also have a `go.sum` file in your repo.

1. Check that everything is installed properly by running `ginkgo`.
	This will fail with something like `Found no test suites` and some help on how to run but that is fine. Let's get testing!

Commit the setup and push to github:

```sh
git add go.sum go.mod
git commit -m "setup"
git push
```

## Part 2: Our first test

Test Driven Development follows a simple pattern: `red -> green -> refactor`.
In reality, this works as follows:
1. Write a test which would pass if the code was implemented correctly.
1. See it fail.
1. Write _just_ enough code to make the test pass.
1. See the test pass.
1. Look over the code and think of ways to improve what you have. Is there any
repetition? Can an algorithm be made more efficient?

Writing code this way has 4 benefits: 1) by only writing
what you need to achieve basic tasks, you end up writing less code, all of which
is used (no ghost functions which you have little memory of); 2) every single
function is tested, which makes adding more or changing bits a breeze as you will
find out immediately what you may have broken; 3) nicely structured tests makes
it easy for others looking at your project to figure out what your code is
supposed to do (a good way to get contributors); and 4) because of 1 and 2, you have
complete confidence that your code does what it is supposed to do.

![alt text](http://turnoff.us/image/en/tdd-vs-ftt.png)
_[turnoff.us](http://turnoff.us/geek/tdd-vs-ftt/)_

So lets get going with our first test:

**Steps**:

1. Ginkgo gives us commands which will generate some boilerplate testing 
    code. Run both in your terminal:

    ```sh
    ginkgo bootstrap
    ginkgo generate fizzbuzz
    ```

    This should generate 2 new files in your repo: `fizzbuzz_suite_test.go` and
    `fizzbuzz_test.go`.

    Let's have a look at those files. `fizzbuzz_suite_test.go` should look
    something like this:

    ```go
    package fizzbuzz_test

    import (
	    "testing"

	    . "github.com/onsi/ginkgo"
	    . "github.com/onsi/gomega"
    )

    func TestFizzbuzz(t *testing.T) {
	    RegisterFailHandler(Fail)
	    RunSpecs(t, "Fizzbuzz Suite")
    }
    ```

    You don't need too worry to much about what is in this file, since we won't be
    touching it again. Let's quickly go over the highlights.

    The first line tells Go that this file is part of the `fizzbuzz_test` "package".
    A package in Go is like a module: a unit of related code. You can have as many
    files as you like in the same package, all adding code to the same module. (This
    doesn't mean you _should_ throw everything into the same package: the code is supposed
    to be _related_. If it is not related, it should go into a different package.)

    The next section (`import (...)`) is where we have specified the other packages that
    this file need to run: in other words the dependencies for the code in this file.
    In this case we have `"testing"`, which is Go's built-in testing framework, and
    of course `ginkgo` and `gomega`. The latter two need to be imported with their full
    Github module path, since they are not built into the language. `testing` is
    which means we need to simply name it.

    Next we have a Function which is using all three dependencies above to run our test
    suite.

    If we now look at `fizzbuzz_test.go`, you should see something like:

    ```go
    package fizzbuzz_test

    import (
	    . "github.com/onsi/ginkgo"
	    . "github.com/onsi/gomega"

	    . "github.com/<YOUR USERNAME>/fizzbuzz"
    )

    var _ = Describe("Fizzbuzz", func() {

    })
    ```

    You should recognise the first line (this file is part of the `fizzbuzz_go_test` package)
    and the `import` section. We are still depending on `ginkgo` and `gomega`, but
    the generator has also anticipated that we will want to import our own package, since
    that is the code we are going to be testing.

    Below that is a `Describe` func, in which we will write our first test in a moment.

    We have to make a small edit to this file before we begin: on the line where your
    package is imported, remove the `.` at the beginning.

    ```go
	  . "github.com/<YOUR USERNAME>/fizzbuzz"
	  // ^ becomes v, observe the absence of '.'
	    "github.com/<YOUR USERNAME>/fizzbuzz"
    ```

    When we use functions from a package in Go, we do so like this `packagename.FunctionName()`.
    "Dot" imports let you call functions _without_ the package name. We are doing this with
    Ginkgo and Gomega: `Describe("...")` could be `ginkgo.Describe("...")` if we removed the `.`
    from the start of the line importing it. In the case of those two packages we will keep
    the "dot" imports, as repeating `ginkgo.` and `gomega.` all over our tests will get messy.
    For our eventual code package, however, we want to be clear that we are using it in our tests.

1. Now that we are using some dependencies in our own packages, it is time to save them in a
    folder called `vendor`. This means that we can clone down a new copy of this project anywhere
    and all the dependencies we need to run it will be included.

    ```sh
    go mod vendor
    ```

    This will create a new directory.

    You should re-run this command whenever you add a new dependency to any project.

1. Let's write our first test.
	Open the `fizzbuzz_test.go` file in your text editor, and alter the `Describe` function,
	adding the following between the curly braces:

	```go
	var _ = Describe("Fizzbuzz", func() {
	  Context("knows when a number", func() {
	    It("is divisible by 3", func() {

	    })
	  })
	})
	```

	So what have we done here?

	You'll notice that we haven't just written `It("works")`. TDD is about ensuring
	that you write your code incrementally by breaking down your problem into small
	chunks and tackling each problem one at a time. Right now we are writing a pretty simple
	game, so you may be thinking TDD is overkill, but when it comes to a large project
	or a very complex problem which you can't possibly envisage how it will turn out,
	TDD is a very useful discipline to learn.

	So let's read our test.

	The first line is to indicate what you are testing. Ginkgo recognises
	everything within this `Describe` block as the scope of what you want to test.
	The bit in the quotes is for your benefit: we are testing "Fizzbuzz".
	On the next line we define a `Context`, which does exactly what it sounds like:
	it gives context to the test, for humans and the compiler.
	Ginkgo understands that tests under the
	same `Context` are grouped together and share the same state. The `It` block will
	contain your actual test (right now there is nothing being tested).
	Likewise the bit in quotes are for your benefit.

1. Now we have to state what we expect to happen when a function in our code runs.
	This is the hardest part of TDD: you have to test code which does not yet
	exist. But you do know the _behaviour_ you want from it.
	We have not written a function yet, but we have explained what we want
	in our test (the `It` quotes), so we know vaguely what it should look like. Put the following line
	inside the curly braces of your `It` block:

	```go
	  It("is divisible by 3", func() {
	    Expect(fizzbuzz.IsDivisibleByThree(3)).To(BeTrue())
	  })
	```

1. Run the test: `ginkgo`
	This should fail with `go build github.com/<username>/fizzbuzz: no non-test Go files in <filepath>`.
	Which makes sense; we haven't written any non-test code yet. Time to write some.

[How your project should look](https://github.com/Callisto13/fizzbuzz-go/tree/fecabe5f7acdf09aba5a5cbf8f2223f8470e4058) at this stage.

## Part 3: Make it green

Now that we have our first failing test we are going to follow the errors given
by Ginkgo to make it pass.

So let's start with the first error we were given: `no non-test Go files`.

**Steps**:

1. Create and open a new file called `fizzbuzz.go`.

1. Add the package name to the first line of the file:

    ```go
    package fizzbuzz
    ```

    This is not a test file, so we don't need to end the package name with the `_test` identifier.

1. If we run the test again (`ginkgo`), we see that we have moved on to the next error:

    ```sh
    Failed to compile fizzbuzz:

    # github.com/<username>/fizzbuzz_test [github.com/<username>/fizzbuzz.test]
    ./fizzbuzz_test.go:12:10: undefined: fizzbuzz.IsDivisibleByThree

    Ginkgo ran 1 suite in 2.140132121s
    Test Suite Failed
    ```

1. In `fizzbuzz.go` define that function which Ginkgo was complaining about. Don't make it do anything,
	just define it:

	```go
	func IsDivisibleByThree(number int) bool {
	  return false
	}
	```

	Here we have defined the function we we promised the test would exist. It takes
	on argument (in the test this is `3`) which is a type of `int` (integer/number) and
	we give it a name `number` so that it can be used within the function.

	The return value is a `bool` meaning that the output of this function will be either
	`true` or `false`.

	Lastly we are simply returning `false` as a default, since Go requires that if your
	function says something will be returned, something actually is.

1. Save the files and run the test again: `ginkgo`.
	Now we should see something more like a test:

	```sh
	Running Suite: Fizzbuzz Suite
	=============================
	Random Seed: 1586604740
	Will run 1 of 1 specs

	• Failure [0.001 seconds]
	FizzbuzzGo
	/.../fizzbuzz/fizzbuzz_test.go:10
	  knows when a number is divisible by 3 [It]
	  /.../fizzbuzz/fizzbuzz_test.go:11

	  Expected
	      <bool>: false
	  to be true

	  /.../fizzbuzz/fizzbuzz_test.go:12
	------------------------------


	Summarizing 1 Failure:

	[Fail] FizzbuzzGo [It] knows when a number is divisible by 3
	/.../fizzbuzz/fizzbuzz_test.go:12

	Ran 1 of 1 Specs in 0.001 seconds
	FAIL! -- 0 Passed | 1 Failed | 0 Pending | 0 Skipped
	--- FAIL: TestFizzbuzz (0.00s)
	FAIL

	Ginkgo ran 1 suite in 1.513106767s
	Test Suite Failed
	```

	Excellent! We have told our test that when function `IsDivisibleByThree` receives
	the number `3`, the answer should be `true`.
	So lets go give it what it wants.

1. Go back to `fizzbuzz.go` and make `IsDivisibleByThree` return `true`

	```go
	func IsDivisibleByThree(number int) bool {
	  return true
	}
	```

1. Run the test again. And we're green! Congrats, you have passed your first test.
	Let's go wreck it.

[How your project should look](https://github.com/Callisto13/fizzbuzz-go/tree/866eca2556734c6e5a1cf66fa8c4b13d6da5dc7d) at this stage.

## Part 4: Make it red

Obviously we are not done yet. Hardcoding `true` like that is seen as a Very
Bad Thing, and also not very useful for our game. So let's write another test
to force ourselves to do the right thing.

**Steps**:

1. In `fizzbuzz_test.go` add a second `It` block under the first. (Make sure
	you stay in the same `Context` block.)

	```go
	  It("is NOT divisible by 3", func() {
	    Expect(fizzbuzz.IsDivisibleByThree(1)).To(BeFalse())
	  })
	```

1. Run the tests again. Back to red? `1 Passed, 1 Failed`? `Expected: true to be false`?
	Excellent. Time for some maths.

1. Back in `fizzbuzz.go` we need to make our function work out whether the
	`num` it is being passed as an argument (which right now we are ignoring) is
	_actually_ divisible by three. To do that we need to use modular arithmetic: if
	`num` can be evenly divided by 3, it should return 0 (i.e. not have a remainder).

	```go
	func IsDivisibleByThree(number int) bool {
	  if num%3 == 0 {
	    return true
	  }
	  return false
	}
	```

1. Now when we run the tests, both should pass.
	We don't have much code right now, but there is already some refactoring to be done.
	There is a way to write that bit of `if` and `return` logic on just one line rather than 4.
	If you can figure it out, update your function to be less verbose. (Don't forget to make
	sure your tests still pass after you change things!)


We added a bunch of new stuff, so let's commit in stages to keep things tidy.

First let's commit the `vendor` dir and module files:

```sh
git add go.mod go.sum vendor
git commit -m "vendor dependencies"
```

And then our code and tests:

```sh
git add fizzbuzz.go fizzbuzz_test.go fizzbuzz_suite_test.go
git commit -m "knows when a number is divisible by 3"
git push
```

[How your project should look](https://github.com/Callisto13/fizzbuzz-go/tree/1d0428bfed5d6ef55b8a14629570f4ae58de9889) at this stage.

## Part 5: Around we go again...

**Steps**:

1. In `fizzbuzz_test.go`, add another `It` block (again still within the same `Context` block)
	to test whether numbers are divisible by 5:

	```go
	  It("is divisible by 5", func() {
	    Expect(fizzbuzz.IsDivisibleByFive(5)).To(BeTrue())
	  })
	```

1. Run the tests. Do you see `Failed to compile fizzbuzz-go:... undefined: fizzbuzz.IsDivisibleByFive`?

1. Go define `IsDivisibleByFive` in `fizzbuzz.go`. (just define and set a default, don't make it
	do anything.)

1. Run the tests. `Expected: false to be true`? Make your new function return `true`.

1. Run the tests. Green again?	Go back to your test file and write the opposing
  `It('is NOT divisible by 5'` block.

1. Run the tests. `Expected: true to be false`? Fix your code with modular arithmetic to make it pass.

Once you have all 4 tests passing, commit your work and push to github:

```sh
git add fizzbuzz.go fizzbuzz_test.go
git commit -m "knows if numbers are divisible by 5"
git push
```

2 functions in and we are starting to see a pattern here, but let's leave off
refactoring just a little while longer and move onto the last calculation.
By now you should know the routine, so go ahead and write 2 more tests for a function
which knows if a number `IsDivisibleByThreeAndFive`. (Hint: you can use just one number to do this division.)

Once you have all 6 tests passing, commit your work and push to github:

```sh
git add fizzbuzz.go fizzbuzz_test.go
git commit -m "knows if numbers are divisible by 3 and 5"
git push
```

[How your project should look](https://github.com/Callisto13/fizzbuzz-go/tree/7dc7459fc2d3e116e019688d1d2bc384a73a77a4) at this stage.

## Part 6: First Refactor

Right now we have three functions which are doing more or less the same thing.
Let's see if we can [DRY](http://programmer.97things.oreilly.com/wiki/index.php/Don't_Repeat_Yourself) this out a bit.

1. In `fizzbuzz.go`, edit your code so that the 3 `IsDivisibleBy*` functions
		are replaced by just 1 function.

	```go
	func IsDivisibleBy(number, divisor int) bool {
	  return number%divisor == 0
	}
	```

1. Run your tests. Are they very unhappy? Update them to use the new function.
  If you are having trouble making them pass, remember to read the failure messages
  carefully: Ginkgo is helpful and will generally point you in the right direction.

Once you are all green again, commit your work and push to github:

```sh
git add fizzbuzz.go fizzbuzz_test.go
git commit -m "first refactor"
git push
```

[How your project should look](https://github.com/Callisto13/fizzbuzz-go/tree/673d0b6b2a93cb367e8d226cb3e3fc3800963649) at this stage.

## Part 7: FizzBuzz says

So now our code can tell us whether a number is divisible by 3, 5 or 3 and 5, but we
can't really play the game with this.

**Steps**:

1. In `fizzbuzz_test.go` define a new `Context` block underneath the end
  of the old one, still within the same `Describe` block.

	```go
	Context("when playing the game, Fizzbuzz says...", func() {
	  It("`fizz` when a number is divisible by 3", func() {
	  })
	})
	```

1. Next, put what you would expect to happen inside your `It` block:

	```go
	Context("when playing the game, Fizzbuzz says...", func() {
	  It("`fizz` when a number is divisible by 3", func() {
	    Expect(fizzbuzz.Says(3)).To(Equal("fizz"))
	  })
	})
	```

1. Run your tests. They should fail in a way which by now should be familiar.

1. Go into `fizzbuzz.go` and create that function. (just define, and return a default empty string `""`.)

1. Run the tests again and follow the error message, remember to do just enough
to make them pass:

	```go
	func Says(number int) string {
	  return "fizz"
	}
	```

1. Add the next test within that `Context` to force yourself to write code which actually evaluates something.

	```go
	  It("`buzz` when a number is divisible by 5", func() {
	    Expect(fizzbuzz.Says(5)).To(Equal("buzz"))
	  })
	```

1. See the test fail, then go back to your code and make your new function process
the argument it is passed by using our `IsDivisibleBy` function:

	```go
	func Says(number int) string {
	  switch {
	  case IsDivisibleBy(number, 3):
	    return "fizz"
	  case IsDivisibleBy(number, 5):
	    return "buzz"
	  }

	  return ""
	}
	```

1. Go back to `fizzbuzz_test.go`, and add the test which check that the
  program says "fizzbuzz" when a number is divisible by 3 and 5.

1. Run the tests, watch it fail.

1. Go to your code and make it pass.

	_note: remember to watch your ordering here. Since you are returning immediately_
	_when the number is first sucessfully divisible, you may end up saying "fizz" rather_
	_than "fizzbuzz". Make sure you check if a number can be divisible by 3 and 5 first._
	_Switch the order of your code to see your tests failing in this way._

1. The last thing we need to do is write a test (and then the code) for when the
  number is not divisible by 3, 5 or 3 and 5. It should just return the number.

Once you have all 10 tests passing, commit your work and push to github:

```sh
git add fizzbuzz.go fizzbuzz_test.go
git commit -m "returns fizz, buzz and fizzbuzz"
git push
```

[How your project should look](https://github.com/Callisto13/fizzbuzz-go/tree/487ac61ebe82c04beff80bc193a2cdc64542e832) at this stage.

&nbsp;

-----------------------------------------------

&nbsp;

# Bonus round! _Trickier stuff ahead._

All our calculations are done and we are ready to play the game... but it doesn't
quite work in the way we planned at the start. We can't do `$ ./fizzbuzz 3`
and expect to see `fizz` in the terminal right now.

So, for bonus points, we are going to write some integration tests. Up to now we have
been testing out each individual `unit` of code in... unit tests (obvs). Now we
need to verify that our code integrates with the tools which are not directly part of that code
but are going to interact with it (in this case the command line).

## Part 8: Building an executable

**Steps:**
1. First let's do some tidying up. Create a new directory in your project and move your
    existing Go files in there:

    ```sh
    mkdir -p pkg/fizzbuzz
    mv fizzbuzz.go fizzbuzz_test.go fizzbuzz_suite_test.go pkg/fizzbuzz
    ```

    This will break your tests (remember those `import` paths?) so open up
    `pkg/fizzbuzz/fizzbuzz_test.go` to update the import to the new location:

    ```go
    "github.com/<username>/fizzbuzz/pkg/fizzbuzz"
    ```

    Check that everything is back to normal by running the tests. This time you
    will need to change the test command to `ginkgo -r` the `-r` means recursive,
    as in "look for all test files in all directories from here".

1. Once you have verified that your old tests still run, we can set up the new ones.
    Create a new directory in your project:

    ```sh
    mkdir integration
    ```

    Your project layout should now look like this:

    ```sh
    .
    ├── go.mod
    ├── go.sum
    ├── integration
    ├── pkg
    │   └── fizzbuzz
    │       ├── fizzbuzz.go
    │       ├── fizzbuzz_suite_test.go
    │       └── fizzbuzz_test.go
    └── vendor
    ```

    Change into the new `integration` directory.

    ```sh
    cd integration
    ```

    And run the Ginkgo generation commands:

    ```sh
    ginkgo bootstrap
    ginkgo generate integration
    ```

    Like before, this will create 2 new test files: `integration_test.go` and
    `integration_suite_test.go`.

    Move back out of this new directory when you are done generating the files.

    ```sh
    cd ..
    ```

1. Open `integration_test.go` in your text editor.

    In these tests we need to compile the `fizzbuzz` binary, execute it, and
    verify that the correct answer is printed out.

    Fortunately, Ginkgo can help us with that.

    Remove everything currently in `integration_test.go` and replace it with the following.
    There is a lot going on here so I have added comments over the key bits.

    ```go
    package integration_test

    import (
	    // the package which will let us execute the binary from here
	    "os/exec"

	    . "github.com/onsi/ginkgo"
	    . "github.com/onsi/gomega"

	    // the packages that we need to build the binary and check output
	    // in the test
	    "github.com/onsi/gomega/gexec"
	    "github.com/onsi/gomega/gbytes"
    )

    var _ = Describe("Integration", func() {
	    // these are declared here so they can be used throughout the test file
	    var (
	      fizzbuzzBinary  string
	      fizzbuzzCommand *exec.Cmd
	    )

	    // a helper function to set things up before the actual tests run
	    BeforeEach(func() {
		    var err error

		    // compile the fizzbuzz binary
		    // swap callisto13 for your username. this string should
		    // match the module name in your go.mod
		    fizzbuzzBinary, err = gexec.Build("github.com/callisto13/fizzbuzz", "-mod=vendor")
		    // verify that compilation did not fail
		    Expect(err).NotTo(HaveOccurred())
	    })

	    // a helper function to delete the compiled binary after tests have run
	    AfterEach(func() {
		    gexec.CleanupBuildArtifacts()
	    })

	    It("when the command line argument is 3, it prints 'fizz'", func() {
		    // get the command ready, passing in 3 as an argument
		    fizzbuzzCommand = exec.Command(fizzbuzzBinary, "3")

		    // run the command
		    session, err := gexec.Start(fizzbuzzCommand, GinkgoWriter, GinkgoWriter)
		    // verify that it did not fail
		    Expect(err).NotTo(HaveOccurred())
		    // verify that it has printed the right thing to the terminal (stdout)
		    Eventually(session.Out).Should(gbytes.Say("fizz\n"))
	    })
    })
    ```

1. After you have copied this, you can run the test: `ginkgo -r` (or `ginkgo -r integration`
    if you only want to run the new tests).

    They should fail with something like `cannot load <package>: no Go source files`.
    This is because we are missing a `main.go` file: the entry point to our binary.

    At the base of your project, create a `main.go` file. Your layout should now look like this:

    ```sh
    .
    ├── go.mod
    ├── go.sum
    ├── integration
    │   ├── integration_suite_test.go
    │   └── integration_test.go
    ├── main.go
    ├── pkg
    │   └── fizzbuzz
    │       ├── fizzbuzz.go
    │       ├── fizzbuzz_suite_test.go
    │       └── fizzbuzz_test.go
    └── vendor
    ```

    Open it up and the following into your `main.go`:

    ```go
    package main

    func main() {

    }
    ```

    Now when you run the test again, it should complain that it is stuck waiting for "fizz".

1. This is easy to fix, update your `main` func to print the string the test expects:

    ```go
    package main

    import "fmt" // don't forget to import new packages

    func main() {
      fmt.Println("fizz")
    }
    ```

    The test passes, and although the binary is still useless, it's a good time to commit what we have so far.

  ```sh
  git add pkg/ integration/ main.go
  # you need to git add the moved files so that git picks up the change
  git add fizzbuzz.go fizzbuzz_test.go fizzbuzz_suite_test.go
  git commit -m "integration test setup and restructuring"
  git push
  ```

[How your project should look](https://github.com/Callisto13/fizzbuzz-go/tree/73543a11e438c9bef3080f77bb35ac8371c79201) at this stage.

## Part 9: Command line args

1. We have hardcoded the output in main, which is fine for our current test.
    Let's add another `It` under the last to force us to use the `fizzbuzz` package we wrote earlier:

    ```go
    It("when the command line argument is 5, it prints 'buzz'", func() {
      fizzbuzzCommand = exec.Command(fizzbuzzBinary, "5")
      session, err := gexec.Start(fizzbuzzCommand, GinkgoWriter, GinkgoWriter)
      Expect(err).NotTo(HaveOccurred())
      Eventually(session.Out).Should(gbytes.Say("buzz\n"))
    })
    ```

    This test should fail, as our binary only prints "fizz". Let's make our main
    recognise command line args, and use our code.

1.  Open `main.go` and update our main func with the following:

    ```go
    package main

    import (
      "fmt"
      "os"
      "strconv"

      "github.com/callisto13/fizzbuzz/pkg/fizzbuzz"
    )

    func main() {
      // get the first command line argument
      numberAsString := os.Args[1]

      // the argument comes in as a string, so we need to turn it into an integer
      number, _ := strconv.Atoi(numberAsString)

      // call our package and print the result to stdout
      fmt.Println(fizzbuzz.Says(number))
    }
    ```

    Now both our integration tests pass: our program can be built and executed
    with args from the command line!

  ```sh
  git add integration/integration_test.go main.go
  git commit -m "can be run from the command line"
  git push
  ```

[How your project should look](https://github.com/Callisto13/fizzbuzz-go/tree/02c00affbd3b9ce1e93990a1daca947fd50df7fe) at this stage.

## Part 10: Error handling

1. We are not done yet. Did you notice that when we convert the argument from a
    string to an int that we are ignoring a return value? That value is an error,
    and we want to catch and process that _before_ it gets used by our package.

    Lets write a new test to verify we do the right thing when something which is
    not a number is passed to our program.

    ```go
    It("when the command line argument is not a number, it prints an error", func() {
      fizzbuzzCommand = exec.Command(fizzbuzzBinary, "!")
      session, err := gexec.Start(fizzbuzzCommand, GinkgoWriter, GinkgoWriter)
      Expect(err).NotTo(HaveOccurred())
      Eventually(session.Out).Should(gbytes.Say("arguments must be numbers\n"))
    })
    ```

    When we run the test, it fails. The conversion in main failed and set `number`
    to `0` which means that when our package processes it, it will return "fizzbuzz".
    `0%15` is `0` so our package is technically correct, but this is not the behaviour we want:
    our users shouldn't be able to pass nonsense to our game and get "fizzbuzz".

1. Update the main func to catch non-number args:

    ```go
    number, err := strconv.Atoi(numberAsString)
    if err != nil {
      fmt.Println("arguments must be numbers")
      // exit the program here
      os.Exit(1)
    }
    ```

    Our integration test should now pass.

    If you want, you can add some extra handling (and tests) in `pkg/fizzbuzz` to
    deal with inputs being `0`.

  Commit and push this stage:

```sh
git add integration/integration_test.go main.go
git commit -m "test that non-numbers are handled"
git push
```

## Part 11: Multiple Args

Currently we can pass a single arg to our binary on the command line, but it would
be nice if we could pass a series of numbers:

```sh
./fizzbuzz 1 2 3 4 5
1
2
fizz
4
buzz
```

1. As usual, let's start by adding a new integration test:

    ```go
    It("when there are several command line argument, it processes them all", func() {
      args := []string{"1", "2", "3", "4", "5"}
      fizzbuzzCommand = exec.Command(fizzbuzzBinary, args...)
      session, err := gexec.Start(fizzbuzzCommand, GinkgoWriter, GinkgoWriter)
      Expect(err).NotTo(HaveOccurred())
      Eventually(session.Out).Should(gbytes.Say("1\n2\nfizz\n4\nbuzz\n"))
    })
    ```

1. And I will leave it to you to update `main.go` to handle this and make the test pass.

Once you have all tests passing, commit your work and push to github:

```sh
git add integration/integration_test.go main.go
git commit -m "can receive multiple args"
git push
```

## Part 12: Zero args

Lastly we need to handle a user passing in no arguments at all. Currently nothing
happens, but it would be cool if there was some nice user feedback, explaining
how to use the program.

1. Write your test.
1. Make it pass.
1. Commit your changes.

## Part 13: Tidying up

Take a look through your code. Is there any tidying up or refactoring we could do?
What about the tests? There is a little repetition, is there a way you can use
`BeforeEach` and `JustBeforeEach` or other such blocks to remove some of this? Check out the
[Ginkgo](http://onsi.github.io/ginkgo/) docs to learn more.

Remember to run your tests frequently as you make changes so that you do not accidentally
break anything. This is what they are for.

[How your project could look](https://github.com/Callisto13/fizzbuzz-go) at this stage.

## WooHoo!!

And that's it! You just test-drove your first Go program.

But don't stop there; test-driven development is a good habit to get into and the
majority of (sensible) companies value it very highly. Think back to small programs
you have written and see if you can do them again through TDD. Or test drive out
another simple game or calculator ([Leap Year](https://github.com/fouralarmfire/square-one/blob/master/tutorials/leap_year.md), maybe?).

There is a testing framework (often much more than one) for every language,
so go ahead and play Fizzbuzz again in the one of your choice.

In fact, small exercises like Fizzbuzz are a great way to get to grips with a new language
and its testing framework; it's my personal goto for the first thing I write in
whatever new thing I am trying. (Many companies also use it to test interview candidates.)

At some point I will be going on to write another, more challenging TDD tutorial
so watch this space. :)

In the meantime, check out [Exercism](https://exercism.io) for more TDD practice.

&nbsp;

<sup>_Mistakes? Corrections? Suggestions?_ <a href="https://github.com/queenofdowntime/queenofdowntime.github.io/tree/master/tutorials/fizzbuzz-go.md"><svg class="svg-icon"><use xlink:href="/assets/minima-social-icons.svg#github"></use></svg></a></sup>

<sup>_Is something unclear? Do you need help?_ <a href="https://github.com/queenofdowntime/queenofdowntime.github.io/issues/new"><svg class="svg-icon"><use xlink:href="/assets/minima-social-icons.svg#github"></use></svg></a></sup>
