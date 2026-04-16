# Laboratórios do curso K8S

Esses exercícios permitem visualizar os objetos funcionando no kubernetes. Foram pensados com o kubernetes instalado pelo k3d, requisito do curso

## Setup do cluster

Instale o [k3d](https://k3d.io) e crie o cluster:

```bash
k3d cluster create lab
```

O contexto do kubectl será configurado automaticamente como `k3d-lab`. Os labs de armazenamento usam `type: DirectoryOrCreate` e `nodeAffinity` para que os diretórios sejam criados automaticamente nos nós sem configuração adicional.

Recomenda-se criar um diretório por lab para que os arquivos criados possam ficar separados

Os fontes desses labs e também outros arquivos estarão no <https://github.com/fams/cursopuc-k8s>

- [Unidade4](labs/unidade4/unidade4.md) Labs de 1 a 6
- [Unidade5](labs/unidade5/unidade5.md) Labs de 7 a 
