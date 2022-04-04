# PHP RFC: MySQLi Execute Query

* Version: 0.1
* Voting Start: ?
* Voting End: ?
* RFC Started: 2022-04-04
* RFC Updated: 2022-04-04
* Author: Kamil Tekiela, and Craig Francis [craig#at#craigfrancis.co.uk]
* Status: Draft
* First Published at: https://wiki.php.net/rfc/mysqli_execute_query
* GitHub Repo: https://github.com/craigfrancis/php-mysqli-execute-query-rfc
* Implementation: [From Kamil Tekiela](https://github.com/php/php-src/compare/master...kamil-tekiela:execute_query)

## Introduction

Make parameterised MySQLi queries easier, with `mysqli_execute_query($sql, $params)`.

This will further reduce the complexity of using parameterised queries - making it easier for developers to move away from `mysqli_query()`, and the dangerous/risky escaping of user values.

## Proposal

This new function is a simple combination of:

- `mysqli_prepare()`
- `mysqli_execute()`
- `mysqli_stmt_get_result()`

It follows the original Draft RFC [MySQLi Execute with Parameters](https://wiki.php.net/rfc/mysqli_execute_parameters), which proposed a single function to execute a parameterised query; and the Implemented RFC [mysqli bind in execute](https://wiki.php.net/rfc/mysqli_bind_in_execute), which addressed the difficulties with `bind_param()`.

Using an example, assume we start with:

```php
$db = new mysqli('localhost', 'user', 'password', 'database');

$sql = 'SELECT * FROM user WHERE name LIKE ? AND type IN (?, ?)';

$name = '%a%';
$type1 = 'admin';
$type2 = 'editor';
```

Before PHP 8.1, a typical parameterised query would look like this:

```php
$statement = $db->prepare($sql);
$statement->bind_param('sss', $name, $type1, $type2);
$statement->execute();

$result = $statement->get_result();

while ($row = $result->fetch_assoc()) {
    print_r($row);
}
```

Since PHP 8.1, we no longer have problems with binding by reference, or needing to specify the variable types via the first argument to `bind_param()`, e.g.

```php
$statement = $db->prepare($sql);
$statement->execute([$name, $type1, $type2]);

$result = $statement->get_result();

while ($row = $result->fetch_assoc()) {
    print_r($row);
}
```

The proposed function will simply this even further, by allowing developers to write:

```php
$result = $db->execute_query($sql, [$name, $type1, $type2]);

while ($row = $result->fetch_assoc()) {
    print_r($row);
}
```

## Notes

### Function Name

The name was inspired by [Doctrine\DBAL\Connection::executeQuery()](https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/data-retrieval-and-manipulation.html#executequery).

### Returning false

Because the implementation is effectively calling [mysqli_stmt_get_result()](https://www.php.net/mysqli_stmt_get_result) last, while it will return `false` on failure, it will also return `false` for queries that do not produce a result set (e.g. `UPDATE`). Historically this has been addressed by using `mysqli_errno()`, but since 8.1 the [Change Default mysqli Error Mode RFC](https://wiki.php.net/rfc/mysqli_default_errmode) so Exceptions are used by default.

### Re-using Statements

The implementation discards the `mysqli_stmt` object immediately, so you cannot re-issue a statement with new parameters. This is a rarely used feature, and anyone who would benefit from this (to skip running prepare again), can still use `mysqli_prepare()`.

### Updating Existing Functions

Cannot change `mysqli_query()` because it's second argument is `$resultmode`.

Cannot replace the deprecated `mysqli_execute()` function, which is an alias for `mysqli_stmt_execute()`, because it would create a backwards compatibility issue.

### Why Now

Because the [Remove support for libmysql from mysqli RFC](https://wiki.php.net/rfc/mysqli_support_for_libmysql) has been accepted, it makes it much easier to implement with `mysqlnd`.

## Backward Incompatible Changes

None

## Proposed PHP Version(s)

PHP 8.2

## RFC Impact

### To SAPIs

None known

### To Existing Extensions

- mysqli, adding a new function.

### To Opcache

None known

### New Constants

None

### php.ini Defaults

None

## Open Issues

### Affected Rows

Currently `$mysqli->affected_rows` and `mysqli_affected_rows($mysqli)` returns -1.

### Properties

Because `mysqli_stmt` is not returned, it's not possible to use its properties: `affected_rows`, `insert_id`, `errno`, `sqlstate`, etc.

`num_rows` and `field_count` are already available on the returned `mysqli_result`.

`insert_id`, you can use `$mysqli->insert_id` or `mysqli_insert_id($mysqli)`.

Errors, which are now handled by exceptions (by default), can also be retrieved via `mysqli_errno($mysqli)`, `$mysqli->errno`, etc.

## Unaffected PHP Functionality

N/A

## Future Scope

N/A

## Voting

Accept the RFC

TODO

## Implementation

[From Kamil Tekiela](https://github.com/php/php-src/compare/master...kamil-tekiela:execute_query)

## References

N/A

## Rejected Features

TODO
