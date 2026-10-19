# STUDENT-QUESTIONS.md — Discussion questions (submit with your repo)

Answer directly under each question. 150–300 words each — **reasoning over length**.

---

## A. Permission design

**A1.** The vault holds `MINTER_ROLE`, so it can `burn` any user's balance. Explain why that is a risk, then write out how you would change `Vault` and `SimpleStablecoin` to remove it.

> Your answer:
This is a risk because a user's sUSD is their only claim on the collateral, and `burn(from, amount)` does not ask that user for an allowance. The vault holds `MINTER_ROLE`, so it can destroy any address's balance in one call. Once that balance is gone, `redeem` cannot help: it burns the caller's own sUSD, and the user no longer has any. The collateral they deposited stays in the vault, but they have no token left to withdraw it. The holder cannot refuse the burn, and cannot move the tokens out first, because the burn does not need their signature.

The same role can also mint. Whoever holds it can burn other users' sUSD, mint the same amount to themselves, and then call `redeem` to take the collateral those users can no longer claim. The burned balance is not just a smaller number. It turns the victim's deposit into collateral that the role holder can redeem.

Now `Vault.redeem` passes `msg.sender`, so an ordinary redemption does not burn someone else. The dangerous permission sits on the token: `burn` accepts any `from`, and the vault is allowed to call it. A rewritten or upgraded vault, a leaked deployer key, or a mistaken `grantRole` is enough to use that permission.

I would make a burn apply only to tokens the user has surrendered. A normal `burn` on `SimpleStablecoin` should destroy only the caller's own balance and should stop taking an arbitrary `from`. For redemption, the user first transfers sUSD to the vault or approves that amount, and the vault burns only those surrendered tokens. Keep `mint` granted only to the vault, and require `deposit` to transfer collateral in before minting, so the same role cannot print unbacked sUSD and redeem it.
<br><br><br>

**A2.** In this contract `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE` and `PAUSER_ROLE` all go to the same address. How would you split them in production, and who holds each?

> Your answer:
I would split the three roles by what each one is allowed to break. Now the deployer receives all of them in the constructor, so one key can grant further roles, mint or burn any balance, and freeze every transfer. Once the token has outside holders, one compromise becomes a full compromise.

`DEFAULT_ADMIN_ROLE` should sit with a governance multisig behind a timelock. Its job is to grant and revoke the other roles and to change settings that are not part of daily flow. `MINTER_ROLE` should be held by the vault contract, not by a person. The vault may mint only inside `deposit`, after `transferFrom` has successfully brought collateral in; role separation alone does not guarantee this order, so the contract must also prevent other minting paths that bypass the deposit flow. Its burn path should be limited as in A1. `PAUSER_ROLE` should be a separate emergency multisig that can be reached quickly. It may pause and unpause, and nothing else: it cannot mint, cannot rewrite roles, and cannot move collateral.

A leaked pause key can then freeze transfers but cannot print tokens or appoint a new minter. Governance can still rotate keys, but only after the delay, so holders can exit before a hostile role change takes effect.
<br><br><br>

---

## B. Pausing and redemption

**B1.** `_update` is the single entry point for every balance change, so `pause()` freezes transfers, minting and redemption together. If you wanted "pause transfers but **allow redemption**", how would you change it? Give the approach — full code not required.

> Your answer:
I would treat transfers, minting, and redemption as three distinct operations rather than using one pause switch for every balance change. I would use separate flags: `transfersPaused`, `mintingPaused`, and `redemptionPaused`.

Ordinary transfers should check only `transfersPaused`; minting should check `mintingPaused` and must still require collateral to arrive first. Redemption should check `redemptionPaused` inside `Vault.redeem`. This means pausing ordinary transfers does not automatically disable redemption, and pausing redemption does not necessarily freeze every other operation.

Importantly, burning must not be exempt from all pause restrictions merely because `to == address(0)`. Ordinary burning and redemption must have explicit authorization and execution-path constraints. For example, only the vault should be able to perform the burn required for redemption, and the amount burned must correspond to a valid redemption request. Otherwise, an address holding `MINTER_ROLE` could still burn tokens arbitrarily while transfers are paused. If `_update` still uses a shared `whenNotPaused` check, the burn inside `Vault.redeem` will also be blocked. Pause checks must therefore distinguish operation types, allowing an authorized redemption burn to pass through `_update` when only ordinary transfers are paused, without permitting arbitrary burns to bypass the restrictions.

This design allows the pauser to control transfers, issuance, and redemption separately. Whether redemption is suspended should be determined by `redemptionPaused` itself, rather than being an unintended side effect of pausing transfers.
<br><br><br>

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

> Your answer:
Closing redemption and leaving it open have different consequences.

In 2008, the Reserve Primary Fund broke the buck following losses connected to Lehman Brothers. Redemptions were suspended, and the fund moved toward liquidation. Investors could not immediately withdraw their money and faced uncertainty about the amount and timing of recovery. Suspending redemption can slow an immediate rush for cash, but it also leaves investors locked into the fund.

In 2023, USDC traded down to about $0.87 after Circle disclosed its exposure to Silicon Valley Bank. Circle did not impose the same formal redemption suspension, but the banking disruption affected access to reserves and the settlement of redemptions. Therefore, a redemption policy remaining in place does not mean every holder can immediately receive dollars.

After banking services resumed and reserves became accessible, arbitrageurs could buy discounted USDC and redeem it for dollars through available channels, helping the market price return toward $1. Open redemption preserved a path back to the peg, but its effectiveness still depended on the actual availability of reserves and users' ability to complete settlement.

The same distinction applies here. A pause that blocks `redeem` can stop collateral from leaving, even while `totalCollateral() == totalSupply()` still holds. The token can still trade below $1 because the market is pricing the closed exit, not just the balance sheet. Leaving redemption open can expose a shortfall faster and preserve an arbitrage path back to the peg when the collateral is real and settlement channels are available.
<br><br><br>

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

> Your answer:
This coin depegs in at least two ways, and one of them shows up in `totalCollateral() >= totalSupply()`.

The first class is undercollateralization. Collateral can be stolen from the vault, or unbacked minting can raise supply without raising collateral. An address with `MINTER_ROLE` calls `mint`, `totalSupply()` rises, and `totalCollateral()` does not. The invariant then fails, and a one-dollar claim is no longer fully backed. Ex3 is this case: supply goes far above collateral.

The second class is a redemption or liquidity failure. Collateral can still sit in the vault, and the invariant can still hold, while users cannot convert. `pause()` freezes `redeem` together with transfers. A broken approval, or collateral that cannot be delivered in time, has the same shape. Holders then sell sUSD for less than one dollar, because the exit is blocked even though the accounting identity is true. The invariant cannot see this. It only compares two numbers. It does not check whether `redeem` can be called.
<br><br><br>

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

> Your answer:
The flow has three stages.

First, the attacker obtains `MINTER_ROLE` by bribing an administrator, exploiting a compromised key, or taking advantage of an incorrect `grantRole`. The attacker then calls `stable.mint` to create 1,000,000 sUSD without transferring any mUSDC into the vault. `totalSupply()` increases while `totalCollateral()` remains unchanged, so the collateral invariant is violated.

Second, the attacker attempts to call `vault.redeem` to withdraw mUSDC. Whether redemption succeeds depends on the collateral actually held by the vault at that time. If the requested amount exceeds the vault's balance, `Vault.redeem` reverts with `InsufficientCollateral()`. The entire transaction is reverted; no partial payment is made. A request can succeed only if sufficient collateral is available and the other redemption conditions are satisfied.

Third, if the contract permits multiple redemption requests and each request satisfies the collateral-balance check and other restrictions, the attacker may gradually drain the collateral the vault can actually pay. Minting 1,000,000 sUSD out of nothing does not guarantee that the attacker can withdraw an equivalent amount of mUSDC.

The key prevention point is minting, not redemption. `MINTER_ROLE` must not be granted to an untrusted address, and the vault must mint sUSD only after `deposit` has successfully transferred in collateral. Redemption must still enforce the collateral-balance check so that each payout stays within the amount the vault can actually pay.
<br><br><br>

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

> Your answer:
No. The current invariant works because MockUSDC is an ERC-20 the vault already holds. `totalCollateral()` is `balanceOf` of the vault, and both tokens use 6 decimals, so one unit of collateral and one unit of sUSD are the same integer. A Treasury bill is not an on-chain balance. It is a claim on the U.S. government, recorded with a custodian. The contract cannot see it by calling `balanceOf`.

Someone off chain has to hold the bills: a custodian, a broker, or a legal entity. The vault's view of how much collateral becomes an attestation or an oracle price, not a storage read. Price is no longer one. Bills trade at a discount, accrue interest, and mature, so the number would be a market value that moves without any token transfer. Redemption also stops being atomic. Selling a bill takes ordinary settlement, so a user who burns sUSD may wait days for dollars. A custodian can also report bills that are lent out or already pledged.
<br><br><br>

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

> Your answer:
What the vault holds is an on-chain token representing rights linked to the building, not the building itself as a `balanceOf`. One possible structure is a special-purpose vehicle (SPV) that holds legal title to the property and issues tokens representing shares or claims on the entity. The vault accepts these RWA tokens as collateral and mints sUSD based on a conservative appraised value and an appropriate collateralization ratio.

This requires several off-chain roles and legal structures. The SPV holds the property title, an independent appraiser periodically assesses its market value, and a trustee or legally authorized agent enforces the collateral arrangement if the debt is not repaid. The property must be sold through a legally enforceable process, and local property law and land registration must recognize the relevant rights.

The vault can track the tokens it holds, but it cannot independently verify the building's market value or guarantee that the property can be sold. Therefore, the system depends on reliable valuation, legally enforceable property rights, and an effective liquidation mechanism.
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex4)

Turn the red tests green in `test/exercises/01_LoopTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Scenario | Your test function name |
|---|---|
| Minting by a non-minter reverts | test_Ex4_Mint_RevertsForNonMinter |
| Transfers revert while paused | test_Ex4_Pause_BlocksTransfers |
| **Redemption** reverts while paused | test_Ex4_Pause_BlocksRedeem |
| An attacker cannot burn someone else's balance | test_Ex4_AttackerCannotBurnOthersBalance |
| ...but the vault holding `MINTER_ROLE` can | test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance |

That last pair is meant to be read together: the guard is written correctly, but the key was handed to the vault. Keep it in mind when you answer A1.

Now write one more scenario you consider **most likely to be attacked**, and say why you picked it:

> Your answer:
The attack I would prioritize is an untrusted address receiving `MINTER_ROLE` and minting sUSD without collateral. Ex3 already demonstrates this path: after receiving the role, the attacker calls `stable.mint` without transferring collateral into the vault. `totalSupply()` increases while `totalCollateral()` remains unchanged, causing the invariant to fail in the same transaction.

I choose this because it exposes a fundamental permission-design problem: minting is not enforced as part of the deposit process. A compromised deployer key, an incorrect `grantRole`, or the role bribery described in C2 could all give an attacker this permission.

The unbacked tokens may be used to redeem real mUSDC, but the amount the attacker can actually withdraw depends on the collateral available in the vault and the redemption logic. A single redemption exceeding the vault's balance reverts in full. The attacker could gradually exhaust the payable collateral only if multiple redemption requests are allowed and each passes the relevant checks.

The main safeguards are to grant `MINTER_ROLE` only to the vault, enforce collateral transfer before minting, and retain strict collateral-balance checks during redemption.