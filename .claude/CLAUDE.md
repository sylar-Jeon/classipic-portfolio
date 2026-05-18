# ClassiPic — CLAUDE.md

AI 코딩 어시스턴트를 위한 프로젝트 지침서.
이 파일을 항상 먼저 읽고, 모든 결정에 아래 규칙을 적용하라.

---

## 계획(plans) 문서들
- 현재 고려해야할 plans 문서들은 root에 존재한다. 
- 그외 이전에 고려했던(사용했던) plans 문서들은 plans_bak에 존재한다. (필요시 참고할 것)

## git, 파일 변경 관련
- 사용자는 항상 원본 프로젝트 소스 기준으로 말하고 작업지시 한다. 
- 작업은 한 step씩 끊어서 한다. 
- 작업 시작시 원본 프로젝트의 소스를 worktree에 반영후 작업한다. 
- git관련은 사용자가 직접 검증후 merge, commit등 직접 관리한다.
- 작업이 완료되면 worktree의 변경 내용을 테스트 할 수 있게 반영(변경내용 copy)후 적당한 commit message를 제시한다.


## 프로젝트 개요

**ClassiPic** — iOS 사진 정리 앱.
AI가 사진을 추억(Moments)과 정보(Records)로 자동 분류해주는 iOS 기본 사진앱 보조 앱.

```
Bundle ID:  com.sylar.classipic
Platform:   iOS 17.6+  (iPhone only, Portrait only)
Stack:      SwiftUI + @Observable
Deps:       없음 (zero external dependencies)
```

---

## 기본 원칙

### 1. 어떤 결정이 필요할때 알아서 하지 않고 구체화 하는 질문을 사용자에게 질문한다. 

### 2. iOS Photos Library = Single Source of Truth
- ClassiPic 내부 DB 없음. PHAsset / PHAssetCollection이 유일한 진실.
- 앱 실행 및 foreground 복귀 시마다 PHAssetCollection fresh read.
- 삭제 = iOS "최근 삭제된 항목"으로 이동 (30일 보관). 영구 삭제 아님.
- 사용자 설정 저장은 UserDefault를 이용한다. 

### 3. UI/UX 구현시 iOS 기본 Photo App의 style을 그대로 따른다. 
- 전체 UI/UX 구현시 iOS 기본 Photo App의 style을 그대로 따른다. 
- 동작이 같은 경우 버튼 색상, 크기등 가능한 최대한 같게 구현한다.

### 4. Dependency 최소화
- 외부 라이브러리 최소화 한다. 
- 외부 라이브러리 추가가 필요시 항상 사용자에게 먼저 허락 맞기
- 아이콘: SF Symbol 우선. 커스텀 이미지 최소화.
- 애니메이션: SF Symbol를 이용한다. 

### 5. 아키텍처 패턴 — MVVM + @Observable
```
View → ViewModel(@Observable) → PhotoLibraryService(singleton)
```
- **View**: 렌더링만. 비즈니스 로직 없음.
- **ViewModel**: `@Observable final class`. 화면당 1개. `@State private var vm = XxxViewModel()`.
- **Service**: `PhotoLibraryService.shared` 싱글톤. PHPhotoLibrary 직접 통신만 담당.

### 6. iPhone only / Portrait only
- iPad 레이아웃 별도 구현 금지. iPad는 phone UI 확대로 표시.


## 구현 진행 방식

### 코드 생성 시 지켜야 할 것
1. 함수/변수명은 영어. 주석은 한국어 허용.
2. 파일 상단에 `// MARK: -` 섹션 구분 사용.
3. 비동기 함수는 `async throws` 패턴 사용. `try?`로 에러 무시 시 반드시 주석 이유 명시.
4. `force unwrap (!)` 사용 금지. `guard let` 또는 `if let` 사용.
   - 단, PhotoKit placeholder에서 `firstObject!` 는 허용 (생성 직후 fetch라 nil 불가).
5. `print()` 디버그 코드는 `#if DEBUG` 블록 안에만.
6. ViewModel에서 View를 import하거나 참조 금지.
7. **Hit-test 경계 명시**: thumbnail/cell처럼 `.onTapGesture`를 갖는 컴포넌트에서 `Image(...).scaledToFill()` 또는 `.scaledToFit()`을 쓸 때는 outer container에 반드시 `.clipShape(...)` 또는 `.contentShape(Rectangle())`을 명시한다. `.clipped()`는 그리기만 자르고 hit-test 영역은 자르지 않아 인접 셀의 탭을 가로채는 버그의 원인이 된다. `.clipped()`는 시각적 마무리 용도에만 사용.


## 참고 문서 (프로젝트 루트에 위치)
- `docs/ClassiPic_Phase1_Plan_v1.0.md` — 기획 & 설계 확정서 (화면 스펙, UX 시나리오)


## 빌드 확인
코드 생성 후 반드시 `xcodebuild` 빌드 시도. 에러 발생 시 스스로 수정 후 재빌드.
빌드 성공을 확인한 후 사용자에게 결과 보고.


## 동작 확인
사용자가 '...후 검증해줘' 라고 요청하면 
빌드 성공후 화면에 떠 있는 simulator(iPhone 16e(identifier: 56EFFBD6-1EE0-4CB7-A80B-1003643B03D1))를 사용하여 직접 동작 확인하여 검증한다. 

