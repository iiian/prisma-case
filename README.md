# prisma-case-format

`prisma introspect` names its model 1:1 with your database's conventions. Most of the time that is probably fine, however you may want or need different conventions in the generated client library. `prisma-case-format` makes it simple and direct to get the case conventions you want, applied across an entire `schema.prisma` with optional overrides per `model` or `field`.

## Use Cases
### As a one-time migration assistant

Did `prisma introspect` on your huge database schema mis-case all your tables & fields? Is it wrecking your hope of using duck-types? Use `--dry-run` in combination with `--(map)?-(table|field|enum)-case` to figure out which case conventions are correct for your project. Once things look correct, drop the flag to save changes to the specified `--file`, which is your local root `schema.prisma` by default.

### As a dedicated linter for your `schema.prisma` 

`prisma-case-format` aims to be idempotent, so you can use it to confirm that case conventions in your `schema.prisma` have not accidentally drifted. `prisma-case-format` can be applied on-commit or on-push, either in a git commit-hook package or as a CI/CD step. Use `--dry-run` to diff changes with the original file, or backup the original and compare to the edit.

### With `NextAuth.js`

If your team is using `NextAuth.js`, you may have encountered an issue where `prisma-case-format` steam rolls the strict data contract expected by the `NextAuth.js` integration. Specify the  `--uses-next-auth` flag in order to protect your `NextAuth.js` tables from your specified conventions.

### With varying conventions

If you inherited or were forced to produce a database schema that has deviations in case conventions, `prisma-case-format` can ensure these conventions remain stable. See the [config file](#config-file) section below.

## Usage

```bash
❯ prisma-case-format --help
Usage: prisma-case-format [options]

Give your schema.prisma sane naming conventions

Options:
  -f, --file <file>                cwd-relative path to `schema.prisma` file (default: "schema.prisma")
  -c, --config-file <cfgFile>      cwd-relative path to `.prisma-case-format` config file (default: ".prisma-case-format")
  -D, --dry-run                    print changes to console, rather than back to file (default: false)
  --table-case <tableCase>         case convention for table names (SEE BOTTOM) (default: "pascal")
  --field-case <fieldCase>         case convention for field names (default: "camel")
  --enum-case <enumCase>           case convention for enum names. In case of not declared, uses value of “--table-case”. (default: "pascal")
  --map-table-case <mapTableCase>  case convention for @@map() annotations (SEE BOTTOM)
  --map-field-case <mapFieldCase>  case convention for @map() annotations
  --map-enum-case <mapEnumCase>    case convention for @map() annotations of enums.  In case of not declared, uses value of “--map-table-case”.
  -p, --pluralize                  optionally pluralize array type fields (default: false)
  --uses-next-auth                 guarantee next-auth models (Account, User, Session, etc) uphold their data-contracts
  -V, --version                    hint: you have v2.1.0
  -h, --help                       display help for command
-------------------------
Supported case conventions: ["pascal", "camel", "snake", "constant"].
Additionally, append ',plural' after any case-convention selection to mark case convention as pluralized.
> For instance:
  --map-table-case=snake,plural

will append `@@map("users")` to `model User`.
Append ',singular' after any case-convention selection to mark case convention as singularized.
> For instance, 
  --map-table-case=snake,singular

will append `@@map("user")` to `model Users`

Deviant case conventions altogether:
> If one or more of your models or fields needs to opt out of case conventions, either to be a fixed name or to disable it,
use the `.prisma-case-format` file and read the documentation on "deviant name mappings".
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
  house_rating house_rating[]
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
model HouseRating {
  id       Int    @id @default(autoincrement())
  houseId  String @map("house_id")
  house    House  @relation(fields: [houseId], references: [id])
  ...

  @@map("house_rating")
}
...
model House {
  id           String        @id @default(uuid())
  houseRating HouseRating[]
  ...

  @@map("house")
}
...
```

## Drift Protection

`prisma-case-format` lets you manage three case conventions: `table`, `field`, and `enum`.

### `table`

```prisma
model Example { // <- controlled by `--table-case`
  id     String @id
  value1 String @map("value_1")
  @@map("example") // <- controlled by `--map-table-case`
}
```

Table conventions are controlled by the `--table-case` & `--map-table-case` flags. `--table-case` specifies the case convention for the models in the generated prisma client library. `--map-table-case` will manage the database name case convention. **`table` args manage models & views**.

### `field`

```prisma
model Example {
  id     String @id
  // r-- controlled by `--field-case`
  // |                   r-- controlled by `--map-field-case`
  // v                   v
  value1 String @map("value_1")
}
```

Field conventions are controlled by the `--field-case` & `--map-field-case` flags. `--field-case` specifies the case convention for the fields in models within the generated prisma client library. `--map-field-case`  will manage the case convention for the field in the database. **`field` args do not apply to enums**.

### `enum`

```prisma
enum Example { // <- controlled by  --enum-case
  Value1
  Value2
  @@map("example") // <- controlled by  --map-enum-case
}
```

Enum conventions are controlled by the `--enum-case` & `--map-enum-case` flags. `--enum-case` specifies the case convention for the enums in the generated prisma client library. `--map-enum-case` will manage the case  convention for the enum within the database. **`enum` args do not apply to enum values, just the model & database names**.

## Config file
`prisma-case-format` supports a config file, which primarily exists as a means to *override* case conventions per model or per field. This can be especially useful if you are coming from an existing database that doesn't have perfect naming convention consistency. For example, some of your model fields are `snake_case`, others are `camelCase`.

By default, this file is located at `.prisma-case-format`. The file is expected to be `yaml` internally. The following section details the config file format.

### Config file format

#### Property: `default?: string`:

An alternative to specifying the commandline arguments. The format is a simple `;`-delimited list, white space allowed.

##### Example: 
```yaml
default: 'table=pascal; mapTable=snake; field=pascal; mapField=snake; enum=pascal; mapEnum=pascal'
uses_next_auth: false
```

#### Property: `override?: Dictionary<string, { default?: string, field?: Field }>`

#### Type: `Field=Dictionary<string, string>`

Controls overrides on a per-model & per-field basis. Works for `models`, `views` and `enums`, in the same scope as the `table`/`field`/`enum` argument groups when running in commandline mode. Each key in `override` & subkey in `field` is allowed to be a regex, in which case it will attempt to match based on the specified pattern. This can be useful if you have several fields with prefixes or suffixes.

##### Example
```yaml
default: 'table=pascal; mapTable=snake; field=pascal; mapField=snake; enum=pascal; mapEnum=pascal'
override: 
  MyTable: 
    # don't fret about case convention here; 
    # prisma-case-format considers "MyTable" to be equivalent to "my_table" & "myTable"
    default: 'table=snake;mapTable=camel;field=snake;mapField=snake;enum=snake;mapEnum=snake'
    field:
      # same here. this is equivalent to "foo_field_on_my_table"
      fooFieldOnMyTable: 'field=pascal;mapField=pascal'
      'my_prefix.*': 'mapField=pascal;' # inherits field=snake from `MyTable.default`
```

Results in:

```prisma
model my_table {
  id                String @id
  FooFieldOnMyTable Integer
  my_prefix_prop_a  Json       @map("MyPrefixPropA")

  @@map("myTable")
}
```

#### Disabling case convention management

The `.prisma-case-format` file supports specifying that a particular model or field has its case convention management "disabled". This is achieved by setting the key under `override` or subkey under `field` to `'disable'`.

##### Example:

```yaml
default: ...
override:
  mYTaBlE: 'disable' # skip convention management for this table
  ...
  YourTable:
    default: '...'
    field:
      unmanaged_property: 'disable' # skip convention management for this field
```

Results in:

```prisma
model mYTaBlE {
  nowican                       String  @id
  cAnFAlL_iNtO_UtTeR__DiSrePaiR String  @map("lol")
  iN_T0T4L_p34C3                Integer @map("great")

  @map("myspace")
}
```

The other tables surrounding `mYTaBlE` will remain managed.

#### Deviant name mappings

In some cases, some users will want to let certain model or field names deviate out of using case-conventions. Primarily, this is useful if you're
inheriting ill-named tables in an at least partialy database-first workflow. 
Overriding case-convention use on a specific target name can be achieved in a `.prisma-case-format` file by using the syntax `map(Table|Field|Enum)=!<name>`,
where `<name>` is your precise **case-sensitive** name that you want to map.

##### Example

```yaml
default: ...
override:
  MyAmazingModel:
    default: 'mapTable=!Amaze_o'
    field:
      fuzz: 'mapField=!fizz_buzz'
  MySuperDuperEnum:
    default: 'mapTable=!myenum'
```

Results in:

```prisma
model MyAmazingModel {
  id Int @id
  fuzz String @map("fizz_buzz")

  @@map("Amaze_o")
}

enum MySuperDuperEnum {
  Opt1
  Opt2

  @@map("myenum")
}
```

#### Property: `uses_next_auth?: boolean`

If `=true`, then [the models added by `NextAuth.js`](https://authjs.dev/reference/adapter/prisma) will not have their case conventions rewritten by user's selected case conventions, preserving the expected contract. **If you are not using `NextAuth.js`, it is suggested to keep this unspecified or `false`.**
This property is equivalent to the following configuration:

```yaml
uses_next_auth: false
default: ...
override:
  Account:
    default: 'table=pascal; mapTable=pascal;'
    field:
      id:                'field=camel; mapField=camel'
      userId:            'field=camel; mapField=camel'
      type:              'field=camel; mapField=camel'
      provider:          'field=camel; mapField=camel'
      providerAccountId: 'field=camel; mapField=camel'
      refresh_token:     'field=snake; mapField=snake'
      access_token:      'field=snake; mapField=snake'
      expires_at:        'field=snake; mapField=snake'
      token_type:        'field=snake; mapField=snake'
      scope:             'field=snake; mapField=snake'
      id_token:          'field=snake; mapField=snake'
      session_state:     'field=snake; mapField=snake'
      user:              'field=snake; mapField=snake'
  Session:
    default: 'table=pascal; mapTable=pascal; field=camel; mapField=camel'
  User:
    default: 'table=pascal; mapTable=pascal; field=camel; mapField=camel'
  VerificationToken:
    default: 'table=pascal; mapTable=pascal; field=camel; mapField=camel'
```

Note that if `uses_next_auth: true` and your `overrides` section contains conflicting specifications for any of the above model names (`Account`, `Session`, `User`, or `VerificationToken`), `uses_next_auth` will blow away your configuration rules for those models and use the above rules.


## Pluralization

Supply the `-p` or `--pluralize` argument to get array pluralizations.

```bash
❯ prisma-case-format -p
✨ Done.
```

```prisma
// schema.prisma after pluralization
...
model House {
  ...
  houseRatings HouseRating[]
  ownerContacts String[] @map("owner_contact")
  ...
}
...
```
