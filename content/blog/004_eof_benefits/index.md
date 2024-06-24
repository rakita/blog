+++
title = "EOF Benefits"
description = "Talking about what EOF brings to Ethereum and addressing questions around EOF inclusion."
date = 2024-06-09T12:00:00+00:00
updated = 2024-06-06T12:00:00+00:00
draft = false
template = "blog/page.html"

[taxonomies]
authors = ["draganrakita"]
+++

EOF (EVM Object Format) is a group of small EIPs that improve EVM. It introduced a new format of the bytecode and prepares EVM for the future.

## Benefits

The value of EOF turns out to be hard to explain as it is not one thing, and with its history of being delayed on multiple forks and development/research that spaned for years and different versions across history, does not help with clarity.

The intention of this post is to summarize the list of benefits and explain them in one sentence:

Benefits:

* EOF allows opcode gas pricing changes.
  * If gas pricing changes, legacy bytecode could act differently.
  * Removal of gas observability, means removal of the GAS opcode and removal of gas limit from CALL/DELEGATECALL/STATICCALL.
  * Allows l2 to change gas depending on their use case. Example of hash being overpriced in zk l2. EIP-7667: Raise gas costs of hash functions  https://eips.ethereum.org/EIPS/eip-7667
  * If present EOF was enabled before berlin, this EIP would not be needed and access list could be removed. https://eips.ethereum.org/EIPS/eip-2930 
* Bytecode are smaller, and it reduces the gas usage
  * Early numbers show reduction in both the code/initcode size and reduction in gas usage: https://notes.ethereum.org/@ipsilon/solidity_eof_poc
  * Uniswap-v3 deployment 6.5% smaller initcode and 6.5% smaller deployed code 
  * deploy UniswapV3Factory take: ~14% less gas and call runTest : ~9% less gas.
  * ENS DNSRegistrar deployment ~6% smaller initcode and ~1.5% smaller deployed code
  * ENS call proveAndClaim: takes ~10% less gas for EOF.
* EOF allows bytecode transformations and upgradability.
  * Removal of code observability means removal of PC, CREATE/CREATE2, EXTCODEHASH, EXTCODESIZE, EXTCODECOPY, CODESIZE, and CODECOPY opcodes.
  * Legacy contracts would break if code is changed.
  * This would allow us to mold EOF bytecode in any form when verkle comes.
* EOF enables opcode immediates
  * opens possibility of SWAPN, DUPN, and EXCHANGE opcodes
  * This allows solidity more freedom on stack size. It removed stack too deep problem in solidity.
* EOF removes expensive jumpdest analysis
  * While in Reth, the analysis is saved with a bytecode, this is not case for other clients. The removal of jumpdest analysis before contract execution increases speed.
  * With analysis removed we can in future increment max bytecode size.
* Static analysis becomes easier
  * Subroutines enforce more structured control flow. This makes fuzzing more effective and linear-time static analysis possible.
  * Data and code are separated and easier to reason about.
  * EOF bytecode can be compiled to faster bytecode.
  * EOF bytecode can be compiled to machine code.
* Future proofing EVM
  * The version and structure of bytecode allow its extensibility. This is especially useful for L2 and standardization.
  * One example of this is `EIP-7701: Native Account Abstraction with EOF` that adds new header section.
* Address space expansion.
   * New EXT*CALL opcodes prepare Ethereum for future address expansion by requiring for address field to be padded with zeros. When Ethereum do decide to extend address space EOF will have prepared those changes.

## Integration with current EVM

An important question for developers is the effort needed to implement the change, the cost of testing, and the cost of maintaining those opcodes.

New opcodes do not clash with legacy opcodes, and EOF cause of verification does not touch deprecated opcodes.

Encoding and decoding of EOF can be fuzzed. Validation is something new, around ~500 loc in Revm but has a lot of edge cases that need to be covered and need focus to do it right across the implementations.

Creation TX is used as a carrier of EOF bytecode, it acts similar to EOFCREATE but requests validation before execution.

Most opcodes are very simple:

* `EXTCALL` (0xf8), `EXTDELEGATECALL` (0xf9), `EXTSTATICCALL` (0xfb)
  * Have same blueprint as deprecated CALLs but with gas_limit and memory output fields removed.
  * Before RETURNDATALOAD (introduced in a very early fork), the memory output of a CALL had to be set before doing a CALL opcode. This did not allow for dynamic outputs.
* EOFCREATE and RETURNCONTRACT
  * are new for EOF and require a special cases to be handled.
  * EOFCREATE introduces new creation call. 
* EXCHANGE (0xe8), SWAPN (0xe7), DUPN (0xe6), DATACOPY (0xd3), DATASIZE (0xd2), DATALOADN (0xd1), DATALOAD (0xd0), RJUMP (0xe0), RJUMPI (0xe1), RJUMPV (0xe2), RETURNDATALOAD 
  * Have simple logic and mostly require 10–20 lines per opcode to implement. Does not have a lot of edge cases that need to be covered.
* CALLF (0xe3), RETF (0xe4), and JUMPF (0xe5)
  * Require subroutine stack, and stack validation, those would require 20-30 loc as measure of complexity.

They require around 2/3 months for one developer. Work on testing has already been started. There are currently around ~2000 handwritten tests for validation and work is beeing done on statetests.

Changes are localized to the EVM, so integration with the rest of the client depends on the client architecture and where you would save bytecode.

EXTCODESIZE and EXTCODEHASH need to be aware if the account is EOF or not and return a predefined value (size and hash of 0xEF00), this could slightly change how integration for the client can be done. One of ideas is to save is_eof flag inside plain accounts table to skip loading of the bytecode when any of EXTCODE type of opcodes are called.

## Effect on L2

Biggest question that is asked is why don’t L2 implement this change? Should we close the EVM improvements on Ethereum L1?

The reality is that L2 are not ready, and not just that, they do not have a platform that would help with integration of those innovations. Versioning of bytecode helps with that as it helps build that platform that L2 can use, removal of code observability helps mitigating upgredability issues, and gas changes help ZK L2 to remove the ddos vector of gas bombs (for example, a hash in zk is worth a lot more).

More importantly, EOF is not just a format, it requires languages (solidity/vyper/huff) to be on board, and it requires tooling to support it so it can be usable. It requires an ecosystem to use it, and this format gives L2 more stability to innovate on top of it. 

## Downside, legacy bytecode still exists

This is a question that is often asked. Legacy bytecode will always be here, if we don’t offer a alternative, we are going to be stuck with it. With alternative formated bytecodes, there is a future where we could transition and remove legacy bytecodes when state expire happens.

## In the end

EOF is not the next big shiny thing, it is the maintenance EIP that fixes problems left from the initial EVM version  and there is no other way to solve those other than breaking it. It is necessary for further development and future proofing of EVM. 

___
Post was originally shared in [hackmd](https://hackmd.io/aZurva_kR2GQdOgJfS7B6A?view) form with Ethereum core devs. This was done a day before [Execution Layer Meeting 189](https://github.com/ethereum/pm/issues/1052). On that ACD call, clients officially included EOF in Prague hardfork.