# 블랙홀 라우팅

라우팅이 불가능한 realm 을 ETLR 로 올려보내지 않고 **즉시 거절**한다.

## 왜

이런 요청은 국제 회선과 ETLR 부하를 소모하고, 100% 실패하므로 통계도 오염시킨다. 사용자 입장에서도 타임아웃을 기다리는 대신 바로 거절을 받는 편이 낫다.

## 구성

| 파일 | 역할 |
|---|---|
| `sites-enabled/auth-reject` | 거절 동작을 하는 가상서버 |
| `blackhole.conf` | **기관과 무관한** 블랙홀 (실재하지 않는 TLD, SIM realm 등) |
| `conf.d/<기관>.conf` | **그 기관의** 서브도메인 거절 → `radius-routing.md` |

기관에 딸린 규칙은 그 기관 파일에 둔다. 기관을 지울 때 파일 하나만 지우면 되도록 하기 위해서다.

`proxy.conf` 에서 **`$INCLUDE blackhole.conf`** 로 읽는다. `conf.d/` 가 아니라 별도 파일인 이유는 두 가지다.

- 기관 설정이 아니다 (`conf.d/` 는 기관당 한 파일이라는 규칙)
- **정규식은 나열 순서가 중요**한데 `conf.d/` 는 파일시스템 순서라 보장이 안 된다. `$INCLUDE` 는 위치가 명확하므로 순서를 통제할 수 있다

### 패턴 표기

glob(`*.local`)은 쓸 수 없다. FreeRADIUS realm 은 **정확일치 아니면 정규식**이다. upstream 권장 표기를 따른다.

```
realm "~(.*\.)*local$" {
	virtual_server = auth-reject
	nostrip
}
```

- **큰따옴표**로 감싼다 — 파서가 특수문자를 그대로 넘긴다
- 끝에 **`$`** — 없으면 `mylocalFOO` 같은 중간 일치가 생긴다
- **`(.*\.)*`** — 도메인 자신(`local`)과 모든 하위 도메인(`isel.local`)을 함께 잡는다

정확일치 realm 은 정규식보다 **항상 우선**한다. 그래서 `ac.kr` 을 정확일치로 넣으면 `~(.*\.)*ac\.kr$`(→ KREN) 보다 먼저 걸려 차단된다.

```
realm polito.guest {
	virtual_server = auth-reject
	nostrip
}
```

### 거절 응답에 사유·대안 realm 알려주기

`auth-reject` 는 오타 realm 에 대해 **올바른 realm 을 Reply-Message 로** 돌려준다.

```
@skku.edu    → "eduroam KR: use @skku.ac.kr (not @skku.edu)"
@knu.edu.kr  → "eduroam KR: use @knu.ac.kr (not @knu.edu.kr)"
@ac.kr       → "eduroam KR: realm must identify your institution, e.g. @yourschool.ac.kr"
그 외        → "eduroam KR: realm is not routable"
```

**⚠️ 이 메시지가 사용자 화면에 뜨는 경우는 드물다.** 기대치를 정확히 알고 쓸 것.

| 대상 | 보이나 |
|---|---|
| 최종 사용자 | ❌ 대부분 안 보인다. supplicant(Windows·Android·iOS)가 Reply-Message 를 표시하지 않는다 |
| **방문기관(SP) RADIUS 로그** | ✅ **여기가 주된 효용.** 캠퍼스 담당자가 문의받았을 때 로그에서 원인을 바로 안다 |
| 우리 로그 | ✅ `radiusd -X` 에 찍힌다 |

또한 응답에 `EAP-Message` 가 함께 있으면 **RFC 3579 2.6.5** 에 따라 `remove_reply_message_if_eap` 이 Reply-Message 를 제거한다. eduroam 인증은 대부분 EAP 이므로 그 경우에는 메시지가 전달되지 않는다.

> **참고**: NRO 는 `eap` 모듈을 비활성화했으므로(→ `certificates.md`)
> 거절 응답에 `EAP-Failure` 를 넣지 못한다. Access-Reject 자체는 유효하지만,
> EAP 대화를 깔끔히 끝내려면 EAP-Failure 를 포함하는 것이 이상적이다.
> 단말이 실패를 인지하지 못하고 재시도하는 양상이 보이면 이 점을 검토한다.

realm 별 판정에 `%{Realm}` 대신 **User-Name 정규식**을 쓴다. 정규식 realm 으로 매칭된 경우 `%{Realm}` 값이 구현에 의존하기 때문이다.

### 거절 응답에 사유 남기기

`auth-reject` 는 `Reply-Message` 로 거절 사유를 남긴다. SP 쪽 로그에서 "왜 거절됐나"를 바로 알 수 있어 문의가 줄어든다.

다만 **RFC 3579 2.6.5 에 따라 `EAP-Message` 와 `Reply-Message` 는 같은 응답에 함께 있으면 안 된다.** eduroam 인증은 대부분 EAP 이므로 그대로 두면 규격 위반이다. upstream 정책 `remove_reply_message_if_eap` 을 `Post-Auth-Type REJECT` 에서 호출해 EAP 응답이면 Reply-Message 를 제거한다.

```
post-auth {
	Post-Auth-Type REJECT {
		remove_reply_message_if_eap
		attr_filter.access_reject
	}
}
```

`attr_filter.access_reject` 는 Access-Reject 에 허용되지 않는 속성을 걸러낸다 (허용: `EAP-Message`, `State`, `Message-Authenticator`, `Error-Cause`, `Reply-Message`, `MS-CHAP-Error`, `Proxy-State`).

> 이 정책들은 이미지의 `policy.d/eap` 에 있으며 **속성 검사만 하고 eap 모듈을
> 호출하지 않는다.** NRO 에서 eap 모듈을 비활성화했어도 문제없다.

### 참고 — 더 정교한 형태

FreeRADIUS 메일링 리스트에는 `suffix` 를 다시 돌리고 `switch "%{Realm}"` 으로 realm 별 다른 메시지를 주는 예가 있다. 우리 구성에서는 채택하지 않았다.

- 블랙홀 가상서버에 들어온 시점에 **이미 realm 이 확정**돼 있어 `suffix` 재실행이 불필요하다
- `NULL`·`DEFAULT` case 는 우리 설정에서 이 가상서버로 오지 않는다 (`DEFAULT` 는 ETLR 로 가고, realm 없는 요청은 프록시되지 않는다)

realm 별로 다른 메시지가 필요해지면 그때 `switch` 를 넣는다.

정규식 블랙홀을 `conf.d/` 에 두지 않는 이유는 다른 정규식과의 순서를 통제해야 하기 때문이다(→ `radius-routing.md`). 현재 `~\.3gppnetwork\.org$` 와 `~\.ac\.kr$` 는 겹치지 않아 순서 의존이 없지만, 원칙상 정규식은 한 파일에 모아 둔다.

## 현재 대상

분류는 세 가지다 — 실재하지 않는 TLD(`.local`, `.internal`, `.guest` 등), 도메인 형태가 아닌 이름, 국내 기관의 `.edu` realm. 목록은 `blackhole.conf` 에 있고 건수 실측은 `docs-priv/measurements.md`
| | `kaist` | 124 |
| 구조 오류 | `ac.kr` | 5,233 |
| SIM realm | `*.3gppnetwork.org` | 2,448 |
| **합계** | | **약 33,000** |

모두 **성공 0건**이며, TLD 가 실재하지 않거나 기관을 특정하지 못해 **어떤 경로로도 인증될 수 없는** 것들이다.

> **`.ac.kr` catch-all 과 혼동하지 말 것.** `kren.kr` 등 KREN **조직 자체** realm 은
> 실사용자가 없어 차단하지만, `~^(.*\.)*ac\.kr$` catch-all(→ `kren_ro.conf`)은
> snu·yonsei 등 국내 대학 대부분이 쓰는 살아 있는 경로다.

## 국내 기관의 `.edu` realm

국내 기관이 메일 도메인(`.edu`)을 eduroam realm 으로 쓰는 경우가 있다. **전부 차단한다.**

이유는 성공률이 아니라 **구조**다. gTLD(`.edu`, `.org`)는 realm 만 보고 어느 나라 페더레이션인지 판별할 수 없어, ETLR 라우팅 테이블에 **기관별 예외 등록**이 없으면 해외에서 인증되지 않는다(RFC 7593).

> "realms ending with the same top-level domain are routed to the same server …
> it is not true for realms residing in generic Top-Level Domains (gTLDs)"
> — [RFC 7593](https://datatracker.ietf.org/doc/rfc7593/)

그래서 **우리가 국내에서만 라우팅해 주면 상황이 더 나빠진다.** 사용자는 국내에서 잘 되니 그 realm 으로 설정해 두고, 해외에 나가서야 안 되는 걸 발견한다. 일관되게 막아 `.ac.kr` realm 을 쓰도록 유도하는 편이 낫다.

2026-09 기준 국내 `.edu` 는 **전부 성공 0** 이다. 한 곳은 원래 활성 realm 이었으나 같은 이유로 블랙홀로 옮겼다. 수치는 `docs-priv/measurements.md`

### ⚠️ 해외 `.edu` 는 절대 막지 않는다

`*.edu` 를 통째로 막으면 정상 로밍이 대량으로 깨진다. **반드시 기관 단위로 개별 지정**한다.

미국 주요 대학 몇 곳만으로도 월 1~2만 건씩 성공한다. 수치는 `docs-priv/measurements.md`

### 기관이 `.edu` 를 정식 운영하려면

ETLR 에 **기관별 예외 등록**(realm → KR 페더레이션)이 필요하며, 이 테이블은 공개되지 않아 GÉANT OT 에 직접 요청해야 한다. → `docs-priv/coordination.md`

## ⚠️ 선정 기준 — "성공률 0" 만으로는 부족하다

성공 0 인 realm 중 대다수는 **정상 realm** 이다. 요청이 수만 건인데 성공 0 인 경우도 그렇다. 어떤 realm 은 **설정 문제**로 10만 건이 전부 실패하고 있던 사례도 있다. 사례는 `docs-priv/measurements.md`

원인은 보통 **단말이 옛 자격증명으로 재시도**하는 것이지 realm 이 잘못된 것이 아니다. 블랙홀에 넣으면 실제 로밍이 깨진다. 넣어도 되는 것은 **이름 자체가 라우팅 불가능한 경우**뿐이다.

판별 기준:
- TLD 가 실재하지 않음 (`.guest`, `.local`, `.internal`, `.invalid`, `.lan`)
- 점이 없어 도메인 형태가 아님 (`guest`, `univieguest`, `12345`)
- 기관을 특정하지 못함 (`ac.kr`)
- 이동통신 SIM realm (`*.3gppnetwork.org`) — WiFi 로 새어 들어온 것

### 오타 realm 은 넣지 않는다

국가 TLD 를 잘못 쓴 realm(`.ac.uk` 를 `.ac.kr` 로 적는 류)이 수백 건씩 들어온다. 구조는 유효한 `.ac.kr` 이라 catch-all 을 타고 RO 로 간다.

넣으면 즉시 거절되어 조금 빨라진다. 그러나 **오타 목록을 끝없이 관리해야 하고, 실재하지만 아직 우리에게 등록되지 않은 기관과 충돌한다.** 미등록이 곧 부재는 아니다. 넣지 않는다.

## 블랙홀에 추가할 때

`blackhole.conf` 에 realm 을 더한다. 정규식도 같은 파일에 둔다 — 순서가 정책이기 때문이다.

`radsecproxy/nro-direct-realms.conf` 에는 **넣지 않는다.** 블랙홀은 FreeRADIUS 에서 끝나며 radsecproxy 와 무관하다.

검증과 반영 절차는 `README.md`.
