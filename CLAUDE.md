# GameServerProject - 코드 분석 문서

## 프로젝트 구조

```
[Client (SFML)] <--TCP--> [RPGServer] <--TCP--> [DB_Server] <--ODBC--> [SQL Server]
```

세 바이너리가 별도 프로세스로 실행되며 TCP 소켓으로 통신한다.

---

## 공통 사항

- **언어/플랫폼**: C++, Windows 전용 (IOCP, WSA, MSWSock)
- **비동기 모델**: Windows IOCP (`CreateIoCompletionPort` + `GetQueuedCompletionStatus`)
- **패킷 구조**: 첫 2바이트 = 전체 크기(short), 3번째 바이트 = 타입(unsigned char)
- **프로토콜 파일**: `protocol_2023.h` (Client↔RPGServer), `DBprotocol.h` (RPGServer↔DB_Server)

---

## RPGServer

### 파일 구성

| 파일 | 역할 |
|------|------|
| `main.cpp` | 서버 초기화, NPC/Zone/Map 초기화, 스레드 시작 |
| `NetWork.h/cpp` | IOCP 이벤트 처리, 패킷 처리, Lua API, Zone 검색 |
| `WorkThread.cpp` | IOCP 워커 스레드 루프 |
| `TimerThread.cpp` | 타이머 이벤트 → IOCP 디스패치 |
| `Object.h/cpp` | 게임 오브젝트 베이스 클래스 |
| `Player.h/cpp` | 플레이어 (패킷 송수신, 전투, 레벨업) |
| `DefaultNPC.h/cpp` | NPC AI (이동, 전투, 부활) |
| `Zone.h/cpp` | 공간 분할 존 시스템 (Lock-free 링크드 리스트) |

### 핵심 전역 변수

```cpp
std::array<Object*, MAXOBJECT> objects;       // [0, MAX_USER) = 플레이어, [MAX_USER, ~) = NPC
std::array<std::array<ZoneManager*, ZONE_Y+1>, ZONE_X+1> zone;  // 2D 존 그리드
std::map<std::pair<short,short>, short> World_Map;               // 벽 타일 좌표
concurrency::concurrent_priority_queue<EVENT> timer_queue;       // 타이머 이벤트 큐
std::set<int> login_player;      // 현재 로그인 중인 DB ID 집합 (login_lock으로 보호)
SOCKET DB_socket;                // DB_Server 연결 소켓 (단일)
```

### 스레딩 구조

- **워커 스레드**: `hardware_concurrency()` 개, 모두 동일 IOCP에서 이벤트 처리
- **타이머 스레드**: `TimerThread()` 1개, `timer_queue`에서 이벤트를 꺼내 `PostQueuedCompletionStatus`로 워커에 위임

### IOCP 오퍼레이션 목록

```
OP_ACCEPT          - 클라이언트 접속
OP_RECV / OP_SEND  - 클라이언트 수신/송신
OP_NPC_RANDOMMOVE  - NPC 랜덤 이동 (평화 NPC)
OP_NPC_MOVE        - NPC 추적 이동 (전투 NPC)
OP_NPC_ATTACK      - NPC 근접 공격
OP_NPC_RANGEATTACK - NPC 원거리 공격
OP_NPC_DEFENCE     - NPC 피격 (플레이어 공격 처리)
OP_NPC_HEAL        - 오브젝트 HP 회복
OP_NPC_RESPAWN     - NPC/플레이어 리스폰
OP_NPC_WAIT        - NPC 대기 상태
DB_RECV / DB_SEND  - DB_Server 수신/송신
```

### 패킷 흐름

**클라이언트 → RPGServer (CS_)**
- `CS_LOGIN`: 이름 = "p{db_id}" 형식, 첫 글자 제거 후 int 변환
- `CS_MOVE`: direction (0=위, 1=아래, 2=왼쪽, 3=오른쪽)
- `CS_ATTACK`: 시야 내 모든 NPC에 `OP_NPC_DEFENCE` 발송
- `CS_CHAT`: 시야 내 플레이어에게 채팅, DB에 로그 저장

**RPGServer → 클라이언트 (SC_)**
- `SC_LOGIN_INFO`: 로그인 성공 후 내 캐릭터 정보
- `SC_ADD_OBJECT` / `SC_REMOVE_OBJECT`: 시야 범위 진입/이탈
- `SC_MOVE_OBJECT`: 오브젝트 이동
- `SC_ATTACK` / `SC_ATTACK_RANGE`: 공격 이펙트
- `SC_HP_CHANGE`: HP 변경 알림
- `SC_LOGIN_FAIL`: 중복 로그인 거부

**RPGServer ↔ DB_Server (SD_ / DS_)**
- `SD_PLAYER_LOGIN` → `DS_PLAYER_LOGIN`: 로그인 DB 조회/생성
- `SD_PLAYER_CHANGE_STAT`: 스탯(exp, level, hp) 저장
- `SD_PLAYER_LOCATION`: 위치 저장 (20회 이동마다)
- `SD_CHAT`: 채팅 로그 저장

### 플레이어 로그인 흐름

```
1. 클라이언트 접속 → OP_ACCEPT → objects[p_id]._state = ST_ALLOC
2. CS_LOGIN 수신 → DB_player_login() → SD_PLAYER_LOGIN을 DB_Server로 전송
3. DS_PLAYER_LOGIN 수신 → 위치 설정, 존 등록, _state = ST_INGAME
4. send_login_info_packet() → 클라이언트에 캐릭터 정보 전송
5. 시야 내 오브젝트 SC_ADD_OBJECT 전송
```

### NPC AI 시스템

**NPC 타입**
```
타입 1: 평화 + 이동 (랜덤 이동, 스폰 반경 10 이내)
타입 2: 평화 + 정지 (제자리 대기, 원거리 공격 가능)
타입 3: 공격적 + 이동 (플레이어 추적, 근접 공격)
타입 4: 공격적 + 정지 (제자리에서 원거리 공격)
```

**Lua 연동** (`npc.lua` 파일 + C++ API)
```cpp
// Lua에서 호출 가능한 C++ 함수
API_get_x(npc_id)          → NPC x좌표 반환
API_get_y(npc_id)          → NPC y좌표 반환
API_SendMessage(my_id, msg) → 시야 내 플레이어에게 채팅
API_Defence(atk_id, def_id) → NPC가 피격 처리 (HP 감소, 사망 체크)
API_Default_Attack(atk, def) → NPC 근접 공격
API_Range_Attack(atk, def, time) → NPC 원거리 공격

// Lua에서 정의해야 할 함수
event_object_Defence(cause_id)  → 피격 시 행동
event_object_Attack(target_id)  → 공격 시 행동
event_range_Attack(target_id)   → 범위 공격 시 행동
event_NPC_Attack_msg()          → 공격 메시지 출력
```

**NPC 수면/기상 (CAS 기반)**
```cpp
// _n_wake: volatile bool (0=수면, 1=기상)
// 플레이어가 시야 진입 시 CAS로 0→1 성공한 스레드만 wake_up_npc() 호출
if (CAS(&nl->_n_wake, 0, 1))
    wake_up_npc(n_id);
// 시야 내 플레이어가 없으면 CAS로 1→0 후 이벤트 루프 종료
```

### 존(Zone) 시스템

- 월드를 `ZONE_SEC` 크기 격자로 분할
- 각 `ZoneManager`는 **Lock-free 링크드 리스트** (`Zone_PTR`의 1비트 mark로 논리적 삭제)
- `zone_check(x, y, z_list)`: 현재 존 + VIEW_RANGE에 걸치는 인접 존 최대 4개 조회
- 이동 시 존 변경 여부 확인: `my_zone != b_my_zone`이면 REMOVE → ADD

### 이동 속도 제한

```cpp
if (player->_p_last_move_time < (now - 100ms))  // 100ms 쿨다운
    허용
```

### 전투 시스템

- **공격 쿨다운**: 플레이어 공격은 1초 쿨다운 (`_last_attack_time`)
- **데미지 공식**: `dmg = 10 + (level-1) * level / 2` (레벨업마다 `_dmg += level`)
- **경험치**: `exp += npc_level² * 2`, 레벨업 조건: `exp >= level * 100`
- **레벨업**: `max_hp *= 1.1`, `hp = max_hp`, `dmg += level`, `level++`
- **사망**: 경험치 50% 삭감, 30초 후 리스폰, 랜덤 위치
- **NPC 부활**: 사망 5초 후 스폰 위치로 복귀
- **HP 회복**: 5초 간격으로 max_hp의 10%씩 회복 (EV_HEAL 이벤트)

### 동기화 전략

```cpp
Object::_s_lock (shared_mutex)  → 상태(_state) 읽기/변경 보호
Object::_vl (shared_mutex)      → view_list 읽기/변경 보호
login_lock (shared_mutex)       → login_player set 보호
npc->_lua_lock (mutex)          → Lua 스테이트 보호 (수동 lock/unlock)
_n_wake (volatile bool + CAS)   → NPC 수면/기상 원자적 전환
```

---

## DB_Server

### 파일 구성

| 파일 | 역할 |
|------|------|
| `DB_Server.cpp` | 초기화, DB 연결, 워커 스레드 시작 |
| `NetWork.h/cpp` | IOCP 처리, 패킷 처리, DB 쿼리 함수 |
| `WorkThread.cpp` | IOCP 워커 스레드 |

### 핵심 구조

- **DB 연결**: 스레드마다 별도 ODBC 핸들 (`henv[i]`, `hdbc[i]`, `hstmt[i]`)
- **DSN**: "DB_Server", 계정: "2019180046" / "2019180046" (하드코딩)
- **G_server[key]**: 연결된 RPGServer 소켓 맵 (키 = 접속 순서 번호)

### DB 처리 함수

```cpp
player_login(s_id, id, name, key, w_id)
    → user_info에서 id 존재 확인
    → 없으면 EXEC Add_User(id, name)
    → SELECT hp, max_hp, exp, level, x, y FROM user_info
    → DS_PLAYER_LOGIN_PACKET 응답

player_change_state(w_id, id, exp, level, hp, max_hp)
    → EXEC STAT_User(id, exp, level, hp, max_hp)

player_change_location(w_id, id, x, y)
    → EXEC Location_User(id, x, y)

player_chat_log(w_id, id, time, mess)
    → EXEC Chating_Logs(id, time, mess)
```

### DB 에러 처리

- `show_DB_error()`: 에러 출력 후 `SQLDisconnect` → `DB_connect()` 재연결
- `DB_connect()`: 실패 시 `goto retry` 무한 재시도

---

## 알려진 버그 및 문제점

### Critical (데이터 손상/크래시 가능)

1. **[DB_Server] 전역 recv 버퍼 공유 (`NetWork.cpp`)**
   - `_wsa_recv_over`가 전역 단일 인스턴스인데 `do_recv(key)`에서 모든 연결이 공유
   - 멀티 워커 스레드 환경에서 동시에 다른 서버 연결의 데이터를 덮어쓸 수 있음

2. **[RPGServer] DB_RECV 포인터 연산 오류 (`WorkThread.cpp:277`)**
   ```cpp
   char* buf = ex_over->_buf - DB_prev_size;  // DB_prev_size > 0이면 버퍼 앞으로 벗어남
   ```
   - `DB_prev_size`가 0이면 우연히 동작하지만 부분 패킷 발생 시 포인터 오류

3. **[RPGServer] disconnect()에서 존 미제거 (`NetWork.cpp:414`)**
   - 플레이어 연결 해제 시 `zone[y][x]->REMOVE(o_id)` 누락
   - 죽은 소켓 ID가 존에 남아 다른 플레이어의 view_list에 계속 포함됨

4. **[RPGServer] HP CAS 불완전 (`NetWork.cpp:876, 936`)**
   ```cpp
   // max_hp일 때만 CAS 성공, 그 외엔 비원자적 -= 사용 → 레이스 컨디션
   if (CAS(&objects[def_id]->_hp, objects[def_id]->_max_hp, ...)) { ... }
   else { objects[def_id]->_hp -= ...; }  // 원자적이지 않음
   ```

### Medium (기능 버그)

5. **[DB_Server] `disconnect()` 빈 함수 (`NetWork.cpp:86`)**
   - RPGServer 연결 해제 시 아무 처리 없음, G_server 맵에서 미제거

6. **[RPGServer] Lua 락 비RAII (`DefaultNPC.cpp`)**
   - `npc->_lua_lock.lock()` 후 예외 발생 시 unlock 미보장

7. **[RPGServer] disconnect() O(n) 전체 탐색 (`NetWork.cpp:415-441`)**
   - 한 플레이어 접속 해제 시 MAX_USER 전체를 순회하며 remove 패킷 전송
   - view_list 기반으로 변경해야 함

8. **[RPGServer] `send_packet` 메모리 누수**
   - `new WSA_OVER_EX(...)` 후 WSASend 실패 시 delete 없음 (Player.cpp:22, NetWork.cpp:126)

### Low (코드 품질)

9. **`goto` 사용**: `initialize_npc()`, `DB_connect()`, `respawn_player()` 등에서 `while + continue`로 대체 가능
10. **하드코딩된 DB 자격증명**: DB_Server.cpp:337에 평문 저장
11. **로그인 인증 없음**: 이름 파싱(`p{id}`)만으로 로그인, 임의 ID 접속 가능
12. **`DB_Server` 최대 8개 연결 고정**: `_prev_size[8]` 배열 크기

---

## 상수/설정값 위치

- `protocol_2023.h`: `PORT_NUM`, `MAX_USER`, `MAXOBJECT`, `MAXMOVEOBJECT`, `BUF_SIZE`, `NAME_SIZE`, `CHAT_SIZE`, `W_WIDTH`, `W_HEIGHT`, `ZONE_SEC`, `ZONE_X`, `ZONE_Y`
- `DBprotocol.h`: `DB_PORT_NUM`, `DB_THREAD_NUM`, `DB_BUF_SIZE`, `DB_NAME_SIZE`, `DB_SERVER_ADDR`

---

## 빌드 의존성

- `WS2_32.lib`, `MSWSock.lib`: Windows 소켓
- `lua54.lib`: Lua 5.4 (RPGServer)
- ODBC (`sqlext.h`): DB_Server
- PPL (`concurrent_priority_queue`, `concurrent_queue`): Visual Studio 번들
