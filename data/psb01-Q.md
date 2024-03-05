### L1 : Typo error in event AdminSet :
**Description :** 

`event AdminSet(address indexed oldAmin, address indexed newAdmin);`

Here in [L48](https://github.com/code-423n4/2024-02-uniswap-foundation/blob/main/src/V3FactoryOwner.sol#L48) by typing mistake oldAmin is written instead of oldAdmin.

**Recommendation :** 

` event AdminSet(address indexed oldAdmin, address indexed newAdmin); `
