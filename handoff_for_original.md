# 개인용(원본) 앱 이식 가이드

> 이 문서는 `JP_vocab_pcm`(사내 배포용) `main` 브랜치에서 작업한 기능들을, 별도로 관리 중인
> **개인용 원본 앱**에 옮길 때 참고하는 핸드오프입니다.
> 두 앱이 같은 `index.html` 단일 React 파일 구조(React 18 + Babel Standalone, 빌드 없음)라는
> 전제로 작성했습니다. 구조가 다르면 함수/CSS 클래스명을 기준으로 대응되는 위치를 찾아 적용하세요.
>
> 기준 커밋: `8de28af` (v0.2.8 + Wayne QA, 2026-07-01) — **1~8번 섹션은 f13ea73(v0.2.2) 기준으로
> 작성된 이전 내용 그대로이며, 9~11번 섹션이 그 뒤 갱신분입니다.**
> **12번 섹션(모달 딤 그라데이션 + 팝업/모달 위치)은 v1.1.0(2026-07-02) 기준 추가분입니다.**

---

## 1. 학습 대기목록 (priority 우선 학습)

### 무엇을 하는 기능인가
단어 상세에서 "학습 대기목록에 넣기"를 누르면, 그 단어가 **다음 학습 세션에서 우선적으로** 나옵니다.
복습 대상이든 신규 단어든 상관없이 priority가 켜진 단어가 먼저 채워집니다.

### 데이터 모델 변경
카드 객체에 필드 1개 추가. `initCard(w)` 함수에서:
```javascript
priority: false,         // '학습 대기목록' — 오늘 학습에서 우선 노출
```
기존 저장된 카드는 `priority` 필드가 없을 텐데, `!!c.priority`처럼 falsy 체크로 쓰면 안전합니다
(굳이 마이그레이션 코드 안 짜도 됨).

### 학습 큐 로직 변경 — `planToday(cards, total, level)`
복습 정렬에 priority를 1순위 키로 추가:
```javascript
const due = cards
  .filter((c) => !c.suspended && c.introducedOn && isDue(c, now))
  .sort((a, b) => {
    const pa = a.priority ? 0 : 1, pb = b.priority ? 0 : 1;
    if (pa !== pb) return pa - pb;
    return String(a.due || "").localeCompare(String(b.due || ""));
  });
```
신규 단어는 priority인 것부터 채우고, 남는 자리를 기존 로직(레벨 비율 또는 무작위)으로 채움:
```javascript
const prioFresh = seededShuffle(freshPool.filter((c) => c.priority), seed).slice(0, newCount);
const remainCount = newCount - prioFresh.length;
const restPool = freshPool.filter((c) => !c.priority);
const restFresh = ratio
  ? sampleByRatio(restPool, ratio, remainCount, seed)
  : seededShuffle(restPool, seed).slice(0, remainCount);
const fresh = [...prioFresh, ...restFresh];
```

### 토글 함수 — `ListView` 내부
```javascript
const togglePriority = (id) => {
  setCards((prev) => prev.map((c) => (c.id === id ? { ...c, priority: !c.priority } : c)));
};
```
`WordDetailModal`/`DetailModal`에 `onTogglePriority={togglePriority}`로 prop 전달.

### UI — 두 군데
**① `WordDetailModal`** (목록 탭에서 단어 누르면 뜨는 모달) — "단어장에 담기" 영역 안에 칩으로:
```jsx
{onTogglePriority && (
  <>
    <div className="modal-decks-label">학습 대기목록</div>
    <div className="modal-deck-chips">
      <button
        className={"deck-chip deck-chip-prio" + (card.priority ? " on" : "")}
        onClick={() => onTogglePriority(card.id)}>
        {card.priority ? "✓ 담김" : "＋ 학습 대기목록에 넣기"}
      </button>
    </div>
  </>
)}
```
> ⚠️ 처음엔 `modal-prio-btn`이라는 별도 큰 버튼+안내문구로 만들었다가, 사용자 피드백으로
> **"새 단어장"과 같은 칩 크기로 통일 + 안내문구 제거**로 바꿨습니다. 새로 만들 때는 처음부터 칩 형태로 가세요.

**② `DetailModal`** (한자/연관단어 상세, 흘려듣기 상세 등에서 공용으로 쓰는 모달) — 액션 버튼 행에:
```jsx
{onTogglePriority && (
  <button className={"wd-act-btn wd-act-prio" + (cur.priority ? " on" : "")}
    onClick={() => onTogglePriority(cur.id)}>
    <span>{cur.priority ? "✓ 학습 대기" : "＋ 학습 대기"}</span>
  </button>
)}
```

### CSS — on 상태는 반드시 연한 노랑
```css
/* 학습 대기목록 칩: on일 때 연한 노랑(진노랑 X) */
.deck-chip-prio.on { background: var(--accent-30); color: var(--on-accent); border-color: var(--accent-60); }
.wd-act-prio.on { background: var(--accent-30); color: var(--on-accent); border-color: var(--accent-60); }
```
> 즐겨찾기(starred) on 색상은 기존 진노랑(`--accent`) 그대로 두고, 학습대기만 연한 노랑으로
> **구분되게** 한 것이 의도입니다. 두 액션의 색을 같게 만들지 마세요(헷갈림 방지).

---

## 2. 연관단어 이동 시 액션 (한자 상세 → 활용단어 클릭)

### 무엇을 하는 기능인가
단어 상세 → 한자 카드를 누르면 그 한자의 음훈독·활용단어 목록이 뜨는데(`DetailModal`의 2단계 "한자" 화면),
거기서 활용단어를 눌러 다른 단어 상세로 이동했을 때 **그 단어에도 즐겨찾기·학습대기·단어장 액션이 보이도록** 한 것.

### 핵심 변경 — `DetailModal`에 props 추가
```javascript
function DetailModal({ card, onClose, voiceRef, allCards, cards, decks, onToggleStar, onTogglePriority, onToggleDeck, onCreateDeck }) {
```
기존엔 `card, onClose, voiceRef, allCards`만 받았습니다. 이제 `cards`(전체 카드 배열, 라이브 상태 조회용)와
액션 콜백들을 추가로 받습니다.

### 라이브 카드 해석 (중요 — 빠뜨리면 토글해도 화면에 즉시 반영 안 됨)
```javascript
const cur0 = top.card || card;              // 현재(또는 직전) 단어(스택 스냅샷)
const cur = (cards && cur0 && cur0.id) ? (cards.find((c) => c.id === cur0.id) || cur0) : cur0;
const isRealCard = !!(cards && cur0 && cur0.id && cards.some((c) => c.id === cur0.id));
const canAct = isRealCard && (onToggleStar || onTogglePriority || onToggleDeck);
```
스택(`stack`)에 쌓인 단어는 클릭 시점의 **스냅샷**이라, 즐겨찾기를 토글해도 객체 자체는 안 바뀝니다.
그래서 `cards`에서 같은 id를 다시 찾아 "최신 상태"로 렌더링해야 토글이 즉시 반영됩니다.

`isRealCard`는 — 한자 상세에서 보여주는 "활용단어"가 **내 단어장에 실제로 등록된 카드가 아닐 수도 있어서**
(SEED 전체에서 뽑아온 것일 수 있음), 그럴 땐 액션 버튼 자체를 숨기기 위한 가드입니다.

### 렌더 — 카테고리 뱃지 + 액션 버튼 + 단어장 칩
단어 상세 화면(`view === "word"`) 맨 위에 카테고리 뱃지, 단어 밑에 액션 버튼들, 그 아래 단어장 칩을 추가했습니다.
(코드는 위 "1. 학습 대기목록" 섹션의 `wd-act-btn`/`wd-decks` 부분과 동일 — 같이 작업했기 때문)

### 호출부 — props 전달 빠뜨리지 않기
`DetailModal`을 호출하는 모든 곳에 `cards`/`decks`/액션 콜백을 같이 넘겨야 합니다:
- `FlashCard`(학습 탭, 카드 누르면 뜨는 상세) — `cards={cards}` 등
- `WordDetailModal`(목록 탭) — `cards={allCards}` 등
- `ListenView`(흘려듣기, 이번에 추가 — 아래 5번 항목 참고)

---

## 3. 예문 속 단어 네비게이션 (★ 이번 세션 핵심 기능)

### 무엇을 하는 기능인가
단어 상세의 예문 아래에, **예문 문장에 실제로 등장하는 단어 중 단어장(words.js/SEED)에 있는 것**을
`[뱃지] 単語 よみがな ⋯ 뜻 ›` 형태로 나열합니다. 누르면 그 단어 상세로 이동(타고타고 탐색 가능).
현재 보고 있는 단어 자신도 (v0.2.1부터) 목록에 포함됩니다.

### 설계 시 반드시 지켜야 하는 4가지 안전장치
요구사항이 명확했던 부분이라 그대로 옮기세요. 어기면 동음이의어/활용형에서 오매칭이 납니다.

1. **표면(한자)+요미가나 동시 검증** — 工作(こうさく)/耕作(こうさく)처럼 요미가나만 같은 다른 한자
   오매칭 방지. 같은 한자를 다른 음/훈으로 읽는 경우(方=ほう/かた)도 요미가나로 구분.
2. **한자 골격(かな 제거) 정규화로 오쿠리가나 변형 호환** — 受付 ↔ 受け付け.
3. **동사/형용사는 어간 접두 일치로 활용형 인식** — 固めましょう→固める, 盗んだ→盗む, 立ち上がった→立ち上がる.
4. **접미사(～)·짧은 조사·기능어 제외**, 그리고 **exact-매칭을 동사 활용 매칭보다 먼저 시도**(중요!).
   동사 매칭을 우선하면 読み(명사, "읽기")가 読む(동사)로 과잉 축약되거나, 通り(명사)가 通る로
   잘못 축약되는 등 오매칭이 늘어남을 실데이터로 확인했습니다. exact 우선이 더 안전합니다.

### 코드 — `index.html` 상단(167번 줄 부근, `SEED_WORDS` 정의 직후)
```javascript
function isKanaChar(ch) { const o = ch.codePointAt(0); return (o >= 0x3040 && o <= 0x30FF) || ch === "ー"; }
function kanjiSkeleton(s) { let o = ""; for (const ch of (s || "")) if (!isKanaChar(ch)) o += ch; return o; }
function hasAnyKanji(s) { for (const ch of (s || "")) if (isKanjiChar(ch)) return true; return false; }
const KANA_STOPWORDS = new Set([
  "する","ある","いる","なる","れる","られる","せる","させる","くる","くれる","いく","しまう",
  "こと","もの","ため","よう","そう","とき","ところ","これ","それ","あれ","どれ","ない","いう",
  "ここ","そこ","あそこ","この","その","あの","どの","から","まで","ので","のに","では","には",
  "です","ます","でも","だけ","ほど","くらい","ぐらい","など","また","もう","とても","ちょっと",
  "わたし","あなた","かれ","かのじょ","みんな","じぶん","とき","つもり",
]);
let _seedIdxBuilt = false;
const _seedExact = new Map();        // "표면|요미" -> entry
const _seedSkel = new Map();         // "한자골격|요미" -> entry (변형 호환)
const _seedVerbsByHead = new Map();  // 표면 첫 글자 -> [{sStem,rStem,entry}] (활용 매칭)
function buildSeedIdx() {
  if (_seedIdxBuilt) return; _seedIdxBuilt = true;
  for (const w of SEED) {
    if (!w || !w.word) continue;
    const surf = w.word, read = w.reading || w.word;
    if (/[～〜~]/.test(surf)) continue;        // 접두/접미사 제외
    const ek = surf + "|" + read; if (!_seedExact.has(ek)) _seedExact.set(ek, w);
    const skel = kanjiSkeleton(surf);
    if (skel && skel !== surf) { const sk = skel + "|" + read; if (!_seedSkel.has(sk)) _seedSkel.set(sk, w); }
    const last = surf[surf.length - 1];
    if (hasAnyKanji(surf) && isKanaChar(last) && read.length >= 3) {
      const sStem = surf.slice(0, -1), rStem = read.slice(0, -1);
      if (rStem.length >= 2 && hasAnyKanji(sStem)) {
        const head = sStem[0];
        if (!_seedVerbsByHead.has(head)) _seedVerbsByHead.set(head, []);
        _seedVerbsByHead.get(head).push({ sStem, rStem, entry: w });
      }
    }
  }
}
function extractExampleWords(card) {
  buildSeedIdx();
  const ex = getExample(card);
  const toks = (ex && ex.tokens) || [];
  if (!toks.length) return [];
  const sSurf = (a, b) => toks.slice(a, b).map((t) => t.t).join("");
  const sRead = (a, b) => toks.slice(a, b).map((t) => (t.r != null ? t.r : t.t)).join("");
  const out = [], seen = new Set();
  const push = (e) => {
    if (!e || seen.has(e.word)) return;
    seen.add(e.word); out.push(e);
  };
  let i = 0;
  while (i < toks.length) {
    let adv = 0;
    // 1) 정확/변형 매칭 (1~3 토큰, 긴 것 우선) — 동사 활용 매칭보다 먼저!
    for (let span = Math.min(3, toks.length - i); span >= 1 && !adv; span--) {
      const surf = sSurf(i, i + span); if (!surf) continue;
      const read = sRead(i, i + span);
      if (!hasAnyKanji(surf) && (surf.length < 3 || KANA_STOPWORDS.has(surf))) continue;
      const e = _seedExact.get(surf + "|" + read) || _seedSkel.get(kanjiSkeleton(surf) + "|" + read);
      if (e) { push(e); adv = span; }
    }
    if (adv) { i += adv; continue; }
    // 2) 동사/형용사 활용 매칭 (1~2 토큰, 어간 접두 일치)
    for (let span = Math.min(2, toks.length - i); span >= 1 && !adv; span--) {
      const surf = sSurf(i, i + span); if (!hasAnyKanji(surf)) continue;
      const read = sRead(i, i + span);
      const cands = _seedVerbsByHead.get(surf[0]) || [];
      for (const v of cands) {
        if (surf.startsWith(v.sStem) && read.startsWith(v.rStem) && read.length >= v.rStem.length) {
          const tail = surf.slice(v.sStem.length);
          if (tail.length === 0 || isKanaChar(tail[0])) { push(v.entry); adv = span; break; }
        }
      }
    }
    if (adv) { i += adv; continue; }
    i += 1;
  }
  return out;
}
```
**전제조건**: 예문 토큰(`card.ex.tokens`)이 `{t: "표면", r: "요미가나"|null}` 형태의 단어 단위 배열이어야
합니다. words.js의 모든 카드가 이미 이 형태(`tokenizeJa`로 생성됨)라 별도 변환 불필요.

### 렌더 — `DetailModal`의 "word" 화면, 예문 블록 바로 아래
```javascript
const exWords = useMemo(() => (ex && ex.ja) ? extractExampleWords(cur) : [], [cur.id]);
```
```jsx
{(() => {
  const ews = exWords;
  if (!ews.length) return null;
  const pool = cards || allCards || [];
  return (
    <div className="wd-exwords">
      <div className="kanji-section-label">예문 속 단어</div>
      {ews.map((e, i) => {
        const live = pool.find((c) => c.word === e.word && (c.reading || "") === (e.reading || "")) || e;
        return (
          <button key={i} className="exw-row" onClick={() => openWord(live)}>
            <CategoryBadge category={cardCategory(e)} />
            <span className="exw-main">
              <span className="exw-word">{e.word}</span>
              {e.reading && e.reading !== e.word && <span className="exw-read">{e.reading}</span>}
            </span>
            <span className="exw-mean">{e.meaning}</span>
            <span className="exw-arrow"><Icon.ChevR /></span>
          </button>
        );
      })}
    </div>
  );
})()}
```
`openWord`는 `DetailModal` 내부에 이미 있는 스택 push 함수(`(w) => setStack((s) => [...s, { kind: "word", card: w.card || w }])`)를 그대로 재사용합니다.

### CSS
```css
.wd-exwords { margin-top: 22px; padding-top: 20px; border-top: 1px solid var(--line); }
.exw-row { display: flex; align-items: center; gap: 9px; width: 100%; text-align: left;
  border: none; background: var(--bg); border-radius: 12px; padding: 11px 12px; cursor: pointer; margin-top: 8px; }
.exw-row:active { background: var(--accent-soft); }
.exw-main { display: flex; align-items: baseline; gap: 7px; flex-shrink: 0; min-width: 0; }
.exw-word { font-size: 17px; font-weight: 700; color: var(--ink);
  font-family: -apple-system, BlinkMacSystemFont, "Hiragino Kaku Gothic ProN", "Noto Sans JP", sans-serif; }
.exw-read { font-size: 12px; color: var(--muted); white-space: nowrap; }
.exw-mean { flex: 1; min-width: 0; font-size: 13px; color: var(--ink-2);
  overflow: hidden; text-overflow: ellipsis; white-space: nowrap; text-align: right; }
.exw-arrow { flex-shrink: 0; color: var(--muted); display: flex; }
```
> ⚠️ `.exw-row` 배경은 처음 `var(--fill)`(다른 버튼들과 같은 회색)로 했다가, 너무 튀어서
> `var(--bg)`(앱 전체 배경색, 더 연함)로 바꿨습니다. 처음부터 `--bg`로 가세요.

### 검증 방법 (이식 후 꼭 재검증할 것)
words.js 데이터가 다른 버전이라면, Node로 빠르게 샘플 검증하는 게 안전합니다:
```bash
node -e '
const fs=require("fs"); const window={};
eval(fs.readFileSync("words.js","utf8"));
// extractExampleWords 등 위 함수들을 동일하게 정의한 뒤
const SEED=window.SEED;
let shown=0;
for (const c of SEED) {
  if (shown>=30) break;
  const ex=c.ex; if (!ex||!ex.ja||ex.ja.length<12) continue;
  const ws=extractExampleWords(c); if (ws.length<2) continue; shown++;
  console.log("["+c.word+"] "+ex.ja+" → "+ws.map(w=>w.word).join(" / "));
}'
```
30~40개를 눈으로 훑어 이상한 매칭(동사 과잉축약, 동음이의어 오매칭)이 없는지 확인하세요.

---

## 4. TTS 보이스 고정 + 터치 발음 지연 단축

### 4-1. 보이스 고정 — `api/proxy.js`
Chirp3-HD 기본 화자 8개(여 Aoede/Kore/Leda/Zephyr, 남 Charon/Fenrir/Orus/Puck) 중
**일본어가 가장 자연스럽다고 평가되는 여 Aoede · 남 Charon 2개로 고정**, random도 이 둘 중에서만 뽑히게:
```javascript
// 일본어가 가장 자연스러운 남/녀 보이스 1개씩으로 고정. random도 이 둘 중에서만 선택.
const LANG = { ja: { code: "ja-JP", f: "Aoede", m: "Charon" }, ko: { code: "ko-KR", f: "Aoede", m: "Charon" } };
function pickVoice(lang, gender, name) {
  const L = LANG[lang] || LANG.ja, prefix = L.code + "-Chirp3-HD-";
  if (name && /^[A-Za-z]+$/.test(name)) return prefix + name;
  if (gender === "male") return prefix + L.m;
  if (gender === "female") return prefix + L.f;
  return prefix + (Math.random() < 0.5 ? L.f : L.m); // random: 지정된 남/녀 중에서만
}
```
> 이 환경에서는 음성을 직접 들어볼 방법이 없어 **"널리 자연스럽다고 평가되는" 화자를 골랐을 뿐**입니다.
> 개인 앱에 적용하기 전에 직접 들어보고, 마음에 안 들면 `f`/`m` 값만 다른 화자명(Kore, Puck, Orus 등)으로
> 바꾸면 됩니다 — 코드 구조 변경 불필요.

### 4-2. 터치 발음 지연 단축 — `index.html`
**(a) 진행 중 요청 중복 방지.** `prefetchAudio`(다음 카드 미리 불러오기)와 사용자가 탭한 즉시 호출이
동시에 겹치면 API를 2번 부르게 되는데, 이를 막기 위해 in-flight 캐시를 추가:
```javascript
const ttsInflight = new Map(); // key -> Promise

async function fetchGoogleAudio(text, s, lang) {
  const key = cacheKey(text, s, lang);
  if (ttsCache.has(key)) return ttsCache.get(key);
  if (ttsInflight.has(key)) return ttsInflight.get(key); // 진행 중 요청 재사용
  const p = (async () => {
    try {
      const cached = await idbGet("clip|" + key);
      if (cached) { const u = URL.createObjectURL(cached); ttsCache.set(key, u); return u; }
    } catch (_) {}
    const data = await apiPost({ type: "tts", text, gender: s.gender, rate: s.rate, lang: lang || "ja" });
    if (!data || !data.audio) throw new Error("no audio");
    const bin = atob(data.audio);
    const bytes = new Uint8Array(bin.length);
    for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
    const blob = new Blob([bytes], { type: "audio/mpeg" });
    try { idbSet("clip|" + key, blob); } catch (_) {}
    const url = URL.createObjectURL(blob);
    ttsCache.set(key, url);
    return url;
  })();
  ttsInflight.set(key, p);
  try { return await p; } finally { ttsInflight.delete(key); }
}
```
**(b) random도 미리 불러오기 허용.** 기존엔 "랜덤은 매번 목소리가 달라서 캐시 효과 없음"이라며
prefetch를 건너뛰었는데, 이제 보이스가 **여/남 2개로 고정**됐으니 prefetch 캐시가 그대로 적중합니다:
```javascript
function prefetchAudio(text, kind) {
  if (!text) return;
  if (ttsSettings.engine !== "google") return;
  const s = effSettings(kind);
  const key = cacheKey(text, s);
  if (ttsCache.has(key) || ttsInflight.has(key)) return;
  fetchGoogleAudio(text, s).catch(() => {});
}
```
**(c) 현재 카드도 워밍.** 기존엔 "다음 카드"만 미리 불러왔는데, **첫 단어를 보자마자 탭해도 즉시 재생**되도록
현재 카드도 함께 워밍하게 `StudyView`를 수정:
```javascript
useEffect(() => {
  const warm = (c) => {
    if (!c) return;
    prefetchAudio(wordSpeech(c), "word");
    const ex = getExample(c);
    if (ex && ex.tokens && ex.tokens.length) prefetchAudio(exampleSpeech(ex), "example");
  };
  warm(cardsRef.current.find((c) => c.id === currentId));
  warm(cardsRef.current.find((c) => c.id === s.queue[s.pos + 1]));
}, [currentId, s.pos]);
```
> IndexedDB 영구 캐시(`idbGet`/`idbSet`)는 그대로 — 같은 단어는 다음부터 네트워크 호출 없이 즉시 재생됩니다.
> 더 줄이고 싶다면, 학습 세션 시작 시 큐 전체를 한 번에 백그라운드 워밍하는 것도 고려할 수 있지만
> (동시 요청 폭주 → 레이트리밋 걸릴 수 있음) 이번엔 현재+다음 2개만으로 충분히 체감 개선됐습니다.

---

## 5. 흘려듣기(연속듣기) 단어 목록 눌러 상세보기

### 무엇을 하는 기능인가
듣기 탭 → 백그라운드 흘려듣기 재생 중에 화면에 보이는 "재생 중인 30단어" 목록(`bg-list`)의
각 항목을 눌러 단어 상세(`DetailModal`)를 열 수 있게 했습니다.

### 끊김 없이 동작하는 이유 (이식할 때 꼭 이해하고 가세요)
흘려듣기 오디오는 **모듈 전역 싱글톤** `sharedAudio`(React 상태가 아님, `let sharedAudio = null;`로 파일
최상단에 선언)로 재생됩니다. 모달은 `ReactDOM.createPortal`로 오버레이만 띄우는 것이라 `ListenView`
컴포넌트 자체를 언마운트하지 않습니다. 그리고 모달이 열릴 때 `stopAudio()`를 호출하는 코드가 없습니다
(`stopAudio()`는 오직 사용자가 모달 안에서 **발음 버튼을 직접 눌렀을 때만** 호출됨). 따라서:
- ✅ 목록 단어 클릭 → 상세 열림 → 흘려듣기 계속 재생
- ⚠️ 상세 안에서 그 단어 발음 버튼을 또 누르면 → 새 소리 재생되며 흘려듣기 끊김(의도된 동작 — 다른 화면에서도 동일)

**다른 구조의 앱에 이식할 때 체크할 것**: 만약 그쪽 앱의 흘려듣기가 `useState`/`useRef`로 관리되는
오디오 엘리먼트이고 `ListenView`(또는 동급 컴포넌트) 안에서만 살아있는 구조라면, 모달을 그 컴포넌트
밖(예: 상위 `App`)에서 portal로 띄우거나, 오디오 재생 상태를 전역으로 끌어올려야 끊김이 안 생깁니다.

### 코드 변경
**상태 추가** (`ListenView` 컴포넌트 내부):
```javascript
const [bgDetailCard, setBgDetailCard] = useState(null); // 흘려듣기 중 살펴보는 단어(상세 모달)
```
**목록 항목을 버튼으로 변경**:
```jsx
{bgList.map((c, i) => (
  <button key={c.id} className="bg-list-item" onClick={() => setBgDetailCard(c)}>
    <div className="bg-list-head">
      <span className="bg-list-num">{i + 1}</span>
      <span className="bg-list-word">{c.word}</span>
      {c.hasKanji && c.reading && <span className="bg-list-reading">{c.reading}</span>}
    </div>
    <div className="bg-list-mean">{c.meaning}</div>
    {c.ex && (c.ex.ja || (c.ex.tokens && c.ex.tokens.length > 0)) && (
      <div className="bg-list-ex">
        <div className="bg-list-ex-ja">
          {c.ex.tokens && c.ex.tokens.length > 0
            ? <FuriganaText tokens={c.ex.tokens} showFuri={true} />
            : c.ex.ja}
        </div>
        {c.ex.ko && <div className="bg-list-ex-ko">{c.ex.ko}</div>}
      </div>
    )}
  </button>
))}
```
**모달 렌더** (목록 바로 아래) — 처음엔 액션 없이 만들었다가, **즐겨찾기·학습대기·단어장 추가까지
필요하다는 피드백을 받아 추가**했습니다. 최종 버전:
```jsx
{/* 단어 상세 — 흘려듣기 오디오는 모듈 전역(sharedAudio)이라 모달을 열어도 끊기지 않는다.
    단, 모달 안에서 발음 버튼을 직접 누르면 그 소리가 새로 재생되며 흘려듣기는 끊긴다(의도된 동작). */}
{bgDetailCard && (
  <DetailModal card={bgDetailCard} allCards={cards} cards={cards} decks={decks}
    onToggleStar={toggleStar} onTogglePriority={togglePriority}
    onToggleDeck={toggleCardDeck} onCreateDeck={createDeck}
    voiceRef={voiceRef} onClose={() => setBgDetailCard(null)} />
)}
```

### 즐겨찾기·학습대기·단어장 추가 액션을 끌어오기
`ListenView`는 원래 `setCards`/`setDecks`를 받지 않는(읽기 전용) 컴포넌트였습니다. 액션을 쓰려면
`App`에서 두 setter를 내려주고, `ListView`에 있던 토글 함수와 **동일한 로직을 `ListenView` 안에 그대로
복제**했습니다(공용 훅으로 뽑아도 되지만, 범위를 작게 유지하려고 로컬 복제를 택함).

**`App`의 호출부**:
```jsx
{tab === "listen" && <ListenView cards={cards} setCards={setCards} voiceRef={voiceRef} decks={decks} setDecks={setDecks} />}
```
**`ListenView` 함수 시그니처 + 로컬 토글 함수**:
```javascript
function ListenView({ cards, setCards, voiceRef, decks, setDecks }) {
  // ...
  const toggleStar = (id) => {
    setCards((prev) => prev.map((c) => (c.id === id ? { ...c, starred: !c.starred } : c)));
  };
  const togglePriority = (id) => {
    setCards((prev) => prev.map((c) => (c.id === id ? { ...c, priority: !c.priority } : c)));
  };
  const toggleCardDeck = (cardId, deckId) => {
    setCards((prev) => prev.map((c) => {
      if (c.id !== cardId) return c;
      const cur = c.decks || [];
      return { ...c, decks: cur.includes(deckId) ? cur.filter((x) => x !== deckId) : [...cur, deckId] };
    }));
  };
  const createDeck = (name) => {
    const nm = (name || "").trim();
    if (!nm) return null;
    const id = "deck_" + uid();
    setDecks((prev) => [...prev, { id, name: nm }]);
    return id;
  };
```
`DetailModal`은 이미 `onToggleStar`/`onTogglePriority`/`onToggleDeck`/`onCreateDeck`가 있을 때만
관련 버튼을 렌더링하도록 가드돼 있어서(`canAct = isRealCard && (onToggleStar || onTogglePriority || onToggleDeck)`),
이 4개 prop만 넘기면 별도 UI 코드 추가 없이 즉시 버튼들이 나타납니다(1번/2번 섹션 참고).

### CSS — `<div>`였던 걸 `<button>`으로 바꿨으니 리셋 필요
```css
.bg-list-item { display: block; width: 100%; text-align: left; border: none; font: inherit; cursor: pointer;
  border-radius: 12px; padding: 10px 12px; background: var(--surface); box-shadow: var(--card-shadow); }
.bg-list-item:active { background: var(--accent-soft); }
```
(`bg-list-head`/`bg-list-word`/`bg-list-mean`/`bg-list-ex` 등 내부 스타일은 기존 그대로 재사용)

---

## 6. 흘려듣기 30단어 선정 — "하루 고정" → "범위 고정" 랜덤으로 변경

### 무엇이 문제였나
기존엔 흘려듣기 30단어를 **하루 단위 고정 시드**로 뽑았습니다:
```javascript
// 기존 코드(변경 전)
const buildBgList = (seed) => {
  const useSeed = (seed != null) ? seed : Math.floor(Date.now() / 86400000); // 날짜 기반 고정 시드
  return seededShuffle(poolForScope(), useSeed).slice(0, BG_COUNT);
};
```
의도는 "같은 날·같은 범위면 같은 30단어 → 오디오를 다시 굽지 않고 캐시 재사용 → 빠른 재생"이었는데,
부작용으로 **같은 날에는 프리뷰 버전을 몇 번을 새로 배포해도, 카테고리를 선택하고 "생성"해도 항상
똑같은 30단어**가 나와서 "랜덤이 맞나?" 하는 혼란을 줬습니다.

### 어떻게 바꿨나
기준을 "날짜"에서 **"범위(소스+카테고리)"** 로 바꿨습니다 — 범위가 같으면 직전 목록을 재사용(캐시 유지),
**범위를 바꾸면 새로 무작위** 30개를 뽑습니다. "새로운 단어 듣기" 버튼은 기존처럼 범위 무관하게
항상 강제 무작위입니다.

**캐시 저장소** (`LISTEN_SESSION_KEY` 근처에 추가):
```javascript
// 흘려듣기 30단어 목록 캐시: 범위(scope)가 같으면 같은 목록을 재사용(오디오 재굽기 방지),
// 범위가 바뀌면 새로 무작위 선정. 날짜는 더 이상 기준으로 쓰지 않는다(범위 기준으로 변경).
const BG_LIST_KEY = "jp-vocab-bglist-v1";
function loadBgListCache() {
  try { const raw = localStorage.getItem(BG_LIST_KEY); return raw ? JSON.parse(raw) : null; } catch { return null; }
}
function saveBgListCache(scopeKey, ids) {
  try { localStorage.setItem(BG_LIST_KEY, JSON.stringify({ scopeKey, ids })); } catch {}
}
```
**`buildBgList` 교체**:
```javascript
// 백그라운드 묶음 단어 고르기: 범위(소스+카테고리)가 같으면 직전 목록을 재사용(오디오 재굽기 방지),
// 범위를 바꾸거나 forceFresh(새로운 단어 듣기)면 새로 무작위 30개를 뽑는다.
const buildBgList = (forceFresh) => {
  const pool = poolForScope();
  const scopeKey = cfg.src + "|" + (cfg.cat || "");
  if (!forceFresh) {
    const cached = loadBgListCache();
    if (cached && cached.scopeKey === scopeKey && cached.ids && cached.ids.length) {
      const byId = new Map(pool.map((c) => [c.id, c]));
      const reused = cached.ids.map((id) => byId.get(id)).filter(Boolean);
      if (reused.length > 0) return reused; // 일부 단어가 삭제/정지돼도 남은 만큼은 그대로 재사용
    }
  }
  const list = seededShuffle(pool, Date.now() & 0x7fffffff).slice(0, BG_COUNT);
  saveBgListCache(scopeKey, list.map((c) => c.id));
  return list;
};
```
**호출부 변경** (`bgStart` 안):
```javascript
// 기존: const list = buildBgList(fresh ? (Date.now() & 0x7fffffff) : undefined);
const list = buildBgList(!!fresh);
```
`fresh`는 "새로운 단어 듣기" 버튼에서만 `true`로 넘어옵니다(기존과 동일).

### 주의할 점
- `bgKeyFor(list)`(오디오 IDB 캐시 키)는 **목록의 정확한 id 배열**로 만들어지므로, 범위가 같아 같은
  id들이 재사용되면 자동으로 오디오 캐시도 적중합니다. `buildBgList` 쪽만 고치면 되고 오디오 베이킹
  로직은 손댈 필요 없습니다.
- 캐시된 id 중 일부가 그 사이 삭제/정지(suspend)됐다면 `byId.get(id)`가 `undefined`라 자동으로
  걸러집니다(`filter(Boolean)`). 남은 단어가 1개라도 있으면 그대로 재사용 — 너무 많이 빠지면 직접
  "새로운 단어 듣기"로 새로 뽑게 하면 됩니다(자동 임계치 없음, 단순하게 유지).

---

## 7. 듣기 "화면 보면서 공부하기"에서도 단어 눌러 상세보기

### 발견한 버그
섹션 5에서 `bgDetailCard`/`setBgDetailCard`와 `DetailModal` 렌더 블록을 추가했는데, 그 렌더 블록을
**`{cfg.mode === "study" ? (...) : (...)}` 삼항연산자의 bg(배경) 쪽 분기 안에만** 넣어버린 실수가 있었습니다.
즉 상태(`bgDetailCard`)는 어느 모드에서든 똑같이 set 되지만, 그걸 그려주는 `<DetailModal>`은 **bg 모드일
때만 화면에 존재** — "화면 보면서 공부하기"(`cfg.mode === "study"`) 모드에서는 단어를 눌러 상태를 바꿔도
모달이 아예 렌더되지 않아 화면에 아무 일도 안 일어나는 것처럼 보였습니다.

**교훈**: 모달처럼 두 분기(study/bg) 모두에서 공유해야 하는 상태/렌더는, 분기 안에 두지 말고
**삼항연산자 바깥, 컴포넌트 return의 최상위 레벨**에 둬야 합니다.

### 고친 위치
`ListenView`의 return 안에서, `{cfg.mode === "study" ? (<>...</>) : (<>...</>)}` 전체가 끝난 직후,
최상위 컨테이너 `</div>` 바로 앞에 모달을 둡니다:
```jsx
{cfg.mode === "study" ? (
  <> {/* 화면 보면서 공부하기 */} </>
) : (
  <> {/* 백그라운드 음성만 */} </>
)}

{/* 단어 상세 — study/bg 모드 둘 다에서 단어를 누르면 뜬다. */}
{bgDetailCard && (
  <DetailModal card={bgDetailCard} allCards={cards} cards={cards} decks={decks}
    onToggleStar={toggleStar} onTogglePriority={togglePriority}
    onToggleDeck={toggleCardDeck} onCreateDeck={createDeck}
    voiceRef={voiceRef} onClose={() => setBgDetailCard(null)} />
)}
```

### "화면 보면서 공부하기" 쪽 단어 영역을 클릭 가능하게
기존엔 `.listen-word`/`.listen-reading`/`.listen-meaning`/`.listen-ex`가 그냥 `<div>`들의 나열이라
탭해도 아무 반응이 없었습니다. 이 묶음 전체를 버튼 하나로 감쌌습니다:
```jsx
<button className="listen-info" onClick={() => setBgDetailCard(cur)}>
  <div className="listen-word">{cur.word}</div>
  {cur.hasKanji && cur.reading && <div className="listen-reading">{cur.reading}</div>}
  <div className="listen-meaning">{cur.meaning}</div>
  {cur.ex && cur.ex.ja && (
    <div className="listen-ex">
      <div className="listen-ex-ja">{cur.ex.ja}</div>
      {cur.ex.ko && <div className="listen-ex-ko">{cur.ex.ko}</div>}
    </div>
  )}
</button>
```
phase-row(점 표시)와 progress-bar는 클릭 영역 밖에 그대로 둡니다(상태 표시일 뿐 탭 대상이 아님).

**CSS**: 원래 `.listen-now`가 `text-align:center; display:flex; flex-direction:column; align-items:center`로
내용을 가운데 정렬했는데, `<button>`으로 감싸면서 똑같은 정렬을 버튼 자체에도 줘야 기존 모양이 유지됩니다:
```css
.listen-info { display: flex; flex-direction: column; align-items: center; width: 100%;
  border: none; background: none; text-align: center; cursor: pointer; padding: 0; }
```

---

## 8. 목록 단어 상세(WordDetailModal) 최대 크기 — 탭바 위 20px

### 무엇을 하는 기능인가
목록 탭에서 단어를 눌러 뜨는 상세 모달(`WordDetailModal`)의 **최대 크기**를, 화면을 거의 꽉 채우던 것에서
**하단 탭바 위로 20px 정도 여백을 남기는 크기**로 제한했습니다. 내용이 그보다 길면 기존처럼
`.modal-body` 안에서 스크롤됩니다(내부 상세 UI는 변경 없음).

### 왜 공용 `.modal`/`.modal-overlay`를 직접 수정하지 않았나
이 두 클래스는 `WordDetailModal`뿐 아니라 `DeckManager`(`className="modal deck-mgr"`)도 같이 씁니다.
공용 클래스의 `max-height`/`padding`을 바꾸면 DeckManager 크기도 같이 바뀌어 버립니다. 그래서
**`WordDetailModal`의 두 wrapper에만 모디파이어 클래스를 추가**해서 범위를 좁혔습니다:
```jsx
<div className="modal-overlay wd-overlay-pad" onClick={onClose}>
  <div className="modal wd-modal-cap" onClick={(e) => e.stopPropagation()}>
```
다른 앱에서 `.modal`/`.modal-overlay`를 여러 모달이 공유하고 있다면, 똑같이 전용 클래스를 추가해서
범위를 좁히는 패턴을 권장합니다. 만약 `WordDetailModal`만 그 클래스를 쓰는 구조라면 굳이 모디파이어
클래스를 만들지 않고 `.modal`/`.modal-overlay`를 직접 고쳐도 됩니다.

### 수치 — 반드시 실측해서 맞출 것
탭바 높이는 폰트·아이콘 크기에 따라 달라지므로 **감으로 추측하지 말고 실제 CSS로 측정**하세요.
이 프로젝트의 탭바(아이콘 22px + 텍스트 11px + 패딩들)는 실측 **58px**이었습니다. 여기에 요청받은
여백 20px을 더해 **78px**을 하단에 예약합니다(상단은 기존 16px 그대로 유지):
```css
.wd-overlay-pad { padding: 16px 16px calc(78px + env(safe-area-inset-bottom)) 16px; }
.wd-modal-cap {
  max-height: calc(100vh - 94px - env(safe-area-inset-bottom)); /* 94 = 16(top) + 78(bottom) */
  max-height: calc(100dvh - 94px - env(safe-area-inset-bottom));
}
```
**왜 overlay의 padding과 modal의 max-height를 둘 다 바꿔야 하나**: `.modal-overlay`는
`display:flex; align-items:center`로 모달을 가운데 정렬합니다. 컨테이너의 padding을 위/아래
비대칭(16 vs 78)으로 주면, flexbox는 **그 padding을 뺀 나머지 영역 안에서** 가운데 정렬하므로
— 내용이 짧을 땐 평범하게 중앙 정렬되고, 내용이 충분히 길어서 **최대 크기에 도달했을 때만**
위 16px·아래 78px이 정확히 지켜집니다(이게 "최대 사이즈일 때 탭바 위 20px"의 의미). 그런데 `.modal`
자체의 기존 `max-height`(예: `calc(100vh - 32px)`, 대칭 16/16 기준)를 그대로 두면 그 값이 새 padding
기준보다 더 커서 모달이 새 padding 경계를 넘어 자랄 수 있습니다 — 그래서 `.wd-modal-cap`으로
정확한 max-height(94px 기준)를 같이 덮어써야 실제로 78px이 보장됩니다.

### 검증 방법 (실측 없이 숫자를 박지 말 것)
실제 CSS를 그대로 뽑아 정적 HTML로 재구성한 뒤, 내용이 충분히 긴 상태(스크롤이 필요할 정도)로 채워서
Playwright 등으로 `modal.getBoundingClientRect().bottom`과 `tabbar.getBoundingClientRect().top`의
차이를 직접 재보세요. 처음에 어림짐작한 "탭바 61px"로 계산했을 땐 실제 간격이 23px(목표 20px과 다름)이
나왔고, 진짜 CSS로 다시 측정해 탭바가 58px임을 확인한 뒤에야 정확히 20px이 나왔습니다 — 어림값이
아니라 항상 실측치로 계산하세요.

---

## 9. 목록 단어 상세/단어장 관리/피드백 팝업 — Portal로 이동 (X버튼 안 닫히는 버그 수정)

### 무엇이 문제였나
사파리(특히 **홈 화면에 추가하지 않은 일반 브라우저 탭**)에서 목록 탭 → 단어를 눌러 뜨는 상세
팝업(`WordDetailModal`)의 X버튼이 **헤더에 가려져서 눌리지 않고**, 그 상태에서 스크롤하면 팝업이 아니라
**뒤의 목록(배경)만 스크롤**되는 버그가 있었습니다.

### 원인
이 앱에는 팝업이 여러 개 있는데, 그중 학습 화면에서 쓰는 `DetailModal`(한자/연관단어 상세)만
`ReactDOM.createPortal(..., document.body)`로 **문서 최상단에 직접** 그려집니다. 반면
`WordDetailModal`(목록 탭), `DeckManager`(단어장 관리), `FeedbackBox`의 오류제보 팝업(`.fb-overlay`)은
포털을 안 쓰고 각자 위치(스크롤되는 콘텐츠 영역 안)에 그냥 `position: fixed`로 렌더되고 있었습니다.

iOS Safari는 `overflow-y: auto` + `-webkit-overflow-scrolling: touch`가 걸린 스크롤 컨테이너 **안에**
중첩된 `position: fixed` 자손의 위치를 표준(뷰포트 기준)과 다르게, **그 스크롤 컨테이너 기준으로**
계산하는 알려진 결함이 있습니다. 그래서 팝업이 스크롤 컨테이너가 시작하는 지점(헤더 바로 아래)부터
그려지며 상단(X버튼)이 헤더에 가려지고, 터치도 포털되지 않은 팝업 뒤의 스크롤 영역에 그대로 전달돼
배경만 스크롤되는 것으로 확인됐습니다.

### 고친 방법 — 이미 정상 동작하는 `DetailModal`과 동일하게 Portal 적용
세 곳의 `return (...)` 을 `return ReactDOM.createPortal((...), document.body)` 로만 바꾸면 됩니다.
조건부 렌더인 `FeedbackBox`는 `{open && (...)}` 부분을 `{open && ReactDOM.createPortal((...), document.body)}`로.

```jsx
// WordDetailModal, DeckManager — 함수 전체가 이 형태
return ReactDOM.createPortal(
  <div className="modal-overlay ..." onClick={onClose}>
    {/* ...내용은 그대로... */}
  </div>,
  document.body
);

// FeedbackBox — 조건부 렌더 부분만
{open && ReactDOM.createPortal(
  <div className="fb-overlay" onClick={() => setOpen(false)}>
    {/* ...내용은 그대로... */}
  </div>,
  document.body
)}
```
> ⚠️ `FirstRunGuide`(`.guide-overlay`)는 원래부터 `App`의 최상위(스크롤 영역 밖) 자식이라 이 버그의
> 영향을 받지 않으므로 포털 처리하지 않았습니다. **팝업을 새로 추가할 때는, 그 팝업이 스크롤되는
> 컨테이너 안쪽 어딘가(리스트 아이템 클릭 등)에서 열리는 구조라면 처음부터 `createPortal`로 만드세요.**

### 이식 시 체크리스트
개인 앱에 `position: fixed` 오버레이 팝업이 스크롤 컨테이너 안에 중첩돼 있는 곳이 있다면(특히 리스트
아이템을 눌러서 여는 상세/모달류), 전부 같은 버그에 노출돼 있을 수 있습니다. `ReactDOM.createPortal`을
이미 쓰는 팝업과 안 쓰는 팝업을 목록으로 뽑아 비교해 보세요.

---

## 10. 온보딩 앱 셸 구조 변경 + 상단 잔상(노란색) 버그 대응 (누적 히스토리)

### 무엇이 문제였나
온보딩(레벨 선택) 화면은 `position: fixed`로 전체화면을 노란 배경(`--accent`)으로 덮는 오버레이였습니다.
온보딩이 끝나고 본 화면으로 전환될 때, iOS Safari에서 **상단(노치/상태바 인접 영역)에 노란 잔상이
남는** 버그가 있었고, 여러 차례 대응을 거쳤습니다.

### 시도 순서 (전부 코드에 남겨둔 안전망 — 나중에 걷어내지 말 것)
1. **`forceRepaint()` 도입** — `documentElement`를 한 프레임 `display:none`→복원해 강제 리플로우.
2. **`body`/`html` 배경 + `theme-color` meta 동기화** — 온보딩 상태(`onboard.done`)에 맞춰
   실제 배경색과 `<meta name="theme-color">`를 JS로 동기화(불일치가 잔상 원인 중 하나로 추정됨).
3. **`forceRepaint()` 강화** — 스크롤 1px 넛지 + opacity 미세 토글 + display 토글 3종 조합.
4. **앱 셸 구조 변경(가장 근본적)** — `.app`을 `position: fixed; inset: 0; overflow: hidden` 에서
   **일반 흐름(in-flow) + `height: 100dvh`(fallback `100vh`)** 로 변경. `.onb`(온보딩)도 같은 이유로
   `position: fixed` 대신 `.app`(flex column)의 `flex: 1` 자식으로 변경. 별도 합성(compositing) 레이어를
   안 만들면 "제거된 fixed 레이어의 잔상" 부류의 버그가 애초에 생길 토대가 없어진다는 논리.
5. **스크롤 넛지의 실효성 문제 발견** — `html, body { overflow: hidden }` 이라 문서 자체의 스크롤
   가능 범위가 항상 0이었고, 3번의 "스크롤 1px 넛지"가 사실상 no-op이었을 가능성이 확인됨. 아래처럼
   `forceRepaint()` 실행 시 잠깐 스크롤을 허용하도록 수정:
```javascript
function forceRepaint() {
  try {
    // html/body가 overflow:hidden이라 스크롤 가능한 범위가 원래 0이었다 — 그래서
    // 아래 스크롤 넛지가 지금까지 실제로는 아무 것도 안 움직이는(no-op) 상태였을 수 있다.
    // 1px짜리 스페이서를 잠깐 붙여 진짜 스크롤 여지를 만든 뒤 넛지하고 원상복구한다.
    const html = document.documentElement, body = document.body;
    const prevHtmlOverflow = html.style.overflow, prevBodyOverflow = body.style.overflow;
    const spacer = document.createElement("div");
    spacer.style.cssText = "width:1px;height:1px;";
    body.appendChild(spacer);
    html.style.overflow = "auto"; body.style.overflow = "auto";
    const x = window.scrollX, y = window.scrollY;
    window.scrollTo(x, y + 1);
    window.scrollTo(x, y);
    html.style.overflow = prevHtmlOverflow; body.style.overflow = prevBodyOverflow;
    spacer.remove();
    // ...opacity 토글 + display 토글은 기존 그대로...
  } catch (e) {}
}
```

### ✅ 실기기(iOS Safari) 검증 완료 (원본 앱, 2026-07-01)
이 문서 작성 시점(v0.2.8)에는 5번(스크롤 넛지 실효성 수정)이 가설 단계였지만, 이후 `JP_vocab_pcm`(사내
배포용) 쪽에서 X버튼 버그와 잔상 버그 **둘 다 실기기에서 해결 확인**됐습니다(추가 코드 변경 없이 위
1~5번 조치로 해결). 이식 대상 앱에서도 같은 구조(스크롤 컨테이너 안 중첩 `position:fixed`, `.app`
fixed 오버레이)였다면 위 조치를 그대로 옮기는 것으로 충분할 가능성이 높습니다. 그래도 재현되면 아래
후보를 순서대로 시도해 보세요:
- `apple-mobile-web-app-status-bar-style`을 `default`에서 `black-translucent`로 (단, **PWA 설치(standalone)
  모드에만 영향** — 일반 브라우저 탭에는 효과 없음)
- 최후 수단: `location.reload()`를 온보딩 종료 시점에 되살리되, reload 직전 0.1초 페이드 처리로
  깜빡임을 숨김

### 온보딩 행간(line-height) 최종값 — 참고용
여러 차례 축소/복원을 거쳐 최종적으로 정착된 값입니다. 개인 앱에도 같은 디자인을 적용한다면 참고하세요:
```css
.onb-title { font-size: 23px; line-height: 1.35; }       /* 최초 26px/1.35에서 폰트만 23px로 축소, 행간은 원복 */
.onb-sub { font-size: 14px; line-height: 1.6; }            /* 원래 값으로 복원 */
.onb-level-desc { font-size: 13px; line-height: 1.2; }     /* 축소된 값 유지(사용자가 이 항목만은 유지 요청) */
.onb-note { font-size: 12.5px; line-height: 1.28; }        /* 축소된 값 유지 */
```

---

## 11. 발음 규칙(WORD_SPEECH_OVERRIDE) 대량 보강 — 1,010건 추가 (Wayne QA)

### 배경
전 단어(8,610개) TTS 발음 QA를 외주(Wayne)로 진행했고, 그 결과로 `WORD_SPEECH_OVERRIDE` 객체에
**1,010건**을 새로 추가했습니다(기존 21건 → 총 1,032건). 개인 앱이 같은/겹치는 단어 데이터를 쓴다면
이 보강분이 곧바로 발음 품질 개선으로 이어집니다.

### 동작 원리 — `wordSpeech(card)`
```javascript
function wordSpeech(card) {
  const ov = WORD_SPEECH_OVERRIDE[card.word] || (card.reading && WORD_SPEECH_OVERRIDE[card.reading]);
  if (ov) return withWordStop(tildeToYomi(ov));
  // 단어는 한자 원문 그대로 보낸다(엔진이 사전형으로 또렷이 읽음).
  // 읽기(히라가나)로 보내면 고립 가나가 오분절된다. 多音 오독 단어만 OVERRIDE에 읽기 등록.
  const clean = card.word.split(/[／/↔≠\s(]/)[0];
  return withWordStop(tildeToYomi(clean));
}
```
**핵심**: TTS는 기본적으로 `card.word`(한자 원문)를 그대로 엔진에 보냅니다. `card.reading`(화면에 표시되는
읽기)은 TTS 발음에 **전혀 영향을 주지 않습니다** — `WORD_SPEECH_OVERRIDE`에 등록된 키만 예외적으로
그 단어를 말할 때 히라가나 값으로 대체됩니다. 즉 "표시되는 읽기가 맞다"와 "TTS가 맞게 읽는다"는
**서로 독립적인 문제**이니, 읽기 필드만 고치고 발음이 좋아졌다고 착각하지 않도록 주의하세요.

### 추가된 항목의 유형 (이식 시 그대로 옮기면 되는 것들)
- **多音 한자 오독 방지**: 여러 음/훈독이 있는 한자를 엔진이 잘못 읽는 경우 히라가나로 강제
  (예: `"心中": "しんちゅう"`, `"日本": "にほん"`).
- **복수 표기/괄호 주석 정리**: `word`나 `reading` 필드에 `"고교; 고등학교"`처럼 여러 표기가 같이
  들어있거나, `"(かん)"`·`"(な)"` 같은 품사 주석이 섞여 있으면 그대로 엔진에 보내면 안 되므로,
  대표 표기 하나만 말하도록 정리(예: `"高校; 高等学校": "こうこう"`).
- **가타카나 축약어의 실제 발음**: 표기와 실제로 읽는 방식이 다른 축약어(예: `"ラジオカセ": "らじかせ"`).
- **속어/신조어 특유 표기**: `daily` 카테고리의 신조어(`ダメ出し`, `ドン引き` 등)는 표기 그대로 읽으면
  부자연스러운 경우가 많아 대량으로 등록됨.

### 이식 방법
`docs/WORD_SPEECH_OVERRIDE_additions.js`(Wayne QA 산출물, 레포 밖 인수 폴더에 있음 — 필요하면 이
저장소의 `WORD_SPEECH_OVERRIDE` 객체 정의(약 586번째 줄)를 열어 직접 복사)를 개인 앱의 동일 객체에
그대로 붙여넣으면 됩니다. **기존 키와 충돌 여부만 확인**하세요(이번 반영 시 충돌 0건이었음).
개인 앱의 단어 데이터가 다르면, 겹치지 않는 단어의 override는 그냥 무해하게 무시됩니다(키가 없으면
`wordSpeech`가 기본 로직으로 폴백).

---

## 12. 모달 딤(어두운 막) 그라데이션 + 팝업/모달 위치 — 상태바 경계선 제거

### 무엇이 문제였나 (iOS standalone PWA 전용)
모달/오버레이가 뜰 때 배경을 반투명 검정(`rgba(0,0,0,.35~.45)`)으로 덮는데, **홈 화면에 설치한 앱
(standalone)** 에서는 상단 **상태바(시계·다이내믹아일랜드) 띠**가 웹 콘텐츠가 아니라 시스템이 따로
그리는 영역이라, 본문만 어두워지고 상태바는 밝게 남아 **헤더 경계에 색이 갈라지는 선**이 보였다.
(열고/닫을 때 상태바와 본문이 서로 다른 시점에 어두워졌다 밝아져 "깜빡이는" 것처럼도 보였다.)

### 시도했다가 실패한 것들 (다시 하지 말 것)
`<meta theme-color>`/`html`·`body` 배경을 딤 색과 동기화, 오버레이·헤더·탭바 GPU 레이어 승격
(`translateZ`), `-webkit-overflow-scrolling:touch` 제거, `forceRepaint` 등 — **전부 실패하거나 탭바 색이
물드는 새 부작용을 냈다.** standalone 상태바는 웹에서 색을 못 맞춘다는 게 결론.

### 해결책 (오리지날 앱 방식이 정답) — **상단을 아예 딤하지 않는다**
상태바를 어둡게 맞추려 하지 말고, **딤을 헤더 아래부터만** 주고 그 경계를 **그라데이션으로 흐린다.**

1. **헤더 아래 y좌표를 CSS 변수로 노출** — `App`에서 `.topbar`의 화면상 아래 좌표를 측정:
```javascript
useEffect(() => {
  const setVar = () => {
    const tb = document.querySelector(".topbar");
    document.documentElement.style.setProperty("--dim-top", (tb ? Math.round(tb.getBoundingClientRect().bottom) : 0) + "px");
  };
  setVar();
  const t = setTimeout(setVar, 300);            // 폰트/레이아웃 안정화 후 재측정
  window.addEventListener("resize", setVar);
  window.addEventListener("orientationchange", setVar);
  return () => { clearTimeout(t); window.removeEventListener("resize", setVar); window.removeEventListener("orientationchange", setVar); };
}, [onboardDone, version]);
```
2. **오버레이 배경을 `--dim-top` 기준 그라데이션으로** — 헤더까지(0~--dim-top)는 완전 투명,
   그 아래 ~200px에 걸쳐 원래 딤 농도에 도달. `inset:0`(전체화면)이라 카드 위치는 그대로:
```css
.kanji-modal-overlay {           /* 바텀시트(단어/한자 상세) */
  position: fixed; inset: 0; z-index: 9999;
  background: linear-gradient(to bottom,
    rgba(0,0,0,0) var(--dim-top, 0px),
    rgba(0,0,0,.45) calc(var(--dim-top, 0px) + 200px));
  display: flex; align-items: flex-end; justify-content: center;
}
/* .guide-overlay(가이드/업데이트안내, 딤 .4), .fb-overlay(제보, .4)도 같은 형태로 통일 */
```

### 팝업/모달 위치 주의점 (실제로 겪은 함정)
- **바텀시트(`.kanji-modal`)**: 높이를 `90dvh` 같은 고정값으로 두면 헤더 바로 아래까지 차올라 붙는다.
  `height:auto; max-height:82dvh`로 두면 콘텐츠 높이만큼만, 헤더 아래 여백을 남기고 뜬다(팝업이 내려온 느낌).
- **세로로 긴 중앙 정렬 모달**(예: 목록 단어 팝업 `WordDetailModal`, `.modal-overlay`): `inset:0`로 전체화면
  중앙 정렬하면 **위로 올라가 헤더에 걸린다.** 이 종류만 오버레이에 `top: var(--dim-top)`를 줘서
  **헤더 아래 영역에서 중앙 정렬**시킨다(그러면 딤 그라데이션은 `...0, .35 200px`처럼 overlay top에서
  0으로 시작하는 형태로 두면 경계선은 여전히 없음).
- **작은/전체화면 중앙 모달**(가이드·업데이트안내·제보): `inset:0` 그대로 두면 원래 위치(화면 중앙).
- 정리: **바텀시트=`inset:0`+dim-top 그라데이션 / 긴 중앙모달=`top:--dim-top`+top0 그라데이션 / 작은 중앙모달=`inset:0`+dim-top 그라데이션.**

### 스와이프로 닫기 애니메이션
아래로 스와이프해 닫을 때 `translateY(2000px)`처럼 큰 값으로 순간이동시키면 초반이 너무 빨라 뚝 끊긴다.
**`translateY(100%)`(자기 높이만큼, 딱 화면 밖까지)** + `320ms` + iOS 감속 이징 `cubic-bezier(.32,.72,0,1)`로
하면 부드럽다. X버튼/오버레이 탭/Escape도 같은 닫힘 애니메이션(`closing` 상태)을 공유시킨다.

---

## 공통 주의사항 (전체 기능 해당)

1. **버전 표기**: `index.html` 상단 `APP_VERSION`/`APP_BUILD`를 기능 추가할 때마다 갱신하는 컨벤션을
   이 프로젝트에서 쓰고 있습니다(`APP_BUILD`는 작업 당일 YYYY-MM-DD). 개인 앱에도 같은 컨벤션이 있다면 맞추세요.
2. **JSX/문법 검증**: 빌드 단계가 없는 구조라 오타가 바로 흰 화면으로 이어집니다. 수정 후
   `@babel/core`의 `transformSync`로 `<script type="text/babel">` 블록만 떼어 컴파일 검증하는 걸 추천합니다.
3. **이 환경에서 못한 검증**: 샌드박스 네트워크 제약으로 React/Babel CDN을 못 불러와서, 실제 브라우저
   렌더(특히 모달 전환 애니메이션, TTS 음질, 흘려듣기 중 모달 열고 닫는 실사용 흐름)는 코드 리뷰와
   정적 mock 렌더로만 확인했습니다. 이식 후 실제 기기에서 한 번씩 직접 확인하시길 권합니다.

---

## 참고 — 원본(사내 배포용) 커밋 위치
| 기능 | 커밋 |
|---|---|
| 1. 학습 대기목록 | `cefc1dd` |
| 2. 연관단어 이동 액션 | `cefc1dd` (1번과 같은 커밋) |
| 3. 예문 속 단어 | `b689396`(최초) → `6923b4e`(배경색·현재단어 포함 개선) |
| 4. TTS 보이스 고정/지연단축 | `b689396` |
| 5. 흘려듣기 목록 → 상세 (기본 탐색) | `f13ea73` |
| 6. 흘려듣기 상세 액션(별·학습대기·단어장) + 범위기반 랜덤 | `d2e50ba` (v0.2.3) |
| 7. 듣기 "화면 보면서 공부하기" 단어 탭 + bg모드 한정 버그 수정 | v0.2.4 커밋 |
| 8. 목록 단어 상세 최대 크기(탭바 위 20px) | v0.2.4 커밋 |
| 9. 팝업 3종 Portal 이동 (X버튼 버그) | v0.2.8 (`5a5877c`) |
| 10. 앱 셸 구조 변경 + 잔상버그 대응 | v0.2.7(`b8ac036`) 구조변경 → v0.2.8(`5a5877c`) 스크롤넛지 수정 |
| 11. WORD_SPEECH_OVERRIDE 1,010건 추가 | Wayne QA 반영 커밋(`0012860`) |

저장소: `nomi-host/JP_vocab_pcm`, 현재 `main` 브랜치에 전부 병합·배포됨(`pitto-voca.vercel.app`).
`git show <커밋>` 으로 정확한 diff를 바로 확인할 수 있습니다.
