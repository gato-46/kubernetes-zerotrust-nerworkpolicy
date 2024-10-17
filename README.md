### 🌟 **Kubernetes의 Zero Trust 보안 모델: 네트워크 폴리시 학습하기**
<br>

### 🎯 **학습 목표**

- Kubernetes의 네트워크 폴리시 개념을 이해하고, 이를 통해 **Zero Trust** 보안 모델을 구현하는 방법을 배운다.
- Minikube 환경에서 실제로 네트워크 폴리시를 설정하고 적용해보며, 정책의 효과를 실험적으로 확인한다.
- 특정 Pod 간의 통신을 허용하는 세부 정책을 설정하여, 보안 설정을 더욱 강화하는 방법을 익힌다.

<br>

### 1. 🛠️ **Minikube 설치 및 설정**

먼저, Minikube를 설치하고 시작합니다. 네트워크 플러그인으로 **Calico**를 사용하여 네트워크 폴리시를 구현할 수 있도록 설정한다.

```bash
minikube start --network-plugin=cni --cni=calico
```
<br>

### 2. 🖥️ **nginx 서비스 및 Pod 생성**

**nginx 서비스**와 **ReplicaSet**을 설정하여 여러 개의 `nginx` Pod를 생성한다.



### 2-1. 🔧 **nginx 서비스 YAML 작성**

`nginx-service.yaml` 파일을 다음과 같이 작성하고, **ClusterIP**로 서비스를 생성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

서비스를 생성한다.

```bash
kubectl apply -f nginx-service.yaml
```



### 2-2. 📂 **nginx ReplicaSet YAML 작성**

`nginx-replicaset.yaml` 파일을 작성하여 3개의 `nginx` Pod를 생성한다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

ReplicaSet을 적용한다.

```bash
kubectl apply -f nginx-replicaset.yaml
```

---
<br>

### 3. 🔒 **네트워크 폴리시 설정**

네트워크 폴리시를 적용하여 **nginx Pod로의 모든 트래픽을 차단**한 후, 이를 테스트한다.

### 3-1. ❌ **네트워크 폴리시 (모든 트래픽 차단)**

모든 트래픽을 차단하는 `deny-all` 네트워크 폴리시를 적용한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

폴리시를 적용한다.

```bash
kubectl apply -f deny-all.yaml
```

### 3-2. 🚫 **deny-all 네트워크 폴리시 테스트 (ClusterIP로 연결)**

네트워크 폴리시가 제대로 적용되었는지 확인하기 위해 **nginx 서비스의 ClusterIP**로 연결을 시도하여 테스트한다.

### 3-2-1. 🛠️ **nginx 서비스 ClusterIP 확인**
`busybox Pod`를 사용하여 `nginx` 서비스의 ClusterIP로 직접 연결을 시도한다.
```bash
kubectl get services
```

### 3-2-2. 🕵️ **ClusterIP로 연결 시도**

```bash
kubectl run busybox --image=busybox --restart=Never --command -- sleep 3600

kubectl exec -it busybox -- wget --spider --timeout=1 <nginx-service-ClusterIP>
```
- **타임아웃 발생** 시: 네트워크 폴리시에 의해 트래픽이 차단되었음을 의미한다.
![네크워크 폴리시](https://github.com/user-attachments/assets/82c0ac22-28ba-421c-af0a-f0661b99b948)

---
<br>

### 4. 🌐 **특정 통신 허용 네트워크 폴리시 설정**

이제 `nginx` Pod 간의 **특정 통신만 허용**하도록 네트워크 폴리시를 설정하고 테스트한다.

### 4-1. 🎛️ **특정 통신 허용 폴리시 YAML 작성**

다음은 `nginx` 레이블을 가진 Pod들 간의 **Ingress 트래픽**만 허용하는 네트워크 폴리시다.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: nginx
    ports:
    - protocol: TCP
      port: 80
```
  podSelector: nginx라는 레이블을 가진 Pod에만 이 정책이 적용된다.
  Ingress: nginx Pod 간의 80번 포트에서 들어오는 트래픽을 허용한다.

### 4-2. ✅ **허용 폴리시 적용**

```bash
kubectl apply -f allow-nginx.yaml
```

### 4-3. 🔄 **허용된 트래픽 테스트 (ClusterIP로 연결)**
  네트워크 폴리시가 제대로 적용되었는지 ClusterIP를 통해 다시 한 번 테스트한다.
### 4-3-1. 🕵️ **ClusterIP로 연결 시도**
  `busybox` Pod에서 `nginx` 서비스의 ClusterIP로 연결을 시도한다.
```bash
kubectl exec -it busybox -- wget --spider --timeout=1 <nginx-service-ClusterIP>
```
- **연결 성공** 시: `nginx` Pod 간의 트래픽이 허용되었음을 의미한다.
- **타임아웃 발생** 시: 폴리시 설정이 잘못되었거나, 다른 문제가 발생했을 수 있다.

---
<br>

### 📝 **결론**

Kubernetes의 네트워크 폴리시를 활용한 Zero Trust 보안 모델은 애플리케이션의 보안을 강화하는 데 필수적이다. 이 시스템을 통해 기업은 각 Pod 간의 통신을 세밀하게 제어하고, 불필요한 접근을 차단함으로써 보안성을 높일 수 있다.

또한, Minikube 환경에서의 네트워크 폴리시 실습은 실제 Kubernetes 클러스터 운영 시 적용 가능한 귀중한 경험을 제공한다. 이를 통해 개발팀은 안전하고 안정적인 애플리케이션 개발 환경을 구축할 수 있으며, 기업은 네트워크 보안을 효율적으로 관리하고, 위험 요소를 최소화하는 데 기여할 수 있다.

이와 같은 접근 방식을 통해 기업은 더 나은 보안 체계를 갖추고, 클라우드 환경에서의 안전한 운영을 지속적으로 유지할 수 있다.
