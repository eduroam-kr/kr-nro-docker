# kr-nro-docker 작업 규약

대한민국 eduroam NRO 의 운영 설정. 무엇을 하는지와 반영 절차는 [README.md](README.md) 에 있고, 이 파일은 어떻게 작업할지만 다룬다.

## 공개 저장소다. 값은 서브모듈에 있다

이 저장소는 공개다. 실제 IP·공유키·운영 기록은 **비공개 서브모듈 하나**(`private/`)에 있고, 여기에는 커밋 해시만 남는다.

```
private/                    ← 서브모듈
├─ secrets.d/               IP, 공유키, 기관별 이력
├─ secrets.disabled/        비활성 기관의 값
├─ docs-priv/               운영 정보가 든 문서
└─ pki/                     인증서 발급 설정
```

바깥의 네 경로는 **같은 이름의 심볼릭 링크**다.

```
freeradius/secrets.d        → ../private/secrets.d
freeradius/secrets.disabled → ../private/secrets.disabled
docs-priv                   → private/docs-priv
pki                         → private/pki
```

이름을 그대로 둔 덕에 다른 파일의 참조를 고칠 필요가 없고, 값은 서브모듈 안에만 있다.

`.gitmodules` 의 URL 은 자리표시다. 접근 권한이 있는 사람이 자기 환경에서 지정한다. 이 값은 `.git/config` 에 들어가고 커밋되지 않는다.

```sh
git config submodule.private.url <실제 경로나 주소>
git submodule update --init
```

URL 이 같은 장비의 경로면 `git submodule update --init` 이 `transport 'file' not allowed` 로 막힌다(CVE-2022-39253). 그 한 번만 `git -c protocol.file.allow=always` 를 앞에 붙인다.

권한이 없으면 네 링크가 빈 곳을 가리킬 뿐, 나머지는 그대로 읽힌다.

**주소는 쓰지 않는다.** 예시가 필요하면 RFC 5737 문서용 대역(`203.0.113.10` 등)을 쓴다. 우리 주소든 남의 주소든 마찬가지다 — 공개된 주소라도 **우리 문서가 그것을 적어 둘 이유**는 별개다.

인증서도 넣지 않는다. 핸드셰이크에서 상대에게 제시되므로 비밀은 아니지만, SAN 에 우리 주소가 들어 있다(실측: IP 두 개).

**기관을 지목해 잘못을 적지 않는다.** 실패 건수, 설정 오류, 장애 귀속은 `docs-priv/` 로 보내고 본문은 "한 기관이" 로 쓴다. 패턴으로 적고, 그 기관에는 따로 연락한다.

### realm 이름은 두 경우를 가른다

`example.ac.kr` 로 **바꿔야 하는 것**은 자리표시다. 어느 realm 이든 상관없이 모양만 보여주는 자리 — 설정 예시, 파일명 예시, 명령 예시가 그렇다.

```
conf.d/example.ac.kr.conf
client 2026_example_ac_kr_10 { ipaddr = ${EXAMPLE_AC_KR_IPADDR} }
```

**그대로 두는 것**은 실제 이름이 교훈인 경우다. 바꾸면 요지가 사라진다.

```
없으면 `gist` 패턴이 `dgist.ac.kr` 을 잡는다      ← 국내에 실재하는 충돌
"eduroam KR: use @skku.ac.kr (not @skku.edu)"    ← 우리가 실제로 돌려주는 문자열
```

첫째 줄을 `example` ⊂ `myexample` 로 바꾸면 지어낸 짝이 되어 "그런 일이 있겠나" 싶게 만든다. 둘째 줄을 바꾸면 문서가 설정과 어긋나, 로그에서 그 문자열을 본 담당자가 문서를 찾아올 길이 막힌다.

가르는 기준은 하나다. **그 이름이 사실을 나르는가, 자리만 채우는가.**

## 일괄 치환은 하지 않는다

구조와 값을 나눈 목적이 이것이다. 치환할 것이 없으면 치환 사고도 없다. 예전에 공개본을 만들며 `radsecproxy` 가 `__KEY__proxy` 가 되고, `gist` 예시가 `ample` 로 망가져 설명이 뜻을 잃은 적이 있다.

## 구조와 값을 나눈다

`conf.d/<realm>.conf` 는 구조만 담는다. IP 와 공유키는 `secrets.d/<realm>.cfg` 에 두고 `${...}` 로 참조한다.

```
client 2026_postech_ac_kr_33 {
	ipaddr = ${POSTECH_AC_KR_IPADDR}
	secret = ${POSTECH_AC_KR_SECRET_1}
	shortname = 2026_postech_ac_kr_33
	operator_name = '1postech.ac.kr'
}
```

확장자가 `.conf` 와 `.cfg` 로 다른 이유는 파일명이 같으면 편집·복사·복원에서 뒤바뀌기 때문이다. 확장자는 파일을 옮겨도 따라다니므로 미러 쪽 `.gitignore` 를 디렉터리가 아니라 확장자로 걸 수 있다.

변수명은 대문자·숫자·언더바만 쓴다. `<REALM>_<FIELD>` 꼴이고 realm 의 점은 언더바로 바꾼다. 장비가 여럿이면 IP 끝자리를, 키가 여럿이면 일련번호를 덧붙인다.

```
KISTI_RE_KR_IPADDR / KISTI_RE_KR_SECRET
KAIST_AC_KR_IPADDR_14 / KAIST_AC_KR_IPADDR_15
POSTECH_AC_KR_SECRET_1 / POSTECH_AC_KR_SECRET_2
```

`$INCLUDE secrets.d/` 는 `proxy.conf` 와 `clients.conf` 보다 **먼저** 와야 한다. `conf.d/` 는 `proxy.conf` 안에서 읽히고, 설정 변수는 참조보다 먼저 정의되어야 한다.

## 문법이 둘이다

설정 항목과 조건문의 문법이 다르다. 섞으면 기동이 실패한다.

| 자리 | 문법 |
|---|---|
| 설정 항목 (`secret =`, `ipaddr =`) | `${VAR}` |
| unlang 조건문 (`if (&Attr == ...)`) | `"%{config:VAR}"` |

조건문에 `${VAR}` 를 쓰면 `Failed to parse value for attribute` 로 파싱 자체가 실패한다. 서브넷 비교(`&NAS-IP-Address < ...`)에서도 `%{config:...}` 가 동작한다.

## 운영에서 시행착오하지 않는다

조건문·분류기·정책을 고칠 때는 `.55` 의 `/opt/nro-test` 에서 먼저 통과시킨다. 호스트 포트를 열지 않는 격리 환경이고, 거절 경로 네 가지를 모두 재현한다.

```
reject.test     홈서버가 거절        → home_reject
dead.test       홈서버 무응답        → home_timeout
local.test      프록시 없이 거절      → no_auth_type
vsreject.test   virtual_server 거절  → local_reject
```

경로를 새로 만들면 시험대에도 그 경우를 더한다. `vsreject.test` 가 없던 탓에 우리가 끊은 건이 `home_reject` 로 잘못 분류되던 것을 오래 못 잡았다.

## 반영 전에 반드시 검사한다

절차는 README 에 있다. 요점만 적으면, 문법 검사는 서비스에 영향이 없고 반영은 재시작이다. FreeRADIUS 는 무중단 reload 를 지원하지 않는다.

`client` 와 `home_server` 는 HUP 으로 반영되지 않는다. `docker-compose.yml` 의 볼륨을 바꿨다면 재시작이 아니라 `up -d` 로 컨테이너를 다시 만들어야 한다.

주석만 고쳤다면 재시작하지 않는다.

## 비활성화는 짝으로 보존한다

기관을 내릴 때 이름이 짝을 이룬다.

```
conf.d/<realm>.conf       →  conf.disabled/<realm>.conf
secrets.d/<realm>.cfg     →  secrets.disabled/<realm>.cfg
```

되살릴 때는 반대로 옮긴다.

둘을 한 디렉터리에 두지 않는 이유는 `conf.disabled/` 가 공개 쪽이기 때문이다. 거기 `.cfg` 를 두면 `.gitignore` 에 걸려 **추적되지 않은 채 조용히 사라진다.** 값은 비공개 서브모듈 안에 있어야 이력이 남는다.

`conf.disabled/` 는 `$INCLUDE` 범위 밖이라 읽히지 않는다. 그래서 그 안의 `${...}` 는 풀리지 않는다 — 참조가 어느 변수를 가리키는지는 같은 이름의 `.cfg` 를 보면 된다.

`secrets.disabled/` 는 `$INCLUDE secrets.d/` 바깥이라 로드되지 않는다. 값이 git 에만 남고 설정에는 영향을 주지 않는다.

realm 을 내릴 때 `radsecproxy/nro-direct-realms.conf` 의 이 NRO 가 담당하는 realm 목록에서도 함께 뺀다.

## 주석은 짧게, 근거는 문서로

머리말이 필요한 파일은 세 줄로 쓴다. 영문 한 줄, 국문 한 줄, 문서 참조 한 줄이다.

```
# Realms that cannot be routed. Rejected here, never sent to the ETLR.
# 라우팅 불가 realm. 여기서 끊고 ETLR 로 올려보내지 않는다.
# docs/blackhole.md
```

**파일명과 내용이 이미 말하는 것은 적지 않는다.** `conf.d/<realm>.conf` 에는 머리말을 두지 않는다 — realm 은 파일명이, SP/IdP 역할은 블록이 말한다. 같은 문서 참조를 열두 번 적는 대신 `proxy.conf` 의 `$INCLUDE conf.d/` 옆에 한 줄 둔다.

`clients.conf — KR-NRO` 처럼 디렉터리가 말하는 것을 되풀이하지도 않는다.

기관의 이력과 사연은 `secrets.d/<realm>.cfg` 에 적는다. 장비가 바뀌거나 키가 어긋났을 때 열어보는 파일이 거기이기 때문이다. 모든 기관에 똑같이 적용되는 규칙은 `docs/` 에 적고 설정에는 되풀이하지 않는다.

## 문서는 두 벌이다

`docs/` 는 일반화된 것, `docs-priv/` 는 운영 정보가 든 것이다. 파일명에 번호를 붙이지 않는다.

하드 랩 금지 — 문단과 목록 항목을 각각 한 줄로 쓰고 줄바꿈은 렌더러에 맡긴다.

## 추정과 측정을 섞지 않는다

측정값은 언제·무엇을·얼마나를 함께 적는다. 재현하지 않은 것은 추정이라고 밝힌다.

```
✗  NAS-Identifier 없는 요청이 많다
✓  실측(2026-09-28): NAS-Identifier 가 없는 요청이 하루 5,523건
```

## 커밋

```text
<type>: <무엇을 왜 했는지>
```

`feat` 새 설정·기능 · `fix` 고침 · `docs` 문서 · `chore` 도구·정리.

- 제목에는 "파일을 바꿈" 이 아니라 **무엇을 왜** 를 쓴다.
- 설정과 문서를 같이 고쳤으면 **설정 쪽 type** 을 쓴다. 문서는 설정을 따라간다.
- `Claude-Session` 은 기재하지 않는다 (공개 저장소).

`private/` 을 함께 고쳤으면 **거기서 먼저 커밋·push 한 뒤** 부모에서 포인터를 커밋한다. 순서가 바뀌면 부모가 없는 커밋을 가리킨다.

upstream 설정 파일을 우리 것으로 바꿀 때는 **커밋을 둘로 쪼갠다.** 먼저 이미지에서 꺼낸 원본을 그대로 커밋하고, 그 위에 우리 수정만 올린다. `git show` 하나로 무엇을 바꿨는지 보이고, 이미지 버전이 올라가도 같은 방식으로 되풀이할 수 있다.

```sh
docker run --rm --entrypoint cat <image> /opt/etc/raddb/radiusd.conf
```

## git 이 기록이다

추적되는 파일은 **사본을 만들지 않는다.** `git diff` 로 무엇이 바뀌었는지 보고 `git restore <파일>` 로 되돌린다. 타임스탬프를 붙인 사본은 어느 것이 최신인지 알기 어렵고, 설정 트리 안에 흘리면 기동이 깨진다.

git 이 지켜주지 않는 것이 셋이다.

| | |
|---|---|
| `private/` 서브모듈 | 부모의 `git restore` 로 돌아오지 않는다. 그 안에서 따로 커밋한다 |
| `.env` | 추적하지 않는다. 잃으면 복구할 수 없다. `.env.example` 을 최신으로 유지한다 |
| 운영 서버 | 아직 체크아웃이 아니다. 거기서 고친 것은 레포로 가져와 커밋해야 기록된다 |

### 설정 트리에 파일을 흘리지 않는다

`$INCLUDE <디렉터리>/` 는 **확장자를 가리지 않고** 그 안의 파일을 전부 읽는다. `sites-enabled/` 에 백업을 하나 두면 `Duplicate virtual server` 로 기동이 실패한다. `conf.d/`·`policy.d/`·`secrets.d/` 도 같다.

점으로 시작하는 파일과 이름에 `[A-Za-z0-9_.-]` 밖의 문자가 있는 파일은 건너뛴다. 그래서 `x.conf~` 는 무시되지만 `x.conf.bak-20261016` 은 **읽힌다.**
