# NeoKlavier Operator — v0.3

외부 MIDI 장치와 로컬 MIDI 파일 플레이리스트를 이용하는 공연·리허설용 브라우저 플레이백 도구입니다.

**GitHub description**
> A local-first MIDI playlist operator with preroll, pedal control, global playback rate and velocity scaling, built as a single-file web app for GitHub Pages.

## 실행

`index.html` 하나로 실행합니다. 빌드, npm 설치, 서버 API, CDN은 필요하지 않습니다. MIDI 파일을 서버에 전송하지 않습니다.

- 파일을 브라우저로 열면 내장 음원으로 확인할 수 있습니다.
- 외부 MIDI 장치는 데스크톱 Chrome/Edge에서 HTTPS 또는 localhost로 실행하고 `MIDI 장치 연결`을 누르세요.
- GitHub Pages: 새 저장소의 루트에 `index.html`을 올리고 Settings → Pages에서 배포 브랜치의 루트를 선택합니다. 이 패키지는 저장소를 생성하거나 Pages 배포를 실행하지 않습니다.
- 로컬 HTTP 실행: `python3 -m http.server 8080` 후 `http://localhost:8080` 접속.

## 사용 순서

1. MIDI 파일을 여러 개 추가하거나 화면에 드롭합니다.
2. 행을 드래그하거나 ↑/↓ 버튼으로 큐 순서를 정합니다. 행을 누르면 다음 재생 대상으로 선택됩니다.
3. 입력·출력 장치와 프리롤, 갭, 속도, 벨로시티를 설정합니다.
4. `재생`은 즉시 시작합니다. `프리롤`은 설정된 초를 기다린 뒤 시작합니다.
5. `자동 진행`을 켜면 곡 종료 후 갭을 기다리고 다음 곡을 재생합니다. 기본값은 Off입니다.
6. 페달 제어는 입력 장치를 선택하고 `페달 CC 매핑` → `매핑 추가` → `Learn` 순으로 등록합니다.

## 구현 범위와 정책

| 항목 | 동작 |
| --- | --- |
| 파일 | SMF 형식 0·1, .mid/.midi, PPQ 템포 맵, SMPTE 시간축, running status |
| 큐 | 순서 유지 다중 추가, 행 드래그, 이동 버튼, 선택·재생 상태 분리, 삭제 확인 |
| 프리롤 / 갭 | 각각 0–999초 정수. 실경과 시간 기준, 갭에는 프리롤을 중복 적용하지 않음 |
| 속도 / 벨로시티 | 각각 10–300%. 새 재생 시작 시 고정, Pause/Resume에는 기존 값 유지 |
| 벨로시티 계산 | `clamp(round(source * percent / 100), 1, 127)`. Note On 0 및 Note Off는 변경하지 않음 |
| 외부 MIDI | Note On/Off, CC, Program Change, Pitch Bend, poly/channel aftertouch 전달 |
| 내장 음원 | Grand Piano·Upright Piano의 감쇠 현 모델링, FM EP, 최대 64보이스, sustain, ±2 semitone pitch bend, 실시간 volume/expression |
| 페달 | 최대 32매핑, 전체/개별 채널, CC 0–127, 감도 0–100%, 임계값 1–127, 반전, 디바운스 0–5000ms |
| 감도 | `round(CC * sensitivity / 100)`, 0–127 제한 후 반전 적용. 채널별 상승 엣지와 재입력 대기 관리 |
| 리버브 | 0–100% Mix 슬라이더, 스테레오 convolution, 재생 중 즉시 적용. Stop/Panic에서 잔향 제거 |
| 저장 | 운영 설정, 장치 ID, 매핑, 음색·리버브, 공연 화면 선호도만 LocalStorage 저장. 파일·재생 상태 제외 |
| 공연 화면 | 큰 곡명·카운트다운·진행·다이내믹, Stop/Panic 접근, 지원 시 브라우저 전체 화면 |

프리롤과 갭은 세 자리 정수만 허용합니다. 잘못된 값을 입력하면 재생 전 입력 오류를 표시합니다. 페달 Learn에서 수신한 이벤트는 매핑 등록에만 사용하고 액션을 즉시 실행하지 않습니다.

### 안전 해제

Stop·Panic·곡 전환·종료·출력 변경·출력 분리·오류 시 실행을 취소하고 다음을 처리합니다.

- 출력 포트의 예약 큐 `clear()`
- 추적된 활성 음의 Note Off
- 16채널에 CC64/66/67 = 0, CC120/123/121 = 0 및 Pitch Bend 중앙값
- 내장 신스 보이스 해제 및 비주얼라이저 초기화
- 비동기 음원 초기화 중 Stop을 눌러도 뒤늦게 재생되지 않도록 실행 세대 번호 검사

**이미 물리적으로 분리된 MIDI 장치에는 해제 메시지가 도달하지 않을 수 있습니다.** 장치 자체의 All Notes Off/Panic도 확인하세요.

Panic은 예약과 위치를 초기화하고 강제 해제 완료 안내를 표시합니다. Stop도 같은 안전 메시지를 보내지만 정상 정지로 처리합니다.

### Pause / Resume

Pause에서 잔류 음을 해제하고 위치를 유지합니다. Resume에서 이전 CC·프로그램·피치 상태를 순서대로 복원하고, 일시정지 지점에서 건반이 눌린 상태의 음을 다시 발음합니다. **페달에만 남아 있던 잔향과 음의 어택·엔벌로프까지 연속 복원하지는 않습니다.**

### 운영상 한계

- 그랜드·업라이트는 타건·현 배음·감쇠를 합성한 모델링 음색이며, EP는 FM 일렉트릭 피아노입니다. 녹음된 어쿠스틱 피아노 샘플은 포함하지 않습니다. 음색 변경은 이후 Note On부터 적용됩니다. GM 프로그램별 악기, 드럼 킷, aftertouch/RPN 전체는 재현하지 않습니다.
- Reverb Mix는 내장 음원에만 적용되며 외부 MIDI 장치의 리버브를 조절하지 않습니다. 0%는 dry, 100%는 wet입니다.
- SysEx는 권한을 요청하지 않고 출력에서도 제외합니다. 포함된 파일에는 안내를 표시합니다.
- 타이밍은 `performance.now()`와 5ms 폴링을 사용합니다. 브라우저·운영체제·MIDI 드라이버 지연이 있어 하드 실시간 정밀도는 보장하지 않습니다.
- 탭이 숨겨지거나 재생 스케줄러가 300ms 이상 지연되면 몰린 이벤트를 한꺼번에 보내지 않고 안전 정지합니다. 공연 중에는 탭을 전면에 두고 화면 절전 설정도 확인하세요.
- 파일은 최대 32MB, 파일당 최대 1,024트랙, 트랙당 최대 2,000,000파싱 이벤트로 제한합니다. 큰 파일은 로딩 시 UI가 잠시 지연될 수 있습니다.
- 브라우저 권한·실물 MIDI 장치·풋 페달·실제 음향 출력은 대상 환경에서 최종 확인이 필요합니다.

## 단축키

| 키 | 동작 |
| --- | --- |
| Space | 재생 / 일시정지 / 재개 |
| Shift + Space | 프리롤 재생 |
| Esc | 정지 및 안전 해제 |
| ↑ / ↓ | 정지 후 이전 / 다음 곡 선택 |
| Enter | 선택 곡 즉시 재생 |
| F | 공연 화면 전환 |
| P | Panic |

모든 앱 단축키는 입력칸·슬라이더·선택 메뉴·버튼·트랙 카드보다 먼저 처리합니다. 버튼에 포커스가 있어도 Space는 재생 토글, Enter는 선택 곡 재생입니다. 슬라이더의 ↑/↓도 곡 이동에 사용하므로 값은 드래그 또는 ←/→로 조절하세요. 길게 누른 키는 반복 실행하지 않습니다. Ctrl/Command/Alt 조합은 브라우저 기본 동작을 유지합니다. 한글 입력 상태에서도 물리 키 F/P로 공연 화면/Panic을 실행합니다.

## v0.2 화면과 조작

- 트랙 카드는 약 50–55px 높이로 압축하고 이동·삭제 버튼을 한 줄에 배치했습니다.
- 상단 바는 축소하고 스크롤 중에도 상단에 유지합니다. 공연 화면에서는 고정하지 않습니다.
- 영문은 시스템 Helvetica 우선, 없으면 Arial 계열로 대체합니다. Helvetica 파일은 포함하지 않습니다. 한글 SUITE Variable은 HTML에 내장했습니다. 글자 두께는 주로 300–500입니다.
- 프리롤·갭·속도·벨로시티는 세 자리 숫자 칸으로 표시합니다. 클릭 후 숫자를 입력하며 15는 015로 표시됩니다.
- CC 번호·감도·임계값·디바운스는 슬라이더와 숫자로 확인합니다. 수신 CC의 처리값과 임계값 위치를 표시합니다.
- 출력 다이내믹은 인접한 두 MIDI 노트 중 큰 값을 한 막대로 표시하는 64개 막대입니다. 기존보다 약 두 배 두껍고, 약한 값은 투명하게, 큰 값은 불투명하고 진하게 표시합니다.

## 파일 구성

- `index.html`: UI, 파서, 스케줄러, 기본 신스, 페달 처리 전체
- `THIRD_PARTY_LICENSES.txt`: 포함 라이브러리 라이선스
- `tests/v02.cjs`: 전역 단축키, 슬라이더, 레이아웃, 음원·리버브 회귀 테스트
- `tests/smoke.cjs`: Playwright 회귀 테스트, MIDI 파일과 가상 장치를 테스트 중 생성
- `samples/Operator-Test.mid`: 합성 테스트 파일. 곡 저작물이나 라이브러리 음원 아님
- `VALIDATION.md`: 검증 범위 및 남은 현장 확인 항목

테스트 실행: `npm install --no-save playwright`, `npx playwright install chromium`, 정적 서버를 8765포트로 실행한 뒤 `node tests/smoke.cjs`. 테스트 기본 URL은 `http://127.0.0.1:8765`. `BASE_URL`, `CHROMIUM_PATH` 환경변수로 변경할 수 있습니다.

## 라이선스

앱 자체의 공개 여부와 라이선스는 미확정입니다. MIT 또는 다른 라이선스를 임의로 부여하지 않았습니다. 저장소 공개 전에 권리자가 결정하세요.

MIDI 파싱에는 MIT 라이선스의 [midi-file 1.2.4](https://github.com/carter-thaxton/midi-file)를 포함했습니다. 파일 경계, 가변 길이 값, End-of-Track 처리에 로컬 보강이 있으며 저작권·허가문은 HTML 내부와 `THIRD_PARTY_LICENSES.txt`에 보존했습니다.

API 참고: [W3C Web MIDI](https://www.w3.org/TR/webmidi/), [Web Audio](https://www.w3.org/TR/webaudio/).

한글 SUITE는 SUNN의 SIL Open Font License 1.1 서체입니다. 원본 파일을 수정하지 않고 내장했으며 허가문은 HTML 및 THIRD_PARTY_LICENSES.txt에 포함했습니다.

## v0.3

- 코발트 블루 #356BFF, NeoKlavier와 메인 트랙 제목 굵기 900.
- No Output: 음원/MIDI 출력 없이 큐·재생 시간·다이내믹 시각화만 실행.
- 벨로시티 64에서 불투명도 85%, 100–110 파랑→Pre-roll 앰버, 110–120 앰버→Error 빨강.
- 변경된 운영 설정은 파란색 표시. 기본값 리셋은 재생을 정지하고 프리롤·갭 0, 속도·벨로시티 100%, 자동 진행 Off, Grand Piano, 리버브 0%로 복원. 장치 선택과 페달 매핑은 유지.
