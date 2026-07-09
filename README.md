# Guess Game - Kubernetes com K3D

Reimplementação da tarefa da Unidade I Docker utilizando Kubernetes.

A aplicação é composta por:

- Frontend React servido por NGINX.
- Backend Flask executado com Gunicorn.
- Banco PostgreSQL.
- HPA no backend.
- Manifests Kubernetes em `/k8s/manifests`.
- Helm Chart em `/k8s/helm/guess-game`.

O acesso principal é feito pela porta do frontend:

```text
http://localhost:3000
```

---

# 1. Requisitos básicos

Para executar esta entrega, a máquina precisa ter:

- Docker
- kubectl
- k3d
- Helm, opcional para execução via Chart

A entrega foi preparada para Kubernetes em K3D, conforme solicitado no enunciado.

Também é possível usar a máquina/OVA disponibilizada pelo curso:

```text
https://storage.googleapis.com/iec-containers-orquestration/iec-containers.zip
```

## 1.1 Instalar Docker

Ubuntu/Debian:

```bash
sudo apt update

sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Adicionar o usuário atual ao grupo Docker:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Validar:

```bash
docker ps
docker version
```

## 1.2 Instalar kubectl

```bash
sudo snap install kubectl --classic
```

Validar:

```bash
kubectl version --client
```

## 1.3 Instalar k3d

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

Validar:

```bash
k3d version
```

## 1.4 Instalar Helm

O Helm é opcional para execução, mas foi incluído como bônus na entrega.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Validar:

```bash
helm version
```

---

# 2. Clonar o repositório

```bash
git clone https://github.com/BryanWille/puc_guess_game.git

cd puc_guess_game

git switch kubernetes
```

---

# 3. Imagens Docker

As imagens da aplicação estão publicadas no Docker Hub do aluno:

```text
docker.io/bryanwille/guess-game-backend:1.0.0
docker.io/bryanwille/guess-game-frontend:1.0.0
```

Durante a avaliação, não é necessário reconstruir as imagens.

Para validar que as imagens estão disponíveis:

```bash
docker manifest inspect docker.io/bryanwille/guess-game-backend:1.0.0 > /dev/null && echo "backend dockerhub OK"

docker manifest inspect docker.io/bryanwille/guess-game-frontend:1.0.0 > /dev/null && echo "frontend dockerhub OK"
```

---

# 4. Criar cluster K3D

O frontend deve ser acessado em:

```text
http://localhost:3000
```

Para isso, o cluster K3D deve mapear a porta local `3000` para o `NodePort 30080` do Service do frontend:

```bash
k3d cluster create puc-k8s --agents 2 -p "3000:30080@server:0"
```

Validar o cluster:

```bash
kubectl cluster-info

kubectl get nodes
```

Resultado esperado:

```text
k3d-puc-k8s-server-0   Ready
k3d-puc-k8s-agent-0    Ready
k3d-puc-k8s-agent-1    Ready
```

---

# 5. Executar aplicação com manifests Kubernetes

Aplicar todos os objetos Kubernetes:

```bash
kubectl apply -k k8s/manifests
```

Acompanhar os pods:

```bash
kubectl -n guess-game get pods -w
```

Quando todos os pods estiverem `Running`, pressione `Ctrl+C`.

Validar recursos criados:

```bash
kubectl -n guess-game get pods

kubectl -n guess-game get svc

kubectl -n guess-game get hpa

kubectl -n guess-game get deployments
```

Resultado esperado dos Services:

```text
backend    ClusterIP
frontend   NodePort    80:30080/TCP
postgres   ClusterIP
```

Resultado esperado do HPA:

```text
backend-hpa   Deployment/backend   MINPODS 2   MAXPODS 5
```

---

# 6. Acessar a aplicação

Health check:

```bash
curl http://localhost:3000/health
```

Resultado esperado:

```json
{"status":"ok"}
```

Acesso pelo navegador:

```text
http://localhost:3000
```

Rotas principais:

```text
http://localhost:3000/maker
http://localhost:3000/breaker
```

---

# 7. Teste funcional

## 7.1 Criar jogo

Acesse:

```text
http://localhost:3000/maker
```

Digite uma senha e clique em criar.

O sistema retornará um `game_id`.

## 7.2 Adivinhar senha

Acesse:

```text
http://localhost:3000/breaker
```

Informe o `game_id` gerado e tente adivinhar a senha.

---

# 8. Arquitetura da solução

```text
Navegador
   |
   | http://localhost:3000
   |
K3D port mapping
   |
   | localhost:3000 -> NodePort 30080
   |
Service frontend - NodePort
   |
Deployment frontend - React + NGINX
   |
   | proxy reverso para /create, /guess e /health
   |
Service backend - ClusterIP
   |
Deployment backend - Flask + Gunicorn
   |
Service postgres - ClusterIP
   |
Deployment postgres + PVC
```

O frontend é o único componente exposto para acesso externo.

O backend e o PostgreSQL ficam acessíveis apenas dentro do cluster.

---

# 9. Componentes Kubernetes instalados

| Componente | Tipo | Arquivo | Descrição |
|---|---|---|---|
| `guess-game` | Namespace | `00-namespace.yaml` | Namespace isolado da aplicação. |
| `postgres-secret` | Secret | `01-postgres-secret.yaml` | Armazena usuário, senha e database do PostgreSQL. |
| `postgres-pvc` | PersistentVolumeClaim | `02-postgres-pvc.yaml` | Volume persistente para dados do PostgreSQL. |
| `postgres` | Deployment | `03-postgres-deployment.yaml` | Executa o banco PostgreSQL. |
| `postgres` | Service ClusterIP | `04-postgres-service.yaml` | Expõe o PostgreSQL internamente. |
| `backend-config` | ConfigMap | `05-backend-configmap.yaml` | Configura variáveis do backend. |
| `backend` | Deployment | `06-backend-deployment.yaml` | Executa a API Flask com Gunicorn. |
| `backend` | Service ClusterIP | `07-backend-service.yaml` | Expõe a API internamente para o frontend. |
| `backend-hpa` | HorizontalPodAutoscaler | `08-backend-hpa.yaml` | Autoscaling horizontal do backend. |
| `frontend` | Deployment | `09-frontend-deployment.yaml` | Executa o frontend React servido por NGINX. |
| `frontend` | Service NodePort | `10-frontend-service.yaml` | Expõe o frontend via NodePort `30080`. |
| `kustomization` | Kustomize | `kustomization.yaml` | Permite aplicar todos os manifests com `kubectl apply -k`. |

---

# 10. HPA do backend

O autoscaling horizontal foi implementado para o backend no arquivo:

```text
k8s/manifests/08-backend-hpa.yaml
```

Configuração:

```text
minReplicas: 2
maxReplicas: 5
averageUtilization: 60%
```

Verificar HPA:

```bash
kubectl -n guess-game get hpa backend-hpa

kubectl -n guess-game describe hpa backend-hpa
```

O HPA escala o Deployment `backend` com base no consumo de CPU.

---

# 11. Estrutura da entrega

Todos os objetos Kubernetes estão dentro de `/k8s`, conforme solicitado.

```text
.
├── Dockerfile.backend
├── Dockerfile.frontend
├── frontend/
├── guess/
├── repository/
├── requirements.txt
├── run.py
└── k8s/
    ├── manifests/
    │   ├── 00-namespace.yaml
    │   ├── 01-postgres-secret.yaml
    │   ├── 02-postgres-pvc.yaml
    │   ├── 03-postgres-deployment.yaml
    │   ├── 04-postgres-service.yaml
    │   ├── 05-backend-configmap.yaml
    │   ├── 06-backend-deployment.yaml
    │   ├── 07-backend-service.yaml
    │   ├── 08-backend-hpa.yaml
    │   ├── 09-frontend-deployment.yaml
    │   ├── 10-frontend-service.yaml
    │   └── kustomization.yaml
    └── helm/
        └── guess-game/
            ├── Chart.yaml
            ├── values.yaml
            └── templates/
```

---

# 12. Execução alternativa com port-forward

O acesso principal desta entrega usa `NodePort`.

Também é possível acessar via port-forward:

```bash
kubectl -n guess-game port-forward svc/frontend 3000:80
```

Acesso:

```text
http://localhost:3000
```

Neste modo, o terminal precisa permanecer aberto enquanto a aplicação estiver sendo usada.

---

# 13. Execução alternativa com Helm Chart

O Helm Chart está em:

```text
k8s/helm/guess-game
```

Instalar via Helm:

```bash
helm upgrade --install guess-game k8s/helm/guess-game
```

Validar:

```bash
kubectl -n guess-game get pods

kubectl -n guess-game get svc

kubectl -n guess-game get hpa
```

Remover instalação via Helm:

```bash
helm uninstall guess-game -n guess-game
```

---

# 14. Remover aplicação

Remover os manifests:

```bash
kubectl delete -k k8s/manifests
```

Remover o cluster K3D:

```bash
k3d cluster delete puc-k8s
```

---

# 15. Build das imagens

Esta etapa não é necessária para avaliação, pois as imagens já estão publicadas no Docker Hub.

Caso seja necessário reconstruir manualmente:

```bash
docker build -t docker.io/bryanwille/guess-game-backend:1.0.0 -f Dockerfile.backend .

docker build -t docker.io/bryanwille/guess-game-frontend:1.0.0 -f Dockerfile.frontend .
```

Publicar no Docker Hub:

```bash
docker login

docker push docker.io/bryanwille/guess-game-backend:1.0.0

docker push docker.io/bryanwille/guess-game-frontend:1.0.0
```

---

# 16. Troubleshooting

## Ver pods

```bash
kubectl -n guess-game get pods
```

## Descrever pods do backend

```bash
kubectl -n guess-game describe pod -l app=backend
```

## Logs do backend

```bash
kubectl -n guess-game logs -l app=backend --tail=100
```

## Logs do frontend

```bash
kubectl -n guess-game logs -l app=frontend --tail=100
```

## Logs do PostgreSQL

```bash
kubectl -n guess-game logs -l app=postgres --tail=100
```

## Ver Services

```bash
kubectl -n guess-game get svc
```

## Reaplicar manifests

```bash
kubectl apply -k k8s/manifests
```

## Reiniciar backend

```bash
kubectl -n guess-game rollout restart deployment/backend
```

## Reiniciar frontend

```bash
kubectl -n guess-game rollout restart deployment/frontend
```

---

# 17. Observações finais

- O sistema foi reimplementado em Kubernetes.
- A execução principal usa K3D.
- O frontend é acessado em `http://localhost:3000`.
- O frontend é exposto via `NodePort`.
- O backend possui HPA configurado.
- O backend e o PostgreSQL usam `ClusterIP`.
- Não é necessário Ingress Controller.
- As imagens estão no Docker Hub do aluno.
- Não é necessário reconstruir imagens durante a avaliação.
- Todos os objetos Kubernetes estão em `/k8s`.
- Foi incluído Helm Chart como bônus.
