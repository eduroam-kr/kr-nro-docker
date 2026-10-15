# User-Name 위생 검사

eduroam 은 **누구나 AP 에 붙어 임의의 EAP identity 를 보낼 수 있다.** 그 값이 그대로 NRO 까지 프록시되므로 입력 검증이 필요하다.

> 다만 1차 방어선은 `clients.conf` 다. 등록되지 않은 출발지에서 온 패킷은
> **조용히 폐기**된다. 즉 공격자는 어느 캠퍼스 AP 뒤에 있어야 한다.

## 검사 순서

`sites-enabled/default` 의 `authorize` 맨 앞에서 순서대로 돈다.

```
filter_username     → upstream 표준 검사
kr_sanity_username  → 추가 검사 (policy.d/kr-nro-sanity)
preprocess
suffix              → realm 분리·조회
```

`utf8` 은 **2026-10-01 에 뺐다.** 아래 참조.

## upstream `filter_username` 이 잡는 것

`policy.d/filter` 에 있다. 이미 상당 부분을 막는다.

| 패턴 | 예 |
|---|---|
| 공백 포함 | `us er`, `user@ site.com`, ` user` |
| `@` 여러 개 | `user@site.com@site.com` |
| 연속된 점 | `user@site..com` |
| realm 에 점이 없음 | `user@site` |
| realm 이 점으로 끝남 | `user@site.com.` |
| realm 이 점으로 시작 | `user@.site.com` |

## 추가로 막는 것 (`kr_sanity_username`)

| 검사 | 이유 |
|---|---|
| `User-Name` 없음 | 비정상 요청 |
| **제어문자** `[\x00-\x1f\x7f]` | `filter_username` 의 공백 검사는 **스페이스만** 본다. 탭·개행·널은 통과한다 |
| **realm 에 `[^a-zA-Z0-9.-]`** | 도메인에 따옴표·괄호·역슬래시·세미콜론이 있을 이유가 없다 |
| realm 253자 초과 | DNS 한계 |
| 공격 패턴 (`../`, `<script`, `union select`, `' or '`) | 스캐너 · 인젝션 시도 |

## `utf8` 을 뺀 이유

`utf8` 모듈은 **모든 문자열 속성**을 검사해 잘못된 UTF-8 이 하나라도 있으면 요청 전체를 거절한다. 사유를 남기지 않아 `Module-Failure-Message` 도 비어 있고, 로그에는 `reason=none` 으로만 보인다. 원인을 찾기가 매우 어렵다.

실측: 이탈리아 한 대학의 AP 가 Latin-1 로 인코딩된 속성을 보내, **그 AP 를 쓴 모든 사용자의 인증이 7일간 100% 실패**했다(106건, 성공 0). `User-Name` 은 멀쩡했고 SSID·장비명 때문이었다.

중계자가 SP 의 인코딩을 이유로 사용자 인증을 끊어서는 안 된다. 그래서 뺐다.

이름 자체의 검증은 `filter_username` 과 `kr_sanity_username` 이 그대로 한다. 깨진 바이트가 로그 JSON 을 깨뜨리는 문제는 `linelog_json` 의 urlquote 로 막는다.

## 로컬파트의 역슬래시

Windows 도메인 형식을 그대로 보내는 단말이 있다. 실측(2026-09-28): 22만 건 중 3건, 한 명이었다.

```
win\sjohndoe@example.dk      ← 로컬파트 앞에 DOMAIN\ 이 붙어 온다
```

RFC 7542 에서 역슬래시는 이스케이프 문자이고 `\s` 는 유효한 이스케이프가 아니다. 홈서버도 어차피 거절할 값이다. 그런데 **우리 구조화 로그에서 그 줄이 통째로 파싱 불가가 된다.** JSON 에서 `\s` 가 유효하지 않은 이스케이프라, Loki 와 ClickHouse 양쪽에서 조용히 유실된다. 그래서 여기서 끊는다.

정규식은 16진 표기 `[\x5c]` 로 쓴다. `/\\\\/` 는 시험대에서 매칭되지 않았다.

## reject 인가 drop 인가

기본은 **reject** 다. 이유는 직관과 반대다.

- **drop(무응답)** 하면 클라이언트가 **재전송**한다. 같은 쓰레기가 3~5회 더 온다. 트래픽을 줄이려는 목적이라면 역효과다.
- **reject** 는 대화를 한 번에 끝낸다.

그래서 `do_not_respond` 는 **명백한 공격 패턴에만** 쓴다. 이 경우는 상대가 정상 클라이언트가 아니라 스캐너이므로, 응답을 주면 "여기 RADIUS 가 있다"는 확인만 해 주는 셈이다.

```
do_not_respond      # policy.d/control — Response-Packet-Type := Do-Not-Respond
```

## 막으면 안 되는 것

**`@realm` (로컬파트 없음)은 정상이다.** eduroam 의 익명 외부 식별자 (anonymous outer identity) 형식이며 매우 흔하다. 차단하면 대량 장애가 난다.

```
@example.ac.kr        정상 — 익명 외부 식별자
anonymous@example.ac.kr 정상
```

대소문자 혼용(`UseRNaMe`)도 막지 않는다. upstream 에 검사가 있지만 주석 처리된 상태이며, realm 매칭은 어차피 대소문자를 무시한다.

## 확인

```bash
alias radclient='docker exec -i kr-nro-freeradius /opt/bin/radclient'

# 정상 — Access-Reject (경로 정상)
echo "User-Name=user@example.ac.kr,User-Password=x" | radclient -x 127.0.0.1:1812 auth radsec

# 공백 — filter_username 에서 거절
echo "User-Name=pro be@test.kr,User-Password=x" | radclient -x 127.0.0.1:1812 auth radsec

# realm 이상문자 — kr_sanity_username 에서 거절
echo "User-Name=user@example.ac.kr;drop,User-Password=x" | radclient -x 127.0.0.1:1812 auth radsec
```

`radiusd -X` 로 띄우면 `Module-Failure-Message` 에 어느 검사에서 걸렸는지 찍힌다.

## 실환경 검증 결과 (2026-09, FreeRADIUS 3.2.10)

`.49` 에서 실제로 돌려 확인했다.

- **`strlen()` 은 조건문에서 파싱되지 않는다.** `Parse error in condition` 으로 기동이 실패했다. 정규식 길이 제한(`/@.{254,}$/`)으로 교체했다.
- `%{1}` 정규식 캡처는 정상 동작한다.
- 블랙홀 realm 의 Reply-Message 가 Access-Reject 에 실려 오는 것을 확인했다.

```
user@skku.edu     → Access-Reject  "eduroam KR: use @skku.ac.kr (not @skku.edu)"
user@ac.kr        → Access-Reject  "eduroam KR: realm must identify your institution..."
user@isel.local   → Access-Reject  "eduroam KR: realm is not routable"
```

## 남은 확인 사항

- 공격 패턴 목록은 실제 로그를 보며 조정한다. 과도하게 넓히면 정상 사용자가 걸린다.
- 위 Reply-Message 는 **비-EAP** 요청 기준이다. EAP 요청에서는 `remove_reply_message_if_eap` 이 제거하므로 전달되지 않는다. → `blackhole.md`
