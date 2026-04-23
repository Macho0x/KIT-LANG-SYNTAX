# Kit Lang Synax

In a brief summary, there are four proposed changes/additions:

1. Computation Expressions (CEs) -- examples from F# however repurposed for Kit's modern syntax design with benefits

2. Native Decimals -- inspired by OCaml's Zarith Z & Q. Z is already in BigInt but Q is missing which is now "Decimal". You could probaly put this all under kit-bigint instead of a new kit-decimal module (as proposed) but that's a design choice to make

3. Units of Measure -- from F#

4. More kit-crypto modules: Keccak-256, secp256k1, and RIPEMD-160

Personal preference would be #2 and #4 are definites to be implemented. #1 is a design choice and #3 is not necessary but nice to have.

NOTE: please excuse the Kit Lang syntax if I typed anything wrong but the core concept is sound. I used Kimi K2.6 to help with this and I haven't had a lot of time to thoroughly double check everything so me writing this brief summary one can simply look-up the concepts independently if they don't agree with the doc.
