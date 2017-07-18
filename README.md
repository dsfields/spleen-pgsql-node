# spleen-pgsql

The [`spleen`](https://www.npmjs.com/package/spleen) module provides high-level abstractions for dynamic filters.  This module will convert a `spleen` [`Filter`](https://www.npmjs.com/package/spleen#class-filter) into a string that is usable within a pgSQL statement's `WHERE` clause.

__Contents__
* [Usage](#usage)
* [API](#api)
* [Security Considerations](#security-considerations)
* [Stringify Behavior](#stringify-behavior)

## Usage

Add `spleen-pgsql` to your `package.json` file's `dependencies`:

```sh
$ npm install spleen-pgsql -S
```

Then use it in your code:

```js
const PgSql = require('spleen-pgsql');
const spleen = require('spleen');

const expression = '/foo eq 42 and (/bar/baz in [1,2,3] or /qux/quux eq 12)';
const filter = spleen.parse(expression);

const mappings = new Map();

mappings.set('/foo', {
  column: 'foo',
  isRequired: true,
});

mappings.set('/bar/baz', {
  column: 'bar_baz',
});

mappings.set('/qux/quux', {
  column: 'quux',
  identifier: 'qux',
});

const pgsqlWhere = PgSql.stringify(filter, mappings, { parameterize: true });

console.log(pgsqlWhere);
// "foo" = $1 AND ("bar_baz" IN ($2,$3,$4) or "qux"."quux" = $5)
```

## API

The `spleen-pgsql` module has a single class.

### Class: `PgSql`

Provides services for converting `spleen` filters into pgSQL.

  * __Properties__

    + `errors`: an object that contains references to the various possible errors thrown by `spleen-pgsql`.  This object has the following keys:

      - `RequiredFieldError`: thrown when a field that has been required is not present in the given `Filter`.

      - `StringifyError`: a general error thrown when `sleen-pgsql` is unable to convert a given `Filter` instance into a pgSQL statement.  This should generally never happen, and is here as a safeguard in the event a `Filter` instance is corrupted.

      - `UnmappedFieldError`: thrown when a field is encountered that not been white-listed by the `allow` option.

  * __Methods__

    + `PgSql.stringify(filter, [options])`: converts an instance of `spleen`'s `Filter`' class into a pgSQL statement.

      _Parameters_

      - `filter`: _(required)_ the instance of `Filter` to stringify.

      - `mappings`: _(required)_ an array of objects that map field targets to columns.  Each object can have the following fields:

        - `column`: a string specifying the column to which the field reference is being mapped.

        - `field`: an RFC 6901 JSON pointer string identifying the field target being mapped.  It is important to note that these references will be matched against `Target.prototype.field`, and, not necessarily, the entire path.

        - `identifier`: a string to use as the contextual identifier used with each column reference.  If this key is given a value, it will override `options.identifier`.
        
        - `isArray`: a Boolean value indicating whether or not the column is an array.  The default is `false`.

        - `isRequired`: a Boolean value indicating whether or not the field must be present in the `Filter`.  The default is `false`.

        Alternatively, the `mappings` option can be an ECMAScript [`Map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) where the key is `field`, and the value is an object with the same fields described above (only sans `field`).

      - `options`: _(optional)_ an object that controls various aspects of the stringification process.  This object can have the keys:

        - `identifier`: a string to use as a contextual identifier with each column reference.

        - `parameterize`: a Boolean value indicating whether or not literals in the `spleen` expression should be parameterized.  When `true` (the default), all string, number, and Boolean literals are set as numeric parameters.

      This method returns an object with the following keys:

      - `params`: an array of values, where the index of each entry corresponds to its `$#` placeholder in the filter statement.

      - `value`: a string containing the pgSQL filter statement.

## Security Considerations

It is highly recommended that you leave the `parameterize` option as `true` to help prevent SQL-injection attacks.

It is also highly recommend that you give leverge `spleen`'s `Filter.prototype.prioritize()` method before converting to a pgSQL expression.  This allows you to reorder a `Filter`, and optimally utilize known indexes.

## Stringify Behavior

There are situations where a `spleen` filter does not neatly translate into a pgSQL expression.  This is particularly true in the case of `spleen`'s `nil` operator.  For example, performing a greater-than on `nil` is technically valid with a `spleen` filter, but does not make much sense when translated to pgSQL.  The `PgSql.stringify()` method will make attempts to reconcile this:

* The operators `eq`, `lt`, `lte` when used with `nil` will result in an `IS NULL` pgSQL expression.

* The operators `neq`, `gt`, `gte` when used with `nil` will result in an `IS NOT NULL` pgSQL expression.
