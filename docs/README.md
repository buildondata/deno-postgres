# deno-postgres

![Build Status](https://img.shields.io/github/workflow/status/denodrivers/postgres/ci?label=Build&logo=github&style=flat-square)
[![Discord server](https://img.shields.io/discord/768918486575480863?color=blue&label=Ask%20for%20help%20here&logo=discord&style=flat-square)](https://discord.gg/HEdTCvZUSf)
![Manual](https://img.shields.io/github/v/release/denodrivers/postgres?color=orange&label=Manual&logo=deno&style=flat-square)
[![Documentation](https://img.shields.io/github/v/release/denodrivers/postgres?color=yellow&label=Documentation&logo=deno&style=flat-square)](https://doc.deno.land/https/deno.land/x/postgres@v0.8.0/mod.ts)
![License](https://img.shields.io/github/license/denodrivers/postgres?color=yellowgreen&label=License&style=flat-square)

```ts
import { Client } from "https://deno.land/x/postgres/mod.ts";

const client = new Client({
  user: "user",
  database: "test",
  hostname: "localhost",
  port: 5432,
});
await client.connect();

const array_result = await client.queryArray("SELECT ID, NAME FROM PEOPLE");
console.log(array_result.rows); // [[1, 'Carlos'], [2, 'John'], ...]

const object_result = await client.queryObject("SELECT ID, NAME FROM PEOPLE");
console.log(object_result.rows); // [{id: 1, name: 'Carlos'}, {id: 2, name: 'John'}, ...]

await client.end();
```

## Connection Management

### Connecting to DB

```ts
import { Client } from "https://deno.land/x/postgres/mod.ts";

let config;

// You can use the connection interface to set the connection properties
config = {
  applicationName: "my_custom_app",
  database: "test",
  hostname: "localhost",
  password: "password",
  port: 5432,
  user: "user",
  tls: {
    enforce: false,
  },
};

// Alternatively you can use a connection string
config =
  "postgres://user:password@localhost:5432/test?application_name=my_custom_app";

const client = new Client(config);
await client.connect();
await client.end();
```

The values required to connect to the database can be read directly from
environmental variables, given the case that the user doesn't provide them while
initializing the client. The only requirement for this variables to be read is
for Deno to be run with `--allow-env` permissions

The env variables that the client will recognize are taken from `libpq` to keep
consistency with other PostgreSQL clients out there (see
https://www.postgresql.org/docs/current/libpq-envars.html)

```ts
// PGUSER=user PGPASSWORD=admin PGDATABASE=test deno run --allow-net --allow-env --unstable database.js
import { Client } from "https://deno.land/x/postgres/mod.ts";

const client = new Client();
await client.connect();
await client.end();
```

### SSL/TLS connection

Using a database that supports TLS is quite simple. After providing your
connection parameters, the client will check if the database accepts encrypted
connections and will attempt to connect with the parameters provided. If the
connection is succesful, the following transactions will be carried over TLS.

However, if the connection fails for whatever reason the user can choose to
terminate the connection or to attempt to connect using a non-encrypted one.
This behavior can be defined using the connection parameter `tls.enforce` (not
available if using a connection string).

If set to true, the driver will fail inmediately if no TLS connection can be
established. If set to false the driver will attempt to connect without
encryption after TLS connection has failed, but will display a warning
containing the reason why the TLS connection failed. **This is the default
configuration**.

Sadly, stablishing a TLS connection in the way Postgres requires it isn't
possible without the `Deno.startTls` API, which is currently marked as unstable.
This is a situation that will be solved once this API is stabilized, however I
don't have an estimated time of when that might happen.

### Clients

You are free to create your clients like so:

```typescript
const client = new Client({
  ...
})
await client.connect()
```

### Pools

For stronger management and scalability, you can use **pools**:

```typescript
import { Pool } from "https://deno.land/x/postgres/mod.ts";
import { PoolClient } from "https://deno.land/x/postgres/client.ts";

const POOL_CONNECTIONS = 20;
const dbPool = new Pool({
  user: "user",
  password: "password",
  database: "database",
  hostname: "hostname",
  port: 5432,
}, POOL_CONNECTIONS);

async function runQuery(query: string) {
  const client: PoolClient = await dbPool.connect();
  const dbResult = await client.queryObject(query);
  client.release();
  return dbResult;
}

await runQuery("SELECT ID, NAME FROM users;"); // [{id: 1, name: 'Carlos'}, {id: 2, name: 'John'}, ...]
await runQuery("SELECT ID, NAME FROM users WHERE id = '1';"); // [{id: 1, name: 'Carlos'}, {id: 2, name: 'John'}, ...]
```

This improves performance, as creating a whole new connection for each query can
be an expensive operation. With pools, you can keep the connections open to be
re-used when requested using the `connect()` method. So one of the active
connections will be used instead of creating a new one.

The number of pools is up to you, but a pool of 20 is good for small
applications, this can differ based on how active your application is. Increase
or decrease where necessary.

## API

### Queries

#### Simple query

```ts
const result = await client.queryArray("SELECT ID, NAME FROM PEOPLE");
console.log(result.rows);
```

#### Prepared statement

```ts
{
  const result = await client.queryArray(
    "SELECT ID, NAME FROM PEOPLE WHERE AGE > $1 AND AGE < $2",
    10,
    20,
  );
  console.log(result.rows);
}

{
  // equivalent using QueryConfig interface
  const result = await client.queryArray({
    text: "SELECT ID, NAME FROM PEOPLE WHERE AGE > $1 AND AGE < $2",
    args: [10, 20],
  });
  console.log(result.rows);
}
```

#### Prepared statement with template strings

```ts
{
  const result = await client.queryArray
    `SELECT ID, NAME FROM PEOPLE WHERE AGE > ${10} AND AGE < ${20}`;
  console.log(result.rows);
}

{
  const min = 10;
  const max = 20;
  const result = await client.queryObject
    `SELECT ID, NAME FROM PEOPLE WHERE AGE > ${min} AND AGE < ${max}`;
  console.log(result.rows);
}
```

##### Why use template strings?

Template string queries get executed as prepared statements, which protects your
SQL against injection to a certain degree (see
https://security.stackexchange.com/questions/15214/are-prepared-statements-100-safe-against-sql-injection).

Also, they are easier to write and read than plain SQL queries and are more
compact than using the `QueryOptions` interface

For example, template strings can turn the following:

```ts
await client.queryObject({
  text: "SELECT ID, NAME FROM PEOPLE WHERE AGE > $1 AND AGE < $2",
  args: [10, 20],
});
```

Into a much more readable:

```ts
await client.queryObject
  `SELECT ID, NAME FROM PEOPLE WHERE AGE > ${10} AND AGE < ${20}`;
```

However, a limitation of template strings is that you can't pass any parameters
provided by the `QueryOptions` interface, so the only options you have available
are really `text` and `args` to execute your query

### Generic Parameters

Both the `queryArray` and `queryObject` functions have a generic implementation
that allow users to type the result of the query

```typescript
{
  const array_result = await client.queryArray<[number, string]>(
    "SELECT ID, NAME FROM PEOPLE WHERE ID = 17",
  );
  // [number, string]
  const person = array_result.rows[0];
}

{
  const array_result = await client.queryArray<[number, string]>
    `SELECT ID, NAME FROM PEOPLE WHERE ID = ${17}`;
  // [number, string]
  const person = array_result.rows[0];
}

{
  const object_result = await client.queryObject<{ id: number; name: string }>(
    "SELECT ID, NAME FROM PEOPLE WHERE ID = 17",
  );
  // {id: number, name: string}
  const person = object_result.rows[0];
}

{
  const object_result = await client.queryObject<{ id: number; name: string }>
    `SELECT ID, NAME FROM PEOPLE WHERE ID = ${17}`;
  // {id: number, name: string}
  const person = object_result.rows[0];
}
```

### Object query

The `queryObject` function allows you to return the results of the executed
query as a set objects, allowing easy management with interface like types.

```ts
interface User {
  id: number;
  name: string;
}

const result = await client.queryObject<User>(
  "SELECT ID, NAME FROM PEOPLE",
);

// User[]
const users = result.rows;
```

However, the actual values of the query are determined by the aliases given to
those columns inside the query, so executing something like the following will
result in a totally different result to the one the user might expect

```ts
const result = await client.queryObject(
  "SELECT ID, SUBSTR(NAME, 0, 2) FROM PEOPLE",
);

const users = result.rows; // [{id: 1, substr: 'Ca'}, {id: 2, substr: 'Jo'}, ...]
```

To deal with this issue, it's recommended to provide a field list that maps to
the expected properties we want in the resulting object

```ts
const result = await client.queryObject(
  {
    text: "SELECT ID, SUBSTR(NAME, 0, 2) FROM PEOPLE",
    fields: ["id", "name"],
  },
);

const users = result.rows; // [{id: 1, name: 'Ca'}, {id: 2, name: 'Jo'}, ...]
```

**Don't use TypeScript generics to map these properties**, this generics only
exist at compile time and won't affect the final outcome of the query

```ts
interface User {
  id: number;
  name: string;
}

const result = await client.queryObject<User>(
  "SELECT ID, SUBSTR(NAME, 0, 2) FROM PEOPLE",
);

const users = result.rows; // TypeScript says this will be User[]
console.log(rows); // [{id: 1, substr: 'Ca'}, {id: 2, substr: 'Jo'}, ...]

// Don't trust TypeScript :)
```

Other aspects to take into account when using the `fields` argument:

- The fields will be matched in the order they were declared
- The fields will override any alias in the query
- These field properties must be unique (case insensitive), otherwise the query
  will throw before execution
- The fields must match the number of fields returned on the query, otherwise
  the query will throw on execution

```ts
{
  // This will throw because the property id is duplicated
  await client.queryObject(
    {
      text: "SELECT ID, SUBSTR(NAME, 0, 2) FROM PEOPLE",
      fields: ["id", "ID"],
    },
  );
}

{
  // This will throw because the returned number of columns don't match the
  // number of defined ones in the function call
  await client.queryObject(
    {
      text: "SELECT ID, SUBSTR(NAME, 0, 2) FROM PEOPLE",
      fields: ["id", "name", "something_else"],
    },
  );
}
```
