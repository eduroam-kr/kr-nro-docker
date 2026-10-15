# Operator-Name

방문기관(SP)이 자기를 식별하는 속성이다(RFC 5580). 해외 홈 IdP 가 알고 싶은 것은 "우리 사용자가 지금 어느 캠퍼스에 있나"지 "어느 국가 프록시를 거쳤나"가 아니다.

형식은 `<네임스페이스ID><이름>` 이고 eduroam 은 `1`(REALM)을 쓴다. 예) `1example.ac.kr`

## 일괄로 붙이지 않는다

NRO 가 모든 프록시 요청에 자기 이름을 붙이면 어느 캠퍼스에서 온 요청이든 출발지가 하나로 보여 **홈 IdP 의 CUI 계산과 통계가 왜곡된다.**

그래서 **client 별 고유값만** 붙인다. 값은 각 client 블록의 `operator_name` 항목에서 가져오므로 기관마다 자기 realm 이 들어간다.

```
client 2026_example_ac_kr_1 {
	operator_name = '1example.ac.kr'
}
```

`operator_name` 이 없는 client(ETLR, RO, 루프백 등)에는 아무것도 하지 않는다.

## 우선순위

```
① SP 가 직접 보낸 값          가장 권위 있다. 절대 덮지 않는다
② NRO 직속 기관               client 블록의 operator_name
③ 그 외(RO 경유 등)           NAS-IP / NAS-Identifier 지문
```

값은 ETLR 을 거쳐 해외 홈 IdP 까지 전달된다. **틀린 값은 국제적으로 퍼진다.** 그래서 ① 을 덮지 않고, ③ 은 확신이 있는 것만 넣는다. 모르면 비워 두는 편이 틀린 값보다 낫다.

구현은 `policy.d/operator-name` 이 ①②를 처리하고, ③ 은 `policy.d/kren-operator-name` 에 넘긴다.

## RO 경유분이 어려운 이유

RO 는 client 하나가 국내 수십 개 대학을 중계한다. client 의 `operator_name` 을 쓰면 **모든 대학이 RO 로 귀속되어** ① 에서 피하려던 왜곡이 그대로 재현된다.

그래서 `NAS-IP-Address` 와 `NAS-Identifier` 를 지문으로 써서 방문기관을 판별한다. 값은 `secrets.d/nas-ip.cfg` 에 있고 정책은 `%{config:...}` 로 참조한다. 판별 근거는 `docs-priv/` 에 있다.

조건문에서는 `${...}` 가 풀리지 않는다. 설정 항목용과 조건문용 문법이 다르다.

```
설정 항목   ipaddr = ${EXAMPLE_AC_KR_IPADDR}
조건문      if (&NAS-IP-Address < "%{config:EXAMPLE_AC_KR_SUBNET}")
```

지문으로 판별되지 않는 요청은 비워 둔 채 보낸다. 실측 기준 RO 경유분의 약 85% 가 여기 해당한다. 줄이려면 RO 쪽이 `NAS-Identifier` 를 기관 식별자로 정규화해 주어야 한다.

## NRO 자신이 SP 로 나설 때

NRO 가 스스로 요청을 낼 때만 예외다. 그때도 `1` + **우리가 소유한 도메인**을 쓴다. 남의 도메인은 쓰지 않는다.

## 기관을 추가할 때

`conf.d/<realm>.conf` 의 client 블록에 `operator_name = '1<realm>'` 한 줄을 넣는다. 정책은 건드리지 않는다.
