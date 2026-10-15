# 인증서

## 두 종류를 구분할 것

| | EAP 인증서 | RadSec 인증서 |
|---|---|---|
| 쓰는 주체 | FreeRADIUS | radsecproxy |
| 검증하는 쪽 | 최종 사용자 **단말** | 상대 **RADIUS 서버** |
| 교체 비용 | 큼(전 사용자 재배포) | 작음(피어 협의) |
| **NRO** | **불필요** | 필요 |
| IdP | 필요 | 필요 |

**하나로 합쳐 쓰지 않는다.** 검증 주체·교체 주기·신뢰 앵커가 달라서, 합치면 단말 재배포와 피어 협의가 서로 묶여 버린다.

## NRO 에 EAP 인증서가 필요 없는 이유

NRO 는 순수 프록시라 EAP 터널을 그대로 중계할 뿐 종단하지 않는다. EAP 터널은 단말 ↔ 홈 IdP 사이에만 형성된다.

다만 freeradius 공식 이미지는 `eap` 모듈이 켜져 있어서, 기동 시 모듈을 인스턴스화하며 인증서 파일을 열려 한다. 파일이 없으면 **기동 자체가 실패**한다. 정책에서 `eap` 를 한 번도 호출하지 않아도 그렇다.

그래서 `freeradius/mods-enabled/eap` 를 **주석만 있는 빈 파일**로 두고 컨테이너에 마운트해 모듈 정의를 없앴다. 결과적으로 NRO 의 FreeRADIUS 는 인증서가 일절 필요 없다.

> 기동 로그에 `Loaded module rlm_eap` 가 **안 떠야** 정상이다.

`mods-enabled/` 디렉터리 전체를 마운트해도 같은 효과지만, 우리가 쓰는 모듈을 하나라도 빠뜨리면 기동이 실패한다(`suffix` 는 `realm` 모듈이 제공하고 `filter_username` 정책은 `always`·`expr` 등에 의존). 표적만 끄는 편이 안전하다.

## RadSec 인증서

radsecproxy 만 사용한다. `pki/certs/` 에 세 파일이 필요하다.

| 파일 | 내용 |
|---|---|
| `server-cert.pem` | 이 NRO 의 RadSec 인증서 |
| `server-key.pem` | 개인키 |
| `eduroam-ca.pem` | 상대 인증서를 검증할 트러스트 앵커 |

생성 설정과 절차는 `../pki/` 에 있다(`pki/README.md`).

RadSec 피어 인증서는 **EKU 에 `serverAuth` 와 `clientAuth` 를 모두** 넣는다. 피어가 서로 검증하므로 한쪽만으로는 안 된다. 정책 OID 는 `1.3.6.1.4.1.25178.3.1.1`(client), `.2`(server).

키 길이·유효기간 등 나머지 요건은 `cert-requirements.md`. 실제 발급 파라미터는 `pki/` 의 설정 파일이 정본이다.

### 진짜 인증서를 받으려면

eduroam 의 RadSec 신뢰는 **eduroam Policy CA / GÉANT TCS** 가 앵커다. NRO 서버 인증서는 연합 운영자를 통해 발급받고, 받은 체인을 `eduroam-ca.pem` 에 넣는다.

전환 전에 **현행 ETLR RadSec 엔드포인트와 요구 CA 를 연합 운영자에게 확인한다.** 예전 `etlr1/2.eduroam.org` 이름과 앵커가 아직 유효하다고 가정하지 않는다.

### 실험실용 임시 인증서

진짜 인증서가 오기 전에 스택만 띄워 보려면 자체서명으로 때운다. **이 인증서는 실제 eduroam 인프라가 신뢰하지 않는다.**

```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout server-key.pem -out server-cert.pem \
  -subj "/C=KR/O=<조직>/CN=<NRO FQDN>"
cp server-cert.pem eduroam-ca.pem   # 자기 자신을 신뢰. 실험실 전용
```

### 도메인 주의

`eduroam.kr` 은 NRO(KISTI)가 아니라 RO 측이 보유한 도메인이다. 인증서 CN/SAN·URI 에 쓰지 않는다. **KREONET 이 통제하는 `*.eduroam.kreonet.net`** 만 쓴다. NRO 권위는 도메인이 아니라 eduroam 연합 등록으로 확립된다.

### 범위 주의

`pki/` 의 사설 CA 는 **국내 RadSec 패브릭(NRO ↔ 국내 IdP)** 용이다. 국제 ETLR 구간은 연합이 신뢰하는 CA(eduPKI / GÉANT TCS)를 요구할 수 있으므로, 전환 전 연합 운영자에게 확인한다. 자세한 사항은 `cert-requirements.md`.
