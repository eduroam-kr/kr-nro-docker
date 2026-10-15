# eduroam 권장 설정 반영 현황

eduroam(GÉANT) 권장사항과 현재 설정을 대조한 것이다.

| 권장 항목 | 현재 | 비고 |
|---|---|---|
| 프록시 시 realm 보존 (`nostrip`) | 적용 | 전 realm |
| 홈서버 pool `fail-over` | 적용 | load-balance 금지 |
| 홈서버 생존확인 `status_check` | **쓰지 않음** | 아래 |
| `filter_username` | 적용 | 이상 User-Name 거절 |
| `preprocess` | 적용 | 깨진 NAS 속성 정규화 |
| Operator-Name | **설정함** (client 별 고유값) | `operator-name.md` |
| `attr_filter.pre-proxy` / `post-proxy` | 파일 준비, **비활성** | 아래 |
| CUI (Chargeable-User-Identity) | 미적용 | IdP 측 사안 |

## attr_filter — 준비만 하고 꺼 둔 이유

eduroam 은 프록시 구간에서 속성을 걸러내되, 다음은 **반드시 통과**시키라고 권한다.

- `pre-proxy`: `Calling-Station-Id`, `Operator-Name`, `Chargeable-User-Identity`
  - 기본 허용목록에 Operator-Name 과 Calling-Station-Id 가 없어 그냥 켜면 잘려나간다
  - Calling-Station-Id(단말 MAC)는 장애추적·F-Ticks 에 필요
- `post-proxy`: `User-Name`, `Chargeable-User-Identity`

허용목록 파일은 `freeradius/mods-config/attr_filter/` 에 넣어 뒀고 컨테이너에도 마운트된다.

다만 `sites-enabled/default` 에서 **호출은 주석 처리**해 두었다.

> **주의**: 허용목록에 이 빌드의 dictionary 에 없는 속성을 적으면
> `Parse error ... Instantiation failed for module "attr_filter.pre-proxy"` 로
> **기동이 실패한다.** 실제로 `WiMAX-Capability` 가 3.2.10-alpine 에 없어 제거했다.
> 속성을 추가할 때는 반드시 `-C` 로 검사한다.

이유: 허용목록이 잘못되면 인증이 **조용히** 깨진다. 국가 NRO 라 잘못 켜면 전국이 영향을 받는다. 기동 후 `freeradius -X` 로 실제 요청을 흘려 확인한 뒤 주석을 푸는 순서로 간다.

post-proxy 목록에서는 홈서버가 방문기관 네트워크를 제어하려는 속성 (`Tunnel-*` = VLAN 강제, `Filter-Id` 등)을 기본적으로 제외했다. 국내 정책상 VLAN 지정을 허용해야 한다면 해당 항목을 풀어 준다.

## status_check 를 쓰지 않는 이유

한때 모든 홈서버에 `status_check = status-server` 를 넣었다가 전부 제거했다.

**Status-Server 에 응답하는 국내 홈서버가 한 곳도 없었다.** 2026-09-18 실측에서 보낸 만큼 그대로 무응답이었다. 수치는 `docs-priv/measurements.md`

결과는 **멀쩡한 서버를 죽은 것으로 오판**하는 것이다. 실제 인증에 백 건 넘게 정상 응답하는 중에 dead 로 마킹된 곳이 있었다. 홈서버가 둘인 기관에서는 페일오버가 되풀이되어 EAP 대화가 두 서버로 쪼개졌고, 그 기관이 보고한 `No EAP session matching the State variable` 의 원인이었다.

구 어플라이언스도 쓰지 않았다. 4일치 로그에 홈서버 상태 전이가 0건이다.

`status_check` 없이도 `response_window` / `zombie_period` / `revive_interval` 로 무응답 서버는 걸러진다. 굳이 켜야 한다면 테스트 계정을 쓰는 `status_check = request` 를 기관별로 합의한 뒤에만 쓴다.

## 참고 문서

- [Server Certificate Practices in eduroam (GÉANT BPD, cbp-33)](https://archive.geant.org/projects/gn3/geant/services/cbp/Documents/cbp-33_server-certificate-practices-in-eduroam.pdf)
- [eduroam RADIUS attribute guide (Internet2)](https://spaces.at.internet2.edu/spaces/eduroam/pages/275743575/eduroam+RADIUS+attribute+guide)
- [Chargeable User Identity for eduroam (Jisc)](https://community.jisc.ac.uk/library/janet-services-documentation/chargeable-user-identity-eduroam-freeradius-implementation)
- [RADIUS attributes for eduroam (eduroam.ac.za)](https://eduroam.ac.za/faq/attributes/)
