# LABS

Esses exercícios permitem visualizar os objetos funcionando no kubernetes. Foram pensados com o kuberntes instalado pelo docker-desktop, requisito do curso

Recomenda-se criar um diretório por lab para que os arquivos criados possam ficar separados

Os fontes desses labs e também outros arquivos estarão no <https://github.com/fams/cursopuc-k8s>

## Lab 7

### Exercício: Criando um Persistent Volume estaticamente provisionado

#### Objetivo

Aprender a pré-provisionar volumes no kubernetes e passar pelas fases do gerenciamento de volumes

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

    3. Esse provisionamento utilizando diretamente o tipo de volume, no caso hostPaht, não é recomendado e só é possível que dois pods possam acessá-lo porque os dois pods estão no mesmo host. Alguns tipos de volumes permitem que mais de um host possam acessar o mesmo disco, mas a forma de provisionamento será outro.

2. Agora vamos fazer o provisionamento direto utilizando o recurso de persistentVolume

    1. Crie os discos pré-provisionados utilizando os manifestos do lab7

        ```bash
        kubectl apply -f lab7/pre-provisioned.yaml
        # Verifique a criação
        kubectl get pv
        ```

        Você vai ver algo parecido com isso:

        ```text
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

        ```text
        NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
        manual-pv-1g                               1Gi        RWO            Delete           Available                                                               <unset>                          161m
        manual-pv-2g                               2Gi        RWO            Delete           Bound       default/static-claim                                        <unset>                          161m
        ```

        Existindo Discos pré-provisionados com as mesmas características do PVC, o kubernetes irá ligar o PVC a ele.

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
        kubectl apply -f lab6/reader-pvc.yaml
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

## Lab 8

### Exercício: Provisionando volumes de forma dinâmica

#### Objetivo

Aprender a utilizar volumes provisionados dinâmicamente no kubernetes e passar pelas fases do gerenciamento de volumes.

##### Introdução

O Provisionamento dinâmico depende do StorageClass, uma espécie de profile de criação de Volumes para o cluster. O storageClass pré-existente no docker-desktop é o HostPath:

```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: hostpath
provisioner: docker.io/hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

O `provisioner` define qual módulo de provisonamento instalado no cluster será utilizado. Hoje em dia os povisionadores utilizam majoritariamente o CSI (Container Storage Interface) que podem ser instalados de terceiros
O `volumeBindingMode` informa se o `PV` deve ser criado ao se ligar ao `PVC` ou quando o `POD` tentar montá-lo.
O `reclaimPolicy` tem o mesmo papel que no `PV`
Exitem outros campos disponíveis, como paramêters que irá passar argumentos para o provisionador.

1. Vamos agora provisionar um `PV` utilizando `PVC` com StorageClass

    1. Aplique o manifesto do `PVC`:

        ```bash
        kubectl apply -f lab8/pvc-sc.yaml
        kubectl get pvc dynamic-claim
        ```

    2. Verifique a criaçao do `PV`:

        ```bash
        kubectl get pv
        ```

2. Podemos agora criar os `deployments` writer e reader utilizando esse `PVC`

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

3. Limpeza:

    ```bash
    kubectl delete -f lab8/writer-pvc.yaml
    kubectl delete -f lab8/reader-pvc.yaml
    kubectl delete -f lab8/pvc-dynamic.yaml
    ```

## LAB 9

### Exercício: RBAC

#### Objetivo

Compreeender o funcionamento do controle de acesso RBAC no kubernetes

##### Introdução

Controle de acesso pode ser dividido entree AuthN AuthZ, respectivamente autenticação e autorizaçao. Nesse lab faremos o foco no AuthZ, uma vez que existem diversas formas de autenticação no Kubernetes.
Além de um kubernetes configurado e accessível, será necessário o comando jq instalado.

1. Vamos criar um usuário `puc-devops` com autenticação por certificado.

   1. Criando o CSR para o usuário:

        ```bash
        # Criando uma chave privada RSA 
        openssl genrsa -out puc-devops.pem
        
        # Gerando um Certificate Signing Request 
        openssl req -new -key puc-devops.pem -out puc-devops.csr -subj "/CN=puc-devops"
        
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

        ```bash
        certificatesigningrequest.certificates.k8s.io/puc-devops created
        NAME         AGE   SIGNERNAME                            REQUESTOR            REQUESTEDDURATION   CONDITION
        puc-devops   0s    kubernetes.io/kube-apiserver-client   docker-for-desktop   24h                 Pending
        ```

   2. Obtento o certificado e configurando o usuário:

        ```bash
        # Aprove o certificado
        kubectl certificate approve puc-devops
        
        # Verifique que o certificado está aprovado e obtenha o PEM 
        kubectl get csr puc-devops
        ```

        ```bash
        NAME         AGE   SIGNERNAME                            REQUESTOR            REQUESTEDDURATION   CONDITION
        puc-devops   45s   kubernetes.io/kube-apiserver-client   docker-for-desktop   24h                 Approved,Issued        
        ```

        ```bash
        # Obter o certificado assinado pela CA do cluster, salvando-o em formatdo pem codificado base64.
        # PS: O formato pem já é codificado base64, porem com headers e trailers.
        # Para colocarmos no manifesto precisamos de uma string contínua)
        kubectl get csr puc-devops -o jsonpath="{.status.certificate}"|tee puc-devops.cert|base64 -d |openssl x509 -text -noout
        ```
