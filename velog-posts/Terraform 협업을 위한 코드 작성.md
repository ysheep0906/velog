<p><img alt="" src="https://velog.velcdn.com/images/ysheep0906/post/e35514f8-f669-4644-8d1c-8fb8a05aee84/image.png" /></p>
<h2 id="상태-파일이란">상태 파일이란?</h2>
<p>Terraform은 인프라의 현재 상태를 상태 파일(terraform.tfstate) 에 저장한다. 이 파일에는 아래와 같은 정보들이 포함된다.</p>
<ul>
<li>실제 클라우드 리소스의 ID</li>
<li>속성값(VPC CIDR, EC2 인스턴스 타입 등)</li>
<li>Provider 및 모듈 버전</li>
<li>리소스 간 의존 관계</li>
</ul>
<blockquote>
<p>즉, tfstate는 Terraform이 *<em>현재 어떤 인프라를 관리 중인지 *</em>아는 핵심 데이터베이스이다!</p>
</blockquote>
<h2 id="로컬-상태-파일의-한계">로컬 상태 파일의 한계</h2>
<p>로컬<code>./terraform.tfstate</code>에서 상태를 관리하면 혼자 쓸 때는 괜찮지만, 협업에선 문제가 생긴다.</p>
<ul>
<li>여러 명이 동시에 apply를 하게 된다면 상태 꼬임이 발생하여 <strong>충돌의 위험성</strong>이 있다.</li>
<li>개인 PC에 저장했다가 파일을 잃어버리면 <strong>복구 불가</strong></li>
<li>팀원마다 다른 인프라 상태 관리로 인해 <strong>버전 불일치</strong></li>
</ul>
<p>이러한 문제를 해결하기 위해 <strong>원격 저장소(Remote Backend)</strong>를 사용한다.</p>
<h2 id="원격-상태-관리">원격 상태 관리</h2>
<p>많은 원격 상태 관리들이 있지만 이 글에서는 S3 버킷을 원격 저장소로 지정하고 DynamoDB를 상태 잠금(State Locking) 용도로 사용한다. </p>
<pre><code>terraform {
  backend &quot;s3&quot; {
    bucket         = &quot;my-terraform-state&quot;
    key            = &quot;network/terraform.tfstate&quot;
    region         = &quot;ap-northeast-2&quot;
    dynamodb_table = &quot;terraform-lock&quot; # 미리 만들어놓은 Table 지정
    encrypt        = true
  }
}</code></pre><ul>
<li>S3 → 상태 파일 저장</li>
<li>DynamoDB → 동시에 두 명이 terraform apply 하지 못하게 잠금 기능 제공</li>
</ul>
<p>이제 누구나 같은 원격 상태를 참조하므로, 팀 전체가 동일한 인프라 상태를 공유하게 된다.</p>
<h2 id="리모트-데이터terrform_remote_state">리모트 데이터(terrform_remote_state)</h2>
<p>그럼 이렇게 저장된 원격 상태를 다른 테라폼 구성에서 재사용할 수 있을까? 예를 들어, <strong>“VPC 구성은 따로 있고 EC2 구성을 별도로 관리하는”</strong> 경우이다. 이를 구현하기 위해 <code>terraform_remote_state</code> 데이터 소스를 사용한다.</p>
<p>vpc/main.tf</p>
<pre><code>output &quot;public_subnet_id&quot; {
  value = aws_subnet.public.id
}

output &quot;web_sg_id&quot; {
  value = aws_security_group.web_sg.id
}</code></pre><p>ec2/main.tf</p>
<pre><code>data &quot;terraform_remote_state&quot; &quot;vpc&quot; {
  backend = &quot;s3&quot;
  config = {
    bucket = &quot;my-terraform-state&quot;
    key    = &quot;vpc/terraform.tfstate&quot;
    region = &quot;ap-northeast-2&quot;
  }
}

resource &quot;aws_instance&quot; &quot;web&quot; {
  ami           = &quot;ami-0abcdef1234567890&quot;
  instance_type = &quot;t3.micro&quot;
  subnet_id     = data.terraform_remote_state.vpc.outputs.public_subnet_id
  vpc_security_group_ids = [data.terraform_remote_state.vpc.outputs.web_sg_id]
}</code></pre><p>EC2 구성은 <strong>VPC의 출력을 직접 참조</strong>한다. 서로 다른 프로젝트여도 동일한 상태 데이터를 공유할 수 있다.</p>
<h2 id="버전-관리">버전 관리</h2>
<p>협업 환경에서 가장 자주 발생하는 문제 중 하나는 버전 불일치다. 개발자는 1.9.3, 운영자는 1.8.8을 쓰고 있다면 동일한 코드라도 결과가 달라질 수 있다. 이를 방지하기 위해<code>required_version</code> 과 <code>required_providers</code> 블록을 사용한다.</p>
<p>providers.tf</p>
<pre><code>terraform {
  required_version = &quot;&gt;= 1.9.5&quot;

  required_providers {
    aws = {
      source  = &quot;hashicorp/aws&quot;
      version = &quot;~&gt; 5.82.0&quot;
    }
  }
}</code></pre><ul>
<li>~&gt; 연산자 → 예: ~&gt; 5.82.0 은 5.82.x까지 허용 (메이저 버전 고정)</li>
<li><blockquote>
<p>= 연산자 → 예: &gt;= 5.82.0은 5.83, 5.84, ... 지정 버전 위로 다 허용</p>
</blockquote>
</li>
</ul>