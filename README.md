# eduroam KR NRO

대한민국 eduroam NRO 의 RADIUS 서버 구성.

- realm 을 보고 국내 기관 홈서버로 중계한다
- 모르는 realm 은 국제 ETLR 로 올려보낸다
- 로컬 사용자도 EAP 종단도 없다. 순수 중계다

국내 경로에는 특이점이 있다.

- `.ac.kr` 대부분은 **RO 가 중계**한다. NRO 에 직결하는 기관은 일부다
- 미등록 `.kr` 은 국제로 내보내지 않고 **여기서 끝낸다**
- 기관 realm 은 정확일치 하나만 둔다. 서브도메인은 거절한다
- 직결 기관을 추가하면 `radsecproxy/nro-direct-realms.conf` 도 함께 고친다

→ [docs/radius-routing.md](docs/radius-routing.md) · [docs/onboarding.md](docs/onboarding.md)

구조와 용어는 [docs/overview.md](docs/overview.md), 작업 규약은 [AGENTS.md](AGENTS.md).

## 시작

```bash
cp .env.example .env         # 배포마다 다른 값은 여기 하나뿐
git submodule update --init  # 비공개 값. 접근 권한이 있을 때
docker compose up -d
```

서브모듈 URL 은 자리표시라 내 환경에서 지정해야 한다. → [AGENTS.md](AGENTS.md)

## 설정 반영

FreeRADIUS 는 무중단 reload 를 지원하지 않는다. **검사하고 재시작한다.**

```bash
docker compose run --rm freeradius -C -lstdout   # 검사. 서비스에 영향 없음
docker compose restart freeradius
docker compose logs -f --tail=50 freeradius
```

> `--entrypoint ""` 를 붙이지 않는다. 이미지의 entrypoint 가 `PATH` 를 잡으므로
> 덮으면 `radiusd` 를 찾지 못한다.
>
> 기동에 성공하면 로그 끝에 `Ready to process requests` 가 나온다.
> `Please use pools instead of authhost` 경고는 블랙홀 realm 마다 나오는 정상 출력이다.

## 문서

| | |
|---|---|
| [overview](docs/overview.md) | 용어, 구성, 파일 배치 |
| [radius-routing](docs/radius-routing.md) | 요청 처리 순서와 realm 매칭 |
| [onboarding](docs/onboarding.md) | **기관 등록·해지** |
| [operator-name](docs/operator-name.md) | 방문기관 식별자 부여 정책 |
| [blackhole](docs/blackhole.md) | 라우팅 불가 realm 즉시 거절 |
| [input-validation](docs/input-validation.md) | User-Name 위생 검사 |
| [logging](docs/logging.md) | 로그 두 종류, reason 분류, 함정 |
| [certificates](docs/certificates.md) | 이 구성에 필요한 인증서 |
| [cert-requirements](docs/cert-requirements.md) | eduroam 공식 인증서 요건 |
| [radsec-migration](docs/radsec-migration.md) | RadSec 전환 |
| [eduroam-settings](docs/eduroam-settings.md) | 권장 설정 반영 현황 |
| [updates](docs/updates.md) | 이미지 버전 고정과 롤백 |

운영 기록은 비공개 서브모듈의 `docs-priv/` 에 있다.
