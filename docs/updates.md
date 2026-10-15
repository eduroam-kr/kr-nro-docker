# 이미지 버전과 업데이트

## 현재 고정 버전

| 대상 | 버전 | 지정 위치 |
|---|---|---|
| FreeRADIUS | `freeradius/freeradius-server:3.2.10-alpine` | `docker-compose.yml` |
| radsecproxy 베이스 | `alpine:3.22` | `radsecproxy/Dockerfile` |
| radsecproxy 패키지 | `1.11.2-r0` | `radsecproxy/Dockerfile` |
| radsecproxy 이미지 | `local/radsecproxy:1.11.2` | `docker-compose.yml` |

`latest` 를 쓰지 않는다. 단순 재시작이나 재빌드로 **버전이 조용히 바뀌는 것**을 막기 위해서다. 국가 NRO 에서 의도치 않은 구성 변경은 장애 원인 추적을 어렵게 한다.

radsecproxy 는 alpine 저장소 버전에 묶인다. alpine 3.20 은 `1.10.1-r0`, 3.22 는 `1.11.2-r0` 이다. **radsecproxy 를 올리려면 베이스 alpine 도 함께 올린다.** 버전 확인: <https://pkgs.alpinelinux.org/packages?name=radsecproxy>

## 현재 버전 확인

```bash
docker exec kr-nro-radsecproxy radsecproxy -v
docker exec kr-nro-freeradius radiusd -v
```

## 업데이트 절차

되돌릴 수 있게 **새 태그로 올리고 이전 태그를 남긴다.**

### radsecproxy

```bash
# 1. 새 버전 확인 후 Dockerfile 의 alpine·radsecproxy 버전을 고친다
# 2. docker-compose.yml 의 image 태그도 새 버전으로 고친다
#      image: local/radsecproxy:1.11.3
# 3. 빌드
docker compose build --pull radsecproxy

# 4. 교체
docker compose up -d radsecproxy

# 5. 확인
docker compose logs --tail=50 radsecproxy
docker exec kr-nro-radsecproxy radsecproxy -v
```

**롤백**: `docker-compose.yml` 의 image 태그를 이전 버전으로 되돌리고 `docker compose up -d radsecproxy`. 이전 이미지가 로컬에 남아 있어 즉시 복구된다.

### FreeRADIUS

```bash
# docker-compose.yml 의 image 태그를 새 버전으로 고친 뒤
docker compose pull freeradius
docker compose run --rm freeradius -C -lstdout   # 설정 호환 확인
docker compose up -d freeradius
```

메이저·마이너가 올라갈 때는 설정 문법이 바뀔 수 있으므로 **`-C` 검사를 반드시 먼저** 한다. 롤백은 태그를 되돌리고 `up -d`.

## 언제 올리나

- **보안 패치는 즉시.** 그 외에는 트래픽이 적은 시간대에.
- radsecproxy 는 국제 구간이 RadSec 으로 전환되기 전까지는 **사실상 유휴**다. 지금 올리는 것은 인증 경로에 영향이 없다. 전환 전에 절차를 익혀 두기 좋다.
- 전환 이후에는 radsecproxy 가 실제 서비스 경로가 되므로, 반드시 롤백 가능한 상태(이전 태그 보존)에서 작업한다.

## 주의

- 설정 파일(`clients.conf`, `proxy.conf` 등)은 bind mount 라 **이미지에 포함되지 않는다.** 설정만 고칠 때는 빌드도 pull 도 필요 없고 재시작이면 된다.
- `docker compose up -d --build` 를 습관적으로 쓰지 않는다. 빌드는 Dockerfile 을 고쳤을 때만 의도적으로 한다.
