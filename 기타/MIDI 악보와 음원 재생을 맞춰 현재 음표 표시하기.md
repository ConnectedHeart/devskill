# MIDI 악보와 음원 재생을 맞춰 현재 음표 표시하기

> 분류: 기타 · 최근 갱신: 2026-11-08

## 한 줄 요약
악보는 MIDI(또는 MusicXML)에서 그리고 mp3는 소리로만 쓰며, MIDI 음표 시각표와 음원 재생 시각 사이의 시작 오프셋만 맞추면 재생 중인 음표·마디를 찾아 강조하고 페이지를 넘길 수 있다.

## 내용
- **mp3만으로 악보를 만들기 어려운 이유**: 녹음에서 음표를 뽑는 자동 채보는 여러 목소리가 동시에 나오는 합창에서 정확도가 크게 떨어진다. 음색이 비슷하고 화음으로 겹쳐 어느 음이 어느 파트인지 가르기 어렵고, 박자·가사까지 맞는 악보는 더 어렵다. 한 파트 독창 녹음도 음 길이·쉼표가 틀리기 쉽다. 그래서 악보 데이터는 MIDI나 MusicXML로 받는다(종이·PDF는 MuseScore 같은 프로그램으로 입력해 내보냄). MusicXML이 셋잇단·가사 위치 등 인쇄 악보 모양에 더 가깝다.
- **MIDI → 시각표**: 트랙별로 음표의 tick을 템포(마이크로초/4분음표)와 PPQ로 초 단위로 바꾸면 `{part, start, end, measure}` 목록이 된다. 파트 이름은 트랙 이름에서 가져오고, 트랙 하나에 채널로만 나뉜 MIDI는 채널별로 나눠야 파트가 갈린다. 못갖춘마디(첫 마디가 짧음)는 마디 번호 계산에 반영한다.
- **악보 렌더링과 연결**: 웹에서는 OpenSheetMusicDisplay(OSMD, BSD)로 그린 뒤, 그려진 음표 요소와 시각표의 음표를 순서대로 짝지어 둔다. 재생 중에는 `currentTime`으로 이진 탐색해 현재 음표를 찾고 해당 요소에 클래스를 붙인다. 매 프레임 DOM을 다시 그리지 말고 바뀐 음표만 클래스를 바꾼다.
- **음원과 MIDI의 시작 위치 맞추기**: 같은 MIDI로 만든 연습 음원이면 템포가 같아 시작 오프셋 하나만 맞추면 끝까지 맞는다. 앞에 카운트인(메트로놈 박)이 있으면 "처음 소리 나는 지점"을 첫 음으로 잡는 단순 방식은 틀린다. 음원의 음 높이(피치) 흐름과 MIDI 첫 몇 음의 음 높이를 비교해 가장 잘 맞는 오프셋을 고른다. 자동 값이 틀릴 때를 위해 수동 보정(±)과 "지금 들리는 음표를 눌러 맞추기"를 둔다.
- **맞지 않는 경우**: 사람이 직접 부른 녹음(템포가 흔들림)은 시작만 맞아도 뒤로 갈수록 어긋난다. 이를 따라가려면 구간별 정렬(DTW 같은 동적 시간 정렬)이 필요하다. 다른 편곡·다른 템포·다른 조 음원은 자동 맞춤이 틀린다.
- **페이지 넘김 기준**: "선택한 파트의 현재 음표" 위치로 넘기면 그 파트가 쉬는 동안 다른 파트가 다음 페이지로 가도 화면이 멈춘다. **재생 중인 마디** 위치로 넘기고, 강조만 선택 파트에 하는 것이 맞다. 손으로 넘긴 뒤에는 몇 초간 자동 넘김을 멈췄다가 다시 따라가게 한다. 넘김이 너무 잦으면 한 페이지에 들어가는 마디를 늘린다(선택 파트에만 가사 표시, 확대율 낮추기, 선택 파트 한 줄만 보기).
- **반주 + 녹음**: 반주 재생과 `MediaRecorder` 녹음을 같이 시작하고 녹음 시작 위치(초)를 함께 저장해 두면, 다시 들을 때 반주를 그 위치부터 같이 틀 수 있다. 출력·입력 지연(특히 블루투스 이어폰 0.2~0.4초) 때문에 목소리가 늦게 들리므로 사용자가 고르는 지연 보정값을 저장해 재생 시 목소리를 그만큼 앞당긴다. 녹음 시작이 실패했을 때 "녹음 중" 상태를 되돌리는 처리를 빠뜨리기 쉽다.

## 핵심 코드
```javascript
// MIDI 음표(tick)를 초로 바꾸기 - 템포 변경 목록(tempoMap)을 순서대로 적용
function tickToSec(tick, ppq, tempoMap) {           // tempoMap: [{tick, usPerQuarter}] 오름차순
  let sec = 0, lastTick = 0, us = 500000;            // 기본 120 BPM
  for (const t of tempoMap) {
    if (t.tick >= tick) break;
    sec += (t.tick - lastTick) / ppq * us / 1e6;
    lastTick = t.tick; us = t.usPerQuarter;
  }
  return sec + (tick - lastTick) / ppq * us / 1e6;
}
```

```javascript
// 재생 중 현재 음표·마디 찾기 (notes는 start 오름차순, offset은 음원 시작 보정값)
function findIndex(notes, t) {
  let lo = 0, hi = notes.length - 1, ans = -1;
  while (lo <= hi) {
    const mid = (lo + hi) >> 1;
    if (notes[mid].start <= t) { ans = mid; lo = mid + 1; } else hi = mid - 1;
  }
  return ans;
}
function onFrame() {
  const t = audio.currentTime - offset;
  const i = findIndex(allNotes, t);                  // 모든 파트 기준 -> 페이지 넘김
  if (i >= 0) showPageOf(allNotes[i].measure);
  const j = findIndex(notesByPart[selectedPart], t); // 선택 파트 기준 -> 강조
  setHighlight(j >= 0 && t < notesByPart[selectedPart][j].end ? notesByPart[selectedPart][j] : null);
  requestAnimationFrame(onFrame);
}
```

```javascript
// 반주와 함께 녹음 시작 - 실패하면 상태를 되돌린다
async function startRecording() {
  recording = true;
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    const rec = new MediaRecorder(stream);
    const chunks = [];
    rec.ondataavailable = e => chunks.push(e.data);
    rec.onstop = () => { stream.getTracks().forEach(t => t.stop());
      saveTake({ blob: new Blob(chunks), startAt, latency: userLatency }); };
    const startAt = audio.currentTime;
    rec.start(); audio.play();
    return rec;
  } catch (e) {
    recording = false;                               // 빠뜨리면 버튼이 "녹음 중"에 멈춤
    throw e;
  }
}
// 다시 듣기: 반주는 startAt부터, 목소리는 latency만큼 앞에서 시작
// voice.currentTime = latency; accomp.currentTime = take.startAt; voice.play(); accomp.play();
```

## 주의할 점
- 연습 음원이 MIDI로 만든 것인지 먼저 확인한다. 같은 MIDI 기반이 아니면 오프셋 하나로 맞지 않는다.
- 파트 이름으로 음자리표를 정할 때 음역만 보면 알토가 낮은음자리표로 그려질 수 있다. 트랙 이름(Soprano·Alto)이 있으면 그것을 우선한다.
- 텍스트 입력 칸에 포커스가 있을 때는 재생 단축키(스페이스바 등)를 막는다.
- 녹음은 이어폰을 끼고 해야 반주가 섞이지 않는다. iOS는 녹음 중 소리가 수화부 스피커로 나올 수 있다.

## 참고
- OpenSheetMusicDisplay (opensheetmusicdisplay.org)
- MDN: MediaRecorder, HTMLMediaElement.currentTime
- Standard MIDI File 형식(PPQ, Set Tempo 메타 이벤트)
