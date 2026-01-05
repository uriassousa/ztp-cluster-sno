# 🏗️ OpenShift Bare Metal Provisioning (ZTP via ACM)

Este repositório contém os manifestos necessários para o provisionamento Zero Touch Provisioning (ZTP) de um cluster OpenShift em hardware Bare Metal, gerenciado via Red Hat Advanced Cluster Management (ACM).

## 📂 Estrutura de Arquivos

A organização segue a lógica de aplicação sequencial via Kustomize ou ArgoCD.

| Arquivo | Descrição |
| :--- | :--- |
| `01-namespace.yaml` | **Criação do Namespace**: Onde os recursos do cluster residirão no Hub. |
| `02-cluster-deployment.yaml` | **Cluster Definition**: Define a "casca" do cluster no ACM/Hive. |
| `03-agent-cluster-install.yaml` | **Instalação**: Define a versão do OCP, CIDRs de rede e VIPs (API/Ingress). |
| `04-infra-env.yaml` | **Ambiente de Boot**: Gera a ISO de discovery e injeta a chave SSH pública. |
| `05-nmstate-configs.yaml` | **Network Config**: Definições de rede dos hosts (IPs estáticos, Bonds, VLANs). |
| `06-baremetal-hosts.yaml` | **Inventário**: Definição do hardware físico e credenciais de acesso ao BMC/IPMI. |

---

## ⚙️ Estratégia de Configuração (Kustomization)

Utilizamos um `kustomization.yaml` para orquestrar os arquivos acima. Abaixo estão os detalhes técnicos das decisões de arquitetura:

### A. Resources (Agregação)
Ao invés do ArgoCD monitorar 6 arquivos soltos, ele aponta apenas para o `kustomization.yaml`.
* **Benefício:** Mantém o controle de versão limpo. Se precisarmos adicionar um `07-extra-manifests.yaml`, basta incluí-lo na lista de *resources*, sem alterar a configuração da aplicação no ArgoCD.

### B. Namespace Enforcing (Segurança)
Embora os arquivos originais contenham `namespace: ocp-cli1`, reforçamos isso no nível do Kustomize.
* **O que ele faz:** Sobrescreve ou injeta o namespace em todos os objetos renderizados.
* **Vantagem:** Evita erros humanos. Se você copiar um `BareMetalHost` de outro projeto e esquecer de alterar o namespace, o Kustomize corrige automaticamente antes do apply, evitando que máquinas sejam provisionadas no tenant errado.

### C. Common Labels (Rastreabilidade)
Aplicamos `commonLabels` (ex: `cluster-name: ocp-cli1`) em todos os objetos.
* **No ACM:** Permite rodar comandos como:
    ```bash
    oc get all -n ocp-cli1 -l cluster-name=ocp-cli1
    ```
* **Governança:** O ACM utiliza essas labels para agrupar aplicações e aplicar políticas de conformidade específicas para este site.

---

## 🔐 Gestão de Secrets

> **⚠️ ATENÇÃO:** Este repositório não contém credenciais em texto plano.

Os arquivos de manifesto fazem referência aos seguintes secrets:
* `pull-secret`: Token da Red Hat (usado no ClusterDeployment e InfraEnv).
* `bmh-secret`: Credenciais de acesso ao BMC/IPMI (usado nos BareMetalHosts).

**Solução Adotada:**
Em um ambiente GitOps real, utilizamos **SealedSecrets (Bitnami)** ou **External Secrets Operator (Vault)** para injetar esses segredos no momento do deploy. Certifique-se de que esses objetos existam no namespace antes de iniciar a sincronização.