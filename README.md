# PHP RFC: Extend error control operator to suppress exceptions
  * Version: 0.1
  * Date: 2020-02-27
  * Author: George Peter Banyard, <girgias@php.net>
  * Status: Draft
  * First Published at: http://wiki.php.net/rfc/error-control-operator-exceptions

## Introduction

Prior to the introduction to exceptions to the PHP language many functions/extensions used diagnostic messages such a `E_WARNING` or `E_NOTICE` while returning `false` to convey errors.

It is however near to impossible to provide a smooth upgrade path for converting such diagnostic messages to the use of exceptions, which would mean redesigning and reimplementing many functions provided by PHP.

## Proposal

Extend the error control operator `@` to be able to suppress any exception emitted by the expression it precedes.

Add the following syntax `@<union_class_list>` to suppress only the exceptions within the class list, this is similar to:
```php
try {
    expression
} catch (union_class_list) {}
```

As such `@<\Throwable>expression` and `@expression` are identical.

The value of the expression when it encounters an exception will be `null`,
with the exception that internal functions can set the return value to another simple type (`true`, `false`, `int`, `float`) using the `RETVAL_*` macros.
This is done for backwards compatibility reasons such that a function which returned `false`
on error will still do so even after it has been converted to throw an exception.

## Rationale

As stated previously many functions in PHP which have existed for a long time use diagnostic messages to indicate failure states, but if they would be designed today they would most likely use an exception instead.
The main area where this is true are functions which relate to I/O and the idiomatic use is to
prefix the function call with `@`, e.g. `$ret = @fopen()` and then check if the return value is identical to false (``if ($ret === false)``).

A drastic change to throwing exceptions is unacceptable as 99% of PHP code which has been written would be broken, and short of writing a new implementation for all functions which could use exceptions instead of emitting diagnostic we would be stuck with the status quo.

It should be noted that just this proposal is insufficient for migrating away from diagnostic messages,
indeed code dealing with such functions could change the `error_reporting` INI directive,
or use a custom `error_handler` specific to this section before restoring the previous one.

But, if the `throw_on_error` declare RFC [1](https://github.com/Girgias/php-rfc-throw-on-error-declare) is accepted and added (via a new declare statement or as part of a PHP edition [2](https://github.com/php/php-rfcs/pull/2)), this would provide an additional way for incremental migration away from diagnostic messages and to exceptions.

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

## Rejected Features 
Keep this updated with features that were discussed on the mail lists.
