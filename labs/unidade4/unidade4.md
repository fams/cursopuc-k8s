# LABS

Esses exercícios permitem visualizar os objetos funcionando no kubernetes. Foram pensados com o kuberntes instalado pelo docker-desktop, requisito do curso

Recomenda-se criar um diretório por lab para que os arquivos criados possam ficar separados

Os fontes desses labs e também outros arquivos estarão no <https://github.com/fams/cursopuc-k8s>

## Lab 1

### Exercício: Criação de Pods - Interativa e Declarativa

#### Objetivo

Criar um pod no Kubernetes com o nome `my-nginx` utilizando tanto o modo interativo (comandos diretos via CLI) quanto o modo declarativo (através de um arquivo YAML).

---

1. Criação de um Pod - Modo Interativo

      ```bash
        # Crie um pod de forma interativa chamado `my-nginx` usando a imagem `nginx`:  
        kubectl run my-nginx --image=nginx --restart=Never
        
        # Verifique se o pod foi criado com sucesso:
        kubectl get pods
        
        # (Opcional) Visualize os logs do pod:
        kubectl logs my-nginx
      ```

2. Criação de um Pod - Modo Declarativo
   1. Crie um arquivo chamado `my-nginx-pod.yaml` com o seguinte conteúdo:

      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: my-nginx-pod
      spec:
        containers:
        - name: nginx
          image: nginx
      ```

   2. Aplique o arquivo YAML para criar o pod:

        ```bash
        # Apply declarative
        kubectl apply -f my-nginx-pod.yaml
     
        # Verifique se o pod foi criado com sucesso:
        kubectl get pods
      ```

3. Limpeza

    ```bash
    # Remova o pod criado de forma interativa:
     kubectl delete pod my-nginx
    
    # Remova o pod criado de forma declarativa (se ainda estiver em execução):
    kubectl delete -f my-nginx-pod.yaml
    ```

---

## Lab 2

### Objetivo

Criar um pod com dois containers: um container principal executando o Nginx e um container sidecar executando BusyBox. Além disso, montar um `ConfigMap` como volume no pod.

1. Criação do `ConfigMap`
   1. Crie o arquivo `my-app-configmap.yaml`:

      ```yaml
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: my-app-config
      data:
        config-file: |
          server {
            listen 80;
            server_name myapp;
          }
      ```

   2. Aplique o `ConfigMap`:

      ```bash
      kubectl apply -f my-app-configmap.yaml

      # Verifique se o `ConfigMap` foi criado corretamente:
      kubectl get configmap my-app-config -o yaml
      ```

2. Criação do Pod com Sidecar
   1. Crie o arquivo `myapp-pod.yaml`:

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

   2. Aplique o Pod:

        ```bash
        kubectl apply -f myapp-pod.yaml

        # Verifique se o Pod foi criado:
        kubectl get pods

        # Verifique os logs do container principal:
      
        kubectl logs myapp-pod -c myapp-container
      
        # Verifique os logs do container sidecar:
      
        kubectl logs myapp-pod -c sidecar-container
      
        # Verifique o conteúdo do `ConfigMap` montado:
      
        kubectl exec myapp-pod -c myapp-container -- cat /etc/config/config-file
        ```

3. Limpeza
   1. Remova os recursos criados:

      ```bash
      kubectl delete pod myapp-pod
      kubectl delete configmap my-app-config
      ```

---

## Lab 3

### Objetivo

Criar e gerenciar um `Deployment` no Kubernetes, verificar seus detalhes, reiniciar o rollout, monitorar o status e escalar o deployment e o replicaset.

---

1. Criação do Deployment
   1. Crie um arquivo chamado `my-deployment.yaml`:

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
              image: nginx:1.26
              ports:
              - containerPort: 80
        strategy:
          type: RollingUpdate
          rollingUpdate:
            maxUnavailable: 1
            maxSurge: 1
      ```

   2. Aplique o arquivo para criar o deployment:

      ```bash
      kubectl apply -f my-deployment.yaml

      # Verifique se o deployment foi criado corretamente:
      kubectl get deployments

      # Descreva o `Deployment` para ver detalhes como a estratégia de atualização e eventos:
      kubectl describe deploy my-deployment
       ```

2. Disparando um rollout por alteração do manifesto
   1. Edite o arquivo `my-deployment.yaml` para atualizar a versão do `nginx` para 1.27:

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
              image: nginx:1.27 # <=====
              ports:
              - containerPort: 80
        strategy:
          type: RollingUpdate
          rollingUpdate:
            maxUnavailable: 1
            maxSurge: 1
      ```

   2. Aplique a nova configuração para iniciar o rollout da versão 1.27:

      ```bash
      kubectl apply -f my-deployment.yaml

      # Monitore o progresso do rollout:
      kubectl rollout status deployment my-deployment
      
      # Descreva o `Deployment` para ver detalhes como a estratégia de atualização e eventos:
      kubectl describe deploy my-deployment
      ```

3. Rollout do Deployment sem alteração de template

      ```bash
      # Reinicie os PODs do  `Deployment`:
      kubectl rollout restart deployment my-deployment
      
      # Monitore o progresso do rollout:
      kubectl rollout status deployment my-deployment
      ```

4. Exercício de Escalonamento
   1. Escale o `Deployment` para 5 réplicas:

      ```bash
      kubectl scale deployment my-deployment --replicas=5
      
      # Verifique se o número de réplicas foi atualizado:
      kubectl get deployments
      ```

   2. Escale o `ReplicaSet` diretamente (não recomendado, mas ilustrativo):

      ```bash
      # Identifique o nome do `ReplicaSet`. Repare no prefixo do RS:
      kubectl get rs |grep my-deployment
      
      # Escale o `ReplicaSet`:
      kubectl scale rs <replica-set-name> --replicas=2
      
      # Verifique o `ReplicaSet`:
      kubectl get rs
      ```

---

## Lab 4

### Objetivo

Criar um serviço `ClusterIP` para o `Deployment` criado anteriormente, verificar seu funcionamento e gerenciar suas propriedades.

1. Criação do Serviço
   1. Crie o serviço `ClusterIP` de forma declarativa:

      ```bash
      kubectl create service clusterip my-svc -o yaml --dry-run=client | \
      kubectl set selector --local 'app.kubernetes.io/name=nginx' -f - -o yaml | \
      kubectl create -f -
      ```

   2. Verifique o serviço:

      ```bash
      kubectl get svc my-svc
      ```

   3. Verifique detalhes do serviço:

      ```bash
      kubectl describe svc my-svc
      ```

2. Verificação do Serviço
   1. Obtenha os pods do `Deployment`:

      ```bash
      kubectl get pods -l app=myapp
      ```

   2. Teste a conectividade com o serviço:

      ```bash
      kubectl exec -it <pod-name> -- curl my-svc
      ```

3. Gerenciamento do Serviço
   1. Escale o `Deployment` para 5 réplicas:

      ```bash
      kubectl scale deployment my-deployment --replicas=5
      ```

   2. Verifique se o serviço está roteando tráfego para os novos pods:

      ```bash
      kubectl exec -it <pod-name> -- curl my-svc
      ```

4. Limpeza
   1. Remova os recursos criados:

      ```bash
      kubectl delete svc my-svc
      kubectl delete deployment my-deployment
      ```
