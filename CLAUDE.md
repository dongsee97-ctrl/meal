# 구내식단 운영판 — 작업 규칙

언빌리버블컴퍼니 사내 구내식당 운영 웹앱입니다.
**이 문서는 Claude Code가 자동으로 읽습니다. 코드를 고치기 전에 반드시 지켜주세요.**

---

## 0. 가장 먼저 — 절대 하면 안 되는 것

이 다섯 가지는 과거에 실제로 사고가 났던 항목입니다.

1. **메뉴·재료 데이터를 임의로 만들지 마세요.**
   예시 메뉴, 샘플 재료, 추정 단가를 넣지 마세요. 전부 사용자가 직접 입력합니다.
   "비어 있길래 예시로 채웠습니다"는 이 프로젝트에서 가장 하면 안 되는 행동입니다.

2. **기존 기능과 데이터를 지우지 마세요.**
   예전에 저장된 데이터도 그대로 열려야 합니다. 형태가 바뀌었으면 정규화 함수에서
   **읽는 시점에만** 보정합니다. 저장된 원본을 마이그레이션으로 덮어쓰지 않습니다.
   수정 전후로 함수 개수를 세어 줄었는지 확인하세요(아래 9번).

3. **Firebase는 v8 compat SDK를 유지하세요.**
   v9 modular로 바꾸면 `snap.exists`가 속성에서 함수로 바뀌어 수십 군데가 한꺼번에 깨집니다.
   "최신 버전으로 올려드릴까요?"는 하지 마세요.

4. **네이티브 `confirm` / `alert` / `prompt`를 쓰지 마세요.**
   실행 환경에서 조용히 차단되어 "눌렀는데 아무 반응이 없는" 증상이 납니다.
   반드시 `customConfirm`, `customAlert`, `customPromptText`, `showModal`을 쓰세요.

5. **파일 전체를 다시 쓰지 마세요.**
   5,300줄짜리 단일 파일입니다. 항상 필요한 부분만 국소 치환하세요.
   전체 재작성은 기능 유실의 가장 큰 원인입니다.

---

## 1. 구조

빌드 과정이 없습니다. `index.html` 하나에 HTML·CSS·JavaScript가 전부 들어 있고,
같은 폴더의 `pretendard.woff2`만 있으면 동작합니다.

| 파일 | 내용 |
|---|---|
| `index.html` | 앱 전체 (약 5,300줄 / 344KB / 함수 350개) |
| `pretendard.woff2` | 본문 글꼴. **index.html과 반드시 같이 배포** |

### 저장소 추상화

Firestore 모양의 같은 API 뒤에 세 가지 구현이 있습니다.

| 백엔드 | 쓰이는 때 |
|---|---|
| Firebase Firestore (v8 compat) | 실제 운영 |
| Artifact `db` capability | Claude Artifact 안에서 구동 |
| `makeLocalDb()` — localStorage | 오프라인·테스트 (구글 도메인이 막히면 자동 폴백) |

덕분에 네트워크를 전부 차단해도 로컬 모드로 정상 구동됩니다. 테스트는 이걸 씁니다.

### 렌더링

전역 상태 `S`를 두고, 바뀌면 `render()`가 현재 탭을 통째로 다시 그립니다.
가상 DOM이 없으므로 **인라인 `onclick`/`onchange` + 전역 함수** 패턴을 씁니다.

- DB 스냅샷은 frozen입니다. 고치기 전에 **`deepClone` 필수**.
- `render()`의 **`renderGen` 레이스 가드를 건드리지 마세요.** 비동기 렌더가 겹칠 때
  옛 결과가 새 화면을 덮는 것을 막는 장치입니다.
- 사용자 자유 입력을 인라인 핸들러 안에 넣을 때는 **`escJsAttr()`**, 본문에는 `esc()`.

### 탭 10개

```
dashboard  대시보드        plan     영업부 식단표
service    용역부 간식표·도시락   menus    메뉴 관리
order      재료구매(발주)   ingredients 재료·상품 관리
cost       원가분석        settings 설정
suppliers  거래처 관리      users    회원 관리
```

사이드바는 영업부 / 용역부 / 관리 / 설정 네 구역입니다.
구역 배경은 반드시 `--surface`를 쓰세요. `--accent-soft`로 칠하면 활성 메뉴 하이라이트가 묻힙니다.

---

## 2. 권한

| 권한 | 보이는 탭 | 수정 | 금액 |
|---|---|---|---|
| 일반 | dashboard · plan · service 셋뿐 | X | **란 자체가 없음** |
| 권한지정 | 전체 | O | O |
| 관리자 | 전체 + 회원 승인 | O | O |

**금액을 마스킹(`••••`)하지 마세요.** 예전 방식이고, 숫자만 가려진 게 어색하다는
피드백으로 **금액 마크업 자체를 렌더링하지 않도록** 바꿨습니다. 되돌리지 마세요.

```js
money(html)        // canSeeMoney()가 false면 빈 문자열 → 금액이 아예 안 나감
TABS_FOR_GENERAL   // ['dashboard','plan','service']
readOnlyBanner()   // 빈 문자열 반환 (안내 띠 제거 요청 반영됨)
```

판정: `isAdmin()` `canEdit()` `canSeeMoney()` `isReadOnly()` `canViewTab()`
적용: `applyNavPermissions()` (`.navgroup` 단위 숨김) / `applyPermissionGuards()`

**새로 만든 쓰기 동작은 `VIEW_ONLY_HANDLERS`에 넣지 마세요.** 기본이 '차단'이라 그냥 두면 보호됩니다.

서버 쪽 Firestore 보안 규칙도 같은 권한을 강제합니다. 화면 처리는 1차 방어이고
**최종 방어선은 보안 규칙**입니다.

---

## 3. 인쇄물

네 가지(영업부 주간·월간, 용역부 간식표, 용역부 도시락)가 **공통 부품**을 씁니다.
디자인을 고칠 때는 부품만 고치면 전부 함께 바뀝니다.

```js
printShell({titleHead, titleEm, titleTail, sub, note, foot, body, kind})
printWeekCards(dates, bodyFor, themeFor)   // 요일 카드 5장
printMonthGrid(weekMondays, monthIdx, bodyFor)  // 달력
catIconLinesHtml(items)   // [{cat,name}] → 아이콘 + 이름 줄
qtyLinesHtml(items)       // [{name,qty}]  → 간식용 줄
```

규칙:

- **인쇄물에 금액을 절대 넣지 마세요.**
- 분류 이름(밥·메인·반찬)을 글자로 적지 않습니다. 순서·아이콘·메인 강조로만 구분합니다.
- 강조는 하루에 하나. `메인`이 있으면 그것, 없으면 `면`(면데이의 주메뉴).
- 아이콘은 **분류로만** 정합니다(`foodIconKey`). 메뉴 이름으로 넘겨짚지 마세요.
- 그림은 전부 파일 안에 그린 SVG입니다. **외부 이미지를 쓰지 마세요**(오프라인 인쇄가 깨집니다).
- 주간은 A4 가로 한 장을 꽉 채우고(`.pwrap-week`), 월간은 5주가 한 장에 들어가도록
  촘촘합니다(`.pwrap-month`). 월간 글자 크기를 키우면 2페이지로 넘어갑니다.

---

## 4. 메뉴 줄 순서

```js
PLAN_CAT_RANK = {밥:10, 면:11, 국:20, 메인:30, 서브메인찬:40, 서브메인:40, 반찬:60, 후식:85, 간식:90}
planCatGroup(cat)   // 0: 밥~서브메인 / 1: 일반 반찬 / 2: 후식·간식 — 그룹 사이에 점선
```

목록에 없는 분류는 `메인` 포함 → 45, 그 외 → 55로 폴백합니다.

---

## 5. 금액 집계

```js
dayTotals(dateKey) → {perPerson, salesMeal, serviceMeal, snack, meal, total}
// meal  = salesMeal + serviceMeal   (대시보드 '식사(도시락포함)')
// total = meal + snack
```

**과거에 고친 버그 두 가지입니다. 되돌리지 마세요.**

1. `dayCost()`가 간식을 `it.qty * it.unitPrice`로 직접 곱해서, 재료·메뉴에 **연결된 간식이
   0원으로 집계**됐습니다 → `snackDayTotal(dateKey)`를 쓰도록 수정.
2. `serviceDayCost()`가 **전날 영업부 식단 원가를 항상 더해서** 도시락이 이중 계상됐습니다
   → `설정 > serviceBaseMealInOrder`가 **켜져 있을 때만** 계상.
   현재 운영은 12인분을 조리해 남는 것을 보내는 방식이라 **꺼두는 것이 맞습니다.**

검증 사례(9/22): 45,936(3,828×12명) + 2,850(용기 270+발열팩 300, ×5명) + 10,983(간식) = **59,769원**

---

## 6. 용역부 간식 품목 연동

간식 한 줄이 세 상태를 가집니다. **기존 데이터는 그대로 동작합니다.**

| 상태 | 저장 필드 | 단가 |
|---|---|---|
| 재료·상품에 연결 | `ingId` | 구매가 ÷ 구매수량 |
| 메뉴에 연결 | `menuId` | 그 메뉴의 1인 원가 |
| 직접 입력 | 없음 | `unitPrice` |

연결한 항목이 삭제되면 **조용히 직접 입력으로 취급**되어 이름·단가가 남습니다(데이터 손실 없음).

---

## 7. 디자인 토큰

색은 하드코딩하지 말고 변수를 쓰세요. 밝게가 기본입니다.

```css
--bg --surface --surface-2 --surface-3
--ink --ink-soft --ink-faint
--border --border-soft --bw(테두리 두께, 현재 1.5px)
--accent --accent-ink --accent-soft --accent-strong
--amber --amber-soft --danger --danger-soft
--font  (Pretendard)
```

- 새 테두리는 `var(--bw) solid var(--border)`.
- 어둡게 정의는 `:root[data-theme="dark"]`와
  `@media (prefers-color-scheme: dark){:root:not([data-theme="light"])}` **양쪽에** 넣으세요.
- 명암비 WCAG AA를 지킵니다(현재 amber 5.15 / danger 5.37 / ink-faint 5.02).

---

## 8. 테스트

Playwright로 돌립니다. **배포 전에 반드시 통과시키세요.**

```bash
node <테스트파일>.js
```

주의사항:

- 모달을 여는 앱 함수는 `page.evaluate(() => { fn(); })`로 감쌀 것
- 앱 전역은 `let` 선언이라 `window.S`가 아니라 `typeof S !== 'undefined'`로 확인
- 외부 요청(구글·CDN)은 abort/stub 처리. 앱이 자동으로 localStorage 폴백으로 구동됩니다
- 로컬 모드에는 **앱 자체의 샘플 데이터**가 비동기로 들어옵니다. 픽스처를 넣으려면
  적재가 끝난 뒤 `saveMealPlanDay()` 같은 **앱의 저장 경로**를 쓰세요.
  `S.mealPlan`에 직접 대입하면 리스너가 되돌립니다
- 스크린샷은 전체 뷰포트로. `clip`을 주면 사이드바가 잘립니다(폭 1560px 권장)

---

## 9. 수정을 끝낼 때마다

```bash
# 함수가 줄지 않았는지 (현재 350개)
grep -oP '^\s*(async )?function \K\w+' index.html | sort -u | wc -l

# 문법 확인
node -e "const h=require('fs').readFileSync('index.html','utf8');
const re=/<script(?:\s[^>]*)?>([\s\S]*?)<\/script>/g;let m,bad=0;
while((m=re.exec(h))){if(!m[1].trim())continue;try{new Function(m[1]);}catch(e){bad++;console.log(e.message);}}
console.log('오류 '+bad);"
```

체크리스트:

- [ ] 기존 함수를 지우지 않았는가
- [ ] 예시 데이터를 넣지 않았는가
- [ ] 네이티브 confirm/alert/prompt를 쓰지 않았는가
- [ ] 새 금액 표시에 `money()`를 씌웠는가
- [ ] 색을 하드코딩하지 않고 CSS 변수를 썼는가
- [ ] 밝게/어둡게 양쪽에서 확인했는가
- [ ] 일반 권한 계정에서 금액이 안 보이는가
- [ ] 인쇄물에 금액이 안 들어갔는가
- [ ] 테스트가 전부 통과하는가

---

## 10. 공동 작업 — 충돌 주의

**단일 파일이라 두 사람이 동시에 고치면 Git이 자동 병합을 못 합니다.**
서로 다른 기능을 만들어도 같은 파일이라 충돌이 납니다. 아래를 지켜주세요.

1. **작업 시작 전에 항상** `git pull --rebase origin main`
2. **작게, 자주 커밋하고 바로 push.** 하루치를 모아서 올리면 충돌 범위가 커집니다
3. **작업 전에 어느 영역을 건드릴지 서로 알리세요**
   (예: "나는 인쇄물 쪽", "나는 발주 집계 쪽")
4. 충돌이 나면 **둘 다 살리는 쪽으로** 해결하세요. 한쪽을 통째로 버리면 기능이 사라집니다
5. `main`에 직접 push하지 말고 브랜치 → PR → 병합

> 동시 작업이 잦아지면 **파일 분할**(CSS·JS를 별도 파일로)을 검토하세요.
> 충돌이 크게 줄지만, 분할 후에는 `file://`로 직접 열 수 없어 로컬 서버가 필요해집니다.

---

## 11. 운영 정보

| 항목 | 값 |
|---|---|
| 접속 주소 | https://dongsee97-ctrl.github.io/meal/ |
| 저장소 | `dongsee97-ctrl/meal` (Public, GitHub Pages, main 브랜치) |
| 데이터 | Firebase Firestore — `food-auto` (asia-northeast3) |
| 기준 인원 | 영업부 12명 / 용역부 5명 |
| 목표 1인 식재료비 | 7,000원 |

**배포**: `index.html`(과 바뀌었다면 `pretendard.woff2`)을 main에 올리면 1~2분 뒤 반영됩니다.

**주의**: 지금은 개발용 Firebase가 따로 없습니다. 테스트로 넣은 메뉴가 **직원들 화면에
바로 뜹니다.** 실제 데이터를 건드리는 테스트는 하지 마세요.

---

## 12. 미해결 — 손대기 전 확인이 필요한 데이터

| 항목 | 내용 |
|---|---|
| 닭갈비 양념육 | 같은 계열 상품의 정확히 10배 — 0 하나 더 들어간 오타 의심 |
| "돼지고기" ↔ "제육볶음 양념육" | 이름이 달라 자동 연결 안 됨. 동일 상품인지 확인 필요 |
| 참치마요 | 캔참치가 1인 `0.9 kg`로 등록됨 |
| 원가 0원 메뉴 12개 | 재료 연결 필요 |

**임의로 고치지 말고 운영자에게 물어보세요.**
