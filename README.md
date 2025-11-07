<div align="center">
  <h1>Repositório de Manifests K8s: GitOps com ArgoCD e FastAPI</h1>
</div>

Este repositório é a **fonte única de verdade (Single Source of Truth)** para o estado desejado da aplicação **FastAPI (CRUD de Produtos)** no ambiente Kubernetes (Rancher Desktop).

## 🎯 Fluxo GitOps

A gestão deste repositório é automatizada pelo pipeline **GitHub Actions** do repositório de código-fonte [fastapi-ci-cd](https://github.com/Lr0cha/fastapi-ci-cd)

1.  **CI/CD (Build & Push):** Após um *push* no código da aplicação, o GitHub Actions faz o **build** de uma nova imagem Docker e o **push** para o Docker Hub com uma nova tag (`lr0cha/fastapi-app:v<run_number>`).
2.  **Atualização do Manifest):** O mesmo workflow faz um **commit automático** neste repositório, **atualizando a tag da imagem** dentro do arquivo `fastapi-deployment.yaml`.
3.  **CD (Sync ArgoCD):** O **ArgoCD** detecta o novo *commit*, identifica o *drift* (diferença entre o estado do Git e o estado do Cluster) e executa a sincronização (*Auto-Sync*), realizando o **rollout** da nova versão da aplicação no Kubernetes.


## 📝 Manifests Kubernetes

Estes arquivos definem os recursos necessários para que a aplicação FastAPI seja executada e acessada no cluster.

### `fastapi-deployment.yaml`

Define o **estado desejado** para a aplicação FastAPI:

  * **`replicas: 2`**: Garante duas instâncias (Pods) da aplicação rodando para alta disponibilidade.
  * **`image:`**: O campo crítico que é **atualizado automaticamente** pelo pipeline CI/CD do GitHub Actions.
  * **`containerPort: 8000`**: A porta na qual o container da aplicação FastAPI está escutando.

<!-- end list -->

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
spec:
  replicas: 2 # Número de instâncias
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
      - name: fastapi
        # ESTE CAMPO É ATUALIZADO PELO FLUXO GITOPS DO GITHUB ACTIONS
        image: lr0cha/fastapi-app:v10 
        ports:
        - containerPort: 8000
```

### `fastapi-service.yaml`

Define como acessar a aplicação dentro do cluster.

  * **`type: ClusterIP`**: Cria um endereço IP interno, tornando a aplicação acessível apenas dentro do cluster Kubernetes (ideal para serviços internos ou para acesso via `kubectl port-forward`).
  * **`selector: app: fastapi`**: O Service aponta para os Pods que possuem o label `app: fastapi` (definido no Deployment).

<!-- end list -->

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  selector:
    app: fastapi # Aponta para os Pods do Deployment
  ports:
  - protocol: TCP
    port: 8000       # Porta que o Service expõe internamente
    targetPort: 8000 # Porta do Container
  type: ClusterIP
```

---

## 💻 Configuração do ArgoCD (Ação Única)

Para consumir este repositório, o ArgoCD precisa ser configurado para monitorá-lo. Esta ação é realizada **uma única vez** no ambiente Kubernetes de destino.

**Comando de Criação da Aplicação ArgoCD:**

```bash
argocd app create fastapi-app \
  --repo https://github.com/<SEU_USER>/<SEU_REPOSITORIO_MANIFESTS>.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automatic
```

> **Resultado Esperado:** O ArgoCD detectará e aplicará o `Deployment` e o `Service`, resultando no estado Sincronizado.


## 🔄 Monitoramento e Acesso

  * **Monitoramento:** O ArgoCD garante que o cluster reflita *exatamente* o que está definido neste repositório. O status da aplicação é monitorado pela UI do ArgoCD.
  * **Acesso (Rancher Desktop):** Para acessar a aplicação **CRUD de Produtos** no seu ambiente local:

```bash
kubectl port-forward svc/fastapi-service 8000:8000 -n default
# Acesso: http://localhost:8000
```

## 📷 Resultados da Execução

### Após o push do código da aplicação, o workflow é executado com sucesso:

<div align="center">
  <img alt="Workflow" src="https://github.com/user-attachments/assets/bafe0989-31f9-4120-8339-1421ef16d9b9"/>
  <br />
  <i>Figura 5 - Workflow do GitHub Actions com status de sucesso</i>
</div>

### A alteração da tag é sincronizada pelo ArgoCD, gerando o Rollout da nova versão:

<div align="center">
  <img alt="Rollout" src="https://github.com/user-attachments/assets/a4aa4f7e-3b9f-4f9f-a946-d72cf86f4dbe"/>
  <br />
  <i>Figura 6 - Imagem da Aplicação no ArgoCD UI no estado Sincronizado após o Rollout</i>
</div>

### O commit de atualização da tag é visível no repositório de manifests:

<div align="center">
  <img alt="Manifests log" src="https://github.com/user-attachments/assets/fd64174c-b37e-4d47-8a09-75cdb7a2e5c9"/>
  <br />
  <i>Figura 7 - Commit no Repositório de Manifests</i>
</div>

### Status dos Pods no Kubernetes:

<div align="center">
  <img alt="kubectl get pods" src="https://github.com/user-attachments/assets/70db3d5a-dc25-4ec4-a610-798c886dfd43"/>
  <br />
  <i>Figura 8 - kubectl get pods</i>
</div>

---

## ✅ Acesso à Aplicação Final

Para acessar o CRUD de Produtos rodando no seu Kubernetes local:

  * **Acesso via Port-Forward:**
    ```bash
    kubectl port-forward svc/fastapi-service 8000:8000 -n default
    ```
    
> [!NOTE]
> O port-forward permite acesso direto, sem NodePort ou LoadBalancer.

**Acesso Final:**
🔗 [http://localhost:8000](https://www.google.com/search?q=http://localhost:8000)

<div align="center">
  <img alt="FastAPI app" src="https://github.com/user-attachments/assets/9387e02d-c60b-4b6a-bf0f-186b9ad5edbc"/>
  <br />
  <i>Figura 9 - Aplicação FastAPI CRUD de Produtos rodando</i>
</div>
