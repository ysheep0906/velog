<p><img alt="" src="https://velog.velcdn.com/images/ysheep0906/post/0b2da850-400a-4776-aa30-c8b9c0c5a5d7/image.png" /></p>
<h2 id="테라폼-설계">테라폼 설계</h2>
<p>테라폼 코드를 잘 짜는 것보다는, 확장 가능한 구조로 설계하는게 중요하다. 프로젝트가 작을 때는 문제가 없지만 프로젝트가 커진다면 코드 중복, 환경 별 분리 어려움, 변경 시 전체 수정과 같은 유지보수 지옥을 느껴야한다. 같은 코드를 <strong>개발/스테이징/운영 환경 모두 재사용</strong>할 수 있고 팀 규모가 커져도 <strong>인프라 구성이 안정적</strong>으로 되어야 한다!</p>
<h2 id="로컬-변수local로-코드-단순화">로컬 변수(local)로 코드 단순화</h2>
<p>예를 들어, 여러 서브넷을 각각 하드코딩했다면 이를 하나의 map 변수로 통합하고 locals 블록으로 관리할 수 있다.</p>
<pre><code>variable &quot;subnets&quot; {
  type = map(object({
    availability_zone = string
    cidr_block        = string
  }))
}

locals {
  nat_subnets = {
    for name, subnet in var.subnets : name =&gt; subnet
    if lookup(subnet, &quot;nat_gateway&quot;, false)
  }
}</code></pre><p>locals 블록을 통해 NAT가 필요한 서브넷(Private Subnet)만을 추출할 수 있다.
리팩터링 후에는 반드시 <code>terraform state mv</code>를 통해 상태 변경을 해야 한다.</p>
<h2 id="조건문으로-라우팅-테이블-자동-생성">조건문으로 라우팅 테이블 자동 생성</h2>
<p>lookup() 함수를 이용해 아래와 같이 자동 분기되도록 설계한다.</p>
<ul>
<li>퍼블릭 서브넷 → <strong>internet_gateway</strong> 사용</li>
<li>프라이빗 서브넷 → <strong>nat_gateway</strong> 사용<pre><code># 라우팅 테이블 생성
resource &quot;aws_route_table&quot; &quot;route_tables&quot; {
for_each = var.subnets
vpc_id   = aws_vpc.main.id
}
</code></pre></li>
</ul>
<h1 id="라우트-생성-igw-또는-natgw-자동-선택">라우트 생성 (IGW 또는 NATGW 자동 선택)</h1>
<p>resource &quot;aws_route&quot; &quot;routes&quot; {
  for_each = var.subnets</p>
<p>  route_table_id         = aws_route_table.route_tables[each.key].id
  destination_cidr_block = &quot;0.0.0.0/0&quot;</p>
<p>  gateway_id     = lookup(each.value, &quot;nat_gateway_subnet&quot;, null) == null ? aws_internet_gateway.main.id : null
  nat_gateway_id = lookup(each.value, &quot;nat_gateway_subnet&quot;, null) != null ? aws_nat_gateway.gateways[each.key].id : null
}</p>
<h1 id="라우트-테이블과-서브넷-연결">라우트 테이블과 서브넷 연결</h1>
<p>resource &quot;aws_route_table_association&quot; &quot;associations&quot; {
  for_each = var.subnets</p>
<p>  subnet_id      = aws_subnet.subnets[each.key].id
  route_table_id = aws_route_table.route_tables[each.key].id
}</p>
<pre><code>
## 동적 블록(dynamic) 사용하기
**보안그룹**은 규칙마다 인바운드/아웃바운드가 다르고, 포트나 CIDR도 다양하다. 이걸 for_each와 dynamic으로 처리하면 훨씬 유연해진다.
main.tf</code></pre><p>resource &quot;aws_security_group&quot; &quot;sg&quot; {
  for_each = var.security_groups</p>
<p>  name   = each.key
  vpc_id = aws_vpc.main.id</p>
<p>  dynamic &quot;ingress&quot; {
    for_each = each.value.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }</p>
<p>  dynamic &quot;egress&quot; {
    for_each = each.value.egress_rules
    content {
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }</p>
<p>  tags = {
    Name = each.key
  }
}</p>
<pre><code>
variables.tf</code></pre><p>variable &quot;security_groups&quot; {
  description = &quot;Map of security groups with rules&quot;</p>
<p>  type = map(object({
    ingress_rules = list(object({
      from_port   = number
      to_port     = number
      protocol    = string
      cidr_blocks = list(string)
    }))
    egress_rules = list(object({
      from_port   = number
      to_port     = number
      protocol    = string
      cidr_blocks = list(string)
    }))
  }))
}</p>
<pre><code>
terraform.tfvars</code></pre><p>security_groups = {
  &quot;permit_ssh&quot; = {
    ingress_rules = [
      {
        from_port   = 22
        to_port     = 22
        protocol    = &quot;tcp&quot;
        cidr_blocks = [&quot;0.0.0.0/0&quot;]
      }
    ]
    egress_rules = [
      {
        from_port   = 0
        to_port     = 0
        protocol    = &quot;-1&quot;
        cidr_blocks = [&quot;0.0.0.0/0&quot;]
      }
    ]
  }</p>
<p>  &quot;permit_http_https&quot; = {
    ingress_rules = [
      {
        from_port   = 80
        to_port     = 80
        protocol    = &quot;tcp&quot;
        cidr_blocks = [&quot;0.0.0.0/0&quot;]
      },
      {
        from_port   = 443
        to_port     = 443
        protocol    = &quot;tcp&quot;
        cidr_blocks = [&quot;0.0.0.0/0&quot;]
      }
    ]
    egress_rules = [
      {
        from_port   = 0
        to_port     = 0
        protocol    = &quot;-1&quot;
        cidr_blocks = [&quot;0.0.0.0/0&quot;]
      }
    ]
  }
}</p>
<pre><code>
이렇게** dynamic + for_each**를 사용하게 되면 보안 그룹 수에 상관없이 코드 하나로 관리가 가능하다. **새 규칙이 생기면 `.tfvars`에만 추가하면 된다!**


## 모듈화 사용하기
VPC, Subnet, Security Group 등은 환경마다 비슷한 형태로 반복된다. 이를 **모듈(module)**로 묶으면 디렉터리만 바꿔도 동일한 구성을 재사용할 수 있다.
![](https://velog.velcdn.com/images/ysheep0906/post/b2cd9db3-ea1f-4793-abda-f8e867fb9082/image.png)</code></pre><p>module &quot;vpc&quot; {
  source = &quot;./modules/vpc&quot;
  vpc_cidr = &quot;10.0.0.0/16&quot;
}</p>
<p>module &quot;subnet&quot; {
  source = &quot;./modules/subnet&quot;
  subnets = var.subnets
}</p>
<pre><code>main.tf에는 module을 따라간다고 적어주면 된다.


## YAML 파일 활용하기
YAML 파일을 사용하면 변수 정의를 외부 파일로 분리할 수 있다. 조금 더 사람이 읽기에 가독성도 좋다.</code></pre><p>locals {
  config = yamldecode(file(&quot;${path.module}/config.yaml&quot;))
}</p>
<p>resource &quot;aws_instance&quot; &quot;web&quot; {
  ami           = local.config.ami
  instance_type = local.config.instance_type
}</p>
<pre><code>
## 보안그룹 개선과 terraform import 활용
보안그룹은 부분 변경만 해도 전체가 교체되는 경우가 있다. 이를 해결하려면 기존의 aws_security_group 단일 리소스를 aws_vpc_security_group_ingress_rule / egress_rule 로 분리해야 한다.
main.tf</code></pre><h1 id="보안-그룹-본체">보안 그룹 본체</h1>
<p>resource &quot;aws_security_group&quot; &quot;sg&quot; {
  for_each = var.security_groups</p>
<p>  name        = each.key
  description = &quot;Managed Security Group&quot;
  vpc_id      = aws_vpc.main.id</p>
<p>  tags = {
    Name = each.key
  }
}</p>
<h1 id="인바운드-규칙">인바운드 규칙</h1>
<p>resource &quot;aws_vpc_security_group_ingress_rule&quot; &quot;ingress&quot; {
  for_each = var.ingress_rules</p>
<p>  security_group_id = aws_security_group.sg[each.value.security_group_name].id
  cidr_ipv4         = each.value.cidr_ipv4
  from_port         = each.value.from_port
  to_port           = each.value.to_port
  ip_protocol       = each.value.ip_protocol
}</p>
<h1 id="아웃바운드-규칙">아웃바운드 규칙</h1>
<p>resource &quot;aws_vpc_security_group_egress_rule&quot; &quot;egress&quot; {
  for_each = var.egress_rules</p>
<p>  security_group_id = aws_security_group.sg[each.value.security_group_name].id
  cidr_ipv4         = each.value.cidr_ipv4
  from_port         = each.value.from_port
  to_port           = each.value.to_port
  ip_protocol       = each.value.ip_protocol
}</p>
<pre><code>
variables.tf</code></pre><h1 id="보안-그룹-정의-껍데기">보안 그룹 정의 (껍데기)</h1>
<p>variable &quot;security_groups&quot; {
  description = &quot;Map of security groups with rules&quot;
  type        = map(object({}))
}</p>
<h1 id="인바운드-규칙-1">인바운드 규칙</h1>
<p>variable &quot;ingress_rules&quot; {
  description = &quot;Ingress rules for security groups&quot;
  type = map(object({
    security_group_name = string
    cidr_ipv4           = string
    from_port           = number
    ip_protocol         = string
    to_port             = number
  }))
}</p>
<h1 id="아웃바운드-규칙-1">아웃바운드 규칙</h1>
<p>variable &quot;egress_rules&quot; {
  description = &quot;Egress rules for security groups&quot;
  type = map(object({
    security_group_name = string
    cidr_ipv4           = string
    from_port           = number
    ip_protocol         = string
    to_port             = number
  }))
}</p>
<pre><code>
config.yaml</code></pre><p>subnets:
  oimarket-apne2-private-subnet-b:
    cidr_block: &quot;10.0.12.0/24&quot;</p>
<p>security_groups:
  oimarket-apne2-permit-http-security-group: null
  oimarket-apne2-permit-ssh-security-group: null</p>
<p>ingress_rules:
  oimarket-apne2-permit-http-security-group-allow-80:
    security_group_name: &quot;oimarket-apne2-permit-http-security-group&quot;
    cidr_ipv4: &quot;0.0.0.0/0&quot;
    from_port: 80
    to_port: 80
    ip_protocol: &quot;tcp&quot;</p>
<p>  oimarket-apne2-permit-http-security-group-allow-443:
    security_group_name: &quot;oimarket-apne2-permit-http-security-group&quot;
    cidr_ipv4: &quot;0.0.0.0/0&quot;
    from_port: 443
    to_port: 443
    ip_protocol: &quot;tcp&quot;</p>
<p>  oimarket-apne2-permit-ssh-security-group-allow-22:
    security_group_name: &quot;oimarket-apne2-permit-ssh-security-group&quot;
    cidr_ipv4: &quot;0.0.0.0/0&quot;
    from_port: 22
    to_port: 22
    ip_protocol: &quot;tcp&quot;</p>
<p>egress_rules:
  oimarket-apne2-permit-http-security-group-allow-permit:
    security_group_name: &quot;oimarket-apne2-permit-http-security-group&quot;
    cidr_ipv4: &quot;0.0.0.0/0&quot;
    from_port: 0
    to_port: 0
    ip_protocol: &quot;-1&quot;</p>
<p>  oimarket-apne2-permit-ssh-security-group-allow-permit:
    security_group_name: &quot;oimarket-apne2-permit-ssh-security-group&quot;
    cidr_ipv4: &quot;0.0.0.0/0&quot;
    from_port: 0
    to_port: 0
    ip_protocol: &quot;-1&quot;</p>
<pre><code>
콘솔에서 만든 SG를 테라폼 상태로 가져와야 한다면 아래처럼 명령어를 실행한다.</code></pre><p>terraform import aws_security_group.web_sg sg-0a12b34c56d78ef90</p>
<pre><code>
</code></pre>