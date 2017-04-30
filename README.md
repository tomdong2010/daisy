# Daisy - a blockchain where blocks are SQLite databases

## Design notes

*Note:* All this is fluid and can be changed as development progresses.

* Block payloads are SQLite database files. Opened read-only for use, of course.
* Blockchain metadata is mostly separate from the block payloads, with the notable exception of the previous block hash (merkle).
* Consensus rules:
    * The validity of the SQLite files
    * The presence of a special tables named `_meta` and `keys`,
    * The validity of the previous block hash in the `_meta` table
    * The previous block hash is signed with a key which is present in the previous blocks' `_keys` table.
    * The `_keys` table contains new key additions and revocations. Both signed by a number of existing keys, where the
      number is given as "1 if height < 149 else floor(log(height)*2)"
    * Longest chain wins.
* Flood-based p2p network: every node can request a list of known connections from the other nodes.

## How blocks are created

Blocks are SQLite database files. Every party in posession of an accepted private key can create new blocks and sign them. Blocks are accepted (if other criteria are satisfied) only if they are signed by one of the accepted keys.

New blocks can contain operations which add or remove keys from a (global) list of accepted keys. See the `_keys` table description in the section on Block metadata.

## Block metadata

### The `_keys` table

This table contains key operations for block creators. Keys can be either accepted or revoked. It is invalid for a `_keys` table to contain both acceptance and revocation records for a single key. Both acceptance and revocation operations require a quorum, where Q different keys which are already accepted sign the hash of the key in question. The number Q is calculated as:

  Q = 1 if H  < 149 else floor(log(H)*2)

Where `H` is the block height of the block containing these records.

For example, if `Q` is 3, to add a key `K` to the list of accepted keys, there must be exactly 3 records in the `_keys` table pertaining to `K`. Each of the records must contain a valid signature by a different, already accepted key. The key `K` can be then used to sign new blocks immediately after the block which contain this records has been accepted.

A table of quorums required for specific block heights is:

  1   1
  149 10
  245 11
  404 12
  666 13
  1097 14
  1809 15
  2981 16
  4915 17
  8104 18
  13360 19
  22027 20
  36316 21
  59875 22
  98716 23
  162755 24

E.g. for block 100000, 23 signatures are required to accept a new signature.
