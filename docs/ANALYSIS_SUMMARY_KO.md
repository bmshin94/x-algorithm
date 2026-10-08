# X For You Feed Algorithm — 전수조사 분석 정리 (한국어)

> 이 문서는 `x-algorithm` 리포지터리를 전수조사(2,080개 파일 / 26개 서브시스템)하여
> **무엇을 하는 코드인지, 언제 쓰는지, 어떻게 활용·수익화할 수 있는지**를 정리한 분석 보고서입니다.

## 📌 리포지터리 정보

| 항목 | 내용 |
|---|---|
| **이 리포 (포크)** | https://github.com/bmshin94/x-algorithm |
| **원본 (업스트림)** | https://github.com/xai-org/x-algorithm |
| **라이선스** | Apache License 2.0 (상업적 이용·수정·재배포 허용, 출처 표기 조건) |
| **투명성 도구** | https://x.com/i/under_the_hood |
| **규모** | 2,080개 파일 / 26개 서브시스템 |
| **언어 구성** | Scala 568 · Rust 451 · Python 448 · Java 313 · 기타(.df 53, .bazel 39, .strato 29, .bot 20, .thrift 12, .proto 9) |
| **분석 일자** | 2026-10-08 |

---

## 📑 목차

1. [무엇을 하는 코드인가](#1-무엇을-하는-코드인가)
2. [요청 1건의 전체 흐름](#2-요청-1건의-전체-흐름-10단계)
3. [실제 랭킹 가중치 (프로덕션 기본값)](#3-실제-랭킹-가중치-프로덕션-기본값)
4. [점수 보정 3단계](#4-점수-보정-3단계)
5. [라벨링 경로 — 가시성 라벨은 어디서 오나](#5-라벨링-경로--가시성-라벨은-어디서-오나)
6. [폴더별 전수조사 결과 (26개)](#6-폴더별-전수조사-결과-26개)
7. [핵심 설계 철학 5가지](#7-핵심-설계-철학-5가지)
8. [실험과 설정 — 알고리즘 변경 추적](#8-실험과-설정--알고리즘-변경-추적)
9. [설치 및 사용법](#9-설치-및-사용법)
10. [플러그인·스킬·MCP 여부](#10-플러그인스킬mcp-여부)
11. [API 토큰 필요 여부](#11-api-토큰-필요-여부)
12. [AI 에이전트 구축에 주는 도움](#12-ai-에이전트-구축에-주는-도움)
13. [React / PHP 구현 가능 범위](#13-react--php-구현-가능-범위)
14. [유튜브 강의 제작 가능성](#14-유튜브-강의-제작-가능성)
15. [수익화 아이디어 상세](#15-수익화-아이디어-상세)
16. [첫 90일 실행 로드맵](#16-첫-90일-실행-로드맵)
17. [한계 및 비공개 영역](#17-한계-및-비공개-영역)

---

## 1. 무엇을 하는 코드인가

**X(옛 트위터)의 "For You(추천) 피드"를 만드는 실제 프로덕션 백엔드 소스코드 공개판**입니다.

- 팔로우한 계정의 글(**인네트워크**)과 ML 리트리벌로 발굴한 글(**아웃네트워크**)을 합치고,
- 여러 입력으로 필터링하고,
- **트랜스포머 모델**로 순위를 매깁니다.

핵심은 **두 가지 판단이 완전히 분리**되어 있다는 점입니다.

```
"순서를 정하는 것(랭킹)"  ≠  "보여줄지 말지(가시성)"
      ↑ phoenix                   ↑ visibility-filtering
   완전히 다른 서비스, 다른 입력, 다른 규칙
```

---

## 2. 요청 1건의 전체 흐름 (10단계)

```
[피드 요청]
   ↓
① 쿼리 하이드레이션  home-mixer/query_hydrators/
   내 최근 행동 시퀀스(모델의 핵심 입력), 팔로잉 목록, 차단/뮤트,
   뮤트 키워드, 이미 본 글 목록, 팔로우한 토픽 수집
   ↓
② 후보 수집 (병렬)   home-mixer/sources/
   ├─ 인네트워크:  thunder/      내가 팔로우한 계정의 최근 글(인메모리 보관)
   └─ 아웃네트워크: phoenix/     리트리벌 모델(벡터 유사도)
                   simclusters/ 커뮤니티 클러스터 유사도
   ↓
③ 후보 하이드레이션  home-mixer/candidate_hydrators/
   본문·미디어·작성자 정보·계정 라벨·인용글·언어·참여수 붙이기
   ↓
④ 사전 필터 (17개)   home-mixer/filters/
   중복 / 48시간 초과 / 내 글 / 차단·뮤트 / 뮤트 키워드 / 이미 본 글 …
   ↓
⑤ 스코어링           home-mixer/scorers/
   PhoenixScorer: 30여 개 행동별 확률 예측
   RankingScorer: 가중합 → 작성자 다양성 감쇠 → OON 할인 → 신규작성자 부스트
   VMRanker:      vm-ranker/ (DPP로 유사한 글 흩뿌리기)
   ↓
⑥ Top-K 선택        home-mixer/selectors/top_k_score_selector.rs
   ↓
⑦ 사후 필터          순서 확정 후 visibility-filtering/ 에 "보여도 되나?" 질의
   VFFilter / AncillaryVFFilter / DedupConversationFilter
   ↓
⑧ 블렌딩             광고 · 팔로우 추천 · 프롬프트 끼워넣기 (BlenderSelector)
   ↓
⑨ 응답 전송
   ↓
⑩ 사이드이펙트       서빙 기록 저장 · 캐시 갱신 · 광고/클라이언트 이벤트 로깅
```

### AI 모델이 하는 일 — "30개 질문"

Phoenix는 `"이 글 종합 점수 몇 점?"`을 묻지 **않습니다**. 대신 행동별로 따로 예측합니다.

| 그룹 | 예측 항목 |
|---|---|
| **참여(Engagement)** | 좋아요 · 답글 · 리트윗 · 인용 · 공유 · DM 공유 · 링크 복사 공유 |
| **클릭(Clicks)** | 포스트 · 프로필 · 링크 · 사진 확대 · 영상 열기 · 인용글 |
| **주목(Attention)** | 영상 품질 시청(VQV) · 체류 · 체류 시간 · 클릭 체류 시간 · 활동 초 |
| **작성자(Author)** | 작성자 팔로우 |
| **부정(Negative)** | 관심 없음 · 작성자 뮤트 · 작성자 차단 · 신고 · 체류 안 함 |

---

## 3. 실제 랭킹 가중치 (프로덕션 기본값)

출처: `home-mixer/params/param.rs:320-462`
계산: `home-mixer/scorers/ranking_scorer.rs`

```
최종 점수 = Σ (weight_i × P(action_i))
```

### 긍정 가중치

| 행동 | 가중치 | 파라미터 |
|---|---|---|
| 🔗 **링크 복사 공유** | **+20.0** | `share_via_copy_link_weight` |
| 💬 **답글** | **+5.0** | `reply_weight` |
| 🔁 인용(Quote) | +5.0 | `quote_weight` |
| 📩 DM 공유 | +5.0 | `share_via_dm_weight` |
| 👤 작성자 팔로우 | +4.0 | `follow_author_weight` |
| 📤 공유 | +2.0 | `share_weight` |
| 🔄 리트윗 | +1.0 | `retweet_weight` |
| ❤️ 좋아요 | +0.5 | `favorite_weight` |
| 🖱️ 클릭 | +0.4 | `click_weight` |
| 🔗 링크 열기 | +0.2 | `open_link_weight` |
| ▶️ 영상 열기 | +0.07 | `video_open_weight` |
| 🖼️ 사진 확대 | +0.05 | `photo_expand_weight` |
| ⏱️ 체류(dwell) | +0.05 | `dwell_weight` |
| 💬 인용글 클릭 | +0.05 | `quoted_click_weight` |
| 🆕 미탐색 포스트 | +0.02 | `post_unexplored_weight` |
| ⏳ 연속 체류 시간 | +0.004 | `cont_dwell_time_weight` |
| 👤 프로필 클릭 | 0.0 | `profile_click_weight` |
| 📺 VQV | 0.0 | `vqv_weight` |

### 부정 가중치

| 행동 | 가중치 | 파라미터 |
|---|---|---|
| 🚨 **신고** | **−234.0** | `report_weight` |
| 🔇 작성자 뮤트 | −58.8 | `mute_author_weight` |
| 🙅 관심 없음 | −43.2 | `not_interested_weight` |
| 🚫 작성자 차단 | −31.2 | `block_author_weight` |
| 😐 체류 안 함 | −0.02 | `not_dwelled_weight` |

### 특수 부스트

| 항목 | 값 | 설명 |
|---|---|---|
| **맞팔 답글 부스트** | **+15.0** | 맞팔로우 상대의 원본 글이면 `reply` 가중치에 가산 |
| 맞팔 체류 부스트 | 0.0 | 같은 A/B에서 테스트했으나 미출시 |

### ⚠️ 가장 흔한 오해 (xAI가 코드 주석에 직접 명시)

`param.rs:291-319` 주석 요지:

- 가중치는 **"예측 확률"에 곱하는 값**이며 **원시 참여 카운트에 곱하는 것이 아닙니다.**
- 따라서 **"신고 1건이 좋아요 468개를 상쇄한다"는 계산은 틀렸습니다.**
- 신고의 기저 확률이 좋아요보다 **1,000배 이상 낮기** 때문에, 랭킹에 영향을 주려면 가중치를 크게 줘야 합니다.
- **집단 신고 공격이 효과가 제한적인 이유:**
  1. 추천은 **개인화**되어 있어, 악성 계정의 신고는 주로 **그 악성 계정과 유사한 유저들의 추천**에만 영향을 줍니다.
  2. **홈 타임라인에 노출된 글에 대한 행동만** 집계됩니다. 단톡방 링크로 직접 들어가서 신고하면 **랭킹 영향 0**입니다.
  3. 유저가 특정 글을 자기 타임라인에 **재현 가능하게 띄울 방법이 없습니다.**

---

## 4. 점수 보정 3단계

출처: `home-mixer/scorers/ranking_scorer.rs:603-795`, `home-mixer/params/param.rs:233-290, 639-670`

### ① 작성자 다양성 (Author Diversity)

```
multiplier(n) = (1 − floor) × decay^n + floor
decay = 0.5,  floor = 0.25,  기본 활성화(true)
```

| 같은 작성자 n번째 글 | 배수 |
|---|---|
| 1번째 (n=0) | ×1.00 |
| 2번째 (n=1) | ×0.625 |
| 3번째 (n=2) | ×0.4375 |
| 4번째 (n=3) | ×0.34 |
| … 수렴 | ×0.25 (하한) |

→ **한 작성자가 피드를 도배하지 못합니다. 연속 폭풍 트윗은 비효율적입니다.**

### ② 아웃네트워크(OON) 할인

| 조건 | 배수 | 파라미터 |
|---|---|---|
| 안 팔로우한 계정의 글 | **×0.75** | `oon_weight_factor` |
| 토픽 기반으로 들어온 글 | **×0.5** | `topic_oon_weight_factor` |
| 팔로우한 계정의 **답글·리트윗**에도 적용 | ×0.75 | `enable_oon_rescore_for_in_network_replies_retweets` = true |

→ **신규 유입이 구조적으로 불리합니다. 그래서 "첫 반응 속도"가 결정적입니다.**

### ③ 신규 작성자 부스트 (Author Cold Start)

| 조건 | 값 | 파라미터 |
|---|---|---|
| 노출 횟수 | **1,000회 미만** | `cold_start_impression_threshold` |
| 팔로워 수 | **1,000명 이하** | `cold_start_follower_cap` |
| 글 작성 경과 | **172,800초 = 48시간 이내** | `cold_start_max_post_age_secs` |
| **승격 슬롯** | **15~16번째 위치** | `cold_start_slot_min` / `cold_start_slot_max` |

추가로 톰슨 샘플링(Thompson Sampling) 기반 탐색 옵션도 존재합니다
(`cold_start_beta_alpha0`, `cold_start_beta_beta0`, `cold_start_ts_top_k`).

→ **신규 계정의 골든타임은 정확히 "48시간 + 노출 1,000회"입니다.**

### ④ VMRanker (DPP 재배열)

`vm-ranker/` 서비스가 **결정점과정(Determinantal Point Process)** 으로 임베딩 기반 재배열을 수행합니다.
점수를 약간 희생해 **이웃한 글끼리의 유사도를 낮춥니다.** (비슷한 글이 연달아 나오는 것 방지)

---

## 5. 라벨링 경로 — 가시성 라벨은 어디서 오나

`visibility-filtering`은 요청 시점에 **미리 저장된 라벨을 조회만** 합니다.
라벨을 만드는 시스템은 요청 경로 밖에서 **상시** 동작합니다.

```
┌─ 1. 콘텐츠 이해 (상시 동작) ─────────────────────────────┐
│  [글·미디어]                    [계정]                      │
│  grox/          분류기           agatha/       좋아요 대비     │
│                 (스팸/성인/폭력)                 차단·신고 비율  │
│  media-model-   이미지·영상      bdsm/         비정상 행동     │
│    proxy/       모델                           (봇·어뷰징)     │
│  clip/          이미지·텍스트    user-cred-v2/ 팔로우 그래프    │
│                 임베딩                          PageRank       │
└──────────────────────────────────────────────────────────┘
                              ↓
┌─ 2. 라벨링 규칙 ────────────────────────────────────────┐
│  scarecrow/  이벤트 발생 즉시 반응. botmaker/ 를 룰 엔진으로   │
│              내장하고 botmaker-rules/scarecrow/ 에서 규칙 로드  │
│              규칙 형식: "이 이벤트에, 이 조건이면, 이 라벨"      │
│                                                             │
│  abuse-enforcement-service/  모델 점수 기반 조치             │
│              → 라벨 적용 / 챌린지 / 계정 정지                 │
│                                                             │
│  safety-label-user-agg/  글에 붙은 라벨을 계정 라벨로 집계      │
└──────────────────────────────────────────────────────────┘
                              ↓
┌─ 3. 저장 ──────────────────────────────────────────────┐
│          라벨을 저장소에 기록 → 요청 경로에서 읽음            │
└──────────────────────────────────────────────────────────┘
                              ↓
┌─ 4. 가시성 필터링  visibility-filtering/ ────────────────┐
│          표시 / 삭제 / 인터스티셜(가림막) 판정                │
└──────────────────────────────────────────────────────────┘
```

### 실제 botmaker 규칙 예시

`botmaker-rules/scarecrow/bot/nsfw_user_write_user_label.bot`

```yaml
id: '22358'
event: health_side_effect
condition: |
  healthSideEffectSubEvent == "nsfw_user" &&
  GetUser(userId) != NULL &&
  !IsTestUser(userId)
action: |
  {
    ApplyNsfwUserLabelOrCreateReport(userId, "NSFW_HIGH_PRECISION", Concat("By bot/", ToString(RULE_ID)));
    ApplyNsfwUserLabelOrCreateReport(userId, "NSFW_HIGH_RECALL", Concat("By bot/", ToString(RULE_ID)));
  }
expiry: '2031-07-31T16:37:35.583Z'
isActive: 'true'
```

→ 프로그래머가 아니어도 읽힐 만큼 단순한 DSL이며, 이 언어를 위해 **Java 313개 파일로 컴파일러와 런타임**을 구현했습니다.

### 가시성 판정 3가지 (`visibility-filtering/rules/registry.rs`)

| 판정 | 의미 |
|---|---|
| ✅ **Allow** | 그대로 표시 |
| ❌ **Drop** | 피드에서 제거 |
| ⚠️ **Interstitial** | "민감한 콘텐츠입니다 — 보기" 가림막 처리 |

**평가 규칙:**
- 규칙을 **등록 순서대로** 평가하며, **첫 `Drop`에서 즉시 종료**합니다.
- `SafetyLevel`은 `FilterAll`, `TimelineHome`, `TimelineHomeRecommendations` 세 종류입니다.
- **`TimelineHomeRecommendations` 전용 추가 규칙 세트**가 있고, 이 세트는 **`Drop`만 가능**합니다.

> 💡 **"쉐도우밴"처럼 느껴지는 현상의 실체**: 같은 글이 **팔로워에게는 보이지만 추천(OON)으로는 안 나갈 수 있습니다.** 추천 경로에만 적용되는 고(高)재현율 스팸 규칙 등이 따로 돌기 때문입니다.

### 사전 필터 17개 (평가 순서대로)

| 필터 | 제거 대상 |
|---|---|
| `DropDuplicatesFilter` | 여러 소스에서 중복 반환된 글 |
| `CoreDataHydrationFilter` | 본문·메타데이터 로드 실패 |
| `AgeFilter` | **48시간 초과** |
| `SelfTweetFilter` | 내가 쓴 글 |
| `OONRetweetReplyFilter` | 안 팔로우한 계정의 리트윗·답글, 부모글 없는 답글 |
| `OONNsfwSimclustersFilter` | SimClusters 글 중 작성자가 성인물 플래그 (미팔로우 시) |
| `RetweetDeduplicationFilter` | 같은 글의 반복 리트윗 |
| `IneligibleSubscriptionFilter` | 접근 불가한 구독자 전용 글 |
| `PreviouslySeenPostsFilter` | 이미 본 글 |
| `PreviouslySeenPostsBackupFilter` | 동일 (두 번째 노출 기록 기반) |
| `PreviouslyServedPostsFilter` | 세션 내 이미 서빙된 글 |
| `MutedKeywordFilter` | 뮤트 키워드 매칭 |
| `AuthorSocialgraphFilter` | 차단·뮤트한 계정 |
| `VideoFilter` | 영상 제외 요청 시 영상 글 |
| `TopicIdsFilter` | 요청 토픽 밖 / 제외 토픽 |
| `NewUserMinEngagementFilter` | 신규 계정에게 참여 임계값 미달 OON 글 |
| `InventoryHoldoutFilter` | 설정 비율만큼 (글·시청자 기준 결정론적 선택) |

**사후 필터:** `VFFilter` · `AncillaryVFFilter`(부모/인용/리트윗 원본이 Drop된 글) · `DedupConversationFilter`

**국가별 법률 대응 필터 실존 예:** `brazil_2026_election_filter.rs`
— 브라질 선거법에 따라 선거법원에 신고된 계정의 글을 For You에서 제거 (명시적 팔로우 시 예외)

---

## 6. 폴더별 전수조사 결과 (26개)

### 🅰️ 피드 조립

| 폴더 | 파일 수 | 역할 |
|---|---|---|
| `home-mixer/` | **228 Rust** | **심장부.** 파이프라인 전 단계 + 가중치 + 필터 32개 + 광고 블렌딩 |
| `candidate-pipeline/` | 11 Rust | home-mixer가 올라탄 프레임워크. 스테이지 타입 정의 및 병렬 실행 |

### 🅱️ 후보 소스

| 폴더 | 파일 수 | 역할 |
|---|---|---|
| `thunder/` | 24 Rust | 인네트워크. 발행 글을 인메모리 보관 → 팔로잉 글 반환 (Kafka) |
| `phoenix/` | **212 Py + 52 Rust** | **ML 본체.** JAX 트랜스포머 리트리벌+랭킹. 학습·서빙·합성데이터 포함 |
| `simclusters/` | **232 Scala** | "누가 무엇에 반응하나"로 계정·글 클러스터링 → 후보 추출 |
| `phoenix-rankall/` | 32 Rust | Phoenix가 조회하는 인덱스 유지·갱신 |
| `phoenix-rankall-strato/` | 17 strato | 글이 어느 인덱스에 속하는지 결정 (가시성 필터링 먼저 조회) |
| `vm-ranker/` | 10 Rust | DPP 재배열 — 점수 희생 ↔ 다양성 확보 |

### 🅲 콘텐츠 이해 (라벨 생성)

| 폴더 | 파일 수 | 역할 |
|---|---|---|
| `grox/` | **183 Python** | 글 발행 시 분류기 — 스팸·성인물·폭력 + 텍스트/이미지 수치표현 |
| `media-model-proxy/` | 57 Scala | 이미지·영상 모델 서빙 — 성인물·폭력·혐오 심볼·주제·기존 미디어 매칭 |
| `clip/` | 13 Py + 3 노트북 | 위 분류기들의 입력이 되는 이미지·텍스트 임베딩 모델 학습 |
| `agatha/` | 64 Scala | 오프라인 배치 — 좋아요 대비 차단/신고/스팸신고 비율로 계정 라벨링 |
| `bdsm/` | 33 Py + 4 Rust | 계정의 시간순 행동 시퀀스로 비정상(봇·어뷰징) 행위 탐지 |
| `user-cred-v2/` | 9 Scala | 팔로우 그래프 + 참여 엣지에 PageRank → 계정 신뢰도 점수 |
| `adult-content/` | 6 Python | 성인 미디어 분류기 학습·캘리브레이션 |
| `pnsfwmedia/` | 1 Python | CLIP 미디어 임베딩 + 계정 점수 결합 성인물 분류기 |

### 🅳 가시성 필터링

| 폴더 | 파일 수 | 역할 |
|---|---|---|
| `visibility-filtering/` | 54 Rust | 표시/삭제/인터스티셜 판정. 규칙은 `rules/registry.rs` |
| `botmaker/` | **313 Java + 147 Scala** | 규칙 엔진 — 전용 DSL + 컴파일러 + 런타임 |
| `botmaker-rules/` | 20 `.bot` + 53 `.df` | scarecrow가 로드하는 실제 규칙 (일부는 게이밍 방지로 비공개) |
| `scarecrow/` | 35 Scala | 이벤트 발생 즉시 라벨 규칙 적용 (botmaker 내장) |
| `abuse-enforcement-service/` | 28 Rust | 모델 점수 기반 조치 — 라벨/챌린지/정지 |
| `safety-label-user-agg/` | 3 strato | 글 라벨 → 계정 라벨 집계 |
| `visibility-filtering-client/` | 8 Rust | VF 호출 클라이언트 + 안전 라벨 타입 |
| `under-the-hood/` | 20 Scala + 9 strato | 계정별 라벨 투명성 리포트 생성 (일일 배치 + 서빙) |
| `takedowns/` | 4 Scala | 테이크다운 사유 목록 (글 사유 + 작성자 계정 레벨 사유 병합) |

---

## 7. 핵심 설계 철학 5가지

| # | 원칙 | 설명 | 왜 이렇게? |
|---|---|---|---|
| 1 | **멀티액션 예측** | 단일 "관련성" 점수가 아니라 30여 개 행동 확률을 각각 예측하고, 하나로 합치는 것은 **별도의 명시적 단계** | 가중치만 바꿔 피드 성격을 즉시 조정 (모델 재학습·재배포 불필요) |
| 2 | **후보 격리** | 트랜스포머 추론 시 후보들이 **서로를 attend 못 함**. 각자 "시청자 컨텍스트"만 봄 | 같은 글은 **항상 같은 점수** → 배치 구성 무관, 일관성 + 캐시 가능 |
| 3 | **해시 임베딩** | 어휘 사전 없이 **다중 해시 함수**로 임베딩 조회 | 방금 올라온 새 글도 **즉시** 표현 가능 (사전 등록 불필요) |
| 4 | **랭킹 ≠ 가시성** | 순서 결정과 표시 여부 결정을 **다른 서비스·다른 입력·다른 규칙**으로 완전 분리 | 안전 정책이 추천 품질에 섞이지 않음, 각각 독립 감사 가능 |
| 5 | **조립식 파이프라인** | `candidate-pipeline` 크레이트가 제공: 실행/모니터링과 비즈니스 로직 분리, 독립 스테이지 자동 병렬, graceful 에러 처리 | 새 소스/필터/스코어러 추가 용이, 단계별 on/off 실험 가능 |

### `candidate-pipeline` 6가지 스테이지 타입 (재사용 가치 높음)

| 타입 | 역할 |
|---|---|
| **Source** | 데이터 가져오기 |
| **Hydrator** | 컨텍스트 보강 |
| **Filter** | 걸러내기 |
| **Scorer** | 점수화 |
| **Selector** | 선택 (Top-K, 라우팅) |
| **SideEffect** | 응답 후 처리 (로깅, 기록) |

---

## 8. 실험과 설정 — 알고리즘 변경 추적

- 많은 튜닝 값이 코드에 하드코딩되지 않고 **설정 시스템에서 읽힙니다** (`xai_feature_switches`).
- **cron 스크립트가 리포의 기본값을 프로덕션 주력값으로 동기화**합니다.
  → **리포의 숫자는 "현재 프로덕션 주력값"입니다.**
- xAI는 **트래픽의 10% 이상을 차지하는 실험은 리포에 반영하는 것을 목표**로 한다고 명시했습니다.
- 가중치 섭동(`weight_perturbation_sigma`, `weight_perturbation_salt`)으로 탐색도 내장되어 있습니다.

### 실제 변경 사례 — 맞팔 답글 부스트 (`docs/BIDIRECTIONAL_BOOST_CHANGE.md`)

```
2026/07/10  A/B 시작 — 부스트 값 5, 10, 15, 20 중 랜덤 배정 (대부분 0 = 미적용)
            (맞팔 체류 부스트도 같은 A/B에 포함되었으나 미출시)
2026/07/13  결과 양호 → 다수 유저에게 20 적용, 나머지 값 계속 실험
2026/07/24  월드컵 기간 "안 팔로우한 계정의 월드컵 글이 덜 보인다"는
            피드백 반영 → 20 → 15 로 하향   ← 현재 코드값 (15.0)
```

> 💡 **알고리즘 변경을 "소문"이 아니라 "git diff"로 추적할 수 있게 된 것이 이 리포의 진짜 가치입니다.**

---

## 9. 설치 및 사용법

### ❗ 전제: 전체를 설치·실행하는 것은 불가능합니다

- 루트에 빌드 파일이 **없습니다** (Cargo.toml / package.json / pyproject.toml / Makefile 전부 없음).
- `xai_service_runner`, `xai_kafka`, `xai_feature_switches` 등 **xAI 내부 인프라 임포트만 남아 있고 실체가 없습니다.**
- 리포의 목적은 **"실행"이 아니라 "가시성 투명성(읽기)"** 입니다.

### ✅ 유형 1 — 읽기용 (전체 26개 폴더, 바로 가능)

```bash
git clone https://github.com/bmshin94/x-algorithm.git
cd x-algorithm

sed -n '285,470p' home-mixer/params/param.rs      # 가중치 전체
cat home-mixer/scorers/ranking_scorer.rs          # 점수 계산 로직
ls home-mixer/filters/                            # 필터 32개
cat visibility-filtering/rules/registry.rs        # 가시성 규칙
cat botmaker-rules/scarecrow/bot/nsfw_user_write_user_label.bot  # 라벨 규칙
cat docs/BIDIRECTIONAL_BOOST_CHANGE.md            # 변경 히스토리 (diff 포함)
```

→ **IDE(VS Code / Cursor)로 열어 읽는 것이 정상적인 사용법입니다.**

### ✅ 유형 2 — 실제 실행 가능: `phoenix/` 단 하나

**이전 릴리스는 Grok-1 기반 "샘플 모델"이었지만, 이번 릴리스는 프로덕션 실물 코드입니다.**
(실제 모델 정의 · 실제 학습 스텝 · 실제 Rust gRPC 서빙 엔진)
`nano` 프리셋은 **프로덕션 손실함수·피처 처리를 유지**하고 폭/깊이/테이블 크기만 축소했으며,
**랭킹 nano는 프로덕션과 기하 구조가 동일**합니다.

**요구사항**

| 항목 | 요구 |
|---|---|
| OS | **Linux** (Mac/Windows 불가) |
| GPU | **NVIDIA GPU + CUDA 12** (필수) |
| Python | 3.11+ 및 `uv` |
| 기타 | Rust 툴체인, `protoc` 3.15+ |

**실행 (`phoenix/QUICKSTART.md` 기준)**

```bash
cd phoenix
uv sync --extra engine
export PYTHONPATH=$PWD

# [0] 스모크 테스트 (랜덤 웨이트)
uv run python xrex/inference/oss_bench/bench.py --smoke --service_type ranking

# [1] 결정론적 합성 데이터 생성 (같은 seed = 같은 데이터)
uv run python reference/world_snapshots.py --out ./synth_index --seed 20260721
export PHOENIX_INDEX_BASE=./synth_index
uv run python reference/dump_gen.py --out ./synth_dump --seed 20260721 \
  --num-rows 12288 --partitions 4 --rows-per-file 1024 \
  --sid ./synth_index/sid_snapshot/post_sid_v5_256x6.parquet --self-check

# [2] 랭킹 모델 학습
uv run python reference/train_synth.py --data ./synth_dump --steps 6 --out "$PWD/checkpoints" --metrics

# [3] 이어서 학습 (모델·옵티마이저·데이터 위치까지 복원)
uv run python reference/train_synth.py --data ./synth_dump --steps 12 --out "$PWD/checkpoints"

# [4] 체크포인트 서빙 (gRPC)
RANK_CKPT=$(ls -d "$PWD"/checkpoints/home_direct_packed_nano_offline_kafka_dump/elapsed_samples_*/*/ | sort | tail -1)
uv run python xrex/inference/oss_bench/bench.py \
  --checkpoint_path "$RANK_CKPT" --service_type ranking \
  --config_name home_direct_packed_nano_offline_kafka_dump

# [5] 리트리벌 학습 → retrieve→rank 전체 루프
uv run python reference/train_synth.py \
  --config xrecsys_two_tower_nano_offline_kafka_dump \
  --data ./synth_dump --steps 6 --out "$PWD/checkpoints"
# (SID 서버 + 리트리벌 서버 + 랭킹 서버를 띄운 뒤)
uv run python reference/retrieve_then_rank.py \
  --data ./synth_dump --sessions 3 --topk 16 \
  --retrieval-port 9990 --ranking-port 9988
# 성공: "retrieve_then_rank: 3 session(s) completed the full loop."
```

**xAI가 명시한 한계:** nano 모델과 합성 데이터는 **추천 품질·프로덕션 성능·스케일을 입증하지 않습니다.**
"통합이 동작한다"는 것만 검증합니다.

**GPU 없을 때 대안:** 클라우드 GPU 1~3시간 대여(Vast.ai / RunPod / Lambda, A10G·L4급 시간당 약 $0.3~0.8) 또는 코드만 읽고 PyTorch로 소규모 재구현.

### ⚠️ 유형 3 — 실행 불가 (읽기 전용)

`home-mixer`, `thunder`, `visibility-filtering`, `vm-ranker` (Rust) /
`simclusters`, `agatha`, `botmaker`, `scarecrow`, `media-model-proxy` (Scala·Java) /
`grox`, `bdsm` (Python, 프롬프트 비공개) — **모두 내부 인프라 의존으로 빌드 불가**

---

## 10. 플러그인·스킬·MCP 여부

### ❌ 셋 다 아닙니다.

| 구분 | 해당? | 근거 |
|---|---|---|
| Claude 플러그인 | ❌ | `plugin.json`, `.claude-plugin/` 없음 |
| Claude 스킬 | ❌ | `SKILL.md` 없음 |
| MCP 서버 | ❌ | MCP SDK·서버 엔트리포인트·`mcp.json` 없음 |
| **대규모 분산 백엔드 서비스군의 소스코드** | ✅ | **정답** |

정확히는 **"X의 프로덕션 마이크로서비스 26개를 투명성 목적으로 공개한 폴리글랏 모노레포"** 입니다.

- **4개 언어 혼용**: Rust(요청 경로 — 지연시간), Scala/Java(배치·규칙엔진), Python(ML 학습)
- **구동 방식**: gRPC(`.proto`), Thrift(`.thrift`), Strato(xAI 내부 데이터 페더레이션, `.strato`), Kafka, Bazel

### 💡 다만, 이걸 감싸는 스킬/MCP는 직접 만들 수 있습니다

```
📦 "X Algorithm Advisor" Claude Skill
   SKILL.md에 가중치 테이블 + 필터 규칙 + 전략 로직 내장
   → "이 트윗 초안 점수 매겨줘" / "왜 안 퍼질까?"

📦 "x-algorithm" MCP Server
   get_weights()            현재 가중치 테이블 조회
   explain_filter(name)     필터 32개 개별 설명
   simulate_score(probs)    점수 시뮬레이션
   diff_params(c1, c2)      커밋 간 가중치 변화 추적
   check_draft(text)        초안 진단
   list_vf_rules()          가시성 규칙 목록
```

---

## 11. API 토큰 필요 여부

### ✅ 대부분 불필요

| 용도 | 토큰 | 상세 |
|---|---|---|
| 리포 클론·읽기 | ❌ 불필요 | 퍼블릭 리포 (Apache 2.0) |
| `phoenix/` 학습·서빙 | ❌ 불필요 | 합성 데이터 생성기 내장. **다운로드할 체크포인트·코퍼스 번들 자체가 없음** |
| X(트위터) API 토큰 | ❌ 불필요 | 이 리포는 X API를 호출하지 않음. 서버 내부 코드 |
| `grox/` 실행 | ⚠️ **필요** | `grox/libs/grok_sampler/eapi_sampler.py`가 `XAI_API_KEY` 요구 (Grok LLM 호출용). 단 **프롬프트(.j2) 비공개로 어차피 실행 불가** |

### 응용 시 필요한 것

| 만들 것 | 필요 | 비용 (변동) |
|---|---|---|
| 내 트윗 성과 분석 도구 | X API (Basic 약 $200/월, Pro 고가) | 높음 |
| 트윗 초안 AI 점수화 | LLM API (Claude / Grok / OpenAI) | 낮음 |
| 알고리즘 변경 모니터링 봇 | GitHub API (무료) | 무료~ |
| 학습용 시각화 웹앱 | 없음 (리포 데이터만 파싱) | **무료** |

> 🔑 **핵심: 리포 자체는 공짜고 토큰 0개. 외부 실데이터를 붙이는 순간부터 비용이 생깁니다.**
> → **수익화 설계에서 "X API 비용 회피"가 마진을 결정하는 최대 변수입니다.**

---

## 12. AI 에이전트 구축에 주는 도움

### ✅ 상당히 — 단 "직접 재료"가 아니라 "설계 교본"으로

### 바로 이식 가능한 5가지 패턴

#### 1️⃣ 멀티액션 예측 + 가중합 분리 (가장 유용)

```
❌ 흔한 설계: LLM에게 "이 콘텐츠 점수 0~100?" (블랙박스, 튜닝 불가)
✅ X 방식:   세부 확률을 따로 예측 → 가중치로 합산

→ 에이전트 적용:
   "유용성 0.8 / 긴급도 0.3 / 위험도 0.05 / 비용 0.2"를 따로 예측하고
   가중치만 config로 분리 → 재학습·재배포 없이 에이전트 성격 변경
```
근거: `ranking_scorer.rs`의 `ScoringWeights::from_params()` — 모든 가중치를 외부 설정에서 읽음

#### 2️⃣ "품질 판단"과 "허용 판단"의 완전 분리 (안전 설계의 핵심)

```
phoenix (순서)  ⟂  visibility-filtering (허용)
완전히 다른 서비스 · 다른 입력 · 다른 규칙

→ 에이전트 적용:
   "유용성 점수"와 "안전 가드레일"을 절대 한 프롬프트에 섞지 말 것.
   가드레일은 별도 모듈 + 별도 규칙 + 별도 감사 로그로.
   (프롬프트 인젝션 방어, 컴플라이언스 감사에 결정적)
```

#### 3️⃣ 조립식 파이프라인 — 6타입 추상화 (그대로 차용 가능)

```
Source    → 데이터 가져오기 (도구 호출)
Hydrator  → 컨텍스트 보강 (RAG 조회)
Filter    → 걸러내기 (가드레일)
Scorer    → 점수화 (LLM 판단)
Selector  → 선택 (Top-K, 라우팅)
SideEffect→ 응답 후 처리 (로깅, 메모리 기록)

특징: 독립 스테이지 자동 병렬 · graceful degradation · 실행/모니터링과 로직 분리
```

#### 4️⃣ 후보 격리 — 캐싱·일관성

```
X: 추론 시 후보들이 서로를 attend 못 함 → 같은 글 = 항상 같은 점수

→ 에이전트 적용:
   후보를 한 프롬프트에 몰아넣으면 "순서 편향(position bias)"이 생기고
   배치 구성에 따라 점수가 흔들립니다.
   각 후보를 독립 평가하면 → 재현 가능 + 점수 캐시 가능 + 비용 절감
```

#### 5️⃣ 단계별 On/Off + A/B 실험 내장

```
모든 단계가 param으로 토글 (EnableRanking, EnableAuthorDiversity, ...)
가중치 섭동(WeightPerturbationSigma)으로 탐색까지 내장

→ 에이전트 적용: 프롬프트/도구/필터를 설정값으로 토글 → 배포 없이 실험
```

### 추가로 차용할 디테일

| 패턴 | 위치 | 에이전트 응용 |
|---|---|---|
| 다양성 감쇠 `×0.5^n`, 하한 0.25 | `ranking_scorer.rs:603` | 같은 소스/도구 결과가 답변을 독점하지 않게 |
| 첫 Drop에서 즉시 종료 | `registry.rs` `Policy::evaluate` | 가드레일 평가 순서 최적화 (비싼 검사는 뒤로) |
| 3단 판정 (허용/차단/**경고 후 표시**) | `VfAction` enum | 이진 차단 대신 "확인 후 진행" UX |
| 골든 코퍼스 테스트 | `rules/golden_corpus.rs` | 규칙 변경 시 회귀 방지 — 평가셋 설계 |
| DPP 다양성 재배열 | `vm-ranker/` | 검색·추천 결과 중복 제거 |
| 해시 임베딩 | `phoenix/` | 어휘 사전 없이 신규 엔티티 즉시 벡터화 |
| PageRank 신뢰도 | `user-cred-v2/` | 멀티에이전트에서 에이전트/소스 신뢰도 점수화 |

### ⚠️ 한계

- **LLM 에이전트 코드가 아닙니다.** 추천 시스템 코드 → 복붙 불가, **아키텍처 사고법 이식**만 가능
- Rust/Scala라서 Python 에이전트 생태계와 직접 호환 안 됨
- 초당 수십만 요청 전제 설계 → 소규모에는 과잉일 수 있음

### 결론

> **프롬프트 복붙 자료로는 ❌, 시니어 레벨 에이전트 아키텍처 교본으로는 최상급 ⭐⭐⭐⭐⭐**
> 특히 ① 멀티액션+가중치 분리 ② 품질/안전 분리 ③ 6타입 파이프라인 — 이 3개는 즉시 적용 가치가 있습니다.

---

## 13. React / PHP 구현 가능 범위

| 만들 것 | React | PHP | 평가 |
|---|---|---|---|
| 가중치 시뮬레이터 (확률 입력 → 점수) | ✅ 완벽 | ✅ 완벽 | 수식 몇 줄. 하루면 완성 |
| 알고리즘 시각화·교육 웹앱 | ⭐ 최적 | ✅ 가능 | React 강력 추천 |
| 필터 32개 설명 백과사전 | ✅ | ✅ | 콘텐츠 사이트 |
| 트윗 초안 점수화 도구 | ✅ + LLM API | ✅ + LLM API | **상업화 1순위** |
| 파라미터 변경 추적 대시보드 | ✅ | ✅ | GitHub API로 diff 파싱 |
| 소규모 추천 피드 클론 (수천~수만 글) | ⚠️ 프론트만 | ⚠️ 백엔드 가능 | 가능하나 ML은 별도 필요 |
| ML 모델 학습 (Phoenix) | ❌ | ❌ | JAX/PyTorch + GPU 필수 |
| 초당 수십만 요청 실시간 랭킹 | ❌ | ❌ | Rust/C++ 영역 |

### 바로 쓸 수 있는 구현 — TypeScript

```typescript
// X For You 랭킹 점수 (프로덕션 기본값, home-mixer/params/param.rs:320-462)
const WEIGHTS = {
  favorite: 0.5,  reply: 5.0,  retweet: 1.0,  quote: 5.0,
  share: 2.0,  shareViaDm: 5.0,  shareViaCopyLink: 20.0,
  click: 0.4,  openLink: 0.2,  profileClick: 0.0,
  photoExpand: 0.05,  videoOpen: 0.07,  vqv: 0.0,
  dwell: 0.05,  contDwellTime: 0.004,  followAuthor: 4.0,
  postUnexplored: 0.02,
  notInterested: -43.2,  blockAuthor: -31.2,
  muteAuthor: -58.8,  report: -234.0,  notDwelled: -0.02,
} as const;

const BIDIRECTIONAL_FOLLOW_REPLY_BOOST = 15.0;  // 맞팔 시 reply 가중치에 가산
const OON_WEIGHT_FACTOR = 0.75;                 // 안 팔로우한 계정
const TOPIC_OON_WEIGHT_FACTOR = 0.5;            // 토픽 기반
const AUTHOR_DIVERSITY_DECAY = 0.5;
const AUTHOR_DIVERSITY_FLOOR = 0.25;

type Probs = Partial<Record<keyof typeof WEIGHTS, number>>;

function baseScore(p: Probs, isBidirectionalFollow = false): number {
  let s = 0;
  for (const [k, w] of Object.entries(WEIGHTS)) {
    const eff = (k === 'reply' && isBidirectionalFollow)
      ? w + BIDIRECTIONAL_FOLLOW_REPLY_BOOST
      : w;
    s += eff * (p[k as keyof typeof WEIGHTS] ?? 0);
  }
  return s;
}

// 작성자 다양성: n번째 글 (0-indexed)
function diversityMultiplier(n: number): number {
  return (1 - AUTHOR_DIVERSITY_FLOOR) * Math.pow(AUTHOR_DIVERSITY_DECAY, n)
         + AUTHOR_DIVERSITY_FLOOR;
}

function finalScore(p: Probs, opts: {
  inNetwork?: boolean; bidirectionalFollow?: boolean;
  authorSeenCount?: number; topicBased?: boolean;
} = {}): number {
  let s = baseScore(p, opts.bidirectionalFollow);
  s *= diversityMultiplier(opts.authorSeenCount ?? 0);
  if (!opts.inNetwork) {
    s *= opts.topicBased ? TOPIC_OON_WEIGHT_FACTOR : OON_WEIGHT_FACTOR;
  }
  return s;
}
```

### 바로 쓸 수 있는 구현 — PHP 8

```php
<?php
final class XRankingScorer
{
    public const WEIGHTS = [
        'favorite' => 0.5,  'reply' => 5.0,  'retweet' => 1.0, 'quote' => 5.0,
        'share' => 2.0,  'share_via_dm' => 5.0,  'share_via_copy_link' => 20.0,
        'click' => 0.4,  'open_link' => 0.2,  'profile_click' => 0.0,
        'photo_expand' => 0.05,  'video_open' => 0.07,  'vqv' => 0.0,
        'dwell' => 0.05,  'cont_dwell_time' => 0.004,  'follow_author' => 4.0,
        'post_unexplored' => 0.02,
        'not_interested' => -43.2,  'block_author' => -31.2,
        'mute_author' => -58.8,  'report' => -234.0,  'not_dwelled' => -0.02,
    ];

    public const BIDIRECTIONAL_FOLLOW_REPLY_BOOST = 15.0;
    public const OON_WEIGHT_FACTOR = 0.75;
    public const TOPIC_OON_WEIGHT_FACTOR = 0.5;
    public const AUTHOR_DIVERSITY_DECAY = 0.5;
    public const AUTHOR_DIVERSITY_FLOOR = 0.25;

    public static function diversityMultiplier(int $n): float
    {
        return (1.0 - self::AUTHOR_DIVERSITY_FLOOR)
               * (self::AUTHOR_DIVERSITY_DECAY ** $n)
               + self::AUTHOR_DIVERSITY_FLOOR;
    }

    public static function score(
        array $probs,
        bool $inNetwork = true,
        bool $bidirectionalFollow = false,
        int $authorSeenCount = 0,
        bool $topicBased = false,
    ): float {
        $s = 0.0;
        foreach (self::WEIGHTS as $k => $w) {
            if ($k === 'reply' && $bidirectionalFollow) {
                $w += self::BIDIRECTIONAL_FOLLOW_REPLY_BOOST;
            }
            $s += $w * (float)($probs[$k] ?? 0.0);
        }
        $s *= self::diversityMultiplier($authorSeenCount);
        if (!$inNetwork) {
            $s *= $topicBased ? self::TOPIC_OON_WEIGHT_FACTOR : self::OON_WEIGHT_FACTOR;
        }
        return $s;
    }
}
```

### 불가능한 것과 대안

| 불가능 | 이유 | 대안 |
|---|---|---|
| Phoenix 트랜스포머 학습 | JAX/XLA + GPU 커널 | Python으로 학습 → PHP/JS는 추론 API 호출 |
| 행동 확률 예측 | ML 본체 (학습된 모델) | 휴리스틱 근사 또는 소형 모델 별도 서빙 |
| 벡터 리트리벌 (수백만 건) | ANN 인덱스 필요 | pgvector / Qdrant / Pinecone |
| 실시간 초고성능 랭킹 | 레이턴시 한계 | 소규모는 문제없음. 대규모는 Rust/Go |
| 인메모리 최근글 저장소(thunder) | 수억 건 인메모리 | Redis로 축소 구현 |

### 권장 하이브리드 설계

```
┌──────────── React / Next.js (프론트엔드) ────────────┐
│  · 가중치 슬라이더 시뮬레이터                          │
│  · 파이프라인 10단계 인터랙티브 시각화                  │
│  · 트윗 초안 입력 → 예상 점수 + 개선 제안               │
│  · 필터 32개 / VF 규칙 백과사전                        │
└──────────────────────┬───────────────────────────────┘
                       │ REST / GraphQL
┌──────────────────────▼───────────────────────────────┐
│  PHP(Laravel) 또는 Node — 백엔드                      │
│  · 가중치 계산 엔진 (위 코드)                           │
│  · param.rs → JSON 자동 파서       ← 핵심 자산          │
│  · GitHub API 커밋 감시 → 가중치 변경 diff 알림          │
│  · 사용자/구독/결제 관리                                │
└──────────────────────┬───────────────────────────────┘
                       │ (선택) HTTP
┌──────────────────────▼───────────────────────────────┐
│  Python 마이크로서비스 (필요 시에만)                    │
│  · LLM 호출로 "행동 확률" 추정 / 소형 모델 추론           │
└──────────────────────────────────────────────────────┘
```

### 결론

> **분석·시뮬레이션·교육·점수화 도구는 React/PHP로 100% 가능하며 오히려 최적입니다.**
> **ML 학습과 초대규모 실시간 서빙은 불가능하지만, 상업적으로 필요하지도 않습니다.**
> 돈이 되는 영역(해석·도구·교육)이 정확히 React/PHP가 잘하는 영역입니다.

---

## 14. 유튜브 강의 제작 가능성

### ✅ 법적으로 완전히 가능하며, 콘텐츠 소재로는 최상급

| 항목 | 상태 |
|---|---|
| 라이선스 | **Apache 2.0** ✅ |
| 코드 화면 노출 / 수정 / 재배포 | ✅ 허용 |
| **상업적 이용 (수익 창출)** | ✅ **허용** |
| 유료 강의 판매 | ✅ 허용 |
| 조건 | 출처·라이선스·NOTICE 표기 (설명란 한 줄) |

**설명란 표기 예시**

```
📂 소스: https://github.com/xai-org/x-algorithm (Apache License 2.0)
   포크: https://github.com/bmshin94/x-algorithm
※ 본 영상은 공개된 오픈소스 코드를 분석한 교육 콘텐츠입니다.
   X Corp. / xAI와 제휴 관계가 없습니다.
```

**주의 2가지**
1. **X 로고·브랜드 자산은 Apache 2.0 범위 밖** → 썸네일에 공식 로고 직접 사용 금지 (텍스트/일반 아이콘 사용)
2. **"알고리즘 해킹/조작법" 톤 회피** → "알고리즘 이해 기반 콘텐츠 전략" 프레이밍이 안전하고 신뢰도도 높음

### 왜 소재로 최상급인가

| 강점 | 설명 |
|---|---|
| 거대한 모수 | X 사용자 수억 + 크리에이터 + 마케터 + 개발자 |
| "충격적 사실" 다수 | "좋아요 0.5 vs 답글 5.0 vs 링크복사공유 20.0" → 썸네일 즉시 완성 |
| 경쟁 거의 없음 | Rust/Scala 2,080개 파일을 실제로 읽은 사람이 극소수. 대부분 영상은 README 요약 |
| 근거가 코드 | "추측이 아니고 `param.rs:456`입니다" → 신뢰도 차원이 다름 |
| 지속 업데이트 | xAI가 계속 커밋 → 시리즈화 + 반복 조회수 |
| 이원 타깃 | 일반(그로스 팁) + 개발자(아키텍처) 둘 다 공략 |

### 시리즈 커리큘럼 (실제 리포 내용 기반)

**A. 일반/크리에이터 트랙 (조회수 중심)**

| # | 제목 | 길이 | 근거 |
|---|---|---|---|
| 1 | "X 알고리즘, 코드로 다 까봤습니다" — 좋아요 0.5, 링크공유 20.0 | 12분 | `param.rs:320-404` |
| 2 | "맞팔하면 15배 유리한 게 사실입니다" — 2026/07 실제 롤아웃 추적 | 10분 | `docs/BIDIRECTIONAL_BOOST_CHANGE.md` |
| 3 | "연속 트윗이 안 뜨는 이유" — 2번째 ×0.5, 하한 ×0.25 | 8분 | `ranking_scorer.rs:603` |
| 4 | "신규 계정 골든타임은 정확히 48시간" | 10분 | `param.rs:639-670` |
| 5 | "내 글이 왜 안 퍼지나? 필터 32개 전부 확인" | 15분 | `home-mixer/filters/` |
| 6 | "쉐도우밴의 실체" — 팔로워엔 보이고 추천엔 안 나가는 구조 | 12분 | `visibility-filtering/rules/registry.rs` |
| 7 | "내 계정 라벨 직접 확인하는 법" — Under the Hood 실습 | 8분 | `under-the-hood/` |
| 8 | "집단 신고로 계정 묻을 수 있나? — 코드가 말하는 진실" | 12분 | `param.rs:291-319` |

**B. 개발자 트랙 (고부가·전환율)**

| # | 제목 | 길이 |
|---|---|---|
| 9 | "억 단위 유저 추천 시스템 아키텍처 완전 해부" | 25분 |
| 10 | "Phoenix 모델 실제로 돌려봤습니다" — 합성데이터 학습→서빙 실습 | 30분 |
| 11 | "왜 Rust + Scala + Python 3개를 섞었나" | 15분 |
| 12 | "candidate-pipeline 11개 파일 — 추천 프레임워크 설계 교본" | 20분 |
| 13 | "Java 313개 파일로 만든 규칙 엔진 botmaker" — DSL 설계 | 20분 |
| 14 | "X 알고리즘을 React로 구현해봤습니다" — 시뮬레이터 라이브코딩 | 35분 |
| 15 | "AI 에이전트에 훔쳐올 5가지 패턴" | 20분 |

**C. 시리즈/루틴 (반복 수익)**

| # | 형식 |
|---|---|
| 16 | "이번 주 X 알고리즘 변경사항" — git diff 추적 주간 시리즈 (구독자 고정화) |
| 17 | "가중치 바뀌었습니다" — 변경 즉시 속보 영상 |

### 수익 구조 (6개 경로)

애드센스 · 제휴/스폰서(SNS 도구·스케줄러·분석툴) · 유료 강의 퍼널 · 전자책/치트시트 · 멤버십(주간 변경 분석) · 컨설팅 리드

### 제작 팁

- 화면 녹화 + 코드 하이라이트 (VS Code 다크테마, 폰트 크게)
- 가중치 표를 그래픽으로 제작해 매 영상 재사용 → 브랜딩
- 썸네일 공식: **숫자 + 반전** ("좋아요 0.5 vs 링크공유 20.0", "−234점의 정체")
- 첫 15초에 결론부터 → 근거 코드로 증명
- 시리즈 재생목록으로 체류시간 극대화
- **결정적 차별점: 줄 번호를 보여주세요.** 경쟁 영상은 README 요약인데, 당신은 `param.rs:456`을 띄울 수 있습니다.

---

## 15. 수익화 아이디어 상세

### 수익화 조건 분석

| 요소 | 평가 | 의미 |
|---|---|---|
| 라이선스 | Apache 2.0 | 상업 이용·수정·재배포 자유 |
| 원가 | **0원** | 리포 무료, API 토큰 불필요 |
| 진입장벽 | **높음(유리)** | 2,080개 파일 해독 가능자 극소수 → **해석이 상품** |
| 시장 규모 | 초대형 | X 사용자 수억 + 크리에이터 + 마케터 + 개발자 |
| 반복성 | 높음 | xAI가 계속 커밋 → 구독 모델 성립 |
| 최대 리스크 | **X API 비용** | 실데이터 연동 시 월 $200~수천 |

> 🔑 **"X API를 안 쓰는 상품"이 마진 100%, "X API를 쓰는 상품"은 마진 압박.**
> → **가중치·규칙 "해석"을 파는 사업이 구조적으로 가장 유리합니다.**

---

### 🏆 Tier 1 — 즉시 시작 · 원가 거의 0 · 추천 최상

#### 1️⃣ 📰 "알고리즘 변경 추적" 유료 뉴스레터 / 멤버십 ⭐⭐⭐⭐⭐

**작동 방식**

```
GitHub API로 리포 감시 (무료)
   ↓
param.rs 가중치 숫자 변경 자동 감지  (예: reply_weight 5.0 → 7.0)
filters/ 디렉토리 diff로 필터 추가·삭제 감지
visibility-filtering 규칙 변경 감지
   ↓
"이게 당신의 X 계정에 무슨 의미인가"로 번역
   ↓
무료 구독자: 요약 알림 / 유료: 상세 분석 + 전략 액션 아이템
```

**수익 모델**

| 티어 | 가격 | 내용 |
|---|---|---|
| 무료 | 0원 | 변경 알림 + 1줄 요약 (리드 수집) |
| **Pro** | **월 9,900~19,900원** | 상세 해설 + 전략 액션 + 과거 아카이브 |
| **Business** | **월 99,000원~** | 브랜드 맞춤 영향 분석 + Q&A |

- **시나리오**: 유료 300명 × 14,900원 = **월 447만원** (원가 메일 발송비 약 3만원)
- **구축**: GitHub Actions(무료) + `param.rs` 파서 + Stibee/Substack/Ghost
- **소요** 1~2주 / **초기비용** 5만원 이하
- **강점**: 완전 자동화 · 반복 수익 · 선점 효과 · 경쟁 없음

#### 2️⃣ 🎯 "트윗 초안 알고리즘 점수화" SaaS ⭐⭐⭐⭐⭐

**핵심: X API가 필요 없습니다.** 사용자가 초안을 붙여넣고, 가중치 모델로 채점합니다.

```
사용자 입력: 트윗 초안
   ↓
① LLM(Claude/Grok)으로 행동 확률 추정
   → 답글 달 확률? 인용할 확률? 링크 복사 공유할 확률?
   ↓
② param.rs 가중치로 최종 점수 계산
   → favorite×0.5 + reply×5.0 + shareViaCopyLink×20.0 + ...
   ↓
③ 리포 기반 구체적 개선 제안
   · "답글 가중치가 좋아요의 10배 → 질문형으로 바꾸세요"
   · "링크 복사 공유가 최고 가중치(20.0) → 저장 가치 있는 정보 추가"
   · "muted_keyword 필터에 자주 걸리는 단어 포함"
   · "48시간 필터 → 지금이 최적 발행 시각인지 확인"
   · "연속 포스팅 감지: ×0.5 감쇠 중. 2시간 간격 권장"
   ↓
④ 개선 전/후 예상 점수 비교 + A/B 초안 2개 생성
```

**수익 모델**

| 티어 | 가격 | 내용 |
|---|---|---|
| Free | 0원 | 1일 3회 (리드) |
| **Creator** | **월 14,900원** | 무제한 + 개선안 + 히스토리 |
| **Pro** | **월 39,900원** | 쓰레드 분석 + 발행 타이밍 + 경쟁 분석 |
| **Agency** | **월 149,000원** | 멀티 계정 + 팀 + API 접근 |

- **시나리오**: 유료 500명 × 평균 20,000원 = **월 1,000만원** (원가 LLM API 약 80만원 → **마진 92%**)
- **스택**: React/Next.js + PHP(Laravel) 또는 Node + Claude API
- **소요** MVP 3~4주 / **초기비용** 20~50만원
- **강점**: X API 불필요 = 원가 혁명 · 즉각 체감 가치 · 바이럴
- ⚠️ **주의**: "점수는 추정치"임을 명확히 고지 (실제 모델 예측값 아님). 과장 금지

#### 3️⃣ 🎓 온라인 강의 + 전자책 ⭐⭐⭐⭐⭐

| 상품 | 가격 | 설명 |
|---|---|---|
| 가중치 치트시트 PDF | **9,900원** | 가중치 전표 + 필터 32개 + 전략 요약 (리드 상품) |
| 전자책 "X 알고리즘 코드 해부" | **29,000원** | 전수조사 결과 전체 |
| 강의 "X 알고리즘 실전 그로스" | **99,000원** | 크리에이터 트랙 8강 |
| 강의 "억대 유저 추천시스템 아키텍처" | **249,000원** | 개발자 트랙 7강 + Phoenix 실습 |
| 라이브 워크샵 | **199,000원/인** | 월 1회 소수 정예 |

- **시나리오**: 월 100건 × 평균 60,000원 = **월 600만원** (원가 플랫폼 수수료 외 0)
- **채널**: 인클래스/클래스101/탈잉/크몽(국내) · Udemy/Gumroad(해외, 영문화 시 시장 10배)
- **소요**: 치트시트 3일 / 전자책 3주 / 강의 6주

#### 4️⃣ 📺 유튜브 채널 ⭐⭐⭐⭐⭐

- **6개 수익 경로**: 애드센스 · 스폰서 · 강의 퍼널 · 전자책 · 멤버십 · 컨설팅 리드
- **시나리오**: 구독 3만 + 월 50만 뷰 → 애드센스 100~300만 + 스폰서 100~300만 + 퍼널 300~1,000만 = **월 500~1,600만원**
- **결정적 차별점**: 경쟁 영상은 README 요약. 당신은 `param.rs:456` 줄 번호를 띄울 수 있습니다.

---

### 🥈 Tier 2 — 개발 역량 필요 · 수익 상한 높음

#### 5️⃣ 🔌 Claude Skill / MCP 서버 ⭐⭐⭐⭐

- **상품**: 10번 섹션의 Skill / MCP 구성 그대로
- **수익**: Gumroad/LemonSqueezy 단품 **$29~79** · 팀 라이선스 **$199~** · 또는 무료 배포 후 SaaS 업셀
- **시나리오**: 월 50개 × $49 ≈ **월 330만원**
- **강점**: 개발자 커뮤니티 바이럴 · 제작 1~2주 · **얼리무버 효과(이 영역 거의 비어 있음)**

#### 6️⃣ 🌐 "X 알고리즘 백과사전" 웹사이트 ⭐⭐⭐⭐

```
🔍 가중치 인터랙티브 시뮬레이터 (슬라이더)
📊 파이프라인 10단계 시각화 (클릭 → 해당 코드)
📖 필터 32개 개별 페이지 — "이 필터에 왜 걸리나"
🚦 가시성 규칙 전체 해설 (표시/삭제/인터스티셜)
📈 파라미터 변경 타임라인 (자동 갱신)
🔄 커밋 추적 대시보드
🌍 다국어 (한/영/일 — SEO 3배)
```

- **수익**: 애드센스 + 제휴 + 프리미엄 구독(월 4,900원) + 전자책/강의 퍼널 + 스폰서 배너
- **시나리오**: 월 20만 PV → 광고 150만 + 구독 300명×4,900 = 147만 + 퍼널 = **월 400~800만원**
- **스택**: Next.js + `param.rs` → JSON 자동 파서 + GitHub Actions 일일 갱신
- **핵심 자산(moat)**: **파서를 만들면 xAI가 값을 바꿀 때마다 사이트가 자동 최신화** — 경쟁자가 수동으로는 못 따라옴
- **소요** 4~6주

#### 7️⃣ 📊 크리에이터 분석 SaaS (X API 사용) ⭐⭐⭐

- **기능**: 내 트윗 실적 역분석 → 부족한 행동 신호 진단, 최적 발행 시각, 맞팔 네트워크 분석(+15 활용), 연속 포스팅 감쇠 경고
- ⚠️ **X API 비용**: Free 사실상 불가 / Basic 약 $200/월 / Pro $5,000/월
- **시나리오**: 유료 200명 × 평균 50,000원 = **월 1,000만원** (원가 API+LLM 약 150만원)
- **전략**: **아이디어 2(API 불필요)로 먼저 유저 확보 → 여기로 업셀.** 처음부터 API 비용을 짊어지지 말 것
- **소요** 6~10주 / **난이도** 상

---

### 🥉 Tier 3 — 고단가 · 개인 역량 의존

#### 8️⃣ 💼 기업 컨설팅 / 자문 ⭐⭐⭐⭐ (단가 최상)

| 서비스 | 단가 |
|---|---|
| 기업 임직원 세미나 (2h) | **200~500만원** |
| 브랜드 X 그로스 전략 수립 | **500~2,000만원/프로젝트** |
| 추천시스템 설계 자문 (리테이너) | **월 300~1,000만원** |
| 알고리즘 투명성·컴플라이언스 자문 | **프로젝트당 1,000만원+** |

- **타깃**: 소셜 마케팅 대행사 · 커머스/미디어(추천 엔진) · AI 스타트업 · 언론(알고리즘 취재)
- **시나리오**: 월 1~2건 = **월 500~2,000만원**
- **리드**: 유튜브·뉴스레터로 전문성 증명 → 인바운드 (**Tier 1이 Tier 3의 영업 채널**)

#### 9️⃣ 🏗️ 추천 엔진 구축 서비스 (B2B) ⭐⭐⭐

- **상품**: 이 리포 아키텍처를 중소 규모로 축소 구현해 납품 (멀티액션+가중합, 6타입 파이프라인, 다양성 감쇠, 안전 필터 분리)
- **타깃**: 커머스 · 콘텐츠 플랫폼 · 커뮤니티 · 뉴스 앱 · 구인구직
- **수익**: 구축 **2,000만~1억원** + 유지보수 **월 200~500만원**
- **강점**: "X 프로덕션 아키텍처 기반" 레퍼런스가 강력한 세일즈 포인트
- **소요** 건당 2~6개월 / **난이도** 최상

#### 🔟 기타 아이디어

| 아이디어 | 수익 | 난이도 |
|---|---|---|
| 모바일 앱 (가중치 계산기 + 체크리스트) | 인앱 월 4,900원 | 중 |
| "내 계정 라벨 진단" — Under the Hood 결과 해석 | 건당 29,000원 / 월 9,900원 | 중 |
| 오프라인 세미나·밋업 | 인당 5~15만원 | 하 |
| 제휴 마케팅 — SNS 스케줄러·분석툴 추천 | 커미션 20~40% | 하 |
| 1:1 코칭 — 크리에이터 그로스 | 세션 15~30만원 | 하 |
| 영문 콘텐츠 전환 | 시장 10배 | 중 |
| 오픈소스 + 스폰서십 (시뮬레이터 OSS 공개) | 월 50~300만원 | 중 |
| 데이터셋 상품 — 가중치 변경 이력 CSV/API | 월 29,000원 | 중 |

---

### 종합 비교표

| # | 아이디어 | 초기비용 | 소요 | 월 수익 잠재 | 난이도 | 마진 | 추천 |
|---|---|---|---|---|---|---|---|
| 1 | 📰 변경 추적 뉴스레터 | 5만↓ | 1~2주 | 100~500만 | 하 | 99% | ⭐⭐⭐⭐⭐ |
| 2 | 🎯 초안 점수화 SaaS | 20~50만 | 3~4주 | 300~1,500만 | 중 | 92% | ⭐⭐⭐⭐⭐ |
| 3 | 🎓 강의·전자책 | 0~20만 | 3~6주 | 200~1,000만 | 하 | 95% | ⭐⭐⭐⭐⭐ |
| 4 | 📺 유튜브 | 10~50만 | 지속 | 300~1,600만 | 중 | 95% | ⭐⭐⭐⭐⭐ |
| 5 | 🔌 Skill/MCP | 0 | 1~2주 | 100~500만 | 중 | 97% | ⭐⭐⭐⭐ |
| 6 | 🌐 백과사전 사이트 | 10~30만 | 4~6주 | 200~800만 | 중 | 93% | ⭐⭐⭐⭐ |
| 7 | 📊 분석 SaaS (X API) | 300만+ | 6~10주 | 500~2,000만 | 상 | 60% | ⭐⭐⭐ |
| 8 | 💼 컨설팅 | 0 | 즉시 | 500~2,000만 | 상 | 95% | ⭐⭐⭐⭐ |
| 9 | 🏗️ 추천엔진 구축 | 0 | 2~6개월 | 1,000만+ | 최상 | 70% | ⭐⭐⭐ |

---

## 16. 첫 90일 실행 로드맵

```
━━━ 1~2주차 : 자산 만들기 (무료, 리스크 0) ━━━━━━━━━━━━━━━━━
✅ 가중치 치트시트 PDF 제작 → 무료 배포로 이메일 리스트 수집
✅ param.rs → JSON 파서 작성          ← 🏰 모든 사업의 핵심 자산
✅ 유튜브 1편: "X 알고리즘 코드로 다 까봤습니다"
✅ GitHub Actions로 리포 변경 감시 세팅

━━━ 3~6주차 : 첫 수익 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💰 뉴스레터 유료 티어 오픈 (아이디어 1)
💰 전자책 출간 (29,000원)
📺 유튜브 주 1~2편 (크리에이터 트랙 8편 소진)
🔌 무료 Claude Skill / MCP 배포 → 개발자 유입

━━━ 7~12주차 : 스케일업 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 초안 점수화 SaaS MVP 출시 (아이디어 2)  ← 가장 큰 수익원
🌐 백과사전 사이트 오픈 (SEO 장기 자산)
🎬 강의 1개 출간 (99,000원)
💼 컨설팅 인바운드 수용 시작

━━━ 3개월 이후 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🌏 영문 전환 (시장 10배)
📊 X API 버전 업셀 (아이디어 7) — 유저 확보된 뒤에만
🏗️ B2B 추천엔진 구축 (아이디어 9)
```

### 전략의 핵심 3가지

1. **🏰 `param.rs` 파서가 진짜 자산입니다.** xAI가 값을 바꿀 때마다 제품 전체가 자동 최신화 — 경쟁자가 수동으로는 못 따라옵니다. **이것부터 만드세요.**
2. **🔗 Tier 1이 Tier 3의 영업 채널입니다.** 유튜브/뉴스레터로 전문성 증명 → 500만~2,000만원 컨설팅 인바운드. 콘텐츠가 곧 영업입니다.
3. **💸 X API는 최대한 늦게 붙이세요.** 아이디어 1·2·3·4는 API 비용 0원입니다. 매출 월 500만원 돌파 후에 확장하세요.

### 리스크 관리

| 리스크 | 대응 |
|---|---|
| 알고리즘 변경으로 콘텐츠가 낡음 | **오히려 기회** — "변경 추적"이 상품 (아이디어 1) |
| "알고리즘 조작법"으로 비칠 우려 | **"이해 기반 전략"** 프레이밍으로 통일 |
| X 상표권 | 공식 로고 미사용, "비제휴" 명시 |
| 점수 과장 논란 | **"추정치"** 명확 고지, 실제 모델 예측값 아님을 반복 안내 |
| X API 정책·가격 변동 | API 비의존 상품을 주력으로 유지 |
| 경쟁자 진입 | **선점 + 파서 자동화 + 아카이브 축적**이 방어선 |

---

## 17. 한계 및 비공개 영역

### 비공개 (xAI가 명시)

| 항목 | 이유 |
|---|---|
| **Grox LLM 프롬프트** (`.j2` 파일) | 어뷰징 우회(게이밍) 방지 |
| **일부 botmaker 규칙** | 동일 |
| **빌드·배포 파일** (`phoenix/` 제외) | 목적이 "가시성 투명성"이라서 |
| **프로덕션 데이터·체크포인트·오케스트레이션·스케일** | 당연히 비공개 |

→ 대신 **"코드 공개 + 결과 공개(Under the Hood)"** 조합으로 투명성을 확보하는 전략입니다.
그래서 사용자는 ① 이 시스템들의 **결과**를 볼 수 있고 ② 자동 시스템 밖에서 **수동 적용된 라벨**도 확인할 수 있고 ③ 라벨을 코드와 **매칭해 영향을 추론·비판**할 수 있습니다.

### 실행 한계 요약

| 구분 | 상태 |
|---|---|
| 전체 리포 빌드·실행 | ❌ 불가 (루트 빌드 파일 없음, 내부 인프라 의존) |
| `phoenix/` 학습·서빙 | ✅ 가능 (Linux + NVIDIA GPU + CUDA 12 필요) |
| 나머지 25개 폴더 | 📖 읽기 전용 |
| 프로덕션 품질 재현 | ❌ 불가 (nano 모델 + 합성 데이터는 통합 검증용) |

---

## 📎 핵심 파일 빠른 참조

| 알고 싶은 것 | 파일 |
|---|---|
| **가중치 전체 + 오해 정정 주석** | `home-mixer/params/param.rs:291-462` |
| 점수 계산·보정 로직 | `home-mixer/scorers/ranking_scorer.rs` |
| 다양성 감쇠 수식 | `home-mixer/scorers/ranking_scorer.rs:603-604` |
| 신규 작성자 부스트 | `home-mixer/scorers/author_cold_start.rs`, `param.rs:639-670` |
| 필터 32개 | `home-mixer/filters/` |
| 국가 법률 대응 필터 예 | `home-mixer/filters/brazil_2026_election_filter.rs` |
| 가시성 규칙 + 평가 순서 | `visibility-filtering/rules/registry.rs` |
| 라벨 규칙 DSL 예시 | `botmaker-rules/scarecrow/bot/*.bot` |
| 파이프라인 프레임워크 | `candidate-pipeline/` (11개 파일) |
| ML 모델 실행 가이드 | `phoenix/QUICKSTART.md`, `phoenix/TRAINING.md`, `phoenix/README.md` |
| **알고리즘 변경 추적 예시** | `docs/BIDIRECTIONAL_BOOST_CHANGE.md` |
| 맞팔 부스트 구현 | `home-mixer/candidate_hydrators/bidirectional_follow_hydrator.rs` |

---

## ✅ 한 문장 요약

> **X 피드는 ① 3곳(thunder/phoenix/simclusters)에서 글을 모아 ② 17개 필터로 거르고 ③ AI가 30여 가지 행동 확률을 예측해 가중합하고 ④ 다양성·OON·신인 보정을 하고 ⑤ 안전 규칙으로 최종 통과 여부를 별도 판정한 뒤 ⑥ 광고를 끼워 내보내는 조립식 실시간 파이프라인이며, 이 리포는 그 코드 원본입니다.**
>
> **활용 가치는 ⓐ X 그로스 전략의 "코드 근거" ⓑ 대규모 추천시스템 설계 교본 ⓒ AI 에이전트 아키텍처 패턴 ⓓ 콘텐츠·교육·SaaS 사업의 원재료 — 네 방향이며, Apache 2.0이라 상업적 활용에 법적 제약이 없습니다.**

---

### 🔗 링크 모음

- **이 리포 (포크)**: https://github.com/bmshin94/x-algorithm
- **원본 (업스트림)**: https://github.com/xai-org/x-algorithm
- **라벨 투명성 도구**: https://x.com/i/under_the_hood
- **라이선스**: [Apache License 2.0](../LICENSE)
- **알고리즘 변경 예시 문서**: [BIDIRECTIONAL_BOOST_CHANGE.md](BIDIRECTIONAL_BOOST_CHANGE.md)
- **Phoenix 퀵스타트**: [../phoenix/QUICKSTART.md](../phoenix/QUICKSTART.md)

---

*분석 일자: 2026-10-08 · 대상: `bmshin94/x-algorithm` (upstream `xai-org/x-algorithm`) · 전수조사 2,080개 파일 / 26개 서브시스템*
*본 문서의 모든 수치는 분석 시점의 리포 코드 기본값이며, xAI의 실험·커밋에 따라 변경될 수 있습니다.*
