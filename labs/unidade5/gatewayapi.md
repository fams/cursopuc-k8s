# LABS — Gateway API

Esses exercícios permitem visualizar a Gateway API funcionando no Kubernetes. Foram pensados com o Kubernetes instalado pelo k3d, requisito do curso.

Recomenda-se criar um diretório por lab para que os arquivos criados possam ficar separados.

```bash
mkdir -p lab-gateway/{lab2,lab3,lab4,lab5}
```

> **Pré-requisito:** O cluster k3d-lab precisa estar em execução.

---

## Lab 1

### Exercício: Instalação da Gateway API e do NGINX Gateway Fabric

#### Objetivo:

Instalar os CRDs da Gateway API no cluster e o NGINX Gateway Fabric como implementação do controller. Ao final, o cluster terá o objeto `GatewayClass` disponível e pronto para uso.

> **Nota k3d:** O k3d já vem com o Traefik como ingress controller. O NGINX Gateway Fabric é instalado **em paralelo**, em seu próprio namespace (`nginx-gateway`), sem interferir no Traefik existente.

---

### Passo 1: Verificar o cluster

```bash
kubectl cluster-info
kubectl get nodes
```

---

### Passo 2: Instalar os CRDs da Gateway API

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

#### Verifique os CRDs instalados:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

A saída esperada deve conter:

```
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
grpcroutes.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
referencegrants.gateway.networking.k8s.io
```

---

### Passo 3: Instalar o NGINX Gateway Fabric

```bash
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/default/deploy.yaml
```

#### Verifique se o pod do controller está em execução:

```bash
kubectl get pods -n nginx-gateway
```

#### Aguarde o pod ficar pronto:

```bash
kubectl wait --timeout=90s --for=condition=ready \
  pod -l app.kubernetes.io/name=nginx-gateway \
  -n nginx-gateway
```

> **Nota k3d:** Se o pod ficar preso em `1/2 Running` após um reinício do cluster, force a recriação:
>
> ```bash
> kubectl delete pod -n nginx-gateway -l app.kubernetes.io/name=nginx-gateway
> kubectl wait --timeout=90s --for=condition=ready \
>   pod -l app.kubernetes.io/name=nginx-gateway \
>   -n nginx-gateway
> ```

---

### Passo 4: Verificar o GatewayClass

```bash
kubectl get gatewayclass
```

A saída esperada é:

```
NAME    CONTROLLER                                    ACCEPTED   AGE
nginx   gateway.nginx.org/nginx-gateway-controller   True       ...
```

```bash
kubectl describe gatewayclass nginx
```

---

### Passo 5: Limpeza

Não remova os recursos instalados neste lab, pois eles são necessários para os labs seguintes.

---

### Conclusão:

Neste lab, você instalou os CRDs da Gateway API e o NGINX Gateway Fabric. Agora o cluster tem um `GatewayClass` chamado `nginx` e está pronto para receber objetos `Gateway` e `HTTPRoute`.

---

## Lab 2

### Exercício: Criando o primeiro Gateway e HTTPRoute

#### Objetivo:

Criar um `Gateway` e uma `HTTPRoute` para expor um serviço HTTP simples. Ao final, você conseguirá acessar o serviço através do Gateway usando `curl`.

---

### Passo 1: Criar o namespace e a aplicação de teste

```bash
kubectl create namespace gw-demo
```

#### Crie o arquivo `demo-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: gw-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
        version: v1
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo:latest
        args:
        - "-text=Olá da versão 1!"
        - "-listen=:5678"
        ports:
        - containerPort: 5678
```

#### Crie o arquivo `demo-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: gw-demo
spec:
  selector:
    app: demo-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

#### Aplique os recursos:

```bash
kubectl apply -f demo-deployment.yaml
kubectl apply -f demo-service.yaml

kubectl get pods,svc -n gw-demo
```

---

### Passo 2: Criar o Gateway

O `Gateway` define o listener — a porta e o protocolo que o controller vai expor.

#### Crie o arquivo `gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
  namespace: gw-demo
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
```

```bash
kubectl apply -f gateway.yaml
```

#### Verifique o status do Gateway:

```bash
kubectl get gateway -n gw-demo
```

Aguarde até que a coluna `PROGRAMMED` mostre `True`.

> **Nota k3d:** O campo `ADDRESS` ficará vazio porque o serviço `nginx-gateway` do tipo `LoadBalancer` não recebe IP externo no k3d. O acesso será feito via `port-forward`, conforme o Passo 4.

```bash
kubectl describe gateway demo-gateway -n gw-demo
```

---

### Passo 3: Criar a HTTPRoute

#### Crie o arquivo `httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "demo.localdev.me"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: demo-app
      port: 80
```

```bash
kubectl apply -f httproute.yaml

kubectl get httproute -n gw-demo
```

A coluna `ACCEPTED` deve mostrar `True`.

---

### Passo 4: Testar o acesso

#### Configure o port-forward para acessar o Gateway localmente:

```bash
kubectl port-forward svc/nginx-gateway -n nginx-gateway 8080:80 &
```

#### Teste com curl:

```bash
# O domínio demo.localdev.me resolve para 127.0.0.1 automaticamente
curl http://demo.localdev.me:8080/

# Se demo.localdev.me não resolver na sua máquina, use o header Host
curl -H "Host: demo.localdev.me" http://localhost:8080/
```

A saída esperada é:

```
Olá da versão 1!
```

#### Teste com um host não configurado na HTTPRoute:

```bash
# Deve retornar 404
curl -H "Host: desconhecido.exemplo.com" http://localhost:8080/
```

---

### Passo 5: Limpeza

Não remova os recursos deste lab. O `demo-app` e o `demo-gateway` serão reutilizados nos próximos labs.

---

### Conclusão:

Neste lab, você criou um `Gateway` e uma `HTTPRoute` para expor um serviço HTTP. Observe a separação de objetos: o `Gateway` cuida do listener (infraestrutura), e a `HTTPRoute` cuida das regras de roteamento (aplicação).

---

## Lab 3

### Exercício: Traffic Splitting — Canary Deployment

#### Objetivo:

Usar a `HTTPRoute` para dividir tráfego entre duas versões da aplicação — 80% para a versão estável (v1) e 20% para a nova versão (v2).

---

### Passo 1: Criar a versão 2 da aplicação

#### Crie o arquivo `demo-app-v2.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app-v2
  namespace: gw-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app-v2
  template:
    metadata:
      labels:
        app: demo-app-v2
        version: v2
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo:latest
        args:
        - "-text=Olá da versão 2 (canary)!"
        - "-listen=:5678"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-v2
  namespace: gw-demo
spec:
  selector:
    app: demo-app-v2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

```bash
kubectl apply -f demo-app-v2.yaml

kubectl get pods -n gw-demo -L version
```

---

### Passo 2: Atualizar a HTTPRoute com traffic splitting

#### Crie o arquivo `httproute-canary.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "demo.localdev.me"
  rules:
  - backendRefs:
    - name: demo-app
      port: 80
      weight: 80
    - name: demo-app-v2
      port: 80
      weight: 20
```

```bash
kubectl apply -f httproute-canary.yaml
```

---

### Passo 3: Verificar a distribuição de tráfego

```bash
for i in $(seq 1 20); do
  curl -s -H "Host: demo.localdev.me" http://localhost:8080/
done | sort | uniq -c | sort -rn
```

A saída esperada (aproximada):

```
 16 Olá da versão 1!
  4 Olá da versão 2 (canary)!
```

> **Nota:** O traffic splitting por peso é probabilístico. Os números exatos podem variar.

---

### Passo 4: Promover a v2 para 100%

#### Crie o arquivo `httproute-v2-full.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "demo.localdev.me"
  rules:
  - backendRefs:
    - name: demo-app-v2
      port: 80
      weight: 100
```

```bash
kubectl apply -f httproute-v2-full.yaml

curl -H "Host: demo.localdev.me" http://localhost:8080/
```

#### Reverta para o traffic splitting original:

```bash
kubectl apply -f httproute-canary.yaml
```

---

### Passo 5: Limpeza

Mantenha os recursos. O `demo-app`, `demo-app-v2` e o `demo-gateway` são usados nos próximos labs.

---

### Conclusão:

Neste lab, você implementou um canary deployment usando o campo `weight` nos `backendRefs` da `HTTPRoute`. Esse recurso é nativo da Gateway API — no Ingress, seria necessário uma annotation proprietária de cada controller.

---

## Lab 4

### Exercício: Roteamento por Header e por Caminho

#### Objetivo:

Criar regras de roteamento baseadas em HTTP headers e em caminhos (path).

---

### Passo 1: HTTPRoute com roteamento por header

#### Crie o arquivo `httproute-header.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "demo.localdev.me"
  rules:
  # Regra 1: requisições com header X-Version: v2 vão para demo-app-v2
  - matches:
    - headers:
      - name: X-Version
        value: v2
    backendRefs:
    - name: demo-app-v2
      port: 80
  # Regra 2: todo o restante vai para demo-app (v1)
  - backendRefs:
    - name: demo-app
      port: 80
```

```bash
kubectl apply -f httproute-header.yaml
```

---

### Passo 2: Testar o roteamento por header

```bash
# Sem header — deve responder com a versão 1
curl -H "Host: demo.localdev.me" http://localhost:8080/

# Com header X-Version: v2 — deve responder com a versão 2
curl -H "Host: demo.localdev.me" -H "X-Version: v2" http://localhost:8080/
```

---

### Passo 3: HTTPRoute com roteamento por caminho

#### Crie o arquivo `httproute-path.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "demo.localdev.me"
  rules:
  # Regra 1: caminho /v2/ vai para demo-app-v2
  - matches:
    - path:
        type: PathPrefix
        value: /v2
    backendRefs:
    - name: demo-app-v2
      port: 80
  # Regra 2: caminho / vai para demo-app (v1)
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: demo-app
      port: 80
```

```bash
kubectl apply -f httproute-path.yaml
```

---

### Passo 4: Testar o roteamento por caminho

```bash
# Caminho raiz — deve ir para v1
curl -H "Host: demo.localdev.me" http://localhost:8080/

# Caminho /v2 — deve ir para v2
curl -H "Host: demo.localdev.me" http://localhost:8080/v2
```

---

### Passo 5: Combinando header e path na mesma regra

É possível combinar múltiplos critérios — **todos** precisam ser satisfeitos (lógica AND).

#### Crie o arquivo `httproute-combined.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "demo.localdev.me"
  rules:
  # Regra 1: caminho /api/ E header X-Version: v2 -> v2
  - matches:
    - path:
        type: PathPrefix
        value: /api
      headers:
      - name: X-Version
        value: v2
    backendRefs:
    - name: demo-app-v2
      port: 80
  # Regra 2: caminho /api/ sem header específico -> v1
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: demo-app
      port: 80
  # Regra 3: tudo o restante -> v1
  - backendRefs:
    - name: demo-app
      port: 80
```

```bash
kubectl apply -f httproute-combined.yaml

# Teste 1: /api sem header -> v1
curl -H "Host: demo.localdev.me" http://localhost:8080/api

# Teste 2: /api com X-Version: v2 -> v2
curl -H "Host: demo.localdev.me" -H "X-Version: v2" http://localhost:8080/api

# Teste 3: / sem header -> v1
curl -H "Host: demo.localdev.me" http://localhost:8080/
```

---

### Passo 6: Limpeza parcial

Restaure a HTTPRoute para o estado simples antes do próximo lab:

```bash
kubectl apply -f httproute.yaml
```

---

### Conclusão:

Neste lab, você usou `matches` com `headers` e `path` para criar regras de roteamento avançadas. Na Gateway API, header matching é um campo nativo e portável entre implementações.

---

## Lab 5

### Exercício: Filtros — Redirect, Rewrite e Response Headers

#### Objetivo:

Usar os `filters` da `HTTPRoute` para redirecionar requisições, reescrever caminhos e adicionar response headers.

---

### Passo 1: Redirect de HTTP para HTTPS

#### Crie o arquivo `httproute-redirect.yaml`:

> **Atenção:** Para testar o redirect, a `HTTPRoute` `demo-route` (que também usa o hostname `demo.localdev.me`) deve ser removida antes, caso contrário suas regras de path mais específicas têm prioridade sobre o redirect.
>
> ```bash
> kubectl delete httproute demo-route -n gw-demo
> ```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-redirect
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "demo.localdev.me"
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

```bash
kubectl apply -f httproute-redirect.yaml

# Teste o redirect — espera-se um HTTP 301
curl -v -H "Host: demo.localdev.me" http://localhost:8080/ 2>&1 | grep -E "< HTTP|< Location"
```

A saída esperada:

```
< HTTP/1.1 301 Moved Permanently
< Location: https://demo.localdev.me/
```

---

### Passo 2: URL Rewrite — remover prefixo de caminho

#### Restaure a HTTPRoute principal antes de continuar:

```bash
kubectl apply -f httproute.yaml
```

#### Crie o arquivo `httproute-rewrite.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-rewrite
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "rewrite.localdev.me"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/v1
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: demo-app
      port: 80
```

```bash
kubectl apply -f httproute-rewrite.yaml

# O cliente acessa /api/v1/ mas o serviço recebe /
curl -H "Host: rewrite.localdev.me" http://localhost:8080/api/v1/
```

---

### Passo 3: Adicionar Response Headers

> **Nota k3d / NGINX Gateway Fabric:** O header `Server` é protegido e não pode ser removido via filtro. O exemplo abaixo adiciona headers customizados sem tentar remover `Server`.

#### Crie o arquivo `httproute-headers-response.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-response-headers
  namespace: gw-demo
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "headers.localdev.me"
  rules:
  - filters:
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        add:
        - name: X-Powered-By
          value: Gateway-API
        - name: X-Environment
          value: producao
    backendRefs:
    - name: demo-app
      port: 80
```

```bash
kubectl apply -f httproute-headers-response.yaml

# Observe os headers na resposta
curl -I -H "Host: headers.localdev.me" http://localhost:8080/
```

A saída deve conter:

```
X-Powered-By: Gateway-API
X-Environment: producao
```

---

### Passo 4: Verificar todos os objetos criados nos labs

```bash
kubectl get gateway -n gw-demo
kubectl get httproute -n gw-demo
kubectl get httproute -n gw-demo -o wide
```

---

### Passo 5: Limpeza geral

Ao finalizar todos os labs, remova todos os recursos criados:

```bash
# Remove todos os recursos do namespace gw-demo
kubectl delete namespace gw-demo

# Remove o NGINX Gateway Fabric
kubectl delete -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/default/deploy.yaml
kubectl delete -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/v1.4.0/deploy/crds.yaml

# Remove os CRDs da Gateway API
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml

# Encerra o port-forward se ainda estiver rodando
kill %1 2>/dev/null || true
```

---

### Conclusão:

Neste lab, você usou filtros nativos da Gateway API: `RequestRedirect`, `URLRewrite` e `ResponseHeaderModifier`. Todos esses recursos são portáveis — funcionam da mesma forma em qualquer implementação da Gateway API compatível.

---

## Resumo dos Objetos Criados

| Lab | Objeto | Nome | Namespace |
|-----|--------|------|-----------|
| Lab 1 | `GatewayClass` | `nginx` | cluster-scoped |
| Lab 2 | `Gateway` | `demo-gateway` | `gw-demo` |
| Lab 2 | `HTTPRoute` | `demo-route` | `gw-demo` |
| Lab 3 | `HTTPRoute` | `demo-route` (atualizada) | `gw-demo` |
| Lab 4 | `HTTPRoute` | `demo-route` (atualizada) | `gw-demo` |
| Lab 5 | `HTTPRoute` | `demo-redirect`, `demo-rewrite`, `demo-response-headers` | `gw-demo` |
