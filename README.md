# State란?

State는 애플리케이션에 의해 생성되고 사용되는 데이터로, 손실되어서는 안된다

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/621504a8-73f4-4a14-962c-b3160a44e406/cb8afb21-d170-4484-8b27-39ab9756fdd7/image.png)

사용자 생성 데이터는 데이터베이스나 파일에 저장된다.

앱에 의해 도출된 중간 결과는 메모리, 임시 DB, 파일에 저장된다.

데이터가 컨테이너 재시작 후에도 살아남아야한다.

따라서 볼륨을 사용한다.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/621504a8-73f4-4a14-962c-b3160a44e406/0c25d4b9-df37-43da-81d1-5c6fe7421b70/image.png)

deployment의 pod의 template에 볼륨을 지정할 수 있다.

다양한 볼륨 유형과 드라이버 지원한다.

쿠버네티스를 사용하면 여러 노드에서 애플리케이션을 실행할 수 있고 다른 클라우드 및 호스팅 프로바이더에서도 실행 가능하다.  따라서 데이터가 실제로 저장되는 위치와 관련해 매우 유연하다.

로컬 볼륨지원. pod가 실행되는 곳은 워커 노드의 폴더이다. 

클라우드 프로바이더 특정 볼륨지원. 

볼륨 수명이 pod 수명에 따라 다르다. [볼륨이 쿠버네티스에 의해 시작되고 관리되는 pod의 일부이기 때문이다.](https://www.notion.so/10a97d6ce88f8032beddc218539bff2d?pvs=21) 

따라서 컨테이너가 제거돼도 살아남는다. pod가 제거되면 볼륨 제거된다.

데이터가 저장되는 위치 완벽 제어

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/621504a8-73f4-4a14-962c-b3160a44e406/64e78cb8-d450-46ab-b8b4-037509b0440e/image.png)

# 볼륨 유형: emptyDir

다양한 볼륨 유형과 드라이버 지원한다.

https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#volume-v1-core

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec: 
  replicas: 1
  selector:
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: seokhyeonhong/kub-data-demo:1
          volumeMounts: # 컨테이너 내부에 마운트 될 볼륨 정의
            - mountPath: /app/story
              name: story-volume
      volumes: # 사용가능한 볼륨 정의
        - name: story-volume
          emptyDir: {}
```

pod가 시작될 때마다 단순히 새로운 빈 디렉토리 생성

**pod가 살아있는한** 이 디렉토리를 활성 상태로 유지하고 데이터를 채운다. 

그래서 컨테이너가 재시작, 제거되어도 데이터는 유지된다.

# 볼륨 유형: hostPath

replica를 2개로 하게되면 `emptyDir` 볼륨은 각 Pod에 대해 독립적으로 생성됩니다. 즉, 레플리카가 2개 이상일 경우 각 Pod는 자신의 `emptyDir` 볼륨을 가지며, 이 볼륨 간에 데이터가 공유되지 않습니다. 따라서 한 Pod에서 생성한 데이터는 다른 Pod에서 접근할 수 없습니다.

hostPath는 여러 Pod가 특정 경로 대신, 호스트 머신의 동일한 경로에서 하나를 공유할 수 있다. 

바인드 마운트와 비슷하다. hostPath의 경로(호스트 머신)의 데이터를 mountPath(컨테이너 내부)의 경로와 공유한다. 
**호스트의 특정 경로를 컨테이너에 직접 연결하여, 호스트와 컨테이너 간에 데이터를 공유**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec: 
  replicas: 2
  selector:
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: seokhyeonhong/kub-data-demo:1
          volumeMounts:
            - mountPath: /app/story
              name: story-volume
      volumes:
        - name: story-volume
          hostPath:
            path: /data # 데이터 저장되는 호스트 머신 경로
            type: DirectoryOrCreate # 동작방식
```

이제 오류 요청이 들어가 하나의 컨테이너가 충돌하더라도 그 데이터를 가져올 수 있다. 
**더 이상 [Pod](https://www.notion.so/10a97d6ce88f8032beddc218539bff2d?pvs=21)에 국한되지 않는다.**

현재는 워커 노드가 1개이다. 프로덕션은 여러 노드를 사용할 것이다.

그러나 서로 다른 노드에서 실행되는 경우에는 **동일한 데이터에 액세스가 불가**하다.

동일 노드의 Pod만 hostPath 데이터에 액세스가 되기 때문이다.
`hostPath`는 호스트 머신의 특정 경로를 참조하기 때문에, 그 경로에 접근할 수 있는 Pod는 반드시 같은 노드에서 실행되어야 합니다.

데이터를 다른 Pod와 독립적으로 만들고 싶다면 hostPath가 유용하다. 

# 볼륨 유형: CSI

EFS를 쿠버네티스 볼륨의 스토리지 솔루션으로 추가하는것이 매우 쉬워진다.

**EFS와 CSI의 통합**

- **CSI 드라이버: CSI는 다양한 스토리지 시스템을 쿠버네티스와 통합하기 위한 표준 인터페이스입니다. Amazon EFS에 대한 CSI 드라이버를 사용하면 EFS 파일 시스템을 쿠버네티스의 Persistent Volume(PV) 및 Persistent Volume Claim(PVC)으로 쉽게 사용할 수 있습니다.**
- **설정 용이성: EFS CSI 드라이버를 사용하면 EFS 파일 시스템을 쿠버네티스 리소스에 쉽게 연결할 수 있습니다. YAML 파일을 통해 PV와 PVC를 정의하고, 이를 Pod에 마운트하는 과정이 간단해집니다.**
- **동시 접근: EFS는 여러 Pod에서 동시에 접근할 수 있는 파일 시스템입니다. 이는 여러 애플리케이션이 동일한 데이터를 공유해야 할 때 유용합니다.**
- **자동화: EFS CSI 드라이버는 EFS 파일 시스템의 생성 및 삭제를 자동화할 수 있는 기능을 제공하여, 관리의 복잡성을 줄여줍니다.**

**기본 사용 예시**

- **EFS 파일 시스템 생성: AWS 콘솔이나 CLI를 통해 EFS 파일 시스템을 생성합니다.**
- **EFS CSI 드라이버 설치: 쿠버네티스 클러스터에 EFS CSI 드라이버를 설치합니다.**
- **Persistent Volume 및 Persistent Volume Claim 정의: EFS 파일 시스템을 참조하는 PV와 PVC를 정의합니다.**
- **Pod에서 사용: PVC를 Pod에 마운트하여 애플리케이션에서 EFS에 저장된 데이터를 사용할 수 있습니다.**

# 볼륨에서 **Persistent Volume 으로**

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/621504a8-73f4-4a14-962c-b3160a44e406/23660cab-0bbf-4146-bde3-e37c1c65f1b6/image.png)

hostPath는 현재 단일 노드 환경이기 때문에 해결책이 된것일뿐이다.

때때로 Pod와 노드에 독립적인 볼륨이 필요하다.

db컨테이너가 있거나 ,Pod 교체 후에도 유지되는 파일을 작성하는 컨테이너가 있는 경우

PV는 Pod와 Node에 독립적이다.

영구 볼륨은 Pod에서 분리되어있다.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/621504a8-73f4-4a14-962c-b3160a44e406/0ec31278-d70f-406e-9ec4-bd51d4eeefce/image.png)

PV Claim 으로 PV에 액세스 요청 할 수 있다. 

AWS EBS(Block), AzureFile(File) 같은 것들은 클라우드 스토리지에 저장한다.

CSI유형을 사용하여 **모든 종류의 스토리지**를 클러스터에 연결할 수 있다. 이것 또한 스토리지가 클러스터 노드에 있지 않고 특정 클라우드 스토리지 서비스에 있다. 

# 영구 볼륨 정의하기

[hostPath는 노드에 독립적인 볼륨이 아니다.](https://www.notion.so/k8s-10c97d6ce88f80e5a90bfdbf7d194a28?pvs=21) 단지 minikube가 단일 노드 이기 때문에 쓸수는 있는것.

host-pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-pv
spec: 
  capacity: 
    storage: 1Gi
  volumeMode: FileSystem # FileSystem or Block
  accessModes:  # 액세스 방법
    - ReadWriteOnce # 이 볼륨이 단일 노드(여러 Pod, 동일한 노드)에 의해 읽기/쓰기 볼륨으로 마운트
    # - ReadOnlyMany # 읽기전용 여러노드 가능
    # - ReadWriteMany # 읽기/쓰기 여러노드
  hostPath:
    path: /data
    type: DirectoryOrCreate
```

# 영구 볼륨 클레임(PVC) 생성하기

클레임은 이 볼륨을 사용하려는 Pod 에서 만들어야한다.

1. 클레임 구성
2. 그것을 사용하고자 하는 모든 Pod에서 사용

**영구 볼륨(Persistent Volume)과 영구 볼륨 클레임(Persistent Volume Claim)**

- **영구 볼륨(PV): 클러스터 내에서 사용할 수 있는 스토리지 리소스를 나타냅니다. PV는 클러스터 관리자가 미리 프로비저닝한 스토리지입니다. 이 스토리지는 클러스터의 수명과 관계없이 지속적으로 존재합니다.**
- **영구 볼륨 클레임(PVC): 사용자가 필요한 스토리지의 요구 사항을 정의하는 요청입니다. PVC는 특정 크기와 접근 모드를 요구하며, 쿠버네티스는 이러한 요구 사항을 충족하는 PV를 찾아 바인딩합니다.**

**동적 볼륨 프로비저닝(Dynamic Volume Provisioning)**

- **동적 프로비저닝: 사용자가 PVC를 생성할 때, 쿠버네티스가 자동으로 PV를 생성하는 방식입니다. 이 경우, 사용자는 PV의 세부 사항을 알 필요 없이 PVC만 정의하면 됩니다. 쿠버네티스는 스토리지 클래스를 기반으로 적절한 PV를 생성합니다.**

**정적 프로비저닝(Static Provisioning)**

- **정적 프로비저닝: 클러스터 관리자가 미리 PV를 생성하고, 사용자는 PVC를 통해 이러한 PV를 요청하는 방식입니다. 이 경우, PV는 수동으로 생성되며, 사용자는 PV의 이름이나 속성을 통해 특정 PV를 클레임할 수 있습니다.**

**요약**

- **유연성: 동적 프로비저닝은 사용자가 스토리지 리소스를 쉽게 요청할 수 있도록 하여 유연성을 제공합니다. 반면, 정적 프로비저닝은 관리자가 스토리지를 미리 설정해야 하므로 유연성이 떨어질 수 있습니다.**
- **클레임 방법: PVC는 PV의 이름으로 클레임할 수 있지만, 동적 프로비저닝에서는 스토리지 클래스와 같은 다른 속성을 기반으로 클레임할 수 있습니다.**

우린 정적 프로비저닝을 한다.

host-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: host-pvc
spec: #클레임에 대한 사양, 구성 설정
  volumeName: host-pv
  accessMode:
    - ReadWriteOnce # 볼륨 클레임을 위해 여기서 사용할 모드 정의
  resources: 
    requests: 
      storage: 1Gi # 동일 볼륨에 대한 클레임이 여러개면 더 적게 요청 가능
      
```

영구 볼륨에 대한 클레임 이다. 이것이 Pod에 대한 연결을 설정하지는 않는다.

Pod를 이 클레임에 연결하려면 deployment.yml의 볼륨키도 바꿔줘야함

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec: 
  replicas: 2
  selector:
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: seokhyeonhong/kub-data-demo:1
          volumeMounts:
            - mountPath: /app/story
              name: story-volume
      volumes:
        - name: story-volume
          persistentVolumeClaim:
            claimName: host-pvc
```

# Pod에서 클레임 사용하기

스토리지 클래스 확인

```yaml
kubectl get sc
```

스토리지 클래스는 쿠버네티스에서 관리자에게 스토리지 관리 방법과 볼륨 구성 방법을 세부적으로 제어할 수 있게 해주는 또다른 개념. 

우리가 설정한 영구 볼륨 구성에 중요한 정보를 제공한다.

```yaml
  storageClassName: standard
```

pv와 pvc .yml 에 추가

pv적용 확인

```yaml
$ kubectl get pv
```

```yaml
$ kubectl get pvc
```

# **Normal Volumes vs Persistent Volumes**

**Normal Volumes**

- **볼륨은 Pod에 연결되어 있으며, Pod의 생명 주기와 관련이 있습니다.**
- **Pod와 함께 정의되고 생성됩니다.**
- **전 세계적으로 관리하기 어렵고 반복적인 설정이 필요합니다.**

**Persistent Volumes**

- **볼륨은 독립적인 클러스터 리소스입니다 (Pod에 연결되지 않음).**
- **독립적으로 생성되며, PVC를 통해 요청됩니다.**
- **한 번 정의하면 여러 번 사용할 수 있습니다.**

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/621504a8-73f4-4a14-962c-b3160a44e406/8e6f38e7-dbde-4653-ac41-d1bc7344c509/image.png)

# 환경변수 사용하기

```jsx
const filePath = path.join(__dirname, process.env.STORY_FOLDER, 'text.txt');
```

process.env.STORY_FOLDER 환경변수 deployment.yaml 에서 지정

```jsx
spec: 
  replicas: 2
  selector:
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: academind/kub-data-demo:2
          env:
            - name: STORY_FOLDER # 이름
              value: 'story' # 값
          volumeMounts:
            - mountPath: /app/story
              name: story-volume
      volumes:
        - name: story-volume
          persistentVolumeClaim:
            claimName: host-pvc
```

# ConfigMaps

컨테이너 spec이 아닌 클러스터 엔티티에 대한 별도의 파일이나 별도의 리소스에 키-값 쌍을 갖고자 한다. 다른 Pod의 각각 컨테이너가 동일한 환경 변수를 사용하도록.

environment.yaml 작성

```jsx
apiVersion: v1
kind: ConfigMap
metadata:
  name: data-store-env
data:
  folder: 'story' 
  # key: value..
```

deployment.yml 수정

```jsx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: story-deployment
spec: 
  replicas: 2
  selector:
    matchLabels:
      app: story
  template:
    metadata:
      labels:
        app: story
    spec:
      containers:
        - name: story
          image: academind/kub-data-demo:2
          env:
            - name: STORY_FOLDER # 코드에 노출되는 환경변수 이름
              # value: 'story'
              valueFrom: 
                configMapKeyRef: # 특정 맵 가리킴
                  name: data-store-env
                  key: folder # 위 환경변수에 대한 값
          volumeMounts:
            - mountPath: /app/story
              name: story-volume
      volumes:
        - name: story-volume
          # emptyDir: {}
          persistentVolumeClaim:
            claimName: host-pvc
```
![image](https://github.com/user-attachments/assets/d542fed5-2667-4cff-9261-ac83176fb9d7)
![1](https://github.com/user-attachments/assets/2d4cf390-90f1-4f9c-b06a-3377ac2f1f84)
![2](https://github.com/user-attachments/assets/71f05440-b4b7-4f84-b071-e360d81830ca)

