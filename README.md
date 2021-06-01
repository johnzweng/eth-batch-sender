# ETH Batch Sending

Simple tool to create batch-transactions to send ETH (not ERC20) on the Ethereum network
to multiple destinations in a single transaction.

### Live deployed at: https://ethbatchsend.zweng.at/

## Demo video

https://user-images.githubusercontent.com/285900/120294783-d521b000-c2c6-11eb-8b3f-294b420897a7.mp4

Transaction from demo video: https://ropsten.etherscan.io/tx/0x0ea60a0caed8034b492e81761859b17d2469bf138b972b4c937ccbb922d23787


## Features

- allows to use less than 21000 gas per receiver, when sending ETH to 4 addresses or more
- validates addresses before sending
- checks if addresses fulfill all needed state-requirements (and does not allow to send if these 
  are not fulfilled).


## Limitations

- [MetaMask](https://metamask.io/) must be installed in the browser 
- It's only cheaper than single transactions when sending to 4 or more destinations at once
- It only supports EAOs (external owned accounts, aka "private key addresses"), **NO SmartContracts**!
- The destination addresses **must have been used before on chain** (must have had some 
  transaction in the past or some balance --> see below for technical details about this)

## Ethereum technical details

The tool simply creates a "contract deployment" transaction (i.e. transaction where `to:` is `null`).
The `data` field ist constructed with Ethereum [EVM OP-Codes](https://ethervm.io/) which send ETH 
to all the destinations.

In essence it just uses one OP-Code [`CALL`](https://ethervm.io/#F1) per destination to send the ETH
and at the end one final OP-Code [`SELFDESTRUCT`](https://ethervm.io/#FF) to collect and refund the 
remaining ETH to the sender and also to benefit from the gas refund (each `SELFDESTRUCT` OP-Code
reduces the gas-counter of a transaction by 24000).

All the remaining OP-Codes are just `PUSH` or `DUP` OP-Codes to set up the values on the stack
which are expected by `CALL` and `SELFDESTRUCT`.

### Per destination address

The code in `data` per destination address looks like this (`xxx...xxx` stands for the address):

```
60 00  80 80 80   68 0000abcdef12345678  73 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 82 f1
```

This is in detail:

- `60 00` = `PUSH1 00`: --> push `0x00` onto the stack
- `80` = `DUP1`: --> duplicate the last value on stack (the `0x00` from above)
- `80` = `DUP1`: --> duplicate the last value on stack (the `0x00` from above)
- `80` = `DUP1`: --> duplicate the last value on stack (the `0x00` from above)
- `68 0000abcdef12345678` = `PUSH9 xxxxxxx`: --> push a 9-byte long value to the stack, which will be 
   the amount to send (in wei, as a hexadecimal number, so the `0000abcdef12345678` in this example 
   represent `0.048358647703819896` ETH to be sent)
- `73 xxxx…xxxxx` = `PUSH20 xxx…xxxx`: --> push a 20-byte long value to the stack, which will be 
   the destination address
- `82` = `DUP3` --> duplicate the 3rd value (copy another `0x00` from the ones we have pushed/dup'ed above) 

At this point the EVM stack looks like this (compare with arguments of OP-Code [`CALL`](https://ethervm.io/#F1):

```
00         // "gas": if we set 0, the default minimum if 2300 will be used, is ok as we only send to addresses (no contracts)
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx    // "addr": destination 
0000abcdef12345678         // "value" wei to send with this call
00                         // "argsOffset": no arguments
00                         // "argsLength": no arguments
00                         // "retOffset": no return data expected
00                         // "retLength": no return data expected
```

- And then one final OP-Code: `f1` --> `CALL`: This will execute this transfer and send the ETH
  to this address (and will also place one element on the stack). For efficiency reason we ignore
  the returned `success` element and do neither check it, nor `POP` it from the stack.

All of this will be added **per destination address**!


### Collect remaining ETH at the end:

After the last destination the following code is added:

```
73  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  ff
```

This is in detail:

- `73 xxxx…xxxxx` = `PUSH20 xxx…xxxx`: --> push a 20-byte long value to the stack, which will be 
  the address where the remaing funds of the transaction should be sent to.
- `ff` = [`SELFDESTRUCT`](https://ethervm.io/#FF): Selfdestruct will send all remaining funds to
  the last address on the stack (which we pushed there in the step before) and self-destructs the
  contract, which will reduce the transaction's gas counter again by 24000.


### Note about `CALL`
When using `CALL` to send ETH there is one important detail: if the receiver account of `CALL` does
not exist yet (is a "new" account) then there will be extra gascosts of 25000 gas 
(see `G_newaccount` in Appendix G in the [Yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf))!

"New account" in this context means, that the account does not exist in the Ethereum state trie yet.
So any address which never had sent any transaction (`nonce` still 0) or has never had any balance
does not exist in the state trie yet, and will be therefore much more expensive to send to.

--> This is the rationale why this tool checks before sending if all destination addresses have
    been used before (otherwise sending in batch would be more expensive than a single transaction).



## Examples:

- Transaction on Ropsten testnet to 30 destinations at once: [Tx on Etherscan](https://ropsten.etherscan.io/tx/0x9a8167343507a17655be2b0daf70178328c9656c386aca723b3192aded2db6d2)
- Transaction on Ropsten testnet to 150 destinations at once: [Tx on Etherscan](https://ropsten.etherscan.io/tx/0x855d7878db975b3beb3e37aa0360539020448a7c4ca5c0b315c7a6ad977885bf)

### Useful tools:

You can look up the example transactions above and copy the content of the `Input data` field and disassemble 
them with a tool like [https://etherscan.io/opcode-tool](https://etherscan.io/opcode-tool).


## Development setup

This project was built with [VueJS 3](https://v3.vuejs.org/guide/introduction.html).

```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```

