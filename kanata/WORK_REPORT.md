# WORK_REPORT — keyd → kanata 이전 작업 인수인계

> 다음 에이전트를 위한 문서. `kanata.cfg` 를 이어서 수정/디버깅할 때 필요한 배경, 문법 출처,
> 하드코어한 함정(gotcha), "이미 시도했다가 버린 접근" 을 정리했다. **kanata 설정 언어는
> 비표준 S-expression 이라 기억에 의존하지 말고 항상 아래 문서/소스를 인출해 확인할 것.**

---

## 0. 목표와 현재 상태

- **목표**: 기존 `keyd`(Linux 전용) 설정을 **여러 OS에서 쓸 수 있는 `kanata`** 로 이전.
  단일 `kanata.cfg` 하나로 **Linux + Windows** 양쪽에서 동작하게 하는 것이 요구사항.
- **원본 keyd 동작**: CapsLock=Ctrl/한영 오버로드, 왼Ctrl→왼Alt, 왼Alt→모드레이어
  (숫자→F1~F12, ijkl→방향키, uo=Home/End, yh=PgUp/PgDn, esc/grave/tab→Alt+Tab, 미디어/밝기).
- **동봉 파일**: `kanata.cfg`(설정), `kanata-tray.png`(Windows 트레이 아이콘).

### 사용자 실기 검증 완료 (working)
- CapsLock: 누르는 즉시 Ctrl(eager), 짧게 탭하면 한/영. `Ctrl+마우스클릭`, 빠른 `Ctrl+C` 정상.
- Alt+Tab: 첫 tab 안 새고 정상 전환, 꾹 누르면 연속 전환.
- Windows 한/영 전환(오른 Alt 출력), 넘패드/방향키 정상 입력.
- 아이콘 "layer" 경고 제거됨(`icon-match-layer-name no`).

### 남은/못 고치는 것 (아래 6절 참고)
- 물리 오른Alt(한/영)·오른Ctrl(한자)·넘패드 누를 때 뜨는 keystate desync **ERROR 로그**.
  → 알림 팝업은 `notify-error no`로 껐음. 콘솔 로그는 근본적으로 못 없앰(끄면 stuck 위험).

---

## 1. 문서/소스 출처 (항상 여기서 확인)

kanata 는 QMK/keyd 와 문법이 다르고 버전마다 바뀐다. **추측 금지, 매번 확인.**

| 용도 | 링크 |
|---|---|
| 설정 가이드(정본) | https://github.com/jtroo/kanata/blob/main/docs/config.adoc |
| 〃 raw(grep용) | https://raw.githubusercontent.com/jtroo/kanata/main/docs/config.adoc |
| 렌더링 HTML | https://jtroo.github.io/config.html |
| **키 이름**(공용 str→oscode) | https://raw.githubusercontent.com/jtroo/kanata/main/parser/src/keys/mod.rs |
| Windows VK 매핑(OsCode↔VK) | https://raw.githubusercontent.com/jtroo/kanata/main/parser/src/keys/windows.rs |
| Linux evdev 매핑 | https://raw.githubusercontent.com/jtroo/kanata/main/parser/src/keys/linux.rs |
| macOS 매핑(HID usage) | https://raw.githubusercontent.com/jtroo/kanata/main/parser/src/keys/macos.rs |
| **defcfg 옵션 전체**(+플랫폼 게이팅) | https://raw.githubusercontent.com/jtroo/kanata/main/parser/src/cfg/defcfg.rs |
| GUI/트레이/아이콘 로직 | https://raw.githubusercontent.com/jtroo/kanata/main/src/gui/win.rs |
| 트레이 아이콘 예제 | https://github.com/jtroo/kanata/blob/main/cfg_samples/tray-icon/tray-icon.kbd |
| deflocalkeys 로케일 모음 | https://github.com/jtroo/kanata/blob/main/docs/locales.adoc |

> **팁**: 큰 adoc 은 WebFetch 가 뒷부분을 잘라버린다. `curl -sL <raw> -o /tmp/x && grep -in ...`
> 로 로컬 grep 하는 게 확실하다. 키 이름/코드/옵션은 문서보다 **위 `.rs` 소스가 진짜 정본**이다.

---

## 2. `kanata.cfg` 블록별 구조

파일은 `(defcfg) (deflocalkeys-*) (defsrc) (deflayer base) (deflayer altmod) (defalias) (defvirtualkeys)` 순.

- **defcfg**: `process-unmapped-keys yes`(tap-hold 인터럽트 판정에 필요), 그리고 Windows GUI 전용
  옵션 `icon-match-layer-name no` / `notify-error no` / `tray-icon`.
- **deflocalkeys-\***: `hangeul` 이름을 플랫폼별 코드로 정의(3절).
- **defsrc / deflayer base / altmod**: 리매핑. 현재 defsrc 는 기본 키들만(오른Alt/오른Ctrl/넘패드는 뺀 상태 — 6절).
- **defalias**: `caps`(eager Ctrl+한영), `mode1`(altmod 레이어), `atab/agrv/aesc`(Alt+Tab).
- **defvirtualkeys**: `valt`(지속 Alt), `vtab/vgrv/vesc`(홀드 대상), `tabseq/…`(순서강제 매크로), `rel-lctl`.

---

## 3. 플랫폼 함정 — deflocalkeys 와 한/영 키 (핵심)

- Linux 에는 한/영 키 **내장 이름이 없다**(`kana`는 Linux 에선 KATAKANAHIRAGANA). 그래서
  `deflocalkeys-linux hangeul 122`(evdev KEY_HANGEUL)로 직접 정의.
- **deflocalkeys 숫자는 "플랫폼별 원시 코드"**: Linux=evdev 스캔코드, **Windows=가상키(VK) 코드**.
  (문서 예제 `ì 187`=VK_OEM_PLUS 가 win/winiov2/wintercept 모두 동일 → Windows 3변종 다 VK 사용)
- **Windows 한/영은 오른쪽 Alt** 였다. `VK_HANGEUL(21)` 은 이 환경에서 토글 안 됨 →
  `hangeul 165`(VK_RMENU=오른 Alt)로 출력. Linux 는 그대로 `122`.
- **winIOv2 바이너리는 `deflocalkeys-winiov2` 만 읽고 `deflocalkeys-win` 은 무시**한다.
  그래서 win/winiov2/wintercept 3개를 전부 정의해 뒀다. (사용자 바이너리: `kanata_windows_gui_winIOv2_x64.exe`)
- macOS 는 HID usage(page 0x07/code 0x90) 라 값 체계가 달라 생략함. 필요 시 내장 `Lang1` 사용.

---

## 4. 비자명 관용구(idiom)와 이유 — 반드시 읽을 것

이 설정의 까다로운 부분들. 함부로 "단순화" 하면 깨진다.

### 4-1. CapsLock = eager Ctrl + 탭 한/영
```
caps    (multi lctl (tap-hold-press 260 260 @han-tap XX))
han-tap (macro (on-press tap-vkey rel-lctl) 10 hangeul)   ;; rel-lctl = (release-key lctl)
```
- kanata `tap-hold` 는 **lazy**(홀드 확정 전엔 아무것도 안 보냄) → keyd `overload` 처럼
  "누르는 즉시 Ctrl" 이 안 된다. **`(multi lctl (tap-hold …))`** 로 바깥 `lctl` 이 항상 즉시
  Ctrl 을 잡고(eager), 안쪽 tap-hold 는 "빠른 단독 탭이면 한/영도 추가" 만 판정하게 했다.
- fcitx5 는 **Ctrl 이 눌린 채 hangeul 이 오면 무시** → 탭 시 `release-key lctl → 10ms → hangeul`
  순서를 강제. 그런데 **`macro` 는 `release-key` 를 직접 못 받는다**(허용 목록에 없음). 그래서
  `(release-key lctl)` 를 담은 **가상키(`rel-lctl`)를 `tap-vkey` 로 호출**한다. 이건 문서의
  `relf24 (release-key f24)` 관용구와 동일(config.adoc 내 `relf24` 검색).

### 4-2. Alt+Tab — 순서 강제 + 홀드
```
atab   (multi (on-press tap-vkey tabseq) (on-release release-vkey vtab))
tabseq (macro (on-press press-vkey valt) 2 (on-press press-vkey vtab))   ;; defvirtualkeys 안
```
- `multi` 는 동시성/순서가 "가끔 예상 밖"(문서 경고). 실제로 alt 와 tab 을 같이 보내면 **tab 이
  alt 보다 먼저 도달**해 포커스 창에 맨 tab 이 샜다. → 매크로로 `alt 누름 → 2ms → tab 누름·홀드`
  순서를 강제. tab 은 **vkey 로 홀드**(꾹 누르면 자동반복 연속전환), 물리키 떼면 vkey 만 release.
- `valt`(Alt)는 여기서 안 풀고 **`mode1` 의 `on-release` 로 모드키 뗄 때** 풀어야 여러 번 탭/방향키
  이동이 이어진다. (grave/esc 도 동일 패턴)
- 2ms 로도 가끔 새면 `tabseq` 들의 `2` 를 `5` 로.

### 4-3. macro / virtual-key 규칙 (외우지 말고 확인)
- `macro` 허용: 딜레이(**맨 숫자 = ms**), 키, 코드, chorded 서브매크로, 그리고 **일부 특수액션**
  (virtual-key 액션, `unmod`, cmd, unicode 등). **`release-key` 직접 불가** → 4-1 우회.
- vkey 액션: `press-vkey`(누르고 유지), `release-vkey`, `tap-vkey`, `toggle-vkey`
  (`*-virtualkey` 롱네임 별칭). `press-vkey` 는 release 전까지 유지된다.
- `XX` = no-op(무동작). `_` = 투과(아래 레이어로). base 레이어의 `_` 는 그 키 자체.

---

## 5. 편집 규칙 (실수 방지)

- **defsrc 와 모든 deflayer 는 토큰 개수·순서가 완전히 일치**해야 한다. 키를 추가하면
  defsrc + base + altmod **세 곳 모두** 같은 위치에 추가할 것. (안 맞으면 컴파일 에러)
- 키 이름은 4절 링크의 `parser/src/keys/mod.rs` `str_to_oscode` 가 정본. 예: 오른Ctrl=`rctl`,
  넘패드=`kp0..kp9`/`kp.`/`kprt`, 밝기=`brup`/`brdown`, 볼륨=`volu`/`vold`/`mute`.
- **Windows 전용 defcfg 키는 Linux 에서 안전**하다: `icon-match-layer-name`/`notify-error`/`tray-icon`/
  `windows-*` 는 defcfg.rs 에서 **match arm 은 모든 플랫폼에 존재**하고 본문만 `#[cfg(windows…)]`
  로 감싸져 Linux 에선 no-op. (단, **완전히 모르는 키는 `bail!` 로 에러** — defcfg.rs 의
  `_ => bail_expr!("Unknown defcfg option")` 참고. 새 옵션 넣기 전 반드시 defcfg.rs 에서 존재 확인)

---

## 6. 못 고치는 이슈 — keystate desync (중요, 재작업 금지)

증상: 물리 **오른Alt(한/영)·오른Ctrl(한자)·넘패드** 를 누르면
`[ERROR] Unexpected keycode is pressed in kanata but not in Windows. Clearing kanata states: RCtrl` 류 로그.

- **원인**: 한국형 키보드에서 오른Alt=한/영, 오른Ctrl=한자. Windows 가 이 IME 키들의 입력
  이벤트를 LLHOOK 훅에 불규칙 전달 → kanata 가 key-up 을 놓쳐 "눌린 채" 로 오인 → 안전장치
  `windows-sync-keystates`(기본 on)가 실제 상태와 대조해 **stuck 상태를 정리**하며 ERROR 로그.
- **시도했다가 버린 것**: 이 키들을 defsrc 에 `자기자신` 으로 넣어 kanata 가 상태를 소유하게
  하는 방법 → **효과 없음**(consumption 이 output 하류라 매핑으로 못 잡음). 그래서 사용자가
  defsrc/layer 에서 오른Alt/오른Ctrl/넘패드를 **도로 제거**함. 다음 에이전트도 재시도 말 것.
- **적용한 완화**: `notify-error no` → ERROR 를 **시스템 알림(토스트)으로 안 띄움**(로그엔 남음).
  이 로그는 **INFO/WARN 아니라 ERROR** 라 notify 대상이었음.
- **콘솔 로그까지 없애려면** `windows-sync-keystates no` — 감지 자체를 끔. **비추천**: 이 sync 가
  놓친 key-up 을 복구하는 것이라, 끄면 **오른Alt/오른Ctrl 이 눌린 채 고착(stuck modifier)** 위험.
- **근본 해결 후보**(설정 밖): ① Interception 드라이버 빌드(`wintercept`)로 낮은 계층에서 잡기
  (드라이버 설치 필요) ② Windows IME 단축키를 오른Alt/오른Ctrl 이 아닌 키로 변경하고 한/영은
  CapsLock 탭으로 대체(CapsLock→RAlt **출력** 경로는 kanata 가 자기 출력이라 desync 없음).

---

## 7. 아이콘 경고 (해결됨) — 참고

- Windows **GUI 빌드만** 트레이 아이콘 로그를 낸다(`src/gui/win.rs`). 전부 INFO/WARN 이라
  알림은 아님, 콘솔 노이즈일 뿐.
- `icon-match-layer-name no` → **레이어별** 아이콘 탐색 끔("...config+layer = ...: altmod" 제거).
- `tray-icon "kanata-tray.png"` → **config 레벨** 아이콘 지정("...for config: conf.kbd" 제거).
  파일명만 주면 config 폴더/실행파일 폴더/`%APPDATA%\kanata` 등에서 탐색하므로,
  **`kanata-tray.png` 를 실제 config 파일과 같은 폴더에 두어야** 한다.
- `internal bug?: icon_0_key should never be empty?` 는 kanata 스스로 붙인 상류 진단(무해).
- 트레이가 필요없으면 **non-GUI(콘솔) winIOv2 바이너리**를 쓰면 아이콘 로그가 아예 안 생긴다.

---

## 8. 검증 방법

- 컨테이너/헤드리스 환경엔 입력 시스템이 없어 **실기 검증은 사용자가 수행**한다.
- 문법만 확인하려면 실기에서 `kanata --cfg kanata.cfg --check`.
- 체크 포인트: CapsLock(즉시 Ctrl / 탭 한영 / 빠른 Ctrl+C 에 한영 안 샘),
  Alt+Tab(첫 tab 안 샘 / 홀드 연속), 한/영 토글, 넘패드·방향키 입력.
