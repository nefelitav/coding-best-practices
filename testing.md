## Assert against the entire array instead of individual items to catch unexpected additions. 

## To check that an exception is thrown in a test and then assert more things, add a try catch and add assertions inside catch. At the end of try, add self::fail. 

## In tests make assertions as generic as possible to make stronger checks, e.g. fetch the whole table to see what was created.

## Create a function to assert that a model is as expected. Add also the exclude option for some fields, e.g. id. Use a class like ProductTestAssertions.

## Fixtures must create only the data required for the test and must make intent explicit.

## Each test must follow a clear Arrange–Act–Assert structure and test one observable behavior.

## Use a Test Process class when tests involve repeated multi-step workflows, complex setup, or when you want scenario-style readability in integration tests.