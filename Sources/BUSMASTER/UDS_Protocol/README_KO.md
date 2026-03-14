# UDS 진단 프로토콜 모듈 (한글 설명)

## 개요

이 모듈은 **UDS (Unified Diagnostic Services, ISO 14229)** 및 **KWP2000** 진단 프로토콜을 구현합니다.
BUSMASTER 애플리케이션의 플러그인 DLL 형태로 동작하며, CAN 버스를 통해 차량 ECU(전자 제어 장치)와 진단 통신을 수행합니다.

---

## 디렉토리 구조

```
UDS_Protocol/
├── UDS_Protocol.cpp / .h      ← 핵심 프로토콜 엔진 (DLL 진입점, 메시지 평가)
├── UDSMainWnd.cpp / .h        ← 메인 진단 UI 창
├── UDSSettingsWnd.cpp / .h    ← 설정 창 (주소, CAN ID, 타이밍 설정)
├── UDS_NegRespMng.cpp / .h    ← 부정 응답 코드 관리
├── UDS_TimmingWnd.cpp / .h    ← 타이밍 설정 창 (P2, STMin 등) ※ 원본 파일명에 오타 있음(Timming)
├── UdsMsgBlocksView.cpp / .h  ← 메시지 블록 뷰어
├── UDS_DataTypes.h            ← 열거형 데이터 타입 정의
├── UDSWnd_Defines.h           ← 상수 및 구조체 정의
├── UDS_Extern.h               ← DLL 외부 인터페이스 (C API)
└── UDS_Resource.h             ← UI 리소스 ID 정의
```

---

## 주요 데이터 타입 (`UDS_DataTypes.h`)

### `UDS_INTERFACE` — CAN 인터페이스 유형

| 값 | 설명 |
|---|---|
| `INTERFACE_NORMAL_11` | 일반 11비트 CAN ID 주소 방식 |
| `INTERFACE_EXTENDED_11` | 확장 11비트 CAN ID 주소 방식 (SA/TA를 CAN ID에 포함) |
| `INTERFACE_NORMAL_ISO_29` | ISO 표준 29비트 CAN ID 주소 방식 |
| `INTERFACE_NORMAL_J1939_29` | J1939 규격의 29비트 CAN ID 주소 방식 |

### `DIAGNOSTIC_STANDARD` — 진단 표준 선택

| 값 | 설명 |
|---|---|
| `STANDARD_UDS` | ISO 14229 UDS 표준 |
| `STANDARD_KWP2000` | ISO 14230 KWP2000 표준 |

### `TYPE_OF_FRAME` — ISO-TP 프레임 유형

| 값 | 설명 |
|---|---|
| `LONG_RESPONSE` | 긴 응답의 첫 번째 프레임 (First Frame, 0x1X) |
| `CONSECUTIVE_FRAME` | 연속 프레임 (Consecutive Frame, 0x2X) |
| `SIMPLE_RESPONSE` | 단순 응답 (Single Frame) |
| `FLOW_CONTROL` | 흐름 제어 메시지 (Flow Control, 0x3X) |

---

## 핵심 클래스 및 함수 설명

### 1. `CUDS_Protocol` (UDS_Protocol.cpp / .h)

DLL 의 핵심 관리 클래스입니다. `CWinApp`을 상속받아 MFC 애플리케이션으로 동작합니다.

#### 주요 멤버 변수

| 변수 | 설명 |
|---|---|
| `SourceAddress` | 진단 도구(Tester)의 주소 (SA) |
| `TargetAddress` | 대상 ECU의 주소 (TA) |
| `MsgID` | 기본 CAN 메시지 ID (기본값: 0x700) |
| `fInterface` | 선택된 CAN 인터페이스 유형 |
| `fDiagnostics` | 선택된 진단 표준 (UDS 또는 KWP2000) |
| `numberOfBytes` | 응답 데이터 행에 표시할 바이트 번호 |
| `Data_Recibida` | 수신된 모든 바이트 데이터 누적 문자열 (스페인어 변수명: "수신된 데이터") |

#### 주요 함수

**`getFrameType(BYTE FirstByte)`**
- 수신된 메시지의 첫 번째 바이트를 분석하여 ISO-TP 프레임 유형을 반환합니다.
- `0x30`~`0x32`: Flow Control (흐름 제어)
- `0x10`~`0x1F`: Long Response (긴 응답의 첫 프레임)
- `0x20`~`0x2F`: Consecutive Frame (연속 프레임)
- 그 외: Simple Response (단순 응답)

**`Show_ResponseData(psMsg[], Datalen, posFirstByte)`**
- ECU로부터 수신한 응답 바이트를 UI의 '응답 데이터' 섹션에 16진수로 표시합니다.
- 한 행에 `NUM_BYTES_SHOWN_RESP`개 바이트를 표시하고 자동 줄바꿈합니다.

**`StartTimer_Disable_Send()` / `KillTimer_Able_Send()`**
- ECU 응답을 기다리는 동안 Send 버튼을 비활성화(타이머 시작) / 활성화(타이머 종료)합니다.
- 기본 대기 시간: `P2_Time = 250ms`, `P2_Time_Extended = 2000ms`

**`UpdateFields()`**
- 설정 창에서 변경된 값(SA, TA, CAN ID, 인터페이스 유형)을 메인 창에 반영합니다.

---

### 2. DLL 외부 API 함수 (`UDS_Extern.h`)

BUSMASTER 메인 프레임에서 호출하는 C 인터페이스 함수들입니다.

| 함수 | 설명 |
|---|---|
| `UDS_Initialise()` | UDS 모듈을 초기화합니다 |
| `DIL_UDS_ShowWnd(hParent, TotalChannels)` | UDS 메인 진단 창을 화면에 표시합니다 |
| `DIL_UDS_ShowSettingWnd(hParent)` | UDS 설정 창을 화면에 표시합니다 |
| `EvaluateMessage(STCAN_MSG)` | CAN 버스에서 수신된 메시지를 평가하여 응답 처리를 수행합니다 |
| `UpdateChannelUDS(hParent)` | 메인 창의 채널 정보를 업데이트합니다 |
| `TX_vSetDILInterfacePtrUDS(ptrDILIntrf)` | CAN 드라이버 인터페이스 포인터를 설정합니다 |

---

### 3. `EvaluateMessage()` — 메시지 수신 처리 흐름

CAN 버스로 들어오는 모든 메시지를 평가하는 핵심 함수입니다.

```
메시지 수신
   │
   ├─ 조건 확인: MainWnd가 생성됨 && FSending(전송 중) && 올바른 CAN ID && 올바른 채널
   │
   ├─ getFrameType() 호출 → 프레임 유형 판별
   │
   ├─ [FLOW_CONTROL] 흐름 제어 수신
   │     ├─ 0 (ContinueToSend): BSizE, STMin 파싱 후 연속 프레임 전송 재개
   │     ├─ 1 (Wait): P2_Time만큼 타이머 재시작 (대기)
   │     └─ 기타: 흐름 제어 대기 해제
   │
   ├─ [LONG_RESPONSE] 긴 응답 첫 프레임 수신
   │     ├─ 긴 응답 길이 계산
   │     ├─ Flow Control 메시지 전송 (PrepareFlowControl)
   │     ├─ 응답 데이터 화면 표시 (Show_ResponseData)
   │     └─ 수신해야 할 연속 프레임 수 계산
   │
   ├─ [CONSECUTIVE_FRAME] 연속 프레임 수신
   │     ├─ TotalFrames 및 Counter_BSize 감소
   │     ├─ 타이머 재시작
   │     ├─ 마지막 프레임이면 FSending=FALSE, Send 버튼 활성화
   │     └─ Counter_BSize가 0이 되면 Flow Control 재전송
   │
   └─ [SIMPLE_RESPONSE] 단순 응답 수신
         ├─ evaluateResp()로 긍정/부정 응답 판별
         ├─ 응답 데이터 화면 표시
         └─ FSending=FALSE (전송 완료)
```

---

### 4. `CUDSMainWnd` — 메인 진단 UI 창 (`UDSMainWnd.cpp`)

진단 요청을 전송하고 ECU 응답을 표시하는 메인 UI 창입니다.

#### 주요 UI 요소

| 요소 | 설명 |
|---|---|
| `m_omSourceAddress` | 진단 도구 소스 주소 (SA) 입력란 |
| `m_omTargetAddress` | 대상 ECU 타겟 주소 (TA) 입력란 |
| `m_omCanID` | CAN ID 입력란 |
| `m_omEditDLC` | DLC(데이터 길이 코드) 입력란 |
| `m_omEditMsgData` | 전송할 메시지 데이터 입력란 (16진수) |
| `m_omDiagService` | 응답 설명 표시 텍스트 |
| `m_abDatas` | 수신된 응답 바이트 표시 영역 |
| `m_omCheckTP` | Tester Present 체크박스 |
| `m_omSendButton` | Send 버튼 |

#### 주요 함수

**`PrepareDiagnosticMessage()`**
- 사용자가 입력한 데이터를 분석하여 메시지 크기에 따라 전송 방식을 결정합니다.
  - 단순 메시지 (≤ 7바이트): `SendSimpleDiagnosticMessage()` 호출
  - 긴 메시지 (> 7바이트): `SendFirstFrame()` + `SendContinuosFrames()` 호출

**`SendFirstFrame()`**
- 긴 요청의 첫 번째 프레임을 전송합니다.
- 첫 번째 바이트: `0x1X` (X: 데이터 길이의 상위 니블)

**`SendContinuosFrames()`**
- 긴 요청의 연속 프레임들을 STMin 간격으로 전송합니다.
- 첫 번째 바이트: `0x20`, `0x21`, `0x22`, `0x23`, ... (순환)

**`PrepareFlowControl()`**
- ECU로부터 긴 응답을 받을 때 Flow Control 메시지를 ECU에 전송합니다.

**`OnBnClickedTesterPresent()`**
- Tester Present (0x3E) 주기적 전송을 시작/중지합니다.
- `FCRespReq=TRUE`이면 0x3E 0x80 형태로 전송 (응답 불필요 옵션 포함)

**`initialEval(CString Data2Send)`**
- 전송할 메시지가 응답을 기다려야 하는지 판별합니다.
- `0x3E 0x80` (No Positive Response Required)이면 응답 대기하지 않습니다.

---

### 5. `CUDS_NegRespMng` — 부정 응답 코드 관리 (`UDS_NegRespMng.cpp`)

ECU로부터 수신한 응답 메시지의 두 번째 바이트가 `0x7F`일 때 부정 응답으로 판단하고,
네 번째 바이트(NRC: Negative Response Code)에 해당하는 설명 문자열을 반환합니다.

#### `evaluateResp(arrayMsg[], Byte2Init)` 로직

```
응답 메시지 분석 (arrayMsg[Byte2Init+1] 기준)
   │
   ├─ 0x7F: 부정 응답 (Negative Response)
   │     ├─ arrayMsg[Byte2Init+2] == 현재 서비스 ID?
   │     │     ├─ YES: VerifyNegResponse(NRC 코드)로 설명 반환
   │     │     │       NRC=0x78이면 응답 대기 재시작 (Response Pending)
   │     │     └─ NO:  FDontShow=TRUE (다른 서비스의 응답이므로 무시)
   │     └─
   │
   ├─ 서비스ID+0x40: 긍정 응답 (Positive Response)
   │     └─ "     Positive Response" 반환 (녹색 표시)
   │
   └─ 기타: FDontShow=TRUE (관련 없는 메시지)
```

#### UDS 표준 부정 응답 코드 (NRC) 목록

| NRC 코드 | 설명 |
|---|---|
| `0x10` | 일반 거부 (General Reject) |
| `0x11` | 서비스 미지원 (Service Not Supported) |
| `0x12` | 서브기능 미지원 (SubFunction Not Supported) |
| `0x13` | 잘못된 메시지 길이/형식 (Incorrect Message Length Or Invalid Format) |
| `0x21` | 재요청 필요 - 처리 중 (Busy Repeat Request) |
| `0x22` | 조건 불충족 (Conditions Not Correct) |
| `0x24` | 요청 순서 오류 (Request Sequence Error) |
| `0x31` | 요청 범위 초과 (Request Out Of Range) |
| `0x33` | 보안 접근 거부 (Security Access Denied) |
| `0x35` | 잘못된 키 (Invalid Key) |
| `0x36` | 시도 횟수 초과 (Exceed Number Of Attempts) |
| `0x37` | 시간 지연 미만료 (Required Time Delay Not Expired) |
| `0x70` | 업로드/다운로드 거부 (Upload Download Not Accepted) |
| `0x72` | 일반 프로그래밍 실패 (General Programming Failure) |
| `0x78` | 응답 대기 중 (Response Pending) — 황색으로 표시 |
| `0x7E` | 현재 세션에서 서브기능 미지원 |
| `0x7F` | 현재 세션에서 서비스 미지원 |
| `0x81` | RPM 너무 높음 |
| `0x82` | RPM 너무 낮음 |
| `0x86` | 온도 너무 높음 |
| `0x88` | 차속 너무 높음 |
| `0x92` | 전압 너무 높음 |
| `0x93` | 전압 너무 낮음 |

---

## 전역 변수 설명 (`UDS_Protocol.cpp`)

| 변수 | 기본값 | 설명 |
|---|---|---|
| `FSending` | `FALSE` | 메시지 전송 중 여부 (응답 수신 대기) |
| `FDontShow` | `FALSE` | 현재 메시지를 UI에 표시하지 않을 경우 TRUE |
| `FWaitFlow` | - | Flow Control 메시지를 기다리는 중 여부 |
| `Length_Received` | `0` | 긴 응답에서 수신된 총 바이트 수 |
| `TotalFrames` | - | 수신 예정인 연속 프레임 수 |
| `STMin` | - | 연속 프레임 간 최소 분리 시간 (ms) |
| `BSize` | - | 설정 창에서 사용자가 지정한 블록 크기 |
| `BSizE` | - | Flow Control 메시지로부터 ECU가 보낸 블록 크기 (대소문자 혼용은 원본 코드 그대로) |
| `P2_Time` | `250` | ECU 응답 대기 시간 (ms) |
| `P2_Time_Extended` | `2000` | ECU 확장 응답 대기 시간 (ms) |
| `respID` | - | 응답 메시지의 예상 CAN ID |
| `CurrentService` | - | 현재 전송한 서비스 ID (16진수 문자열) |
| `BytesShown_Line` | `1` | 현재 응답 행에 표시된 바이트 수 |
| `initialByte` | - | 인터페이스 유형에 따른 첫 데이터 바이트 인덱스 |
| `aux_Finterface` | - | 확장 주소 방식일 때 인덱스 오프셋 (0 또는 1) |

---

## ISO-TP (ISO 15765-2) 통신 흐름

UDS는 7바이트 초과 데이터 전송을 위해 ISO-TP(CAN Transport Protocol)를 사용합니다.

### 단순 메시지 (≤ 7바이트)
```
Tester → ECU:  [길이][SID][데이터...]
ECU → Tester:  [길이][SID+0x40][데이터...]  (긍정 응답)
           또는 [03][7F][SID][NRC]           (부정 응답)
```

### 긴 메시지 (> 7바이트) — 긴 요청
```
Tester → ECU:  [1X][길이하위][SID][데이터...]  (First Frame)
ECU → Tester:  [30][BS][STMin]                  (Flow Control - ContinueToSend)
Tester → ECU:  [21][데이터...]                  (Consecutive Frame 1)
Tester → ECU:  [22][데이터...]                  (Consecutive Frame 2)
               ... (BS개 프레임마다 Flow Control 요청)
```

### 긴 응답 (> 7바이트) — 긴 응답
```
Tester → ECU:  [길이][SID][데이터...]           (Single Frame 요청)
ECU → Tester:  [1X][길이하위][SID+0x40][데이터] (First Frame)
Tester → ECU:  [30][BS][STMin]                  (Flow Control - ContinueToSend)
ECU → Tester:  [21][데이터...]                  (Consecutive Frame 1)
ECU → Tester:  [22][데이터...]                  (Consecutive Frame 2)
               ...
```

---

## UI 응답 색상 코드

| 색상 | 의미 |
|---|---|
| 🟢 녹색 (`RGB(0,100,0)`) | 긍정 응답 (Positive Response) |
| 🔴 빨간색 (`RGB(255,0,0)`) | 부정 응답 (Negative Response) |
| 🟡 황금색 (`RGB(184,134,11)`) | 응답 대기 중 (NRC 0x78 - Response Pending) |

---

## 주요 상수 (`UDSWnd_Defines.h`)

| 상수 | 값 | 설명 |
|---|---|---|
| `MASK_SA_ID_29Bits` | `0x3FF800` | 29비트 ID에서 SA 추출 마스크 |
| `MASK_TA_ID_29Bits` | `0x7FF` | 29비트 ID에서 TA 추출 마스크 |
| `NEG_MASK_SA_ID_29Bits` | `0x1FC007FF` | 29비트 ID에서 SA 변경용 역 마스크 |
| `MAX_VALUE_29BIT_ID` | `0x1FFFFFFF` | 29비트 CAN ID 최대값 |
| `TIME_UNABLE_SEND_BUTTON` | `10` | Send 버튼 비활성화 타이머 기본값 |
