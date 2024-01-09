# Adversarial inputs

A topic that often accompanies the more general trust question is Lean's robustness against adversarial inputs.

A correct type checker will restrict the input it receives to the rules of Lean's type system under whatever axioms the operator allows. If the operator restricts the permitted axioms to the three "official" ones (`propext`, `Quot.sound`, `Classical.choice`), an input file should not be able to offer a proof of the prelude's `False` which is accepted by the type checker under any circumstances.

However, a minimal type checker will not actively protect against inputs which provide Lean declarations that are logically sound, but are designed to fool a human operator. For example, redefining deep dependencies an adversary knows will not be examined by a referee, or introducing unicode lookalikes to produce a pretty printer output that conceals modification of key definitions.

The idea that "a user might think a theorem has been formally proved, while in fact he or she
is misled about what it is that the system has actually done" is addressed by the idea of Pollack consistency and is explored in this publication[^pollack] by Freek Wiedijk. 

Note that there is nothing in principle preventing developers from writing software or extending a type checker to provide protection against such attacks, it's just not captured by the minimal functionality required by the kernel. However, the extent to which Lean's users have embraced its powerful custom syntax and macro systems may pose some challenges for those interested in improving the story here. Readers should consider this somewhat of an [open issue for future work](./future_work.md#improving-pollack-consistency)

[^pollack]: Freek Wiedijk. Pollack-inconsistency. Electronic Notes in Theoretical Computer Science, 285:85–100, 2012
