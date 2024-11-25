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
            - name: kube-tester
              image: sverrirab/kube-test-container:v1.0
              ports:
              - containerPort: 8000
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
   1. Edite o arquivo `my-deployment.yaml` para atualizar a versão do `kube-test-container` para 1.1:

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
            - name: kube-tester
              image: sverrirab/kube-test-container:v1.1 # <============
              ports:
              - containerPort: 8000
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
        kubectl create service clusterip my-svc --tcp '8000:8000' -o yaml --dry-run=client | \
        kubectl set selector --local -f - 'app=myapp,version=v1.0' -o yaml | \
        kubectl create service -f -
      ```

   2. Verifique o serviço e veja os detalhees:

      ```bash
      kubectl get svc my-svc
      kubectl describe svc my-svc
      ```

   3. Certifique-se que o service está encaminhando para os pods do deployment

      ```bash
      kubectl get pods -l app=myapp -o wide
      kubectl get endpoints my-svc
      ```

2. Verificação do Serviço
   1. Obtenha os pods do `Deployment`:

      ```bash
      kubectl get pods -l app=myapp
      ```

   2. Teste a conectividade com o serviço, mas atenção! O nome `my-svc` só é resolvivel dentro do cluster. Ele é provido pelo kube-dns ou core-dns.

      ```bash
      kubectl exec -it <pod-name> -- curl my-svc
      ```

   3. Teste o aceesso ao serviço pela sua máquina utilizando o port-forward. O Comnando port-forward te permite contectar-se a uma porta sendo ouvida no cluster, por um pod ou serviço, utilizando uma porta local como ponte

      ```bash
      kubectl port-forward svc/my-svc 8080:8080 &
      curl http://localhost:8080
      ```


3. Gerenciamento do Serviço
   1. Verifique quantos endpoints existem para o serviço:

      ```bash
      kubectl get endpoints my-svc -o wide
      ```

   2. Escale o `Deployment` para 5 réplicas e verifique que o número aumentou:

      ```bash
      kubectl scale deployment my-deployment --replicas=5
      kubectl get pods -l app=myapp
      ```

   3. Verifique se o serviço está roteando tráfego para os novos pods:

      ```bash
      kubectl get endpoints my-svc -o wide
      ```

4. Limpeza
   1. Remova os recursos criados:

      ```bash
      kubectl delete svc my-svc
      kubectl delete deployment my-deployment
      ```

---

## Lab 5 Criando confiMaps

### Objetivo

Vimos como criar e configMap no primeiro Lab. Vamos aprender a criar e usá-los de outras formas

---

1. Criando configMaps a partir de um arquivo:

   1. Utilizando os arquivos no diretório lab5 crie o configmap `api-schemas`:

      ```bash
      kubectl create configmap apitool-schemas --from-file=lab5/users.json --from-file=lab5/products.json --namespace=default --dry-run=client -o yaml > api-schemas.yaml
      ```

   2. Você deve ter obtido algo parecido com isso:

      ```yaml
        apiVersion: v1
        data:
          products.json: |-
            {
                "colletction_name": "products",
                "schema": {
                    "type": "object",
                    "properties": {
                        "name": {
                            "type": "string"
                        },
                        "price": {
                            "type": "number",
                            "minimum": 0
                        },
                        "stock": {
                            "type": "integer",
                            "minimum": 0
                        }
                    },
                    "required": [
                        "name",
                        "price"
                    ]
                }
            }
          users.json: |-
            {
                "colletction_name": "users",
                "schema": {
                    "type": "object",
                    "properties": {
                        "username": {
                            "type": "string"
                        },
                        "email": {
                            "type": "string",
                            "format": "email"
                        },
                        "age": {
                            "type": "integer",
                            "minimum": 0
                        }
                    },
                    "required": [
                        "username",
                        "email"
                    ]
                }
            }
        kind: ConfigMap
        metadata:
          creationTimestamp: null
          name: apitool-schemas
          namespace: default
        
      ```

   3. Isso acontece porque o yaml é um mapa, os nomes dos campos são ordenados alfabeticamente no mesmo nível para representação. Reordanize na ordem apiVersion, kind, metadata, data. Repare que o configmap não possui spec. Não é um objeto implementável, somente possui dados.
   4. Aplique o configmap:

      ```bash
        kubectl apply -f api-schemas.yaml      
      ```

2. Agora vamos ver outra formas de criar um configmap.

   1. Crie o configmap a partir da linha de comando :

      ```bash
      kubectl create configmap -n default apitool-conf --from-literal=MONGO_URI=mongodb://mongodb.default.svc:27017/my_database --dry-run=client -o yaml > apitool-conf.yaml
      ```

   2. Vefirique o arquivo criado. Ele deve ser algo parecido com isso:

      ```yaml
        apiVersion: v1
        data:
          MONGO_URI: mongodb://mongo.default.svc:27017/my_database
        kind: ConfigMap
        metadata:
          creationTimestamp: null
          name: apitool-conf
          namespace: default
      ```

   3. Repare que a ordem apiversion, kind e metaadata não é respeitada, isso acontece porque o yaml gera um mapa e o comando, na falta de uma ordem pré-definida, coloca os elementos do mesmo nível em ordem alfabética. Veja também o _creationTimestamp: null_. Como executamos o comando com dry-run, não foi adicionado um marcador de tempo. Se desejar, pode reorganizar o arquivo. Após isso, aplique o configmap.

      ```bash
      kubectl apply -f apitool-conf.yaml
      ```

---

## Lab 6 Montando configMaps

### Objetivo

Aprender as várias formas de uso de um configmap em um pod

1. Para esse lab iremos utilizar uma aplicação simples de api. Essa api cria um CRUD simples, salvando em um banco mongodb utilizando arquivos json especificados. Por padrão, veremos que a api está provendo a coleçao users e a configuração do banco mongodb está diretamente feita no manifesto do deployment. Vamos aos passos de instalação

   1. Implante o mongo no seu cluster com o arquivo do lab6/mongo.yaml

        ```bash
        kubectl apply -f lab6/mongo.yaml
        ```

   2. Iremos implantar uma api genérica CRUD. Inicialmente iremos simplesmente instalá-la

        ```bash
        kubectl apply -f lab6/simple-crud.yaml
        ```

   3. Verifique que a api está funcionando e roteando corretamente:

        ```bash
        kubectl get pods |grep apitool
        kubectl get endpoints apitool-svc -o wide
        ```

   4. Verifique que a api está provendo a coleção users:

        ```bash
        kubectl port-forward svc/apitool-service 5000:5000 & # Esse é o nome do serviço criado pelo arquiv simple-crud.yaml
        curl http://localhost:5000/collections
        ```

   5. Você deve obter algo parecido com isso, se for formatado:

        ```json
            {
              "collections": [
                {
                  "created": true,
                  "name": "users",
                  "schema": {
                    "properties": {
                      "age": {
                        "minimum": 0,
                        "type": "integer"
                      },
                      "email": {
                        "format": "email",
                        "type": "string"
                      },
                      "password": {
                        "minLength": 8,
                        "type": "string"
                      },
                      "username": {
                        "type": "string"
                      }
                    },
                    "required": [
                      "username",
                      "email",
                      "password"
                    ],
                    "type": "object"
                  }
                }
              ]
            }
        ```

2. Vamos agora configurar utilizando os configmaps criados no lab de configmaps:

    1. Vamos obter o endpoint do mongo utilizando o configmap apitool-conf, edite o deployment do apitool e troque a variável fixa `MONGO_URI` para uma oriunda do configmap.

       ```yml
        # De 
        env:
        - name: MONGO_URI
          value: "mongodb://mongodb:27017/my_database"
        # Para 
        env :
        - name: MONGO_URI
          valueFrom:
            configMapKeyRef:
              name: apitool-conf
              key: MONGO_URI
        ```

    2. Aplique a alteração no deployment. Refaça as verificações do passo 1 para testar o funcionamento.

        ```bash
        kubectl port-forward svc/apitool-service 5000:5000 &
        curl http://localhost:5000/collections            
        ```

    3. Agora vamos montar o configmap de schemas como diretório dentro do deployment. Precisamos adicionar o configmap como volume ao pod e definir no container o seu ponto de montagem:

        No nível da definiçao de containers, adicione o volume:

        ```yaml
        volumes:
          - name: schemas
            configMap:
              name: apitool-schemas
        ```

        No nível do container, adicione o ponto de montagem:

        ```yaml
        ...
        ports:
        - containerPort: 5000
          name: http
          volumeMounts:
            - name: schemas
              mountPath: /app/schemas
        ...
        ```

        Repare que o nome do volumeMount referencia o nome dado ao colume e não ao nome do configMap. Eles podem ser iguais mas não é obrigatório

    4. Aplique a mudança no deployment e vamos verificar o funcionamento da api:

        ```kubectl
        kubectl port-forward svc/apitool-service 5000:5000 &
        curl http://localhost:5000/collections
        ```

       Você deve obeter algo parecido com isso:

       ```yml
        {
          "collections": [
            {
              "created": true,
              "name": "users",
              "schema": {
                "properties": {
                  "age": {
                    "minimum": 0,
                    "type": "integer"
                  },
                  "email": {
                    "format": "email",
                    "type": "string"
                  },
                  "username": {
                    "type": "string"
                  }
                },
                "required": [
                  "username",
                  "email"
                ],
                "type": "object"
              }
            },
            {
              "created": false,
              "name": "products",
              "schema": {
                "properties": {
                  "name": {
                    "type": "string"
                  },
                  "price": {
                    "minimum": 0,
                    "type": "number"
                  },
                  "stock": {
                    "minimum": 0,
                    "type": "integer"
                  }
                },
                "required": [
                  "name",
                  "price"
                ],
                "type": "object"
              }
            }
          ]
        }
       ```

3. Remova todos os recursos criados:

    ```bash
    kubectl delete deployment apitool
    kubectl delete statefulset mongodb
    kubectl delete configmap apitool-conf apitool-schemas
    ```
