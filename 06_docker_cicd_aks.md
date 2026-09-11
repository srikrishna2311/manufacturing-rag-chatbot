# Phase 6 — Docker + CI/CD + AKS

## Goal
Containerise the chatbot, set up the CI/CD pipeline, and deploy to AKS — same pattern as Project 1 but with secrets management for API keys.

> **Prerequisite:** Complete [Phase 5](05_ragas_evaluation.md). If you completed Project 1 first, this phase will feel very familiar.

---

## Step 1 — Dockerfile

```dockerfile
# deployment/Dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

EXPOSE 8000

CMD ["uvicorn", "src.app:app", "--host", "0.0.0.0", "--port", "8000"]
```

Build and test locally:
```bash
docker build -f deployment/Dockerfile -t rag-chatbot:latest .

docker run -p 8000:8000 \
  -e AZURE_OPENAI_ENDPOINT="your_endpoint" \
  -e AZURE_OPENAI_KEY="your_key" \
  -e AZURE_SEARCH_ENDPOINT="your_search_endpoint" \
  -e AZURE_SEARCH_KEY="your_search_key" \
  -e AZURE_SEARCH_INDEX="maintenance-manuals" \
  rag-chatbot:latest
```

---

## Step 2 — Create ACR and Push Image

```bash
az acr create \
  --resource-group rag-chatbot-rg \
  --name ragchatbotacr \
  --sku Basic

az acr login --name ragchatbotacr

docker tag rag-chatbot:latest ragchatbotacr.azurecr.io/rag-chatbot:latest
docker push ragchatbotacr.azurecr.io/rag-chatbot:latest
```

---

## Step 3 — Kubernetes Manifests

**`deployment/k8s-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rag-chatbot
  labels:
    app: rag-chatbot
spec:
  replicas: 2
  selector:
    matchLabels:
      app: rag-chatbot
  template:
    metadata:
      labels:
        app: rag-chatbot
    spec:
      containers:
        - name: rag-chatbot
          image: IMAGE_PLACEHOLDER
          ports:
            - containerPort: 8000
          env:
            - name: AZURE_OPENAI_ENDPOINT
              valueFrom:
                secretKeyRef:
                  name: openai-secrets
                  key: endpoint
            - name: AZURE_OPENAI_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-secrets
                  key: api-key
            - name: AZURE_SEARCH_ENDPOINT
              valueFrom:
                secretKeyRef:
                  name: search-secrets
                  key: endpoint
            - name: AZURE_SEARCH_KEY
              valueFrom:
                secretKeyRef:
                  name: search-secrets
                  key: api-key
            - name: AZURE_SEARCH_INDEX
              value: "maintenance-manuals"
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 20
            periodSeconds: 10
```

**`deployment/k8s-service.yaml`:** (same as Project 1)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rag-chatbot-service
spec:
  type: LoadBalancer
  selector:
    app: rag-chatbot
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

---

## Step 4 — Create AKS Cluster and Secrets

```bash
# Create AKS cluster
az aks create \
  --resource-group rag-chatbot-rg \
  --name rag-chatbot-aks \
  --node-count 1 \
  --node-vm-size Standard_DS3_v2 \
  --enable-managed-identity \
  --attach-acr ragchatbotacr \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group rag-chatbot-rg --name rag-chatbot-aks

# Create Kubernetes secrets for API keys (never put these in your YAML files)
kubectl create secret generic openai-secrets \
  --from-literal=endpoint="https://rag-chatbot-openai.openai.azure.com/" \
  --from-literal=api-key="your_openai_key"

kubectl create secret generic search-secrets \
  --from-literal=endpoint="https://rag-chatbot-search.search.windows.net" \
  --from-literal=api-key="your_search_key"
```

---

## Step 5 — GitHub Actions Pipeline

```yaml
# .github/workflows/ci-cd.yaml
name: RAG Chatbot CI/CD

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  ACR_NAME: ragchatbotacr
  IMAGE_NAME: rag-chatbot
  RESOURCE_GROUP: rag-chatbot-rg
  AKS_CLUSTER: rag-chatbot-aks

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Login to ACR
        run: az acr login --name $ACR_NAME

      - name: Build and push Docker image
        run: |
          docker build -f deployment/Dockerfile \
            -t $ACR_NAME.azurecr.io/$IMAGE_NAME:$GITHUB_SHA \
            -t $ACR_NAME.azurecr.io/$IMAGE_NAME:latest .
          docker push $ACR_NAME.azurecr.io/$IMAGE_NAME:$GITHUB_SHA
          docker push $ACR_NAME.azurecr.io/$IMAGE_NAME:latest

      - name: Set AKS context
        uses: azure/aks-set-context@v3
        with:
          resource-group: ${{ env.RESOURCE_GROUP }}
          cluster-name: ${{ env.AKS_CLUSTER }}

      - name: Deploy to AKS
        run: |
          sed -i "s|IMAGE_PLACEHOLDER|$ACR_NAME.azurecr.io/$IMAGE_NAME:$GITHUB_SHA|g" \
            deployment/k8s-deployment.yaml
          kubectl apply -f deployment/k8s-deployment.yaml
          kubectl apply -f deployment/k8s-service.yaml
          kubectl rollout status deployment/rag-chatbot --timeout=180s
```

---

## Step 6 — Test Live Endpoint

```bash
kubectl get service rag-chatbot-service
# Wait for EXTERNAL-IP

export EXTERNAL_IP="<your-external-ip>"

curl -X POST http://$EXTERNAL_IP/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the maintenance interval for pump bearings?", "session_id": "test"}'
```

---

## Checkpoint ✅

- [ ] Docker image built and pushed to ACR
- [ ] AKS cluster created with K8s secrets for API keys
- [ ] Pods running and responding at public endpoint
- [ ] CI/CD pipeline triggering on `git push`

**Next:** [Interview Talking Points](07_interview_talking_points.md)
