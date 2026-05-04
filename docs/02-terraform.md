# 02 - Estrutura de Repositórios e Módulos Terraform

## Visão resumida

A arquitetura proposta separa claramente responsabilidades entre aplicação, infraestrutura e estado desejado (GitOps).

A infraestrutura é provisionada com Terraform modularizado, com ambientes isolados (staging e production), preferencialmente em contas AWS distintas.

O deploy segue o modelo GitOps com FluxCD, garantindo que a produção seja atualizada apenas via Pull Request e revisão.

A segurança da cadeia de software (supply chain) é garantida com scan de imagens, geração de SBOM, assinatura com Cosign e validação via Kyverno no cluster.

Secrets são centralizados no AWS Secrets Manager, com rotação automatizada e integração ao Kubernetes via External Secrets Operator e IRSA.

## Objetivo

Esta seção apresenta a estrutura proposta de repositórios e módulos Terraform para suportar a infraestrutura do novo produto.

A proposta considera os seguintes requisitos do desafio:

- API pública com alta disponibilidade multi-zone;
- Dois ambientes: staging e production;
- Promoção via GitOps;
- Time de desenvolvimento sem acesso direto à produção;
- Validação de imagens em produção;
- Secrets centralizados com rotação automatizada.

## Estratégia geral

A infraestrutura será gerenciada com Terraform, utilizando módulos reutilizáveis e separação clara entre ambientes.

A proposta separa responsabilidades em três repositórios principais:

```text
desafio-devops-dito/
├── app-api/
├── platform-infra/
└── platform-gitops/
```

## Repositório da aplicação

```text
app-api/
├── src/
├── tests/
├── Dockerfile
├── .dockerignore
├── README.md
└── .github/
    └── workflows/
        └── ci.yml
```

### Decisão

O repositório **app-api** contém apenas o código da aplicação, Dockerfile, testes e pipeline de CI.

Ele não deve conter manifests de produção diretamente, nem secrets, nem configuração sensível de infraestrutura.

### Responsabilidades

- armazenar o código da API;
- executar testes automatizados;
- construir a imagem Docker;
- executar scan de segurança;
- gerar SBOM;
- publicar imagem no Amazon ECR;
- assinar imagem com Cosign;
- atualizar o repositório GitOps de staging ou abrir Pull Request de promoção para production.

### Justificativa

Essa separação evita misturar código de aplicação com infraestrutura. Também deixa claro que a pipeline da aplicação gera artefatos, mas não aplica mudanças diretamente em produção.

## Repositório de infraestrutura

```text
platform-infra/
├── README.md
├── modules/
│   ├── vpc/
│   ├── eks/
│   ├── ecr/
│   ├── iam/
│   ├── kms/
│   ├── secrets-manager/
│   ├── observability/
│   ├── security/
│   ├── flux-bootstrap/
│   └── network/
│
└── envs/
    ├── staging/
    │   ├── backend.tf
    │   ├── providers.tf
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── terraform.tfvars
    │
    ├── production/
    │   ├── backend.tf
    │   ├── providers.tf
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── terraform.tfvars
    │
    └── shared/
        ├── backend.tf
        ├── providers.tf
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── terraform.tfvars
```

### Decisão

O repositório **platform-infra** concentra toda a infraestrutura provisionada via Terraform.

A separação por módulos permite reutilização, padronização e evolução controlada. A separação por ambientes permite que staging e production tenham configurações próprias, estados separados e permissões distintas.

## Ambientes

### A proposta considera três escopos

```text
shared     → recursos compartilhados
staging    → ambiente de homologação
production → ambiente produtivo
```

#### Shared

O ambiente **shared** pode conter recursos comuns, como:

- Amazon ECR;
- DNS compartilhado, se aplicável;
- observabilidade centralizada;
- buckets de logs;
- KMS keys compartilhadas, quando fizer sentido;
- roles de CI/CD.

#### Staging

O ambiente **staging** contém:

- VPC staging;
- EKS staging;
- ALB staging;
- Secrets Manager staging;
- FluxCD staging;
- políticas Kyverno staging;
- integração com observabilidade.

#### Production

O ambiente production contém:

- VPC production;
- EKS production;
- ALB production;
- Secrets Manager production;
- FluxCD production;
- políticas Kyverno production;
- integração com observabilidade;
- controles de segurança mais restritivos.

## Estados Terraform

### Backend remoto

Cada ambiente utiliza backend remoto em S3 com DynamoDB para lock.

Boas práticas aplicadas:

- versionamento habilitado no bucket;
- criptografia com KMS;
- acesso restrito via IAM;
- separação de state por ambiente;
- bloqueio de deleção acidental;
- utilização de locking com DynamoDB para evitar concorrência;
- separação de roles de execução do Terraform por ambiente;

Exemplo:

```text
terraform-state-shared
terraform-state-staging
terraform-state-production
```

Recomendação:

- S3 para armazenamento do state;
- DynamoDB para lock;
- versionamento habilitado no bucket;
- criptografia com KMS;
- acesso restrito por IAM.

### Justificativa

Separar os states evita que uma alteração em staging afete production. Também facilita controle de acesso e auditoria por ambiente.

## Módulos Terraform propostos

### Módulo ```vpc```

```text
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- criar VPC;
- criar subnets públicas e privadas em múltiplas AZs;
- criar Internet Gateway;
- criar NAT Gateway;
- criar route tables;
- criar tags necessárias para integração com EKS e Load Balancer Controller.
- subnets privadas distribuídas em pelo menos 2 ou 3 Availability Zones;

#### Decisão

A VPC deve ser multi-AZ para garantir alta disponibilidade.

Os Load Balancers ficam em subnets públicas. Os nodes e pods do EKS ficam em subnets privadas.

#### Justificativa

Essa decisão reduz exposição direta dos workloads e mantém apenas a camada de entrada pública.

### Módulo ```eks```

```text
modules/eks/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- criar cluster EKS;
- criar node groups gerenciados;
- habilitar OIDC provider;
- configurar logs do control plane;
- instalar add-ons essenciais;
- preparar integração com IRSA;
- expor outputs para FluxCD, IAM e observabilidade.
- node groups distribuídos entre múltiplas AZs;

#### Add-ons recomendados

- VPC CNI;
- CoreDNS;
- kube-proxy;
- EBS CSI Driver;
- AWS Load Balancer Controller;
- Metrics Server;
- External Secrets Operator;
- Kyverno;
- FluxCD.

#### Decisão

O EKS será usado como plataforma principal para execução da API.

O cluster deve rodar em múltiplas zonas de disponibilidade e utilizar nodes em subnets privadas.

#### Justificativa

EKS reduz a complexidade operacional do control plane e permite padronizar deploys via Kubernetes, GitOps e políticas de segurança.

### Módulo ```ecr```

```text
modules/ecr/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- criar repositórios ECR;
- habilitar scan on push;
- configurar lifecycle policy;
- configurar imutabilidade de tags, se aplicável;
- configurar permissões cross-account;
- permitir push pela pipeline;
- permitir pull pelos clusters staging e production.

#### Decisão

As imagens serão armazenadas no Amazon ECR.

A mesma imagem validada em staging deve ser promovida para production, preferencialmente por digest.

#### Justificativa

Promover a mesma imagem reduz o risco de diferença entre ambientes. A imagem não deve ser reconstruída para produção.

### Módulo ```iam```

```text
modules/iam/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- criar roles de CI/CD;
- criar roles para FluxCD;
- criar roles para operação/SRE;
- criar políticas de menor privilégio;
- criar permissões para IRSA;
- configurar acesso cross-account quando necessário.

#### Decisão

A estratégia de IAM será baseada em menor privilégio, separação por papel e autenticação sem credenciais estáticas sempre que possível.

#### Justificativa

Essa separação reduz riscos e facilita auditoria. Cada papel recebe apenas o conjunto mínimo de permissões necessário.

### Módulo ```kms```

```text
modules/kms/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- criar KMS keys por ambiente;
- criptografar secrets;
- criptografar logs;
- criptografar buckets de state;
- criptografar recursos sensíveis.

#### Decisão

Cada ambiente deve possuir sua própria chave KMS.

#### Justificativa

Chaves separadas por ambiente reduzem o impacto em caso de comprometimento e permitem políticas de acesso mais granulares.

## Módulo ```secrets-manager```

```text
modules/secrets-manager/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- criar secrets no AWS Secrets Manager;
- configurar rotação automatizada;
- configurar política de acesso;
- integrar com External Secrets Operator;
- usar KMS para criptografia.

#### Decisão

Secrets não devem ser armazenados no Git, nem como variável fixa em pipeline.

O AWS Secrets Manager será a fonte central de secrets.

#### Justificativa

Centralizar secrets melhora segurança, auditoria e rotação. O Kubernetes recebe os valores necessários via External Secrets Operator.

### Módulo ```observability```

```text
modules/observability/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- configurar CloudWatch Logs;
- configurar Prometheus;
- configurar Grafana;
- configurar dashboards iniciais;
- configurar alertas;
- integrar logs dos workloads;
- coletar métricas do cluster e da aplicação.

#### Decisão

A observabilidade deve ser incluída desde o início da plataforma.

#### Métricas importantes

- disponibilidade;
- latência;
- taxa de erro;
- saturação;
- uso de CPU e memória;
- reinícios de pods;
- erros 4xx e 5xx;
- tempo de resposta da API;
- status dos deploys via FluxCD.

#### Justificativa

A operação da API deve ser orientada por sinais de confiabilidade, não apenas por métricas básicas de infraestrutura.

### Módulo ```security```

```text
modules/security/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- configurar AWS WAF;
- configurar Security Groups;
- configurar políticas Kyverno;
- configurar Network Policies;
- configurar baseline de segurança do cluster;
- bloquear configurações inseguras.

#### Políticas recomendadas

- bloquear uso de tag latest;
- exigir imagens vindas do ECR autorizado;
- exigir assinatura de imagem;
- bloquear container privilegiado;
- bloquear execução como root;
- exigir requests e limits;
- exigir readiness e liveness probes;

#### Decisão

As políticas de segurança devem ser aplicadas no cluster, não apenas na pipeline.

#### Justificativa

Mesmo que a pipeline falhe ou alguém tente aplicar um manifesto manualmente, o cluster deve bloquear configurações fora do padrão.

### Módulo ```flux-bootstrap```

```text
modules/flux-bootstrap/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

#### Responsabilidades

- instalar FluxCD no cluster;
- configurar o repositório GitOps;
- configurar sources;
- configurar Kustomizations;
- separar staging e production;
- aplicar bootstrap inicial do cluster.

#### Decisão

FluxCD será o mecanismo de GitOps da plataforma.

#### Justificativa

O FluxCD mantém o cluster reconciliado com o estado desejado definido no Git. Isso garante rastreabilidade, revisão por Pull Request e menor necessidade de acesso direto à produção.

## Repositório GitOps

```text
platform-gitops/
├── README.md
├── clusters/
│   ├── staging/
│   │   ├── flux-system/
│   │   ├── apps/
│   │   ├── infrastructure/
│   │   └── policies/
│   │
│   └── production/
│       ├── flux-system/
│       ├── apps/
│       ├── infrastructure/
│       └── policies/
│
├── apps/
│   └── api/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── ingress.yaml
│       │   ├── externalsecret.yaml
│       │   └── kustomization.yaml
│       │
│       └── overlays/
│           ├── staging/
│           │   ├── kustomization.yaml
│           │   └── values.yaml
│           │
│           └── production/
│               ├── kustomization.yaml
│               └── values.yaml
│
└── policies/
    ├── kyverno/
    │   ├── require-signed-images.yaml
    │   ├── disallow-latest-tag.yaml
    │   ├── require-requests-limits.yaml
    │   └── disallow-root-container.yaml
    │
    └── network-policies/
        ├── default-deny.yaml
        └── allow-api-ingress.yaml
```

### Decisão

O repositório platform-gitops representa o estado desejado dos clusters Kubernetes.

Staging e production possuem overlays separados, mas compartilham uma base comum.

### Justificativa

Esse modelo evita duplicação excessiva e permite que diferenças entre ambientes sejam explícitas.

#### Exemplo

- staging pode ter menos réplicas;
- production pode ter mais réplicas;
- production pode ter HPA mais restritivo;
- production pode exigir aprovação manual;
- production pode ter políticas mais rigorosas.

## Fluxo entre os repositórios

```text
app-api
  → gera imagem
  → envia imagem ao ECR
  → assina imagem
  → atualiza platform-gitops staging

platform-gitops staging
  → FluxCD aplica em staging

validação em staging
  → Pull Request para production

platform-gitops production
  → FluxCD aplica em production

platform-infra
  → cria VPC, EKS, ECR, IAM, Secrets, Flux e políticas base
```

### Decisão

A pipeline da aplicação não aplica diretamente recursos no cluster production.

Ela atua apenas até o ponto de gerar artefato confiável e propor mudança no GitOps.

### Justificativa

Isso atende ao requisito de que o time de desenvolvimento não deve ter acesso direto à produção.

## Estratégia de promoção entre ambientes

### Controle de promoção

A promoção entre ambientes utiliza a mesma imagem por digest, evitando divergência entre staging e production.

O controle é feito via Pull Request no repositório GitOps, garantindo:

- revisão por pares;
- rastreabilidade;
- rollback via revert de commit;

### Staging

O deploy em staging pode ser automático após a pipeline passar com sucesso.

#### Fluxo

```text
merge na branch principal
  → pipeline
  → build
  → scan
  → assinatura
  → atualização do overlay staging
  → FluxCD aplica staging
```

### Production

A promoção para production deve ser feita por Pull Request.

#### Fluxo

```text
imagem validada em staging
  → PR alterando overlay production
  → revisão/aprovação
  → merge
  → FluxCD aplica production
```

### Decisão

Production não deve reconstruir a imagem.

Production deve usar a mesma imagem validada em staging.

### Justificativa

Isso garante consistência entre ambientes e reduz risco de promover um artefato diferente daquele que foi testado.

## Trade-offs da abordagem

### Vantagens

- alta segurança e governança;
- rastreabilidade completa via Git;
- isolamento entre ambientes;
- redução de acesso direto à produção;
- padronização de infraestrutura com Terraform;
- validação forte da cadeia de software.

### Desvantagens

- maior complexidade inicial;
- mais componentes para operar (Flux, Kyverno, External Secrets);
- curva de aprendizado mais alta;
- necessidade de maturidade do time em GitOps.

### Decisão

Apesar da complexidade maior, a arquitetura é adequada para um produto crítico, pois prioriza segurança, confiabilidade e controle de mudanças.
