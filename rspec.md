# Tips for Testing with RSpec

## Basics

### `describe` vs `context`

`describe`:
What’s being testing?

Most tests are going to be unit tests which means a specific method is usually being tested. In these cases `describe` should represent the method that's being tested.

Prepend the method name with `#` to indicate that it's an instance method. eg. `describe “#my_instance_method”`

Use either `.` or `::` to indicate that it's a class method. eg. `describe “::my_class_method”`

`context`:
What scenario is being tested?

Tests need to cover more than just the happy path. That’s where `context` comes in. It's used to specify the variation that's being tested. The string passed into context explains exactly what that variation is. eg. `context “when argument foo is 0”`

For example, if the tested method accepts an argument between an upper and lower limit, there should be three contexts:
1. When the argument is just below the lower limit
2. When the argument is within the acceptable range
3. When the argument is just above the upper limit

### `subject`

`subject` is used to define how to execute the tested code so it can be used across different tests.

It should almost always match up with the `describe` it’s nested in. `subject` should be something like `subject { Foo.new.bar }` if an instance method `bar` of class `Foo` is being tested.

The value returned by calling `subject` is memoized across the same test so `subject` can be referenced multiple times without re-executed the code.

If a test is checking for a return value and doesn’t require extra explanation, `subject` can be implicitly referenced with shorthand: `it { is_expected.to eq “value” }`

### `let` vs `let!`
`let` is lazy so the block will be evaluated when its method is first invoked. The return value is memoized so any subsequent call won’t evaluate the block. Use this for values that are directly interacted with.

`let!` is always evaluated in a before hook. It’s typically used to set state that's indirectly interacted with. For example, creating a Database (DB) record that is queried by the method that’s being tested.

### `build_stubbed` vs `build` vs `create`

A lot of code in a Rails app interacts with Active Record models. FactoryBot is used to make this a little easier. `build_stubbed`, `build` and `create` are the primary ways of interacting with FactoryBot in a test.

`build` and `create` are pretty straight forward and behave similarly to the corresponding Active Record methods. `build` creates a new instance, `create` will persist it.

`build_stubbed` creates a new instance but will prevent the record from being persisted. This is great! Making DB calls is expensive. Not only will `build_stubbed` ensure that specs don't make unnecessary DB calls, but it also enforces that it doesn't get persisted by any code being tested by raising an exception. If the code being tested interacts with an Active Record model but doesn't need to interact with the DB, use `build_stubbed`!

## Gotchas

### Global state

Everyone hates flaky specs. One of the most common causes is a spec that changes some global state and doesn't reset it. This means that if some other spec also interacts with that same state, it will behave differently depending on which test ran first.

When interacting with global state, ensure that the state is always reset. This is done automatically in a lot of cases. If the DB is considered global state, running tests inside a transaction ensures state is reset.

For something like an environment variable, resetting state needs to be dealt with explicitly. One way of doing this is by using an `around` hook.
```ruby
around do |example|
    # store the initial value
    initial_value = ENV["FOO"]
    # set it to whatever you want
    ENV["FOO"] = "BAR"

    # run the spec
    example.run

    # reset the value back to its original state
    ENV["FOO"] = initial_value
end
```
Any other test using the `ENV["FOO"]` won't be affected because it's being reset after each test.

### `:each` vs `:all`

Unlike `before(:each)` and `after(:each)`, the `before(:all)` and `after(:all)` hooks run outside a transaction which means any DB writes are persisted across specs. This is bad, because it changes global state, so avoide using `:all` when doing anything that persists to the DB.

### Absolute counts

When testing that state was changed, like an Active Record model was saved successfully to the DB, it's best to test for a relative change and not an absolute value.

A common example is checking the count of an Active Record model.

Checking for an absolute value can be affected by global state which can lead to a couple of issues.

Example
```ruby
it "creates one Foo record" do
    subject
    expect(Foo.count).to eq(1)
end
```

First, the spec may be flaky. If there's another spec that changes this state, then the spec will fail when that spec is run in an unexpected order.

Second, the test can't guarantee where the state was changed. In the example, the DB may have been seeded with a `Foo` record before the tested code was run. The test will pass, even though the tested code didn't create that record.

Checking for a change solves both these issues. It evaluates the block passed to `change` before and after evaluating the block passed to `expect`. Using `by` then compares the two results. This makes sure that a relative change is being checked and isolates the change to the tested code.
```ruby
it "creates one Foo record" do
    expect { subject }.to change { Foo.count }.by(1)
end
```

### False positives

Having a passing test isn't always enough. The test should also fail expectedly. Otherwise, the test may return false positives.

This is already part of the process if Test-Driven Development (TDD) is used. If not, change the tested code with the expectation that the test will fail and validate that it does.

### Time dependent specs

Freezing time is helpful when the result that's being tested is dependent on system time.

Example
```ruby
it "sets updated_at" do
    example.update
    expect(example.updated_at).to eq Time.current
end
```

Without freezing time, `Time.current` may return different values between setting `updated_at` and being called in the expectation. Wrapping the spec in a block that freezes time means we can guarantee we're working with the same time values.
```ruby
it "sets updated_at" do
    frozen_current_time = Time.current
    travel_to frozen_current_time do
        example.update
        expect(example.updated_at).to eq frozen_current_time
    end
end
```

Time travel on the other hand, comes in handy when testing code that depends on the system time being in a certain state. Like a query that returns records created a day ago. Using relative times can also work for this, but time travel gives more control and allows testing right up against boundaries. Like any other test, make sure to test both sides of the boundary.
```ruby
let(:record) { create :record }

it "returns records from the previous day (inclusive)" do
    travel_to record.created_at + 24.hours do
        expect(ExampleQuery.call).to include record
    end

    travel_to record.created_at + 24.hours + 1.second do
        expect(ExampleQuery.call).not_to include record
    end
end
```

Here are some gotchas to look out for when dealing with time dependant specs.
- Be aware of how a spec might be affected by Daylight Savings Time. For example, adding `24.hours` the day before the beginning or end of Daylight Savings Time won't produce the same result as adding `1.day`.
- Freezing or traveling to times changes global state. Always undo the change by resetting time to the original state. The easiest way to ensure this is by changing time using a block instead of setting and returning it explicitly.

## Patterns I like (and you may too)
These are just suggestions based on my own experiences but I think they generally make specs easier to read and maintain.

### Limit nesting
In general, code can become hard to follow when there's a lot of nesting.

There aren't many cases where there should be multiple nested `describe`s. This can happen pretty easily for `context`s, especially if there are many variables that change the outcome of a spec. Instead of nesting `context`s in those cases, I prefer to flatten them by making the `context` a combination of the many variables. Understanding what scenario is being tested and what variables go into it is now simpler to understand at a glance.
```ruby
describe "#bar" do
    subject { Foo.bar(a, b, c) }

    context "when a is true" do
        let(:a) { true }
        context "when b is positive" do
            let(:b) { 1 }
            context "when c is nil" do
                let(:c) { nil }
                ...
            end
        end
    end
end
```
Becomes
```ruby
describe "#bar" do
    subject { Foo.bar(a, b, c) }

    context "when a is true, b is positive and c is null" do
        let(:a) { true }
        let(:b) { 1 }
        let(:c) { nil }
        ...
    end
end
```

### Overriding `let`s
I like to set top-level inputs using `let` that represent the happy path when testing something that has a happy path with some variations, like code that validates the input. Then for each variation I create a `context` and override the exact input that leads to the new result. It removes the noise of the other inputs and makes the input affecting the result clear.
```ruby
describe "#bar" do
    subject { Foo.bar(a, b, c) }

    let(:a) { true }
    let(:b) { 1 }
    let(:c) { "yo" }

    it "succeeds" do
        ...
    end

    context "when b is negative" do
        let(:b) { -1 }

        it "fails" do
            ...
        end
    end

    context "when c is nil" do
        let(:c) { nil }

        it "fails" do
            ...
        end
    end
end
```
