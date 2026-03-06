## Each test must follow a clear Arrange – Act – Assert structure and verify one observable behavior.

## Always assert both positive and negative outcomes.

## Make assertions as generic and global as possible: assert against the entire array instead of individual items to catch unexpected additions. 

## To check that an exception is thrown in a test and then assert more things, add a try catch and add assertions inside catch. At the end of try, add self::fail. 

## Create reusable assertion helpers for models and aggregates e.g. ProductTestAssertions::assertProductCreated($product, $expectedData).

## Fixtures must create only the data required for the test and must make intent explicit e.g. createInvoiceableOrganization().

## Use a Test Process class when tests involve repeated multi-step workflows, complex setup, or when you want scenario-style readability in integration tests. Prefer Process classes over duplicating setup logic.

## Do not create interfaces solely for testing. Use in-memory implementations or concrete test doubles instead.

## Inject test doubles via the constructor, never via method parameters. Initialize all test doubles in setUp(). Override implementations using $this->getDi()->overrideImplementation() so wiring mirrors production behavior.

## Explicitly test idempotency.

## Test all enum values explicitly.

## Prefer domain-language method names in tests, processes, and assertions.

## Do not pass ORM models between test helpers. Fixtures, Processes, and Assertions should query repositories themselves. This keeps tests closer to real production behavior and avoids leaking implementation details.

## Freeze time in tests to ensure consistent, predictable results.

## Use data providers only to test meaningful variations of the same behavior, keep the data minimal and clearly named, and ensure failures can be traced to a specific data set.