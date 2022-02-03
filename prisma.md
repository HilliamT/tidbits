# 💎 Prisma

## Overview
- [Create If Not Exists](#create-if-not-exists)
    - [Native CRUD Functions](#native-crud-functions)
    - [Interactive Transactions](#interactive-transactions)
    - [Upsert in One](#upsert-in-one)


## Create If Not Exists

### Native CRUD Functions
Creating a database record only if it doesn't exist already is a functionality that may be useful in some cases. For example, if a user has yet to log in before performing a certain action, we can create a new user record in the database on the fly rather than having a separate hook for this.

```typescript
const user = await prisma.user.findUnique({
  where: {
    username: "johndoe",
  }
});

if (user == null) {
    user = await prisma.user.create({
        data: {
            username: "johndoe",
            email: "johndoe@example.com",
        }
    });
}

// ASSERT: user != null;
// do something with the user object...
```

We note that these are two separate interactions with the database. For an application that frequently will update data records, it is very possible for two executions running at the same time to receive the result of `null` for the `user` variable, and then both attempt to create a new user record.

As a result, the slower execution will fail to create a new user record and continue its necesasry processing, as the first execution will have already created it.

### Interactive Transactions
Prisma supports transactions, particularly interactive transactions that allows control for conditionally running CRUD operations based upon prior results.

A transaction is a user-defined set of operations that the database will execute synchronously one after another, without the interference of other database operations that may be running at the same time on the same data records.

This can be enabled in Prisma by enabling the preview feature within `schema.prisma`.

```
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["interactiveTransactions"]
}
```

The `prisma.$transaction` function can then take in a callback that is to be ran within one single database transaction.
```typescript
await prisma.$transaction(async () => {
    const user = await prisma.user.findUnique({
        where: {
            username: "johndoe",
        }
    });

    if (user == null) {
        user = await prisma.user.create({
            data: {
                username: "johndoe",
                email: "johndoe@example.com",
            }
        });
    }

    // ASSERT: user != null;
    // do something with the user object...
});
```

[🔗 Source: Transactions and batch queries (Reference) | Prisma Docs](https://www.prisma.io/docs/concepts/components/prisma-client/transactions#interactive-transactions-in-preview)

### Upsert in One
Another way to avoid a race condition in the above scenario is to use `upsert`. The `upsert` function is a convenience function that allows you to create or update a record in one operation.

```typescript
const user = await prisma.user.upsert({
    where: {
        username: "johndoe",
    },
    update: {},
    create: {
        username: "johndoe",
        email: "johndoe@example.com",
    },
});

// ASSERT: user != null;
// do something with the user object...
```

This eliminates the race condition, whilst ensuring that there is a user record within the database. No updates will be made to the user database record if it already exists.

[🔗 Source: Create if not exists · Discussion #5815 · prisma/prisma](https://github.com/prisma/prisma/discussions/5815#discussioncomment-404334)