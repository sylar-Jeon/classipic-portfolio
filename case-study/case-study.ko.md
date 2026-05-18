# ClassiPic — 1인 iOS 앱을 기획부터 App Store 출시까지 (Case Study)

[English](case-study.en.md) | 한국어 · [← README](../README.md)

> 13년차 iOS 시니어가 Claude Code 등 AI 도구와 협업해 3개월 만에 App Store에 출시한 사진 정리 앱. 이 글은 그 과정에서 내린 결정과 트레이드오프, 막혔던 지점과 풀어낸 방식, 그리고 다시 한다면 달리할 부분을 정리한 기록이다.

- **결과물**: [App Store](https://apps.apple.com/app/id6769269007) · [GitHub](https://github.com/sylar-Jeon/classipic-portfolio) · [Privacy Policy](https://sylar-jeon.github.io/classipic-privacy/)
- **프로토타입**: [AlbumManager](https://github.com/sylar-Jeon/AlbumManager) — 이 프로젝트의 출발점
- **기간**: 약 3개월 (2026년 2월 ~ 5월)
- **역할**: 1인 — 기획·설계·구현·App Store 심사·계약·세금까지 단독

---

## 1. 왜 사진 정리 앱이었나

매일 보는 사진첩이 가장 짜증나는 사용자 인터페이스였다. iOS 기본 사진 앱은 좋지만, 영수증·명함·스크린샷이 추억 사진과 뒤섞여 있는 구조 자체를 손볼 도구가 없었다.

몇 해 전 이 문제를 풀어보려고 만든 게 **AlbumManager**라는 데모 프로젝트였다. 포트폴리오용 단순 분류 기능 수준이었고, 출시 가능한 형태는 아니었다. AI 코딩 도구가 빠르게 성숙해진 시점에서 이걸 끄집어내, "출시 가능한 형태로 끌어올리는 데 정말 얼마나 걸리나"를 직접 확인해 보고 싶었다.

세 가지 목적이 있었다.

1. **본인 도구로 쓰고 싶음** — 가장 강력한 동기
2. **1인 출시 사이클의 전 과정을 직접 경험** — 13년 동안 iOS 개발만 했지, App Store 계약·세금·심사 대응을 단독으로 끌고 가본 적은 없었다
3. **AI 협업의 한계를 검증** — Claude Code 등의 도구를 일상 워크플로에 통합한다는 게 실제로 어디까지 가능한지

---

## 2. 설계 단계 — "이 앱은 무엇이 되어야 하는가"

가장 먼저 결정한 건 **포지셔닝**이었다. iOS 기본 사진 앱을 대체할 것인가, 보조할 것인가.

대체 시도는 99% 실패한다. 사용자가 이미 사진 앱에 익숙하고, iCloud 동기화·공유 앨범·메모리 기능 같은 거대한 기능 묶음을 별도 앱이 따라잡기는 불가능하다. 그래서 **"보조 앱(complementary app)"** 포지션을 잡았다.

이 결정 하나에서 코드베이스의 5가지 원칙이 자동으로 따라 나왔다.

| 원칙 | 포지셔닝과의 연결 |
|---|---|
| iOS Photos Library = Single Source of Truth | 보조 앱이니까 사진 앱과 같은 데이터를 봐야 함 |
| 100% on-device | 사용자가 사진을 다른 앱에 넘긴다는 거부감 없애기 |
| iOS Photos 앱 UI/UX 패리티 | 보조 앱이니까 시스템 앱과 자연스럽게 섞여야 함 |
| Zero external dependencies | 1인 운영의 현실 — 의존성 업데이트 부담 최소화 |
| SwiftUI + @Observable MVVM | 같은 이유. 의존성 0 원칙과도 맞물림 |

이 5가지는 [`CLAUDE.md`](https://github.com/sylar-Jeon/classipic-portfolio/blob/main/.claude/CLAUDE.md)에 명문화해, 이후 모든 결정·코드 리뷰의 기준선이 됐다.

---

## 3. 트레이드오프 — 네 가지 핵심 결정

### Decision 1 — 내부 DB 없이 iOS Photos Library를 SSOT로

Core Data나 SwiftData로 메타데이터를 캐싱하는 게 직관적이다. PhotoKit fetch가 큰 라이브러리에서 살짝 느리다는 알려진 문제도 있고.

그런데 그 길로 가면 **양쪽에 같은 정보를 따로 저장**하게 된다. 동기화 버그가 끝없이 생긴다. 기존에 비슷한 패턴(타임라인 데이터를 in-memory와 디스크에 따로 둔 구조)이 만든 버그를 수년간 봐 왔다.

그래서 **내부 DB 0개**로 결정했다. PHAsset·PHAssetCollection이 유일한 진실. 백업·복원·iCloud 동기화는 iOS가 알아서 한다. ClassiPic이 만든 앨범도 iOS 사진 앱에서 그대로 보인다. 이건 보조 앱 포지션의 자연스러운 귀결이기도 하다.

감수한 비용: 매 fetch가 PhotoKit을 친다. 큰 라이브러리에서 첫 로드가 0.5~1초 느려진다. 분류 결과만 따로 `ScanRegistry`라는 가벼운 캐시 레이어(in-memory + UserDefaults)에 두어 보완했다.

**일반화한 판단**: "데이터를 우리가 만든 게 아니면, 우리가 소유하지 말자."

### Decision 2 — @Observable로 통일, TCA·Combine 미사용

이게 가장 흥미로운 결정이었다. 직전 회사에서 2021년에 TCA(The Composable Architecture)를 국내 상용 앱 중 가장 이른 시점에 도입했었다. 6~8명 팀에서 31만 라인 코드베이스의 60% 이상에 TCA를 적용해 운영했다.

그래서 ClassiPic에 TCA를 쓸까 진지하게 고민했다. 결론은 **안 쓴다**였다.

TCA의 진짜 가치는 **여러 명이 같은 도메인 reducer를 동시 수정**할 때 나온다. store/effect 모델이 머지 충돌의 표면적을 압축하고, action enum을 통해 사이드이펙트 위치를 강제한다. 팀 규모와 도메인 복잡도가 일정 임계치를 넘으면 이 구조가 보일러플레이트 비용을 능가한다.

ClassiPic은 1인 프로젝트 + 70 파일 규모다. 임계치를 한참 밑돈다. `@Observable`은 iOS 17부터 충분히 안정적이고, 의존성 0 원칙과도 맞물린다 (TCA는 SPM 의존성 +1, 빌드 시간 ↑).

Combine도 같은 이유로 의도적으로 배제했다. Swift Concurrency(async/await)와 `@Observable`의 자동 트래킹으로 reactive 흐름은 다 처리된다.

**일반화한 판단**: "도구는 팀 규모·도메인 복잡도에 종속된다. 같은 사람이 다른 프로젝트에서 다른 선택을 하는 게 일관성 없는 게 아니라, 그게 시니어가 하는 일이다."

이 결정은 인터뷰에서 다음과 같은 질문에 대한 답이 된다.
> "TCA 도입 경험이 있다고 했는데, 본인이 새 프로젝트를 만든다면 또 TCA를 쓸 건가요?"

### Decision 3 — 외부 의존성 0

CocoaPods·SPM 의존성 0개. 아이콘과 애니메이션도 SF Symbol만 쓴다.

극단적인 결정이지만 1인 운영의 현실이 강제했다. 의존성 5개만 있어도 매주 업데이트 알림·보안 패치·breaking change 대응에 적지 않은 시간이 든다. 1인 운영자에게 그 시간은 곧 개발 정체다.

그리고 iOS native API가 충분히 풍부했다. 정말 필요한 외부 라이브러리가 없었다.

감수한 비용: 일부 컴포넌트는 직접 만들어야 했다. 대표적으로 `DragSelectableScrollView` — SwiftUI의 ScrollView 안에서 UIKit drag-select gesture를 구현해, 사진 다중 선택을 iOS 기본 사진 앱과 동일한 인터랙션으로 작동하게 만든 컴포넌트다. 직접 만든 만큼 디버깅도 직접 해야 한다.

그 디버깅 중 Swift 6.3.2 컴파일러의 SIL pass 버그를 만났다. 이 얘기는 다음 섹션에서.

예외 둘: **Google AdMob**과 **UserMessagingPlatform**. 광고 수익화 요구사항상 precompiled framework로 들어왔다. 트레이드오프상 받아들였다.

### Decision 4 — 수익화: 구독 없음, Lifetime 단일 IAP

사진 정리 앱 시장의 표준은 월간/연간 구독이다. 시장 표준을 따르지 않기로 했다.

이유는 도구의 본질이다. "사진은 한 번 정리하면 끝"인 도구를 매월 결제 모델로 묶으면 사용자 이탈만 빨라진다. 그리고 구독 운영은 갱신·환불·등급별 권한 관리 등 부가 비용이 상당하다. 1인 운영에 적합하지 않다.

대신 **Free + Lifetime 단일 IAP** 구조로 갔다. 보조 수익으로 Free 사용자에게 AdMob rewarded 광고 시청 시 일부 기능을 한 달간 한시 해금하게 했다. 광고를 충분히 본 사용자에게는 매몰비용을 활용한 paywall 변형(`paywall.title.sunkCost`)이 한 번 노출된다.

감수한 비용: 평생 수익 상한(LTV ceiling). 받아들였다. 도구의 본질에 맞는 가격 구조가 장기적으로 더 건강하다고 판단했다.

---

## 4. 막힌 곳과 풀어낸 방식

### 4-1. Swift 6.3.2 컴파일러 크래시 (Release 빌드 한정)

Archive를 만들려고 Release 빌드를 돌리니 `swift-frontend`가 SIGSEGV로 죽었다. Debug 빌드는 멀쩡했다.

빌드 로그를 보니 `EarlyPerfInliner` SIL pass에서 죽고 있었다. 위치는 `DragSelectableScrollView.Coordinator` 클래스 — UIGestureRecognizerDelegate를 채택한 NSObject 서브클래스다. 컴파일러가 자동 합성한 `deinit`을 최적화하는 단계에서 SIGSEGV가 났다.

진단: Swift 6.3.2의 알려지지 않은 컴파일러 버그. Apple에 보고해도 다음 Xcode 패치를 기다려야 한다. Release 빌드가 안 되면 App Store 출시가 막힌다.

회피: 명시적 `deinit`을 추가했다.

```swift
deinit {
    // Swift 6.3.2 컴파일러 버그(EarlyPerfInliner SIGSEGV) 회피용 명시적 deinit.
    // CADisplayLink는 dealloc 시 자동 해제되지 않으므로 명시 invalidate 필요.
    autoScrollTimer?.invalidate()
}
```

컴파일러의 자동 합성 경로를 안 거치게 하는 우회. Release 빌드가 정상으로 돌아왔다. 주석에 이유를 명확히 남겨, 다음 Xcode 버전에서 버그가 고쳐지면 이 코드를 제거할 수 있게 했다.

**얻은 교훈**: 1인 운영은 컴파일러 버그도 본인이 회피해야 한다. 그게 의존성 0의 그림자다. 받아들이는 게 맞다.

### 4-2. 첫 App Store 심사 거부 — IAP 결제 버튼 무반응

심사가 통과될 줄 알았는데 거부됐다. Guideline 2.1(b) — "결제 버튼이 탭에 반응하지 않음(unresponsive to tap)" on iPhone 17 Pro Max, iOS 26.4.2.

코드를 다시 봤다. `PaywallViewModel.purchaseLifetime()`:

```swift
func purchaseLifetime() async {
    guard let product = lifetimeProduct else { return }   // ← 무음 리턴
    isPurchasing = true
    ...
}
```

상품(`lifetimeProduct`)이 `nil`이면 버튼을 눌러도 **에러도 없이** 그냥 return한다. 심사자가 본 "무반응"이 정확히 이 상태다.

그런데 왜 상품이 nil이었을까? `Product.products(for:)`가 빈 배열을 반환한 거다. 이유는: **유료 앱 계약(Paid Applications Agreement)이 활성화되지 않았기 때문**. 첫 IAP 제출의 전형적 거부 패턴이다.

이 진단은 advisor 모델에게 한 번 검증받고 확신을 굳혔다. 코드 버그와 계약 미활성 — **두 가지가 겹친 문제**였다. 둘 중 하나만 고치면 다음 심사도 떨어진다.

대응을 두 트랙으로 나눴다.

**트랙 A — 코드 수정 (즉시 가능)**
- `purchaseLifetime()`의 무음 return 제거 → 재로드 시도 후 실패하면 사용자에게 명시적 에러 메시지
- `PaywallView`에 상품 로드 실패 시 "다시 시도" UI 추가
- 로컬라이제이션 키 2개 (en/ko) 추가

**트랙 B — 계약 활성화 (외부 의존)**
- 통신판매업 신고 (정부24, 3~5 영업일 소요)
- 대한민국 세금 양식 작성 (사업자번호 + 통신판매업 번호)
- 미국 W-8BEN 양식
- 은행 계좌 등록
- 위 모두 활성화되면 유료 앱 계약 자동 활성화

트랙 B가 외부 의존이라 며칠 기다리는 동안, 트랙 A 코드 수정을 완료해 Debug/Release 둘 다 빌드 검증을 끝냈다. 계약 활성화 확인 후 빌드 번호 `1.1.0 (1001)`로 재제출. 심사 메모에 "이전 거부 원인이 무엇이었고, 어떻게 고쳤는지"를 명시했다.

**얻은 교훈**: "결제 버튼이 무반응"이라는 한 줄짜리 거부 사유 뒤에 **코드와 비즈니스 절차 두 가지가 동시에 깨진 상태**가 숨어 있을 수 있다. 디버깅의 본질은 "심사자가 본 증상"으로부터 시작해 원인 사슬을 끝까지 따라가는 거다.

---

## 5. AI 협업 — 실제로 어떻게 일했나

Claude Code를 일상 워크플로에 통합한다는 게 무엇인지 구체적으로.

**일상 워크플로**:
1. Claude Code 세션을 열어두고 작업한다
2. 새 기능을 시작할 때 먼저 "이 화면은 무엇을 해야 하고, 어떤 데이터를 어떤 흐름으로 다룬다"를 사람이 정한다 (Claude에게 정하라고 안 한다)
3. 그 방향대로 보일러플레이트·컴포넌트 생성을 Claude에게 위임한다
4. 생성된 코드를 사람이 리뷰한다. 아키텍처 원칙(5가지)을 위반하면 되돌린다
5. 디버깅·리팩토링·로컬라이제이션 키 추가는 Claude가 대부분 처리한다
6. 트레이드오프 결정이 필요하면 사람이 멈추고 따로 사고한다. 필요하면 advisor 모델에 검증받는다

**역할 분담 표**:

| AI가 잘하는 것 | 사람만이 해야 하는 것 |
|---|---|
| 보일러플레이트·UI 컴포넌트 생성 | 화면 단위 기획·UX 시나리오 |
| 반복 리팩토링·rename | 아키텍처 결정 (5가지 원칙) |
| 컴파일 에러 디버깅 보조 | 트레이드오프 판단 |
| 로컬라이제이션 키 추가 | 코드 리뷰·머지 결정 |
| App Store 메타데이터 영문 카피 초안 | App Store 심사 대응·계약·세금 처리 |

**가속 효과**: 동일 규모 앱을 기존 방식으로 만들 때 대비 약 3~5배 빠른 출시 사이클. KineMaster 시절 비슷한 규모 모듈을 만들었던 경험으로 비교한 체감치다.

**한계**: AI는 디자인 결정을 대신 내려주지 않는다. "왜 TCA를 안 쓰는가" 같은 판단은 본인 책임이다. AI는 그 판단을 따라 빠르게 코드를 만들어 줄 뿐이다. 결정을 AI에게 위임하기 시작하면 코드베이스의 일관성이 흩어진다.

이게 핵심이다. **AI는 "결정 이후"를 가속할 뿐, "결정 자체"를 대신해 주지 않는다.** 시니어가 시니어인 이유는 결정을 잘 내리는 것이지, 코드를 빨리 치는 게 아니다. 그래서 AI 협업이 시니어 가치를 떨어뜨리는 게 아니라 오히려 더 명확하게 드러낸다.

---

## 6. 회고 — 다시 한다면

**다시 한다면 달리할 것**:
- **통신판매업 신고를 출시 전에 미리 받았을 것**. 사업자등록만 받고 통신판매업 신고를 미룬 게 첫 심사 거부의 외부 원인이었다. 재제출 사이클을 며칠 단축할 수 있었다.
- **AppStore_PrivacyNutritionLabel.md 작성 시점에 위치 권한을 정확히 분류했을 것**. 처음에 위치를 "No"로 잘못 분류해 둔 것이, 본 심사 단계에서 다시 잡혔다.

**잘했다고 보는 것**:
- **의존성 0과 SSOT 결정**. 70 파일 규모에서 어디를 봐도 일관된 구조다. 1년 후 새 기능 추가할 때 이 일관성이 비용을 결정할 거다.
- **TCA 안 쓴 결정**. 익숙한 도구라 쓰고 싶은 충동이 컸지만, 프로젝트 규모에 맞춰 자제한 게 옳았다.
- **CLAUDE.md를 첫 주에 작성한 것**. AI 협업의 일관성을 잡아준 결정적 문서. AI에게 매번 "원칙은 이거야"를 다시 설명할 필요가 없었다.

**아직 모르는 것**:
- 실제 사용자 반응 (출시 직후). 분류 정확도가 실 데이터에서 충분한지, 매몰비용 paywall의 전환율이 가정과 맞는지 등은 출시 후 데이터로 검증해야 한다.

---

## 마치며

3개월. 1인. 외부 의존성 0. 13년의 iOS 경험 + Claude Code 협업 워크플로.

이 조합이 무엇을 할 수 있는지의 한 가지 증명이다. 더 중요한 건 무엇을 하지 말아야 하는지의 증명이기도 하다는 거다 — AI에게 결정을 위임하지 말 것, 도구를 프로젝트 규모에 맞출 것, 시장 표준을 의심할 수 있을 것.

이런 결정들은 13년의 iOS 경험이 없었으면 내릴 수 없는 것이었다. AI는 그 결정의 실행을 빠르게 했을 뿐이다.

---

- 📱 [App Store: ClassiPic](https://apps.apple.com/app/id6769269007)
- 💻 [GitHub: sylar-Jeon/ClassiPic](https://github.com/sylar-Jeon/classipic-portfolio)
- 🌱 [Prototype: AlbumManager](https://github.com/sylar-Jeon/AlbumManager)
- 📮 sylar32a+dev@gmail.com

— **전재민** (JaeMin Jeon), 13+ years iOS, ex-KineMaster Lead
