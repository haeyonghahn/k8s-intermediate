# k8s-intermediate
해당 문서 출처는 [대세는 쿠버네티스 [초급~중급]](https://www.inflearn.com/course/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EA%B8%B0%EC%B4%88) 기반으로 작성되었습니다. 

## Pod - Lifecycle
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/e3018d06-4e42-4438-a6a3-b2ec0b2ff2b8)

라이프사이클의 특징은 각 단계에 따라 행해지는 행동이 다르다. ReadinessProbe, LivenessProbe, QoS, 각각의 Policy 등 여러 기능들이 Pod에 특정 단계와 밀접한 관련이 있다.

![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/8b674deb-28f3-41f1-8ca7-faef3179e666)

Pod가 있고 status 안에 Pod 전체 상태를 대표하는 속성이 있다. 그리고 Pod가 생성되면서 실행하는 단계들이 있는데 그 단계와 상태를 알려주는 게 conditions라는 속성이다. Pod 안에는 컨테이너가 있다. 컨테이너마다 State가 있고 컨테이너의 상태가 있다. Pod의 Phase에는 Pending, Running, Succeeded, Failed, Unkown 상태가 있다. Pending은 Pod가 처음 만들어지기 시작하는 시점이다. conditions는 4가지 종류가 있고 reason이라고 해서 status가 false인 경우 false의 원인이 무엇인지 알아야 되기 때문에 reason이 존재하고 ContainersNotReady라고 해서 컨테이너가 아직 작업이 진행중이라는 것을 알 수 있다. 다음엔 ContainerStatuses를 보면 상태는 Wating, Running, Terminateed 세 가지 상태가 있다. conditions와 마찬가지로 세부 내용을 알기 위한 Reason이 있다. yml 파일을 보면 유추할 수 있는 부분이 imageID 값이 없는 것으로 보아 image가 아직 다운로드되지 않았다는 것을 알 수 있다.

```yml
status:
  phase: Pending
  conditions:
    - type: Initialized
      status: 'True'
      lastProbeTime: null
      lastTransitionTime: '2019-09-26T22:07:56Z'
    - type: PodScheduled
      status: 'True'
      lastProbeTime: null
      lastTransitionTime: '2019-09-26T22:07:56Z'
    - type: ContainersReady
      status: 'False'
      lastProbeTime: null
      lastTransitionTime: '2019-09-26T22:08:11Z'
      reason: ContainersNotReady
    - type: Ready
      status: 'False'
      lastProbeTime: null
      lastTransitionTime: '2019-09-26T22:08:11Z'
      reason: ContainersNotReady
containerStatuses:
  - name: container
    state:
      waiting:
        reason: ContainerCreating
    lastState: {}
    ready: false
    restartCount: 0
    image: tmkube/init
    imageID: ''
    started: false
```

![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/124fc046-28d4-4d56-bba3-7f22bd25becb)

먼저, Pod의 최초 상태는 Pending이고 해당 상태일 때 Pod 안에서 일어나는 일들을 보자.   
- initContainer : 본 컨테이너가 기동되기 전에 초기화시켜야 되는 내용들이 있을 경우이다. 만약 볼륨이나 보안 세팅을 위해 사전 설정을 해야 되는 일이 있을 경우 Pod 생성 내용 안에 initContainer라는 항목으로 초기화 스크립트를 넣을 수가 있고 해당 스크립트가 본 컨테이너보다 먼저 실행이 돼서 그 작업이 성공적으로 끝났거나 또는 아예 설정을 하지 않았을 경우 True, 그리고 실패를 하게 되면 False가 된다.
- PodScheduled : Pod가 어느 노드 위에 올라갈지 직접 노드를 지정했을 경우에는 지정한 해당 노드에, 아니면 Kubernetes가 알아서 자원의 상황을 판단해서 Node를 결정하기도 하는데 이게 완료가 되면 PodScheduled는 True가 된다. Pod의 동작 순서 과정은 PodScheduled가 먼저 동작한 후 Initialized를 한다.

다음으로 컨테이너의 이미지를 다운로드하는 동작이 있고 컨테이너 상태는 Wating이고 reason은 ContainerCreating이다. 그리고 이제 본격적으로 컨테이너가 기동되면서 Pod와 컨테이너의 상태는 Running이 된다. 이 때 컨테이너가 기동 중에 문제가 발생해서 재시작될 수도 있다. 그럼 이때 컨테이너의 상태는 wating이 되고 crash-rollback-off라는 reason이 나온다. Pod는 이러한 상태에 대해서 Running이라고 간주하고 내부 conditions의 containerReady와 Ready는 False이다. 그래도 결국 모든 컨테이너들이 정상적으로 기동이 돼서 원활하게 돌아간다면 condition은 모두 True로 변경이 된다.

그리고 이제 Job이나 CronJob으로 생성된 Pod의 경우 자신의 일을 수행했을 때는 Running 중이지만 일을 마치게 되면 Pod는 더 이상 일을 하지 않는 상태가 되는데 여기서 Pod의 상태는 Failed나 Succeeded 두 가지로 갈릴 수가 있다. 만약 작업을 하고 있는 컨테이너 중에 하나라도 문제가 생겨서 에러가 되면 Pod의 상태는 Failed가 되는 것이고 컨테이너들이 모두 Completed로 해당 일을 맞췄을 때 Succeeded가 된다. 이때 또 Pod의 컨디션 값이 변하게 되는데 성공이건 실패건 간에 ContainerReady와 Ready의 값이 false로 바뀌게 된다.

추가적으로 Pending 중에 바로 Failed로 빠지는 경우가 있다. Pending이라 Running 중에 통신 장애가 발생하면 Pod가 unknown 상태로 바뀌는데 통신 장애가 빨리 해결이 되면 다시 기준 상태로 변경이 되지만 계속 지속이 되면 Failed로 가기도 한다.

## Pod - ReadinessProbe, LivenessProbe
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/539b4e68-6b11-46c2-bbe5-0cdcd7e40e1a)

Pod를 만들면 그 안에 컨테이너가 생기고 Pod와 컨테이너의 상태가 Running되면서 그 안에 있는 앱도 정상적으로 구동이 되고 있을 것이다. 그리고 서비스에 연결이 되고 이 서비스의 ip가 외부에 알려지면서 외부에는 이 서비스를 통해 많은 사람들이 실시간으로 접근하게 되는 상황이 된다. 그럼, 한 서비스에 2개의 Pod가 연결되어 있고 50%씩 트래픽이 나눠진다고 가정해보자. 이렇게 서비스가 운영되고 있다가 Node2 서버가 다운이 되었다. 그럼 Pod2도 Fail이 되면서 사용자는 Pod1로 100% 접속을 하게 될 것이다. 그리고 Pod1이 트래픽을 견뎌준다면 서비스가 유지되는데 문제는 없을 것이다. 이제 여기서 Pod2는 오토 힐링 기능에 의해서 다른 노드에 재생성될 것이다. 그 과정에서 Pod와 컨테이너가 Running 상태가 되면서 서비스와 연결이 되는데 아직 앱이 구동 중인 순간에 서비스와 연결이 되자마자 트래픽이 Pod2로 유입이 되게 때문에 앱이 구동되고 있는 순간에 사용자는 50% 확률로 에러 페이지를 보게 된다. 근데 Pod를 만들 때 `ReadinessProbe`를 주게 되면 이런 문제를 피할 수 있다.  `ReadinessProbe`는 앱이 구동되기 전까지 서비스와 연결이 되지 않게 해준다. 그래서 Pod2의 상태는 Running 중이지만 트래픽은 계속 Pod1한테만 가게 된다. 앱이 완전히 준비된 게 확인이 되면 서비스와 연결이 되면서 다시 트래픽은 50대 50으로 흐르게 된다.

![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/d3b8ea00-ab03-4ee0-a114-09578ac76f68)

그래서 다시 앱이 구동되고 있는데 장애가 발생했다. 근데 Pod는 러닝 상태이다. 이럴 때 흔히 서비스를 Tomcat으로 돌릴 때 Tomcat은 돌고 있지만 그 위에 띄워져 있는 앱이 메모리 오버플로우라든지 문제가 생겨서 500 Interval Error 자체의 프로세스가 죽은 게 아니라 그 위에 돌고 있는 서비스에 문제가 생긴 거라 Tomcat 프로세스를 보고 있는 Pod 입장에서는 계속 Running 상태로 있게 되고 이 상황이 되면 흐르던 트래픽은 다시 문제가 된다. 이때 앱에 대한 장애 상황을 감지해주는 게 `LivenessProbe`인데 마찬가지로 Pod를 만들 때 `LivenessProbe`를 달아주면 해당 앱에 문제가 생기면 Pod를 재실행하게 만들어서 잠깐의 트래픽 에러는 발생하겠지만 지속적으로 에러 상태가 되는 것을 방지해준다. 

## Pod - QoS Classes
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/9c08f6a4-ba2b-4708-ba42-b171030d9443)

노드 위에 Pod가 3개 만들어져서 균등하게 자원을 모두 사용하고 있는 상태라고 가정해보자. 근데 이 상태에서 Pod1이 추가적으로 자원을 더 사용해야 되는 상황이 발생했다. 근데 노드에는 추가적으로 할당받을 수 있는 자원이 없기 때문에 Pod1이 리소스 부족으로 에러가 나고 다운이 되어야 할까? 아니면 Pod2나 Pod3 중에 하나를 다운시키고 Pod1에 필요한 만큼의 자원을 적용해줘야할까? Kubernetes에서는 앱의 중요도에 따라 이런 걸 관리할 수 있도록 세 가지 단계로 `Quality of Service`를 지원해주고 있다. 위의 그림에서 `BestEffort`가 부여된 Pod가 가장 먼저 다운이 돼서 자원이 반환되고 Pod1은 필요한 양의 자원을 사용할 수 있게 된다. 다음엔 노드에 어느 정도 자원이 남아있지만 Pod2에서 그보다 많은 자원을 요구하는 상황이 됐다면 `Burstable`이 적용된 Pod가 다음으로 다운이 되면서 자원이 회수가 된다. 그래서 `Guaranteed`가 마지막으로 Pod를 안정적으로 유지시켜주는 클래스이다. Pod에 이런 클래스들을 보여할 수 있는지 알아보면 QoS는 특정 속성이 있어서 설정하는 것은 아니고 컨테이너에 리소스 설정이 있는데 여기에 Request와 Limit에 따라서 메모리와 CPU를 어떻게 설정하는지로 결정이 된다.

먼저, `Guaranteed`는 Pod에 여러 컨테이너가 있다면 모든 컨테이너마다 Request와 limit가 있어야 한다. 그리고 그 안에 메모리와 CPU 둘 다 설정이 되어 있어야 한다. 그리고 각 컨테이너 안에 설정된 메모리와 CPU 값은 Request와 limit가 같아야 한다. 이 규칙에 맞아야 Kubernetes가 이 Pod를 Guaranteed로 판단한다. 그리고 `BestEffort`는 Pod의 어떤 컨테이너도 Request와 limit가 설정되어 있지 않다면 BestEffort가 된다. `Burstable`은 Guaranteed와 BestEffort의 중간인데 Request와 limit는 설정되어 있지만 Request의 수치가 작던지, 아니면 Request만 설정되어 있고 Limit는 설정되어 있지 않는 경우. 또는 Pod에 컨테이너가 두 개인데 하나는 완벽하게 설정이 되어 있지만 다른 하나가 설정되어 있지 않다면 모두 Burstable 클래스가 된다.

Burstable이 여러 Pod가 있을 경우 누가 먼저 삭제가 되는지도 알 필요가 있다. OOM 스코어라고 해서 Out of Memory 스코어라는 것에 따라 결정이 된다. 예를 들어, 두 개의 Pod의 Request 메모리가 5Gi와 8Gi로 설정된 상태인데 그 안에 돌아가고 있는 앱에 실제 메모리가 똑같이 4Gi만큼 사용되고 있다고 가정하면 Pod2의 메모리 사용량은 75%이고 Pod3은 50% 이다. 그럼 OOM 스코어가 Pod2 75%로 더 크기 때문에 Kubernetes는 OOM 스코어가 더 큰 Pod2를 먼저 제거한다.

## Pod - Node Scheduling
Pod가 기본적으로 스케줄러에 의해서 노드에 할당이 되지만 사용자의 의도에 의해서 노드를 지정할 수도 있고 운용자가 특정 노드를 사용하지 못하도록 관리할 수 있다. Kubernetes는 다양한 기능들로 이것을 지원하고 있다. 

## Service - Headless, Endpoint, ExternalName
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/37a1a33b-864c-442a-a096-a9edb9cea8b8)

![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/4e44b84a-cc61-4545-ae56-4409b82ab056)

### Service - Headless
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/994b277a-2692-4bdc-9025-8cc6e0df9880)

default라는 Namespace에 Pod 2개와 서비스가 연결되어 있다. 이 서비스는 ClusterIp로 만든 서비스이다. 그리고 Kubernetes DNS가 있는데 이 DNS의 이름은 cluster.local이다. DNS는 파드건 서비스건 생성을 하면 긴 도메인 이름과 IP가 저장이 된다. 이름의 구조는 서비스의 경우 서비스 이름과 네임스페이스. 그리고 서비스의 약어인 svc 끝으로 DNS 이름이 붙어져서 만들어진다. 파드의 경우 IP의 네임스페이스, pod, DNS의 이름이 결합돼서 생성이 된다. 이렇게 규칙을 가지고 만들어지는데 이것을 `Fully Qualified Domain Name`이라고 부른다. 네임스페이스 안에서는 서비스는 앞자리만 짧게 써도 되지만 파드는 다 입력해야 한다. 그리고 앞부분이 IP이기 때문에 거의 쓸 수가 없다. 그럼 파드의 입장에서 서비스의 도메인 이름을 DNS 질의를 통해 IP를 가져오기 때문에 우리는 서비스의 이름만 알아도 해당 서비스에 접근할 수 있고 모든 이름들은 사용자가 직접 만드는 것이니까 미리 서비스의 이름을 예상해서 Pod에 미리 심어놓을 수 있다. 그래서 단순히 서비스에만 연결을 하는 데는 ClusterIp로 서비스르 만들어도 문제는 없다. 하지만 동일한 상황에서 Pod가 Pod4에 직접 연결하고 싶다면 서비스를 headless 서비스로 만들어야 한다. 만드는 방법은 ClusterIp 속성에 `none`이라고 넣는다. 이 서비스에 IP를 안 만들겠다는 내용이고 실제로 안만들어진다. 그리고 파드를 만들 때 신경 써 줘야 하는 것은 `hostname`이라는 속성에 도메인 이름을 넣어야 하고 `subdomain`에 이 서비스의 이름을 넣어야 한다.

### Service - Endpoint, ExternalName
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/92d9b527-22cb-4408-9e23-ee4d66516456)

서비스와 파드를 연결할 때 라벨을 통해 연결한다. 이것은 사용자 측면에서 이 둘의 연결을 하기 위한 도구이고 Kubernetes는 매칭이 됐을 때 `Endpoint`를 만들어서 실제 연결고리 관리를 한다. Kubernetes가 Endpoint를 만드는 방법은 서비스의 이름과 동일한 이름으로 엔드포인트의 이름을 설정하고 엔드포인트 안에는 파드의 접속 정보를 넣어준다. 그렇기 때문에 해당 규칙을 알면 라벨과 셀렉터를 안 만들어도 직접 연결을 할 수가 있다. 그런데 IP는 변경 가능성이 있기 때문에 도메인 이름을 사용하는 것이다. Github의 IP 주소도 변경이 될 수 있고 그래서 도메인 이름을 지정하는 방법도 필요한데 이럴 때 사용하는 것이 `External Name`이다. 서비스에 ExternalName이라는 속성을 달아서 이 안에 도메인 이름을 넣을 수가 있다. 이렇게 넣으면 DNS 캐시가 내부와 외부 DNS를 찾아서 IP를 알아낸다. 그래서 결국 Pod는 서비스를 가리키고만 잇으면 서비스에서 필요시마다 해당 도메인 주소를 변경할 수가 있어서 접속할 곳이 변경되더라도 Pod를 수정하고 재배포하는 일은 없어지게 된다.

## Volume - Dynamic Provisioning, StorageClass, Status, ReclaimPolicy
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/a40f57d5-8ced-4b4b-9744-2042639552d2)

먼저 볼륨은 데이터를 안정적으로 유지하기 위해서 사용을 하는 것이다. 그러기 위해서 실제 데이터는 Kubernetes 클러스터와 분리가 되서 관리가 된다. 그리고 이런 방식으로 관리할 수 있는 볼륨의 종류들이 많은데 크게 내부망에서 관리하는 경우와 외부망에서 관리하는 경우로 나눌 수 있고 외부망에서는 AWS, GCP와 같은 클라우드 스토리지를 두고 여기에 Kubernetes cluster와 연결을 해서 사용할 수 있다. 내부망에는 Kubernetes를 구성하는 노드들이 있는데 기본적으로 Kubernetes에서 이 노드들의 실제 물리적은 공간에 데이터를 만들 수 있는 hostPath나 local 볼륨이 있다. 그리고 별도의 On-Premise Solution들을 노드에 설치할 수 있다. 그래서 Kubernetes 클러스터 밖에 실제 볼륨이 마련되어 있다면 관리자는 PV를 만드는데 Storage와 AccessMode를 정하고 볼륨을 선택해서 연결한다. 그리고 사용자는 원하는 Storage와 AccessMode로 PVC를 만들면 Kubernetes가 알아서 적절한 PV와 연결을 해주고 PVC를 Pod에서 사용한다.

볼륨을 사용하면 볼륨이 필요할 때마다 PV를 만들어줘야 하고 원하는 PV랑 연결을 하고 스토리지랑 액세스를 맞춰야하고 복잡하다. 그래서 Kubernetes는 `Dynamic Provisioning`이라고 해서 사용자가 PVC를 만들면 알아서 PV를 만들어주고 실제로 볼륨과 연결해주는 기능이 존재한다. PV에는 Pod처럼 각각의 상태가 존재한다. 이 상태를 통해서 PV가 PVC에 연결되어 있는 상태인지 아니면 끊겼는지 또는 에러가 났는지 알 수 있고 PV를 삭제하는 부분에 있어서 정책적인 요소가 있다.

![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/1714a3cd-21e3-4438-9c99-9f558e4c98a4)

`Dynamic Provisioning`은 사전작업으로 `스토리지 솔루션`을 설치해야 한다. 설치를 하면 서비스나 파드 등 여러 오브젝트들이 생성이 되지만 이 중에서 중요한 것은 `StorageClass`라는 오브젝트이다. 이 스토리지 클래스를 사용해서 동적으로 PV를 만들 수 있는데 PVC를 만들 때 `StorageClassName`이라는 부분이 있다. 여기에 StorageClass 이름을 넣으면 자동으로 스토리지 OS 볼륨을 가진 PV가 만들어진다. 그리고 스토리지 클래스는 추가로 만들 수 있고 Default를 설정해서 만들어 놓으면 PVC에 스트리지 클래스 이름을 생략했을 때 Default 스토리지 클래스가 적용이 되서 PV가 만들어진다. 

`Status & ReclaimPolicy`에서 `Status`는 최초 PV가 만들어졌을 때 `Available` 상태고 PVC와 연결이 되면 `Bound` 상태로 변하게 된다. 이렇게 PV를 만드는 경우에 볼륨의 실제 데이터가 만들어진 상태는 아니고 Pod의 서비스가 유지되다가 파드가 삭제될 경우에는 PVC와 PV에는 아무런 변화가 없기 때문에 파드가 삭제되더라도 데이터에 문제가 없게 된다. PVC를 삭제해야지만 PV와 연결이 끊어지면서 PVE의 상태는 `Released` 상태로 변하게 된다. 이런 과정 중에 PVE와 실제 데이터 간의 연결에 문제가 생기면 `Failed` 상태로 변하기도 한다. PVC가 삭제 됐을 때 상황에 대해서 PV에 설정해 놓은 `ReclaimPolicy`에 따라서 PV에 대한 상태가 달라진다. 

## Accessing API - Overview
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/9da8bd9e-a72e-4e93-8c0e-89c50e9ba888)

마스터노드에 Kubernetes API 서버가 있는데 이 API 서버를 통해서만 Kubernetes의 자원을 만들거나 조회를 할 수 있다. Kubernetes를 설치할 때 `kubectl`도 설치해서 CLI로 자원을 조회할 수 있는 것도 다 이 API 서버에 접근을 해서 정보를 가져오는 것이다. 내부에서는 이렇게 접근할 수 있지만 내부에서 이 API 서버로 접근을 하려면 인증서를 가지고 있는 사람만 `HTTPS`로 접근할 수 있다. 근데 만약 내부 관리자가 kubectl 명령으로 프록시를 열어줬다면 인증서 없이 HTTP로 API 서버에 접근할 수 있다. 그리고 kubectl은 마스터 내부에만 설치할 수 있는 것은 아니고 외부 PC에서도 설치해서 사용할 수 있는데 config 기능을 활용하면 만약 Kubernetes Cluster가 여러 대가 있을 때 간편한 명령을 통해서 접근할 수 있는 클러스터의 연결 상태를 유지할 수 있다. 연결이 된 상태에서는 `kubectl get` 명령으로 해당 클러스터에 있는 Pod 정보들을 가져올 수 있다. 이러한 접근 방법들은 유저 입장에서 API 서버에 접근을 위한 방법이고 `User Account`라고 한다. 

Pod 입장에서 생각해보자. Pod 입장에서는 API 서버에 마음껏 접근할 수 있을까? 만약 그렇게 된다면 Pod를 만들기만 하면 누구나 Pod를 통해서 API 서버에 접근이 가능해지기 때문에 보안상 문제가 있게 된다. 그래서 Kubernetes에서는 `Service Account`라고 해서 Pod들이 API 서버로 접근하는 방법이 있다. 여기까지의 설명이 `Authentication` 이다.

만약 Namespace로 Pod가 분리되어 있는 상태에서 이 Namespace에 있는 Pod가 API 서버에 접근할 수 있다고 해서 A Namespace에 있는 Pod를 조회해도 될까? 이것은 권한 여부에 따라 가능하게 할 수도 있고 못하게 할 수가 있다. 이러한 부분이 `Authorization`이다. 그리고 권한까지 문제가 없다면 `Admission Control`이라고 해서 만약 PV를 만들 때 관리자가 용량을 1GB 이상으로 만들지 못하도록 설정해 놓았다면 Pod를 만들라는 API 요청이 들어왔을 때 Kubernetes는 설정된 크기를 넘지 못하도록 체크를 해야 한다. 

## Authentication - X509 Certs, kubectl, ServiceAccount
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/324f9607-8afc-40f6-9c74-55957390479c)

### X509 Certs
클러스터에 6443 포트로 API 서버가 열려 있다. 사용자가 여기로 HTTPS 접근을 하려면 Kubernetes 설치 시에 `kube-config`라고 해서 이 클러스터에 접근할 수 있는 정보들이 들어있는 파일이 있는데 이 파일 안에 인증서 내용이 있다. 그래서 여기 클라이언트 키와 인증서를 복사해서 가지고 오면 된다. 이 파일이 무엇이고 어떻게 만드는지에 대해 알아보자. 

최초에 발급 기간과 클라이언트에 대한 개인 키를 만들고 이 개인 키를 가지고 인증서를 만들기 위한 각각의 인증 요청서라는 `csr` 파일을 만든다. 그래서 `ca`의 경우 이 인증 요청서를 가지고 인증서를 만드는데 kube-config에 있는 `cacrt`가 이것이다. 클라이언트 인증서의 경우 발급기간 개인키와 인증서 그리고 클라이언트의 요청서를 합쳐서 만든다. 이렇게 만들어진 클라이언트 인증서가 kube-config 파일에 `Client crt` 파일로 들어있고 kube-config 파일에는 `Client key`도 있다. 

Kubernetes 설치할 때 kubectl도 설치하고 설정 내용 중 kube-config 파일을 kubectl에서 사용하도록 복사하면 kubectl로 API에 인증이 돼서 리소스들을 조회할 수 있다. accept-hosts 옵션을 통해서 8001번 포트로 프록시를 열어두면 외부에서도 HTTP로 접근을 할 수 있게 되는데 그러면 kubectl이 인증서를 가지고 있기 때문에 사용자는 아무런 인증서 없이 접근할 수 있게 된다. 

### kubectl
그리고 외부 서버에 kubectl을 설치해서 멀티 클러스터에 접근을 하는 내용이다. 사전에 각 클러스터에 있는 kube-config 파일이 내 kubectl에도 있어야 된다. 이렇게 돼 있다면 사용자는 원하는 클러스터에 접근을 해서 자원을 조회하고 만들 수가 있다. kube-config 안에는 `clusters`라는 항목으로 클러스터를 등록할 수가 있고 내용으로는 name과 url 그리고 CA 인증서가 있다. 그리고 `users`라는 항목으로 사용자를 등록할 수가 있는데 내용으로는 유저 이름과 이 유저에 대한 개인 키와 인증서가 있다. 그래서 이렇게 셋팅이 되어 있다면 contexts 항목을 통해서 이 둘을 연결할 수가 있다. 내용으론 컨텍스트 이름과 연결할 클러스터, 유저 이름이 있다. 

이렇게 kube-config가 만들어졌을 때 사용자가 클러스터 A에 연결하고 싶으면 `kubectl config` 명령으로 현재 사용하고 싶은 컨텍스트를 지정할 수가 있고 지정을 했을 때 `kubectl get node` 명령을 알리게 되면 클러스터 A에 대한 노드 정보들이 조회가 된다.

### ServiceAccount
Kubernetes 클러스터와 API 서버가 있고 Namespace를 만들게 되면 기본적으로 default라는 이름의 `ServiceAccount`가 자동으로 만들어진다. 그리고 이 ServiceAccount에는 Secret 하나 달려있는데 내용으로는 `CA crt` 정보와 토큰값이 들어있다. 그리고 Pod를 만들면 이 ServiceAccount가 연결이 되고 Pod는 토큰값을 통해서 API 서버에 연결을 할 수 있는데 결국 토큰값만 알면 사용자도 이 값을 가지고 API 서버에 접근할 수가 있다.

## Authorization
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/3a0e5ad3-cc57-4a54-8fbd-909d59186550)

## Dashboard
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/4657231a-a833-49c3-bd05-2e45f9afc68c)

## Controller
### StatefulSet
![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/41bdbf31-f73b-41b4-924b-a18cbac38fd1)

어플리케이션 종류에는 Stateless Application과 Stateful Application이 있다. Stateless는 대표적으로 웹서버가 있다. 그리고 Stateful은 대표적으로 데이터베이스가 있다. Stateless는 앱이 배포되더라도 다 똑같은 서비스의 역할을 한다. 반면 Stateful은 각각의 앱마다 자신의 역할이 있다. MongoDB의 경우 하나는 Primary 역할, 또 하나는 Secondary 그리고 Arbiter의 역할이 있는데, 간단히 설명하면 Primary가 main이고 Primary가 죽으면 Arbiter가 감지해서 Secondary가 Primary의 역할을 할 수 있도록 변경해준다. Stateless 앱은 단순 복제인 것과 달리 Stateful 앱은 각 앱마다 자신의 고유 역할을 가지고 있다. 그래서 만약 앱이 하나 죽으면 Stateless 앱은 단순히 같은 서비스의 역할을 하는 앱을 복제해주면 되고 Stateful 앱은 Arbiter의 역할을 하는 앱이 죽으면 반드시 Arbiter의 역할을 하는 앱이 만들어져야한다. 이름도 이름 자체가 고유 아이덴티티의 식별 요소이기 때문에 변경이 되면 안된다. 그리고 또 다른 특징은 Stateless 앱은 볼륨이 반드시 필요하진 않다. 반면 Stateful 앱은 각각의 역할이 다른 만큼 볼륨도 각각 써야 한다. 다음은 앱들에 연결이 되는 대상과 네트워킹 흐름에 대한 차이도 있는데 Stateless 앱은 대체로 사용자들이 접속을 하고 접속된 네트워크 트래픽은 한 앱에만 집중이 되면 서버에 문제가 생기기 때문에 그렇지 않도록 여러 앱에 분산이 되는 형태로 흘러가게 된다. 그렇지만 Stateful 앱은 대체로 내부 시스템들이 데이터베이스 저장을 위해서 연결을 하고 이 때, 트래픽은 각 앱의 특징에 맞게 들어와야 한다. Primary 앱은 Read/Write 권한이 있기 때문에 연결을 하려는 시스템에서 CRUD 모두 하려면 여기에 접속을 해야 하고 Secondary는 Read 권한만 있기 때문에 조회만 해도 되는 내부 시스템은 트래픽 분산을 위해서 App2 에 접속을 할 수가 있다. 

이렇게 Stateful 앱은 역할에 따라 의도가 있는 연결이고 Stateless 앱은 단순 분산의 목적만 가지고 있는 연결인데 Kubernetes에서 이런 두 앱의 특징에 따라 활용할 수 있도록 나눠진 것이 Stateless 앱은 Replica Set 컨트롤러이다. 그리고 Stateful 앱은 Stateful Set 컨트롤러이다.

![image](https://github.com/haeyonghahn/k8s-intermediate/assets/31242766/1b2864f2-3808-4eac-bae5-6e9c6d2a1024)

해당 컨트롤러의 특징을 ReplicaSet Controller와 비교하면서 보자. 먼저 replicas1로 컨트롤러를 만들게 되면 StatefulSet도 ReplicaSet과 마찬가지로 해당 개수만큼 파드가 관리된다. 다른 점은 Pod가 생성될 때 ReplicaSet은 뒷부분이 랜덤으로 생성되지만 StatefulSet은 0부터 순차적으로 숫자가 붙어서 생성이 된다. 

다음으로 StatefulSet PVC와 Headless Service를 보자. 마찬가지로 ReplicaSet과 비교해보면 ReplicaSet은 파드의 볼륨을 연결하려면 PVC를 별도로 직접 생성을 해놔야 한다. 그리고 ReplicaSet을 만들면 Template에 Pod의 내용의 PVC1을 지정을 해놓기 때문에 바로 연결이 된다. 근데 StatefulSet의 경우 Template을 통해 Pod가 만들어지고 추가적으로 volumeClaimTemplates가 있는데 이것을 통해서 PVC가 동적으로 생성이 되고 Pod와도 바로 연결이 된다. 그리고 replicas를 3으로 변경하면 ReplicaSet은 모든 Pod들이 똑같은 PVC 이름으로 PVC1을 가지고 있기 때문에 PVC1로 모두 연결이 되고 StatefulSet은 volumeClaimTemplates 때문에 Pod가 추가될 때마다 새로운 PVC가 생성이 되서 연결이 된다. 그래서 Pod마다 각자의 역할에 따른 데이터를 저장할 수가 있다. 그리고 Pod2가 삭제가 되면 다시 Pod2가 그대로 만들어지면서 기존에 Pod2가 연결이 돼 있던 PVC에 붙게 된다. 

주의할 점은 ReplicaSet은 PVC가 Node1에 만들어졌다면 그 PVC에 연결하는 Pod도 Node1 위에 있어야 한다. 만약 Node2의 Pod가 생성되면 PVC랑 연결이 안되서 Pod가 생성이 안된다. 그래서 템플릿 안에 노드 셀렉터를 Node1로 해줘야 이러한 문제를 막을 수 있지만 StatefulSet은 동적으로 Pod와 PVC가 같은 노드에 만들어지기 떼문에 모든 노드에 균등하게 배포가 된다. 

그리고 replicas를 0으로 줄이게 되면 StatefulSet은 파드를 순차적으로 삭제하지만 PVC는 삭제하지 않는다. 볼륨은 함부로 지우면 안되기 때문에 사용자가 직접 삭제를 해야 한다. 마지막으로 StatefulSet을 만들 때 ServiceName이라는 속성으로 서비스 이름을 넣을 수 있는데 이 이름과 매칭되는 Headless 서비스를 만들게 되면 파드의 예측 가능한 도메인 이름이 만들어지기 때문에 internal Server의 특정 파드 입장에서 원하는 StatefulSet에 연결할 수가 있다. 
