# FLUX Protocol on Fractal Bitcoin

FLUX is the first Fractal-native token protocol based on UTXO

Like BRC-20, FLUX consists of 3 "functions": Deploy, Mint, Transfer. Deploy signals that a new token has been deployed, while Mint allows to mint from this token, based on the deployment's rules (supply, limits). Eventually, Transfer is being used to send tokens to selected recipients.

This document describes how each of the functions should be reflected within Fractal transactions and how indexers/wallets must treat those.
Further upgrades are not ruled out and may progressively be enhanced within this document.

## PLEASE NOTE

Indexer: https://github.com/inscrib3/flux-indexer. You can deploy, mint, bridge, sell and buy from the official website: https://flux.inspip.com/

## General Rules

- A FLUX TX must consist of at least 2 outputs:
- - The beneficiary receiver
  - An OP_RETURN output, followed by the 'F' and function identifier (D, M, T) and its data
- Tickers are base26 encoded
- - base26 value encoding: A=1, AA=27, Z=26, BA=53
  - Ticker must be unique.
- All integers passed after OP_RETURN must be unsigned
- All unsigned integers from 0 - 16 must be encoded as OP_0/OP_FALSE, OP_1/OP_TRUE to OP_16
- Only 1 OP_RETURN is allowed per TX
- Burning
- - If no change address is specified as quadruple with the Transfer function, remaining tokens of a transfer will be burned.
  - Tokens associated with UTXOs in inputs will be burned if a function of a TX is rejected.
  - Therefore, clients have to apply careful validations of all rules before pushing a transaction.
- Indexers/wallets must detect reorgs and re-index from the first reorg'ed block - 7.
- Indexing starts with block 0 (included)

## Deploy Rules

On a high level, the output of a Deploy function is structured as follows:

```
OP_RETURN
F
D
[BASE26 ENCODED TICKER]
0
[OUTPUT]
[DECIMALS]
[MAX]
[LIMIT]
```

Looking closer the values as follows:

"F": shortcut for FLUX, signalling this is a FLUX function.

"D": shortcut for Deploy, signalling the following data is to define a new token.

"[BASE26 ENCODED TICKER]": human readable ticker name, encoded as described in General Rules.

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary

"[DECIMALS]": the decimals for the token from 0 - 8 as unsigned integer.

"[MAX]": the max. amount of tokens as hex encoded string, ever for this token as in supply.

"[LIMIT]": the max. amount of tokens as hex encoded string that may be minted per tx.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules. This allows for applications like marketplaces sending royalties to the benificiary on trading fees.

Clients must skip UTXOs being used for inputs that already contain tokens.

[MAX] and [LIMIT] values must be a hex encoded string, representing a human readable number. Leading zeros are not allowed. Trailing zeros in decimals are not allowed. One decimal point can be used or omitted. No other characters are allowed.

Transactions with decimal points beyond [DECIMALS] are rejected. Max. number is '18446744073709551615'.

Examples:

```
2100 => ok
2100.5 => ok, if decimal length <= [DECIMALS]
 2100 => not ok
2100.50 => not ok
2,100 => not ok
18446744073709551616 => not ok (note exceeding the max number)
``` 

Indexers must transform the given [MAX] and [LIMIT] internally into bigints based on [DECIMALS] and perform calculations on those to maintain precision. No calculations or rounding on the original human readable format allowed.

## Mint Rules

Mint structure:

```
OP_RETURN
F
M
[BASE26 ENCODED TICKER]
0
[OUTPUT]
[MINT AMOUNT]
```

Values:

"F": shortcut for FLUX, signalling this is a FLUX function.

"M": shortcut for Mint, signalling the following data is to mint tokens.

"[BASE26 ENCODED TICKER]": human readable ticker name, encoded as described in General Rules.

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary.

"[MINT AMOUNT]": The amount to mint as hex encoded string, between 0 and [LIMIT] (inclusive), given with the deploy function.

If the mint amount does not exceed the limit and supply that is left from the deployment, the amount of tokens must be credited to the beneficiary as assigned in [OUTPUT].

Remaining supply must be credited to the beneficiary, as long as the limit isn't exceeded. Indexers/wallets have to associate the amount of credited tokens to the resulting UTXO of the output linked with <OUTPUT> and store in its index.

Clients must skip UTXOs being used for inputs that already contain tokens.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules.

Deploy and Mint can happen in the same block but any Mint will be rejected if its transaction index is < the deploy transaction index.

[MINT AMOUNT] value must be a hex encoded string, representing a human readable number. Leading zeros are not allowed. Trailing zeros in decimals are not allowed. One decimal point can be used or omitted. No other characters are allowed.

Transactions with decimal points beyond [DECIMALS] (see Deploy Rules) are rejected. Max. number is '18446744073709551615'. 

Examples:

```
2100 => ok
2100.5 => ok, if decimal length <= [DECIMALS]
 2100 => not ok
2100.50 => not ok
2,100 => not ok
18446744073709551616 => not ok (note exceeding the max number)
``` 

Indexers must transform the given [MINT AMOUNT] internally into bigint based on [DECIMALS] (see Deploy Rules) and perform calculations on those to maintain precision. No calculations or rounding on the original human readable format allowed.

Only hex encoded strings must be accepted and raw values lead to invalid token transactions.

## Transfer Rules

Transfer may contain a number quadruple of pushes after 'T'. Each quadruple may address different tickers, IDs, outputs and amounts.
This allows to specify change addresses (and limited multi-send) within a single transaction. There is no limit on the amount of quadruples other than the max. allowed script size for the output.

Transfer structure:

```
OP_RETURN
F
T
...begin quadruple
[BASE26 ENCODED TICKER]
[ID]
0
[TRANSFER AMOUNT]
...end quadruple
...next quadruple...
```

"F": shortcut for FLUX, signalling this is a FLUX function.

"T": shortcut for Transfer, signalling the following data is to send tokens.

Quadruple:

"[BASE26 ENCODED TICKER]": human readable ticker name, encoded as described in General Rules.

"[OUTPUT]": the index as unsigned integer of the output containing the address/pubkey of the beneficiary.

"[TRANSFER AMOUNT]": The amount to transfer as hex encoded string.

The transfer amount must be deducted from the amount of tokens associated with the UTXOs of the inputs. Remaining tokens _should_ be sent to a change address using another quadruple, unless they should be burned. If there are UTXOs with enough token balances, the transfer amount must be credited to the beneficiary as assigned in each quadruple's [OUTPUT]. 

It is important to note that one [OUTPUT] can only be used once to prevent multiple token types being associated with a single utxo. The transaction will be rejected if the combined amount of UTXO balances are insufficient _or_ there are duplicate [OUTPUT] associations. It's not recommended to use different token types (ticker:id) in one OP_RETURN. Due to the size limitations, there might not be enough space left for wallets to create enough quadruples to assign change for each type.

A transaction containing this function, must be assigned to a beneficiary address as described in General Rules.

Deploy, Mint and Transfer can happen in the same block but any Transfer will be rejected if its transaction index is < the deploy transaction index or < the Mint transaction that _would_ credit for the amount to transfer.

[TRANSFER AMOUNT] value must be a hex encoded string, representing a human readable number. Leading zeros are not allowed. Trailing zeros in decimals are not allowed. One decimal point can be used or omitted. No other characters are allowed.

Transactions with decimal points beyond [DECIMALS] (see Deploy Rules) are rejected. Max. number is '18446744073709551615'.

Examples:

```
2100 => ok
2100.5 => ok, if decimal length <= [DECIMALS]
 2100 => not ok
2100.50 => not ok
2,100 => not ok
``` 

Indexers must transform the given [TRANSFER AMOUNT] internally into bigint based on [DECIMALS] (see Deploy Rules) and perform calculations on those to maintain precision. No calculations or rounding on the original human readable format allowed.

The operation must be atomic: If one quadruple fails, all fail. No token balance-changing operations will be applied in this case.
