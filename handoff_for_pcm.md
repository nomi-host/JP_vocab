# pcm(사내 배포용) 이식 가이드 — 한자 부수별 그리드 기능

> 이 문서는 **개인용 원본 앱**(`JP_vocab`)에서 새로 만든/정리한 기능을, `JP_vocab_pcm`
> (사내 배포용, 이하 "pcm")으로 옮길 때 참고하는 핸드오프입니다. 방향은
> `handoff_for_original.md`(pcm → 원본)와 반대입니다.
>
> 기준 커밋: `6cdd9db` (원본 `main`, 2026-07-02).
>
> **먼저 확인한 것**: 이번 세션에서 원본 앱에 적용한 수정사항 대부분(모달 스크롤 프리징,
> 스와이프-다운 닫기, 학습카드 구분선 고정, 단어장/학습대기 칩 정렬, 예문 후리가나)을
> pcm 코드에서 직접 대조해봤는데, **pcm은 이미 더 개선된/문제없는 버전을 갖고 있습니다**
> (아래 "포팅 불필요 확인" 참고). 그래서 실제로 새로 옮겨야 하는 건
> **한자 부수별(같은 부수 한자) 그리드 기능 하나**뿐입니다 — pcm에는 이 기능 자체가 없습니다
> (`KANJI` 데이터에 `radical`/`radicalKr` 필드는 이미 있음 — 3,009개 중 2,879개, 기능만 없음).

---

## 1. 한자 부수별(같은 부수 한자) 그리드 — pcm에 없는 기능, 신규 추가 필요

### 무엇을 하는 기능인가
`DetailModal`의 2단계(한자 상세) 화면에서, 그 한자와 **부수가 같은 다른 한자들**을
그리드로 보여줍니다. 각 칸에는 한자 글자 + 뜻·음(예: "있을 재")이 같이 나오고, 누르면
그 한자의 상세로 바로 이동합니다(스택에 push). pcm의 `kanji-rows`(음독/훈독/부수/획수/뜻)
바로 아래, `활용 단어` 섹션 바로 위에 넣으면 됩니다 — pcm의 `DetailModal` 구조가 원본과
거의 1:1로 같아서(`relWords`/`wordsWithKanji`/`openWord`/`kanji-section-label` 등 이름도 동일),
아래 코드를 거의 그대로 붙여넣을 수 있습니다.

### 핵심 함수 — `kanjiByRadical` (다른 유틸 함수들 근처에 추가)
```javascript
/* 부수(radical)가 같은 한자 찾기 — 한자 상세에서 '같은 부수 한자' 표시용.
   예) 踏(부수 足) → 足·跡·践・路… 처럼 같은 발(足) 부수 한자. 누르면 그 한자 상세로.
   인덱스는 한 번만 만들어 캐시. 흔한 한자(grade 낮음, 획수 적음) 순으로 정렬. */
let _radicalIndex = null;
function kanjiByRadical(radical, excludeCh, limit) {
  if (!radical) return [];
  if (!_radicalIndex) {
    _radicalIndex = {};
    for (const ch in KANJI) {
      const r = KANJI[ch] && KANJI[ch].radical;
      if (!r) continue;
      (_radicalIndex[r] = _radicalIndex[r] || []).push(ch);
    }
  }
  const list = (_radicalIndex[radical] || []).filter((ch) => ch !== excludeCh);
  list.sort((a, b) => {
    const ga = Number(KANJI[a].grade) || 99, gb = Number(KANJI[b].grade) || 99;
    if (ga !== gb) return ga - gb;
    return (Number(KANJI[a].strokes) || 99) - (Number(KANJI[b].strokes) || 99);
  });
  return limit ? list.slice(0, limit) : list;
}
```

> ⚠️ **`excludeCh`를 넘기지 마세요.** 처음엔 "현재 보고 있는 한자는 목록에서 빼자"는
> 생각으로 `kanjiByRadical(info.radical, sel, 40)`처럼 현재 선택된 한자(`sel`)를
> `excludeCh`로 넘겼는데, 그러면 **고를 때마다 배열 구성 자체가 바뀝니다**(직전 선택은
> 다시 나타나고 새로 고른 건 목록에서 사라짐) — 사용자가 보기엔 "누를 때마다 아래 배열이
> 제멋대로 바뀌는" 버그로 보입니다. `excludeCh`는 항상 `null`로 호출하고, 대신 현재
> 선택된 항목은 지우지 않고 **CSS `on` 클래스로 표시**하세요(아래 3번 참고).

### `DetailModal` 안에서 호출 (2단계 화면 렌더 직전, `info`/`relWords` 계산하는 곳 근처)
```javascript
const sameRad = (info && info.radical) ? kanjiByRadical(info.radical, null, 40) : [];
```

### 화면이 바뀔 때 모달을 맨 위로 스크롤 (필수 — 없으면 이상하게 느껴짐)
그리드에서 다른 한자를 누르면 스택이 바뀌면서(`setStack`) 새 한자 상세로 이동하는데,
스크롤 위치는 그대로라 **누른 위치(그리드가 있던 아래쪽)**가 그대로 보이고, 새 한자의
큰 글자·읽기(화면 맨 위)는 안 보여서 "눌렀는데 아무 반응 없는 것처럼" 느껴집니다.
`kanji-modal-body`에 ref를 달고, 스택이 바뀔 때마다 스크롤을 0으로 리셋하세요.

pcm의 `DetailModal`은 이미 `bodyRef`를 스와이프-닫기 기능에서 쓰고 있을 가능성이 높습니다
(같은 이름일 수 있음) — 있으면 그 ref를 그대로 재사용하고, 없으면 새로 만드세요:
```javascript
const bodyRef = useRef(null); // 이미 있으면 재사용
useEffect(() => {
  if (bodyRef.current) bodyRef.current.scrollTop = 0;
}, [stack]);
```
그리고 두 `kanji-modal-body` div(1단계 word-detail용, 2단계 kanji용) 모두에
`ref={bodyRef}`를 달아주세요.

### 렌더 — `kanji-rows` 블록 다음, `relWords` 블록 앞
```jsx
{sameRad.length > 0 && (
  <div className="kanji-related">
    <div className="kanji-section-label">같은 부수 한자 · {info.radical}{info.radicalKr ? " " + info.radicalKr : ""} ({sameRad.length})</div>
    <div className="kanji-radical-grid">
      {sameRad.map((ch) => (
        <button key={ch} className={"kanji-radical-cell" + (ch === sel ? " on" : "")} onClick={() => openKanji(ch)}>
          <span className="kanji-radical-glyph">{ch}</span>
          {KANJI[ch] && KANJI[ch].kr && <span className="kanji-radical-sound">{KANJI[ch].kr}</span>}
        </button>
      ))}
    </div>
  </div>
)}
```
`.kanji-related` 클래스는 pcm에 이미 있습니다(활용 단어 섹션에서 씀) — 그대로 재사용됩니다.

**셀에 `KANJI[ch].kr`을 쓰는 이유(한자/뜻/음 다 보이게)**: `kr` 필드는 이미 "뜻+음"이
합쳐진 한 줄입니다(예: `在` → `"있을 재"`, `介` → `"낄 개"`). `krSound`(음만, "재"/"개")
대신 `kr`을 쓰면 글자를 안 눌러도 뜻과 음을 바로 알 수 있습니다.

### CSS
```css
.kanji-radical-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(72px, 1fr)); gap: 8px; }
.kanji-radical-cell { display: flex; flex-direction: column; align-items: center; gap: 2px; cursor: pointer;
  border: 1px solid var(--line); border-radius: 10px; background: var(--surface); padding: 8px 4px; }
.kanji-radical-cell:active { background: var(--accent-soft); }
.kanji-radical-cell.on { background: var(--accent-soft); border-color: var(--accent); }
.kanji-radical-glyph { font-size: 24px; color: var(--ink); line-height: 1.1; }
.kanji-radical-sound { font-size: 11px; color: var(--muted); text-align: center; word-break: keep-all; line-height: 1.3; }
```
pcm은 한자 뱃지에 JLPT 레벨별 색상(`cat-nX`) 컨벤션이 있던데(`kanji-jlpt cat-n{N}` 등),
이 그리드 셀에는 필요 없습니다(부수가 같은 한자는 레벨이 섞여 있는 게 자연스러움) — 굳이
맞추려 하지 마세요.

### 검증 방법
1. 아무 한자 상세를 열고 "같은 부수 한자" 그리드에서 다른 칸을 2~3번 연속으로 눌러보고,
   **그리드 배열 자체(항목 구성·순서)가 바뀌지 않는지**, 방금 고른 것만 파란 테두리로
   표시되는지 확인하세요.
2. 그리드가 화면 아래쪽에 보이는 상태에서 칸을 누르면, **모달이 자동으로 맨 위로
   스크롤돼 새 한자의 큰 글자가 바로 보이는지** 확인하세요.
3. 부수 정보가 없는 한자(`info.radical`이 없는 경우, 주로 일본 고유 한자/변형자)에서는
   섹션 자체가 안 보이는 게 맞습니다(`sameRad.length > 0` 가드).

---

## 포팅 불필요 확인 (이번 세션에 원본에서 고친 것들 vs pcm 현재 상태)

원본 앱(`JP_vocab`)은 pcm에서 포크된 뒤 오래 손을 안 대서 아래 항목들이 낙후돼 있었고,
이번 세션에서 고쳤습니다. **포팅 전에 pcm 코드를 직접 대조해봤는데, pcm은 이미 아래처럼
더 나은 상태였습니다 — 그대로 두세요(원본의 수정 방식을 pcm에 덮어쓰지 마세요).**

| 항목 | 원본에서 발견한 문제 | pcm 현재 상태 |
|---|---|---|
| 모달 스와이프-다운 닫기 | 기능 자체가 없었음(원본) | 이미 있음 (`dragY`/`closing`/`onSheetTouch*` 패턴, `kanji-modal` 컴포넌트) |
| 학습카드 구분선 흔들림 | `.card-body { justify-content: center }`라 카드마다 위치가 흔들림 | 이미 `justify-content: flex-start`로 되어 있어 문제없음 |
| 단어장/학습대기 칩 정렬 | `.modal-decks` 폭이 안 늘어나고 칩이 좌측 정렬로 라벨과 어긋남 | 이미 `.modal-decks { width:100%; align-self:stretch }` + `.modal-deck-chips { justify-content:center }`로 되어 있어 문제없음 |
| 단어상세 예문 후리가나 | `wd-ex-ja`가 후리가나 없이 텍스트만 표시 | 이미 `<FuriganaText tokens={ex.tokens} showFuri={true} />` 사용 중 |

**참고(확실치 않아 "수정 필요"로 못 올리는 것)**: 모달(Portal)이 열려 있을 때 그 안에서
스크롤하면 터치가 React 트리를 타고 `.scroll-area`의 pull-to-refresh 핸들러까지 버블링될
수 있는 구조적 여지가 pcm에도 있습니다(`usePullToRefresh`가 원본과 동일한 형태). 다만
pcm의 `DetailModal`은 시트를 드래그하는 동안에만 `e.stopPropagation()`을 호출해서 실사용
중 문제가 보고된 적이 없는 것 같습니다(이미 실기기 QA를 여러 번 거친 상태고, 관련 커밋들이
보임). **명확한 재현 버그가 아니라면 손대지 마세요** — 원본에서 쓴 "모달 열림 카운터로
pull-to-refresh를 통째로 끄는" 방식은 원본엔 잘 맞았지만, pcm은 이미 다른(더 정교한) 방식으로
잘 동작하고 있어서 굳이 바꾸면 오히려 회귀 위험이 있습니다. 혹시 나중에 "모달에서 스크롤이
가끔 멈춘다"는 리포트가 pcm에서도 나오면, 원본의 `openModalCount` 카운터 패턴(모달이 열려
있는 동안만 `usePullToRefresh`/카드 스와이프 채점을 꺼두는 방식)을 참고하세요.

---

## 공통 주의사항
1. **버전 표기**: pcm 컨벤션대로 `index.html`의 `APP_VERSION`/`APP_BUILD`를 갱신하세요.
2. **JSX/문법 검증**: 빌드 단계가 없으니 `@babel/core`의 `transformSync`로
   `<script type="text/babel">` 블록만 떼어 컴파일 검증하는 걸 추천합니다.
3. **이 문서를 쓴 환경에서 못한 검증**: 실제 브라우저 렌더(그리드 클릭 시 스크롤 애니메이션
   느낌, 실기기 터치)는 원본 앱에서 Playwright 터치 시뮬레이션으로만 확인했습니다. pcm에
   적용 후 실기기에서 한 번 확인해 주세요.

## 참고 — 원본(개인용) 커밋 위치
| 기능 | 커밋 |
|---|---|
| 한자 부수별 그리드 배열 고정 + 뜻/음 표시 + 상단 스크롤 | `6cdd9db` |
| (참고) 모달 스크롤 프리징/스와이프닫기/구분선 고정 — pcm엔 불필요 | `8570f5e` |

저장소: `nomi-host/JP_vocab`, `main` 브랜치에 병합됨.
`git show <커밋>`으로 정확한 diff를 확인할 수 있습니다.
