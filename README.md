# PHP RFC: Extend error control operator to suppress exceptions
  * Version: 0.1
  * Date: 2020-02-27
  * Author: George Peter Banyard, <girgias@php.net>
  * Status: Draft
  * First Published at: http://wiki.php.net/rfc/error-control-operator-exceptions

## Introduction

Prior to the introduction to exceptions to the PHP language many functions/extensions used diagnostic messages such a `E_WARNING` or `E_NOTICE` while returning `false` to convey errors.

It is however near to impossible to provide a smooth upgrade path for converting such diagnostic messages to the use of exceptions, which would mean redesigning and reimplementing many functions provided by PHP.

PHP's modern way to signal errors is via `Throwable` errors which is comprised of a parallel hierarchy, `Error` which are mostly emitted by the engine to signal programming error bugs, and `Exception` which are there to signal unusual situations from which the program can recover.

## Proposal

Extend the error control operator `@` to be able to suppress `Throwable` errors which are an instance of `Exception` emitted by the expression it precedes, on top of suppressing diagnostic messages.

Add the following syntax `@<union_class_list>` to suppress any `Throwable` error which is an instance of a class within the class list, without suppressing diagnostic messages. This is equivalent to:
```php
try {
    expression
} catch (union_class_list) {}
```

Where `union_class_list` refers to any type union of `Throwable` classes, for example:
```
$result = @<ExceptionOne|ExceptionTwo>foo() ?? $default;
```
Is equivalent to
```
try {
	$result = foo();
} catch (ExceptionOne|ExceptionTwo) {
	$result = $default;
}
```

The value of the expression when it encounters an exception will be `null`,
with the exception that internal functions can set the return value to another simple type (`true`, `false`, `int`, `float`) using the `RETVAL_*` macros.
This is done for backwards compatibility reasons such that a function which returned `false`
on error will still do so even after it has been converted to throw an exception.

## Rationale

The decision to only suppress `Exception` and not `Throwable` is because instances of `Error` should *never* be caught as they signal a programming error.

As stated previously many functions in PHP which have existed for a long time use diagnostic messages to indicate failure states, but if they would be designed today they would most likely use an exception instead.
The main area where this is true are functions which relate to I/O and the idiomatic use is to
prefix the function call with `@`, e.g. `$ret = @fopen()` and then check if the return value is identical to false (``if ($ret === false)``).

A drastic change to throwing exceptions is unacceptable as 99% of PHP code which has been written would be broken, and short of writing a new implementation for all functions which could use exceptions instead of emitting diagnostic we would be stuck with the status quo.

It should be noted that just this proposal is insufficient for migrating away from diagnostic messages,
indeed code dealing with such functions could change the `error_reporting` INI directive,
or use a custom `error_handler` specific to this section before restoring the previous one.

But, if the `throw_on_error` declare RFC [1](https://github.com/Girgias/php-rfc-throw-on-error-declare) is accepted and added (via a new declare statement or as part of a PHP edition [2](https://github.com/php/php-rfcs/pull/2)), this would provide an additional way for incremental migration away from diagnostic messages and to exceptions.

## Alternative proposals

The recurrent issue with any of the alternative proposals is that their require old code to be updated, or will remain opt-in instead of becoming the default, or are orthogonal by proposing new mechanisms to handle errors.

### Converting diagnostics to exceptions

This proposal was mostly being floated around during the RFC discussion for adding Attributes [3](https://wiki.php.net/rfc/attributes_v2) to PHP as it would allow to use `@` as the attribute sigil.

This is the most disruptive proposal out of any as it's a complete backwards incompatible break which would be similar to the transition between Python 2 and 3, and is unsound.

This requires that any code written in the last 25 years would need to be updated just to cater to this change, which is in stark contrast with PHP's typical backwards compatibility philosophy.

Moreover, if any such conversion is missed, or one want to convert some different diagnostics to exception at later date, the same situation would arise once again.

As such we deem this proposal as unacceptable.

### Using an attribute

It would very likely look akin to `#[Suppress(ConcreteException::class)] expression;`, which we consider a very odd choice as attributes [3] are meant to represent metadata and not affect control flow.

We believe `@<ConcreteException>` is more semantic in error handling as `@` is already named and knows as the "error control operator".

### Creating a new implementation

One opinion is that it's better to create a new implementation instead of trying to fix the old one.
On top of the documentation an implementation maintenance burden added upon the PHP project this requires people to migrate and use the newly provided API, which has turned out to be largely ineffective as the I/O API has been revamped within the SPL extension but the usage of the traditional I/O functions remains the de facto standard within the community.

Although a reasonable proposal on a case by case basis, we deem this proposal not generic enough to handle all cases within the PHP ecosystem.

### Go-style returns

Error handling in Golang [4](https://blog.golang.org/error-handling-and-go) is achieved by having two return values, where one is the "real" return value and the secondary one is populated when an error is encountered.

In PHP this could look something akin to:
```php
$fp, $err = fopen('file-path', 'r');

if ($err !== null) {
	// handle error
}

// do things with $fp
```

Where `$err` would provide the error information regardless if it is a diagnostic message (e.g. `E_WARNING`) or a `Throwable` error.

We consider this an excellent orthogonal proposal which makes it more reasonable to do error handling without using exceptions for these functions, which is a style which can be more appropriate for low level code.
However, in our opinion extending the error control operator is still a necessary step.

### Optional trailing out parameter

This proposal is a different take on how to improve error handling by passing an optional out parameter,
this proposal is still a draft and can be found on GitHub [5](https://github.com/Danack/RfcErrorOrExceptionChooseWisely/blob/master/rfc_words.md).

As this is a similar proposal to Go-style returns we consider this orthogonal to this proposal.


## Backward Incompatible Changes

Any expression which threw a `Throwable` error that happens to be prefixed with the error control operator (`@`) would now be suppress and execution of the script would continue whereas previously it would halt.
We deem this to be unlikely and as such a reasonable compromise for introducing this feature which can improve the long term health of the project.

## Proposed PHP Version

Next minor version, i.e. PHP 8.1.

## RFC Impact 

### To Opcache

TBD

## Open Issues

None currently.

## Unaffected PHP Functionality

This RFC does not change any `false` returns which emits a diagnostic message to exceptions,
this is left as a future scope and on a case by case basis.

## Future Scope

 - Promote diagnostic message error conditions to exceptions
 - `throw_on_error` declare [1]
 - PHP editions [2]

## Proposed Voting Choices

As per the voting RFC a yes/no vote with a 2/3 majority is needed for this proposal to be accepted.

## Patches and Tests

Prototype patch (partially complete): <https://github.com/php/php-src/compare/master...Girgias:error-silence-exception-adding-try-catch-opcodes>

### Current implementation detail

"Virtual" catch blocks (using the `ZEND_CATCH` opcode) and a new `ZEND_SILENCE_CATCH` opcode are appended prior to `ZEND_END_SILENCE` OPCode

#### Known issues

Memory leaks when a function argument which throws an exception is suppressed or `new Object()` where the constructor throws, and an OPcache test hangs.

## Implementation 
After the project is implemented, this section should contain
  - the version(s) it was merged into
  - a link to the git commit(s)
  - a link to the PHP manual entry for the feature
  - a link to the language specification section (if any)

## References

[1]: `throw_on_error` PHP RFC: <https://github.com/Girgias/php-rfc-throw-on-error-declare>  
[2]: Language evolution overview proposal: <https://github.com/php/php-rfcs/pull/2>  
[3]: Attributes V2 RFC: <https://wiki.php.net/rfc/attributes_v2>  
[4]: Error handling in Golang <https://blog.golang.org/error-handling-and-go>  
[5]: "Improved error handling" draft RFC <https://github.com/Danack/RfcErrorOrExceptionChooseWisely/blob/master/rfc_words.md>  


## Rejected Features 
Keep this updated with features that were discussed on the mail lists.
