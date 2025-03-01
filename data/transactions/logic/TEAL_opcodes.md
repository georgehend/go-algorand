# Opcodes

Ops have a 'cost' of 1 unless otherwise specified.


## err

- Opcode: 0x00
- Pops: _None_
- Pushes: _None_
- Error. Fail immediately. This is primarily a fencepost against accidental zero bytes getting compiled into programs.

## sha256

- Opcode: 0x01
- Pops: *... stack*, []byte
- Pushes: []byte
- SHA256 hash of value X, yields [32]byte
- **Cost**:
   - 7 (LogicSigVersion = 1)
   - 35 (LogicSigVersion >= 2)

## keccak256

- Opcode: 0x02
- Pops: *... stack*, []byte
- Pushes: []byte
- Keccak256 hash of value X, yields [32]byte
- **Cost**:
   - 26 (LogicSigVersion = 1)
   - 130 (LogicSigVersion >= 2)

## sha512_256

- Opcode: 0x03
- Pops: *... stack*, []byte
- Pushes: []byte
- SHA512_256 hash of value X, yields [32]byte
- **Cost**:
   - 9 (LogicSigVersion = 1)
   - 45 (LogicSigVersion >= 2)

## ed25519verify

- Opcode: 0x04
- Pops: *... stack*, {[]byte A}, {[]byte B}, {[]byte C}
- Pushes: uint64
- for (data A, signature B, pubkey C) verify the signature of ("ProgData" || program_hash || data) against the pubkey => {0 or 1}
- **Cost**: 1900

The 32 byte public key is the last element on the stack, preceded by the 64 byte signature at the second-to-last element on the stack, preceded by the data which was signed at the third-to-last element on the stack.

## ecdsa_verify v

- Opcode: 0x05 {uint8 curve index}
- Pops: *... stack*, {[]byte A}, {[]byte B}, {[]byte C}, {[]byte D}, {[]byte E}
- Pushes: uint64
- for (data A, signature B, C and pubkey D, E) verify the signature of the data against the pubkey => {0 or 1}
- **Cost**: 1700
- LogicSigVersion >= 5

`ECDSA` Curves:

| Index | Name | Notes |
| --- | --- | --- |
| 0 | Secp256k1 | secp256k1 curve |


The 32 byte Y-component of a public key is the last element on the stack, preceded by X-component of a pubkey, preceded by S and R components of a signature, preceded by the data that is fifth element on the stack. All values are big-endian encoded. The signed data must be 32 bytes long, and signatures in lower-S form are only accepted.

## ecdsa_pk_decompress v

- Opcode: 0x06 {uint8 curve index}
- Pops: *... stack*, []byte
- Pushes: *... stack*, []byte, []byte
- decompress pubkey A into components X, Y => [*... stack*, X, Y]
- **Cost**: 650
- LogicSigVersion >= 5

`ECDSA` Curves:

| Index | Name | Notes |
| --- | --- | --- |
| 0 | Secp256k1 | secp256k1 curve |


The 33 byte public key in a compressed form to be decompressed into X and Y (top) components. All values are big-endian encoded.

## ecdsa_pk_recover v

- Opcode: 0x07 {uint8 curve index}
- Pops: *... stack*, {[]byte A}, {uint64 B}, {[]byte C}, {[]byte D}
- Pushes: *... stack*, []byte, []byte
- for (data A, recovery id B, signature C, D) recover a public key => [*... stack*, X, Y]
- **Cost**: 2000
- LogicSigVersion >= 5

`ECDSA` Curves:

| Index | Name | Notes |
| --- | --- | --- |
| 0 | Secp256k1 | secp256k1 curve |


S (top) and R elements of a signature, recovery id and data (bottom) are expected on the stack and used to deriver a public key. All values are big-endian encoded. The signed data must be 32 bytes long.

## +

- Opcode: 0x08
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A plus B. Fail on overflow.

Overflow is an error condition which halts execution and fails the transaction. Full precision is available from `addw`.

## -

- Opcode: 0x09
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A minus B. Fail if B > A.

## /

- Opcode: 0x0a
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A divided by B (truncated division). Fail if B == 0.

`divmodw` is available to divide the two-element values produced by `mulw` and `addw`.

## *

- Opcode: 0x0b
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A times B. Fail on overflow.

Overflow is an error condition which halts execution and fails the transaction. Full precision is available from `mulw`.

## <

- Opcode: 0x0c
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A less than B => {0 or 1}

## >

- Opcode: 0x0d
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A greater than B => {0 or 1}

## <=

- Opcode: 0x0e
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A less than or equal to B => {0 or 1}

## >=

- Opcode: 0x0f
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A greater than or equal to B => {0 or 1}

## &&

- Opcode: 0x10
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A is not zero and B is not zero => {0 or 1}

## ||

- Opcode: 0x11
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A is not zero or B is not zero => {0 or 1}

## ==

- Opcode: 0x12
- Pops: *... stack*, {any A}, {any B}
- Pushes: uint64
- A is equal to B => {0 or 1}

## !=

- Opcode: 0x13
- Pops: *... stack*, {any A}, {any B}
- Pushes: uint64
- A is not equal to B => {0 or 1}

## !

- Opcode: 0x14
- Pops: *... stack*, uint64
- Pushes: uint64
- X == 0 yields 1; else 0

## len

- Opcode: 0x15
- Pops: *... stack*, []byte
- Pushes: uint64
- yields length of byte value X

## itob

- Opcode: 0x16
- Pops: *... stack*, uint64
- Pushes: []byte
- converts uint64 X to big endian bytes

## btoi

- Opcode: 0x17
- Pops: *... stack*, []byte
- Pushes: uint64
- converts bytes X as big endian to uint64

`btoi` fails if the input is longer than 8 bytes.

## %

- Opcode: 0x18
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A modulo B. Fail if B == 0.

## |

- Opcode: 0x19
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A bitwise-or B

## &

- Opcode: 0x1a
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A bitwise-and B

## ^

- Opcode: 0x1b
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A bitwise-xor B

## ~

- Opcode: 0x1c
- Pops: *... stack*, uint64
- Pushes: uint64
- bitwise invert value X

## mulw

- Opcode: 0x1d
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: *... stack*, uint64, uint64
- A times B out to 128-bit long result as low (top) and high uint64 values on the stack

## addw

- Opcode: 0x1e
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: *... stack*, uint64, uint64
- A plus B out to 128-bit long result as sum (top) and carry-bit uint64 values on the stack
- LogicSigVersion >= 2

## divmodw

- Opcode: 0x1f
- Pops: *... stack*, {uint64 A}, {uint64 B}, {uint64 C}, {uint64 D}
- Pushes: *... stack*, uint64, uint64, uint64, uint64
- Pop four uint64 values.  The deepest two are interpreted as a uint128 dividend (deepest value is high word), the top two are interpreted as a uint128 divisor.  Four uint64 values are pushed to the stack. The deepest two are the quotient (deeper value is the high uint64). The top two are the remainder, low bits on top.
- **Cost**: 20
- LogicSigVersion >= 4

## intcblock uint ...

- Opcode: 0x20 {varuint length} [{varuint value}, ...]
- Pops: _None_
- Pushes: _None_
- prepare block of uint64 constants for use by intc

`intcblock` loads following program bytes into an array of integer constants in the evaluator. These integer constants can be referred to by `intc` and `intc_*` which will push the value onto the stack. Subsequent calls to `intcblock` reset and replace the integer constants available to the script.

## intc i

- Opcode: 0x21 {uint8 int constant index}
- Pops: _None_
- Pushes: uint64
- push Ith constant from intcblock to stack

## intc_0

- Opcode: 0x22
- Pops: _None_
- Pushes: uint64
- push constant 0 from intcblock to stack

## intc_1

- Opcode: 0x23
- Pops: _None_
- Pushes: uint64
- push constant 1 from intcblock to stack

## intc_2

- Opcode: 0x24
- Pops: _None_
- Pushes: uint64
- push constant 2 from intcblock to stack

## intc_3

- Opcode: 0x25
- Pops: _None_
- Pushes: uint64
- push constant 3 from intcblock to stack

## bytecblock bytes ...

- Opcode: 0x26 {varuint length} [({varuint value length} bytes), ...]
- Pops: _None_
- Pushes: _None_
- prepare block of byte-array constants for use by bytec

`bytecblock` loads the following program bytes into an array of byte-array constants in the evaluator. These constants can be referred to by `bytec` and `bytec_*` which will push the value onto the stack. Subsequent calls to `bytecblock` reset and replace the bytes constants available to the script.

## bytec i

- Opcode: 0x27 {uint8 byte constant index}
- Pops: _None_
- Pushes: []byte
- push Ith constant from bytecblock to stack

## bytec_0

- Opcode: 0x28
- Pops: _None_
- Pushes: []byte
- push constant 0 from bytecblock to stack

## bytec_1

- Opcode: 0x29
- Pops: _None_
- Pushes: []byte
- push constant 1 from bytecblock to stack

## bytec_2

- Opcode: 0x2a
- Pops: _None_
- Pushes: []byte
- push constant 2 from bytecblock to stack

## bytec_3

- Opcode: 0x2b
- Pops: _None_
- Pushes: []byte
- push constant 3 from bytecblock to stack

## arg n

- Opcode: 0x2c {uint8 arg index N}
- Pops: _None_
- Pushes: []byte
- push Nth LogicSig argument to stack
- Mode: Signature

## arg_0

- Opcode: 0x2d
- Pops: _None_
- Pushes: []byte
- push LogicSig argument 0 to stack
- Mode: Signature

## arg_1

- Opcode: 0x2e
- Pops: _None_
- Pushes: []byte
- push LogicSig argument 1 to stack
- Mode: Signature

## arg_2

- Opcode: 0x2f
- Pops: _None_
- Pushes: []byte
- push LogicSig argument 2 to stack
- Mode: Signature

## arg_3

- Opcode: 0x30
- Pops: _None_
- Pushes: []byte
- push LogicSig argument 3 to stack
- Mode: Signature

## txn f

- Opcode: 0x31 {uint8 transaction field index}
- Pops: _None_
- Pushes: any
- push field F of current transaction to stack

`txn` Fields (see [transaction reference](https://developer.algorand.org/docs/reference/transactions/)):

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


TypeEnum mapping:

| Index | "Type" string | Description |
| --- | --- | --- |
| 0 | unknown | Unknown type. Invalid transaction |
| 1 | pay | Payment |
| 2 | keyreg | KeyRegistration |
| 3 | acfg | AssetConfig |
| 4 | axfer | AssetTransfer |
| 5 | afrz | AssetFreeze |
| 6 | appl | ApplicationCall |


FirstValidTime causes the program to fail. The field is reserved for future use.

## global f

- Opcode: 0x32 {uint8 global field index}
- Pops: _None_
- Pushes: any
- push value from globals to stack

`global` Fields:

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


## gtxn t f

- Opcode: 0x33 {uint8 transaction group index} {uint8 transaction field index}
- Pops: _None_
- Pushes: any
- push field F of the Tth transaction in the current group

for notes on transaction fields available, see `txn`. If this transaction is _i_ in the group, `gtxn i field` is equivalent to `txn field`.

## load i

- Opcode: 0x34 {uint8 position in scratch space to load from}
- Pops: _None_
- Pushes: any
- copy a value from scratch space to the stack. All scratch spaces are 0 at program start.

## store i

- Opcode: 0x35 {uint8 position in scratch space to store to}
- Pops: *... stack*, any
- Pushes: _None_
- pop value X. store X to the Ith scratch space

## txna f i

- Opcode: 0x36 {uint8 transaction field index} {uint8 transaction field array index}
- Pops: _None_
- Pushes: any
- push Ith value of the array field F of the current transaction
- LogicSigVersion >= 2

## gtxna t f i

- Opcode: 0x37 {uint8 transaction group index} {uint8 transaction field index} {uint8 transaction field array index}
- Pops: _None_
- Pushes: any
- push Ith value of the array field F from the Tth transaction in the current group
- LogicSigVersion >= 2

## gtxns f

- Opcode: 0x38 {uint8 transaction field index}
- Pops: *... stack*, uint64
- Pushes: any
- push field F of the Xth transaction in the current group
- LogicSigVersion >= 3

for notes on transaction fields available, see `txn`. If top of stack is _i_, `gtxns field` is equivalent to `gtxn _i_ field`. gtxns exists so that _i_ can be calculated, often based on the index of the current transaction.

## gtxnsa f i

- Opcode: 0x39 {uint8 transaction field index} {uint8 transaction field array index}
- Pops: *... stack*, uint64
- Pushes: any
- push Ith value of the array field F from the Xth transaction in the current group
- LogicSigVersion >= 3

## gload t i

- Opcode: 0x3a {uint8 transaction group index} {uint8 position in scratch space to load from}
- Pops: _None_
- Pushes: any
- push Ith scratch space index of the Tth transaction in the current group
- LogicSigVersion >= 4
- Mode: Application

`gload` fails unless the requested transaction is an ApplicationCall and T < GroupIndex.

## gloads i

- Opcode: 0x3b {uint8 position in scratch space to load from}
- Pops: *... stack*, uint64
- Pushes: any
- push Ith scratch space index of the Xth transaction in the current group
- LogicSigVersion >= 4
- Mode: Application

`gloads` fails unless the requested transaction is an ApplicationCall and X < GroupIndex.

## gaid t

- Opcode: 0x3c {uint8 transaction group index}
- Pops: _None_
- Pushes: uint64
- push the ID of the asset or application created in the Tth transaction of the current group
- LogicSigVersion >= 4
- Mode: Application

`gaid` fails unless the requested transaction created an asset or application and T < GroupIndex.

## gaids

- Opcode: 0x3d
- Pops: *... stack*, uint64
- Pushes: uint64
- push the ID of the asset or application created in the Xth transaction of the current group
- LogicSigVersion >= 4
- Mode: Application

`gaids` fails unless the requested transaction created an asset or application and X < GroupIndex.

## loads

- Opcode: 0x3e
- Pops: *... stack*, uint64
- Pushes: any
- copy a value from the Xth scratch space to the stack.  All scratch spaces are 0 at program start.
- LogicSigVersion >= 5

## stores

- Opcode: 0x3f
- Pops: *... stack*, {uint64 A}, {any B}
- Pushes: _None_
- pop indexes A and B. store B to the Ath scratch space
- LogicSigVersion >= 5

## bnz target

- Opcode: 0x40 {int16 branch offset, big endian}
- Pops: *... stack*, uint64
- Pushes: _None_
- branch to TARGET if value X is not zero

The `bnz` instruction opcode 0x40 is followed by two immediate data bytes which are a high byte first and low byte second which together form a 16 bit offset which the instruction may branch to. For a bnz instruction at `pc`, if the last element of the stack is not zero then branch to instruction at `pc + 3 + N`, else proceed to next instruction at `pc + 3`. Branch targets must be aligned instructions. (e.g. Branching to the second byte of a 2 byte op will be rejected.) Starting at v4, the offset is treated as a signed 16 bit integer allowing for backward branches and looping. In prior version (v1 to v3), branch offsets are limited to forward branches only, 0-0x7fff.

At v2 it became allowed to branch to the end of the program exactly after the last instruction: bnz to byte N (with 0-indexing) was illegal for a TEAL program with N bytes before v2, and is legal after it. This change eliminates the need for a last instruction of no-op as a branch target at the end. (Branching beyond the end--in other words, to a byte larger than N--is still illegal and will cause the program to fail.)

## bz target

- Opcode: 0x41 {int16 branch offset, big endian}
- Pops: *... stack*, uint64
- Pushes: _None_
- branch to TARGET if value X is zero
- LogicSigVersion >= 2

See `bnz` for details on how branches work. `bz` inverts the behavior of `bnz`.

## b target

- Opcode: 0x42 {int16 branch offset, big endian}
- Pops: _None_
- Pushes: _None_
- branch unconditionally to TARGET
- LogicSigVersion >= 2

See `bnz` for details on how branches work. `b` always jumps to the offset.

## return

- Opcode: 0x43
- Pops: *... stack*, uint64
- Pushes: _None_
- use last value on stack as success value; end
- LogicSigVersion >= 2

## assert

- Opcode: 0x44
- Pops: *... stack*, uint64
- Pushes: _None_
- immediately fail unless value X is a non-zero number
- LogicSigVersion >= 3

## pop

- Opcode: 0x48
- Pops: *... stack*, any
- Pushes: _None_
- discard value X from stack

## dup

- Opcode: 0x49
- Pops: *... stack*, any
- Pushes: *... stack*, any, any
- duplicate last value on stack

## dup2

- Opcode: 0x4a
- Pops: *... stack*, {any A}, {any B}
- Pushes: *... stack*, any, any, any, any
- duplicate two last values on stack: A, B -> A, B, A, B
- LogicSigVersion >= 2

## dig n

- Opcode: 0x4b {uint8 depth}
- Pops: *... stack*, any
- Pushes: *... stack*, any, any
- push the Nth value from the top of the stack. dig 0 is equivalent to dup
- LogicSigVersion >= 3

## swap

- Opcode: 0x4c
- Pops: *... stack*, {any A}, {any B}
- Pushes: *... stack*, any, any
- swaps two last values on stack: A, B -> B, A
- LogicSigVersion >= 3

## select

- Opcode: 0x4d
- Pops: *... stack*, {any A}, {any B}, {uint64 C}
- Pushes: any
- selects one of two values based on top-of-stack: A, B, C -> (if C != 0 then B else A)
- LogicSigVersion >= 3

## cover n

- Opcode: 0x4e {uint8 depth}
- Pops: *... stack*, any
- Pushes: any
- remove top of stack, and place it deeper in the stack such that N elements are above it. Fails if stack depth <= N.
- LogicSigVersion >= 5

## uncover n

- Opcode: 0x4f {uint8 depth}
- Pops: *... stack*, any
- Pushes: any
- remove the value at depth N in the stack and shift above items down so the Nth deep value is on top of the stack. Fails if stack depth <= N.
- LogicSigVersion >= 5

## concat

- Opcode: 0x50
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- pop two byte-arrays A and B and join them, push the result
- LogicSigVersion >= 2

`concat` fails if the result would be greater than 4096 bytes.

## substring s e

- Opcode: 0x51 {uint8 start position} {uint8 end position}
- Pops: *... stack*, []byte
- Pushes: []byte
- pop a byte-array A. For immediate values in 0..255 S and E: extract a range of bytes from A starting at S up to but not including E, push the substring result. If E < S, or either is larger than the array length, the program fails
- LogicSigVersion >= 2

## substring3

- Opcode: 0x52
- Pops: *... stack*, {[]byte A}, {uint64 B}, {uint64 C}
- Pushes: []byte
- pop a byte-array A and two integers B and C. Extract a range of bytes from A starting at B up to but not including C, push the substring result. If C < B, or either is larger than the array length, the program fails
- LogicSigVersion >= 2

## getbit

- Opcode: 0x53
- Pops: *... stack*, {any A}, {uint64 B}
- Pushes: uint64
- pop a target A (integer or byte-array), and index B. Push the Bth bit of A.
- LogicSigVersion >= 3

see explanation of bit ordering in setbit

## setbit

- Opcode: 0x54
- Pops: *... stack*, {any A}, {uint64 B}, {uint64 C}
- Pushes: any
- pop a target A, index B, and bit C. Set the Bth bit of A to C, and push the result
- LogicSigVersion >= 3

When A is a uint64, index 0 is the least significant bit. Setting bit 3 to 1 on the integer 0 yields 8, or 2^3. When A is a byte array, index 0 is the leftmost bit of the leftmost byte. Setting bits 0 through 11 to 1 in a 4-byte-array of 0s yields the byte array 0xfff00000. Setting bit 3 to 1 on the 1-byte-array 0x00 yields the byte array 0x10.

## getbyte

- Opcode: 0x55
- Pops: *... stack*, {[]byte A}, {uint64 B}
- Pushes: uint64
- pop a byte-array A and integer B. Extract the Bth byte of A and push it as an integer
- LogicSigVersion >= 3

## setbyte

- Opcode: 0x56
- Pops: *... stack*, {[]byte A}, {uint64 B}, {uint64 C}
- Pushes: []byte
- pop a byte-array A, integer B, and small integer C (between 0..255). Set the Bth byte of A to C, and push the result
- LogicSigVersion >= 3

## extract s l

- Opcode: 0x57 {uint8 start position} {uint8 length}
- Pops: *... stack*, []byte
- Pushes: []byte
- pop a byte-array A. For immediate values in 0..255 S and L: extract a range of bytes from A starting at S up to but not including S+L, push the substring result. If L is 0, then extract to the end of the string. If S or S+L is larger than the array length, the program fails
- LogicSigVersion >= 5

## extract3

- Opcode: 0x58
- Pops: *... stack*, {[]byte A}, {uint64 B}, {uint64 C}
- Pushes: []byte
- pop a byte-array A and two integers B and C. Extract a range of bytes from A starting at B up to but not including B+C, push the substring result. If B+C is larger than the array length, the program fails
- LogicSigVersion >= 5

## extract_uint16

- Opcode: 0x59
- Pops: *... stack*, {[]byte A}, {uint64 B}
- Pushes: uint64
- pop a byte-array A and integer B. Extract a range of bytes from A starting at B up to but not including B+2, convert bytes as big endian and push the uint64 result. If B+2 is larger than the array length, the program fails
- LogicSigVersion >= 5

## extract_uint32

- Opcode: 0x5a
- Pops: *... stack*, {[]byte A}, {uint64 B}
- Pushes: uint64
- pop a byte-array A and integer B. Extract a range of bytes from A starting at B up to but not including B+4, convert bytes as big endian and push the uint64 result. If B+4 is larger than the array length, the program fails
- LogicSigVersion >= 5

## extract_uint64

- Opcode: 0x5b
- Pops: *... stack*, {[]byte A}, {uint64 B}
- Pushes: uint64
- pop a byte-array A and integer B. Extract a range of bytes from A starting at B up to but not including B+8, convert bytes as big endian and push the uint64 result. If B+8 is larger than the array length, the program fails
- LogicSigVersion >= 5

## base64_decode e

- Opcode: 0x5c {uint8 alphabet index}
- Pops: *... stack*, []byte
- Pushes: []byte
- decode X which was base64-encoded using _encoding alphabet_ E. Fail if X is not base64 encoded with alphabet E
- **Cost**: 25
- LogicSigVersion >= 6

decodes X using the base64 encoding alphabet E. Specify the alphabet with an immediate arg either as URL and Filename Safe (`URLAlph`) or Standard (`StdAlph`). See <a href="https://rfc-editor.org/rfc/rfc4648.html#section-4">RFC 4648</a> (sections 4 and 5)

## balance

- Opcode: 0x60
- Pops: *... stack*, any
- Pushes: uint64
- get balance for account A, in microalgos. The balance is observed after the effects of previous transactions in the group, and after the fee for the current transaction is deducted.
- LogicSigVersion >= 2
- Mode: Application

params: Before v4, Txn.Accounts offset. Since v4, Txn.Accounts offset or an account address that appears in Txn.Accounts or is Txn.Sender). Return: value.

## app_opted_in

- Opcode: 0x61
- Pops: *... stack*, {any A}, {uint64 B}
- Pushes: uint64
- check if account A opted in for the application B => {0 or 1}
- LogicSigVersion >= 2
- Mode: Application

params: Txn.Accounts offset (or, since v4, an account address that appears in Txn.Accounts or is Txn.Sender), application id (or, since v4, a Txn.ForeignApps offset). Return: 1 if opted in and 0 otherwise.

## app_local_get

- Opcode: 0x62
- Pops: *... stack*, {any A}, {[]byte B}
- Pushes: any
- read from account A from local state of the current application key B => value
- LogicSigVersion >= 2
- Mode: Application

params: Txn.Accounts offset (or, since v4, an account address that appears in Txn.Accounts or is Txn.Sender), state key. Return: value. The value is zero (of type uint64) if the key does not exist.

## app_local_get_ex

- Opcode: 0x63
- Pops: *... stack*, {any A}, {uint64 B}, {[]byte C}
- Pushes: *... stack*, any, uint64
- read from account A from local state of the application B key C => [*... stack*, value, 0 or 1]
- LogicSigVersion >= 2
- Mode: Application

params: Txn.Accounts offset (or, since v4, an account address that appears in Txn.Accounts or is Txn.Sender), application id (or, since v4, a Txn.ForeignApps offset), state key. Return: did_exist flag (top of the stack, 1 if the application and key existed and 0 otherwise), value. The value is zero (of type uint64) if the key does not exist.

## app_global_get

- Opcode: 0x64
- Pops: *... stack*, []byte
- Pushes: any
- read key A from global state of a current application => value
- LogicSigVersion >= 2
- Mode: Application

params: state key. Return: value. The value is zero (of type uint64) if the key does not exist.

## app_global_get_ex

- Opcode: 0x65
- Pops: *... stack*, {uint64 A}, {[]byte B}
- Pushes: *... stack*, any, uint64
- read from application A global state key B => [*... stack*, value, 0 or 1]
- LogicSigVersion >= 2
- Mode: Application

params: Txn.ForeignApps offset (or, since v4, an application id that appears in Txn.ForeignApps or is the CurrentApplicationID), state key. Return: did_exist flag (top of the stack, 1 if the application and key existed and 0 otherwise), value. The value is zero (of type uint64) if the key does not exist.

## app_local_put

- Opcode: 0x66
- Pops: *... stack*, {any A}, {[]byte B}, {any C}
- Pushes: _None_
- write to account specified by A to local state of a current application key B with value C
- LogicSigVersion >= 2
- Mode: Application

params: Txn.Accounts offset (or, since v4, an account address that appears in Txn.Accounts or is Txn.Sender), state key, value.

## app_global_put

- Opcode: 0x67
- Pops: *... stack*, {[]byte A}, {any B}
- Pushes: _None_
- write key A and value B to global state of the current application
- LogicSigVersion >= 2
- Mode: Application

## app_local_del

- Opcode: 0x68
- Pops: *... stack*, {any A}, {[]byte B}
- Pushes: _None_
- delete from account A local state key B of the current application
- LogicSigVersion >= 2
- Mode: Application

params: Txn.Accounts offset (or, since v4, an account address that appears in Txn.Accounts or is Txn.Sender), state key.

Deleting a key which is already absent has no effect on the application local state. (In particular, it does _not_ cause the program to fail.)

## app_global_del

- Opcode: 0x69
- Pops: *... stack*, []byte
- Pushes: _None_
- delete key A from a global state of the current application
- LogicSigVersion >= 2
- Mode: Application

params: state key.

Deleting a key which is already absent has no effect on the application global state. (In particular, it does _not_ cause the program to fail.)

## asset_holding_get i

- Opcode: 0x70 {uint8 asset holding field index}
- Pops: *... stack*, {any A}, {uint64 B}
- Pushes: *... stack*, any, uint64
- read from account A and asset B holding field X (imm arg) => {0 or 1 (top), value}
- LogicSigVersion >= 2
- Mode: Application

`asset_holding_get` Fields:

| Index | Name | Type | Notes |
| --- | --- | --- | --- |
| 0 | AssetBalance | uint64 | Amount of the asset unit held by this account |
| 1 | AssetFrozen | uint64 | Is the asset frozen or not |


params: Txn.Accounts offset (or, since v4, an account address that appears in Txn.Accounts or is Txn.Sender), asset id (or, since v4, a Txn.ForeignAssets offset). Return: did_exist flag (1 if the asset existed and 0 otherwise), value.

## asset_params_get i

- Opcode: 0x71 {uint8 asset params field index}
- Pops: *... stack*, uint64
- Pushes: *... stack*, any, uint64
- read from asset A params field X (imm arg) => {0 or 1 (top), value}
- LogicSigVersion >= 2
- Mode: Application

`asset_params_get` Fields:

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


params: Before v4, Txn.ForeignAssets offset. Since v4, Txn.ForeignAssets offset or an asset id that appears in Txn.ForeignAssets. Return: did_exist flag (1 if the asset existed and 0 otherwise), value.

## app_params_get i

- Opcode: 0x72 {uint8 app params field index}
- Pops: *... stack*, uint64
- Pushes: *... stack*, any, uint64
- read from app A params field X (imm arg) => {0 or 1 (top), value}
- LogicSigVersion >= 5
- Mode: Application

`app_params_get` Fields:

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


params: Txn.ForeignApps offset or an app id that appears in Txn.ForeignApps. Return: did_exist flag (1 if the application existed and 0 otherwise), value.

## min_balance

- Opcode: 0x78
- Pops: *... stack*, any
- Pushes: uint64
- get minimum required balance for account A, in microalgos. Required balance is affected by [ASA](https://developer.algorand.org/docs/features/asa/#assets-overview) and [App](https://developer.algorand.org/docs/features/asc1/stateful/#minimum-balance-requirement-for-a-smart-contract) usage. When creating or opting into an app, the minimum balance grows before the app code runs, therefore the increase is visible there. When deleting or closing out, the minimum balance decreases after the app executes.
- LogicSigVersion >= 3
- Mode: Application

params: Before v4, Txn.Accounts offset. Since v4, Txn.Accounts offset or an account address that appears in Txn.Accounts or is Txn.Sender). Return: value.

## pushbytes bytes

- Opcode: 0x80 {varuint length} {bytes}
- Pops: _None_
- Pushes: []byte
- push the following program bytes to the stack
- LogicSigVersion >= 3

pushbytes args are not added to the bytecblock during assembly processes

## pushint uint

- Opcode: 0x81 {varuint int}
- Pops: _None_
- Pushes: uint64
- push immediate UINT to the stack as an integer
- LogicSigVersion >= 3

pushint args are not added to the intcblock during assembly processes

## callsub target

- Opcode: 0x88 {int16 branch offset, big endian}
- Pops: _None_
- Pushes: _None_
- branch unconditionally to TARGET, saving the next instruction on the call stack
- LogicSigVersion >= 4

The call stack is separate from the data stack. Only `callsub` and `retsub` manipulate it.

## retsub

- Opcode: 0x89
- Pops: _None_
- Pushes: _None_
- pop the top instruction from the call stack and branch to it
- LogicSigVersion >= 4

The call stack is separate from the data stack. Only `callsub` and `retsub` manipulate it.

## shl

- Opcode: 0x90
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A times 2^B, modulo 2^64
- LogicSigVersion >= 4

## shr

- Opcode: 0x91
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A divided by 2^B
- LogicSigVersion >= 4

## sqrt

- Opcode: 0x92
- Pops: *... stack*, uint64
- Pushes: uint64
- The largest integer B such that B^2 <= X
- **Cost**: 4
- LogicSigVersion >= 4

## bitlen

- Opcode: 0x93
- Pops: *... stack*, any
- Pushes: uint64
- The highest set bit in X. If X is a byte-array, it is interpreted as a big-endian unsigned integer. bitlen of 0 is 0, bitlen of 8 is 4
- LogicSigVersion >= 4

bitlen interprets arrays as big-endian integers, unlike setbit/getbit

## exp

- Opcode: 0x94
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: uint64
- A raised to the Bth power. Fail if A == B == 0 and on overflow
- LogicSigVersion >= 4

## expw

- Opcode: 0x95
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: *... stack*, uint64, uint64
- A raised to the Bth power as a 128-bit long result as low (top) and high uint64 values on the stack. Fail if A == B == 0 or if the results exceeds 2^128-1
- **Cost**: 10
- LogicSigVersion >= 4

## b+

- Opcode: 0xa0
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- A plus B, where A and B are byte-arrays interpreted as big-endian unsigned integers
- **Cost**: 10
- LogicSigVersion >= 4

## b-

- Opcode: 0xa1
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- A minus B, where A and B are byte-arrays interpreted as big-endian unsigned integers. Fail on underflow.
- **Cost**: 10
- LogicSigVersion >= 4

## b/

- Opcode: 0xa2
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- A divided by B (truncated division), where A and B are byte-arrays interpreted as big-endian unsigned integers. Fail if B is zero.
- **Cost**: 20
- LogicSigVersion >= 4

## b*

- Opcode: 0xa3
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- A times B, where A and B are byte-arrays interpreted as big-endian unsigned integers.
- **Cost**: 20
- LogicSigVersion >= 4

## b<

- Opcode: 0xa4
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: uint64
- A is less than B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1}
- LogicSigVersion >= 4

## b>

- Opcode: 0xa5
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: uint64
- A is greater than B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1}
- LogicSigVersion >= 4

## b<=

- Opcode: 0xa6
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: uint64
- A is less than or equal to B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1}
- LogicSigVersion >= 4

## b>=

- Opcode: 0xa7
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: uint64
- A is greater than or equal to B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1}
- LogicSigVersion >= 4

## b==

- Opcode: 0xa8
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: uint64
- A is equals to B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1}
- LogicSigVersion >= 4

## b!=

- Opcode: 0xa9
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: uint64
- A is not equal to B, where A and B are byte-arrays interpreted as big-endian unsigned integers => { 0 or 1}
- LogicSigVersion >= 4

## b%

- Opcode: 0xaa
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- A modulo B, where A and B are byte-arrays interpreted as big-endian unsigned integers. Fail if B is zero.
- **Cost**: 20
- LogicSigVersion >= 4

## b|

- Opcode: 0xab
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- A bitwise-or B, where A and B are byte-arrays, zero-left extended to the greater of their lengths
- **Cost**: 6
- LogicSigVersion >= 4

## b&

- Opcode: 0xac
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- A bitwise-and B, where A and B are byte-arrays, zero-left extended to the greater of their lengths
- **Cost**: 6
- LogicSigVersion >= 4

## b^

- Opcode: 0xad
- Pops: *... stack*, {[]byte A}, {[]byte B}
- Pushes: []byte
- A bitwise-xor B, where A and B are byte-arrays, zero-left extended to the greater of their lengths
- **Cost**: 6
- LogicSigVersion >= 4

## b~

- Opcode: 0xae
- Pops: *... stack*, []byte
- Pushes: []byte
- X with all bits inverted
- **Cost**: 4
- LogicSigVersion >= 4

## bzero

- Opcode: 0xaf
- Pops: *... stack*, uint64
- Pushes: []byte
- push a byte-array of length X, containing all zero bytes
- LogicSigVersion >= 4

## log

- Opcode: 0xb0
- Pops: *... stack*, []byte
- Pushes: _None_
- write bytes to log state of the current application
- LogicSigVersion >= 5
- Mode: Application

`log` fails if called more than MaxLogCalls times in a program, or if the sum of logged bytes exceeds 1024 bytes.

## itxn_begin

- Opcode: 0xb1
- Pops: _None_
- Pushes: _None_
- begin preparation of a new inner transaction in a new transaction group
- LogicSigVersion >= 5
- Mode: Application

`itxn_begin` initializes Sender to the application address; Fee to the minimum allowable, taking into account MinTxnFee and credit from overpaying in earlier transactions; FirstValid/LastValid to the values in the top-level transaction, and all other fields to zero values.

## itxn_field f

- Opcode: 0xb2 {uint8 transaction field index}
- Pops: *... stack*, any
- Pushes: _None_
- set field F of the current inner transaction to X
- LogicSigVersion >= 5
- Mode: Application

`itxn_field` fails if X is of the wrong type for F, including a byte array of the wrong size for use as an address when F is an address field. `itxn_field` also fails if X is an account or asset that does not appear in `txn.Accounts` or `txn.ForeignAssets` of the top-level transaction. (Setting addresses in asset creation are exempted from this requirement.)

## itxn_submit

- Opcode: 0xb3
- Pops: _None_
- Pushes: _None_
- execute the current inner transaction group. Fail if executing this group would exceed 16 total inner transactions, or if any transaction in the group fails.
- LogicSigVersion >= 5
- Mode: Application

`itxn_submit` resets the current transaction so that it can not be resubmitted. A new `itxn_begin` is required to prepare another inner transaction.

## itxn f

- Opcode: 0xb4 {uint8 transaction field index}
- Pops: _None_
- Pushes: any
- push field F of the last inner transaction to stack
- LogicSigVersion >= 5
- Mode: Application

## itxna f i

- Opcode: 0xb5 {uint8 transaction field index} {uint8 transaction field array index}
- Pops: _None_
- Pushes: any
- push Ith value of the array field F of the last inner transaction to stack
- LogicSigVersion >= 5
- Mode: Application

## itxn_next

- Opcode: 0xb6
- Pops: _None_
- Pushes: _None_
- begin preparation of a new inner transaction in the same transaction group
- LogicSigVersion >= 6
- Mode: Application

## txnas f

- Opcode: 0xc0 {uint8 transaction field index}
- Pops: *... stack*, uint64
- Pushes: any
- push Xth value of the array field F of the current transaction
- LogicSigVersion >= 5

## gtxnas t f

- Opcode: 0xc1 {uint8 transaction group index} {uint8 transaction field index}
- Pops: *... stack*, uint64
- Pushes: any
- push Xth value of the array field F from the Tth transaction in the current group
- LogicSigVersion >= 5

## gtxnsas f

- Opcode: 0xc2 {uint8 transaction field index}
- Pops: *... stack*, {uint64 A}, {uint64 B}
- Pushes: any
- pop an index A and an index B. push Bth value of the array field F from the Ath transaction in the current group
- LogicSigVersion >= 5

## args

- Opcode: 0xc3
- Pops: *... stack*, uint64
- Pushes: []byte
- push Xth LogicSig argument to stack
- LogicSigVersion >= 5
- Mode: Signature
