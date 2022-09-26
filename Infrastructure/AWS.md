## 인프라 관련 고민

### Index
* [AWS ECS 컨테이너 멀티포트 연결](#AWS-ECS-Service에서-컨테이너의-포트를-2개-이상-열어보자)
* [AWS에서 Route53, Loadbalancer 없이 서비스를 찾는 법](#AWS-Cloud-Map-도입)

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