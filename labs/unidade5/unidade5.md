# LABS

Esses exercícios permitem visualizar os objetos funcionando no kubernetes. Foram pensados com o kubernetes instalado pelo k3d, requisito do curso

Recomenda-se criar um diretório por lab para que os arquivos criados possam ficar separados

Os fontes desses labs e também outros arquivos estarão no <https://github.com/fams/cursopuc-k8s>

## LAB 7

### Objetivo: Criando um Persistent Volume estaticamente provisionado. Aprender a pré-provisionar volumes no kubernetes e passar pelas fases do gerenciamento de volumes.

1. Vamos criar dois pods um gravando e outro lendo no mesmo disco via provisionamento direto

    1. Crie os deployments gravador e leitor

        ```bash
        Crie o escritor
        kubectl apply -f lab7/writer.yaml
        # Espere pelo provisionamentod do pod
        kubectl get pod -w # Quando o escritor estiver no ar, digite CTRL+C
        # Crie o Leitor
        kubectl apply -f lab7/reader.yaml
        ```

    2. Você pode verificar que um está gravando no disco e o outro lendo utilizando o comando `kubectl logs`. Para interrompoer o log contínuo, digite CTRL+C ou não utilize o -f

        ```bash
        # Logs do escritor     
        kubectl logs $(kubectl get pod -l app=alpine-writer -o name) -f
        # logs do leitor
        kubectl logs $(kubectl get pod -l app=alpine-reader -o name) -f
        ```

    3. Esse provisionamento utilizando diretamente o tipo de volume, no caso hostPath, não é recomendado e só é possível que dois pods possam acessá-lo porque os dois pods estão no mesmo host. Alguns tipos de volumes permitem que mais de um host possam acessar o mesmo disco, mas a forma de provisionamento será outro.

        > **k3d:** Os manifestos do lab7 já incluem `nodeSelector: kubernetes.io/hostname: k3d-lab-agent-0` nos deployments writer e reader, garantindo que ambos caiam no mesmo nó. O `type: DirectoryOrCreate` faz o kubelet criar o diretório automaticamente no primeiro uso.

2. Agora vamos fazer o provisionamento direto utilizando o recurso de persistentVolume

    1. Crie os discos pré-provisionados utilizando os manifestos do lab7

        ```bash
        kubectl apply -f lab7/pre-provisioned.yaml
        # Verifique a criação
        kubectl get pv
        ```

        Você vai ver algo parecido com isso:

        ```output
        NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
        manual-pv-1g                               1Gi        RWO            Delete           Available                                                               <unset>                          160m
        manual-pv-2g                               2Gi        RWO            Delete           Available                                                               <unset>                          160m
        ```

    2. Vamos agora criar um PVC, um persistentVolumeClaim que se ligue em um dos volumes

        ```bash
        kubectl apply -f lab7/pvc-2G.yaml
        kubectl get pv
        ```

    3. Verifique que os PVs foram criados, repare nas colunas STATUS e CLAIM

        ```output
        NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
        manual-pv-1g                               1Gi        RWO            Delete           Available                                                               <unset>                          161m
        manual-pv-2g                               2Gi        RWO            Delete           Bound       default/static-claim                                        <unset>                          161m
        ```

        Existindo Discos pré-provisionados com as mesmas características do PVC, o kubernetes irá ligar o PVC a ele. As duas tabelas abaixo são só pra comparação lado a lado -- não precisam ser reaplicadas, o PV e o PVC já foram criados nos passos anteriores.

        <!--send:off-->

        <table>
        <tr><th>PV</th><th>PVC</th></tr>
        <tr><td>

        ```yaml
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: manual-pv-2g
        spec:
          accessModes:
          - ReadWriteOnce            # <- AccessMode
        
          capacity:
            storage: 2Gi             # <- Storage Size
          hostPath:
            path: /var/lib/k8s-pvs/manual-pv-2
          persistentVolumeReclaimPolicy: Delete
          volumeMode: Filesystem
        ```

        </td>
        <td>

        ```yaml
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: static-claim
        spec:
          accessModes:
          - ReadWriteOnce           # <- AccessMode
          resources:
            requests:
              storage: 2Gi          # <- Storage Size
          storageClassName: ""
        
        
          volumeMode: Filesystem
        ```

        </td></tr>
        </table>

    4. Agora vamos montar os pods reader e writer usando o PVC:

        ```bash
        kubectl apply -f lab7/writer-pvc.yaml
        kubectl get pod -w
        kubectl apply -f lab7/reader-pvc.yaml
        ```

    5. Verifique o funcionamento dos pods reader e writer utilizando o volume com o `PVC`:

       ```bash
        # Logs do escritor     
        kubectl logs $(kubectl get pod -l app=alpine-writer -o name) -f
        # logs do leitor
        kubectl logs $(kubectl get pod -l app=alpine-reader -o name) -f
        ```

3. Limpeza:

     ```bash
     kubectl delete -f lab7/writer-pvc.yaml
     kubectl delete -f lab7/reader-pvc.yaml
     kubectl delete -f lab7/pvc-2G.yaml
     kubectl delete -f lab7/pre-provisioned.yaml
    ```

---

## LAB 8

### Objetivo: Provisionando volumes de forma dinâmica. Aprender a utilizar volumes provisionados dinâmicamente no kubernetes e passar pelas fases do gerenciamento de volumes.

1. Entenda o StorageClass: o provisionamento dinâmico depende dele, uma espécie de perfil de criação de Volumes para o cluster. O StorageClass pré-existente no k3d é o `local-path`:

    <!--send:off-->

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-path
    provisioner: rancher.io/local-path
    reclaimPolicy: Delete
    volumeBindingMode: WaitForFirstConsumer
    ```

    > **Atenção:** O `volumeBindingMode: WaitForFirstConsumer` significa que o `PV` só será criado quando um `Pod` tentar montar o `PVC`, e não no momento em que o `PVC` é criado.

2. Entenda os campos do StorageClass: o `provisioner` define qual módulo de provisionamento instalado no cluster será utilizado (hoje em dia majoritariamente via CSI, Container Storage Interface, que pode ser de terceiros); o `volumeBindingMode` informa se o `PV` deve ser criado ao se ligar ao `PVC` ou quando o `POD` tentar montá-lo; o `reclaimPolicy` tem o mesmo papel que no `PV`. Existem outros campos disponíveis, como parâmetros que passam argumentos para o provisionador.

3. Vamos agora provisionar um `PV` utilizando `PVC` com StorageClass

    1. Aplique o manifesto do `PVC`:

        ```bash
        kubectl apply -f lab8/pvc-dynamic.yaml
        kubectl get pvc dynamic-claim
        ```

    2. Verifique a criaçao do `PV`:

        ```bash
        kubectl get pv
        ```

4. Podemos agora criar os `deployments` writer e reader utilizando esse `PVC`

    1. Agora vamos montar os pods reader e writer usando o PVC:

        ```bash
        kubectl apply -f lab8/writer-pvc.yaml
        kubectl get pod -w
        kubectl apply -f lab8/reader-pvc.yaml
        ```

    2. Verifique o funcionamento dos pods reader e writer utilizando o volume com o `PVC`:

       ```bash
        # Logs do escritor     
        kubectl logs $(kubectl get pod -l app=alpine-writer -o name) -f
        # logs do leitor
        kubectl logs $(kubectl get pod -l app=alpine-reader -o name) -f
        ```

5. Limpeza:

    ```bash
    kubectl delete -f lab8/writer-pvc.yaml
    kubectl delete -f lab8/reader-pvc.yaml
    kubectl delete -f lab8/pvc-dynamic.yaml
    ```

---

## LAB 9

### Objetivo: RBAC. Compreender o funcionamento do controle de acesso RBAC no kubernetes -- o controle de acesso pode ser dividido entre AuthN e AuthZ (autenticação e autorização); o foco aqui é AuthZ, já que existem diversas formas de autenticação no Kubernetes. Pra melhor visualização das saídas, recomenda-se ter o comando `jq` instalado.

1. Vamos criar um usuário `puc-devops` com autenticação por certificado.

   1. Criando o CSR para o usuário:

        ```bash
        # Criando uma chave privada RSA 
        openssl genrsa -out puc-devops.pem
        
        # Gerando um Certificate Signing Request. Na autenticação por mTLS, o CN do subject será lido como username no kubernetes e o Organization (O) será lido como grupo 
        openssl req -new -key puc-devops.pem -out puc-devops.csr -subj "/CN=puc-devops/O=devs"
        
        # Gerando um manifesto de CSR no kubernetes
        cat <<EOF | kubectl apply -f - 
        apiVersion: certificates.k8s.io/v1
        kind: CertificateSigningRequest
        metadata:
          name: puc-devops
        spec:
          request: $(cat puc-devops.csr |base64 -w0)
          signerName: kubernetes.io/kube-apiserver-client
          expirationSeconds: 86400  # one day
          usages:
          - digital signature
          - key encipherment
          - client auth
        EOF
        
        # Obtendo o status
        kubectl get csr
        ```

        Resultado será algo assim

        ```output
        certificatesigningrequest.certificates.k8s.io/puc-devops created
        NAME         AGE   SIGNERNAME                            REQUESTOR   REQUESTEDDURATION   CONDITION
        puc-devops   0s    kubernetes.io/kube-apiserver-client   k3d-lab     24h                 Pending
        ```

   2. Obtento o certificado e configurando o usuário:

        ```bash
            # Aprove o certificado
        kubectl certificate approve puc-devops
        
        # Verifique que o certificado está aprovado e obtenha o PEM 
        kubectl get csr puc-devops
        ```

        ```output
        NAME         AGE   SIGNERNAME                            REQUESTOR   REQUESTEDDURATION   CONDITION
        puc-devops   45s   kubernetes.io/kube-apiserver-client   k3d-lab     24h                 Approved,Issued        
        ```

        ```bash
        # Obter o certificado assinado pela CA do cluster, salvando-o em formatdo PEM e mostrando seus atributos na tela
        
        kubectl get csr puc-devops -o jsonpath="{.status.certificate}"|base64 -d|tee puc-devops.crt |openssl x509 -text -noout
        ```

   3. Configurando o acesso com o novo usuário

        ```bash
        # Criando o usuário no arquivo de configuração usando o .crt obtido do cluster e o .pem gerado
        kubectl  config set-credentials puc-devops --client-certificate=puc-devops.crt --client-key=puc-devops.pem --embed-certs=true

        # Visualize a configuração
        kubectl config view -o jsonpath='{.users[?(@.name=="puc-devops")]}' | jq
        ```

        ```bash
        # Obtendo dados da configuração atual
        
        # Obtendo o contexto. Contexto é uam configuração de acesso ao kubernets que possui os dados de acesso à API e os dados de autenticação.Cluster + User
        kubectl config view -o jsonpath='{.current-context}'
        # No meu caso:
        "k3d-lab"

        # Obtendo os dados do contexto
        kubectl config view -o jsonpath='{.contexts[?(@.name =="k3d-lab")]}' | jq

        {
          "name": "k3d-lab",
          "context": {
            "cluster": "k3d-lab",
            "user": "admin@k3d-lab"
          }
        }

        # Criando um contexto com o novo usuário com o cluster do contexto atual
        kubectl config set-context k3d-lab-puc-devops --cluster=k3d-lab --user=puc-devops

        kubectl config view -o jsonpath='{.contexts[?(@.name =="k3d-lab-puc-devops")]}'
        ```

        Agora você pode fazer chamadas para o cluster com o novo usuário, mas esse novo usuário não tem nenhuma permissão:

        ```bash
        kubectl --context k3d-lab-puc-devops get pod
        ```

        Também é possível definir o contexto default para o novo contexto, sem ter que passar o nome na linha de comando, porem será mais difícil fazer a configuração das pemissões.

2. Iremos dar permissão para o novo usuário ler os pods no namespace kube-system:

    1. Criemos a role de permissão de acesso ler e listar os pods e também a rolebinding que permite o usuário puc-devops utilizá-la

       ```bash
       # Crie a role pod-reader utilizando o manifesto presente no diretório lab9
       kubectl apply -f lab9/role-pod-reader.yaml
       
       # Faça a ligação da role, que contem as permissões com o usuário que será autenticado através do certificado
       kubectl apply -f lab9/rolebinding-pod-reader-user-puc-devops.yaml
       ```

       Agora é possível ler os pods da namespace kube-system com o usuário puc-devops

       ```bash
       kubectl --context k3d-lab-puc-devops -n kube-system get pod
       ```

       ```output
       NAME                                      READY   STATUS    RESTARTS   AGE
       coredns-6799fbcd5-xxxxx                   1/1     Running   0          5m
       local-path-provisioner-6c86858495-xxxxx   1/1     Running   0          5m
       metrics-server-54fd9b65b-xxxxx            1/1     Running   0          5m
       ```

3. A permissão de role e role-binding vale somente para a namespace que os objetos foram criados. Vamos utilizar permissões que valem para todo o cluster

    1. Testando o acesso em outro namespacee

       ```bash
       kubectl --context k3d-lab-puc-devops -n default get pod
       ```

       ```output
       Error from server (Forbidden): pods is forbidden: User "puc-devops" cannot list resource "pods" in API group "" in the namespace "default"
       ```

    2. Permissões para todo o cluster exigem Roles e Bindings globais chamadas ClusterRoles e ClusterRolebindings. Aplique os manifestos de cluster presente no lab

       ```bash
       kubectl apply -f lab9/cluster-role-pod-reader.yaml
       kubectl apply -f lab9/cluster-rolebinding-pod-reader-user-puc-devops.yaml
       ```

       Agora as operações com esse usuário tem permissão de ler pods em todo o cluster

       ```bash
       kubectl --context k3d-lab-puc-devops get pod
       ```

       ```output
       NAME    READY   STATUS    RESTARTS   AGE
       sleep   1/1     Running   0          26s
       ```

       ```bash
       kubectl --context k3d-lab-puc-devops get pod -A
       ```

       ```output
       NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
       default       sleep                                     1/1     Running   0          51s
       kube-system   coredns-6799fbcd5-xxxxx                   1/1     Running   0          5m
       kube-system   local-path-provisioner-6c86858495-xxxxx   1/1     Running   0          5m
       kube-system   metrics-server-54fd9b65b-xxxxx            1/1     Running   0          5m
       ```

4. Vamos utilizar permissões de grupo

   1. A permissão do usuário puc-devops se limita a ler os pods. Vamos tentar ler outro recurso

       ```bash
       kubectl --context k3d-lab-puc-devops get svc -A
       ```

       ```output
       Error from server (Forbidden): services is forbidden: User "puc-devops" cannot list resource "services" in API group "" at the cluster scope
       ```

   2. Vamos dar permissao para o Grupo `devs` ler serviços em todas as namespaces

       ```bash
       # Criando a ClusterRole
       kubectl apply -f lab9/cluster-role-svc-reader.yaml
       
       #Criando a ClusterRoleBinding
       kubectl apply -f lab9/cluster-rolebinding-svc-reader-group-devs.yaml
       ```

   3. Agora todos os usuários do grupo Devs, incluíndo o devops-puc, podem ver todos os serviços do cluster

       ```bash
       kubectl --context k3d-lab-puc-devops get svc -A
       ```

       ```output
       NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
       default       kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  131d
       kube-system   kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   131d
       ```

5. Limpeza:

    ```bash
    kubectl delete -f lab9/cluster-rolebinding-svc-reader-group-devs.yaml
    kubectl delete -f lab9/cluster-role-svc-reader.yaml
    kubectl delete -f lab9/cluster-rolebinding-pod-reader-user-puc-devops.yaml
    kubectl delete -f lab9/cluster-role-pod-reader.yaml
    kubectl delete -f lab9/rolebinding-pod-reader-user-puc-devops.yaml
    kubectl delete -f lab9/role-pod-reader.yaml
    kubectl delete csr puc-devops
    kubectl config delete-context k3d-lab-puc-devops
    kubectl config unset users.puc-devops
    rm -f puc-devops.pem puc-devops.csr puc-devops.crt
    ```

---

## LAB 10

<!--console:k8s-->

### Objetivo: Instalando o Istio no k3d e observando a injeção de sidecar. Instalar o Istio em modo sidecar, subir o Bookinfo e confirmar visualmente a diferença entre um pod com e sem sidecar.

1. Suba o cluster k3d deste grupo de labs (10 a 14 usam um cluster próprio, separado do que os Labs 7-9 usam — o Istio tem requisitos de porta e recursos específicos), e baixe o `istioctl`

    ```bash
    # Cria o cluster k3d com uma porta exposta para o ingress gateway do Istio.
    # --disable=traefik: o k3s vem com o Traefik habilitado por padrão como
    # ingress controller; sem desabilitá-lo, ele ocupa a porta 80 do load
    # balancer antes do Istio conseguir, e o ingress gateway do Istio nunca
    # fica acessível.
    k3d cluster create istio-lab --k3s-arg "--disable=traefik@server:*" --api-port 6550 -p "8080:80@loadbalancer" --agents 2

    kubectl get nodes

    curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.30.2 sh -
    export PATH="$PWD/istio-1.30.2/bin:$PATH"
    istioctl version --remote=false
    ```

    ```output
    NAME                       STATUS   ROLES                  AGE   VERSION
    k3d-istio-lab-server-0     Ready    control-plane,master   30s   v1.30.x+k3s1
    k3d-istio-lab-agent-0      Ready    <none>                 25s   v1.30.x+k3s1
    k3d-istio-lab-agent-1      Ready    <none>                 25s   v1.30.x+k3s1
    ```

    Todos os labs a seguir usam o [Bookinfo](https://istio.io/latest/docs/examples/bookinfo/), a aplicação de exemplo oficial do próprio projeto Istio (`reviews-v1`, `reviews-v2`, `reviews-v3`).

2. Instale o Istio com o profile `demo` (inclui ingress gateway, adequado para lab)

    ```bash
    istioctl install --set profile=demo -y

    # Confirme os componentes instalados
    kubectl get pod -n istio-system
    ```

    ```output
    NAME                                    READY   STATUS    RESTARTS   AGE
    istio-egressgateway-6d8f9c9b7-abcde     1/1     Running   0          40s
    istio-ingressgateway-7f6b8d5c4-fghij     1/1     Running   0          40s
    istiod-5c7b9f8d6-klmno                  1/1     Running   0          55s
    ```

3. Antes de fazer o deploy, habilite a injeção automática de sidecar no namespace `default`

    ```bash
    kubectl label namespace default istio-injection=enabled

    kubectl get namespace -L istio-injection
    ```

4. Suba o Bookinfo

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/bookinfo/platform/kube/bookinfo.yaml

    kubectl get pod -w
    # Espere todos ficarem 2/2 Running, depois CTRL+C
    ```

    ```output
    NAME                              READY   STATUS    RESTARTS   AGE
    details-v1-...                    2/2     Running   0          25s
    productpage-v1-...                 2/2     Running   0          25s
    ratings-v1-...                     2/2     Running   0          25s
    reviews-v1-...                     2/2     Running   0          25s
    reviews-v2-...                     2/2     Running   0          25s
    reviews-v3-...                     2/2     Running   0          25s
    ```

    O `2/2` é o container da aplicação **e** o sidecar Envoy — o Modo Sidecar na prática.

5. Compare agora com um pod **sem** injeção

    ```bash
    kubectl create namespace sem-mesh

    kubectl run debug-pod --image=curlimages/curl -n sem-mesh -- sleep 3600

    kubectl get pod -n sem-mesh
    ```

    ```output
    NAME         READY   STATUS    RESTARTS   AGE
    debug-pod    1/1     Running   0          10s
    ```

    `1/1` em vez de `2/2` porque o namespace `sem-mesh` não tem o label `istio-injection=enabled` — o sintoma clássico de sidecar não injetado.

6. Exponha o Bookinfo via ingress gateway e confirme o acesso

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/bookinfo/networking/bookinfo-gateway.yaml

    curl -s http://localhost:8080/productpage | grep -o "<title>.*</title>"
    ```

    ```output
    <title>Simple Bookstore App</title>
    ```

7. Limpeza (opcional — os próximos labs reaproveitam esse Bookinfo)

    ```bash
    kubectl delete namespace sem-mesh
    ```

---

## LAB 11

<!--console:k8s-->
<!--continua:unidade5-lab10-->

### Objetivo: Traffic Management — canary com VirtualService e DestinationRule. Aplicar `DestinationRule` (define os subsets v1/v2/v3 por label) e `VirtualService` (decide o peso), e observar o split de tráfego 90/10 de verdade.

> Pré-requisito: Lab 10 concluído (Bookinfo no ar).

1. Aplique o `DestinationRule` que define os subsets

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/bookinfo/networking/destination-rule-all.yaml

    kubectl get destinationrule reviews -o yaml | grep -A2 "name: v"
    ```

    ```output
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
    ```

2. Sem `VirtualService`, o Istio faz round-robin entre as 3 versões. Confirme isso gerando algumas chamadas

    ```bash
    for i in $(seq 1 6); do
      curl -s http://localhost:8080/productpage | grep -o "reviews-v[0-9]" || true
      sleep 1
    done
    ```

    Você deve ver uma distribuição praticamente igual entre v1/v2/v3.

3. Aplique o `VirtualService` com split 90/10 entre v1 e v3

    ```bash
    kubectl apply -f lab11/virtualservice-canary.yaml
    ```

    ```yaml
    # lab11/virtualservice-canary.yaml
    apiVersion: networking.istio.io/v1
    kind: VirtualService
    metadata:
      name: reviews
    spec:
      hosts:
        - reviews
      http:
        - route:
            - destination:
                host: reviews
                subset: v1
              weight: 90               # <- 90% do tráfego
            - destination:
                host: reviews
                subset: v3
              weight: 10               # <- 10% do tráfego
    ```

4. Gere um volume maior de chamadas e conte a distribuição real

    > Por que um pod auxiliar em vez de rodar `curl` dentro do `istio-proxy`: o
    > Envoy exclui o próprio tráfego da interceptação de iptables (pra evitar
    > loop). Rodar `curl` de dentro do container `istio-proxy` faz a chamada
    > sair sem passar pela lógica do próprio Envoy — vira round-robin plano do
    > Kubernetes Service, ignorando o `VirtualService`. Por isso usamos um pod
    > auxiliar com sidecar próprio, cujo tráfego é interceptado normalmente.

    ```bash
    # Sobe um pod auxiliar com sidecar (namespace default já tem istio-injection=enabled)
    kubectl run meshclient --image=curlimages/curl -n default -- sleep 3600
    kubectl get pod meshclient
    # Espere ficar 2/2 (sidecar injetado) antes de continuar
    ```

    > Contamos pelo campo `podname` da resposta, não pela cor das estrelas: a
    > v1 não tem sistema de rating e por isso **não retorna o campo `color`
    > de jeito nenhum** (v2 retorna `"color": "black"`, v3 retorna
    > `"color": "red"`) — contar cor não distingue v1 de v3.

    ```bash
    for i in $(seq 1 50); do
      kubectl exec meshclient -c meshclient -- curl -s http://reviews:9080/reviews/0
    done | grep -o '"podname": "reviews-v[0-9]' | sort | uniq -c
    ```

    ```output
     45 "podname": "reviews-v1
      5 "podname": "reviews-v3
    ```

    Confirme que a proporção fica perto de 90/10 — não exatamente 45/5 numa amostra pequena, mas convergindo com mais chamadas.

5. Limpeza

    ```bash
    kubectl delete -f lab11/virtualservice-canary.yaml
    kubectl delete destinationrule reviews
    kubectl delete pod meshclient
    ```

---

## LAB 12

<!--console:k8s-->
<!--continua:unidade5-lab10-->

### Objetivo: Segurança — mTLS (PeerAuthentication) e AuthorizationPolicy. Aplicar `PeerAuthentication` em modo `STRICT` e uma `AuthorizationPolicy` restringindo quem pode chamar o serviço `reviews`.

> Pré-requisito: Lab 10 concluído (Bookinfo no ar).

1. Confirme que, por padrão, qualquer serviço do mesh consegue chamar `reviews`

    ```bash
    kubectl run sniffer --image=curlimages/curl -n default --restart=Never -- sleep 3600
    kubectl exec sniffer -- curl -s -o /dev/null -w "%{http_code}\n" http://reviews:9080/reviews/0
    ```

    ```output
    200
    ```

2. Aplique `PeerAuthentication` em modo `STRICT` para o namespace `default`

    ```bash
    kubectl apply -f lab12/peer-authentication-strict.yaml
    ```

    ```yaml
    # lab12/peer-authentication-strict.yaml
    apiVersion: security.istio.io/v1
    kind: PeerAuthentication
    metadata:
      name: default
      namespace: default
    spec:
      mtls:
        mode: STRICT                  # <- exige mTLS de quem chamar
    ```

    O pod `sniffer` não tem sidecar (foi criado antes da label de injeção ou sem reiniciar) — então ele passa a falhar:

    ```bash
    kubectl exec sniffer -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 3 http://reviews:9080/reviews/0
    ```

    ```output
    000
    ```

    Isso ilustra o "Never trust, always verify": sem certificado mTLS válido, a chamada nem se completa.

3. Recrie o `sniffer` agora COM sidecar, e confirme que ele volta a funcionar

    ```bash
    kubectl delete pod sniffer
    kubectl run sniffer --image=curlimages/curl -n default --restart=Never -- sleep 3600
    kubectl exec sniffer -c sniffer -- curl -s -o /dev/null -w "%{http_code}\n" http://reviews:9080/reviews/0
    ```

    ```output
    200
    ```

4. Agora restrinja por identidade: só o `productpage` pode chamar `reviews` — o `sniffer`, mesmo com sidecar e mTLS válido, deve ser barrado

    ```bash
    kubectl apply -f lab12/authorizationpolicy-reviews-viewer.yaml

    kubectl exec sniffer -c sniffer -- curl -s -o /dev/null -w "%{http_code}\n" http://reviews:9080/reviews/0
    ```

    ```yaml
    # lab12/authorizationpolicy-reviews-viewer.yaml
    apiVersion: security.istio.io/v1
    kind: AuthorizationPolicy
    metadata:
      name: reviews-viewer
      namespace: default
    spec:
      selector:
        matchLabels:
          app: reviews                # <- aplica-se aos pods do "reviews"
      action: ALLOW
      rules:
        - from:
            - source:
                # Identidade do ServiceAccount do productpage: única permitida
                principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
    ```

    ```output
    403
    ```

    `403`, não `000` — a diferença importante: mTLS (passo 2) barra por falta de identidade nenhuma; `AuthorizationPolicy` (esse passo) barra por identidade errada, mesmo com mTLS válido. Confirme que o `productpage` continua funcionando normalmente:

    ```bash
    curl -s http://localhost:8080/productpage | grep -o "<title>.*</title>"
    ```

5. Limpeza

    ```bash
    kubectl delete -f lab12/authorizationpolicy-reviews-viewer.yaml
    kubectl delete -f lab12/peer-authentication-strict.yaml
    kubectl delete pod sniffer
    ```

---

## LAB 13

<!--console:k8s-->
<!--continua:unidade5-lab10-->

### Objetivo: Resiliência — outlier detection. Fazer uma réplica de `reviews` responder mal de propósito — continuando `Ready` o tempo todo — e observar os três momentos: saudável → ejetado → de volta ao pool.

> Pré-requisito: Lab 10 concluído (Bookinfo no ar).

Você já usou o `DestinationRule` no Lab 11 pra definir `subsets` por versão. Ele volta aqui, mas pra outro propósito: em vez de `subsets`, o campo que importa agora é `outlierDetection` — a parte do `DestinationRule` que ejeta um host que continua `Ready` no Kubernetes mas está respondendo mal. Pra provocar isso sem quebrar de verdade nenhum pod do Bookinfo, no lugar da v2 real vamos subir o [`chaos-http`](https://github.com/fams/chaos-http): um servidor HTTP minúsculo, feito pra isso, cujo código de resposta é controlável via `curl` (`GET /control?httpCode=500`) sem nunca deixar de responder `/health`/`/ready`.

1. Tire a v2 real de cena e suba o `chaos-http` no lugar dela, e aplique o `DestinationRule` com `outlierDetection`

    ```bash
    kubectl scale deployment reviews-v2 --replicas=0

    kubectl apply -f lab13/chaos-v2.yaml
    kubectl apply -f lab13/destinationrule-outlier.yaml
    ```

    ```yaml
    # lab13/chaos-v2.yaml (resumido — veja o arquivo completo pros comentários)
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: chaos-v2
    spec:
      replicas: 1
      selector:
        matchLabels: {app: reviews, chaos-control: chaos-v2}
      template:
        metadata:
          labels: {app: reviews, chaos-control: chaos-v2}   # <- app: reviews entra no pool
        spec:
          containers:
            - name: chaos-v2
              image: fams/chaos-http:1.1.0
              env: [{name: PORT, value: "9080"}]
              ports: [{containerPort: 9080}]
              readinessProbe: {httpGet: {path: /ready, port: 9080}, periodSeconds: 3}
              livenessProbe: {httpGet: {path: /health, port: 9080}, periodSeconds: 5}
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: chaos-v2-control        # <- Service separado, só pra controlar o chaos-v2
    spec:
      selector: {chaos-control: chaos-v2}
      ports: [{port: 9080, targetPort: 9080}]
    ```

    ```yaml
    # lab13/destinationrule-outlier.yaml
    apiVersion: networking.istio.io/v1
    kind: DestinationRule
    metadata:
      name: reviews
    spec:
      host: reviews
      trafficPolicy:
        outlierDetection:
          consecutive5xxErrors: 3      # <- ejeta depois de 3 erros seguidos
          interval: 10s                # <- reavalia a cada 10s
          baseEjectionTime: 30s        # <- fica de fora por 30s
          maxEjectionPercent: 50       # <- nunca ejeta mais que 50% do pool
    ```

    O `chaos-v2` leva o rótulo `app: reviews` — o mesmo que `reviews-v1` e `reviews-v3` — e é exatamente esse rótulo que o `Service reviews` usa como seletor (sem filtrar por versão). Por isso o `chaos-v2` cai automaticamente nos `Endpoints` do `reviews`, junto com v1 e v3: sem `VirtualService`, o tráfego já é round-robin entre eles (como no Lab 10), e é um host individual desse pool — não uma versão inteira — que o `outlierDetection` ejeta.

    Assim como no Lab 11, as chamadas a `reviews` são feitas de um pod auxiliar com sidecar próprio (`meshclient`), não de dentro do `istio-proxy` — veja lá o porquê.

    ```bash
    kubectl run meshclient --image=curlimages/curl -n default -- sleep 3600
    kubectl get pod meshclient chaos-v2
    # Espere os dois ficarem 2/2 (sidecar injetado) antes de continuar
    ```

2. **Momento 1 — tudo saudável.** Confirme que as 3 versões respondem normalmente

    ```bash
    for i in $(seq 1 10); do
      kubectl exec meshclient -c meshclient -- curl -s -o /dev/null -w "%{http_code} " http://reviews:9080/reviews/0
    done; echo
    ```

    ```output
    200 200 200 200 200 200 200 200 200 200
    ```

3. **Momento 2 — provocando falha.** Mande o `chaos-v2` responder 500, via API — repare que ele continua `Ready` o tempo todo

    ```bash
    kubectl exec meshclient -c meshclient -- curl -s "http://chaos-v2-control:9080/control?httpCode=500"
    kubectl get pod chaos-v2
    ```

    ```bash
    # Gere chamadas — as primeiras tentativas pra chaos-v2 vão falhar (500),
    # até o outlier detection ejetar completamente esse host do pool. A
    # ejeção só é reavaliada a cada "interval" (10s), então repita algumas
    # rodadas em vez de olhar só a primeira leva de chamadas.
    for r in 1 2 3; do
      for i in $(seq 1 10); do
        kubectl exec meshclient -c meshclient -- curl -s -o /dev/null -w "%{http_code} " --max-time 2 http://reviews:9080/reviews/0
      done
      echo
      sleep 10
    done
    ```

    Nas primeiras chamadas você deve ver algum `500` — depois de `consecutive5xxErrors: 3`, o Envoy para de rotear pro `chaos-v2` e as chamadas seguintes vão só pras réplicas saudáveis, voltando a `200` consistente. Confirme a ejeção como dado real do Envoy, não estimado pela proporção de respostas:

    ```bash
    kubectl exec meshclient -c istio-proxy -- \
      curl -s http://localhost:15000/clusters | grep "outbound|9080||reviews" | grep health_flags
    ```

    ```output
    outbound|9080||reviews.default.svc.cluster.local::10.42.0.4:9080::health_flags::healthy
    outbound|9080||reviews.default.svc.cluster.local::10.42.0.5:9080::health_flags::healthy
    outbound|9080||reviews.default.svc.cluster.local::10.42.1.9:9080::health_flags::/failed_outlier_check
    ```

4. **Momento 3 — recuperação.** Conserte o `chaos-v2` via API e confirme que ele volta ao pool depois do `baseEjectionTime`

    ```bash
    kubectl exec meshclient -c meshclient -- curl -s "http://chaos-v2-control:9080/control?httpCode=200"

    # Espere passar o baseEjectionTime (30s) e gere tráfego de novo
    sleep 35
    for i in $(seq 1 10); do
      kubectl exec meshclient -c meshclient -- curl -s -o /dev/null -w "%{http_code} " http://reviews:9080/reviews/0
    done; echo
    ```

    ```output
    200 200 200 200 200 200 200 200 200 200
    ```

    Confirme no Envoy que nenhum host aparece mais como ejetado:

    ```bash
    kubectl exec meshclient -c istio-proxy -- \
      curl -s http://localhost:15000/clusters | grep "outbound|9080||reviews" | grep health_flags
    ```

5. Limpeza

    ```bash
    kubectl delete pod meshclient
    kubectl delete -f lab13/chaos-v2.yaml
    kubectl scale deployment reviews-v2 --replicas=1
    kubectl delete -f lab13/destinationrule-outlier.yaml
    ```

---

## LAB 14

<!--console:k8s-->
<!--continua:unidade5-lab10-->

### Objetivo: Troubleshooting — analyze, proxy-status e proxy-config. Introduzir dois problemas de propósito e usar `istioctl analyze`, `proxy-status` e `proxy-config` para encontrá-los, sem olhar a resposta antes.

> Pré-requisito: Lab 10 concluído (Bookinfo no ar).

1. **Bug 1 — VirtualService com host inexistente.** Aplique este manifesto quebrado de propósito

    ```bash
    kubectl apply -f lab14/virtualservice-quebrado.yaml
    ```

    ```yaml
    # lab14/virtualservice-quebrado.yaml
    apiVersion: networking.istio.io/v1
    kind: VirtualService
    metadata:
      name: reviews-quebrado
    spec:
      hosts:
        - reviews
      http:
        - route:
            - destination:
                host: reviews
                subset: v9             # <- subset inexistente, de propósito
    ```

    Use `istioctl analyze` para encontrar o problema antes de qualquer chamada falhar

    ```bash
    istioctl analyze
    ```

    ```output
    Error [IST0101] (VirtualService default/reviews-quebrado) Referenced host+subset in destinationrule not found: "reviews+v9"
    Error: Analyzers found issues when analyzing namespace: default.
    ```

    Corrija removendo o manifesto quebrado:

    ```bash
    kubectl delete -f lab14/virtualservice-quebrado.yaml
    ```

2. **Bug 2 — sidecar não injetado.** Suba um novo deployment de `reviews` num namespace sem a label de injeção

    ```bash
    kubectl create namespace reviews-v4-ns
    kubectl apply -n reviews-v4-ns -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/bookinfo/platform/kube/bookinfo-versions.yaml 2>/dev/null || true

    kubectl get pod -n reviews-v4-ns
    ```

    Repare no `READY` — se aparecer `1/1` em vez de `2/2`, use `proxy-status` para confirmar que esse pod nem aparece na lista de proxies sincronizados (porque não tem sidecar):

    ```bash
    istioctl proxy-status | grep reviews
    ```

    Corrija habilitando a injeção e reiniciando os pods:

    ```bash
    kubectl label namespace reviews-v4-ns istio-injection=enabled
    kubectl rollout restart deployment -n reviews-v4-ns
    kubectl get pod -n reviews-v4-ns -w
    # Espere ficar 2/2, depois CTRL+C
    ```

3. Para fechar, inspecione a config efetiva do Envoy do `productpage` e confirme que ele enxerga o cluster `reviews` corretamente

    ```bash
    istioctl proxy-config cluster deploy/productpage-v1 | grep reviews
    ```

4. Limpeza geral de todos os labs de Istio

    ```bash
    kubectl delete namespace reviews-v4-ns
    k3d cluster delete istio-lab
    ```
