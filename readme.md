# PHP RFC: MySQLi Execute Query

* Version: 1
* RFC Started: 2022-04-21
* RFC Updated: 2022-05-11
* Voting Start: 2022-05-11 15:00 UTC / 16:00 BST
* Voting End: 2022-05-25 15:00 UTC / 16:00 BST
* Author: Kamil Tekiela, and Craig Francis [craig#at#craigfrancis.co.uk]
* Status: Accepted
* First Published at: https://wiki.php.net/rfc/mysqli_execute_query
* GitHub Repo: https://github.com/craigfrancis/php-mysqli-execute-query-rfc
* Implementation: [From Kamil Tekiela](https://github.com/php/php-src/compare/master...kamil-tekiela:execute_query) (proof of concept)

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

$name = '%a%';
$type1 = 1; // Admin
$type2 = 2; // Editor
```

Traditionally someone might use escaping, which is [very error prone](https://github.com/craigfrancis/php-is-literal-rfc/blob/main/justification/escaping.php?ts=4), e.g.

```php
foreach ($db->query('SELECT * FROM user WHERE name LIKE "' . $db->real_escape_string($name) . '" AND type_id IN (' . $db->real_escape_string($type1) . ', ' . $db->real_escape_string($type2) . ')') as $row) { // INSECURE
    print_r($row);
}
```

To avoid mistakes, parameterised queries should be used (with a [literal-string](https://eiv.dev/)), but can be fairly complex:

```php
$statement = $db->prepare('SELECT * FROM user WHERE name LIKE ? AND type_id IN (?, ?)');
$statement->bind_param('sii', $name, $type1, $type2);
$statement->execute();

foreach ($statement->get_result() as $row) {
    print_r($row);
}
```

Since PHP 8.1, we no longer have problems with binding by reference, or needing to specify the variable types via the first argument to `bind_param()`, e.g.

```php
$statement = $db->prepare('SELECT * FROM user WHERE name LIKE ? AND type_id IN (?, ?)');
$statement->execute([$name, $type1, $type2]);

foreach ($statement->get_result() as $row) {
    print_r($row);
}
```

This proposed function will simplify this even further, by allowing developers to write this in a one line foreach:

```php
foreach ($db->execute_query('SELECT * FROM user WHERE name LIKE ? AND type_id IN (?, ?)', [$name, $type1, $type2]) as $row) {
    print_r($row);
}
```

In pseudo-code it's basically:

```php
function mysqli_execute_query(mysqli $mysqli, string $sql, array $params = null)
{
    $driver = new mysqli_driver();

    $stmt = $mysqli->prepare($sql);
    if (!($driver->report_mode & MYSQLI_REPORT_STRICT) && $mysqli->error) {
        return false;
    }

    $stmt->execute($params);
    if (!($driver->report_mode & MYSQLI_REPORT_STRICT) && $stmt->error) {
        return false;
    }

    return $stmt->get_result();
}
```

## Notes

### Function Name

The name was inspired by [Doctrine\DBAL\Connection::executeQuery()](https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/data-retrieval-and-manipulation.html#executequery).

### Returning false

The implementation is effectively calling [mysqli_stmt_get_result()](https://www.php.net/mysqli_stmt_get_result) last. While it will return `false` on failure, it will also return `false` for queries that do not produce a result set (e.g. `UPDATE`). Historically this has been addressed by using `mysqli_errno()`, but since 8.1 the [Change Default mysqli Error Mode RFC](https://wiki.php.net/rfc/mysqli_default_errmode) was accepted, and Exceptions are used by default.

### Properties

Because [mysqli_stmt](https://www.php.net/manual/en/class.mysqli-stmt.php) is not returned, it's not possible to use its properties directly:

- int|string **$affected_rows** - use `$mysqli->affected_rows` or `mysqli_affected_rows($mysqli)`
- int|string **$insert_id** - use `$mysqli->insert_id` or `mysqli_insert_id($mysqli)`
- int|string **$num_rows** - also available on `mysqli_result`
- int **$param_count**
- int **$field_count** - also available on `mysqli_result`
- int **$errno** - use `mysqli_errno($mysqli)`, `$mysqli->errno`
- string **$error** - use `mysqli_error($mysqli)`, `$mysqli->error`
- array **$error_list** - use `mysqli_error_list($mysqli)`, `$mysqli->error_list`
- string **$sqlstate** - use `mysqli_sqlstate($mysqli)`, `$mysqli->sqlstate`
- int **$id**

It's also worth noting the error property usage will hopefully reduce, as more developers use `mysqli_sql_exception` for errors (because the mysqli Error Mode now defaults to `MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT`).

### Re-using Statements

The implementation discards the `mysqli_stmt` object immediately, so you cannot re-issue a statement with new parameters. Anyone who would benefit from this (to skip running prepare again), can still use `mysqli_prepare()`.

### Updating Existing Functions

Cannot change `mysqli_query()` because its second argument is `$resultmode`.

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

None

## Unaffected PHP Functionality

N/A

## Future Scope

N/A

## Voting

Accept the RFC

<doodle title="mysqli_execute_query" auth="craigfrancis" voteType="single" closed="true">
   * Yes
   * No
</doodle>

## Implementation

[From Kamil Tekiela](https://github.com/php/php-src/compare/master...kamil-tekiela:execute_query) (proof of concept)

This implementation copies some details to the mysqli object, but not the affected rows. This means `mysqli_affected_rows($mysqli)` and `$mysqli->affected_rows` will currently return -1.

## References

N/A

## Rejected Features

None
