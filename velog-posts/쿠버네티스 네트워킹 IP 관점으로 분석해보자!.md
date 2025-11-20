<p><img alt="" src="https://velog.velcdn.com/images/ysheep0906/post/c5f7eec0-2e71-4a3a-8057-586d13f1e55d/image.png" /></p>
<p>이 글에서는 IP가 어떻게, 어떤 순서로 생기고 사용되는지 정리해보려고 한다.</p>
<h2 id="node-ip">Node IP</h2>
<p>일단 쿠버네티스 노드를 이루고 있는 리눅스 서버에는 노드 자체에 IP가 존재한다. 온프레미스 환경이든 클라우드 환경이든 노드들은 IP를 가지고 있다. 
저자는 VMware를 사용한 노드를 사용하고 있기 때문에 아래와 같이 아이피가 존재한다</p>
<pre><code># master 노드
$ ip addr show ens32
inet 10.10.10.10/24 brd 10.10.10.255 scope global ens32

# worker1
inet 10.10.10.11/24 brd 10.10.10.255 scope global ens32

# worker2
inet 10.10.10.12/24 brd 10.10.10.255 scope global ens32
</code></pre><p>이 IP들은 VMware 네트워크 설정을 통해 10.10.10.0/24 서브넷을 통해 받은 것들이다.</p>
<h2 id="클러스터-생성-시-pod-cidr--service-cidr">클러스터 생성 시 Pod CIDR / Service CIDR</h2>
<p><code>kubeadm init</code> 같은 걸로 클러스터를 만들 때 보통 이런 옵션을 준다.</p>
<pre><code>kubeadm init \
  --pod-network-cidr=10.32.0.0/12 \
  --service-cidr=10.96.0.0/12</code></pre><ul>
<li><strong>Pod 네트워크 CIDR</strong>: 10.32.0.0/12 -&gt; Pod IP들이 사용할 네트워크 대역</li>
<li><strong>Servive 네트워크 CIDR</strong>: 10.96.0.0/12 -&gt; Service IP들이 사용할 네트워크 대역</li>
</ul>
<h2 id="cni-설치-후-pod용-가상-네트워크-준비">CNI 설치 후, Pod용 가상 네트워크 준비</h2>
<p>지금 상태는 각 Node들은 자체 IP를 가지고 있고, kube-apiserver, kubelet 등 컨트롤 플레인은 돌고 있다. 하지만 Pod가 생성되어도 네트워크를 연결해줄 주체가 없다!</p>
<p>그렇기 때문에 Weave / Flannel / Calico 같은 CNI를 설치한다. (이 글에서는 Waeve로 설치)</p>
<pre><code>kubectl apply -f https://.../weave-daemonset.yaml</code></pre><ol>
<li>각 Node에 CNI DaemonSet Pod가 뜬다(예: weave-net-xxxx)</li>
<li>Node 안에 브리지 인터페이스를 하나 만든다</li>
<li>노드 별로 Pod CIDR 일부를 잘라서 사용한다.<ul>
<li>master: 10.32.0.0/24</li>
<li>worker1: 10.36.0.0/24</li>
<li>worker2: 10.44.0.0/24
```<h1 id="worker1에서-ip-route-를-보면">worker1에서 ip route 를 보면</h1>
</li>
</ul>
</li>
<li>36.0.0/24 dev weave  proto kernel  scope link</li>
<li>32.0.0/24 via  dev ens32</li>
<li>44.0.0/24 via  dev ens32<pre><code></code></pre></li>
</ol>
<h2 id="pod가-생성될-때-ip가-붙는-과정">Pod가 생성될 때 IP가 붙는 과정</h2>
<p>이제 Pod를 하나 띄워보자.</p>
<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx</code></pre><p><code>kubectl apply</code>하면 아래 일이 자동으로 일어난다.</p>
<ol>
<li><p>kubelet이 Pod를 하나 만들어야겠다고 판단
스케쥴러가 “worker1에 띄워라”라고 결정 -&gt; worker1의 kubelet이 Pod 생성을 시작</p>
</li>
<li><p>kubelet → CNI 플러그인 호출
kubelet은 CNI 규격에 따라 “컨테이너 네트워크 만들어줘” 라는 요청을 한다.</p>
</li>
</ol>
<ul>
<li>JSON으로 Pod 이름, Namespace, 필요한 정보 전달</li>
</ul>
<ol start="3">
<li>CNI(Weave)가 실제로 하는 일</li>
</ol>
<ul>
<li>리눅스 네트워크 네임스페이스를 하나 만든다. (Pod용)</li>
<li><strong>veth pair (가상 이더넷 케이블)</strong>를 만든다
  <code>veth-podX</code> : Pod 네임스페이스 안에 들어가는 쪽
  <code>veth-hostX</code>: Host 영역(노드)에 붙는 쪽</li>
<li>veth-hostX를 weave 브리지에 연결</li>
<li>Pod 네임스페이스 안의 eth0에 IP 할당</li>
</ul>
<h2 id="pod-↔-pod-통신-경로">Pod ↔ Pod 통신 경로</h2>
<h3 id="같은-node-내-pod-↔-pod">같은 Node 내 Pod ↔ Pod</h3>
<p>예: worker1 안에 Pod 두 개</p>
<ul>
<li>PodA: 10.36.0.5</li>
<li>PodB: 10.36.0.6</li>
</ul>
<ol>
<li>PodA → PodB로 TCP 패킷 전송 (목적지: 10.36.0.6)</li>
<li>PodA의 eth0 → veth-podA → veth-hostA → weave 브리지</li>
<li>weave 브리지가 10.36.0.6이 어느 veth인지 알고 → veth-hostB로 전달</li>
<li>veth-podB → PodB eth0</li>
</ol>
<p>즉, <strong>같은 노드에서는 그냥 브리지 스위칭</strong>만 일어난다.
<em>IP는 모두 10.36.0.x 대역</em></p>
<h3 id="다른-node-간-pod-↔-pod">다른 Node 간 Pod ↔ Pod</h3>
<p>예: worker1 (10.36.0.5) → worker2 (10.44.0.7)</p>
<ol>
<li>Pod(10.36.0.5)에서 10.44.0.7로 패킷 전송</li>
<li>worker1의 CNI가 라우팅 테이블을 보고, 10.44.0.0/24는 worker2로 전송</li>
<li>패킷을 VXLAN으로 캡슐화해서 바깥의 Node IP들간 터널로 보냄<pre><code>Inner packet: # 대부 패킷
src: 10.36.0.5 (Pod)
dst: 10.44.0.7 (Pod)
</code></pre></li>
</ol>
<p>Outer packet: # 캡슐화 후 외부 패킷
  src: 10.0.0.11 (worker1 Node IP)
  dst: 10.0.0.12 (worker2 Node IP)</p>
<pre><code>4. worker2가 VXLAN 캡슐을 벗겨서(디캡슐화), 다시 10.44.0.7 대상 패킷을 weave 브리지로 보냄
5. 해당 Pod의 veth를 찾아서 전달 → Pod에 도착

&gt; 그래서 Pod IP가 서로 완전히 다른 대역인데도(Node가 달라서) 붙는 이유는 CNI가 Node들 사이에 **가상 터널(Overlay)**을 만들어 놨기 때문이다. Node끼리는 원래 있던 10.0.0.x 네트워크로 통신!

## Service IP(ClusterIP)
Pod IP만 있다면 Pod가 죽고 다시 뜨면 IP가 바뀌고 여러 Pod들이 로드밸런싱 되야하는 점들이 해결이 되지 않는다. 그래서 Service IP(ClusterIP)가 등장한다.
</code></pre><p>apiVersion: v1
kind: Service
metadata:
  name: my-nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:</p>
<ul>
<li>port: 80       # 서비스 포트
targetPort: 80 # Pod 포트<pre><code>### Service가 생성될 때 일어나는 일
1. api-server가 Service 오브젝트 저장
2. Service CIDR (예: 10.96.0.0/12) 안에서 사용 가능한 IP를 하나 할당(예: 10.96.0.15)
3. `kubectl get svc`로 확인</code></pre>$ kubectl get svc
NAME       TYPE        CLUSTER-IP     PORT(S)
my-nginx   ClusterIP   10.96.0.15     80/TCP<pre><code>&gt; 10.96.0.15는 실제로 어떤 “물리 인터페이스에 붙은 IP”가 아니다. Node에 가서 ip addr 쳐봐도 10.96.0.15는 없음. 이 IP는 **kube-proxy + iptables 규칙에 의해 가상으로 존재하는 IP**다.
</code></pre></li>
</ul>
<h3 id="endpoints-연결">Endpoints 연결</h3>
<p>Service는 selector로 Pod들을 찾는다.</p>
<pre><code>$ kubectl get endpoints my-nginx
NAME       ENDPOINTS                        PORT(S)
my-nginx   10.36.0.5:80,10.44.0.7:80       80/TCP</code></pre><p>여기서 <strong>Endpoints 리스트는 실제 Pod IP 목록</strong>이다.
Service는 그냥 이 IP들 묶음에 대한 대표 이름 + ClusterIP를 주는 개념</p>
<h3 id="kube-proxy가-실제-iptables-규칙-생성">kube-proxy가 실제 iptables 규칙 생성</h3>
<p>각 Node에 있는 <strong>kube-proxy</strong>가 api-server에서 Service/Endpoints 목록을 감시하고 각 Service에 대해 iptables 규칙을 만든다.</p>
<ol>
<li>클러스터 안의 어떤 Pod가 10.96.0.15:80으로 요청을 보냄</li>
<li>Node의 iptables/kube-proxy가 이걸 잡음</li>
<li>Endpoints 중 하나인 10.36.0.5:80 또는 10.44.0.7:80으로 NAT해서 전달<blockquote>
<p>그래서 <strong>ClusterIP는 실제로 존재하는 IP가 아니라, iptables로 구현된 가상의 IP</strong>라고 볼 수 있다.</p>
</blockquote>
</li>
</ol>
<h2 id="nodeport--loadbalancer">NodePort &amp; LoadBalancer</h2>
<h3 id="nodeport">NodePort</h3>
<p>Service를 NodePort로 하나 만든다.</p>
<pre><code>spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080</code></pre><p>이 때 IP/Port 구조</p>
<ul>
<li>ClusterIP: 10.96.0.15:80</li>
<li>NodePort: &lt;모든 Node IP&gt;:30080</li>
</ul>
<p>kube-proxy는 각 Node에서 30080으로 들어온 요청 → 해당 서비스로 라우팅</p>
<pre><code>[외부 클라이언트]
   ↓
Node1(10.0.0.11):30080
   ↓ (kube-proxy iptables)
10.96.0.15:80 (ClusterIP로 간다고 생각해도 됨)
   ↓
Endpoints(10.36.0.5:80 / 10.44.0.7:80)
   ↓
실제 Pod</code></pre><p>여기서 NAT는 두 번쯤 일어난다.</p>
<ul>
<li>NodePort → ClusterIP (또는 바로 Endpoints)</li>
<li>Endpoints → Pod</li>
</ul>
<blockquote>
<p>NodePort가 열리는 IP는 Node의 실제 IP(10.0.0.x)이고 내부에서 다시 Pod IP(10.36/10.44 등)로 흘러간다.</p>
</blockquote>
<h3 id="loadbalancer">LoadBalancer</h3>
<p>클라우드 환경에서</p>
<pre><code>spec:
  type: LoadBalancer</code></pre><p>라고 하면, 클라우드에서 이런 걸 자동으로 해준다.</p>
<ul>
<li>외부용 퍼블릭 IP: 예) 3.3.3.3:80</li>
<li>LB → 각 Node의 NodePort (10.0.0.11:30080, 10.0.0.12:30080)</li>
</ul>
<pre><code>[인터넷 클라이언트]
   ↓
퍼블릭 IP (3.3.3.3:80, Cloud LB)
   ↓
Node1 10.0.0.11:30080
Node2 10.0.0.12:30080
   ↓ (각 Node의 kube-proxy/IPTables)
ClusterIP / Endpoints
   ↓
Pod IP</code></pre><p>즉, LoadBalancer는 <strong>NodePort를 여러 Node에 분산해서 때려주는 앞단 장치</strong>라고 보면 된다.</p>
<h2 id="ingress까지-포함한-전체-흐름">Ingress까지 포함한 전체 흐름</h2>
<p>이제 Ingress + Ingress Controller가 들어오면 IP 관점에서 한 단계 더 추가된다.</p>
<p>Ingress Controller (Nginx)가 ingress-nginx-controller라는 Service(LB 타입)로 노출</p>
<pre><code>apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: my-nginx
            port:
              number: 80</code></pre><ol>
<li>클라이언트가 test.example.com → 3.3.3.3 (LB IP)로 접속</li>
<li>3.3.3.3:80 → NodePort → Ingress Controller Pod(예: 10.36.0.20)</li>
<li>Ingress Controller(Nginx)가 Ingress 규칙을 보고, /app 요청은 my-nginx 서비스로 보내야 한다고 판단</li>
<li>Controller는 my-nginx 서비스의 Endpoints 리스트를 본다.<ul>
<li>10.36.0.5:80, 10.44.0.7:80</li>
</ul>
</li>
<li>직접 해당 Pod IP로 프록시(L7 레벨에서 HTTP 요청 전달)</li>
</ol>
<p>참고) ChatGPT가 IP들이 왜 이렇게 생기고, 왜 이런 구조를 가지는가? 에 대해서 표로 정리</p>
<table>
<thead>
<tr>
<th>구분</th>
<th>예시 IP</th>
<th>왜 이런 IP인가?</th>
<th>누가 관리?</th>
</tr>
</thead>
<tbody><tr>
<td>Node IP</td>
<td>10.0.0.10</td>
<td>VPC/Subnet or 물리망에서 할당</td>
<td>클라우드/하이퍼바이저/관리자</td>
</tr>
<tr>
<td>Pod CIDR</td>
<td>10.32.0.0/12</td>
<td>kubeadm init 시 지정</td>
<td>클러스터 설정</td>
</tr>
<tr>
<td>Node별 Pod CIDR</td>
<td>10.36.0.0/24</td>
<td>CNI가 Pod CIDR을 Node별로 분할</td>
<td>CNI</td>
</tr>
<tr>
<td>Pod IP</td>
<td>10.36.0.5</td>
<td>Node별 Pod CIDR 중에서 CNI가 할당</td>
<td>CNI</td>
</tr>
<tr>
<td>Service CIDR</td>
<td>10.96.0.0/12</td>
<td>kubeadm init 시 지정</td>
<td>클러스터 설정</td>
</tr>
<tr>
<td>ClusterIP</td>
<td>10.96.0.15</td>
<td>Service CIDR 중 사용 가능한 IP</td>
<td>API 서버 + kube-proxy</td>
</tr>
<tr>
<td>Endpoints</td>
<td>10.36.0.5:80</td>
<td>실제 Pod IP &amp; Port</td>
<td>kube-controller-manager</td>
</tr>
<tr>
<td>NodePort</td>
<td>NodeIP:30080</td>
<td>Node 실제 IP + Cluster에서 예약한 포트</td>
<td>kube-proxy</td>
</tr>
<tr>
<td>LoadBalancer IP</td>
<td>3.3.3.3</td>
<td>클라우드가 제공하는 외부 퍼블릭 IP</td>
<td>클라우드 LB</td>
</tr>
<tr>
<td>Ingress Controller Pod</td>
<td>10.36.0.20</td>
<td>다른 Pod와 동일하게 CNI로 할당</td>
<td>CNI</td>
</tr>
</tbody></table>