# 8-Character Code Generator

Turns sequential numbers into unique, random-looking 8-character codes using only letters (A-Z, a-z) and digits (0-9). Every code can be decoded back into its original number.

```
1      ->  AScOjQxW
2      ->  AkDwVWXP
12345  ->  (try it in the demo)
```

No randomness, no database lookups, no collision checks. Uniqueness is guaranteed by the math.

## Why

Random code generators can only make duplicates unlikely, not impossible. With 8 characters from a 62-character set, a random generator hits a 50% chance of at least one duplicate after roughly 17 million codes. The usual fix is to check every new code against stored data and retry on conflict.

This approach skips that entirely. You feed it a number that is already unique (a database identity column, a sequence, an auto-increment counter) and it maps that number to exactly one code. Two different numbers can never produce the same code.

## How it works

1. Take a unique number `id`.
2. Scramble it with `(id * MULT + OFFSET) mod 62^8`.
3. Convert the result to base 62 and pad it to 8 characters.

Because `MULT` shares no factors with `62^8`, step 2 is a one-to-one shuffle of every number from `0` to `62^8 - 1`. Decoding runs the same steps in reverse using the modular inverse of `MULT`.

## Capacity

| Item | Value |
|---|---|
| Character set | `A-Z a-z 0-9` (62 characters) |
| Code length | 8 |
| Total unique codes | 218,340,105,584,896 (62^8) |
| Valid input range | 0 to 218,340,105,584,895 |

The upper limit is well below JavaScript's `Number.MAX_SAFE_INTEGER`, so decoded IDs are returned as regular numbers.

## Usage

Copy the core logic block from the demo page (between the `Core logic` and `End of core logic` comments) into your project.

```javascript
const code = encodeId(1);       // "AScOjQxW"
const id = decodeCode(code);    // 1
```

`encodeId` accepts a `Number`, a `BigInt`, or a numeric string. That last one matters because some database drivers return `BIGINT` columns as strings.

### Getting the unique number

Any source of unique integers works. In SQL Server, for example:

```sql
CREATE SEQUENCE dbo.CodeSeq
    AS BIGINT
    START WITH 1
    INCREMENT BY 1;

SELECT NEXT VALUE FOR dbo.CodeSeq;
```

An existing identity column on the table the code belongs to works just as well.

### Storing codes is optional

Since codes decode back to their ID, you can look up a record straight from a code without storing the code at all. Store it anyway if you want to index or search by code directly.

## API

### `encodeId(id)`

Returns the 8-character code for `id`.

Throws `RangeError` if `id` is negative or larger than 218,340,105,584,895.

### `decodeCode(code)`

Returns the number that `code` was generated from.

Throws `Error` if `code` is not exactly 8 letters or digits. Codes are case-sensitive.

## Configuration

```javascript
const CHARSET = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
const MULT = 999999999989n;     // must be odd and not divisible by 31
const OFFSET = 48271234567n;    // any value below 62^8
```

To get a different-looking sequence for your own project, change `MULT` and `OFFSET` before you issue any codes. `MULT` must be coprime with `62^8`, which just means it can't be divisible by 2 or 31. Any prime other than those two works.

**Once codes are live, never change `CHARSET`, `MULT`, or `OFFSET`.** Changing any of them makes every existing code decode to a different number.

## Demo

Open `code-generator-demo.html` in a browser. It includes:

- A counter that issues the next code on each click, simulating a database sequence
- Encode and decode fields for testing any number or code
- A uniqueness test that encodes up to 500,000 numbers, checks for duplicates, and confirms every code decodes back correctly
- A log of issued codes and the current settings

The page has no dependencies beyond Google Fonts and runs entirely in the browser.

## Security note

This is an obfuscation scheme, not encryption. Someone with enough consecutive codes could work out the pattern and predict others. It's a good fit for:

- Order and invoice numbers
- Reference or tracking codes
- Short links
- Public-facing IDs that shouldn't reveal record counts at a glance

Don't use it for anything that needs to stay secret, such as password reset tokens, API keys, or session IDs. For those, use a cryptographically random generator with a uniqueness check.

Also run encoding on the server. If `MULT` and `OFFSET` ship in browser code, anyone can read them and reverse the mapping.
