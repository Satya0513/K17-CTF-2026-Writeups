# K17 CTF 2026 — Blowfish
Category: Crypto
Difficulty: Hard
## The Challenge

The challenge gave us a service where we could interact with something called a "fish".
The first thing I noticed was that the data was split into 8-byte blocks, which immediately made sense because Blowfish has a block size of 64 bits.
The service also gave us two options:

```text
1. blow
2. unblow
```

Along with the fish, we were given a signature.
At first, I thought this would probably be a straightforward Blowfish-related crypto challenge. But after looking at how the signatures were generated, things became much more interesting.

## Looking at the Signature

The fish was divided into 8-byte blocks, and the signature wasn't one MAC over the entire fish.

Instead, it looked like the server was doing something similar to:

```text
Block 1 → MAC 1
Block 2 → MAC 2
Block 3 → MAC 3
...
```

and then simply joining all those MACs together:

```text
MAC(Block 1) || MAC(Block 2) || MAC(Block 3) || ...
```

That immediately raised a question for me:

> If every block is authenticated separately, does the signature actually care about where the block appears?

If the answer was no, then maybe I could take valid signed blocks and rearrange or reuse them.

That was the first weakness.

---

## The More Interesting Part

The challenge also gave us `blow` and `unblow` oracles.

They didn't just transform the data. They returned the resulting blocks with fresh valid signatures.

So we had something like:

```text
Input blocks
     ↓
 blow / unblow
     ↓
New blocks + valid signatures
```

This was interesting because it meant I wasn't simply trying to forge a MAC myself.

I could ask the server to generate signed intermediate values for me.

At this point I started looking at whether the two operations could be combined somehow.

---

## Playing With the Oracles

Suppose we have three signed blocks:

```text
A
B
X
```

I first send:

```text
A || B
```

to the `blow` operation.

Because of the way the challenge's Blowfish operation worked, the second resulting block gives us a value based on:

```text
B ⊕ A
```

Let's call that new signed block:

```text
C = E(B ⊕ A)
```

Now comes the interesting part.

I take:

```text
X || C
```

and send that to `unblow`.

The second decrypted block becomes:

```text
D(C) ⊕ X
```

Since `C` was created from `B ⊕ A`, this gives:

```text
(B ⊕ A) ⊕ X
```

which is:

```text
A ⊕ B ⊕ X
```

So we have effectively turned:

```text
A, B, X
```

into:

```text
A ⊕ B ⊕ X
```

and, importantly, the server gives us a valid signature for the resulting block.

So the oracle combination was effectively giving us an XOR operation on authenticated blocks.

---

## Why This Was Useful

Once I realized this, the challenge became more of a linear algebra problem.

We already had a collection of signed 64-bit blocks.

The question became:

> Can I represent the block I want as an XOR of the blocks I already have?

The target we ultimately wanted was:

```json
{"admin":true}
```

Because Blowfish works on 8-byte blocks, I padded it with JSON whitespace:

```python
admin_json = b'{"admin":true}  '
```

That gives us two 8-byte blocks:

```text
{"admin":
true}··
```

where `·` represents the padding spaces.

Now we needed to construct signed blocks equal to those two values.

---

## Turning It Into Linear Algebra

Every block is 64 bits, and XOR is basically addition in GF(2).

So instead of treating the blocks as normal integers, I could treat them as vectors.

The goal was:

```text
target = block1 ⊕ block2 ⊕ block3 ⊕ ...
```

I used Gaussian elimination to find which of the original blocks could be combined to produce each target block.

There was one extra detail.

Our oracle construction combines three blocks at a time, so every resulting value is an XOR of an odd number of original blocks.

To keep track of that constraint, I added an extra parity bit:

```python
value = int.from_bytes(block, "big") | (1 << 64)
```

So each value effectively became:

```text
64 bits → actual block
1 bit  → parity
```

The solver could then find an odd-sized combination that produced the target.

---

## Building the Target Blocks

Once I had the required combinations, I could repeatedly use the oracle trick.

For example:

```text
A ⊕ B ⊕ C
```

could be generated first.

Then that result could be combined with two more blocks:

```text
(A ⊕ B ⊕ C) ⊕ D ⊕ E
```

and so on.

This allowed me to gradually construct the exact blocks I needed while still getting valid signatures from the server.

The important thing was that I wasn't trying to calculate the signature myself.

I was making the server sign the intermediate values for me.

---

## The Final Payload

After generating the two target blocks, the final payload looked conceptually like:

```text
[decoy block]
[{"admin":]
[true}··]
```

The first block wasn't important to the JSON value we wanted.

The important part was that the target JSON blocks were correctly signed.

I then submitted the reconstructed fish through the normal flow.

The server accepted it and interpreted the JSON as:

```json
{
    "admin": true
}
```

And that was enough to reach the admin path.

---

## Getting the Flag

The server finally returned:

```text
K17{great_work_infiltrating_as_the_head_fish_perhaps_one_could_call_you_james_pond}
```

---

# Exploit Chain

Looking back, the complete attack was:

```text
Per-block MAC
      ↓
Block position isn't authenticated
      ↓
blow + unblow oracles
      ↓
Create XOR combinations
      ↓
Get fresh signatures for those blocks
      ↓
Solve for target blocks
      ↓
Construct {"admin":true}
      ↓
Submit forged fish
      ↓
Admin access
      ↓
Flag
```

---

# What Actually Went Wrong?

The interesting thing about this challenge is that I wasn't really "breaking Blowfish".

The problem was the way the cryptographic operations were being used by the application.

The authentication was effectively:

```text
MAC(block1) || MAC(block2) || MAC(block3)
```

instead of authenticating the complete message:

```text
HMAC(key, complete_message)
```

Because the blocks were authenticated independently, the application didn't properly bind the complete sequence together.

Then the `blow` and `unblow` oracles made the situation worse because they allowed us to obtain valid signatures for values derived from existing signed blocks.

So the real weakness was in the protocol design, not in the underlying Blowfish algorithm.

---

# What I Took Away From This Challenge

The biggest thing I took from this challenge was to not immediately assume that a challenge is about breaking the algorithm mentioned in its name.

When I see a crypto challenge, I now try to look at the whole construction:

* What exactly is being authenticated?
* Is the entire message authenticated?
* Are blocks authenticated independently?
* Does the authentication include position or context?
* Are there encryption or decryption oracles?
* Can I submit previously signed values back to the service?
* Can I combine multiple operations to create something the developer never intended?

In this challenge, the important breakthrough wasn't some complicated mathematical attack on Blowfish.

It was noticing that the authentication model and the available oracles could be combined in an unintended way.

That was enough to turn valid authenticated blocks into a valid admin payload.

