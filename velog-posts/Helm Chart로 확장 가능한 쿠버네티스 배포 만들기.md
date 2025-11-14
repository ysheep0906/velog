<p><img alt="" src="https://velog.velcdn.com/images/ysheep0906/post/a3a62fcf-920d-4013-b2e1-66f0b42a054b/image.png" /></p>
<h2 id="helm이란">Helm이란</h2>
<p>Helm은 흔히 쿠버네티스의 패키지 매니저(Kubernetes Package Manager)라고 불린다. Helm은 사실상 쿠버네티스의 애플리케이션 생명주기(Lifecycle)를 관리하는 도구이다. 쿠버네티스 애플리케이션을 <strong>설치·업그레이드·롤백·삭제까지 전 과정을 자동화하고 표준화하는 시스템</strong>이라고 생각하면 된다. </p>
<h2 id="등장-배경">등장 배경</h2>
<p>쿠버네티스의 원래 배포 방식은 다음과 같았다.
deployment.yaml
service.yaml
ingress.yaml
configmap.yaml
secret.yaml
hpa.yaml
pvc.yaml
...
관리해야 하는 YAML 파일이 점점 늘어나고,
환경(dev, stage, prod)에 따라 값도 달라지고,
버전이 올라가면서 설정도 바뀌고…</p>
<ul>
<li>YAML 파일 폭증: 하나의 앱 배포에 5~15개의 YAML 필요</li>
<li>환경별 설정 변경: dev/prod에서 포트, 도메인, replica 등 달라짐</li>
<li>사람이 직접 수정: 실수 위험 증가, 설정 불일치 발생</li>
<li>반복 작업 증가: 앱 버전만 바꿔도 여러 YAML 수정해야 함</li>
<li>재사용 어려움: 동일 앱을 다른 팀에서 쓰려면 YAML 복사해야 함</li>
</ul>
<p>즉, 정적 YAML 파일로는 빠르게 바뀌는 마이크로서비스 환경을 관리하기 어려웠다.</p>
<h2 id="helm의-장점">Helm의 장점</h2>
<p><strong>1. 배포 자동화</strong>
helm install, helm upgrade, helm rollback 명령어로 전체 배포 과정을 관리</p>
<p><strong>2. 패키징</strong>
하나의 Chart에 Deployment, Service, Ingress, ConfigMap, Secret, PVC 모든 리소스 템플릿을 담아 배포 가능</p>
<p><strong>3. 재사용</strong>
템플릿 + values 구조로 수백 번 배포해도 같은 구조 유지</p>
<p><strong>4. 환경별 설정 분리</strong>
values-dev.yaml, values-stage.yaml, values-prod.yaml을 만들어 환경마다 설정만 바꿀 수 있음</p>
<p><strong>5. 버전 관리</strong>
helm history, helm rollback 기능으로 배포 버전을 되돌릴 수 있음</p>
<p><strong>6. 팀 표준화</strong>
팀원이 10명이든 100명이든 “같은 Chart + 환경별 values 파일” 구조로 유지 가능</p>
<h2 id="helm의-구성요소">Helm의 구성요소</h2>
<pre><code>&lt;chart-name&gt;/
├── Chart.yaml        # 차트 메타데이터
├── values.yaml       # 기본 설정 값
├── charts/           # 차트 의존성 관리
├── templates/        # 템플릿 파일들
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl  # 템플릿 헬퍼 파일
└── README.md         # 차트에 대한 설명 (선택 사항)</code></pre><ul>
<li><strong>Chart</strong>: Helm 패키지로, Kubernetes 애플리케이션, 툴, 또는 서비스 실행에 필요한 모든 Kubernetes 리소스 정의 파일과 메타데이터를 포함한 집합. Chart는 애플리케이션의 청사진 역할을 담당.<ul>
<li><strong>Chart.yaml</strong>: Chart의 메타데이터 파일로, Chart의 이름, 버전, 애플리케이션에 대한 설명, 차트 작성자 정보 등을 포함</li>
<li><strong>Values</strong>: Chart에서 사용하는 설정 값들을 정의한 YAML 파일. 사용자는 이 값을 통해 Chart의 설정을 커스터마이징. .</li>
<li><strong>Templates</strong>: Kubernetes 리소스 정의 파일에서 변수와 로직을 사용해 유연하게 생성되는 파일들. Helm은 템플릿 엔진을 통해 <code>Values</code> 파일에서 제공된 값들을 반영하여 최종 Kubernetes 리소스 파일을 생성한다.</li>
<li><strong>Release</strong>: 특정 차트의 배포 단위로 릴리즈를 통해 특정 버전의 차트를 추적하고 관리할 수 있음.</li>
</ul>
</li>
<li><strong>Repository</strong> : helm chart를 모아두고 공유하는 저장소  <strong>원격 저장소</strong>(<strong>bitnami</strong>, <strong>Artifact HUB),</strong> <strong>로컬저장소(</strong>Chart Museum)</li>
</ul>
<h2 id="valuesyaml">values.yaml</h2>
<p>Helm의 가장 큰 장점은 환경별로 다른 값만 바꿀 수 있다는 것이다. </p>
<pre><code>replicaCount: 2

image:
  repository: my-registry/myapp
  tag: &quot;v1.0.0&quot;
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: &quot;100m&quot;
    memory: &quot;128Mi&quot;
  limits:
    cpu: &quot;500m&quot;
    memory: &quot;512Mi&quot;</code></pre><ul>
<li><p>dev 환경: replica 1, 작은 리소스</p>
</li>
<li><p>prod 환경: replica 3, 더 큰 리소스</p>
<pre><code># dev 환경
helm install myapp-dev ./myapp -f values-dev.yaml
</code></pre></li>
</ul>
<h1 id="prod-환경">prod 환경</h1>
<p>helm install myapp-prod ./myapp -f values-prod.yaml</p>
<pre><code>Chart는 그대로인데, values 파일만 바꿔서 완전히 다른 설정으로 배포하는 패턴이 가능하다.


## 템플릿 문법 – .Values, .Chart, .Release
Helm의 템플릿은 기본적으로 Go 템플릿 문법을 사용한다. 
자주 쓰는 내장 변수들
- .Values : values.yaml + -f로 넘긴 값들이 합쳐진 것
- .Chart : Chart의 이름, 버전 등 메타 정보
- .Release : 릴리스 이름, 네임스페이스 등 배포 정보

`templates/deployment.yaml`</code></pre><p>apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include &quot;myapp.fullname&quot; . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include &quot;myapp.name&quot; . }}
  template:
    metadata:
      labels:
        app: {{ include &quot;myapp.name&quot; . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: &quot;{{ .Values.image.repository }}:{{ .Values.image.tag }}&quot;
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80</p>
<pre><code>- **{{ .Values.replicaCount }}** → values.yaml의 값 참조
- **{{ .Chart.Name }}** → Chart.yaml의 이름
- **include &quot;myapp.fullname&quot;** . → _helpers.tpl에 정의한 헬퍼 템플릿 호출

## 템플릿 제어문 – if, range, with
Helm 템플릿은 코드처럼 조건/반복을 넣을 수 있다. 

### if 조건문</code></pre><p>{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}</p>
<pre><code>values.yaml 에서 ingress.enabled: true/false 에 따라 Ingress 리소스를 생성할지 말지 결정 가능

### range 반복문
서비스 포트를 배열로 정의해 놓고 반복 생성하기</code></pre><p>spec:
  ports:
  {{- range .Values.service.ports }}
    - name: {{ .name }}
      port: {{ .port }}
      targetPort: {{ .targetPort }}
  {{- end }}</p>
<pre><code>
`values.yaml`</code></pre><p>service:
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics
      port: 9100
      targetPort: 9100</p>
<pre><code>
### with – 중첩 경로 줄이기</code></pre><p>{{- with .Values.resources }}
resources:
  requests:
    cpu: {{ .requests.cpu }}
    memory: {{ .requests.memory }}
  limits:
    cpu: {{ .limits.cpu }}
    memory: {{ .limits.memory }}
{{- end }}</p>
<pre><code>with 안의 .는 .Values.resources 로 바뀐다고 생각하면 된다.

## _helpers.tpl – 네이밍 규칙을 한 곳에서 관리하기
Helm Chart 안에 _helpers.tpl 파일이 있다. 이 파일에는 **자주 반복되는 패턴을 함수처럼 재사용**할 수 있다.</code></pre><p>{{- define &quot;myapp.name&quot; -}}
{{ .Chart.Name }}
{{- end }}</p>
<p>{{- define &quot;myapp.fullname&quot; -}}
{{ printf &quot;%s-%s&quot; .Release.Name .Chart.Name | trunc 63 | trimSuffix &quot;-&quot; }}
{{- end }}</p>
<pre><code>
이제 Deployment, Service, Ingress 등 어느 곳에서든</code></pre><p>metadata:
  name: {{ include &quot;myapp.fullname&quot; . }}</p>
<pre><code>
## Chart 설치/업그레이드/롤백
### 설치 (install)</code></pre><p>helm install myapp ./myapp</p>
<pre><code>./myapp 디렉터리의 Chart를 읽어서 템플릿을 렌더링하고 쿠버네티스에 배포 → Release: myapp 생성

### 값 오버라이드</code></pre><p>helm install myapp ./myapp <br />  --set image.tag=v1.2.3 <br />  -f values-prod.yaml</p>
<pre><code>--set (가장 우선)

### 업그레이드 (upgrade)</code></pre><h1 id="values-prodyaml-수정-후">values-prod.yaml 수정 후</h1>
<p>helm upgrade myapp ./myapp -f values-prod.yaml</p>
<pre><code>Helm은 이전 릴리스와 비교해서 diff만 적용한다.

### 롤백 (rollback)</code></pre><h1 id="릴리스-히스토리-확인">릴리스 히스토리 확인</h1>
<p>helm history myapp</p>
<h1 id="특정-리비전으로-롤백">특정 리비전으로 롤백</h1>
<p>helm rollback myapp 2</p>
<pre><code>k8s YAML을 다시 쓰지 않고도, 버전 개념으로 배포 이력을 관리할 수 있다.


## Chart Repository
이미 잘 만들어진 오픈소스 Chart를 가져다 쓰는 것도 Helm의 강점이다.</code></pre><h1 id="bitnami-repo-추가">Bitnami repo 추가</h1>
<p>helm repo add bitnami <a href="https://charts.bitnami.com/bitnami">https://charts.bitnami.com/bitnami</a>
helm repo update</p>
<h1 id="redis-설치">Redis 설치</h1>
<p>helm install my-redis bitnami/redis</p>
<pre><code>우리가 직접 _Deployment, Service, PVC를 작성할 필요 없이_ values만 조절해서 원하는 형태로 설치할 수 있다. 

## helm template</code></pre><p>helm template myapp ./myapp -f values-prod.yaml &gt; out.yaml</p>
<p>```
Helm이 쿠버네티스에 적용하기 전에 최종적으로 어떤 YAML이 생성되는지 확인 가능</p>