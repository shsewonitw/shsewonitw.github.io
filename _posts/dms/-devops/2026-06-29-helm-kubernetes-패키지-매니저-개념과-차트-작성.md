---
layout: post
title: "[Daily morning study] Helm: Kubernetes 패키지 매니저 개념과 차트 작성"
description: >
  #daily morning study
category: 
    - dms
    - dms-devops
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

## Helm이란

Helm은 Kubernetes의 패키지 매니저다. `apt`, `brew`, `npm` 같은 패키지 매니저를 Kubernetes에 적용한 개념으로 이해하면 쉽다. Kubernetes에 애플리케이션을 배포할 때 Deployment, Service, ConfigMap, Ingress 등 여러 YAML 파일이 필요한데, Helm은 이를 하나의 **Chart**로 묶어서 관리하고 배포할 수 있게 해준다.

### Helm이 해결하는 문제

Kubernetes 리소스를 raw YAML로 관리하면 다음 문제가 생긴다.

- 환경(dev/staging/prod)마다 설정 값이 달라도 YAML 파일을 일일이 수정해야 함
- 여러 리소스를 한 번에 배포/롤백/삭제하는 게 번거로움
- 이미 잘 구성된 오픈소스 앱(Nginx, Prometheus 등)을 처음부터 작성해야 함

Helm은 **템플릿 기반 패키지(Chart)** 와 **값 주입(Values)** 을 통해 이 문제를 해결한다.

---

## 핵심 개념

| 개념 | 설명 |
|------|------|
| **Chart** | Kubernetes 리소스를 묶은 패키지. 디렉토리 또는 `.tgz` 아카이브 형태 |
| **Release** | Chart를 클러스터에 배포한 인스턴스. 같은 Chart를 여러 번 배포하면 Release가 여러 개 생긴다 |
| **Values** | Chart 템플릿에 주입하는 설정 값. `values.yaml`에 기본값을 정의하고 배포 시 오버라이드 가능 |
| **Repository** | Chart를 저장하고 공유하는 저장소. `https://artifacthub.io`에서 공개 차트를 검색할 수 있다 |

---

## Chart 디렉토리 구조

```
mychart/
├── Chart.yaml          # 차트 메타데이터 (이름, 버전, 설명)
├── values.yaml         # 기본 Values 정의
├── templates/          # 템플릿 YAML 파일들
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl    # 공통 템플릿 헬퍼 (named template 정의)
└── charts/             # 의존 차트 (서브차트)
```

### Chart.yaml 예시

```yaml
apiVersion: v2
name: mychart
description: A sample Helm chart for a web application
type: application
version: 0.1.0        # 차트 자체의 버전
appVersion: "1.0.0"   # 배포하는 앱의 버전
```

---

## Values와 템플릿

### values.yaml

```yaml
replicaCount: 2

image:
  repository: my-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 500m
    memory: 128Mi
  requests:
    cpu: 250m
    memory: 64Mi
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

`{{ .Values.xxx }}`로 `values.yaml`의 값을 참조하고, `{{ .Chart.xxx }}`로 `Chart.yaml`의 값을 참조한다. `{{- include "..." . | nindent N }}`은 _helpers.tpl에 정의한 named template을 호출하는 패턴이다.

---

## 주요 Helm 명령어

```bash
# 차트 저장소 추가
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 차트 검색
helm search repo nginx

# 차트를 클러스터에 배포 (Release 생성)
helm install my-release ./mychart

# values 파일로 오버라이드
helm install my-release ./mychart -f custom-values.yaml

# 특정 값만 오버라이드
helm install my-release ./mychart --set replicaCount=3

# 배포된 Release 목록 확인
helm list

# Release 업데이트
helm upgrade my-release ./mychart -f custom-values.yaml

# 이전 버전으로 롤백
helm rollback my-release 1

# Release 삭제
helm uninstall my-release

# 배포 전 YAML 렌더링 확인 (dry-run)
helm template my-release ./mychart -f custom-values.yaml
helm install my-release ./mychart --dry-run --debug
```

---

## 환경별 Values 관리 패턴

하나의 Chart에 환경별 values 파일을 두는 방식이 일반적이다.

```
mychart/
├── values.yaml            # 공통 기본값
├── values-dev.yaml        # dev 환경 오버라이드
├── values-staging.yaml    # staging 환경 오버라이드
└── values-prod.yaml       # prod 환경 오버라이드
```

```bash
# dev 환경 배포
helm install my-app ./mychart -f values.yaml -f values-dev.yaml

# prod 환경 배포
helm install my-app ./mychart -f values.yaml -f values-prod.yaml
```

`-f` 플래그는 순서대로 merge되며, 나중에 오는 파일이 앞의 값을 덮어쓴다.

---

## _helpers.tpl — Named Template

`_`로 시작하는 파일은 Kubernetes 리소스로 렌더링되지 않고 헬퍼 정의용으로만 사용된다.

```
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

이 패턴 덕분에 여러 템플릿에서 이름과 레이블을 일관되게 유지할 수 있다.

---

## Release 이력 관리와 롤백

Helm은 Release마다 revision 이력을 Kubernetes Secret으로 클러스터 내에 저장한다.

```bash
# revision 이력 확인
helm history my-release

# REVISION  STATUS     CHART          DESCRIPTION
# 1         superseded mychart-0.1.0  Install complete
# 2         superseded mychart-0.1.1  Upgrade complete
# 3         deployed   mychart-0.1.2  Upgrade complete

# revision 2로 롤백
helm rollback my-release 2
```

롤백 역시 새 revision이 추가되는 방식이므로 이력이 보존된다.

---

## 공개 Chart 활용 예시

```bash
# Prometheus + Grafana 스택 설치
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Nginx Ingress Controller 설치
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.replicaCount=2
```

공개 Chart를 쓸 때는 `helm show values <chart-name>`으로 오버라이드 가능한 값 목록을 먼저 확인하는 게 좋다.

---

## Helm vs Kustomize

두 도구 모두 Kubernetes 배포를 관리하는 도구지만 접근 방식이 다르다.

| | Helm | Kustomize |
|---|------|-----------|
| **방식** | 템플릿 렌더링 (Go template) | 패치/오버레이 기반 |
| **패키지 배포** | Chart로 공유/재사용 용이 | 배포보다 환경별 커스터마이징에 집중 |
| **러닝 커브** | 템플릿 문법 학습 필요 | Kubernetes YAML 그대로 사용 |
| **롤백** | Helm이 revision 이력 관리 | Git 이력으로 관리 |
| **GitOps 연동** | 가능 (ArgoCD, Flux 지원) | 기본 내장 (kubectl -k) |

복잡한 패키지를 공유하거나 재사용할 때는 Helm, 간단한 환경별 설정 관리에는 Kustomize를 택하는 경우가 많다. ArgoCD에서는 두 가지 모두 지원한다.
