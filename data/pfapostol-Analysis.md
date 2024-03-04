### Introduction

The Uniswap ecosystem was familiar to me from previous contests and experience using Solidity.
This contest interested me in the quality of the code base, the use of niche functions of the solidity language (.wrap() and .unwrap()) as well as the relatively simple, but at the same time effective business logic.

### Aproach and thoughts

I started by studying the readme, which was a short documentation that gave enough information to start studying the contracts directly.
Having studied each contract and how they interact, I started writing tests for edge cases: when the depositor withdraw the deposit in the middle of the reward period, but returns later, when the rewards are increased, at the moment when the depositor left staking, and many others. During the testing, I did not find any critical vulnerabilities, as a result of which I started writing a QA report and Analysis.

Overall, the codebase appears to be extremely secure, with the exception of a few shortcomings. Such a high level of code quality was achieved thanks to the systematization of the code (the order of functions corresponds to the lifetime of the project, internal and private functions follow the order of public and external functions that call them). And also 100% test coverage of all functions:

| File                                 | % Lines           | % Statements      | % Branches     | % Funcs         |
|--------------------------------------|-------------------|-------------------|----------------|-----------------|
| src/UniStaker.sol                    | 100.00% (137/137) | 100.00% (155/155) | 95.83% (23/24) | 100.00% (37/37) |
| src/V3FactoryOwner.sol               | 100.00% (21/21)   | 100.00% (27/27)   | 100.00% (8/8)  | 100.00% (6/6)   |


### General overview:


1. `UniStaker`:
    The UniStaker contract manages the distribution of rewards to stakers. It allows users to stake a designated ERC20 governance token and receive rewards denominated in another ERC20 token. Stakers can delegate their voting power and designate beneficiaries to receive rewards on their behalf. Rewards are distributed over a specified duration, and stakers can add or withdraw their stake at any time.

    1. What usable state does this contract hold:
        1. `admin` : -
        2. `totalStaked`: - 
        3. `depositorTotalStaked` : - track amount for each depositor, no use in code
        4. `earningPower`: - current beneficiary  reward rate relative to other beneficiaries
        5. `deposits`: - storage of deposit data
        6. `surrogates`: - 
        7. `rewardEndTime` : -
            1. Started from 0
            2. Updated when reward is notified `notifyRewardAmount`
            3. Has an impact on `lastTimeRewardDistributed` getter
            4. Has an impact on partial rewards re-balancing in `notifyRewardAmount`
    2. Logic:
        1. Graph displaying how main external functions affect or get affected by state variables:
        
        ![UniStaker Stake Stuff.jpg](https://private-user-images.githubusercontent.com/50257230/309678615-0f5a88b6-4376-4a17-9313-106ff9eb9a29.jpg?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk1NDAzNzcsIm5iZiI6MTcwOTU0MDA3NywicGF0aCI6Ii81MDI1NzIzMC8zMDk2Nzg2MTUtMGY1YTg4YjYtNDM3Ni00YTE3LTkzMTMtMTA2ZmY5ZWI5YTI5LmpwZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA0VDA4MTQzN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWQ5ZThjNWQ5YjMzZmQyMGI4NzVkNjVmNjY0ZjEwZThhMjk2OTlkNmE0YmEzMGU4NjdlMzU5NzY1YzZmNWI0MDMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.7Ih4fSW_Z5pRYkNiYsZmYGqCMHyCBcE267OBFsfJxk0)
    
        2. abbreviations:
            1. RET — `rewardEndTime`
            2. BT — `block.timestamp`
            3. RPTA — `rewardPerTokenAccumulated`
            4. RPTA(c) — `rewardPerTokenAccumulatedCheckpoint`
            5. SRR — `scaledRewardRate`
            6. LTRD — `lastTimeRewardDistributed`
            7. LCT — `lastCheckpointTime`
            8. TS — `totalStaked`
            9. URC(b) — `unclaimedRewardCheckpoint` for beneficiary
            10. EP(b) — `earningPower` for beneficiary
            11. BRPTC(b) — `beneficiaryRewardPerTokenCheckpoint` for beneficiary
        3. `lastTimeRewardDistributed`:
            1. $\begin{cases} RET &\text{ if RET <= BT} \\ BT &\text{ else}\end{cases}$
            2. Has impact on current value of reward per staked token `rewardPerTokenAccumulated`
            3. Has impact on `lastCheckpointTime` each time when `_checkpointGlobalReward` called 
        4. `rewardPerTokenAccumulated`:
            1. $\begin{cases}RPTA(c) + \dfrac{(SRR * (LTRD - LCT))}{TS} &\text{ if TS } \ne 0 \\ RPTA(c) &\text{ if TS} = 0 \end{cases}$
            2. Affected by `rewardPerTokenAccumulatedCheckpoint`, which depend on itself via `_checkpointGlobalReward` and `notifyRewardAmount`
        5. `unclaimedReward`:
            1. $URC(b)+\dfrac{(EP(b)*(RPTA-BRPTC(b)))}{SCALE\_FACTOR}$
        6. `_checkpointGlobalReward`:
            1. Update $RPTA(c)$ with  `rewardPerTokenAccumulated`
            2. Update $LCT$ with `lastTimeRewardDistributed`
            3. Checkpoints used at:
                1. staking
                2. increasing stake
                3. beneficiary change
                4. withdrawing stake
                5. claiming rewards
            4. Not used when:
                1. delegatee change - since delegate is not affected by rewards, but only by voting power of stake
        7. `_checkpointReward`:
            1. Update $URC(b)$ with `unclaimedReward`
            2. Update $BRPTC(b)$ with $RPTA(c)$
    3. RBAC:
        1. Only admin can:
            1. `setAdmin`
            2. `setRewardNotifier`
        2. Owner of the deposit:
            1. `stakeMore`and signature alternatives
                1. Call `_checkpointGlobalReward` and `_checkpointReward` before staking
                2. Transfer token, update state
            2. `alterDelegatee`and signature alternatives
            3. `alterBeneficiary` and signature alternatives
                1. Call `_checkpointGlobalReward` and `_checkpointReward`(for both old and new beneficiary) before altering beneficiary
            4. `withdraw` and signature alternatives
                1. Call `_checkpointGlobalReward` and `_checkpointReward` before withdrawing
        3. Beneficiary of deposit:
            1. `claimReward`
                1. Call `_checkpointGlobalReward` and `_checkpointReward` before claiming
                2. Set $URC(b)$ to 0
        4. Reward Notifier:
            1. `notifyRewardAmount`:
                1. Adjust rewards rate and duration in `UniStaker` 
        5. Public Access:
            1. `stake` and signature alternatives
                1. Call `_checkpointGlobalReward` and `_checkpointReward` before staking
                2. Call `_fetchOrDeploySurrogate`
                3. Transfer token, update state
2. `V3FactoryOwner`:
    1. What usable state does this contract hold:
        1. `payoutAmount` : - The price of claiming fee from the pools, controlled by the factory, controlled by the factory owner contract
        2. `admin` : - Uniswap timelock governance 
    2. RBAC:
        1. Admin:
            1. `setAdmin`
            2. `setPayoutAmount`
            3. `enableFeeAmount` : - pass to controlled factory
            4. `setFeeProtocol` : - pass to controlled pool
        2. Public Access:
            1. `claimFees` : - transfer `payoutAmount` to `UniStaker` and collect fees from protocol



### Risks indentifying:

Contract itself contains a lot of functions that becomes weak points if admin is corrupted. 
 - Admin can disable reward notifiers in `UniStaker`, which block the whole contract. 
 - Admin of `V3FactoryOwner` can stop reward to ever being completly distributed, by making `payoutAmount` to low (in this case reward would be rebalanced much frequently) or to high, in this case no one will ever provide rewards.
But all this prevented externally, since admin is set to Uniswap timelock governance.

No systemic risk detected, since both tokens WETH and UNI are controlled. But if instead of WETH will be used another token (which is possible according to sponsors answers in discord) reentrancy maybe possible, but not feasible.

Integration risks was not detected, since all contract for interaction is either Uniswap ecosytem or WETH token, so everything works smoothly. As a minor risk i can suggest the absence of the upgradebility functionality, while sponsor does not considered Upgradability as required, it can provide better security in exchange for some gas on deploying.

### Time spent:
32 hours