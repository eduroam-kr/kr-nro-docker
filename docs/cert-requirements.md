# eduroam 인증서 요건 — NRO / RO / IdP (공식 문서 정리)

> eduroam 마이그레이션(FreeRADIUS + radsecproxy)에서 어떤 인증서를 어떤 속성·기간으로
> 발급/이관할지 판단하기 위한 참고. 값은 GÉANT eduroam **Best Practice Document
> (cbp-33)** 본문과 eduroam wiki / eduPKI 문서에서 교차 확인했다.
> 정리 일자 2026-09-17.

인증서는 **두 종류**이고 검증 주체가 다르다. 헷갈리면 안 됨.

| 인증서 | 어디 쓰나 | 누가 검증하나 |
|---|---|---|
| **EAP 서버 인증서** | IdP가 PEAP/TTLS **터널 안에서** 제시 | **최종 사용자 단말**(그 realm 사용자) |
| **RadSec 인증서** | 서버 간 RADIUS/TLS(2083) 전송 — NRO↔ETLR, IdP↔NRO 등 | **상대 RADIUS 서버(피어)** |

서버 간 링크가 **UDP + 공유키**라면 RadSec 인증서는 아직 불필요하다. RadSec 로 전환할 때 §2 가 적용된다.

---

## 1. IdP — EAP 서버 인증서

**출처**: eduroam *Best Practice Document: Server Certificate Practices in eduroam*
(cbp-33), Table 2.2 「Certificate properties」.

| 속성 | 공식 권장값 | 비고 |
|---|---|---|
| **키 길이** | **RSA 2048-bit 이상** (신규 환경). 일부 OS는 1024 미만 거부 | 최신 지침은 **ECC(P-256 등)** 도 권장 — 벤더 지원 충분 |
| **서명 알고리즘** | **SHA-2 (SHA-256)** | SHA-1 지원 종료 중(Apple·최신 Android 거부), MD5 금지 |
| **Server name (CN)** | Subject CN = **FQDN** | 실제 서버 FQDN 사용 |
| **subjectAltName** | CN과 **동일 이름**을 dNSName SAN에 | **와일드카드 금지**(일부 Windows 문제) |
| **BasicConstraints** | **`CA:FALSE` (critical)** | 서버 인증서는 CA 아님. OS X에서 문제 사례 |
| **Extended Key Usage** | **serverAuth**("Server Authentication") | Windows가 최소 1개 EKU 요구 |
| **CRL 확장** | CRL URL(정상 문법) 두면 Windows 8/Phone 8 호환 | 실제 CRL 다운로드는 안 함 |
| **유효기간** | 공인 CA **통상 1–3년** | 실무상 공인 TLS는 ~1년 + 자동갱신 |
| **발급 CA** | **공인 CA 권장**. NREN이면 **GÉANT TCS** | 자체서명 가능하나 배포·재설정 부담 |

### 체인·핸드셰이크 주의
- 인증서 체인은 **작게**. 크면 EAP 핸드셰이크가 느려지고, 많은 AP가 ~50 EAP
  패킷 / **64KB** 부근에서 연결을 끊는다.
- 중간 CA는 단말에 **사전 배포**(eduroam CAT 프로파일)하는 게 유리 — 왕복 감소.
- **CA가 동일하면** 서버 인증서 갱신은 단말 영향 없음. **CA가 바뀌면** 단말이
  새 앵커를 신뢰해야 함 → CAT 프로파일 재배포 필요.

---

## 2. NRO / RO / proxy — RadSec 인증서 (RADIUS/TLS)

**출처**: eduroam wiki *radsec certificates*, *eduPKI eduroam RA*.

| 속성 | 공식 권장값 | 비고 |
|---|---|---|
| **키 / 해시** | **RSA 4096-bit + SHA-256** | RadSec CSR 권장값 |
| **subjectAltName** | **URI 타입**으로 피어 역할 식별 | eduroam **SP / IdP / proxy / confederation root**. 복수 URI 가능 |
| **Policy OID** | eduroam RadSec 정책 OID 포함 | **client `1.3.6.1.4.1.25178.3.1.1`**, **server `1.3.6.1.4.1.25178.3.1.2`** |
| **Extended Key Usage** | **clientAuth + serverAuth** (둘 다) | 피어가 클라이언트이자 서버(RFC 6614) |
| **발급 CA** | eduroam 연합 신뢰 CA | 역사적 **eduPKI eduroam CA**, 현재 **GÉANT TCS** 계열 |
| **유효기간** | 발급 CA 정책(보통 수년) | 연합 안정성 위해 과도히 짧게 안 함 |

- radsecproxy에서 `TLS_PolicyOID`를 정의하면 인증서 경로에 위 OID가 있어야 통과.
- SAN URI의 **정확한 문자열 포맷**과 GÉANT TCS 현행 발급 절차는 wiki 원문
  (로그인/사내망) 확인 권장 — 아래 출처 참조.

> OID `1.3.6.1.4.1.25178` = GÉANT 기업 OID 공간, `.3.1` 가지 = eduroam 인증서 정책.

---

---

## 출처

- [Best Practice Document: Server Certificate Practices in eduroam (cbp-33, PDF)](https://archive.geant.org/projects/gn3/geant/services/cbp/Documents/cbp-33_server-certificate-practices-in-eduroam.pdf) — §1 Table 2.2의 1차 근거
- [EAP Server Certificate considerations — GÉANT wiki](https://wiki.geant.org/spaces/H2eduroam/pages/121346323/EAP+Server+Certificate+considerations)
- [radsec certificates — GÉANT wiki](https://wiki.geant.org/display/H2eduroam/radsec+certificates)
- [eduPKI eduroam RA — eduroam.org](https://www.eduroam.org/support/edupki-eduroam-ra/)
- [eduroam certificates FAQ (eduroam.ac.za, GÉANT 지침 미러)](https://eduroam.ac.za/faq/certificates/)

> 주의: wiki.geant.org 두 페이지는 봇 차단으로 본문 직접 추출은 못 했고, 값은
> 공식 Best Practice PDF(직접 텍스트 추출) + 검색 스니펫(해당 wiki·eduPKI 인용)에서
> 교차 확인했다. RadSec SAN URI 정확 포맷·TCS 현행 절차는 원문 재확인 권장.
