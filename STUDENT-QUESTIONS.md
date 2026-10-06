# STUDENT-QUESTIONS.md — Discussion questions (submit with your repo)

Answer directly under each question. 150–300 words each — **reasoning over length**.

---

## A. Permission design

**A1.** The vault holds `MINTER_ROLE`, so it can `burn` any user's balance. Explain why that is a risk, then write out how you would change `Vault` and `SimpleStablecoin` to remove it.

> Your answer:
Minting and burning are both controlled by `MINTER_ROLE`, and the Vault holds that role. This gives the protocol the power to burn any user's balance without their approval. If the Vault is compromised, its private key leaks, or the role is mistakenly assigned to an untrusted address, an attacker could destroy arbitrary sUSD instantly. That breaks the fundamental trust that a user's balance cannot be confiscated unilaterally by the protocol.
To fix it, I would separate the minting privilege from the privilege to burn other users' balances, and more importantly restrict which balances can be burned. A normal `burn()` should only burn the caller's own balance, so that no third party can touch a user's tokens. For redemption, the Vault should use a dedicated burn path that only burns sUSD already transferred into the Vault, rather than naming an arbitrary user. Or use `burnFrom()`, which requires the user to approve the Vault first.
<br><br><br>

**A2.** In this contract `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE` and `PAUSER_ROLE` all go to the same address. How would you split them in production, and who holds each?

> Your answer:
I will separate these roles based on their responsibilities. `DEFAULT_ADMIN_ROLE` should be held an admin account with a high signing threshold and a time delay. It should only manage role assignments and critical governance settings, and should not perform routine minting or emergency actions. `MINTER_ROLE` should be given to the audited Vault or a dedicated minting contract, which can mint only according to the collateral and accounting rules, and should not have governance authority. `PAUSER_ROLE` should be given to a separate emergency-security account or multisig, which needs to pause the system quickly during an exploit but cannot mint tokens or change role assignments.
<br><br><br>

---

## B. Pausing and redemption

**B1.** `_update` is the single entry point for every balance change, so `pause()` freezes transfers, minting and redemption together. If you wanted "pause transfers but **allow redemption**", how would you change it? Give the approach — full code not required.

> Your answer:
I would decouple transfer pausing from redemption pausing instead of applying one condition to every balance change. The simplest approach is two independent flags, to paused transfers and to paused Redemption. Normal ERC-20 `transfer` and `transferFrom` would check only `pausedTransfers`, while the redemption function checks only `pausedRedemption`. That way the protocol can stop normal transfers during an incident while keeping the redemption channel open. Another approach is to give redemption its own burn entry point that bypasses the transfer pause and only burns sUSD already held by the Vault. 
<br><br><br>

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

> Your answer:
In this cases, closing redemption can stop outflows short-term, but it removes the user's ability to convert and weakens the stablecoin's core promise: redeemability. Even if the on-chain invariant still holds, the token can trade below $1. In 2008, the money-market fund froze redemptions, slowing the run but trapping investors and deepening uncertainty. In 2023, USDC stayed redeemable, participants could still convert, and arbitrageurs bought discounted USDC to redeemed it, the price recovered toward $1 once reserves were accessible. Redemption is both a liquidity exit and a credibility anchor, while closing it contains pressure but damages trust, keeping it open exposes reserve problems faster but preserves convertibility and the arbitrage that restores the peg.
<br><br><br>

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

> Your answer:
The first is undercollateralization: the collateral loses value or is stolen, or unbacked minting increases `totalSupply()`, such as an attacker with `MINTER_ROLE` minting tokens without collateral, until `totalCollateral() < totalSupply()`. This is a depeg that detect directly. The second is redemption or liquidity problem: there is enough collateral and `totalCollateral() >= totalSupply()` still holds, but redemption is paused, delayed, or the collateral cannot be sold. Users cannot convert their tokens, so the token may trade below $1. The invariant cannot detect this type of depeg.
<br><br><br>

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

> Your answer:
Flow of funds: the attacker obtains `MINTER_ROLE` via a leaked key or misconfiguration, calls `mint()` to create 1,000,000 sUSD from nothing, which leads `totalSupply()` rises, while `totalCollateral()` does not, then calls `vault.redeem()` to burn sUSD and receive real mUSDC. The attacker exits with real collateral while the remaining supply loses backing, eventually reaching `totalCollateral() < totalSupply()`. 
The first and most fundamental defense is at the minting step: an untrusted address must never hold `MINTER_ROLE`. A second defense is to make minting conditional on an actual collateral deposit, so unbacked sUSD cannot enter the redemption path.
<br><br><br>

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

> Your answer:
Not. MockUSDC is an on-chain ERC-20 the Vault can read via `balanceOf()`, but treasuries are held off-chain by custodians, so the contract cannot verify their existence from a token balance. 
New problems: (1) the assets must be held by a custodian or legal entity; (2) `totalCollateral()` becomes an estimate that need market value, interest, and discounts, relying on external pricing; (3) Treasuries cannot be instantly transferred like ERC-20, so redemption needs traditional settlement, the invariant changes from a directly verifiable on-chain balance to a collateral value that depends on custody, valuation, and external attestations.
<br><br><br>

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

> Your answer:
Use an RWA tokenization structure. First create an SPV that legally owns the building, then issue an on-chain token representing a claim on that entity, which the Vault accepts as collateral. Off-chain roles includes the SPV that holds title, provides legal separation, a custodian that manages the building and income, an independent appraiser to do periodic valuation, a liquidation party that sells and transfers the property if collateral is insufficient and corresponding laws that deal with title, investor rights.
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
The most likely attack is the incorrect assignment of `MINTER_ROLE`. This is a path that verified in Ex3: after granting the attacker `MINTER_ROLE`, he can call `stable.mint` to create sUSD from nothing, pushing `totalSupply()` to 1e12 while `totalCollateral()` stayed 0, instantly breaking `totalCollateral() >= totalSupply()`. 
I chose it for two reasons. First, it hits the root of the `SimpleStablecoin` permission design that minting is decoupled from collateral and the role is a single point. Second, it causes direct economic loss at near-zero cost without stealing user funds.