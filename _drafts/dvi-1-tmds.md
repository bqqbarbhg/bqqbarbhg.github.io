---
layout: post
title: "DVI - Reducing transitions"
---

## Background

DVI uses TMDS *"Transition-minimized differential signaling"* to reduce electromagnetic
interference over a physical cable. The *[differential signaling][wk-diffsig]* part means
we bundle a signal and it's inverse into a twisted pair wire reducing interference.
For an FPGA implementation the *transition minimized* causes more trouble: TMDS tries
to minimize transitions (rising or falling edges), ie `01` and `10` patterns in the signal.
Minimizing transitions improves signal integrity and somehow helps with clock recovery,
but that is way out of my league.

## 8b/10b encoding

TMDS encodes each 8-bit `word` using a 10-bit `code`, we start with computing two possible
encodings for the 8-bit data `word`: One accumulates `XOR(code[n-1], word[n])` and the other one
`XNOR(code[n-1], word)`, the initial bit is copied straight from the input `word` in both.

```python
# Compute both XOR and XNOR encodings
code_xor[0] = word[0]
code_xnor[0] = word[0]
for n in range(1, 8):
    code_xor[n] = code_xor[n - 1] ^ word[n]
    code_xnor[n] = ~(code_xor[n - 1] ^ word[n])
```

We can treat these two encodings as a black box for now, but assume that they probably have a
different number of transitions. We're going to indicate which encoding we chose in the ninth
bit `code[8]` where `1` is XOR and `0` is XNOR.

```python
def count_transitions(bits):
    """Returns the number of 01/10 patterns"""
    count = 0
    for n in range(len(bits) - 1):
        if bits[n] != bits[n + 1]:
            count += 1
    return count

# Use the encoding which has fewer transitions
use_xnor = num_transitions(code_xnor) < num_transitions(code_xor)
code[0:8] = code_xnor if use_xnor else code_xor
code[8] = 0 if use_xnor else 1
```

We want to use the other control bit to indicate if we should invert `code[0:8]`. For electrical
purposes we want to send as many zeros as ones (or as close as possible) to keep a zero *DC bias*.
The encoder keeps a counter `dc_bias` that contains the difference of ones to zeros encoded, for
example after sending `100100` the bias would be -2. By inverting the code word if necessary we
can reduce the bias (or at least *not* increase it). If either the accumulated or code word bias
are zero it doesn't really matter which way we choose, so we use the inverse of `code[8]` to keep
the control bits `code[8:10]` unbiased (either `01` or `10`).

```python
def bit_bias(bits):
    """Returns the difference of high bits to low bits set"""
    bias = 0
    for n in range(len(bits)):
        bias += +1 if bits[n] else -1
    return count

# Count the bias and invert the data word if necessary
code_bias = bit_bias(code[0:8])
if dc_bias == 0 or code_bias == 0:
    # No bias in `code[0:8]` encode control bits as either `01` or `10`
    invert = not code[8]
else:
    # Invert `code[0:8]` if the bias has a different sign than `dc_bias`
    invert = (dc_bias < 0 and code_bias > 0) or (dc_bias > 0 and code_bias < 0)

code[0:8] = ~code_word if invert else code_word
code[9] = invert

# Accumulate DC bias for the final 10-bit code to be sent
dc_bias += bit_bias(code)
```

## The dreaded flowchart

That's really all there is to the encoding step of TMDS as described by the DVI specification text,
however the spec includes a neat flowchart of the encoding algorithm that does not match the description.

Instead of selecting between the XOR and XNOR encoding by which one has fewer transitions we select XNOR
if the input data `word` has more than 4 transitions *or* it has exactly 4 transitions and `word[0] == 0`.

```python
# use_xnor = num_transitions(code_xnor) < num_transitions(code_xnor)
word_bits = count_bits(word)
use_xnor = word_bits > 4 or (word_bits == 4 and word[0] == 0)
```

This bothered me as I couldn't see the connection between transitions and the input bits, so I started to
think why the specific encodings were chosen. I guess the usual way to think about XOR is that it's true
if only one of the inputs is true. However you can also think of it as the second input conditionally
inverting the first one (or vice versa as it's symmetric).

```python
def xor(a, b):
    return ~a if b else a
```

So now we can interpret the chain of XOR operators: Every `1` bit in the input corresponds to a transition
in the output and every `0` bit keeps the output fixed! Furthermore as `not xor(a, b)` is equivalent to
`xor(a, not b)` we can see that the XNOR case turns all zeroes into transitions. Now the decision is clear:
if there's less ones than zeros in the last 7 bits `code[1:8]` we want to use XOR, otherwise XNOR.
This is equivalent to using XOR up to 3 bits set then switching to XNOR giving us a maximum of three transitions.

```python
# word_bits = count_bits(word)
# use_xnor = word_bits > 4 or (word_bits == 4 and word[0] == 0)
tail_bits = count_bits(word[1:8])
use_xnor = tail_bits >= 4
```

Now this is not exactly the same expression as on the flowchart, but it's close enough that we can show
that it's equivalent by enumerating all the cases, if this doesn't feel correct feel free to expand the section below.

<details markdown="1">
<summary>Case by case analysis</summary>

If `tail_bits<3` both methods don't use XNOR as `word_bits` is at most `tail_bits+1 <= 3`.

If `tail_bits>4` both methods always use XNOR as `word_bits` is at least `tail_bits >= 5`.

If `tail_bits==3` we should never use XNOR. If `code[0]==0` then `word_bits==3` which checks out,
otherwise `word_bits==4` but we don't match the case `(word_bits==4 and word[0]==0)`.

If `tail_bits==4` we should always use XNOR. If `code[0]==0` then `word_bits==4` and we satisfy
the `(word_bits==4 and word[0]==0)` case, otheriwse `word_bits==5 > 4` due to the extra bit.
</details>
