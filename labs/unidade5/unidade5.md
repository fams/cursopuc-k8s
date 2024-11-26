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
