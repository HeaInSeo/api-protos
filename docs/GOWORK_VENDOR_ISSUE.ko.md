# go.work 로컬 경로와 Docker 빌드 불일치 문제

## 개요

`api-protos` 의 `gen/go` 패키지는 Go module proxy(pkg.go.dev)에 게시되어 있지 않고, 로컬 파일시스템 경로로만 참조됩니다. 이 구조는 개발 환경에서는 편리하지만, Docker 빌드 컨텍스트 안에서는 경로가 존재하지 않아 빌드가 실패합니다.

## 문제 상황

NodeForge 의 `go.work` 는 다음과 같이 api-protos 의 생성 코드를 절대 경로로 참조합니다:

```
# NodeForge/go.work
use (
    ./
    /opt/go/src/github.com/HeaInSeo/api-protos/gen/go/nodeforge/v1
)
```

Docker 빌드 시 이 경로는 컨테이너 안에 존재하지 않습니다. 결과적으로 `go build` 또는 `go work vendor` 가 실패합니다.

```
go: cannot find module providing package github.com/HeaInSeo/api-protos/gen/go/nodeforge/v1
```

## 원인

- `api-protos/gen/go/nodeforge/v1` 패키지는 GitHub에 push 되지 않거나 별도 Go module registry 에 배포되지 않음
- `go.work` 의 `use` directive 가 절대 경로를 가리키므로, 호스트 경로 구조 외부에서는 해석 불가
- `GOWORK=off` 로 workspace 를 무시하면 `go.mod` 에 `replace` directive 가 없어 동일하게 실패

## 현재 적용된 임시 해결책 (Dockerfile 내)

NodeForge 의 Dockerfile 에서 빌드 컨텍스트에 `api-protos/` 디렉토리를 포함시키고, `go.work` 경로를 `sed` 로 치환합니다.

```dockerfile
# 빌드 컨텍스트에 api-protos 포함 필요 (make vendor 이전에 rsync 또는 COPY)
COPY api-protos/ ./api-protos/
COPY . .

# go.work 의 절대 경로를 컨테이너 내 상대 경로로 치환
RUN sed -i \
  's|/opt/go/src/github.com/HeaInSeo/api-protos/gen/go/nodeforge/v1|./api-protos/gen/go/nodeforge/v1|g' \
  go.work

RUN go build -mod=vendor ...
```

빌드 컨텍스트에 `api-protos/` 를 포함시키려면 Docker build 실행 전 Makefile 또는 CI 스크립트에서 다음을 수행해야 합니다:

```bash
# NodeForge 루트 기준
rsync -a --delete \
  /opt/go/src/github.com/HeaInSeo/api-protos/ \
  ./api-protos/
```

## 근본 해결 방향

아래 중 하나를 선택해 장기적으로 해결합니다.

### 방법 A — api-protos gen/go 를 Go module 로 별도 배포

```
api-protos/gen/go/nodeforge/v1/ → github.com/HeaInSeo/api-protos-go 별도 repo
```

- `go get github.com/HeaInSeo/api-protos-go/nodeforge/v1@vX.Y.Z` 로 의존 가능
- Docker 빌드 시 외부 접근이 필요하므로 빌드 환경에 인터넷 접근이 없으면 불가

### 방법 B — vendor 에 gen/go 를 포함

`go work vendor` 실행 시 workspace 가 resolve 되므로, 빌드 전에 `make vendor` 를 실행하면 `vendor/` 안에 api-protos gen 코드가 포함됩니다. 이후 Docker 에서는 `vendor/` 만 복사하면 됩니다.

```bash
# 개발 머신 (api-protos 경로가 존재하는 환경)
make vendor   # = go work vendor

# Dockerfile
COPY vendor/ ./vendor/
# api-protos/ 별도 복사 불필요
# go.work sed 치환 불필요
RUN go build -mod=vendor ...
```

> **주의**: `go work vendor` 는 Go 1.22+ 에서 workspace vendor 를 지원합니다.
> 실행 환경에 api-protos 경로가 존재해야 하므로, CI 빌드 에이전트에서도 api-protos 클론이 있어야 합니다.

### 방법 C — go.work 를 제거하고 go.mod replace 사용

```go
// NodeForge/go.mod
replace github.com/HeaInSeo/api-protos/gen/go/nodeforge/v1 => ../api-protos/gen/go/nodeforge/v1
```

- workspace 없이 단일 `go.mod` 수준에서 로컬 경로를 replace
- Docker 빌드 시에도 Dockerfile 에서 `api-protos/` 복사 + 경로 치환이 동일하게 필요

## 현재 상태 요약

| 항목 | 상태 |
|------|------|
| 개발 머신 빌드 | 정상 (절대 경로 존재) |
| Docker 빌드 (make vendor 후 api-protos rsync 포함) | 정상 (임시 해결) |
| Docker 빌드 (vendor 없이) | 실패 |
| CI / 원격 빌드 에이전트 | api-protos 클론 필요 |

## 관련 파일

- `NodeForge/go.work` — 절대 경로 참조 위치
- `NodeForge/Dockerfile` — sed 치환 및 api-protos COPY 적용
- `NodeForge/Makefile` — `make vendor` 타겟 (go work vendor)
- `api-protos/gen/go/nodeforge/v1/` — 대상 패키지 (protoc 생성 코드)
