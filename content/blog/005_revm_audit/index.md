+++
title = "Revm Audit/Fuzzing"
description = "Revm reached its next big milestone with audit/fuzzing."
date = 2024-06-25T12:00:00+00:00
updated = 2024-06-25T12:00:00+00:00
draft = false
template = "blog/page.html"

[taxonomies]
authors = ["draganrakita"]
+++

I am thrilled that this milestone has been reached!

[Revm](https://github.com/bluealloy/revm) is one of the leading EVMs in the Ethereum ecosystem. Many major projects and companies depend on it, and the number is growing, especially since the development of Reth. Revm has solidified its position as a foundational component of the Ethereum ecosystem, and its significance continues to rise.

It is used by clients, block builders, tooling, and many searchers and hobbyists. Revm's stability has been stellar and its execution consistent, leading to heavy reliance on it. Its speed and flexibility make it highly desirable for integration. With `no_std` support, it fits nicely in the zkVM world, where Rust is predominantly used. There are many requirements that Revm fulfills.

There was already a lot of confidence in its consensus correctness, and bugs were squashed, which helped develop the client [Reth](https://reth.rs) on top of it. We wanted to increase that confidence even further and have more assurance. The next step for this was an audit.

[Guido Vranken](https://guidovranken.com/), a top Ethereum [bug hunter](https://bounty.ethereum.org), was hired to stress test Revm. With his expertise in fuzzing and finding consensus bugs, it has been a pleasure to have him on board.

Despite its already extensive usage, Guido managed to find a few undiscovered consensus issues and created a tool for fuzzing that we can reuse.

When we discussed how to conduct the audit, the idea of involving others arose: "If there are multiple parties who rely on Revm, then it may make sense to have them co-fund it."" So we did exactly that and made this audit community-driven and sponsored by six companies that use Revm in various ways.

* **Paradigm**
    Foundry, which Paradigm created, was started in the same month as Revm. After some time, the Foundry team switched to Revm, making it the first big project to use it.
    They were the first to see the potential and are a major reason for Revm's popularity today.
    More than a year later, an idea was born to build Reth as there was already a working EVM. This is when I finally joined Paradigm, as I enjoyed working on client code.
    I am really grateful to Georgios and Paradigm for their support and for later hiring me to work on Reth and continue working on Revm. The Reth team is amazing, and it is my privilege to work with them!

* **Ethereum Foundation**
    We wouldn’t be here without Ethereum, so it made sense to include the Ethereum Foundation. After the audit of Revm and Reth is finished, Reth is scheduled to be included in the EF Bug Bounty along with its dependencies, so any bugs found within Revm that has an impact on the Ethereum network will be receiving rewards through the [Ethereum Foundation Bug Bounty Program](https://bounty.ethereum.org).

* **Optimism** 
    The Optimism stack and their team are amazing! Revm has support for Optimism changes, which made me think about the future of Revm and support for other EVM chains. The RetroPGF mechanism is a great way to incentivize builders to build even more.

* **Nomic Foundation**
    he builders of Hardhat recently enabled their EDR (Ethereum Development Runtime) framework, which uses Revm at its core. They recognized Revm's potential and started transitioning to it years ago.

* **Risc0**
    Risc0 launched Zeth, their zkEVM system, which is leveraging Revm for execution. It is an awesome company that uses RISC-V for universal zk execution.

* **Taiko**
    At one of the conferences, they approached me about Revm audit efforts and offered to help, so when the list was made, they were included. They care about the lower stack and are one of the first projects to make the [1% Pledge](https://x.com/taikoxyz/status/1755609928167981330?lang=en) for [Protocol Guild](https://protocol-guild.readthedocs.io/en/latest/index.html). I appreciate their support.

There are many more projects that depend on Revm, and I am glad that this is the case, as it means the library is doing something good.

This is a huge milestone for Revm and me. When I started the project, I didn't imagine it would have such an impact on Ethereum. I appreciate the community, as without it, this would not be possible. I am committed to fulfilling the expectations and obligations of the users who depend on it.

The work continues. The future is primarily focused on two things:
1. The next hard fork that includes EOF (EVM Object Format)
2. Revm endgame, but that is a topic for another post.

___
Post was originally shared in [hackmd](https://hackmd.io/G7zazTX4TtekCnj6xlgctQ?view) and twitted [here](https://x.com/rakitadragan/status/1803540273907245293).