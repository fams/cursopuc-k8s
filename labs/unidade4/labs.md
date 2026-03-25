# LABS

## Lab 1
### Exercício: Criação de Pods - Interativa e Declarativa

#### Objetivo:
Criar um pod no Kubernetes com o nome `my-nginx` utilizando tanto o modo interativo (comandos diretos via CLI) quanto o modo declarativo (através de um arquivo YAML).

---

### Passo 1: Criação de um Pod - Modo Interativo

No modo interativo, utilizaremos o comando `kubectl` para criar o pod diretamente a partir da linha de comando.

```bash
# Crie um pod de forma interativa chamado my-nginx usando a imagem nginx
kubectl run my-nginx --image=nginx --restart=Never
```

Este comando cria um pod chamado `my-nginx` usando a imagem oficial do Nginx, sem controle de replicação (com `--restart=Never`).

#### Verifique se o pod foi criado com sucesso:

```bash
# Verifica o status do pod
kubectl get pods
```

#### Visualize os logs do pod (opcional):

```bash
# Visualiza os logs do pod my-nginx
kubectl logs my-nginx
```

---

### Passo 2: Criação de um Pod - Modo Declarativo

Agora, criaremos o mesmo pod, mas utilizando o modo declarativo com um arquivo YAML.

#### Etapa 1: Crie um arquivo chamado `my-nginx-pod.yaml` com o seguinte conteúdo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

#### Etapa 2: Aplique o arquivo YAML para criar o pod:

```bash
# Crie o pod a partir do arquivo YAML
kubectl apply -f my-nginx-pod.yaml
```

#### Verifique se o pod foi criado com sucesso:

```bash
# Verifica o status do pod criado pelo arquivo YAML
kubectl get pods
```

---

### Passo 3: Limpeza

Após concluir os exercícios, você pode limpar o ambiente removendo o pod criado:

```bash
# Remove o pod criado de forma interativa
kubectl delete pod my-nginx

# Remove o pod criado de forma declarativa (se ainda estiver em execução)
kubectl delete -f my-nginx-pod.yaml
```

---

### Conclusão:
Neste exercício, você aprendeu como criar um pod no Kubernetes usando duas abordagens diferentes: a forma interativa com `kubectl run` e a forma declarativa com um arquivo YAML.


## Lab 2
### Objetivo:
Neste exercício, você criará um pod com dois containers: um container principal executando o Nginx e um container sidecar executando BusyBox. Além disso, você irá montar um `ConfigMap` como volume no pod.

---

### Passo 1: Criação do `ConfigMap`

Primeiro, crie um `ConfigMap` que será montado no pod.

#### Crie o arquivo `my-app-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  config-file: |
    # Configurações para o myapp-container
    server {
      listen 80;
      server_name myapp;
    }
```

#### Aplique o `ConfigMap`:

```bash
kubectl apply -f my-app-configmap.yaml
```

Verifique se o `ConfigMap` foi criado corretamente:

```bash
kubectl get configmap my-app-config -o yaml
```

---

### Passo 2: Criação do Pod com Sidecar

Agora, crie o pod com o arquivo YAML abaixo.

#### Crie o arquivo `myapp-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: myapp-container
      image: nginx:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: app-config
          mountPath: /etc/config
    - name: sidecar-container
      image: busybox
      command: ['sh', '-c', 'while true; do echo Sidecar running...; sleep 3600; done']
  volumes:
    - name: app-config
      configMap:
        name: my-app-config
```

#### Aplique o Pod:

```bash
kubectl apply -f myapp-pod.yaml
```

#### Verifique se o Pod foi criado:

```bash
kubectl get pods
```

---

### Passo 3: Verificação

#### Verifique se o container principal está rodando corretamente:

```bash
kubectl logs myapp-pod -c myapp-container
```

#### Verifique se o container sidecar está rodando corretamente:

```bash
kubectl logs myapp-pod -c sidecar-container
```

#### Verifique o conteúdo do `ConfigMap` montado no container:

```bash
kubectl exec myapp-pod -c myapp-container -- cat /etc/config/config-file
```

---

### Passo 4: Limpeza

Após completar o exercício, você pode remover os recursos criados com os seguintes comandos:

```bash
kubectl delete pod myapp-pod
kubectl delete configmap my-app-config
```

---

### Conclusão:
Neste exercício, você aprendeu a criar um pod com dois containers: um container principal executando Nginx e um container sidecar executando BusyBox. Também montamos um `ConfigMap` como volume no pod.


## Lab 3
### Objetivo:
Neste exercício, você aprenderá a trabalhar com `Deployments` no Kubernetes. Iremos criar um deployment, verificar seus detalhes, reiniciar o rollout, monitorar o status e realizar o escalonamento tanto no deployment quanto no replicaset.

---

### Passo 1: Criação do Deployment

#### Crie um arquivo chamado `my-deployment.yaml` com o seguinte conteúdo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: "v1.0"
    spec:
      containers:
      - name: nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

#### Aplique o arquivo para criar o deployment:

```bash
kubectl apply -f my-deployment.yaml
```

Verifique se o deployment foi criado corretamente:

```bash
kubectl get deployments
```

---

### Passo 2: Descrição do Deployment

Use o comando abaixo para descrever o `Deployment` e visualizar detalhes importantes, como a estratégia de atualização, número de réplicas, status e eventos.

```bash
kubectl describe deployment my-deployment
```

---

### Passo 3: Rollout do Deployment

#### Reinicie o `Deployment`:

Em certos casos, pode ser necessário reiniciar o rollout de um deployment, por exemplo, após atualizar a imagem de um container. Use o seguinte comando para reiniciar o `Deployment`:

```bash
kubectl rollout restart deployment my-deployment
```

#### Verifique o status do rollout:

Após o reinício, você pode monitorar o progresso do rollout com o seguinte comando:

```bash
kubectl rollout status deployment my-deployment
```

---

### Passo 4: Exercício de Escalonamento

Agora, vamos escalar o deployment e o replicaset.

#### 1. Escalando o Deployment

Escale o `Deployment` para 5 réplicas usando o comando:

```bash
kubectl scale deployment my-deployment --replicas=5
```

Verifique se o número de réplicas foi atualizado:

```bash
kubectl get deployments
```

#### 2. Escalando o ReplicaSet

Identifique o nome do `ReplicaSet` associado ao `Deployment` com o seguinte comando:

```bash
kubectl get rs
```

Agora, escale o `ReplicaSet` diretamente (observe que isto não é a prática comum, pois a gestão de réplicas deve ser feita através do `Deployment`):

```bash
kubectl scale rs <replica-set-name> --replicas=2
```

Verifique se o `ReplicaSet` foi escalado corretamente:

```bash
kubectl get rs
```

---

### Conclusão:
Neste exercício, você aprendeu a criar e gerenciar um `Deployment` no Kubernetes, utilizar comandos para descrever e monitorar o rollout, e realizar escalonamento tanto no `Deployment` quanto no `ReplicaSet`.

## Lab 4
### Objetivo:
Neste laboratório, você irá criar um serviço do tipo `ClusterIP` para o deployment que criamos no exercício anterior. Além disso, aprenderemos a verificar o serviço e a gerenciar suas propriedades.

---

### Passo 1: Criação do Serviço

#### Comando para criar o serviço

Utilize o comando abaixo para criar o serviço `ClusterIP` de forma declarativa, adicionando um selector que associará o serviço ao `Deployment` existente:

```bash
kubectl create service clusterip my-svc --tcp '8080:80' -o yaml --dry-run=client  | \
kubectl set selector --local -f - 'app=myapp,version=v1.0' -o yaml | \
kubectl create -f -
```

Esse comando irá criar um serviço com de nome `my-svc` que escuta na porta 8080, mas direciona para a porta 80 dos containers que possuem os labels `app=myapp` e `version=v1.0`

#### Verifique o Serviço

Após  criar o serviço, verifique se ele foi criado corretamente:

```bash
kubectl get svc my-svc
```

Verifique os detalhes completos do serviço:

```bash
kubectl describe svc my-svc
```

Verifique se o serviço tem como endpoints os pods do deployment

```bash
# Obtenha os IPs dos pods
kubectl get pods -l app=myapp -o wide
# verifique os endpoisn
kubectl get endpoints my-svc
```

---

### Passo 2: Verificação do Funcionamento do Serviço

#### Obtenha os pods do `Deployment`:

```bash
kubectl get pods -l app=myapp
```

#### Teste a conectividade com o serviço dentro do cluster:

Use um pod temporário com curl para acessar o serviço `my-svc` de dentro do cluster:

```bash
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://my-svc:8080
```

##### Atenção!! o nome my-svc só é resolvivel dentro do cluster. Ele é provido pelo kube-dns ou core-dns.


#### Conecte no serviço de sua própria máquina com o port-forward

```bash
kubectl port-forward svc/my-svc 8080:8080 &
curl http://localhost:8080
```

O Comnando port-forward te permite contectar-se a uma porta sendo ouvida no cluster, por um pod ou serviço, utilizando uma porta local como ponte

---

### Passo 3: Gerenciamento do Serviço

#### Escale o Deployment

Vamos novamente escalar o deployment para garantir que o serviço consiga rotear o tráfego para vários pods.

Escale o `Deployment` para 5 réplicas:

```bash
kubectl scale deployment my-deployment --replicas=5
```

Verifique se os pods foram criados:

```bash
kubectl get pods
```

Agora verifique se o serviço está roteando tráfego para os novos pods:

```bash
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://my-svc:8080
```

---

### Passo 4: Limpeza

Quando terminar, você pode limpar os recursos criados:

```bash
kubectl delete svc my-svc
kubectl delete deployment my-deployment
```

---

### Conclusão:
Neste laboratório, você aprendeu a criar e gerenciar um serviço no Kubernetes, associando-o a um deployment existente. Além disso, você praticou a verificação do serviço e realizou o escalonamento para garantir que ele distribuísse o tráfego entre múltiplos pods.


## Lab4
### Objetivo: Uso de ConfigMaps no Kubernetes

### Passo 1: Criação dos ConfigMaps

#### Criação do ConfigMap `contador-conf`

Crie um arquivo `contador-conf.yaml` com o seguinte conteúdo:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: contador-conf
  namespace: default
data:
  REDIS_HOST: redis.default.svc.cluster.local
  REDIS_PORT: "6379"
```

Aplique o ConfigMap com o comando:

```bash
kubectl apply -f contador-conf.yaml
```

#### Criação do ConfigMap `contador-schema`

Crie um arquivo `contador-schema.yaml` com o seguinte conteúdo:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: contador-schema
data:
  base.schema: |
    Base:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        count: 
          type: integer
  users.schema: |
    Users:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
        phone:
          type: string
```

Aplique o ConfigMap com o comando:

```bash
kubectl apply -f contador-schema.yaml
```

---

### Passo 2: Usando o ConfigMap como Variável de Ambiente (env)

#### Crie um arquivo `deployment-env.yaml` para usar o ConfigMap `contador-conf` como variáveis de ambiente:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contador-env
spec:
  replicas: 1
  selector:
    matchLabels:
      app: contador
  template:
    metadata:
      labels:
        app: contador
    spec:
      containers:
      - name: contador
        image: contador-latest:latest
        ports:
        - containerPort: 80
        env:
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: contador-conf
              key: REDIS_HOST
        - name: REDIS_PORT
          valueFrom:
            configMapKeyRef:
              name: contador-conf
              key: REDIS_PORT
```

Aplique o deployment:

```bash
kubectl apply -f deployment-env.yaml
```

#### Verifique as variáveis de ambiente:

Para verificar as variáveis de ambiente, execute o comando abaixo para abrir um shell no pod criado:

```bash
kubectl exec -it <pod-name> -- /bin/sh
```

Dentro do shell, execute:

```bash
echo $REDIS_HOST
echo $REDIS_PORT
```

---

### Passo 3: Usando todas as variáveis com `envFrom`

#### Crie um arquivo `deployment-envfrom.yaml` para usar todas as variáveis do ConfigMap de uma vez:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contador-envfrom
spec:
  replicas: 1
  selector:
    matchLabels:
      app: contador
  template:
    metadata:
      labels:
        app: contador
    spec:
      containers:
      - name: contador
        image: contador-latest:latest
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: contador-conf
```

Aplique o deployment:

```bash
kubectl apply -f deployment-envfrom.yaml
```

#### Verifique as variáveis de ambiente:

Entre no pod criado e veja todas as variáveis de ambiente:

```bash
kubectl exec -it <pod-name> -- env | grep REDIS
```

---

### Passo 4: Montando ConfigMap como Arquivo (subPath)

#### Crie um arquivo `deployment-file.yaml` para montar o `contador-schema` como arquivos específicos com `subPath`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contador-file
spec:
  replicas: 1
  selector:
    matchLabels:
      app: contador
  template:
    metadata:
      labels:
        app: contador
    spec:
      containers:
      - name: contador
        image: contador-latest:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: schemas
          mountPath: /etc/config/base.schema
          subPath: base.schema
        - name: schemas
          mountPath: /etc/config/users.schema
          subPath: users.schema
      volumes:
      - name: schemas
        configMap:
          name: contador-schema
```

Aplique o deployment:

```bash
kubectl apply -f deployment-file.yaml
```

#### Verifique os arquivos montados:

Entre no pod criado e verifique o conteúdo dos arquivos:

```bash
kubectl exec -it <pod-name> -- cat /etc/config/base.schema
kubectl exec -it <pod-name> -- cat /etc/config/users.schema
```

---

### Passo 5: Montando ConfigMap como Diretório

#### Crie um arquivo `deployment-dir.yaml` para montar o ConfigMap inteiro como diretório:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contador-dir
spec:
  replicas: 1
  selector:
    matchLabels:
      app: contador
  template:
    metadata:
      labels:
        app: contador
    spec:
      containers:
      - name: contador
        image: contador-latest:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: schemas-dir
          mountPath: /etc/schemas
      volumes:
      - name: schemas-dir
        configMap:
          name: contador-schema
```

Aplique o deployment:

```bash
kubectl apply -f deployment-dir.yaml
```

#### Verifique o diretório montado:

Entre no pod criado e liste os arquivos no diretório `/etc/schemas`:

```bash
kubectl exec -it <pod-name> -- ls /etc/schemas
```

Para verificar o conteúdo dos arquivos:

```bash
kubectl exec -it <pod-name> -- cat /etc/schemas/base.schema
kubectl exec -it <pod-name> -- cat /etc/schemas/users.schema
```

---

### Passo 6: Limpeza

Remova todos os recursos criados:

```bash
kubectl delete deployment contador-env contador-envfrom contador-file contador-dir
kubectl delete configmap contador-conf contador-schema
```