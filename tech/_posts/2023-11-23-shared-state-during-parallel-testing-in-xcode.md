---
layout: post
title: Shared State During Parallel Testing in Xcode
---

Starting with Xcode 10, developers have been able to run their unit tests in parallel. This can greatly reduce the time it takes to run your tests. But what if your project depends on some shared state? What if you are writing to files? Will your test data clobber each other and lead to flaky tests?

<!--excerpt-->

Of course, the best practice is to inject dependencies into the objects that need them so that you can mock them out during testing. But not every project follows best practices.

For example, let's say we had a `SharedState` object that stored some data that was needed throughout the app:

```swift
public class SharedState {
  public static var shared = SharedState()

  public var value = "default-value"

  public func reset() {
    value = "default-value"
  }
}
```

If you wanted to provide mock data for this value in tests, would this present a problem? This was the question I set out to answer.
## Sample App Setup
To investigate this, I wrote a number of tests in a number of different modules, and configured them to all run in parallel. I was doing this in a codebase I cannot share, but perhaps I will write a sample app for it and update this article when I do.

The app and tests were set up like this:
* One dynamically linked framework that contained the `SharedState` object
* Many dynamically linked frameworks with many tests referencing `SharedState`
* Multiple top-level targets (App, Widget, etc.) with tests referencing `SharedState`
* Test cases had about 20 test methods
- Test cases set the value in the `SharedState` to a random identifier in its `setUp` function
- Test methods would sleep for a random interval up to 1 second and then verify that the value in the `SharedState` was the same as it was in the `setUp` method
- Test cases would reset the value in `SharedState` to the default in the `tearDown` method

In essence, a typical test case class looked like this:

```swift
class SomeTestCase: XCTestCase {
  /// Store the test case name for debugging
  static var testCaseID: String { "\(Self.self)" }

  /// Test ID for the specific test method for debugging
  var testID: String = "\(Self.testCaseID)-\(randomID)"

  /// Random ID defined for each test that is run
  var randomID = Int.random(in: 0...Int.max)

  /// Random amount of time to pause execution of the test
  func randomDuration() -> UInt64 { 
    UInt64.random(in: 0...1_000_000_000)
  }

  override func setUp() {
    super.setUp()
    // Change the shared value to our custom test ID
    SharedState.shared.value = testID
  }

  override func tearDown() {
    super.tearDown()
    // Reset the shared value after the test
    SharedState.shared.reset()
  }

  func test_sharedState_01() async {
    // Yield execution to other tests
    try? await Task.sleep(nanoseconds: randomDuration())

    // Verify that the value we set during setUp is still there
    XCTAssertEqual(SharedState.shared.value, testID)
  }

  // Have about 20 of the shared state tests
}

// Have about 20 more of these shared state test cases
```

All of the test schemes were set to run in parallel and in randomized order when possible. Framework unit tests did not have an App Host, but the top-level targets did (the app they were written for).
## What Happened
By following along in the logs for the logs as the unit tests were running, it was possible to follow along and see what was happening. Keeping a long delay during the tests made it run slow enough that it was possible to see what was being executed and when, in real time.

I'm going to use the term *Test Runner* here. What I mean is the process that is running the tests. When a test target has an App Host, the *Test Runner* is the app running in the simulator. When there is no App Host, it is an instance of *xctest* running.

This is what happened:
- Xcode started a number of test runners
- Test cases were assigned to test runners
- Test cases would run their tests serially, no two tests were running concurrently
- Each test method would run inside its own instance of the test case class
- When the test case on a test runner finished, a new test case would be assigned to it

All of my tests passed. They were mutating shared state before the test ran; while the test ran, the value had not been overwritten by other tests; they reset the state after they were done.

What this means is that _it is safe_ to mutate globally shared state within your test methods _as long as_ you clean it up after you are done.

Just to check, I added print statements to the tests and stopped cleaning up the value afterwards. The value of the shared state from the previous test would be maintained in the `setUp` method of the next test or test case. This makes sense, because the tests are running in the same test runner process and use the same memory. It does indicate that if you are going to mutate shared state _you must clean it up_ when you are done.
## Testing Persisted Values
Mutating shared state that is stored in memory is important and can make it easier to test a codebase that perhaps deserves a little bit of refactoring. But does this work when the data lives outside of the app process? Can we extend this thinking to data that is persisted outside the test process? 

I tried this out too by storing some data to a file and storing data to `UserDefaults.standard` to see what would happen.

I did this by adding computed properties to `SharedState` that:
- Atomically read and wrote to a file
	- I tested both `.libraryDirectory` and `.documents` directories, but not more than that
- Read and write from a value in `UserDefaults.standard`

This is what I added to the `SharedState` object to access some externally shared state:

```swift
private let files = FileManager.default

public var fileContents: String {
  get {
    let url = files.urls(for: .libraryDirectory, in: .userDomainMask)[0]
    let fileURL = url.appendingPathComponent("SharedState.fileContents.txt")
    let string = (try? Data(contentsOf: fileURL))
      .flatMap { String(data: $0, encoding: .utf8) }
    return string ?? "uninitialized"
  }
  set {
    let url = files.urls(for: .libraryDirectory, in: .userDomainMask)[0]
    let fileURL = url.appendingPathComponent("SharedState.fileContents.txt")
    let data = Data(newValue.utf8)
    try? data.write(to: fileURL)
  }
}

private let defaults = UserDefaults.standard

public var userDefaultsValue: String {
  get {
    return defaults.string(forKey: "testing-userDefaultsValue") ?? "uninitialized"
  }
  set {
    defaults.set(newValue, forKey: "testing-userDefaultsValue")
  }
}
```

Now when I run the tests, I will be reading and writing to pieces of data that is *external* to the process running the test, and it might actually cause problems.
## What Happened
After early success with values stored in memory, I was half expecting my tests mutating persisted data to Just Work as well. What was confusing at first was that some tests did Just Work, and some tests didn't.

All of the steps above about tests running in different test runners and never running more than one test at a time on a single test runner was still true. 

What was different this time was that tests running in an iOS Simulator did all seem to Just Work and continue passing, whereas tests that were running in instances of xctest were failing.

To find out why this was happening, I printed out where the files it was writing to was, and where the UserDefaults plist was being stored. This made things pretty clear to me.

Tests running in iOS Simulators had their own sandboxed environment. They were saving files to their own private directory. Tests running in *xctest* instances were all writing to files in the same shared directory.
## What Have We Learned
If you are working in a codebase where components depend on elements of shared state, you may have been hesitant to write tests for them because you thought "running tests in parallel" would mean that your tests might overwrite your other tests' mock data. This isn't the case.

* Tests run serially in independent processes running in parallel
* It is safe to mutate shared state in tests, as long as you reset it when you are done
* It *may be* safe to read and write to external data, as long as you reset it when you are done
	* It is safe if your tests have an App Host
	* It is *not* safe if your tests *do not* have an App Host

And last of all, we remember that of course that it's safest to do proper dependency injection so that you don't have to rely on mocking out globally shared state 😅