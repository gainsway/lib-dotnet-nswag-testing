# NSwag generated-client testing

**Use generated interfaces and plain NSubstitute for business-unit tests.**
Enable client interfaces through the generation tooling supplied by
`Gainsway.LiquidApi.Auth`, inject `IBankingServiceClient` (for example), and
configure/verify typed operations directly:

```csharp
var bankingClient = Substitute.For<IBankingServiceClient>();
bankingClient.ApiBankTransfersWithdrawAsync(Arg.Any<WithdrawRequest>())
    .Returns(response);

await activity.TriggerBankWithdrawalWithStatusAsync(/* business inputs */);

await bankingClient.Received(1).ApiBankTransfersWithdrawAsync(
    Arg.Is<WithdrawRequest>(request => request.Amount == expectedAmount));
```

A changed operation, request or response type then fails compilation. These unit
tests do not need this package or string-based HTTP paths.

## When this package is useful

`Gainsway.NSwag.Testing` is an **optional HTTP boundary-test utility**, not a
replacement for interface substitutes. Use it only when a test must execute the
actual generated client to verify outbound JSON, URI/query encoding, wire error
mapping or required-field deserialization. Controller/component tests exercise
the inbound API; they do not by themselves execute a downstream generated client's
serializer. C# `required` checks source initializers, not JSON received from a
remote service. An interface substitute cannot establish those wire contracts.

Account-service keeps these checks in its integration/contract test project and
uses plain NSubstitute interfaces in business-unit tests. No live network or
authentication calls are made by this helper.

## Optional installation and wire-level example

```sh
dotnet add <contract-test-project> package Gainsway.NSwag.Testing
```

```csharp
var client = NswagClientMockExtensions.GetMockableClient<BankingServiceClient>();
client.MockPostAsync("/api/bank-transfers/withdraw", response);

await client.ApiBankTransfersWithdrawAsync(request);

await client.VerifyPostAsync("/api/bank-transfers/withdraw", Times.Once)
    .WithBody<WithdrawRequest>(body => body.Amount == expectedAmount);
```

Here the route is intentionally part of the wire assertion. Do not use this
pattern to mock ordinary business-unit dependencies.

## Strict requests and explicit errors

**An unmatched method, path, query or predicate throws `InvalidOperationException`**
with the method/URI. It never fabricates a 404 that production code could treat as
an expected not-found result. The request remains recorded for diagnosis.
Configure expected failures explicitly:

```csharp
client.MockGetError("/api/items/404", HttpStatusCode.NotFound);
client.MockGetAsync("/api/items/5", new HttpRequestException("offline"));
```

Helpers support GET/POST/PUT/PATCH/DELETE, collections, no-content/error/transport
responses, `GetMockRequests()`, `Times` and request-body verification. Rules match
HTTP method/path segments (`{id}` placeholders are supported), optional query
strings and predicates; the last matching rule wins. JSON uses Newtonsoft.Json
with camelCase properties, matching NSwag's default serializer.

See [MIGRATION.md](MIGRATION.md) for the split between typed unit tests and explicit
wire-contract tests. The existing `Gainsway.Kiota.Testing` package is unchanged.
