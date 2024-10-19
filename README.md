# 🌟 Kubernetes의 Zero Trust 보안 모델: Service Mesh, 네트워크 폴리시, 그리고 RBAC 학습하기

## 🎯 학습 목표

- Kubernetes에서 **Service Mesh**, **네트워크 폴리시**, 그리고 **RBAC**를 결합해 **Zero Trust** 보안 모델을 구현하는 방법을 학습한다.
- **Minikube** 환경에서 **Istio**를 설치해 **서비스 간 통신 보호**를 구성하고, **네트워크 폴리시**를 통해 Pod 간 통신을 제어한다.
- 최종적으로 **RBAC**를 사용해 사용자와 애플리케이션의 접근 권한을 제어하는 방법을 실습한다.

<br>

    
## 💡 왜 3개의 기술을 결합하여 Zero Trust를 구축하는가?

**Zero Trust** 보안 모델은 시스템 내의 모든 부분이 잠재적으로 위험하다는 가정 하에 설계된다. Kubernetes에서 **Service Mesh**, **네트워크 폴리시**, 그리고 **RBAC**를 결합해 다층적인 보안 체계를 구축한다.

1. **Service Mesh**는 서비스 간 통신을 보호한다. **MTLS**(Mutual TLS)를 통해 서비스 간 트래픽을 **암호화**하고, **인증된 서비스끼리만 통신**할 수 있도록 한다.
2. **네트워크 폴리시**는 Pod 간의 **네트워크 트래픽을 세밀하게 제어**한다. 허용된 트래픽만 전달되도록 설정해 내부 네트워크 침입을 방지한다.
3. **RBAC**는 Kubernetes 리소스에 대한 **사용자와 애플리케이션의 접근 권한을 관리**한다. 불필요한 권한을 최소화함으로써 권한이 없는 사용자나 애플리케이션이 시스템에 접근하지 못하도록 한다.

이 세 가지 기술을 결합하면 시스템 내 모든 통신 경로와 자원을 보호하는 **완전한 Zero Trust 보안 모델**을 제공할 수 있다.

<br>

## 1. 🛠️ Minikube 설치 및 설정

Minikube를 설치하고 시작한다. 네트워크 플러그인으로 **Calico**를 사용하여 네트워크 폴리시를 구현할 수 있도록 설정한다.

```bash
minikube start --driver=docker --network-plugin=cni --cni=calico
```

<br>

## 2. 🖥️ nginx 서비스 및 Pod 생성

**nginx 서비스**와 **ReplicaSet**을 설정하여 여러 개의 `nginx` Pod를 생성한다.

### Step 2-1: nginx 서비스 YAML 파일 생성 (`nginx-service.yaml`)

다음 YAML 파일을 작성하여 **nginx 서비스**를 정의한다.

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

### Step 2-2: nginx ReplicaSet YAML 파일 생성 (`nginx-replicaset.yaml`)

다음 YAML 파일을 작성하여 3개의 `nginx` Pod를 생성한다.

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

### Step 2-3: 서비스 및 ReplicaSet 배포

작성한 YAML 파일을 사용하여 **nginx 서비스**와 **ReplicaSet**을 배포한다.

```bash
kubectl apply -f nginx-service.yaml
kubectl apply -f nginx-replicaset.yaml
```

<br>

## 3. 🔒 네트워크 폴리시 설정

네트워크 폴리시를 적용하여 **nginx Pod로의 모든 트래픽을 차단**한 후 이를 테스트한다.

### 3-1. ❌ 네트워크 폴리시 (모든 트래픽 차단)

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

### 3-2. 🚫 deny-all 네트워크 폴리시 테스트 (ClusterIP로 연결)

네트워크 폴리시가 제대로 적용되었는지 확인하기 위해 **nginx 서비스의 ClusterIP**로 연결을 시도하여 테스트한다.

### 3-2-1. 🛠️ nginx 서비스 ClusterIP 확인

```bash
kubectl get services
```

### 3-2-2. 🕵️ ClusterIP로 연결 시도

`busybox` Pod를 사용하여 `nginx` 서비스의 ClusterIP로 직접 연결을 시도한다.

```bash
kubectl run busybox --image=busybox --restart=Never --command -- sleep 3600

kubectl exec -it busybox -- wget --spider --timeout=1 <nginx-service-ClusterIP>
```

![네크워크 폴리시](https://github.com/user-attachments/assets/d890b069-9eba-476e-bdc1-9d8bc6ae7d8a)

- **타임아웃 발생** 시: 네트워크 폴리시에 의해 트래픽이 차단되었음을 의미한다.

<br>

## 4. 🌐 특정 통신 허용 네트워크 폴리시 설정

이제 `nginx` Pod 간의 **특정 통신만 허용**하도록 네트워크 폴리시를 설정하고 테스트한다.

### 4-1. 🎛️ 특정 통신 허용 폴리시 YAML 작성 (`allow-nginx.yaml`)

다음은 `nginx` 레이블을 가진 Pod들 간의 **Ingress 트래픽**만 허용하는 네트워크 폴리시이다.

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

### 4-2. ✅ 허용 폴리시 적용

```bash
kubectl apply -f allow-nginx.yaml
```

### 4-3. 🔄 허용된 트래픽 테스트 (ClusterIP로 연결)

네트워크 폴리시가 제대로 적용되었는지 **ClusterIP**를 통해 다시 한 번 테스트한다.

### 4-3-1. 🕵️ ClusterIP로 연결 시도

`busybox` Pod에서 `nginx` 서비스의 **ClusterIP**로 연결을 시도한다.

```bash
kubectl exec -it busybox -- wget --spider --timeout=1 <nginx-service-ClusterIP>
```

![네트워크 폴리시 성공](https://github.com/user-attachments/assets/c1059ffa-7137-4e59-b737-63823df38917)

- **연결 성공** 시: `nginx` Pod 간의 트래픽이 허용되었음을 의미한다.
- **타임아웃 발생** 시: 폴리시 설정이 잘못되었거나, 다른 문제가 발생했을 수 있다.

<br>

## 5. 👤 RBAC 설정

- *Role-Based Access Control(RBAC)**을 사용하여 **사용자 및 애플리케이션의 접근 권한**을 제한한다.

### 5-1. RBAC Role YAML 파일 생성 (`developer-role.yaml`)

다음 YAML 파일을 작성하여 **Pod**와 **Service**에 대한 권한을 부여하는 Role을 정의한다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "create", "delete"]
```

### 5-2. RBAC RoleBinding YAML 파일 생성 (`developer-binding.yaml`)

다음 YAML 파일을 작성하여 **developer 사용자**에게 위에서 정의한 **developer-role** 권한을 부여한다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: default
subjects:
- kind: User
  name: developer
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### 5-3. RBAC Role 및 RoleBinding 적용

작성한 YAML 파일을 사용해 **RBAC Role**과 **RoleBinding**을 적용한다.

```bash
kubectl apply -f developer-role.yaml
kubectl apply -f developer-binding.yaml
```

### 5-4. ✅ RBAC 테스트

RBAC가 제대로 설정되었는지 테스트한다.

```bash
# developer 사용자가 Pod 생성 가능 여부 확인
kubectl auth can-i create pods --as=developer

# developer 사용자가 네임스페이스 생성 시도 (권한 거부)
kubectl auth can-i create namespaces --as=developer
```

<br>

## 🔑 결론

이번 실습을 통해 **Service Mesh**, **네트워크 폴리시**, 그리고 **RBAC**를 결합하여 Kubernetes에서 **Zero Trust 보안 모델**을 구축하는 방법을 학습하였다.

- **Service Mesh**는 서비스 간의 통신을 보호하고, 상호 인증된 서비스 간에만 트래픽을 허용한다.
- **네트워크 폴리시**를 통해 Pod 간의 불필요한 통신을 차단하고, 허용된 트래픽만 통과시키는 세밀한 네트워크 제어를 할 수 있었다.
- **RBAC**는 사용자가 필요한 최소한의 권한만 가지고 시스템을 사용하도록 권한을 제어해, 보안을 더욱 강화할 수 있었다.

이로써 Kubernetes 환경에서 다층적 보안 체계를 성공적으로 구축할 수 있었으며, **Zero Trust** 원칙을 실현하는 데 중요한 3가지 기술의 결합을 이해할 수 있었다.
