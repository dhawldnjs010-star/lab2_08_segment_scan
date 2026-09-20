# 실험 전 레포트: LAB2-08 7세그먼트 자리 스캔

작성자: 상혁 (2025440084) / 작성일: 2026-09-20 / 소스 커밋: `d71b650` / workspace: `LAB1.code-workspace` (템플릿 v2.0.1) / OS: `Windows 11 Home 10.0.26200` / Python: `Python 3.14.7` / 시뮬레이터: Icarus Verilog `Icarus Verilog version 12.0 (devel) (s20150603-1539-g2693dd32b)`

> 이 레포트는 VS Code(Icarus) 시뮬레이션까지의 사전 검증이다. Vivado GUI와 실물 보드 결과는 실험 후 레포트([post](../post/post_report.md))에서 다룬다. 시각은 clk 상승 에지(5, 15, 25, … ns)의 1 ns 뒤, 즉 TB가 비교하는 시각으로 적었다.

## 목적과 예상 동작

8자리 중 한 자리만 켜고 순서대로 반복하는 스캔 회로를 설계한다. 자리 사이에 모든 자리 선택을 0으로 하는 blank 구간을 두고, 각 4비트 숫자를 7세그먼트 패턴으로 바꾼다.

### 포트 (`segment_scan8.v`의 `segment_scan8`)

| 포트 | 방향 | 비트 폭 | 설명 |
|---|---|---|---|
| clk | in | 1 | 상승 에지 기준 클록 |
| rst | in | 1 | 동기 active-high 리셋. index=0, blank=1 |
| enable | in | 1 | 1이면 슬롯 진행, 0이면 현재 슬롯 유지 |
| digits | in | 32 | 8개의 4비트 숫자. index=0이 하위 4비트 |
| select | out (reg) | 8 | active-high 자리 선택(one-hot 또는 00) |
| segments | out (reg) | 8 | abcdefg, dp의 active-high 패턴 |
| index | out (reg) | 3 | 현재 자리 번호, 7 다음 0 |

최상위(`lab2_segment_scan.v`, `lab2_segment_scan`): `enable`은 1'b1로 고정해 자동 스캔한다. `digits = {28'h7654321, switches[7:4]}`, `seg_com = ~{select[0..7]}`(active-low, index 0이 COM[7]), `seg_data = segments`, `led = {5'b00000, index}`.

### 동작 규칙과 경계 입력

- 규칙: 슬롯 하나 = 활성 구간 + blank 구간. enable=1인 에지마다 blank가 반전하고, blank=0에서 blank=1로 넘어갈 때 index가 1 증가한다. select = blank ? 00 : (1<<index). segments는 digits의 index번째 4비트를 7세그먼트로 변환(`0`→FC, `1`→60, … `F`→8E).
- 정상·경계: 리셋 직후 select=00(blank), index=0. enable=0이면 현재 슬롯(활성이든 blank든) 유지. index=7 다음은 0으로 순환. 숫자 A~F 패턴도 확인.
- 시간 기준: TB 클록은 10 ns 주기(상승 에지 5, 15, 25, … ns)이고 입력은 에지 1 ns 뒤에 바꾼다. 레지스터는 다음 상승 에지에서 갱신된다. 실제 보드 클록(1 kHz)은 시뮬레이션의 10 ns와 별개다.

## 소스와 테스트벤치

- 설계 top: `lab2_segment_scan` / 시뮬레이션 top: `tb_segment_scan8`
- 소스: [`src/segment_scan8.v`](../../src/segment_scan8.v), [`src/input_frontend.v`](../../src/input_frontend.v), [`src/lab2_segment_scan.v`](../../src/lab2_segment_scan.v)
- 테스트벤치: [`sim/tb_segment_scan8.sv`](../../sim/tb_segment_scan8.sv)
- 제약: [`constraints/lab2_segment_scan.xdc`](../../constraints/lab2_segment_scan.xdc)
- 설정: [`simulation.json`](../../simulation.json) (sources 3개, testbench `sim/tb_segment_scan8.sv`, simulation_top `tb_segment_scan8`)

| 파일 | 역할 |
|---|---|
| `src/segment_scan8.v` | 핵심 동작을 담은 코어 `segment_scan8`. TB가 이 모듈을 직접 검사한다. |
| `src/input_frontend.v` | 버튼·스위치를 클록에 맞추는 입력 회로. 리셋 2단 해제 동기화, 버튼·스위치 2단 동기화 플립플롭, STABLE_CYCLES=20(1 kHz에서 20 ms) 안정 확인 뒤 한 클록짜리 `press` 펄스를 만든다. |
| `src/lab2_segment_scan.v` | 보드 top. 프런트엔드와 코어를 연결하고 LED로 출력한다. |
| `sim/tb_segment_scan8.sv` | 입력 자극, 기대값 계산, 자동 비교(`check`), PASS/FAIL 출력, VCD 생성, watchdog. |
| `constraints/lab2_segment_scan.xdc` | 핀 번호·전압과 1 kHz 클록 정의. Icarus는 XDC를 읽지 않으므로 이 사전 시뮬레이션에는 사용되지 않는다. |
| `simulation.json` | VS Code 시뮬레이션 작업이 읽는 소스 목록·테스트벤치·시뮬레이션 top. |

### 테스트벤치 동작

- 자극 순서: digits=76543210과 FEDCBA98 두 묶음에 대해 2바퀴씩, 자리마다 (enable=1 활성 → enable=0 유지 → enable=1 blank → enable=0 blank 유지)를 4스텝으로 검사한다. index 순서, select one-hot, 진리표(expected[])와의 일치를 비교하고 마지막에 스캔 중 리셋을 확인한다.
- 검사 횟수: 1(reset)+2(bank)×2(lap)×8(position)×6(index, select, segments, hold, blank, blank hold)+1(reset from active scan) = 194
- 종료·watchdog: 마지막 검사 뒤 `finish` task가 `LAB2_PASS segment_scan8 checks=N`을 출력하고 `$finish`한다. 별도로 100000 ns(100 µs) 뒤에 `watchdog timeout`으로 `$fatal` 처리한다. 예상 종료 시각은 1306 ns이다.
- 이 TB는 코어 `segment_scan8`만 시험한다. 입력 동기화·디바운스와 실제 핀·타이밍이 통과했다는 뜻은 아니다.

### XDC 설명

`lab2_segment_scan.xdc`은 포트 이름을 `lab2_segment_scan.v`과 맞춰 핀을 지정한다. 모든 I/O는 `LVCMOS33`이고 `create_clock -name trainer_1khz -period 1000000.000 [get_ports clk]`로 주 클록을 1 kHz(주기 1,000,000 ns)로 정의하며 `set_false_path -from [get_ports {rst button sw[*]}]`로 비동기 입력을 타이밍 경로에서 제외한다.

| 포트 | 핀 | 보드 대응 |
|---|---|---|
| clk | B6 | 1 kHz 주 클록 |
| rst | K4 | 리셋 (active-high) |
| button | N8 | 스텝 버튼 |
| sw[7:0] | U4(sw[0]), V4, W1, W4, T1, U2, W3, Y1(sw[7]) | DIPSW8..DIPSW1 (DIPSW1..8 = sw[7]..sw[0]) |
| led[7:0] | N5(led[0]), M1, M3, M7, N7, M2, M4, L4(led[7]) | LED0..LED7 |
| seg_data[7:0] | H2, J7, J3, J1, E4, E2, F5, F1 | 세그먼트 데이터 (abcdefg, dp) |
| seg_com[7:0] | K5, K3, K1, L6, G3, G1, H6, H4 | 자리 공통 단자 (active-low, index 0 = COM[7]) |

## VS Code 실행 과정

1. File → New Window → File → Open Workspace from File...로 `LAB1.code-workspace`를 연다. 확장(slang, VaporView, vscode-pdf)을 설치한다.
2. RTL·TB·XDC·`simulation.json`을 직접 입력하고 File → Save All.
3. Terminal → Run Task... → `01 Check tools`로 Git·Python·iverilog·vvp 버전을 확인한다.
4. `02 Simulate`를 실행해 `LAB2_PASS`와 종료 시각을 확인한다.
5. `03 Open waveform`으로 `build/sim/wave.vcd`를 VaporView로 연다. 신호: clk, rst, enable, digits, index, select, segments (index는 10진수, select·segments는 16진수).

정상 실행 로그(본인 로그로 교체하고 `evidence/pre/`에 복사):

```text
LAB2_PASS segment_scan8 checks=194
sim/tb_segment_scan8.sv:19: $finish called at 1306000 (1ps)
```

- 본인 실행 로그: [`../../evidence/pre/lab2_08_normal.log`](../../evidence/pre/lab2_08_normal.log)
- VCD: [`../../evidence/pre/lab2_08_wave_normal.vcd`](../../evidence/pre/lab2_08_wave_normal.vcd)
- 파형 캡처(VaporView): `evidence/pre/lab2_08_wave_zoom.png`(상승 에지 확대)
- 오류: 첫 실행에서 발생한 오류가 있으면 첫 오류 → 수정 → 재실행 로그 순서로 기록한다. (없으면 "없음", Python 실행 경로를 고쳤다면 그 내용 기입)

### 사전 파형 해석

| 시간 구간 | 입력 | 예상 | 실제 파형 | 해석 |
|---|---|---|---|---|
| 6 ns | rst=1 | index=0, select=00, segments=FC | index=0, select=00, segments=FC | 리셋: blank이므로 select=00. segments는 index 0의 숫자 0 패턴(FC)이지만 어떤 자리도 켜지지 않는다. |
| 16 ns | enable=1 · 에지 15 ns | index=0, select=01, segments=FC | index=0, select=01, segments=FC | blank 해제: 자리 0 활성(one-hot 01), 숫자 0 패턴 FC. |
| 26 ns | enable=0 · 에지 25 ns | index=0, select=01, segments=FC | index=0, select=01, segments=FC | enable=0이면 활성 슬롯 유지. |
| 36 ns | enable=1 · 에지 35 ns | index=1, select=00, segments=60 | index=1, select=00, segments=60 | blank 구간: select=00, index는 다음 자리 1로 준비(패턴 60은 숫자 1). |
| 46 ns | enable=0 · 에지 45 ns | index=1, select=00, segments=60 | index=1, select=00, segments=60 | blank 구간에서도 enable=0이면 유지. |
| 56 ns | enable=1 · 에지 55 ns | index=1, select=02, segments=60 | index=1, select=02, segments=60 | 자리 1 활성(02), 숫자 1 패턴 60. |
| 296 ns | position=7 활성 · 에지 295 ns | index=7, select=80, segments=E0 | index=7, select=80, segments=E0 | 마지막 자리 7, 숫자 7 패턴 E0. |
| 336 ns | 다음 lap의 position=0 · 에지 335 ns | index=0, select=01, segments=FC | index=0, select=01, segments=FC | 경계: index가 7에서 0으로 순환. |
| 1306 ns | enable=1, rst=1 · 에지 1305 ns | index=0, select=00, segments=FE | index=0, select=00, segments=FE | 스캔 중 리셋: select=00, index=0. 이때 digits는 FEDCBA98이라 index 0의 패턴은 숫자 8(FE). |

표의 값은 상승 에지 직후 안정된 값이다. `LAB2_PASS`만 적지 않고 각 행에서 입력, 이전 상태, 다음 상태를 비교한다. 파형 캡처에서 위 시각을 확대해 본인 화면으로 확인한다.

## 코드 수정·실패·복구 실험

- 변경: 자리 전환 blanking을 제거한다(`select = blank ? 8'h00 : (8'h01 << index);` → `select = (8'h01 << index);`).
- 변경한 파일과 위치: `src/segment_scan8.v` 24행
- 테스트벤치 기대값은 바꾸지 않는다.

```diff
-    select = blank ? 8'h00 : (8'h01 << index);
+    select = (8'h01 << index);
```

실행 전 계산: 이 변형은 자리 사이의 blank뿐 아니라 리셋 직후의 blank도 없앤다. 리셋 에지 뒤 6 ns에 TB가 `select===0`을 기대하지만 변경 회로는 index=0이므로 select=01이 되어, 자리 전환 검사(16 ns 이후)보다 먼저 리셋 검사가 실패한다.

| 단계 | 소스 커밋 또는 해시 | 실행 폴더·로그 링크 | 입력·기대값·실제값 | 해석 |
|---|---|---|---|---|
| 정상 코드 | `d71b650` | [normal.log](../../evidence/pre/lab2_08_normal.log) | 6 ns 기대 select=00, 실제 select=00. `LAB2_PASS segment_scan8 checks=194` | 모든 검사 통과, 1306 ns 종료. |
| 지정한 RTL 변경 | 미커밋 수정본(`d71b650` 기준, 로컬 실행) | [mod.log](../../evidence/pre/lab2_08_mod.log) | 6 ns 기대 select=00, 실제 select=01. `LAB2_FAIL reset blanks digit zero time=6000`, `FATAL: sim/tb_segment_scan8.sv:12: check failed` | `reset blanks digit zero` 검사가 변경을 발견했다(로그의 time은 ps 단위, 6000 ps = 6 ns). |
| 원래 코드로 복구 | `d71b650` | [recover.log](../../evidence/pre/lab2_08_recover.log) | 복구 후 전체 검사 재실행. `LAB2_PASS segment_scan8 checks=194`, `$finish called at 1306000 (1ps)` | PASS와 종료 시각이 정상 실행과 같고 새 VCD를 확인한다. |

- 첫 실패 이후에는 `$fatal`로 시뮬레이션이 끝나므로 뒤의 검사는 실행되지 않는다. 변경 전후 파형은 각각 별도 폴더에 보관한다.
- 문법 오류를 경험했다면 오류 위치로 이동한 화면, 원인, 수정 내용과 재실행 로그도 이 절에 연결한다.

## 보드 실험 계획

- 부품·프로젝트: Vivado 2026.1, RTL Project `lab2_segment_scan`, 부품 `xc7s75fgga484-1`(정확히 -1). Design Sources: `segment_scan8.v`, `input_frontend.v`, `lab2_segment_scan.v`(Copy sources 해제). Simulation Sources: `tb_segment_scan8.sv`(Set as Top: `tb_segment_scan8`). Constraints: `lab2_segment_scan.xdc`. Project Summary의 Top module name은 `lab2_segment_scan`이다.
- 장비 클록·제약: Combo II-DLD S75 주 클록 B6을 1 kHz로 맞추고 XDC의 `trainer_1khz`(1,000,000 ns)와 일치시킨다.
- 입력: DIPSW1..4=`sw[7:4]`=첫 자리 숫자, K4=리셋. N8은 스캔 진행에 필요하지 않다. 주 클록은 1 kHz.
- 출력: LED[2:0]=현재 index, LED[7:3]=0.

| 조작 | 예상 LED / 동작 |
|---|---|
| K4 초기화 후 SW1..4=1001 | COM[7]부터 9, 1, 2, 3, 4, 5, 6, 7 표시 |
| SW1..4를 바꿈 | 첫 자리(index 0)만 변화 |
| LED[2:0] | 현재 index (빠른 변화라 눈으로 모두 구분 불가) |

1 kHz 클록에서 슬롯당 blank 포함 2클록이라 16클록에 한 바퀴(약 16 ms)이다. 카메라 줄무늬나 일부 자리 누락은 촬영 주기와 스캔 주기의 차이일 수 있다.

촬영할 장면: 보드 전체(배선·입력·출력이 함께 보이는 사진)와 위 표의 조작별 LED 상태 사진·영상. 예상되는 차이: 버튼·스위치는 동기화·안정 확인 지연이 있고, 빠른 신호는 눈으로 구분되지 않을 수 있다.

이 단계에서는 Vivado GUI와 실물 보드의 결과를 수행한 것처럼 기록하지 않는다. 합성·구현·bit 생성과 실제 장치 기록은 실험 후 레포트에서 다룬다.
