# Transaction Execution Approval Language (TEAL)

TEAL is a bytecode based stack language that executes inside Algorand transactions. TEAL programs can be used to check the parameters of the transaction and approve the transaction as if by a signature. This use of TEAL is called a _LogicSig_. Starting with v2, TEAL programs may
also execute as _Applications_ which are invoked with explicit application call transactions. Programs have read-only access to the transaction they are attached to, transactions in their atomic transaction group, and a few global values. In addition, _Application_ programs have access to limited state that is global to the application and per-account local state for each account that has opted-in to the application. For both types of program, approval is signaled by finishing with the stack containing a single non-zero uint64 value.

## The Stack

The stack starts empty and contains values of either uint64 or bytes
(`bytes` are implemented in Go as a []byte slice and may not exceed
4096 bytes in length). Most operations act on the stack, popping
arguments from it and pushing results to it.

The maximum stack depth is currently 1000. If the stack depth is
exceed or if a `bytes` element exceed 4096 bytes, the program fails.

## Scratch Space

In addition to the stack there are 256 positions of scratch space,
also uint64-bytes union values, each initialized as uint64
zero. Scratch space is acccesed by the `load(s)` and `store(s)` ops
moving data from or to scratch space, respectively.

## Execution Modes

Starting from version 2 TEAL evaluator can run programs in two modes:
1. LogicSig (stateless)
2. Application run (stateful)

Differences between modes include:
1. Max program length (consensus parameters LogicSigMaxSize, MaxAppTotalProgramLen & MaxExtraAppProgramPages)
2. Max program cost (consensus parameters LogicSigMaxCost, MaxAppProgramCost)
3. Opcode availability. For example, all stateful operations are only available in stateful mode. Refer to [opcodes document](TEAL_opcodes.md) for details.

## Execution Environment for LogicSigs

TEAL LogicSigs run in Algorand nodes as part of testing a proposed transaction to see if it is valid and authorized to be committed into a block.

If an authorized program executes and finishes with a single non-zero uint64 value on the stack then that program has validated the transaction it is attached to.

The TEAL program has access to data from the transaction it is attached to (`txn` op), any transactions in a transaction group it is part of (`gtxn` op), and a few global values like consensus parameters (`global` op). Some "Args" may be attached to a transaction being validated by a TEAL program. Args are an array of byte strings. A common pattern would be to have the key to unlock some contract as an Arg. Args are recorded on the blockchain and publicly visible when the transaction is submitted to the network. These LogicSig Args are _not_ part of the transaction ID nor of the TxGroup hash. They also cannot be read from other TEAL programs in the group of transactions.

A program can either authorize some delegated action on a normal private key signed or multisig account or be wholly in charge of a contract account.

* If the account has signed the program (an ed25519 signature on "Program" concatenated with the program bytes) then if the program returns true the transaction is authorized as if the account had signed it. This allows an account to hand out a signed program so that other users can carry out delegated actions which are approved by the program. Note that LogicSig Args are _not_ signed.

* If the SHA512_256 hash of the program (prefixed by "Program") is equal to the transaction Sender address then this is a contract account wholly controlled by the program. No other signature is necessary or possible. The only way to execute a transaction against the contract account is for the program to approve it.

The TEAL bytecode plus the length of all Args must add up to no more than 1000 bytes (consensus parameter LogicSigMaxSize). Each TEAL op has an associated cost and the program cost must total no more than 20000 (consensus parameter LogicSigMaxCost). Most ops have a cost of 1, but a few slow crypto ops are much higher. Prior to v4, the program's cost was estimated as the static sum of all the opcode costs in the program (whether they were actually executed or not). Beginning with v4, the program's cost is tracked dynamically, while being evaluated. If the program exceeds its budget, it fails.

## Constants

Constants are loaded into the environment into storage separate from the stack. They can then be pushed onto the stack by referring to the type and index. This makes for efficient re-use of byte constants used for account addresses, etc. Constants that are not reused can be pushed with `pushint` or `pushbytes`.

The assembler will hide most of this, allowing simple use of `int 1234` and `byte 0xcafed00d`. These constants will automatically get assembled into int and byte pages of constants, de-duplicated, and operations to load them from constant storage space inserted.

Constants are loaded into the environment by two opcodes, `intcblock` and `bytecblock`. Both of these use [proto-buf style variable length unsigned int](https://developers.google.com/protocol-buffers/docs/encoding#varint), reproduced [here](#varuint). The `intcblock` opcode is followed by a varuint specifying the length of the array and then that number of varuint. The `bytecblock` opcode is followed by a varuint array length then that number of pairs of (varuint, bytes) length prefixed byte strings. This should efficiently load 32 and 64 byte constants which will be common as addresses, hashes, and signatures.

Constants are pushed onto the stack by `intc`, `intc_[0123]`, `pushint`, `bytec`, `bytec_[0123]`, and `pushbytes`. The assembler will handle converting `int N` or `byte N` into the appropriate form of the instruction needed.

### Named Integer Constants

#### OnComplete

An application transaction must indicate the action to be taken following the execution of its approvalProgram or clearStateProgram. The constants below describe the available actions.

| Value | Constant name | Description |
| --- | --- | --- |
| 0 | NoOp | Only execute the `ApprovalProgram` associated with this application ID, with no additional effects. |
| 1 | OptIn | Before executing the `ApprovalProgram`, allocate local state for this application into the sender's account data. |
| 2 | CloseOut | After executing the `ApprovalProgram`, clear any local state for this application out of the sender's account data. |
| 3 | ClearState | Don't execute the `ApprovalProgram`, and instead execute the `ClearStateProgram` (which may not reject this transaction). Additionally, clear any local state for this application out of the sender's account data as in `CloseOutOC`. |
| 4 | UpdateApplication | After executing the `ApprovalProgram`, replace the `ApprovalProgram` and `ClearStateProgram` associated with this application ID with the programs specified in this transaction. |
| 5 | DeleteApplication | After executing the `ApprovalProgram`, delete the application parameters from the account data of the application's creator. |

#### TypeEnum constants
| Value | Constant name | Description |
| --- | --- | --- |
| 0 | unknown | Unknown type. Invalid transaction |
| 1 | pay | Payment |
| 2 | keyreg | KeyRegistration |
| 3 | acfg | AssetConfig |
| 4 | axfer | AssetTransfer |
| 5 | afrz | AssetFreeze |
| 6 | appl | ApplicationCall |


## Operations

Most operations work with only one type of argument, uint64 or bytes, and panic if the wrong type value is on the stack.

Many instructions accept values to designate Accounts, Assets, or Applications. Beginning with TEAL v4, these values may always be given as an _offset_ in the corresponding Txn fields (Txn.Accounts, Txn.ForeignAssets, Txn.ForeignApps) _or_ as the value itself (a bytes address for Accounts, or a uint64 ID). The values, however, must still be present in the Txn fields. Before TEAL v4, most opcodes required the use of an offset, except for reading account local values of assets or applications, which accepted the IDs directly and did not require the ID to be present in they corresponding _Foreign_ array. (Note that beginning with TEAL v4, those IDs are required to be present in their corresponding _Foreign_ array.) See individual opcodes for details. In the case of account offsets or application offsets, 0 is specially defined to Txn.Sender or the ID of the current application, respectively.

Many programs need only a few dozen instructions. The instruction set has some optimization built in. `intc`, `bytec`, and `arg` take an immediate value byte, making a 2-byte op to load a value onto the stack, but they also have single byte versions for loading the most common constant values. Any program will benefit from having a few common values loaded with a smaller one byte opcode. Cryptographic hashes and `ed25519verify` are single byte opcodes with powerful libraries behind them. These operations still take more time than other ops (and this is reflected in the cost of each op and the cost limit of a program) but are efficient in compiled code space.

This summary is supplemented by more detail in the [opcodes document](TEAL_opcodes.md).

Some operations 'panic' and immediately fail the program.
A transaction checked by a program that panics is not valid.
A contract account governed by a buggy program might not have a way to get assets back out of it. Code carefully.

### Arithmetic, Logic, and Cryptographic Operations

For one-argument ops, `X` is the last element on the stack, which is typically replaced by a new value.

For two-argument ops, `A` is the penultimate element on the stack and `B` is the top of the stack. These typically result in popping A and B from the stack and pushing the result.

For three-argument ops, `A` is the element two below the top, `B` is the penultimate stack element and `C` is the top of the stack. These operations typically pop A, B, and C from the stack and push the result.

| Op | Description |
| --- | --- |
| `sha256` | SHA256 hash of value X, yields [32]byte |
| `keccak256` | Keccak256 hash of value X, yields [32]byte |
| `sha512_256` | SHA512_256 hash of value X, yields [32]byte |
| `ed25519verify` | for (data A, signature B, pubkey C) verify the signature of ("ProgData" \|\| program_hash \|\| data) against the pubkey => {0 or 1} |
| `ecdsa_verify v` | for (data A, signature B, C and pubkey D, E) verify the signature of the data against the pubkey => {0 or 1} |
| `ecdsa_pk_recover v` | for (data A, recovery id B, signature C, D) recover a public key => [*... stack*, X, Y] |
| `ecdsa_pk_decompress v` | decompress pubkey A into components X, Y => [*... stack*, X, Y] |
| `+` | A plus B. Fail on overflow. |
| `-` | A minus B. Fail if B > A. |
| `/` | A divided by B (truncated division). Fail if B == 0. |
| `*` | A times B. Fail on overflow. |
| `<` | A less than B => {0 or 1} |
| `>` | A greater than B => {0 or 1} |
| `<=` | A less than or equal to B => {0 or 1} |
| `>=` | A greater than or equal to B => {0 or 1} |
| `&&` | A is not zero and B is not zero => {0 or 1} |
| `\|\|` | A is not zero or B is not zero => {0 or 1} |
| `shl` | A times 2^B, modulo 2^64 |
| `shr` | A divided by 2^B |
| `sqrt` | The largest integer B such that B^2 <= X |
| `bitlen` | The highest set bit in X. If X is a byte-array, it is interpreted as a big-endian unsigned integer. bitlen of 0 is 0, bitlen of 8 is 4 |
| `exp` | A raised to the Bth power. Fail if A == B == 0 and on overflow |
| `==` | A is equal to B => {0 or 1} |
| `!=` | A is not equal to B => {0 or 1} |
| `!` | X == 0 yields 1; else 0 |
| `len` | yields length of byte value X |
| `itob` | converts uint64 X to big endian bytes |
| `btoi` | converts bytes X as big endian to uint64 |
| `%` | A modulo B. Fail if B == 0. |
| `\|` | A bitwise-or B |
| `&` | A bitwise-and B |
| `^` | A bitwise-xor B |
| `~` | bitwise invert value X |
| `mulw` | A times B out to 128-bit long result as low (top) and high uint64 values on the stack |
| `addw` | A plus B out to 128-bit long result as sum (top) and carry-bit uint64 values on the stack |
| `divmodw` | Pop four uint64 values.  The deepest two are interpreted as a uint128 dividend (deepest value is high word), the top two are interpreted as a uint128 divisor.  Four uint64 values are pushed to the stack. The deepest two are the quotient (deeper value is the high uint64). The top two are the remainder, low bits on top. |
| `expw` | A raised to the Bth power as a 128-bit long result as low (top) and high uint64 values on the stack. Fail if A == B == 0 or if the results exceeds 2^128-1 |
| `getbit` | pop a target A (integer or byte-array), and index B. Push the Bth bit of A. |
| `setbit` | pop a target A, index B, and bit C. Set the Bth bit of A to C, and push the result |
| `getbyte` | pop a byte-array A and integer B. Extract the Bth byte of A and push it as an integer |
| `setbyte` | pop a byte-array A, integer B, and small integer C (between 0..255). Set the Bth byte of A to C, and push the result |
| `concat` | pop two byte-arrays A and B and join them, push the result |

These opcodes return portions of byte arrays, accessed by position, in
various sizes.

### Byte Array Manipulation

| Op | Description |
| --- | --- |
| `substring s e` | pop a byte-array A. For immediate values in 0..255 S and E: extract a range of bytes from A starting at S up to but not including E, push the substring result. If E < S, or either is larger than the array length, the program fails |
| `substring3` | pop a byte-array A and two integers B and C. Extract a range of bytes from A starting at B up to but not including C, push the substring result. If C < B, or either is larger than the array length, the program fails |
| `extract s l` | pop a byte-array A. For immediate values in 0..255 S and L: extract a range of bytes from A starting at S up to but not including S+L, push the substring result. If L is 0, then extract to the end of the string. If S or S+L is larger than the array length, the program fails |
| `extract3` | pop a byte-array A and two integers B and C. Extract a range of bytes from A starting at B up to but not including B+C, push the substring result. If B+C is larger than the array length, the program fails |
| `extract_uint16` | pop a byte-array A and integer B. Extract a range of bytes from A starting at B up to but not including B+2, convert bytes as big endian and push the uint64 result. If B+2 is larger than the array length, the program fails |
| `extract_uint32` | pop a byte-array A and integer B. Extract a range of bytes from A starting at B up to but not including B+4, convert bytes as big endian and push the uint64 result. If B+4 is larger than the array length, the program fails |
| `extract_uint64` | pop a byte-array A and integer B. Extract a range of bytes from A starting at B up to but not including B+8, convert bytes as big endian and push the uint64 result. If B+8 is larger than the array length, the program fails |
| `base64_decode e` | decode X which was base64-encoded using _encoding alphabet_ E. Fail if X is not base64 encoded with alphabet E |

These opcodes take byte-array values that are interpreted as
big-endian unsigned integers.  For mathematical operators, the
returned values are the shortest byte-array that can represent the
returned value.  For example, the zero value is the empty
byte-array. For comparison operators, the returned value is a uint64

Input lengths are limited to a maximum length 64 bytes, which
represents a 512 bit unsigned integer. Output lengths are not
explicitly restricted, though only `b*` and `b+` can produce a larger
output than their inputs, so there is an implicit length limit of 128
bytes on outputs.

| Op | Description |
| --- | --- |
| `b+` | A plus B, where A and B are byte-arrays interpreted as big-endian unsigned integers |
| `b-` | A minus B, where A and B are byte-arrays interpreted as big-endian unsigned integers. Fail on underflow. |
| `b/` | A divided by B (truncated division), where A and B are byte-arrays interpreted as big-endian unsigned integers. Fail if B is zero. |
| `b*` | A times B, where A and B are byte-arrays interpreted as big-endian unsigned integers. |
| `b<` | A is less than B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1} |
| `b>` | A is greater than B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1} |
| `b<=` | A is less than or equal to B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1} |
| `b>=` | A is greater than or equal to B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1} |
| `b==` | A is equals to B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1} |
| `b!=` | A is not equal to B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1} |
| `b%` | A modulo B, where A and B are byte-arrays interpreted as big-endian unsigned integers. Fail if B is zero. |

These opcodes operate on the bits of byte-array values.  The shorter
array is interpreted as though left padded with zeros until it is the
same length as the other input.  The returned values are the same
length as the longest input.  Therefore, unlike array arithmetic,
these results may contain leading zero bytes.

| Op | Description |
| --- | --- |
| `b\|` | A bitwise-or B, where A and B are byte-arrays, zero-left extended to the greater of their lengths |
| `b&` | A bitwise-and B, where A and B are byte-arrays, zero-left extended to the greater of their lengths |
| `b^` | A bitwise-xor B, where A and B are byte-arrays, zero-left extended to the greater of their lengths |
| `b~` | X with all bits inverted |

### Loading Values

Opcodes for getting data onto the stack.

Some of these have immediate data in the byte or bytes after the opcode.

| Op | Description |
| --- | --- |
| `intcblock uint ...` | prepare block of uint64 constants for use by intc |
| `intc i` | push Ith constant from intcblock to stack |
| `intc_0` | push constant 0 from intcblock to stack |
| `intc_1` | push constant 1 from intcblock to stack |
| `intc_2` | push constant 2 from intcblock to stack |
| `intc_3` | push constant 3 from intcblock to stack |
| `pushint uint` | push immediate UINT to the stack as an integer |
| `bytecblock bytes ...` | prepare block of byte-array constants for use by bytec |
| `bytec i` | push Ith constant from bytecblock to stack |
| `bytec_0` | push constant 0 from bytecblock to stack |
| `bytec_1` | push constant 1 from bytecblock to stack |
| `bytec_2` | push constant 2 from bytecblock to stack |
| `bytec_3` | push constant 3 from bytecblock to stack |
| `pushbytes bytes` | push the following program bytes to the stack |
| `bzero` | push a byte-array of length X, containing all zero bytes |
| `arg n` | push Nth LogicSig argument to stack |
| `arg_0` | push LogicSig argument 0 to stack |
| `arg_1` | push LogicSig argument 1 to stack |
| `arg_2` | push LogicSig argument 2 to stack |
| `arg_3` | push LogicSig argument 3 to stack |
| `args` | push Xth LogicSig argument to stack |
| `txn f` | push field F of current transaction to stack |
| `gtxn t f` | push field F of the Tth transaction in the current group |
| `txna f i` | push Ith value of the array field F of the current transaction |
| `txnas f` | push Xth value of the array field F of the current transaction |
| `gtxna t f i` | push Ith value of the array field F from the Tth transaction in the current group |
| `gtxnas t f` | push Xth value of the array field F from the Tth transaction in the current group |
| `gtxns f` | push field F of the Xth transaction in the current group |
| `gtxnsa f i` | push Ith value of the array field F from the Xth transaction in the current group |
| `gtxnsas f` | pop an index A and an index B. push Bth value of the array field F from the Ath transaction in the current group |
| `global f` | push value from globals to stack |
| `load i` | copy a value from scratch space to the stack. All scratch spaces are 0 at program start. |
| `loads` | copy a value from the Xth scratch space to the stack.  All scratch spaces are 0 at program start. |
| `store i` | pop value X. store X to the Ith scratch space |
| `stores` | pop indexes A and B. store B to the Ath scratch space |
| `gload t i` | push Ith scratch space index of the Tth transaction in the current group |
| `gloads i` | push Ith scratch space index of the Xth transaction in the current group |
| `gaid t` | push the ID of the asset or application created in the Tth transaction of the current group |
| `gaids` | push the ID of the asset or application created in the Xth transaction of the current group |

**Transaction Fields**

| Index | Name | Type | Notes |
| --- | --- | --- | --- |
| 0 | Sender | []byte | 32 byte address |
| 1 | Fee | uint64 | micro-Algos |
| 2 | FirstValid | uint64 | round number |
| 3 | FirstValidTime | uint64 | Causes program to fail; reserved for future use |
| 4 | LastValid | uint64 | round number |
| 5 | Note | []byte | Any data up to 1024 bytes |
| 6 | Lease | []byte | 32 byte lease value |
| 7 | Receiver | []byte | 32 byte address |
| 8 | Amount | uint64 | micro-Algos |
| 9 | CloseRemainderTo | []byte | 32 byte address |
| 10 | VotePK | []byte | 32 byte address |
| 11 | SelectionPK | []byte | 32 byte address |
| 12 | VoteFirst | uint64 | The first round that the participation key is valid. |
| 13 | VoteLast | uint64 | The last round that the participation key is valid. |
| 14 | VoteKeyDilution | uint64 | Dilution for the 2-level participation key |
| 15 | Type | []byte | Transaction type as bytes |
| 16 | TypeEnum | uint64 | See table below |
| 17 | XferAsset | uint64 | Asset ID |
| 18 | AssetAmount | uint64 | value in Asset's units |
| 19 | AssetSender | []byte | 32 byte address. Causes clawback of all value of asset from AssetSender if Sender is the Clawback address of the asset. |
| 20 | AssetReceiver | []byte | 32 byte address |
| 21 | AssetCloseTo | []byte | 32 byte address |
| 22 | GroupIndex | uint64 | Position of this transaction within an atomic transaction group. A stand-alone transaction is implicitly element 0 in a group of 1 |
| 23 | TxID | []byte | The computed ID for this transaction. 32 bytes. |
| 24 | ApplicationID | uint64 | ApplicationID from ApplicationCall transaction. LogicSigVersion >= 2. |
| 25 | OnCompletion | uint64 | ApplicationCall transaction on completion action. LogicSigVersion >= 2. |
| 26 | ApplicationArgs | []byte | Arguments passed to the application in the ApplicationCall transaction. LogicSigVersion >= 2. |
| 27 | NumAppArgs | uint64 | Number of ApplicationArgs. LogicSigVersion >= 2. |
| 28 | Accounts | []byte | Accounts listed in the ApplicationCall transaction. LogicSigVersion >= 2. |
| 29 | NumAccounts | uint64 | Number of Accounts. LogicSigVersion >= 2. |
| 30 | ApprovalProgram | []byte | Approval program. LogicSigVersion >= 2. |
| 31 | ClearStateProgram | []byte | Clear state program. LogicSigVersion >= 2. |
| 32 | RekeyTo | []byte | 32 byte Sender's new AuthAddr. LogicSigVersion >= 2. |
| 33 | ConfigAsset | uint64 | Asset ID in asset config transaction. LogicSigVersion >= 2. |
| 34 | ConfigAssetTotal | uint64 | Total number of units of this asset created. LogicSigVersion >= 2. |
| 35 | ConfigAssetDecimals | uint64 | Number of digits to display after the decimal place when displaying the asset. LogicSigVersion >= 2. |
| 36 | ConfigAssetDefaultFrozen | uint64 | Whether the asset's slots are frozen by default or not, 0 or 1. LogicSigVersion >= 2. |
| 37 | ConfigAssetUnitName | []byte | Unit name of the asset. LogicSigVersion >= 2. |
| 38 | ConfigAssetName | []byte | The asset name. LogicSigVersion >= 2. |
| 39 | ConfigAssetURL | []byte | URL. LogicSigVersion >= 2. |
| 40 | ConfigAssetMetadataHash | []byte | 32 byte commitment to some unspecified asset metadata. LogicSigVersion >= 2. |
| 41 | ConfigAssetManager | []byte | 32 byte address. LogicSigVersion >= 2. |
| 42 | ConfigAssetReserve | []byte | 32 byte address. LogicSigVersion >= 2. |
| 43 | ConfigAssetFreeze | []byte | 32 byte address. LogicSigVersion >= 2. |
| 44 | ConfigAssetClawback | []byte | 32 byte address. LogicSigVersion >= 2. |
| 45 | FreezeAsset | uint64 | Asset ID being frozen or un-frozen. LogicSigVersion >= 2. |
| 46 | FreezeAssetAccount | []byte | 32 byte address of the account whose asset slot is being frozen or un-frozen. LogicSigVersion >= 2. |
| 47 | FreezeAssetFrozen | uint64 | The new frozen value, 0 or 1. LogicSigVersion >= 2. |
| 48 | Assets | uint64 | Foreign Assets listed in the ApplicationCall transaction. LogicSigVersion >= 3. |
| 49 | NumAssets | uint64 | Number of Assets. LogicSigVersion >= 3. |
| 50 | Applications | uint64 | Foreign Apps listed in the ApplicationCall transaction. LogicSigVersion >= 3. |
| 51 | NumApplications | uint64 | Number of Applications. LogicSigVersion >= 3. |
| 52 | GlobalNumUint | uint64 | Number of global state integers in ApplicationCall. LogicSigVersion >= 3. |
| 53 | GlobalNumByteSlice | uint64 | Number of global state byteslices in ApplicationCall. LogicSigVersion >= 3. |
| 54 | LocalNumUint | uint64 | Number of local state integers in ApplicationCall. LogicSigVersion >= 3. |
| 55 | LocalNumByteSlice | uint64 | Number of local state byteslices in ApplicationCall. LogicSigVersion >= 3. |
| 56 | ExtraProgramPages | uint64 | Number of additional pages for each of the application's approval and clear state programs. An ExtraProgramPages of 1 means 2048 more total bytes, or 1024 for each program. LogicSigVersion >= 4. |
| 57 | Nonparticipation | uint64 | Marks an account nonparticipating for rewards. LogicSigVersion >= 5. |
| 58 | Logs | []byte | Log messages emitted by an application call (itxn only). LogicSigVersion >= 5. |
| 59 | NumLogs | uint64 | Number of Logs (itxn only). LogicSigVersion >= 5. |
| 60 | CreatedAssetID | uint64 | Asset ID allocated by the creation of an ASA (itxn only). LogicSigVersion >= 5. |
| 61 | CreatedApplicationID | uint64 | ApplicationID allocated by the creation of an application (itxn only). LogicSigVersion >= 5. |


Additional details in the [opcodes document](TEAL_opcodes.md#txn) on the `txn` op.

**Global Fields**

Global fields are fields that are common to all the transactions in the group. In particular it includes consensus parameters.

| Index | Name | Type | Notes |
| --- | --- | --- | --- |
| 0 | MinTxnFee | uint64 | micro Algos |
| 1 | MinBalance | uint64 | micro Algos |
| 2 | MaxTxnLife | uint64 | rounds |
| 3 | ZeroAddress | []byte | 32 byte address of all zero bytes |
| 4 | GroupSize | uint64 | Number of transactions in this atomic transaction group. At least 1 |
| 5 | LogicSigVersion | uint64 | Maximum supported TEAL version. LogicSigVersion >= 2. |
| 6 | Round | uint64 | Current round number. LogicSigVersion >= 2. |
| 7 | LatestTimestamp | uint64 | Last confirmed block UNIX timestamp. Fails if negative. LogicSigVersion >= 2. |
| 8 | CurrentApplicationID | uint64 | ID of current application executing. Fails in LogicSigs. LogicSigVersion >= 2. |
| 9 | CreatorAddress | []byte | Address of the creator of the current application. Fails if no such application is executing. LogicSigVersion >= 3. |
| 10 | CurrentApplicationAddress | []byte | Address that the current application controls. Fails in LogicSigs. LogicSigVersion >= 5. |
| 11 | GroupID | []byte | ID of the transaction group. 32 zero bytes if the transaction is not part of a group. LogicSigVersion >= 5. |


**Asset Fields**

Asset fields include `AssetHolding` and `AssetParam` fields that are used in the `asset_holding_get` and `asset_params_get` opcodes.

| Index | Name | Type | Notes |
| --- | --- | --- | --- |
| 0 | AssetBalance | uint64 | Amount of the asset unit held by this account |
| 1 | AssetFrozen | uint64 | Is the asset frozen or not |


| Index | Name | Type | Notes |
| --- | --- | --- | --- |
| 0 | AssetTotal | uint64 | Total number of units of this asset |
| 1 | AssetDecimals | uint64 | See AssetParams.Decimals |
| 2 | AssetDefaultFrozen | uint64 | Frozen by default or not |
| 3 | AssetUnitName | []byte | Asset unit name |
| 4 | AssetName | []byte | Asset name |
| 5 | AssetURL | []byte | URL with additional info about the asset |
| 6 | AssetMetadataHash | []byte | Arbitrary commitment |
| 7 | AssetManager | []byte | Manager commitment |
| 8 | AssetReserve | []byte | Reserve address |
| 9 | AssetFreeze | []byte | Freeze address |
| 10 | AssetClawback | []byte | Clawback address |
| 11 | AssetCreator | []byte | Creator address. LogicSigVersion >= 5. |


**App Fields**

App fields used in the `app_params_get` opcode.

| Index | Name | Type | Notes |
| --- | --- | --- | --- |
| 0 | AppApprovalProgram | []byte | Bytecode of Approval Program |
| 1 | AppClearStateProgram | []byte | Bytecode of Clear State Program |
| 2 | AppGlobalNumUint | uint64 | Number of uint64 values allowed in Global State |
| 3 | AppGlobalNumByteSlice | uint64 | Number of byte array values allowed in Global State |
| 4 | AppLocalNumUint | uint64 | Number of uint64 values allowed in Local State |
| 5 | AppLocalNumByteSlice | uint64 | Number of byte array values allowed in Local State |
| 6 | AppExtraProgramPages | uint64 | Number of Extra Program Pages of code space |
| 7 | AppCreator | []byte | Creator address |
| 8 | AppAddress | []byte | Address for which this application has authority |


### Flow Control

| Op | Description |
| --- | --- |
| `err` | Error. Fail immediately. This is primarily a fencepost against accidental zero bytes getting compiled into programs. |
| `bnz target` | branch to TARGET if value X is not zero |
| `bz target` | branch to TARGET if value X is zero |
| `b target` | branch unconditionally to TARGET |
| `return` | use last value on stack as success value; end |
| `pop` | discard value X from stack |
| `dup` | duplicate last value on stack |
| `dup2` | duplicate two last values on stack: A, B -> A, B, A, B |
| `dig n` | push the Nth value from the top of the stack. dig 0 is equivalent to dup |
| `cover n` | remove top of stack, and place it deeper in the stack such that N elements are above it. Fails if stack depth <= N. |
| `uncover n` | remove the value at depth N in the stack and shift above items down so the Nth deep value is on top of the stack. Fails if stack depth <= N. |
| `swap` | swaps two last values on stack: A, B -> B, A |
| `select` | selects one of two values based on top-of-stack: A, B, C -> (if C != 0 then B else A) |
| `assert` | immediately fail unless value X is a non-zero number |
| `callsub target` | branch unconditionally to TARGET, saving the next instruction on the call stack |
| `retsub` | pop the top instruction from the call stack and branch to it |

### State Access

| Op | Description |
| --- | --- |
| `balance` | get balance for account A, in microalgos. The balance is observed after the effects of previous transactions in the group, and after the fee for the current transaction is deducted. |
| `min_balance` | get minimum required balance for account A, in microalgos. Required balance is affected by [ASA](https://developer.algorand.org/docs/features/asa/#assets-overview) and [App](https://developer.algorand.org/docs/features/asc1/stateful/#minimum-balance-requirement-for-a-smart-contract) usage. When creating or opting into an app, the minimum balance grows before the app code runs, therefore the increase is visible there. When deleting or closing out, the minimum balance decreases after the app executes. |
| `app_opted_in` | check if account A opted in for the application B => {0 or 1} |
| `app_local_get` | read from account A from local state of the current application key B => value |
| `app_local_get_ex` | read from account A from local state of the application B key C => [*... stack*, value, 0 or 1] |
| `app_global_get` | read key A from global state of a current application => value |
| `app_global_get_ex` | read from application A global state key B => [*... stack*, value, 0 or 1] |
| `app_local_put` | write to account specified by A to local state of a current application key B with value C |
| `app_global_put` | write key A and value B to global state of the current application |
| `app_local_del` | delete from account A local state key B of the current application |
| `app_global_del` | delete key A from a global state of the current application |
| `asset_holding_get i` | read from account A and asset B holding field X (imm arg) => {0 or 1 (top), value} |
| `asset_params_get i` | read from asset A params field X (imm arg) => {0 or 1 (top), value} |
| `app_params_get i` | read from app A params field X (imm arg) => {0 or 1 (top), value} |
| `log` | write bytes to log state of the current application |

### Inner Transactions

The following opcodes allow for "inner transactions". Inner
transactions allow stateful applications to have many of the effects
of a true top-level transaction, programatically.  However, they are
different in significant ways.  The most important differences are
that they are not signed, duplicates are not rejected, and they do not
appear in the block in the usual away. Instead, their effects are
noted in metadata associated with the associated top-level application
call transaction.  An inner transaction's `Sender` must be the
SHA512_256 hash of the application ID (prefixed by "appID"), or an
account that has been rekeyed to that hash.

Currently, inner transactions may perform `pay`, `axfer`, `acfg`, and
`afrz` effects.  After executing an inner transaction with
`itxn_submit`, the effects of the transaction are visible begining
with the next instruction with, for example, `balance` and
`min_balance` checks.

Of the transaction Header fields, only a few fields may be set:
`Type`/`TypeEnum`, `Sender`, and `Fee`. For the specific fields of
each transaction types, any field, except `RekeyTo` may be set.  This
allows, for example, clawback transactions, asset opt-ins, and asset
creates in addtion to the more common uses of `axfer` and `acfg`.  All
fields default to the zero value, except those described under
`itxn_begin`.

Fields may be set multiple times, but may not be read. The most recent
setting is used when `itxn_submit` executes. (For this purpose `Type`
and `TypeEnum` are considered to be the same field.) `itxn_field`
fails immediately for unsupported fields, unsupported transaction
types, or improperly typed values for a particular field. `itxn_field`
makes aceptance decisions entirely from the field and value provided,
never considering previously set fields. Illegal interactions between
fields, such as setting fields that belong to two different
transaction types, are rejected by `itxn_submit`.

| Op | Description |
| --- | --- |
| `itxn_begin` | begin preparation of a new inner transaction in a new transaction group |
| `itxn_next` | begin preparation of a new inner transaction in the same transaction group |
| `itxn_field f` | set field F of the current inner transaction to X |
| `itxn_submit` | execute the current inner transaction group. Fail if executing this group would exceed 16 total inner transactions, or if any transaction in the group fails. |
| `itxn f` | push field F of the last inner transaction to stack |
| `itxna f i` | push Ith value of the array field F of the last inner transaction to stack |


# Assembler Syntax

The assembler parses line by line. Ops that just use the stack appear on a line by themselves. Ops that take arguments are the op and then whitespace and then any argument or arguments.

The first line may contain a special version pragma `#pragma version X`, which directs the assembler to generate TEAL bytecode targeting a certain version. For instance, `#pragma version 2` produces bytecode targeting TEAL v2. By default, the assembler targets TEAL v1.

Subsequent lines may contain other pragma declarations (i.e., `#pragma <some-specification>`), pertaining to checks that the assembler should perform before agreeing to emit the program bytes, specific optimizations, etc. Those declarations are optional and cannot alter the semantics as described in this document.

"`//`" prefixes a line comment.

## Constants and Pseudo-Ops

A few pseudo-ops simplify writing code. `int` and `byte` and `addr` followed by a constant record the constant to a `intcblock` or `bytecblock` at the beginning of code and insert an `intc` or `bytec` reference where the instruction appears to load that value. `addr` parses an Algorand account address base32 and converts it to a regular bytes constant.

`byte` constants are:
```
byte base64 AAAA...
byte b64 AAAA...
byte base64(AAAA...)
byte b64(AAAA...)
byte base32 AAAA...
byte b32 AAAA...
byte base32(AAAA...)
byte b32(AAAA...)
byte 0x0123456789abcdef...
byte "\x01\x02"
byte "string literal"
```

`int` constants may be `0x` prefixed for hex, `0` prefixed for octal, or decimal numbers.

`intcblock` may be explicitly assembled. It will conflict with the assembler gathering `int` pseudo-ops into a `intcblock` program prefix, but may be used if code only has explicit `intc` references. `intcblock` should be followed by space separated int constants all on one line.

`bytecblock` may be explicitly assembled. It will conflict with the assembler if there are any `byte` pseudo-ops but may be used if only explicit `bytec` references are used. `bytecblock` should be followed with byte constants all on one line, either 'encoding value' pairs (`b64 AAA...`) or 0x prefix or function-style values (`base64(...)`) or string literal values.

## Labels and Branches

A label is defined by any string not some other op or keyword and ending in ':'. A label can be an argument (without the trailing ':') to a branch instruction.

Example:
```
int 1
bnz safe
err
safe:
pop
```

# Encoding and Versioning

A program starts with a varuint declaring the version of the compiled code. Any addition, removal, or change of opcode behavior increments the version. For the most part opcode behavior should not change, addition will be infrequent (not likely more often than every three months and less often as the language matures), and removal should be very rare.

For version 1, subsequent bytes after the varuint are program opcode bytes. Future versions could put other metadata following the version identifier.

It is important to prevent newly-introduced transaction fields from breaking assumptions made by older versions of TEAL. If one of the transactions in a group will execute a TEAL program whose version predates a given field, that field must not be set anywhere in the transaction group, or the group will be rejected. For example, executing a TEAL version 1 program on a transaction with RekeyTo set to a nonzero address will cause the program to fail, regardless of the other contents of the program itself.

This requirement is enforced as follows:

* For every transaction, compute the earliest TEAL version that supports all the fields and and values in this transaction. For example, a transaction with a nonzero RekeyTo field will have version (at least) 2.

* Compute the largest version number across all the transactions in a group (of size 1 or more), call it `maxVerNo`. If any transaction in this group has a TEAL program with a version smaller than `maxVerNo`, then that TEAL program will fail.

## Varuint

A '[proto-buf style variable length unsigned int](https://developers.google.com/protocol-buffers/docs/encoding#varint)' is encoded with 7 data bits per byte and the high bit is 1 if there is a following byte and 0 for the last byte. The lowest order 7 bits are in the first byte, followed by successively higher groups of 7 bits.

# What TEAL Cannot Do

Design and implementation limitations to be aware of with various versions of TEAL.

* Stateless TEAL cannot lookup balances of Algos or other assets. (Standard transaction accounting will apply after TEAL has run and authorized a transaction. A TEAL-approved transaction could still be invalid by other accounting rules just as a standard signed transaction could be invalid. e.g. I can't give away money I don't have.)
* TEAL cannot access information in previous blocks. TEAL cannot access most information in other transactions in the current block. (TEAL can access fields of the transaction it is attached to and the transactions in an atomic transaction group.)
* TEAL cannot know exactly what round the current transaction will commit in (but it is somewhere in FirstValid through LastValid).
* TEAL cannot know exactly what time its transaction is committed.
* TEAL cannot loop prior to v4. In v3 and prior, the branch instructions `bnz` "branch if not zero", `bz` "branch if zero" and `b` "branch" can only branch forward so as to skip some code.
* Until v4, TEAL had no notion of subroutines (and therefore no recursion). As of v4, use `callsub` and `retsub`.
* TEAL cannot make indirect jumps. `b`, `bz`, `bnz`, and `callsub` jump to an immediately specified address, and `retsub` jumps to the address currently on the top of the call stack, which is manipulated only by previous calls to `callsub`.
