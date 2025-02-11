\newpage
## Script, P2SH, and Miniscript {#sec:miniscript}


![Ep. 04 {l0pt}](qr/ep/04.png)

This chapter will talk about Miniscript and how it makes using Bitcoin Script much easier. It’ll break down how Script works, how you can do more complicated and even absurd things with it, and how Miniscript emerged to make transactions less complicated and more secure. Additionally, it’ll cover what policy language is and how it can make it easier to create scripts.

### Constraints

Scripts are how the Bitcoin blockchain constrains how a given coin can be spent: When you want to receive bitcoin, you tell the person sending it what rules apply to the transaction. For example, with no constraints, anybody can touch whatever’s in your wallet. Usually you would add the constraint that only you can spend these coins. Or, to be more precise, “These coins can only be spent if the transaction includes a signature that’s made with a specific public key, i.e. my key.”

So you’d tell the person sending bitcoin either your public key or the hash of it. This is what addresses are for, as we explained in chapter @sec:address. Then, they’d put that on the blockchain with a note on it saying that only the owner of that public key can spend the bitcoin.

However, although this is by far the most common type of constraint, there are all sorts of other types of constraints, and you can even specify several different options for the spender, such as “I can spend this, but my mom also needs to sign it. But after two years, maybe I can sign it alone.” In such a scenario, if you want to spend the money, you need to specify which option you’re using and fulfill only the specific criteria for that option.

### How Script Works

Script^[<https://en.bitcoin.it/wiki/Script>] is a stack-based language, so think of it like a stack of plates. You can put plates on it, and you can take the top plate off, but you can’t manipulate plates in the middle.

A stack works differently than regular memory where you can read and write to arbitrary addresses (such as a hard disk or RAM — random-access memory). A stack is easier to implement and reason about.^[In contrast, Ethereum smart contracts have a stack as well as regular memory and even longterm storage. As a consequence, it’s much more difficult for developers to reason about its behavior. <https://dlt-repo.net/storage-vs-memory-vs-stack-in-solidity-ethereum/>].

The most commonly used (before SegWit, see chapter @sec:segwit) Bitcoin Script reads as follows:

- `OP_DUP` (as in duplicate)
- `OP_HASH160` (which takes the SHA-256 hash twice, and then the RIPEMD-160 hash)
- `pubKeyHash` (the public key hash)
- `OP_EQUAL_VERIFY`
- `OP_CHECKSIG`

The value for `pubKeyHash` is generated by the recipient wallet by performing a double SHA-256 hash followed by RIPEMD-160, and the result is inserted into the above script. As explained in chapter @sec:address, a Bitcoin _address_ only contains the `pubKeyHash`; the rest is implied. It’s actually the sender’s wallet that generates the full script before publishing it on the blockchain.

On the blockchain this script ends up in the output of a transaction. The output of a transaction, i.e. a coin, consists of this script that it’s locked with (`scriptPubKey`) and the amount. Now, if you want to spend that, you create a transaction input that instructs the blockchain to add certain things to the stack _before_ the above script is executed.

The Bitcoin interpreter will see what you put on the stack, and it’ll start running the program from the output. In this case, what you put on the stack is your signature and your public key, because the original script didn’t have your public key; it had the hash of your public key.

Continuing with the example from above, we start with a stack that has two plates. The plate at the bottom is your signature, and on top of that is a plate with your public key, and then the script says `OP_DUP`. It pops the stack, i.e. it takes the top element from the stack
 — the top plate, which is the public key — and duplicates it. The original and duplicate are then pushed onto the stack. So now you have two plates with a public key at the top of the stack, and your signature’s still at the bottom.

The next instruction is `OP_HASH160`. This pops one of those two duplicated public keys off the stack and hashes it, and then pushes that hash on the stack.

The signature is still at the bottom, and then there’s a public key, and then there’s the hash of the public key (three plates).

The next operation is `pubKeyHash`, which pushes the hash of your public key on the stack. So now, the hash of your public key occurs twice at the top of the stack.

The operation `OP_EQUAL_VERIFY` pops both these hashes of the stack and checks to see if they’re the same.

What’s left on the stack is your signature and your public key, so `OP_CHECKSIG` checks the signature using your public key, and then the stack is empty.

In a nutshell, that’s how a Script program is run, and you can do arbitrarily complicated things along the way.

### Script Hash and P2SH

In general, whenever you want to receive coins from someone, you have to specify exactly what script to use. In the above example, all that’s needed is to provide the hash of the public key in a standardized address format, and the sender wallet creates the correct script.

But in the earlier more complicated example, with alternative conditions such as having a parent sign after a few years, communicating this becomes awkward. Even if there was an address standard, it would be a very long address indeed, due to all these different possible constraints.

Fortunately, there’s an alternative to giving the counterparty (the sender) the full script — you can give them the hash of the script, which is always the same length, and also happens to be the same length as a normal address.

In 2012, the Pay-to-Script-Hash (P2SH) standard was introduced.^[<https://en.bitcoin.it/wiki/BIP_0016>] These kinds of transactions let you send to a script hash, which is an address beginning with 3, in stead of sending to a public key hash, which is an address beginning with 1.

The person on the other end has to copy-paste it, put it in their Bitcoin wallet, and send money to it. Now, when you want to spend that money, you need to reveal the actual script to the blockchain, which your wallet will handle automatically. Because all you need to share is a hash,  the person that’s sending you money doesn’t need to care what this hash actually hides. Only when you spend the coins do you need to reveal the constraints. From a privacy point of view, this is much better than immediately putting the script on the chain. Chapter @sec:taproot_basics will explain how Taproot takes this even further.

Similar to the workflow with regular P2PKH addresses, what you communicate to the sender is just the hash of the script. Before the sender’s wallet puts that on the blockchain, it prepends `OP_HASH160` and appends `OP_EQUAL`. So this is essentially a script within a script. The outer script, which the wallet puts on the blockchain, tells the blockchain there’s an inner script that must be revealed _and satisfied_ by the recipient in order to spend from it.

This last requirement does not actually follow from the script on the blockchain, which only requires the hash of the script to match. This is why the new P2SH address type came with a soft fork to enforce that when such a script within a script is found, it is also executed. This usually means that the spender doesn’t just put the script on the stack, but also the ingredients necessary to satisfy the script, such as a signature.

### Really Absurd Things

Script is a programming language that was introduced in Bitcoin, though it resembles a preexisting language known as Forth. It also seems to have been cobbled together as an afterthought. In fact, a lot of the operations that were part of the language were removed almost immediately, because there were all sorts of ways that you could just crash a node or do other bad things.^[
Unfortunately, with Bitcoin, you can’t just start with a draft language and then clean it up later. But this only became clear once developers realized the only safe way to upgrade Bitcoin is through very carefully crafted soft forks. Every change has to be backward compatible and not break any existing script. But developers can’t always know the intention of scripts that are already out there, and worse still, as explained above, most scripts are hashed, so they could contain anything.
<!-- The double \ is needed to separate paragraphs in a footnote. Without it everything but the first paragraph disappears entirely. -->
\
\
As a result, it’s been a complete nightmare to make sure upgrades to the Script language don’t do anything surprising or bad. If it turns out that existing nodes can be negatively impacted, e.g. crashed, by some obscure script, developers have to very carefully work around that issue; they have to fix the problem without accidentally making coins unspendable and without introducing new bugs, including in any unknown (hashed) script potentially out there.
\
\
Worse still, because Bitcoin is a live system and users can’t be forced to all update at once, an ideal fix should not tip off an attacker as to what the issue is. But at the same time, it’s an open source and transparent system, where changes can’t go through without public justification. This makes Responsible Disclosure (<https://github.com/bitcoin/bitcoin/blob/master/SECURITY.md>) very complicated. So it’s really best to go above and beyond to avoid such problems in first place.
]

The script’s language is diverse enough to allow for weird stuff. If you just want somebody to send money to you, you only need this very simple standard script explained above: `OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`.

But let’s say you’re collaborating, and you want to do a multi-signature, or multisig. To spend coins, two signatures need to provided, rather than just one. Now you could just use `OP_CHECKMULTISIG`, but let’s say that didn’t yet exist. Instead, you could take the script for one signature from above and more or less duplicate it, like so: `<KEY_A> OP_CHECKSIGVERIFY <KEY_B> OP_CHECKSIG`. In this example you’re B, the second key that’s checked (we also don’t bother with hashing the public keys).

Essentially, if you start with those two public keys and two signatures on the stack, and you run the script one instruction at a time, then if A and B put a valid signature on the stack, it’s all good. This would be a poor man’s multisig.

However, a malicious actor could insert an op code called `OP_RETURN` in the middle: `<KEY_A> OP_CHECKSIGVERIFY OP_RETURN <KEY_B> OP_CHECKSIG`.

This `OP_RETURN` code instructs the blockchain to stop evaluating the program — in other words, skipping the signature check for B, your signature.

If you naively looked at this script, you might think that your signature is checked at the end, and so the rest of the script isn’t relevant. If you had a vigilant electronic lawyer (i.e. a person or computer program that does due diligence on transactions), who would properly check that this “smart contract” does what it says it does, they might say, “Careful there, your signature isn’t getting checked.” This hypothetical electronic lawyer should see that `OP_RETURN` “fine print” and warn you. But the problem is there are countless ways in which scripts can go wrong, which is why we need a standardized way of dealing with these scripts.

In an interview with Bitcoin Magazine,^[<https://bitcoinmagazine.com/technical/miniscript-how-blockstream-engineers-are-making-bitcoin-programming-easyer>] Andrew Poelstra said, “There are opcodes in Bitcoin Script which do really absurd things, like, interpret a signature as a true/false value, branch on that; convert that boolean to a number and then index into the stack, and rearrange the stack based on that number. And the specific rules for how it does this are super nuts.”

This quote exemplifies the complexity of potential ways to mess around with script.

To return to the plate analogy, you’d take a hammer and smash one, and then you’d confuse two and paint one red and then it would still work, if you do it correctly. It’s completely absurd.

So that’s the long^[If you can’t get enough of this, watch Andrew Poelstra’s two-hour presentation at London Bitcoin Devs, where he goes on and on and on about the problems in script: <https://www.youtube.com/watch?v=_v1lECxNDiM>] and short of the problem with scripts: It’s easy to make mistakes or hide bugs and make all sorts of complex arrangements that people might or might not notice. And then your money goes places you don’t want it to go. We’ve already seen in other projects, famously with the Ethereum DAO hack and resulting hard fork,^[<https://ogucluturk.medium.com/the-dao-hack-explained-unfortunate-take-off-of-smart-contracts-2bd8c8db3562>] how bad things can get if you have a very complicated language that does things you’re not completely expecting. But Bitcoin dodged many bullets in the early days, and despite its relative simplicity,^[<https://blockstream.com/2018/11/28/en-simplicity-github/>] it still requires vigilance.

### Enter Miniscript

Miniscript^[<https://medium.com/blockstream/miniscript-bitcoin-scripting-3aeff3853620>] is a project that was designed by a few Blockstream engineers: Pieter Wuille, Andrew Poelstra, and Sanket Kanjalkar. It’s “a language for writing (a subset of) Bitcoin Scripts in a structured way, enabling analysis, composition, generic signing and more.” You can see examples and try it yourself at <http://bitcoin.sipa.be/miniscript>.

Miniscript consists of a few dozen script fragments, each a sequence of op codes. These fragments can be combined. If individual script op codes are like an alphabet then Miniscript fragments are like words. By building a script that only uses these words, rather than just any combination of letters from the alphabet, you lose some Script features, but you gain certain guarantees about safety and correct behavior.

A simple example of Miniscript is `pkh(A)` — which consists of only a single fragment. It’s the equivalent of the standard P2PKH script analyzed above (`OP_DUP OP_HASH160 <pubKeyHashA> OP_EQUALVERIFY OP_CHECKSIG`). The poor man’s multisig above requires several Miniscript fragments: `and_v(v:pk(pubKeyA),pk(pubKeyB))`.

Miniscript makes sure there’s no funny stuff in the fine print. It removes some of the foot guns,^[An unsafe piece of code that causes users to shoot themselves in the foot. Early Bitcoin developer Gregory Maxwell was using this term as early as 2012, see e.g. <https://github.com/bitcoin/bitcoin/pull/1889#issuecomment-9638527>, but it may be older] but it also allows you to do very cool stuff safely. In particular, it lets you do things like `AND`. So you can say condition `A` must be true `AND` condition `B` must be true, and you can do things like `OR`. And whatever’s inside the `OR` or inside the `AND` can be arbitrarily complex.

In contrast, with Bitcoin Script, you have `if` and `else` statements, but if you’re not careful, those `if` and `else` statements won’t do what you think they’re going to do, because there’s complexity hidden after these statements.

Meanwhile, with Miniscript, the templates make sure you’re only doing things that are actually doing what you think they’re doing. Let’s say you’re a company and you offer a semi-custodial wallet solution, where you have one of the keys of the user and the user has the other has two keys. You don’t have a majority of the keys, but maybe there’s a five-year timeout where you do have control in case the user dies or something else happens.

This would be like a multisig set up. Normally, when you set up a multisig, everybody gives their public key^[Usually everyone would provide not just one public key, but a whole series of public keys, by using an extended public key, or xpub] for example, and you create a simple script that has three keys and three people sign. But the problem is, because you’re a big business that offers a service, you have some really complicated internal accounting department and you maybe want to have five different signatures by specific people with varied levels of complexity.

There’s a lot of complex stuff you can do with it, and all the complex stuff should count as one key.

The problem with that is how does the customer know the script is OK? They’d have to hire their own electronic lawyer to check that the script doesn’t have any little gimmicks in it.

Miniscript allows you to check that. A futuristic wallet could show you a little pie chart, saying “You’re this one piece of the pie, and there’s this other piece of the pie that’s really complicated, but you don’t have to worry about it. It’s not going to do anything sneaky.”

### Policy Language

A policy language is a way to express your intentions. It’s easier than writing a Miniscript directly, let alone writing Bitcoin Script directly. A compiler then does the hard work.

Our earlier example of a poor man’s multisig was actually found this way. Starting with a policy `and(pk(KEY_A),pk(KEY_B))`, the compiler produced `and_v(v:pk(KEY_A),pk(KEY_B))`, which is equivalent to the script `<KEY_A> OP_CHECKSIGVERIFY <KEY_B> OP_CHECKSIG`. It turns out this actually produces a lower fee transaction than `<KEY_A> <KEY_B> 2 OP_CHECKMULTISIG`. This is the kind of optimization a human might overlook, which is what compilers are good for.

Basically, you write a policy language, which is like a higher-level programming language, which the compiler turns into low level op codes. These are instructions like the ones we described above for popping things off the stack and duplicating them. Miniscript also lives at that very low level, even if it’s slightly more readable and a lot safer. It’s the compiler’s job to take a high level language like Policy Language and turn into the most efficient low-level code.

In the case of multisig, you might say, “I just want two out of two signatures. I don’t care how you do that.” The compiler knows there are multiple ways to execute the intention. And then, the question is, which of them will be picked? The answer to that depends on the transaction weight and the fees that might be involved.

However, you can also tell the compiler, “OK, I think most of the time it’s condition A, but only 10 percent of the time it’s condition B.” The compiler would then calculate the fee for condition A, multiply by 9, add the fee for condition B and divide the total by 10 in order to get the average expected fee. It can optimize for typical use cases, worst case scenarios, all these things, and it then spits out a Miniscript which can then be transpiled to Bitcoin Script.^[The technical term for going from Miniscript to Script — or for transforming source code from any language into another similar one — is transpiling, which can basically be done in two directions. So you can go from Miniscript to Script, or from Script to Miniscript, but you can’t trivially go back to a policy language. However, using automated analysis tools, you can often still figure out what policy language was used to produce a given piece of Miniscript.]

With Taproot (see chapter @sec:taproot_basics), rather than splitting different conditions using _and_ / _or_, they can be split into a Merkle tree of scripts. You don’t have to worry about how to build the Merkle tree, as the compiler takes care of that. In principle, each leaf can also contain _and_ / _or_ statements. Does it make sense to do that? Or is it better to stick to one condition per leaf? Who knows? A future Miniscript compiler can just try all permutations and decide what’s optimal.

### Limitations

All this said, there are some limitations when you’re using policy language or Miniscript in general.

To ensure Miniscript and its corresponding Bitcoin Script can be safely reasoned about, it does not provide access to the full power of script. Sometimes, however, doing things safely results in a script that’s unacceptably long and expensive to execute. In that case, a human may be able to construct a better solution. In regard to the example Poelstra mentioned in the context of how transactions for the Lightning network deal with time locks, hashes, or nonces, there are some optimizations. As he put it: “Oh, you do some weird switching of the stack and you interpret things, not the way they were.” You put a public key on it, but you interpret it as a number — those kind of weird tricks.

Those might be very hard to reason about, and a human might be able to do it, but the Miniscript compiler wouldn’t, which means the compiler would end up with potentially longer Lightning scripts. Perhaps one day Miniscript can be expanded so it can also find these shortcuts. But the Miniscript developers have to be careful, because they really want to make sure there’s nothing in Miniscript that brings back the scary properties of the underlying language.

Another limitation is policy language is just one of several tools needed to make very complicated multisig wallets a practical reality. There are still questions left to answer, such as: How exactly do you do this setup? What are you emailing to each other? Are you emailing your keys or are you emailing something a little bit more abstract that you agree on first and then you exchange keys? These are practical things that aren’t solved inside a Miniscript.

At the time of writing, integrating Miniscript into the Bitcoin Core wallet is still very much a work in progress.^[<https://github.com/bitcoin/bitcoin/pull/24149>]
