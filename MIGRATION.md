# Migrating tests from Kiota to NSwag

## Business-unit tests: generated interfaces and NSubstitute

Follow the owning `Gainsway.LiquidApi.Auth` library's generation instructions.
Its packaged generator emits client interfaces; register the concrete generated
client with the auth library and inject the corresponding interface into callers.
Do not create a second generator or handwritten client interface in this package.

Replace Kiota request-builder mocks with typed interface substitutes:

```csharp
// Before
_bankingServiceClient.Api.BankTransfers.Withdraw.MockPostAsync(response);

// After
var bankingClient = Substitute.For<IBankingServiceClient>();
bankingClient.ApiBankTransfersWithdrawAsync(Arg.Any<WithdrawRequest>())
    .Returns(response);

await activity.TriggerBankWithdrawalWithStatusAsync(/* business inputs */);

await bankingClient.Received(1).ApiBankTransfersWithdrawAsync(
    Arg.Is<WithdrawRequest>(request => request.Amount == expectedAmount));
```

Preserve meaningful checks on business inputs, error handling, retries,
idempotency and call counts. Stub boundary exceptions directly when testing how
business logic handles them. Unit tests need NSubstitute, **not** this HTTP package.
Use the generated DTOs; do not add a route-string adapter around interface mocks.

## Generated-client contract tests: optional HTTP boundary checks

Only keep real HTTP-level mocks where the behavior under test belongs to the
generated transport: exact serialized decimals, omitted request fields, query
encoding, status/error parsing and malformed/null response bodies. Keep these
checks in a clearly named boundary/contract test fixture or project, separate
from business-unit tests.

```csharp
var client = NswagClientMockExtensions.GetMockableClient<BankingServiceClient>();
client.MockPostAsync("/api/bank-transfers/withdraw", response);
await client.ApiBankTransfersWithdrawAsync(request);
await client.VerifyPostAsync("/api/bank-transfers/withdraw", Times.Once)
    .WithBody<WithdrawRequest>(body => body.Amount == expectedAmount);
```

These paths are explicit wire expectations, not a substitute for typed operation
mocks. `GetMockRequests()` exposes serialized JSON for precision/omission checks.
`required` catches missing members in C# initializers but cannot validate remote
JSON; controller tests do not automatically test the outbound client parser.

## Missing mocks are configuration failures

An unmatched HTTP request throws `InvalidOperationException` with its method and
URI, rather than returning 404. Configure every expected HTTP error explicitly:

```csharp
client.MockGetError("/api/items/404", HttpStatusCode.NotFound);
client.MockGetAsync("/api/items/5", new HttpRequestException("offline"));
```

Explicit no-content responses are available through e.g.
`client.MockPostAsync("/api/items")`; ensure 204 is a declared success status for
the generated operation. HTTP errors are parsed by the real client into its
configured API exception family.

Remove `Gainsway.Kiota.Testing`, request builders, `IRequestAdapter`,
`RequestInformation` and Kiota serialization helpers from migrated tests.
Add `Gainsway.NSwag.Testing` **only** to projects retaining wire-contract tests.
The package does not reference or install Kiota.
