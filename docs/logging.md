# 인증 로그

사람이 읽는 기본 로그와 기계가 읽는 구조화 로그를 **둘 다** 남긴다.

```
Wed Oct 14 17:46:19 2026 : Auth: (3427609) Login OK: [user@example.ac.kr] (from client 2015_etlr1_eduroam_org port 70017 cli BE-A7-77-BA-A9-72)

{"ts":"2026-10-14 17:46:19","req":"3427609","result":"ok","reason":"","mfm":"",
 "user":"user@example.ac.kr","idp":"example.ac.kr","op":"1visited.example.edu",
 "client":"2015_etlr1_eduroam_org","mac":"BE-A7-77-BA-A9-72",
 "peer":"192.0.2.10","nas_ip":"198.51.100.5","nas_id":"ap-01"}
```

둘을 잇는 열쇠는 `req` 다. 기본 로그의 `Auth: (3427609)` 과 같은 값이다. 서버를 재시작하면 0 부터 다시 시작하므로 전역 유일 키가 아니다. `ts` 와 같이 봐야 한다.

구조화 로그는 `post-auth` 에서만 부른다. EAP 왕복이 아니라 **최종 결정 1건당 1줄**이다.

## 기본 로그가 답하지 못하는 것

`client` 는 우리에게 직접 보낸 **경유지**다. RO 를 거친 요청은 전부 같은 이름으로 찍혀 "어느 캠퍼스에서 접속했는가"에 답할 수 없다. `op`(Operator-Name)와 `nas_ip` / `nas_id` 가 그 자리를 메운다.

`peer` 는 그 요청을 실제로 보낸 주소다. client 이름은 등록 시점의 값이라 기관이 장비를 바꾸면 낡은 이름이 된다. `peer` 는 그 순간의 사실이므로 이름이 낡았을 때 로그가 바로 알려준다.

## 필드

| 필드 | 뜻 |
|---|---|
| `req` | FreeRADIUS 요청 번호. 기본 로그와 조인하는 키 |
| `result` | `ok` / `reject` |
| `reason` | 실패 분류. 아래 표 |
| `mfm` | `Module-Failure-Message` 원문을 urlquote 한 것. 진단용 |
| `user` / `idp` | 사용자와 realm |
| `op` | Operator-Name = 실제 방문기관(SP). `client` 와 다르다 |
| `client` | 우리에게 직접 보낸 곳(경유지) |
| `peer` | 그 요청의 실제 출발 주소 |
| `mac` | `AA-BB-CC-DD-EE-FF` 로 정규화된 Calling-Station-Id |
| `nas_ip` / `nas_id` | `op` 가 비어 있는 건을 나중에 라벨링할 근거 |

## 원문이 아니라 분류값을 남기는 이유

`Module-Failure-Message` 를 그대로 넣으면 안 된다. 메시지에 큰따옴표가 들어가 JSON 이 깨진다.

```
Failing proxied request for user "user@example.ac.kr", due to lack of ...
```

예전에 이것 때문에 **1.3% 가 invalid JSON** 이 되어 `reason` 을 통째로 뺐던 적이 있다. 분류값은 따옴표가 없고, 카디널리티가 낮아 `sum by (reason)` 집계가 되며, 원문에 박힌 사용자명이 `user` 필드와 중복되지 않는다.

원문이 필요할 때를 위해 `mfm` 에 urlquote 로 인코딩해 함께 남긴다. `%22` 가 `"`, `%20` 이 공백이다.

## reason

| 값 | 뜻 |
|---|---|
| `home_reject` | 홈서버가 거절. 대개 비밀번호 오류 |
| `home_timeout` | 홈서버 무응답 |
| `no_home_server` | 그 realm 의 살아 있는 홈서버 없음 |
| `local_reject` | **우리가** 끊음 (blackhole, 서브도메인 거절, 미등록 realm) |
| `sanity` | User-Name 위생 검사에 걸림 |
| `no_auth_type` | 프록시하지 않고 로컬에서 거절 |
| `cond_error` | 정책 조건 평가 실패. 설정 버그 신호 |
| `none` / `other` | 사유 메시지가 없거나 위 어디에도 안 맞음 |

성공일 때는 빈 값이다.

`home_reject` 와 `local_reject` 의 구분이 중요하다. 전자는 상대 기관이 거절한 것이고 후자는 우리가 라우팅을 거부한 것이다. **책임 소재가 정반대다.** "우리가 막는 건가 상대가 막는 건가" 라는 문의에 이 필드 하나로 답할 수 있다.

## 함정 넷

**① `Home Server says so` 는 로그 문구일 뿐이다.** `Module-Failure-Message` 에는 없다. 홈서버 거절은 `&proxy-reply:Packet-Type` 으로 판별해야 한다.

**② 없는 속성은 `"%{...}"` 로 감싸도 조건 평가가 실패한다.** 빈 문자열이 되지 않는다. 반드시 `if (&Attr)` 로 존재를 먼저 확인하고 중첩한다. 존재 확인 자체는 안전하다 — proxy 리스트가 없어도 absent 로 떨어진다.

**③ 조건 평가가 실패하면 원인이 가려진다.** 실패 사실이 다시 `Module-Failure-Message` 를 `Failed retrieving values...` 로 덮어써 원래 원인을 지운다.

**④ `realm { virtual_server = ... }` 는 프록시로 취급된다.** FreeRADIUS 가 **가상서버로의 프록시**로 다루므로 `proxy-reply` 가 채워진다. 그래서 우리가 끊은 건도 `home_reject` 로 분류됐다.

> 실측(2026-10-15): 7일 `home_reject` 8,119건(0.69%)이 이 경우였다.

`auth-reject` 가 `Post-Auth-Type REJECT` **맨 끝**에 표식을 세우고, 분류기가 그것을 먼저 본다. 끝에 두는 것이 요점이다 — `remove_reply_message_if_eap` 와 `attr_filter.access_reject` 보다 앞에 두면 둘 중 하나에 지워진다. 내부 속성이라 회선으로 나가지 않으므로 SP 에게 전달되지도 않는다.

## 운영에서 시행착오하지 않는다

분류기를 고칠 때는 시험대에서 먼저 통과시킨다. 거절 경로 네 가지를 모두 재현한다.

```
reject.test     홈서버가 거절        → home_reject
dead.test       홈서버 무응답        → home_timeout
local.test      프록시 없이 거절      → no_auth_type
vsreject.test   virtual_server 거절  → local_reject
```

경로를 새로 만들면 시험대에도 그 경우를 더한다. `vsreject.test` 가 없던 탓에 ④ 를 오래 못 잡았다.

## 아직 없는 것

`Chargeable-User-Identity` 는 남기지 않는다. MAC 랜덤화와 무관한 사용자 키라 있으면 유용하지만 현재 포맷에 빠져 있다.
