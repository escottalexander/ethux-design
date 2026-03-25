---
title: "Shared UX Patterns"
description: "Cross-cutting patterns referenced by all skill files: loading states, confirmations, empty states, error formatting, address display, stuck transactions, and EIP-5792 capability detection."
---

# Shared UX Patterns

These patterns apply across all skill files. Individual skills reference this file for consistent handling of common UX scenarios. Every async operation, every post-action state, and every error message should follow these standards.

---

## Loading States

Every async operation must have a defined loading state. NEVER show blank space, "$0.00", "undefined", or flash content while data loads.

### Data Loading (read operations)
Show a skeleton/shimmer placeholder in the exact layout the data will occupy. As individual pieces resolve, populate them incrementally. Do not wait for all data before showing any data.

Examples:
- Balance fetching: shimmer placeholder where the balance number will appear
- Allowance checking: disable the action button with text "Checking permissions..."
- ENS resolution: show the hex address immediately, replace with ENS name when resolved. Show a subtle spinner next to the address during resolution.
- Cross-chain balances: show chain icons/names with shimmer placeholders for amounts. Populate each chain as it resolves.

### Button/Action Loading (write operations)
Disable the button and replace its label with a spinner + context text:
- "Estimating fee..." (gas estimation in progress)
- "Waiting for signature..." (wallet popup is open)
- "Confirming..." (transaction submitted, waiting for block inclusion)

NEVER show a submit/confirm button with no fee estimate displayed. The button should remain disabled until the fee is known.

### Calculation Loading
For derived values (fiat conversion, swap output estimates, gas estimates):
- Show "Estimating..." or a shimmer in the value area
- NEVER show "$0.00" or "0" while calculating

---

## Post-Action Confirmation

After every signature or transaction submission, show a confirmation state. NEVER move immediately to the next step or show nothing.

### For on-chain transactions:
```
Transaction submitted. Waiting for confirmation...
Estimated wait: ~[X] seconds
[Transaction hash as truncated link to block explorer] [Copy button]
```

### For off-chain signatures:
```
Signed successfully. [Next step description or completion message]
```

### After transaction confirmation:
Show the actual fee paid alongside the estimate: "Fee: $0.08 (estimated $0.12)." This builds trust.

### Rules:
- NEVER show a "success" state for a submitted-but-unconfirmed transaction. "Submitted" and "confirmed" are different states. Show "Processing..." until the transaction is included in a block.
- NEVER auto-redirect the user away from a confirmation screen. Let them stay until they choose to navigate away.
- NEVER describe a signature as "free" without qualification. Off-chain signatures are gasless but they DO authorize actions. Say "No network fee" instead of "Free."

---

## Empty States

Every list, data view, or balance display must define what appears when there is no data. NEVER render a blank area.

| Context | Empty state message |
|---------|-------------------|
| Zero token balance | "You don't have any tokens yet. [Buy/bridge tokens] to get started." |
| No transaction history | "No transactions yet. Your activity will appear here." |
| No active approvals | "You haven't given any permissions yet." |
| No wallets detected | Show embedded wallet option and WalletConnect. Add "What is a wallet?" link. |
| No search results | "No results found. Try a different search term." |

---

## Error Message Formatting

Use three-tier progressive disclosure for all errors:

1. **First layer (always visible):** Plain-language summary. No jargon.
2. **Second layer (expandable):** More context and suggested action.
3. **Third layer (copy-able):** Full technical error, transaction hash, and raw message for support.

### Error Jargon Translation Table

Apply this translation to every RPC or contract error before displaying to users:

| Technical error | User-facing message |
|---|---|
| Execution reverted | Transaction failed. |
| Insufficient output amount | Price changed too much. Try increasing your price tolerance. |
| Nonce too low | This transaction was already processed. Refresh and try again. |
| Insufficient funds for gas | You don't have enough [native token] to cover the network fee. |
| User rejected transaction / User rejected | You cancelled the transaction. |
| Out of gas | The transaction ran out of processing power. Try again. |
| Contract execution error | Something went wrong with this operation. |
| Transfer from failed | The token transfer was rejected. Check your balance and permissions. |
| Overflow / Underflow | The amount is outside the allowed range. Try a smaller value. |

### Rules:
- NEVER show raw Solidity error strings to end users in the primary display.
- NEVER show gas cost in scientific notation or with more than 6 decimal places. Format for readability: "$0.12" or "0.00004 ETH", never "4.2e-5 ETH".
- NEVER show "Fee: $0.00" for sponsored transactions without the "(sponsored)" label.

---

## Address Display Standards

Every displayed address must follow these rules:

### Truncation
Default format: `0x1234...5678` (first 6 + last 4 characters after 0x). Use this in headers, lists, and non-critical contexts.

### Full address
Show the full address in critical contexts: send confirmation screens, approval targets, and anywhere the user must verify an exact address.

### ENS resolution
When available, show ENS name as primary with truncated address as secondary:
```
vitalik.eth (0xd8dA...6045)
```
Use `useEnsName` (wagmi) or `getEnsName` (viem) for reverse resolution. Also resolve ENS avatar if available.

### Interactive elements
- **Copy button:** Every displayed address must be copyable with a click/tap. Show brief "Copied!" feedback.
- **Explorer link:** Link the address to the relevant block explorer for the current chain.
- **QR code:** For receive flows and mobile-to-mobile scenarios, provide QR code access.

### Input validation
After pasting an address into an input field, immediately validate format and checksum. Trim whitespace from pasted content.

---

## Stuck Transaction Handling

When a transaction has been pending longer than expected:

| Time pending | Action |
|---|---|
| > 2 minutes | Show explanation: "Your transaction is still processing. This is taking longer than usual, which can happen during high network activity." |
| > 5 minutes | Show options: "Still waiting. You can: **Speed up** (resubmit with higher fee, ~$X.XX) or **Cancel** (submit a zero-value transaction with the same nonce, ~$X.XX)." |

### Rules:
- NEVER let a pending transaction spinner run indefinitely without context.
- Show estimated additional cost for speed-up before the user confirms.
- If the user refreshes during a pending transaction, detect and restore the pending state. Show where they left off.

---

## EIP-5792 Capability Detection

This is the canonical explanation. Other skill files reference this section instead of re-explaining.

EIP-5792 defines `wallet_sendCalls` (batch multiple calls in one user confirmation) and `wallet_getCapabilities` (discover what the wallet supports).

### Detection pattern:
```
1. Call wallet_getCapabilities (or useCapabilities from wagmi).
2. Response is keyed by chainId. Check the current chain's capabilities.
3. Look for specific capabilities:
   - atomic: wallet can execute multiple calls atomically
   - paymasterService: wallet supports ERC-7677 paymaster sponsorship
   - auxiliaryFunds: wallet can source funds from other chains
4. If wallet_getCapabilities is not available or throws, the wallet does not support EIP-5792.
```

### Usage:
```
// With wagmi
const { data: capabilities } = useCapabilities()
const currentChainCaps = capabilities?.[chainId]

// Check specific capability
const supportsBatching = currentChainCaps?.atomic?.status === "supported"
const supportsPaymaster = currentChainCaps?.paymasterService?.status === "supported"
```

### Decision tree:
```
Check atomic.status on the current chain:
  "supported" -> Wallet can batch now. Use wallet_sendCalls with multiple calls.
  "ready"     -> Wallet can upgrade via EIP-7702 to gain batching. Prompt upgrade, then batch.
  undefined   -> No batching support. Fall back to sequential transactions.
```

### Fallback:
If EIP-5792 is not supported, degrade gracefully:
- `atomic.status` is `"supported"`: use `wallet_sendCalls` with multiple calls
- `atomic.status` is `"ready"`: wallet can upgrade via EIP-7702 to gain batching. Prompt the user to upgrade, then use `wallet_sendCalls`
- No `atomic` capability: use sequential transactions (approve, then action)
- No `paymasterService`: user pays gas in native token
- NEVER block functionality because EIP-5792 is unavailable. Always implement a non-5792 path.

---

## Irreversibility Framing

Before every value-transferring transaction, include a clear but non-alarming note: "This action cannot be undone once confirmed." Do NOT use red warning colors for this. Use muted text below the confirm button.

For approvals (which CAN be revoked): "You can remove this permission later from your approvals page."

---

## Network Failure and Offline States

Detect network connectivity. If the connection drops:
- Immediately disable all submit/confirm buttons.
- Show an inline banner: "You're offline. Reconnect to continue."
- Preserve all input state so the user does not lose their work.

On reconnection:
- Re-enable buttons.
- Refresh stale data (balances, gas estimates, nonce).
- Show "Back online" briefly.

NEVER submit a transaction while offline.

---

## Accessibility Baseline

These requirements apply to every component and screen across all skills.

- **Status changes:** Use `aria-live="polite"` for all status transitions (loading to loaded, submitted to confirmed). Screen readers will announce the new state without interrupting the user.
- **Error messages:** Use `role="alert"` on error message containers so they are announced immediately by assistive technology.
- **Loading placeholders:** Set `aria-busy="true"` on skeleton/shimmer containers while data is loading. Remove the attribute when content populates.
- **Icon-only buttons:** Every icon-only button must have screen reader text via `aria-label` or a visually hidden `<span>`. Examples: copy button (`aria-label="Copy address"`), explorer link (`aria-label="View on block explorer"`), close button (`aria-label="Close"`).
- **Focus management:** After async operations complete (transaction confirmed, data loaded, modal opened), move focus to the new content or confirmation message. Do not leave focus on a disabled or removed element.
- **Non-color indicators:** Warnings and errors must use icon + text label + color together. Never rely on color alone to convey severity or state.
- **Keyboard navigation:** All interactive elements must be reachable via Tab. All modals, dropdowns, and bottom sheets must be dismissible via Escape. Focus must be trapped inside open modals.
- **Skip links:** For long repeated content (solution tables, checklist items, approval lists), provide a "Skip to [section]" link at the top of the repeated block.

---

## Mobile & Touch

These requirements apply to every component and screen when rendered on touch devices.

- **Touch targets:** Minimum 24x24px for all tappable elements (WCAG AA). Recommend 44x44px for primary actions and frequently used controls.
- **Thumb zone:** Place primary actions (confirm, submit, next) in the bottom 40% of the screen on mobile layouts. Secondary actions and navigation can be higher.
- **Bottom sheets over modals:** On mobile, prefer bottom sheets over centered modals for confirmations, wallet selection, and option menus. Bottom sheets are easier to dismiss and stay within thumb reach.
- **No hover-dependent interactions:** Tooltips, info popovers, and contextual help must be tap-to-toggle on touch devices. Never require hover to reveal critical information.
- **Swipe gestures:** Support swipe-to-dismiss for bottom sheets and notifications. Ensure swipe targets do not conflict with horizontal scroll areas.
- **Viewport:** All content must be usable without horizontal scroll at 320px viewport width. Test all layouts at this minimum.
- **Input zoom prevention:** Set font-size to minimum 16px on all input fields. iOS auto-zooms the viewport when focusing inputs with font-size below 16px, which disorients users.
