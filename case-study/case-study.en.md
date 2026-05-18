# ClassiPic — Shipping a Solo iOS App from Concept to App Store (Case Study)

English | [한국어](case-study.ko.md) · [← README](../README.en.md)

> A 13-year iOS senior engineer collaborating with Claude Code and other AI tools shipped a photo-organizer app to the App Store in three months. This is the record of the decisions and trade-offs along the way, the points where the project got stuck and how it got unstuck, and what I'd do differently next time.

- **Output**: [App Store](https://apps.apple.com/app/id6769269007) · [GitHub](https://github.com/sylar-Jeon/classipic-portfolio) · [Privacy Policy](https://sylar-jeon.github.io/classipic-privacy/)
- **Prototype**: [AlbumManager](https://github.com/sylar-Jeon/AlbumManager) — where this project began
- **Duration**: ~3 months (Feb–May 2026)
- **Role**: Solo — design, build, App Store review handling, agreements, and tax setup end-to-end

---

## 1. Why a photo-organizer app

The camera roll was the most irritating UI I looked at every day. iOS Photos is good, but no tool fixes the underlying structure where receipts, business cards, and screenshots are interleaved with memory photos.

Years ago I'd built a portfolio project called **AlbumManager** to scratch this itch. It was a basic classification demo — nowhere near shippable. With AI coding tools maturing rapidly, I wanted to take it out of the drawer and find out, concretely, *how long it actually takes* to bring a prototype to shippable form.

Three goals:

1. **A tool I actually want to use myself** — the strongest motivation
2. **Experience the full solo-shipping cycle** — I'd spent 13 years on iOS dev, but had never personally driven App Store agreements, taxes, and review handling end-to-end
3. **Test the limits of AI collaboration** — find out how far integrating Claude Code into a daily workflow actually goes in practice

---

## 2. Design phase — "What should this app *be*?"

The first decision was **positioning**: replace the iOS Photos app, or complement it?

Replacing it fails 99% of the time. Users are already used to Photos, and you cannot catch up to iCloud sync, shared albums, Memories, and the rest of that gigantic feature bundle from a standalone app. So I committed to the **"complementary app"** position.

That single decision made the five codebase principles fall into place automatically.

| Principle | How it ties to positioning |
|---|---|
| iOS Photos Library = Single Source of Truth | A companion app must see the same data as Photos |
| 100% on-device | Eliminate user resistance to handing photos to another app |
| UI/UX parity with iOS Photos | A companion app should blend with the system app naturally |
| Zero external dependencies | The reality of solo operation — minimize upkeep on dependencies |
| SwiftUI + @Observable MVVM | Same reasoning — aligns with the zero-deps principle |

I codified these five into [`CLAUDE.md`](https://github.com/sylar-Jeon/classipic-portfolio/blob/main/.claude/CLAUDE.md), which became the baseline for every decision and code review from that point on.

---

## 3. Trade-offs — Four key decisions

### Decision 1 — No internal DB; iOS Photos Library as SSOT

Caching metadata in Core Data or SwiftData is the intuitive choice. PhotoKit fetches are known to be slow on large libraries.

But that path means **storing the same information in two places**. Sync bugs are endless. I'd previously watched a similar pattern (timeline data held in-memory and on disk separately) generate bugs for years.

So I chose **zero internal DBs**. `PHAsset` and `PHAssetCollection` are the only truth. iOS handles backup, restore, and iCloud sync. Albums ClassiPic creates also appear in the iOS Photos app — a natural consequence of the companion-app position.

Cost accepted: every fetch hits PhotoKit. First-load latency on large libraries is 0.5–1 second longer. I compensated by caching only *classification results* in a lightweight layer (`ScanRegistry`, in-memory + UserDefaults).

**Generalized principle**: *"If we didn't create the data, we shouldn't own it."*

### Decision 2 — Standardize on @Observable; no TCA, no Combine

This was the most interesting decision. At a previous company, TCA (The Composable Architecture) was introduced to production in 2021 — one of the earliest commercial TCA adoptions in Korea. It was applied across more than 60% of a 310k-line codebase with a team of 6–8 engineers.

So whether to use TCA in ClassiPic was a real question. The answer was **no**.

TCA's real value emerges when **multiple engineers concurrently edit the same domain reducer**. The store/effect model compresses the surface area of merge conflicts, and the action enum forces side effects to live in predictable places. Above a certain threshold of team size and domain complexity, this structure outweighs the boilerplate cost.

ClassiPic is a 1-person, ~70-file project. Far below that threshold. `@Observable` has been stable since iOS 17, and it aligns with the zero-deps principle (adding TCA = +1 SPM dependency, longer build times).

I excluded Combine for the same reason. Swift Concurrency (`async/await`) plus `@Observable`'s automatic tracking handles the reactive flow.

**Generalized principle**: *"Tools depend on team size and domain complexity. The same engineer making different choices on different projects isn't inconsistency — it's the work seniority is supposed to do."*

This decision becomes the answer to an interview question that will inevitably come:
> "You said you have TCA introduction experience. Would you reach for TCA again on a new project?"

### Decision 3 — Zero external dependencies

Zero CocoaPods/SPM dependencies. SF Symbols only for icons and animations.

An extreme choice, but solo operation forced it. Even five dependencies generate weekly update notices, security patches, and breaking changes that eat real time. For a solo operator, that time is development standstill.

And native iOS APIs were rich enough — no external library was genuinely necessary.

Cost accepted: some components had to be built from scratch. The most prominent: `DragSelectableScrollView` — implementing UIKit drag-select gestures inside a SwiftUI ScrollView so the multi-select interaction matches the iOS Photos app exactly. Owning the components means owning the debugging.

During that debugging I hit a Swift 6.3.2 compiler SIL pass bug. That story is in the next section.

Two exceptions: **Google AdMob** and **UserMessagingPlatform**. The ad monetization requirement pulled them in as precompiled frameworks. I accepted the trade-off.

### Decision 4 — Monetization: no subscription, only Lifetime

The market standard for photo-organizer apps is monthly/yearly subscriptions. I chose not to follow the standard.

The reasoning is the nature of the tool. "Photo organization is a one-time job." Locking that into a monthly billing model only accelerates churn. Subscription operations also carry significant overhead — renewals, refunds, tier permissions, etc. Not a fit for solo operation.

Instead: **Free + a single Lifetime IAP**. As supplementary revenue, free users can watch an AdMob rewarded ad to unlock specific features for one month. Users who've watched enough ads to feel the sunk cost see a sunk-cost variant of the paywall (`paywall.title.sunkCost`) once.

Cost accepted: lifetime revenue ceiling (LTV cap). Accepted. A pricing structure that matches the tool's nature is healthier long-term.

---

## 4. Where it got stuck, and how it got unstuck

### 4-1. Swift 6.3.2 compiler crash (Release builds only)

Running Release builds to make an Archive, `swift-frontend` died with SIGSEGV. Debug builds were fine.

Build log showed the crash at the `EarlyPerfInliner` SIL pass. The location: `DragSelectableScrollView.Coordinator` — an `NSObject` subclass adopting `UIGestureRecognizerDelegate`. The compiler's auto-synthesized `deinit` was crashing during optimization.

Diagnosis: an unreported Swift 6.3.2 compiler bug. Reporting it to Apple meant waiting for the next Xcode patch. Without Release builds, App Store release was blocked.

Workaround: I added an explicit `deinit`.

```swift
deinit {
    // Workaround for Swift 6.3.2 compiler bug (EarlyPerfInliner SIGSEGV).
    // CADisplayLink isn't auto-released on dealloc, so explicit invalidate is needed anyway.
    autoScrollTimer?.invalidate()
}
```

Bypassing the compiler's auto-synthesis path. Release builds came back. I left a clear comment so this code can be removed once the bug is fixed in a future Xcode.

**Takeaway**: Solo operation means you work around the compiler's bugs yourself too. That's the shadow side of zero dependencies. Accept it.

### 4-2. First App Store rejection — unresponsive purchase button

I expected the first review to pass. It was rejected. Guideline 2.1(b) — "purchase button was unresponsive to tap" on iPhone 17 Pro Max, iOS 26.4.2.

I went back to the code. `PaywallViewModel.purchaseLifetime()`:

```swift
func purchaseLifetime() async {
    guard let product = lifetimeProduct else { return }   // ← silent return
    isPurchasing = true
    ...
}
```

If the product (`lifetimeProduct`) is `nil`, tapping the button silently returns. No error. That's exactly what the reviewer saw as "unresponsive."

But why was the product `nil`? Because `Product.products(for:)` returned an empty array. The reason: **the Paid Applications Agreement wasn't active**. The classic first-IAP-submission rejection pattern.

I had an advisor model verify the diagnosis before committing. Two problems compounded — a code bug and an inactive contract. Fixing only one wouldn't have passed the next review either.

Two parallel tracks:

**Track A — Code fix (available immediately)**
- Remove the silent return in `purchaseLifetime()` → reload, then show an explicit error on failure
- Add a "Retry" UI in `PaywallView` for the product-load-failed state
- Add two localization keys (en/ko)

**Track B — Activate the agreement (external dependency)**
- Korean e-commerce business registration (정부24, 3–5 business days)
- Korean tax form (business reg number + e-commerce reg number)
- W-8BEN tax form
- Bank account registration
- Once all are active, the Paid Apps Agreement activates automatically

While Track B's external dependency ran for a few days, I finished Track A and verified both Debug and Release builds. Once the agreement was confirmed active, I resubmitted as `1.1.0 (1001)`. The reviewer notes spelled out the previous rejection's cause and what was fixed.

**Takeaway**: A one-line rejection reason like "unresponsive purchase button" can hide **a code bug and a business-process gap, both broken at the same time**. The essence of debugging is to start from the symptom the reviewer saw and follow the causal chain all the way down.

---

## 5. AI collaboration — what it actually looks like

What does it concretely mean to integrate Claude Code into a daily workflow?

**Daily workflow**:
1. Keep a Claude Code session open while working
2. When starting a new feature, *I* decide what the screen needs to do and how the data flows (I don't ask Claude to decide that)
3. Delegate boilerplate and component generation to Claude in that direction
4. Review the generated code. If it violates any of the five architecture principles, revert
5. Debugging, refactoring, and adding localization keys are largely handled by Claude
6. When a trade-off decision is needed, *I* stop and think separately. If necessary, run the question by an advisor model

**Role split**:

| What AI is good at | What only a human should do |
|---|---|
| Boilerplate and UI component generation | Per-screen design and UX scenarios |
| Repetitive refactoring and renaming | Architecture decisions (the five principles) |
| Compiler-error debugging assistance | Trade-off judgment |
| Adding localization keys | Code review and merge decisions |
| Drafting App Store metadata copy in English | App Store review handling, agreements, tax setup |

**Speed-up**: Roughly 3–5× faster shipping cycle compared to building the same scope without AI assistance. A subjective comparison based on building similar-sized modules at KineMaster.

**Limit**: AI doesn't make design decisions for you. Judgments like "why not TCA" remain your responsibility. AI just builds the code that follows those judgments, fast. If you start delegating decisions to AI, the codebase loses coherence.

That's the core point. **AI accelerates what comes *after* the decision; it doesn't make the decision itself.** What makes seniority valuable is making good decisions, not typing code fast. AI collaboration doesn't lower the value of seniority — it makes that value more visible.

---

## 6. Retrospective — what I'd do differently

**What I'd do differently**:
- **Complete the Korean e-commerce business registration before submitting**. Getting only the business registration and deferring the e-commerce registration was the external cause of the first rejection. Would have saved a few days on the resubmission cycle.
- **Classify location permission correctly when writing AppStore_PrivacyNutritionLabel.md**. I initially marked location as "No," and it was caught at the actual submission stage.

**What I'd keep doing**:
- **Zero dependencies + SSOT**. At 70 files, the codebase is consistent everywhere you look. A year from now, when adding a new feature, that consistency will determine the cost.
- **Not using TCA**. The familiarity made it tempting, but holding back to fit the project's scale was correct.
- **Writing CLAUDE.md in the first week**. The single document that anchored AI collaboration's consistency. I didn't have to re-explain "the principles are these" to AI every time.

**What I don't know yet**:
- Real user response (just after release). Whether classification accuracy is sufficient on real data, whether the sunk-cost paywall's conversion matches the hypothesis — those have to be validated with post-release data.

---

## Closing

Three months. Solo. Zero external dependencies. 13 years of iOS experience + a Claude Code collaboration workflow.

One proof of what this combination can do. More importantly, also a proof of what *not* to do — don't delegate decisions to AI, fit tools to project scale, be willing to question market defaults.

None of these decisions could have been made without 13 years of iOS experience. AI only made executing on them faster.

---

- 📱 [App Store: ClassiPic](https://apps.apple.com/app/id6769269007)
- 💻 [GitHub: sylar-Jeon/ClassiPic](https://github.com/sylar-Jeon/classipic-portfolio)
- 🌱 [Prototype: AlbumManager](https://github.com/sylar-Jeon/AlbumManager)
- 📮 sylar32a+dev@gmail.com

— **JaeMin Jeon** (전재민), 13+ years iOS, ex-KineMaster Lead
