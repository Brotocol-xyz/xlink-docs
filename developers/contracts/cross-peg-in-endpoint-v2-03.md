# cross-peg-in-endpoint-v2-03.clar

- Location: `xlink/packages/contracts/bridge-stacks/contracts/cross-peg-in-endpoint-v2-03.clar`
- [Deployed contract]()

This technical document provides a detailed overview of the contract responsible for managing the peg-in process, enabling the transfer of assets from external EVM-like blockchains into the Stacks network. The contract serves as the operational interface for relayers to submit orders, which are validated against a threshold of required validators as determined in the `cross-bridge-registry-v2-01`. The contract's primary functionality is implemented through a suite of public functions. Let's review them.


## Storage
### `is-paused`
| Data     | Type   |
| -------- | ------ |
| Variable | `bool` |

A flag indicating whether the peg-in operations are currently active. If set to `true`, all operations are paused, preventing relayers from submitting new peg-in orders. The contract is initialized in a paused state by default.

### `use-whitelist`
| Data     | Type   |
| -------- | ------ |
| Variable | `bool` |

A flag that determines whether the whitelist mechanism is enforced. When set to `true`, only whitelisted users are authorized to perform restricted actions within the contract. By default, this value is `false`.

### `order-hash-to-iter`
| Data     | Type   |
| -------- | ------ |
| Variable | `buff 32` |

A temporary variable used internally to store the hash of the current order being processed.

### `src-chain-id-to-iter`
| Data     | Type   |
| -------- | ------ |
| Variable | `uint` |

A temporary variable used to store the source chain ID of the order currently being processed.

### `whitelisted-users`
| Data     | Type   |
| -------- | ------ |
| Map | `principal → bool`|

A map that keeps track of the whitelist status of specific users. Each entry associates a user's principal with a boolean value indicating whether they are authorized to interact with the contract under a whitelist-enabled configuration. If `true`, the user is allowed to perform actions that are otherwise restricted.

### Relevant constants

* `structured-data-prefix`, `message-domain-main`, `message-domain-test`: These constants are utilized in the `validate-order` function to verify the signature consistency with the order hash.

## Features

### Peg-in features

#### `transfer-to-cross`
This function enables peg-in operations to transfer tokens from an external EVM-like blockchain to Stacks. It validates the provided order by checking its hash and verifying signatures to meet a threshold of validators defined in `cross-bridge-registry-v2-01`. If the order is valid, it mints or transfers the bridged tokens and updates the token reserve for the source EVM chain. It then utilizes `cross-router-v2-02` to route the tokens based on the destination chain. In the event of validation failure, the function initiates a refund process.

##### Parameters
```lisp
(order { 
        from: (buff 128), 
        to: (buff 128),
        token: principal,
        amount-in-fixed: uint,
        src-chain-id: uint,
        dest-chain-id: (optional uint),
        salt: (buff 256)
        })
(token-trait <ft-trait>)
(signature-packs (list 100 { 
                            signer: principal,
                            order-hash: (buff 32),
                            signature: (buff 65)
                            }))
```

#### `transfer-to-cross-swap`
This function facilitates advanced peg-in operations by incorporating token swapping during cross-chain transfers. It validates the order hash and signatures using `cross-bridge-registry-v2-01`, ensuring compliance with routing configurations, such as token paths and output amounts. Upon successful validation, the bridged tokens are swapped and routed to the recipient using `cross-router-v2-02`, following the same logic outlined in the `transfer-to-cross` feature. If validation fails, the function processes a refund.

##### Parameters
```lisp
(order { 
        from: (buff 128),
        to: (buff 128),
        amount-in-fixed: uint,
        routing-tokens: (list 5 principal),
        routing-factors: (list 4 uint),
        min-amount-out-fixed: (optional uint),
        src-chain-id: uint,
        dest-chain-id: (optional uint),
        salt: (buff 256) 
        })
(routing-traits (list 5 <ft-trait>))
(signature-packs (list 100 {
                            signer: principal,
                            order-hash: (buff 32),
                            signature: (buff 65)
                            }))
```

#### `transfer-to-launchpad`
This function enables peg-in operations linked to launchpad projects. It validates the order and signatures through `cross-bridge-registry-v2-01` and confirms compatibility with the launchpad parameters. Once validated, the function mints bridged tokens, transfers them to the recipient, and registers the operation in the `alex-launchpad-v2-01` contract. In case of validation failure, a refund is processed.

##### Parameters
```lisp
(order { 
        address: (buff 128),
        chain-id: uint,
        launch-id: uint,
        dest: (buff 128),
        token: principal,
        amount-in-fixed: uint,
        salt: (buff 256)
        })
(token-trait <ft-trait>)
(signature-packs (list 100 {
                            signer: principal,
                            order-hash: (buff 32),
                            signature: (buff 65)
                            }))
```

### Governance features
#### `is-dao-or-extension`
This standard protocol function checks whether a caller (`tx-sender`) is the DAO executor or an authorized extension, delegating the extensions check to the `executor-dao` contract.

#### `is-whitelisted`
A read-only function that checks whether a specific user is included in the `whitelisted-users` map. It returns `true` if the user is whitelisted; otherwise, it returns `false`.

#### `get-paused`
A read-only function that checks the operational status of the contract.

#### `set-paused`
A public function, governed through the `is-dao-or-extension`, that can change the contract's operational status.

##### Parameters
```lisp
(paused bool)
```

#### `apply-whitelist`
A public function that toggles the use of the whitelist in the contract. When enabled, only users who are on the whitelist are authorized to execute restricted actions within the contract.

##### Parameters
```lisp
(new-use-whitelist bool)
```

#### `whitelist`
A public function that allows authorized extensions, to add or remove a single user from the whitelist. It updates the `whitelisted-users` map, where the user's principal is mapped to a `bool` value (`true` for accepted users and `false` for denied users).

##### Parameters
```lisp
(user principal)
(whitelisted bool)
```

#### `whitelist-many`
A public function that allows to batch add or remove multiple users from the whitelist. This function iteratively calls `whitelist`, which handles the mapping of each user's principal to a bool value in the `whitelisted-users` map.

##### Parameters
```lisp
(users (list 2000 principal))
(whitelisted (list 2000 bool))
```

#### `execute`
A public function that deactivates earlier versions of the `cross-peg-in-endpoint` contract and ensures that only the latest version (`cross-peg-in-endpoint-v2-03`) remains active.

##### Parameters
```lisp
(sender principal)
```

### Getters
#### `get-use-whitelist``
#### `get-validator-or-fail`
##### Parameters
```lisp
(validator principal)
```
#### `get-required-validators`
#### `get-approved-chain-or-fail`
##### Parameters
```lisp
(src-chain-id uint)
```
#### `get-token-reserve-or-default`
##### Parameters
```lisp
(pair { token:principal, chain-id: uint })
```
#### `get-min-fee-or-default`
##### Parameters
```lisp
(pair { token:principal, chain-id: uint })
```
#### `get-approved-pair-or-fail`
##### Parameters
```lisp
(pair { token:principal, chain-id: uint })
```

## Contract calls (interactions)
- `executor-dao`: Calls are made to verify whether a certain contract-caller is designated as an extension.
- `cross-bridge-registry-v2-01`: This contract is called to validate key components of the peg-in process, such as approved tokens, chain IDs, and relayers. It also handles updates to transaction statuses, token reserves, and validator signatures. 
- `cross-router-v2-02`: This contract is used to validate routing details and execute cross-chain token transfers and swaps. It ensures that the provided routing tokens and factors align with the transfer requirements, performs swaps through `amm-pool-v2-01` when multiple tokens are involved, validates minimum output amounts, and handles the final routing process for peg-in operations, including cross and cross-swap transactions.
- `alex-launchpad-v2-01`: This contract is used to validate and register peg-in operations specifically related to launchpad projects on the Stacks network.
- `token-trait`: In cross peg-in operations (featuring `transfer-to-cross`, `transfer-to-cross-swap`, and `transfer-to-launchpad`) this trait is employed to interact with relevant tokens for obtaining their principals, minting, and executing necessary transfers within the transaction. It is a customized version of Stacks' standard definition for Fungible Tokens (`sip-010`), with support for 8-digit fixed notation.
- `cross-peg-out-endpoint-v2-01`: This contract is invoked to manage refunds for peg-in operations that failed external validations.

## Errors

| Error Name       | Value         |
| ---------------- | ------------- |
| `ERR-NOT-AUTHORIZED` | `(err u1000)` |
| `ERR-TOKEN-NOT-AUTHORIZED`    | `(err u1001)` |
| `ERR-DUPLICATE-SIGNATURE` | `(err u1009)` |
| `ERR-ORDER-HASH-MISMATCH`    | `(err u1010)` |
| `ERR-INVALID-SIGNATURE`   | `(err u1011)` |
| `ERR-UKNOWN-RELAYER`   | `(err u1012)` |
| `ERR-REQUIRED-VALIDATORS`    | `(err u1013)` |
| `ERR-ORDER-ALREADY-SENT`    | `(err u1014)` |
| `ERR-PAUSED`    | `(err u1015)` |
| `ERR-INVALID-VALIDATOR` | `(err u1016)` |
| `ERR-INVALID-INPUT`    | `(err u1020)` |
| `ERR-NOT-IN-WHITELIST`    | `(err u1021)` |
| `ERR-INVALID-TOKEN`    | `(err u1022)` |
| `ERR-SLIPPAGE`    | `(err u1023)` |

<!-- Documentation Contract Template v0.1.0 -->