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
노드 위에 Pod가 3개 만들어져서 균등하게 자원을 모두 사용하고 있는 상태라고 가정해보자. 근데 이 상태에서 Pod1이 추가적으로 자원을 더 사용해야 되는 상황이 발생했다. 근데 노드에는 추가적으로 할당받을 수 있는 자원이 없기 때문에 Pod1이 리소스 부족으로 에러가 나고 다운이 되어야 할까? 아니면 Pod2나 Pod3 중에 하나를 다운시키고 Pod1에 필요한 만큼의 자원을 적용해줘야할까? Kubernetes에서는 앱의 중요도에 따라 이런 걸 관리할 수 있도록 세 가지 단계로 `Quality of Service`를 지원해주고 있다. 
