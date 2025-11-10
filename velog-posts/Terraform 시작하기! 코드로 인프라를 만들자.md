<p><img alt="" src="https://velog.velcdn.com/images/ysheep0906/post/f0afd9bd-86bb-40bc-81a8-01b654334db5/image.png" /></p>
<h2 id="terraform이란-왜-써야할까">Terraform이란? 왜 써야할까</h2>
<p>지금의 인프라는 단순히 서버 몇 대로 끝나지 않는다. <strong>클라우드 환경</strong>에서는 수십 개의 리소스(VPC, 서브넷, 보안그룹, EC2, IAM 등)가 유기적으로 연결되어 동작한다. 이걸 사람이 일일이 콘솔에서 클릭으로 설정한다면 굉장히 어렵다.
그래서 등장한 개념이 바로 <strong>Infrastructure as Code(IaC)</strong>이다. 이것은 인프라를 코드로 정의하고 관리하는 방식이다. <strong>Terraform</strong>은 이 IaC 도구 중에서도 가장 널리 쓰이는 표준 도구이다. HashiCorp에서 개발한 오픈소스이며, AWS·GCP·Azure 등 멀티 클라우드를 모두 지원한다. </p>
<p>그리고 인프라를 _<strong>&quot;코드&quot;</strong>_로 선언한다면 반복 가능성, 형상관리, 협업 등에서 한층 쉬워진다. 그렇기에 이 Terraform을 배우고 실전으로 다뤄보면 좋을 것 같다.</p>
<h2 id="기본-구조">기본 구조</h2>
<p><img alt="" src="https://velog.velcdn.com/images/ysheep0906/post/5f5a7d71-264c-4f88-b9e9-f98f3ce03786/image.png" /></p>
<p>providers.tf: 인프라가 어디에 만들어질지
main.tf: 실제로 무엇을 만들지
variables.tf: 입력값(변수)
outputs.tf: 출력값
backend.tf: 상태 저장 위치</p>
<p>아래 워크플로우로 동작한다
<code>terraform init</code>(프로바이더/백엔드 준비) → <code>terraform plan</code>(변경 예고) → <code>terraform apply</code>(실제 반영)</p>
<h2 id="vpc-만들기">VPC 만들기</h2>
<p>providers.tf</p>
<pre><code>provider &quot;aws&quot; {
    region: &quot;ap=northest-2&quot;
}

terraform {
    required_version = &quot;= 1.9.5&quot;
}</code></pre><p>main.tf</p>
<pre><code>resource &quot;aws_vpc&quot; &quot;main&quot; {
    cidr_block = &quot;10.0.0.0/16&quot;

    tags = {
        Name = &quot;oimarket-apne2&quot;
    }
}</code></pre><h2 id="변수-사용하기">변수 사용하기</h2>
<p>variable 블록을 통해서 변수를 정의할 수 있다. <strong>var.&lt;변수 이름&gt;</strong>의 형태로 정의한 변수를 테라폼 구성 내에서 사용할 수 있다.
main.tf</p>
<pre><code>resource &quot;aws_vpc&quot; &quot;main&quot; {
    cidr_block = var.cidr_block

    tags = {
        Name = var.vpc_name
    }
}</code></pre><p>variables.tf</p>
<pre><code>variable &quot;cidr_block&quot; {
    description = &quot;The cidr block of vpc&quot;
}

variable &quot;vpc_name&quot; {
    description = &quot;The name of vpc&quot;
}</code></pre><p>그럼 변수 값은 어디에 있을까? <strong>terraform.tfvars</strong>
terraform.tfvars</p>
<pre><code>vpc_name = oimarket-apne2
cidr_block = 10.0.0.0/16</code></pre><h2 id="출력-사용하기">출력 사용하기</h2>
<p>output은 중요한 식별자나 주소를 *<em>외부로 내보낼 때 *</em>유용하다 (다른 구성과 연결하기 쉽다)
outputs.tf</p>
<pre><code>output &quot;vpc_id&quot; {
  value = aws_vpc.main.id
}</code></pre><h2 id="리소스-간-참조의존성-사용하기">리소스 간 참조/의존성 사용하기</h2>
<p>main.tf</p>
<pre><code>resource &quot;aws_vpc&quot; &quot;main&quot; {
    cidr_block = var.cidr_block

    tags = {
        Name = var.vpc_name
    }
}

resource &quot;aws_internet_gateway&quot; &quot;igw&quot; {
  vpc_id = aws_vpc.main.id  # 암시적 의존성
  tags = { Name = &quot;demo-igw&quot; }
}</code></pre><p>대부분의 의존성은 속성 참조만으로 암시적으로 잡힌다. 필요한 경우에만 <code>depends_on</code>으로 명시적 의존성을 추가할 수 있다.</p>
<h2 id="반복문-사용하기">반복문 사용하기</h2>
<p>동일한 타입의 리소스를 여러 개 만들 땐 반복이 답이다. 대표적으로 AZ별 서브넷을 반복 생성한다.</p>
<pre><code>resource &quot;aws_subnet&quot; &quot;public_a&quot; {
  vpc_id                   = aws_vpc.main.id
  cidr_block                 = &quot;10.0.1.0/24&quot;
  availability_zone       = &quot;ap-northeast-2a&quot;
  map_public_ip_on_launch = true
  tags = { 
      Name = &quot;${var.vpc_name}-public-subnet-a&quot;
  }
}

resource &quot;aws_subnet&quot; &quot;public_b&quot; {
  vpc_id                   = aws_vpc.main.id
  cidr_block                 = &quot;10.0.2.0/24&quot;
  availability_zone       = &quot;ap-northeast-2b&quot;
  map_public_ip_on_launch = true
  tags = { 
      Name = &quot;${var.vpc_name}-public-subnet-b&quot;
  }
}</code></pre><p>아래와 같이 반복문은 통해서 바꿔보자
main.tf</p>
<pre><code>resource &quot;aws_subnet&quot; &quot;public&quot; {
  count                   = length(var.public_azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone       = &quot;ap-northeast-2${var.public_azs[count.index]}&quot;
  map_public_ip_on_launch = true
  tags = { 
      Name = &quot;${var.vpc_name}-public-subnet-${var.public_azs[count.index]}&quot;
  }
}</code></pre><p>variables.tf</p>
<pre><code>variable &quot;public_azs&quot; {
    type = list(string)
}</code></pre><p>terraform.tfvars</p>
<pre><code>public_azs = [&quot;a&quot;, &quot;b&quot;]</code></pre><h2 id="상태-파일-사용하기">상태 파일 사용하기</h2>
<p>반복문 사용하기에서 리팩터링을 한 후 리소스 상태를 이전하고 관리할 수 있다.
<img alt="" src="https://velog.velcdn.com/images/ysheep0906/post/abc7e8ca-dcca-4c5e-b6b9-3ec2644d3cc7/image.png" /></p>
<pre><code>terraform state mv aws_subnet.public_a aws_subnet.public[0]
terraform state mv aws_subnet.public_b aws_subnet.public[1]
terraform plan</code></pre><h2 id="복잡한-변수-타입-사용하기">복잡한 변수 타입 사용하기</h2>
<p><img alt="" src="https://velog.velcdn.com/images/ysheep0906/post/96232b85-759f-40bd-babb-e7b7f41ed300/image.png" /></p>
<p>variables.tf</p>
<pre><code>variable &quot;subnets&quot; {
  type = map(object({
    availability_zone = string
    cidr_block        = string
  }))
}</code></pre><p>terraform.tfvars</p>
<pre><code>subnets = {
  public-a = {
    availability_zone = &quot;ap-northeast-2a&quot;
    cidr_block        = &quot;10.0.1.0/24&quot;
  }
  public-b = {
    availability_zone = &quot;ap-northeast-2b&quot;
    cidr_block        = &quot;10.0.2.0/24&quot;
  }
}</code></pre><p>main.tf</p>
<pre><code>resource &quot;aws_subnet&quot; &quot;public&quot; {
  for_each                = var.subnets
  vpc_id                  = aws_vpc.main.id
  availability_zone       = each.value.availability_zone
  cidr_block              = each.value.cidr_block
  map_public_ip_on_launch = true
  tags = { Name = &quot;public-${each.key}&quot; }
}</code></pre><h2 id="조건문-사용하기">조건문 사용하기</h2>
<p>테라폼의 조건문은 <strong>condition ? A : B</strong> 예를 들어 _<strong>프라이빗 서브넷이 존재할 때만 NAT GW를 AZ 수만큼 생성하도록 제어</strong>_할 수 있다. 
variables.tf</p>
<pre><code>variable &quot;private_subnets&quot; {
  description = &quot;Private subnets map&quot;
  type = map(object({
    availability_zone = string
    cidr_block        = string
  }))
}</code></pre><p>terraform.tfvars</p>
<pre><code>private_subnets = {
  &quot;oimarket-apne2-private-subnet-a&quot; = {
    availability_zone = &quot;ap-northeast-2a&quot;
    cidr_block        = &quot;10.0.11.0/24&quot;
  },
  &quot;oimarket-apne2-private-subnet-b&quot; = {
    availability_zone = &quot;ap-northeast-2b&quot;
    cidr_block        = &quot;10.0.12.0/24&quot;
  }
}</code></pre><p>main.tf</p>
<pre><code># NAT Gateway용 EIP
resource &quot;aws_eip&quot; &quot;nat_ips&quot; {
  count = var.private_subnets != {} ? length(var.private_subnets) : 0
}

# NAT Gateway 생성
resource &quot;aws_nat_gateway&quot; &quot;gateways&quot; {
  count = var.private_subnets != {} ? length(var.private_subnets) : 0

  allocation_id = aws_eip.nat_ips[count.index].id

  # public subnet의 values()를 리스트로 변환해 순서 접근
  subnet_id     = tolist(values(aws_subnet.public))[count.index].id

  tags = { Name = &quot;nat-gateway-${count.index}&quot; }
}</code></pre>