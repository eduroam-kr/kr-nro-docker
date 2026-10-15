# radsecproxy has no official image, so we build it.
# radsecproxy 는 공식 이미지가 없어 직접 빌드한다.
#
# Both the base and the package version are pinned: a rebuild must give the
# same result. When raising either, check both together.
# 베이스와 패키지 버전을 모두 고정한다. 다시 빌드해도 같은 결과가 나와야 한다.
# 올릴 때는 alpine 버전과 radsecproxy 버전을 함께 확인한다.
#   https://pkgs.alpinelinux.org/packages?name=radsecproxy
FROM alpine:3.22
RUN apk add --no-cache radsecproxy=1.11.2-r0 ca-certificates gettext
EXPOSE 2083/tcp
# radsecproxy has no variable expansion, so resolve ${...} at start-up.
# radsecproxy 에는 변수 치환이 없어 기동할 때 ${...} 를 푼다.
CMD ["sh", "-c", "envsubst < /etc/radsecproxy.conf > /tmp/radsecproxy.conf && exec radsecproxy -f -c /tmp/radsecproxy.conf"]
