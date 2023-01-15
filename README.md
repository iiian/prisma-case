# prisma-case-format

As of `prisma@2.19.0`, `prisma introspect` will name its model 1:1 with your database's conventions. That's great, but if your db is purely `snake_case`, or _worse_ if your tables lack any common convention, it can make the autogenerated client code less pleasant to consume. `prisma-case-format` will format your table and field names for every model to enforce a more conventional experience.

## Usage

```bash
❯ prisma-case-format --help
Usage: prisma-case-format [options]

Give your schema.prisma sane naming conventions

Options:
  --file <file>             schema.prisma file location (default: "schema.prisma")
  --table-case <tableCase>  case convention for table names. allowable values: "pascal", "camel", "snake" (default: "pascal")
  --field-case <fieldCase>  case convention for field names. allowable values: "pascal", "camel", "snake" (default: "camel")
  -D, --dry-run             print changes to console, rather than back to file (default: false)
  -h, --help                display help for command
```

## Example

```prisma
// schema.prisma before
...
model house_rating {
  id       Int    @id @default(autoincrement())
  house_id String
  house    house  @relation(fields: [house_id], references: [id])
  ...
}
...
model house {
  id  String  @id @default(uuid())
  house_ratings house_rating[]
  ...
}
...
```

```bash
❯ prisma-case-format
✨ Done.
```

```prisma
// schema.prisma after
...
model HouseRatings {
  id       Int    @id @default(autoincrement())
  houseId  String @map("house_id")
  house    House  @relation(fields: [houseId], references: [id])
  ...

  @@map("house_ratings")
}
...
model House {
  id           String        @id @default(uuid())
  houseRatings HouseRating[]
  ...

  @@map("house")
}
...
```
