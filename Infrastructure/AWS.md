## 인프라 관련 고민

### Index
* [AWS ECS 컨테이너 멀티포트 연결](#AWS-ECS-Service에서-컨테이너의-포트를-2개-이상-열어보자)
* [AWS에서 Route53, Loadbalancer 없이 서비스를 찾는 법](#AWS-Cloud-Map-도입)
* [ECS의 서비스 컨테이너에 ssh 처럼 접속해보자](#Fargate로-돌아가는-서비스-컨테이너에-접속하는-방법)
* [Config Server 구축 대신 AWS Parameter Store 사용](#환경변수-파라미터-관리용-aws-parameter-store를-써보자)

---
#### AWS ECS Service에서 컨테이너의 포트를 2개 이상 열어보자

- 보통 멀티 컨테이너 환경에서 같은 도메인으로 묶을 경우 Load balancer를 앞에 달아 놓는데 아마존 콘솔 웹페이지에서는 해당 로드 밸런서 설정에서 컨테이너 포트는 단 하나만 연결 할 수 있다.

- 예를 들어 API서버 프로토콜 HTTP/1.1 :80 하나에 추가적으로 RPC(GRPC) 통신을 위해 HTTP/2 :8080을 추가로 띄운다고 가정하면 모두 로드밸런서에 달아야하나 불가능 함

- 콘솔이 안되니 **aws-cli**
    
    - command (요렇게 하면 서비스 생성에 대한 스켈레톤 스펙을 얻을 수 있다)
    ```bash
    aws ecs create-service --generate-cli-skeleton > create-service-skeleton.json
    ```
    - point (로드밸런서 설정이 일단 List니 설정이 되긴 되는구나)
    ```vim
    "loadBalancers": [
        {
            "targetGroupArn": "",
            "loadBalancerName": "",
            "containerName": "",
            "containerPort": 0
        }
    ],
    ```
    - sample (컨테이너 정의에서 사용되는 포트를 매치 시킨다)
    ```vim
    "loadBalancers": [
        {
            "targetGroupArn": "arn:aws:elasticloadbalancing:<Region>:<Account>:targetgroup/<targetGroupName>/<UNIQUE>",
            "containerName": "api_container",
            "containerPort": 80
        },
        {
            "targetGroupArn": "arn:aws:elasticloadbalancing:<Region>:<Account>:targetgroup/<targetGroupName>/<UNIQUE>",
            "containerName": "api_container",
            "containerPort": 10080
        }
    ]
    ```
    - execute
    ```bash
    aws ecs create-service --cli-input-json file://<작성파일명>.json --region <리전>
    ```
#### AWS Cloud Map 도입
- 쿠버네티스나 ECS 환경(EC2)에서 컨테이너 간의 통신의 경우 노드(호스트)의 가상 네트워크 레이어에 구성되기 때문에 서로 간의 주소를 모르게 된다.
- 보통 컨테이너끼리 통신을 할경우 link 설정하나 이거도 결국 한계가 명확
- AWS의 경우 ECS의 컨테이너를 묶어서 로드밸런서(ALB) 도메인으로 호출
- 서비스간 관리(헬스 체크, 스케일링)를 하기 위해 LB 서비스를 추가로 넣어야하지만 LB서비스도 관리 대상이 된다. (타겟 그룹 관리 등)
- cloud map을 설정하면 해당 서비스 간의 위치를 공통적인 리소스 검색을 통해 찾을 수 있다.
- 일단 LB 같은 추가 서비스를 사용하는 것 보단 쉽게 설정이 가능 (ECS, EKS)
- 그룹핑할 인프라 이름은 Namespace
- 그 Namespace를 구성하는 각각의 Service들을 생성하면 서비스 별로 DNS 쿼리검색 제공 및 서비스의 상태관리를 도와준다.
- DNS 쿼리 주소는 <서비스이름>.<네임스페이스> 형식
- 예를 들어 cloudmap에서 echoservice라는 서비스 이름에 네임스페이스는 domaina라는 네임스페이스면 해당 서비스의 네임서버는 echoservice.domaina가 된다.
- `nslookup echoservice.domaina`로 DNS 쿼리가 가능
- 이렇게 하면 load balancer 서비스를 사용 하지 않고 서비스 관리 및 검색이 가능

#### Fargate로 돌아가는 서비스 컨테이너에 접속하는 방법
- ec2로 호스트 노드를 띄울 경우 EC2 -> ssh 접속 -> docker attach로 붙을 수 있다.
- fargate의 경우 호스트 노드를 aws에서 직접 관리하기 때문에 호스트 노드의 주소를 모르기 때문에 접속을 하지 못한다.
- 컨테이너에 sshd를 설치 & 세팅하고 접속이 가능하지만 그만큼 이미지 크기 및 빌드의 시간 코스트가 발생
- 개인적인 생각으론 컨테이너 자체에 접근해야 하는 게 마이크로서비스를 사용하는 기본 이데올로기에 반한다고 생각함 -> 컨테이너가 하는 기능이 복잡해지면 모노리스랑 별 다를 바가 없다. (로그를 파일로 남기고 서비스 기능도 해야하고)
- 모노리스 시스템에서 문제가 생기면 전체가 문제가 생기기 때문에 들어가서 무슨 문제인지 봐야하지만 컨테이너는 문제가 생기면 새롭게 컨테이너를 띄우고 비정상 컨테이너를 내리기만 하면 된다. (오케스트레이션에게 맡기자) 이렇게 하지 않으면 틀린게 아니라 굳이 관리해준다는데 내가 일을 벌리자는 주의는 아니라서 그렇다.
- 하지만 개발 중의 컨테이너의 정상동작과 테스트를 하기 위한 경우엔 간혹 앱 기동 스크립트가 안 돌아간다는지 등의 확인이 필요한 경우에 컨테이너에 접속해야 한다. (물론 기동 자체를 안하면 바로 종료지만..)
- ecs 서비스에 접속하기 위해서는 ECS execute-command 옵션을 켜줘야한다 (aws-cli)로 해당 서비스가 옵션이 켜져있는지 확인하고 서비스 수정을 하거나 새 서비스를 만들 때 옵션을 켜고 만들어야한다.
- 서비스 확인 커맨드
    ```bash
    aws ecs describe-tasks --cluster cluster-name --tasks task-id
    ```
    ```json
    {
    "tasks": [
        {
            ...
            "containers": [
                {
                    ...
                    "managedAgents": [
                        {
                            "lastStartedAt": "2021-03-01T14:49:44.574000-06:00",
                            "name": "ExecuteCommandAgent",
                            "lastStatus": "RUNNING"
                        }
                    ]
                }
            ],
            ...
            "enableExecuteCommand": true, // <-- 해당 플래그
            ...
        }
    ]
    }
    ```
- 서비스 생성 커맨드
    ```bash
    aws ecs create-service \
    --cluster cluster-name \
    --task-definition task-definition-name \
    --enable-execute-command \
    --service-name service-name \
    --desired-count 1
    ```
- 접속 커맨드
    ```bash
    aws ecs execute-command --cluster cluster-name \
    --task task-id \
    --container container-name \
    --interactive \
    --command "/bin/sh" # 물론 bash를 지원하면 bash로 해도 됨
    ```

#### 환경변수-파라미터 관리용 AWS Parameter Store를 써보자
- 개발 서비스가 점점 복잡해지면 복잡해질수록 관리해야하는 변수들이 늘어나는게 현실
- 클라우드 환경 + 마이크로 서비스 일수록 변수관리등의 실수가 벌어질 수 있다.
- Config Server 등의 변수 관리 서버를 두고 관리가 용이하게 하도록 도입하고 있는 추세
- Server 구성을 위한 약간의 코스트도 귀찮은데 클라우드(AWS)를 쓴다면 Parameter Store 서비스를 이용해서 간단하게 구성
- Spring 스택을 사용한다면 Spring Cloud에 있는 Config Server로 구성할 수 있지만 [awspring](https://docs.awspring.io/spring-cloud-aws/docs/2.4.1/reference/html/index.html#integrating-your-spring-cloud-application-with-the-aws-parameter-store) 라이브러리를 사용하여 클라우드 인프라 서비스를 쉽게 통합하여 사용할 수 있다.
- <details><summary><b>yml example</b></summary>

    ```yml
    aws:
    paramstore:
        prefix: /prefix  # 파라미터 prefix
        fail-fast: true  # 못 가져오면 기동 실패
        profile-separator: "-" # 이름과 프로파일을 나누는 seperator
        name: testServer # 이름
    spring:    
    application:
        name: testServer
    config:
        activate:
          on-profile: <profile>
        import: "aws-parameterstore:" #파라미터스토어에서 import 받겠다
    datasource:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: <DatasourceURI>
        username: <USERNAME>
        password: ${db.password}
    ```
</details>

- 위 yml 파일이 가르키는 parameter store의 name은 `/prefix/testServer-{profile}/db.password`가 된다
- 템플릿처럼 바꾸면 `/{prefix}/{name}{seperator}{profile}/{tag}`