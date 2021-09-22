<img
src="https://user-images.githubusercontent.com/79303/134366631-9c6cfe67-a9c0-4096-bbea-ba1698a85b0b.png"
width="90%"/>

# Mocktail 🍸

Mocktail is a [test
double](https://github.com/testdouble/contributing-tests/wiki/Test-Double)
library for Ruby. It offers a simple API and robust feature-set.

## First, an aperitif

Before getting into the details, let's demonstrate what Mocktail's API looks
like. Suppose you have a class `Negroni`:

```ruby
class Negroni
  def self.ingredients
    [:gin, :campari, :sweet_vermouth]
  end

  def shake!(shaker)
    shaker.mix(self.class.ingredients)
  end

  def sip(amount)
    raise "unimplemented"
  end
end
```

1. Create a mocked instance: `negroni = Mocktail.of(Negroni)`
2. Stub a response with `stubs { negroni.sip(4) }.with { :ahh }`
    * Calling `negroni.sip(4)` will subsequently return `:ahh`
    * Another example: `stubs { |m| negroni.sip(m.numeric) }.with { :nice }`
3. Verify a call with `verify { negroni.shake!(:some_shaker) }`
    * `verify` will raise an error unless `negroni.shake!(:some_shaker)` has
      been called
    * Another example: `verify { |m| negroni.shake!(m.that { |arg|
      arg.respond_to?(:mix) }) }`
4. Deliver a mock to your code under test with `negroni =
Mocktail.of_next(Negroni)`
    * `of_next` will return a fake `Negroni`
    * The next call to `Negroni.new` will return _exactly the same_ fake
      instance, allowing the code being tested to seamlessly instantiate and
      interact with it
    * This means no dependency injection is necessary, nor is a sweeping
      override like
      [any_instance](https://relishapp.com/rspec/rspec-mocks/docs/working-with-legacy-code/any-instance)
    * `Negroni.new` will be unaffected on other threads and will continue
      behaving like normal as soon as the next `new` call

Mocktail can do a whole lot more than this, and was also designed with
descriptive error messages and common edge cases in mind:

* Entire classes and modules can be replaced with `Mocktail.replace(type)` while
  preserving thread safety
* Arity of arguments and keyword arguments is enforced on faked methods to
  prevent isolated unit tests from continuing to pass after an API contract
  changes
* For mocked methods that take a block, `stubs` & `verify` can inspect and
  invoke the passed block to determine whether the call satisfies their
  conditions
* Dynamic stubbings that return a value based on how the mocked method was
  called
* Advanced stubbing and verification options like specifying the number of
  `times` a stub can be satisfied or a call should be verified, allowing tests
  to forego specifying arguments and blocks, and temporarily disabling arity
  validation
* Built-in matchers as well as custom matcher support
* Argument captors for complex, multi-step call verifications

## Getting started

### Install

The main ingredient to add to your Gemfile:

```ruby
gem "mocktail", group: :test
```

### Add the DSL

Then, in each of your tests or in a test helper, you'll probably want to include
Mocktail's DSL. (This is optional, however, as every method in the DSL is also
available as a singleton method on `Mocktail`.)

In Minitest, you might add the DSL with:

```ruby
class Minitest::Test
  include Mocktail::DSL
end
```

Or, in RSpec:

```ruby
RSpec.configure do |config|
  config.include Mocktail::DSL
end
```

### Clean up after each test

When making so many concoctions, it's important to keep a clean bar! To reset
Mocktail's internal state between tests and avoid test pollution, you should
also call `Mocktail.reset` after each test:

In Minitest:

```ruby
class Minitest::Test
  # Or, if in a Rails test, in a `teardown do…end` block
  def teardown
    Mocktail.reset
  end
end
```

And RSpec:

```ruby
RSpec.configure do |config|
  config.after(:each) do
    Mocktail.reset
  end
end
```

## API

The public API is a pretty quick read of the [top-level module's
source](lib/mocktail.rb). Here's a longer menu to explain what goes into each
feature.

### Mocktail.of

`Mocktail.of(module_or_class)` takes a module or class and returns an instance
of an object with fake methods in place of all its instance methods which can
then be stubbed or verified.

```ruby
class Clothes; end;
class Shoe < Clothes
  def tie(laces)
  end
end

shoe = Mocktail.of(Shoe)
shoe.instance_of?(Shoe) # => true
shoe.is_a?(Clothes) # => true
shoe.class == Shoe # => false!
shoe.to_s # => #<Mocktail of Shoe:0x00000001343b57b0>"
```

### Mocktail.of_next

`Mocktail.of_next(klass, [count: 1])` takes a class and returns one mock (the
default) or an array of multiple mocks. It also effectively overrides the
behavior of that class's constructor to return those mock(s) in order and
finally restoring its previous behavior.

For example, if you wanted to test the `Notifier` class below:

```ruby
class Notifier
  def initialize
    @mailer = Mailer.new
  end

  def notify(name)
    @mailer.deliver!("Hello, #{name}")
  end
end
```

You could write a test like this:

```ruby
def test_notifier
  mailer = Mocktail.of_next(Mailer)
  subject = Notifier.new

  subject.notify("Pants")

  verify { mailer.deliver!("Hello, Pants") }
end
```

There's nothing wrong with creating mocks using `Mocktail.of` and passing them
to your subject some other way, but this approach allows you to write very terse
isolation tests without foisting additional indirection or dependency injection
in for your tests' sake.

### Mocktail.stubs

Configuring a fake method to take a certain action or return a particular value
is called "stubbing". To stub a call with a value, you can call `Mocktail.stubs`
(or just `stubs` if you've included `Mocktail::DSL`) and then specify an effect
that will be invoked whenever that call configuration is satisfied using `with`.

The API is very simple in the simple case:

```ruby
class UserRepository
  def find(id, debug: false); end

  def transaction(&blk); end
end
```

You could stub responses to a mock of the `UserRepository` like this:

```ruby
user_repository = Mocktail.of(UserRepository)

stubs { user_repository.find(42) }.with { :a_user }
user_repository.find(42) # => :a_user
user_repository.find(43) # => nil
user_repository.find # => ArgumentError: wrong number of arguments (given 0, expected 1)
```

The block passed to `stubs` is called the "demonstration", because it represents
an example of the kind of calls that Mocktail should match.

If you want to get fancy, you can use matchers to make your demonstration more
dynamic. For example, you could match any number with:

```ruby
stubs { |m| user_repository.find(m.numeric) }.with { :another_user }
user_repository.find(41) # => :another_user
user_repository.find(42) # => :another_user
user_repository.find(43) # => :another_user
```

Stubbings are last-in-wins, which is why the stubbing above would have
overridden the earlier-but-more-specific stubbing of `find(42)`.

A stubbing's effect can also be changed dynamically based on the actual call
that satisfied the demonstration by looking at the `call` block argument:

```ruby
stubs { |m| user_repository.find(m.is_a(Integer)) }.with { |call|
  {id: call.args.first}
}
user_repository.find(41) # => {id: 41}
# Since 42.5 is a Float, the earlier stubbing will win here:
user_repository.find(42.5) # => :another_user
user_repository.find(43) # => {id: 43}
```

It's certainly more complex to think through, but if your stubbed method takes a
block, your demonstration can pass a block of its own and inspect or invoke it:

```ruby
stubs {
  user_repository.transaction { |block| block.call == {id: 41} }
}.with { :successful_transaction }

user_repository.transaction {
  user_repository.find(41)
} # => :successful_transaction
user_repository.transaction {
  user_repository.find(40)
} # => nil
```

There are also several advanced options you can pass to `stubs` to control its
behavior.

`times` will limit the number of times a satisfied stubbing can have its effect:

```ruby
stubs { |m| user_repository.find(m.any) }.with { :not_found }
stubs(times: 2) { |m| user_repository.find(1) }.with { :someone }

user_repository.find(1) # => :someone
user_repository.find(1) # => :someone
user_repository.find(1) # => :not_found
```

`ignore_extra_args` will allow a demonstration to be considered satisfied even
if it fails to specify arguments and keyword arguments made by the actual call:

```
stubs { user_repository.find(4) }.with { :a_person }
user_repository.find(4, debug: true) # => nil

stubs(ignore_extra_args: true) { user_repository.find(4) }.with { :b_person }
user_repository.find(4, debug: true) # => :b_person
```

And `ignore_block` will similarly allow a demonstration to not concern itself
with whether an actual call passed the method a block—it's satisfied either way:

```ruby
stubs { user_repository.transaction }.with { :transaction }
user_repository.transaction {} # => nil

stubs(ignore_block: true) { user_repository.transaction }.with { :transaction }
user_repository.transaction {} # => :transaction
```

### Mocktail.verify

In practice, we've found that we stub far more responses than we explicitly
verify a particular call took place. That's because our code normally returns
some observable value that is _influenced_ by our dependencies' behavior, so
adding additional assertions that they be called would be redundant. That
said, for cases where a dependency doesn't return a value but just has a
necessary side effect, the `verify` method exists (and like `stubs` is included
in `Mocktail::DSL`).

Once you've gotten the hang of stubbing, you'll find that the `verify` method is
intentionally very similar. They almost rhyme.

For this example, consider an `Auditor` class that our code might need to call
to record that certain actions took place.

```ruby
class Auditor
  def record!(message, user_id:, action: nil); end
end
```

Once you've created a mock of the `Auditor`, you can start verifying basic
calls:

```ruby
auditor = Mocktail.of(Auditor)

verify { auditor.record!("hello", user_id: 42) }
# => raised Mocktail::VerificationError
# Expected mocktail of Auditor#record! to be called like:
#
#   record!("hello", user_id: 42)
#
# But it was never called.
```

Wups! Verify will blow up whenever a matching call hasn't occurred, so it
should be called after you've invoked your subject under test along with any
other assertions you have.

If we make a call that satisfies the `verify` call's demonstration, however, you
won't see that error:

```ruby
auditor.record!("hello", user_id: 42)

verify { auditor.record!("hello", user_id: 42) } # => nil
```

There, nothing happened! Just like any other assertion library, you only hear
from `verify` when verification fails.

Just like with `stubs`, you can any built-in or custom matchers can serve as
garnishes for your demonstration:

```ruby
auditor.record!("hello", user_id: 42)

verify { |m| auditor.record!(m.is_a(String), user_id: m.numeric) } # => nil
# But this will raise a VerificationError:
verify { |m| auditor.record!(m.is_a(String), user_id: m.that { |arg| arg > 50}) }
```

When you pass a block to your demonstration, it will be invoked with any block
that was passed to the actual call to the mock. Truthy responses will satisfy
the verification and falsey ones will fail:

```ruby
auditor.record!("ok", user_id: 1) { Time.new }

verify { |m| auditor.record!("ok", user_id: 1) { |block| block.call.is_a?(Time) } } # => nil
# But this will raise a VerificationError:
verify { |m| auditor.record!("ok", user_id: 1) { |block| block.call.is_a?(Date) } }
```

`verify` supports the same options as `stubs`:

* `times` will require the demonstrated call happened exactly `times` times (by
  default, the call has to happen 1 or more times)
* `ignore_extra_args` will allow the demonstration to forego specifying optional
  arguments while still being considered satisfied
* `ignore_block` will similarly allow the demonstration to forego specifying a
  block, even if the actual call receives one

### Mocktail.matchers

You'll probably never need to call `Mocktail.matchers` directly, because it's
the object that is passed to every demonstration block passed to `stubs` and
`verify`. By default, a stubbing (e.g. `stubs { email.send("text") }`) is only
considered satisfied if every argument passed to an actual call was passed an
`==` check. Matchers allow us to relax or change that constraint for both
regular arguments and keyword arguments so that our demonstrations can match
more kinds of method invocations.

Matchers allow you to specify stubbings and verifications that look like this:

```ruby
stubs { |m| email.send(m.is_a(String)) }.with { "I'm an email" }
```

#### Built-in matchers

These matchers come out of the box:

* `any` - Will match any value (even nil) in the given argument position or
  keyword
* `is_a(type)` - Will match when its `type` passes an `is_a?` check against the
  actual argument
* `includes(thing, [**more_things])` - Will match when all of its arguments are
  contained by the corresponding argument—be it a string, array, hash, or
  anything that responds to `includes?`
* `matches(pattern)` - Will match when the provided string or pattern passes
  a `match?` test on the corresponding argument; usually used to match strings
  that contain a particular substring or pattern, but will work with any
  argument that responds to `match?`
* `not(thing)` - Will only match when its argument _does not_ equal (via `!=`)
  the actual argument
* `numeric` - Will match when the actual argument is an instance of `Integer`,
  `Float`, or (if loaded) `BigDecimal`
* `that { |arg| … }` - Takes a block that will receive the actual argument. If
  the block returns truthy, it's considered a match; otherwise, it's not a
  match.

#### Custom matchers

If you want to write your own matchers, check out [the source for
examples](lib/mocktail/matchers/includes.rb). Once you've implemented a class,
just pass it to `Mocktail.register_matcher` in your test helper.

```ruby
class MyAwesomeMatcher < Mocktail::Matchers::Base
  def self.matcher_name
    :awesome
  end

  def match?(actual)
    "#{@expected}✨" == actual
  end
end

Mocktail.register_matcher(MyAwesomeMatcher)
```

Then, a stubbing like this:

```ruby
stubs { |m| user_repository.find(m.awesome(11)) }.with { :awesome_user }

user_repository.find("11")) # => nil
user_repository.find("11✨")) # => :awesome_user
```

### Mocktail.captor

An argument captor is a special kind of matcher… really, it's a matcher factory.
Suppose you have a `verify` call for which one of the expected arguments is
_really_ complicated. Since `verify` tends to be paired with fire-and-forget
APIs that are being invoked for the side effect, this is a pretty common case.
You want to be able to effectively snag that value and then run any number of
specific assertions against it.

That's what `Mocktail.captor` is for. It's easiest to make sense of this by
example. Given this `BigApi` class that's presumably being called by your
subject at the end of a lot of other work building up a payload:

```ruby
class BigApi
  def send(payload); end
end
```

You could capture the value of that payload as part of the verification of the
call:

```ruby
big_api = Mocktail.of(BigApi)

big_api.send({imagine: "that", this: "is", a: "huge", object: "!"})

payload_captor = Mocktail.captor
verify { big_api.send(payload_captor.capture) } # => nil!
```

The `verify` above will pass because _a_ call did happen, but we haven't
asserted anything beyond that yet. What really happened is that
`payload_captor.capture` actually returned a matcher that will return true for
any argument _while also sneakily storing a copy of the argument value_. 

That's why we instantiated `payload_captor` with `Mocktail.captor` outside the
demonstration block, so we can inspect its `value` after the `verify` call:

```ruby
payload_captor = Mocktail.captor
verify { big_api.send(payload_captor.capture) } # => nil!

payload = payload_captor.value # {:imagine=>"that", :this=>"is", :a=>"huge", :object=>"!"}
assert_equal "huge", payload[:a]
```

### Mocktail.replace

Mocktail was written to support isolated test-driven development, which usually
results in a lot of boring classes and instance methods. But sometimes you need
to mock singleton methods on classes or modules, and we support that too.

When you call `Mocktail.replace(type)`, all of the singleton methods on the
provided type are replaced with fake methods available for stubbing and
verification. It's really that simple.

[**Obligatory warning:** Mocktail does its best to ensure that other threads
won't be affected when you replace the singleton methods on a type, but your
mileage may very! Singleton methods are global and code that introspects or
invokes a replaced method in a peculiar-enough way could lead to hard-to-track
down bugs. (If this concerns you, then the fact that class methods are
effectively global state may be a great reason not to rely too heavily on
them!)]

### Mocktail.reset

This one's simple: you probably want to call `Mocktail.reset` after each test,
but you _definitely_ want to call it if you're using `Mocktail.replace` or
`Mocktail.of_next` anywhere, since those will affect state that is shared across
tests.

## Acknowledgements

Mocktail is created & maintained by the software agency [Test
Double](https://twitter.com). If you've ever come across our eponymously-named
[testdouble.js](https://github.com/testdouble/testdouble.js/), you might find
Mocktail's API to be quite similar. The term "test double" was originally coined
by Gerard Meszaros in his book [xUnit Test
Patterns](http://xunitpatterns.com/Test%20Double.html).

The name is inspired by the innovative Java mocking library
[Mockito](https://site.mockito.org). Mocktail also the spiritual successor to
[gimme](https://github.com/searls/gimme), which offers a similar API but which
fell victim to the limitations of Ruby 1.8.7 (and
[@searls](https://twitter.com/searls)'s Ruby chops). Gimme was also one of the
final projects we collaborated with [Jim Weirich](https://github.com/jimweirich)
on, so this approach to isolated unit testing holds a special significance to
us.

## Code of Conduct

This project follows Test Double's [code of
conduct](https://testdouble.com/code-of-conduct) for all community interactions,
including (but not limited to) one-on-one communications, public posts/comments,
code reviews, pull requests, and GitHub issues. If violations occur, Test Double
will take any action they deem appropriate for the infraction, up to and
including blocking a user from the organization's repositories.

