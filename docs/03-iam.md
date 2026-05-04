# 03 - Estratégia de IAM

## Objetivo

Esta seção descreve a estratégia de IAM para os diferentes papéis envolvidos na operação da plataforma:

- desenvolvimento;
- CI/CD;
- operação/SRE;
- FluxCD;
- workloads Kubernetes.

A estratégia segue os princípios de menor privilégio, separação de responsabilidades, auditoria e ausência de credenciais estáticas sempre que possível.

## Princípios adotados

- Menor privilégio;
- Separação entre staging e production;
- Sem acesso direto de desenvolvimento à produção;
- Autenticação federada sempre que possível;
- Sem credenciais fixas em pipeline;
- MFA para acessos humanos sensíveis;
- Acesso break-glass controlado;
- Auditoria via CloudTrail;
- Permissões específicas por ambiente;
- IRSA para workloads Kubernetes.

---

# Visão geral dos papéis

```text
Dev
  → acesso a staging, logs e métricas
  → sem acesso direto à produção

CI/CD
  → build, scan, push no ECR, assinatura e alteração no GitOps
  → sem permissão de admin no cluster production

Operação/SRE
  → acesso controlado à produção
  → permissões elevadas apenas quando necessário

FluxCD
  → reconcilia o estado desejado no cluster
  → aplica manifests autorizados pelo repositório GitOps

Workloads
  → acessam somente os recursos AWS necessários
  → permissões via IRSA
```

## Papel: Desenvolvimento

### Objetivo

Permitir que o time de desenvolvimento consiga validar aplicações, consultar logs e acompanhar métricas, sem alterar diretamente produção.

### Permissões em staging

O time de desenvolvimento pode ter acesso a:

- visualizar recursos Kubernetes;
- consultar logs da aplicação;
- consultar métricas e dashboards;
- executar testes em staging;
- abrir Pull Requests no repositório GitOps;
- acompanhar status de deploys via FluxCD.

### Permissões em production

Em production, o time de desenvolvimento não deve ter acesso direto de escrita.

#### Permissões recomendadas

- leitura limitada de logs;
- leitura limitada de métricas;
- leitura de dashboards;
- sem permissão para executar kubectl apply;
- sem permissão para alterar secrets;
- sem permissão para alterar workloads;
- sem acesso direto ao banco de dados;
- sem acesso administrativo ao cluster.

## Papel: CI/CD

### Objetivo

Permitir que a pipeline gere artefatos confiáveis, publique imagens e proponha mudanças no GitOps sem possuir acesso administrativo à produção.

### Autenticação

A pipeline deve usar autenticação federada via OIDC.

#### Exemplos

- GitHub Actions OIDC para assumir IAM Role na AWS;
- GitLab CI OIDC/JWT para assumir IAM Role na AWS.

Não devem ser usadas access keys fixas armazenadas como secrets da pipeline.

### Permissões necessárias

A role de CI/CD pode ter permissões para:

- autenticar no Amazon ECR;
- fazer push de imagens no ECR;
- executar scan de imagem;
- assinar imagem com Cosign;
- acessar chave KMS necessária para assinatura, se aplicável;
- gerar e publicar SBOM;
- atualizar manifests de staging no repositório GitOps;
- abrir Pull Request de promoção para production.

### Permissões que a CI/CD não deve ter

A role de CI/CD não deve ter:

- permissão de administrador na conta production;
- permissão ampla em IAM;
- permissão para alterar secrets diretamente;
- permissão para aplicar manifests diretamente no cluster production;
- permissão para executar comandos administrativos no EKS production;
- permissão para acessar banco de dados production.

## Papel: Operação / SRE

### Objetivo

Permitir que o time de operação mantenha a plataforma, responda incidentes e execute ações administrativas controladas.

### Permissões em staging

O time de operação pode ter permissões administrativas em staging para:

- validar mudanças de infraestrutura;
- testar políticas;
- validar upgrades;
- simular incidentes;
- ajustar componentes de plataforma.

### Permissões em production

Em production, o acesso deve ser controlado.

#### Permissões recomendadas

- acesso de leitura ao cluster;
- acesso a logs e métricas;
- acesso a eventos do Kubernetes;
- acesso a CloudWatch;
- acesso a CloudTrail;
- acesso a dashboards;
- permissão para operar FluxCD;
- permissão para pausar/reconciliar FluxCD, quando necessário;
- permissão para executar ações emergenciais via break-glass.

### Break-glass

O acesso break-glass deve ser usado apenas em situações excepcionais, como:

- incidente crítico;
- indisponibilidade de produção;
- falha no processo GitOps;
- necessidade de recuperação urgente.

#### Boas práticas para break-glass

- MFA obrigatório;
- permissão temporária;
- aprovação prévia ou posterior;
- sessão auditada;
- registro do motivo do acesso;
- revisão pós-incidente;
- remoção automática da permissão após o uso.

Operação/SRE possui acesso mais amplo que desenvolvimento, mas com controles fortes em production.

## Papel: FluxCD

### Objetivo

Permitir que o FluxCD aplique no cluster o estado desejado definido no repositório GitOps.

### Permissões no Kubernetes

O FluxCD precisa ter permissões para:

- ler fontes Git;
- aplicar manifests;
- reconciliar Kustomizations;
- reconciliar HelmReleases, se usado;
- gerenciar namespaces autorizados;
- aplicar políticas;
- atualizar recursos Kubernetes definidos no GitOps.

### Escopo recomendado

Separar FluxCD por ambiente:

```text
FluxCD Staging
  → observa paths de staging
  → aplica recursos no cluster staging

FluxCD Production
  → observa paths de production
  → aplica recursos no cluster production
```

## Papel: Workloads Kubernetes

### Objetivo

Permitir que aplicações dentro do EKS acessem recursos AWS necessários, sem usar credenciais fixas.

### Estratégia

Utilizar IAM Roles for Service Accounts, conhecido como IRSA.

Cada workload possui um Service Account próprio e uma IAM Role específica.

Exemplo:

```text
api-service-account
  → assume role api-production-role
  → pode ler somente secrets específicos da API
```

### Permissões comuns para a API

A aplicação pode precisar de permissões como:

- ler secrets específicos no AWS Secrets Manager;
- descriptografar usando KMS;
- publicar logs ou métricas, se necessário;
- acessar filas, buckets ou outros serviços AWS específicos do produto.

### Permissões que devem ser evitadas

A aplicação não deve receber:

- AdministratorAccess;
- acesso amplo a todos os secrets;
- acesso amplo a todos os buckets;
- permissão para alterar IAM;
- permissão para alterar infraestrutura;
- credenciais estáticas em variáveis de ambiente.

Cada aplicação recebe uma role mínima, vinculada ao seu Service Account no Kubernetes.
Se um pod for comprometido, o impacto fica limitado às permissões daquele workload.

## Exemplo de matriz de acesso

 Papel | Staging | Production | Observações |
|--------|---------------------|--------------------------|--------------------------|
| Dev | Leitura, logs, métricas e deploy via PR/GitOps | Somente leitura limitada | Sem acesso direto de escrita em produção |
| CI/CD | Push ECR, scan, assinatura e atualização GitOps | Abrir PR de promoção | Não aplica direto no cluster production |
| Operação/SRE | Admin controlado | Read-only + break-glass | Acesso elevado apenas em incidente |
| FluxCD | Aplica manifests de staging | Aplica manifests de production | Separado por ambiente |
| Workload API | Acesso mínimo a recursos staging | Acesso mínimo a recursos production | Via IRSA |


A estratégia garante que nenhum usuário ou sistema possua acesso direto irrestrito à produção, reduzindo significativamente o risco de erro humano ou comprometimento de credenciais.