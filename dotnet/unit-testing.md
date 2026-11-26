# Unit Testing Standards (.NET)

## Tooling
- **Framework:** TUnit.
- **Test platform:** Microsoft.Testing.Platform (MTP). Use MTP rather than VSTest wherever possible, including for test projects that don't use TUnit.
- **Assertions:** TUnit's fluent assertions (`await Assert.That(x).IsEqualTo(y)`). Don't add Shouldly, FluentAssertions or the `TUnit.Assertions.Should` syntax.
- **Mocking:** TUnit.Mocks. It is source generated and needs C# 14 (`<LangVersion>14</LangVersion>` or later). Only mock types you own at architectural boundaries.
- **Time:** use `FakeTimeProvider` (from `Microsoft.Extensions.TimeProvider.Testing`) rather than mocking `TimeProvider`.
- **Coverage:** `Microsoft.Testing.Extensions.CodeCoverage`, which the `TUnit` package already includes. Don't add `coverlet.collector` or `coverlet.msbuild`, because they don't work with MTP.

## Microsoft.Testing.Platform setup
- Put this in the repo's `global.json` so `dotnet test` runs in MTP mode:
  ```json
  { "test": { "runner": "Microsoft.Testing.Platform" } }
  ```
- Filter with `--treenode-filter`, not `--filter`. VSTest-style `--filter` expressions run zero tests.
  - `dotnet test --treenode-filter "/*/*/OrderServiceTests/*"`
- Collect coverage with `dotnet test --coverage --coverage-output-format cobertura`.
- In the IDE, turn on MTP support: "Use testing platform server mode" in Visual Studio, "Enable Testing Platform support" in Rider, "Use Testing Platform Protocol" in VS Code.

## Project layout
- Each production project `X` gets one test project, `X.Tests`, under `tests/`, mirroring `src/`.
- Make test folders and namespaces mirror the code under test.
- Name test classes after the class under test plus `Tests`: `OrderService` -> `OrderServiceTests`.
- Put integration tests in a separate `X.IntegrationTests` project so unit test runs stay fast.

## Naming
- Name test methods as a sentence that describes the scenario being tested and the expected outcome. Put an underscore between every word and capitalize only the first word.
  - `When_two_integers_given_the_sum_is_returned_as_the_result`
  - `When_the_cart_is_empty_the_total_is_zero`
  - `When_payment_is_declined_placing_the_order_throws_a_payment_exception`
- This is an exception to the PascalCase method naming rule in the C# code style. It applies only to test methods. If an analyzer flags the underscores (for example CA1707), suppress that rule for test projects only.
- Describe behavior, not implementation. Don't put method names in the test name unless the method name is the clearest way to describe the behavior.

## Structure
- Use Arrange / Act / Assert, with the three sections separated by a blank line. Start each section with an `// Arrange`, `// Act` or `// Assert` comment.
  - Exception: when every section is a single line, leave the comments out. Three lines separated by blank lines are already clear, and the comments would only add lines.
- Test one behavior per test. Several asserts are fine if they check the same outcome. Chain them with `.And`, or wrap them in `using (Assert.Multiple())` so every failure is reported.
- Name the system under test `sut`.
- Keep test setup local and readable. Prefer small factory or builder methods in the test class over large shared fixtures. TUnit creates a new class instance per test, so field initializers and the constructor are the place for per-test setup; use `[Before(Test)]` only for async setup.
- Use `[Arguments]` / `[MethodDataSource]` for input variations instead of copy-pasting `[Test]`s.
- Create mocks with `IFoo.Mock()`, set them up with the generated methods and matchers (`Any()`, raw values, lambdas), and verify with `WasCalled(Times.Once)` / `WasNeverCalled()`. Only verify calls when the call itself is the behavior under test.

```csharp
public sealed class OrderServiceTests
{
    private readonly Mock<IPaymentGateway> paymentGateway = IPaymentGateway.Mock();
    private readonly FakeTimeProvider time = new(new DateTimeOffset(2026, 1, 1, 0, 0, 0, TimeSpan.Zero));

    [Test]
    public async Task When_payment_is_declined_placing_the_order_throws_a_payment_exception(CancellationToken cancellationToken)
    {
        // Arrange
        this.paymentGateway.ChargeAsync(Any(), Any()).Returns(PaymentResult.Declined);
        var sut = this.CreateSut();

        // Act
        var act = () => sut.PlaceOrderAsync(OrderBuilder.Valid(), cancellationToken);

        // Assert
        await Assert.That(act).Throws<PaymentException>();
    }

    [Test]
    public async Task When_the_cart_is_empty_the_total_is_zero()
    {
        var sut = new Cart();

        var total = sut.CalculateTotal();

        await Assert.That(total).IsEqualTo(0m);
    }

    private OrderService CreateSut() => new(this.paymentGateway.Object, this.time);
}
```

## Rules
- Always `await` assertions. An assertion that isn't awaited never runs, and the test passes silently.
- Tests must be deterministic. Don't touch the real clock, random numbers without a seed, the network, the file system or shared static state.
- Tests must be independent and runnable in any order. TUnit runs tests in parallel by default; don't use `[NotInParallel]` to hide shared state. If tests truly need an exclusive resource, use `[NotInParallel("resource-key")]` with a key.
- Never use `Thread.Sleep` or `Task.Delay` to wait. Advance `FakeTimeProvider` or await the actual signal.
- Test behavior through the public API. Don't test private methods through reflection or `InternalsVisibleTo` hacks.
- Don't put logic (loops, conditionals) in tests.
- Take a `CancellationToken` parameter in async tests and pass it to the calls under test.
- Every bug fix includes a test that fails without the fix.

## What to test
- Business logic, branching, edge cases (null, empty, boundary values) and error paths.
- Don't unit test framework behavior, trivial properties or generated code.
- Coverage is a signal, not a target. Aim for meaningful coverage of domain logic (roughly 80% or more). Don't write tests just to raise the number.

## Running
- `dotnet test` must pass locally before you open a pull request.
- Don't mark tests `[Skip]` without a ticket reference in the skip reason: `[Skip("Flaky on CI, see #1234")]`.
