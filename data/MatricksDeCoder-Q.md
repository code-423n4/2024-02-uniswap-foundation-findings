#### NC-1 Spelling, typos and grammar - 

There are some spelling, typos, grammar issues that may impact understandability, readability, maintainability of code. Best to check all code for such or run through a spellchecker. Take care to change parts like 'is vs are', capitalizations, typos etc See examples below 

parm  should be param 
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L140

#### NC-2 Time Delays on Critical Changes of Parameters is missing. - 

e.g operation such as changing admin, setting reward notifier, setting fee protocol etc. See examples below 

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L201

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L210

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L131

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L142

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L110

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L119

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L131

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L142


#### NC-3 Setter values should check new value is not equal to the old value - 

Setters should have initial value check. Not only does it save gas it also prevents errors of assuming value has been changed when not changed or affects off-chain reporting tools expected certain number of unique changes etc . ensure the old value != new value checks

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L771

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L210

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L110

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L119

#### NC-4 Event-based reentrancy - 

may result in events being emitted out of order in cases where e.g no reentrancy protection, ERC777 potential reentrancy, events emitted after external calls etc See examples below 

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L638

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L669

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L723

https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L181

#### NC-5 Function naming can be improved for readability - 

functions must be clear and indicative of purpose and not misleading. The functions below may better named

enabelFeeAmount makes it seem as if fee can be enabled and disabled when its a write once permanent. It also makes it seems as if the fee is being enabled to a boolean when its to a value based on tickss

It may be more appropriate to name => setPermanentFeeAmount
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/V3FactoryOwner.sol#L131

#### NC-6 Function parameter use of underscore consistency 

Most function parameters make use of prevailing underscore e.g 
```
function _revertIfSignatureIsNotValidNow(address _signer, bytes32 _hash, bytes memory _signature)
    internal
    view
```
_signer, _hash etc 

However this is not followed consistently in the code see the following examples 

Deposit storage deposit
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L669

Deposit storage deposit
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L688

Maybe above is because its a struct but still its not clear if that's reason

Other examples are below even if above may not be valid 

address owner should be address _owner 
https://github.com/code-423n4/2024-02-uniswap-foundation/blob/5a2761c8277541a24bc551fbd624413b384bea94/src/UniStaker.sol#L787









