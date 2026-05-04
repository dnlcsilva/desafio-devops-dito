# Desafio DevOps - Dito

## Como navegar nesta entrega

Esta entrega está organizada conforme os itens solicitados no desafio:

| Item solicitado | Arquivo |
|---|---|
| (1) Diagrama de arquitetura comentado | `docs/01-arquitetura.md` |
| (2) Estrutura de repositórios e módulos Terraform | `docs/02-terraform.md` |
| (3) Estratégia de IAM para dev, CI/CD e operação | `docs/03-iam.md` |
| (4) README técnico com principais trade-offs | `README.md` |

## Resumo da proposta

A solução propõe uma arquitetura baseada em AWS e Amazon EKS, com ambientes separados de `staging` e `production`, promoção via GitOps com FluxCD, validação de imagens com scan, SBOM e assinatura com Cosign, políticas de segurança com Kyverno, secrets centralizados no AWS Secrets Manager e acesso controlado com IAM e IRSA.

O objetivo principal foi atender aos requisitos de alta disponibilidade, segurança, rastreabilidade e controle de mudanças, evitando acesso direto do time de desenvolvimento à produção.

## Objetivo

Esta arquitetura foi proposta para atender aos requisitos do desafio:

- API pública com alta disponibilidade em múltiplas zonas de disponibilidade;
- Ambientes separados de staging e production;
- Promoção entre ambientes usando GitOps com FluxCD;
- Time de desenvolvimento sem acesso direto à produção;
- Validação de imagens antes do deploy em produção;
- Gestão centralizada de secrets com rotação automatizada.

## Stack proposta

A solução utiliza AWS como cloud provider e Kubernetes gerenciado com Amazon EKS.

Principais componentes:

- Amazon Route 53 para DNS;
- AWS WAF para proteção da API pública;
- Application Load Balancer público para entrada HTTP/HTTPS;
- Amazon EKS em múltiplas Availability Zones;
- Nodes em subnets privadas;
- Amazon ECR para registry de imagens;
- FluxCD para GitOps;
- Terraform para infraestrutura como código;
- AWS Secrets Manager para secrets centralizados;
- External Secrets Operator para sincronizar secrets no Kubernetes;
- IAM Roles for Service Accounts (IRSA) para permissões granulares;
- Cosign/Sigstore para assinatura de imagens;
- Kyverno para validar políticas de segurança no cluster;
- Prometheus, Grafana e CloudWatch para observabilidade.

## Visão geral da arquitetura

O tráfego externo entra pelo Route 53, passa pelo AWS WAF e chega ao Application Load Balancer. O ALB encaminha as requisições para o Ingress Controller no EKS, que direciona o tráfego para os Services e Pods da aplicação.

Os workloads rodam em subnets privadas, distribuídos em múltiplas zonas de disponibilidade. O acesso a secrets é feito via AWS Secrets Manager, integrado ao cluster através do External Secrets Operator e IRSA.

O deploy é feito via GitOps usando FluxCD. A pipeline de CI gera a imagem, executa validações de segurança, envia a imagem para o ECR e assina o artefato. O FluxCD sincroniza o estado desejado a partir do repositório GitOps e aplica as mudanças no cluster.

## Separação de ambientes

A arquitetura considera separação forte entre staging e production.

Recomendação:

- Uma conta AWS para staging;
- Uma conta AWS para production;
- Uma conta compartilhada para componentes comuns, como ECR, observabilidade centralizada e DNS, se necessário.

Essa separação reduz o blast radius e evita que permissões ou falhas em staging afetem produção.


## Decisão principal

A principal decisão arquitetural foi usar GitOps com FluxCD como mecanismo de entrega contínua. Dessa forma, a produção não depende de acesso direto do time de desenvolvimento nem de credenciais privilegiadas na pipeline.

O Git passa a ser a fonte da verdade. Toda mudança em produção ocorre via Pull Request, revisão e reconciliação automática pelo FluxCD.

---

# Principais decisões arquiteturais

## 1. Uso de Kubernetes (EKS)

### Decisão

Utilizar Amazon EKS como plataforma de execução da API.

### Justificativa

- suporte nativo a alta disponibilidade multi-AZ;
- padronização de deploy com containers;
- integração com GitOps;
- suporte a políticas de segurança (Kyverno);
- escalabilidade com HPA e autoscaling de nodes.

### Trade-offs

**Vantagens:**

- alta flexibilidade e portabilidade;
- forte ecossistema;
- controle granular de deploys;
- integração com ferramentas modernas (Flux, External Secrets, etc).

**Desvantagens:**

- maior complexidade operacional;
- curva de aprendizado;
- necessidade de observabilidade mais robusta;
- custo potencial maior comparado a soluções mais simples (ex: ECS Fargate).

---

## 2. Separação de ambientes (staging e production)

### Decisão

Separar staging e production em ambientes isolados, preferencialmente em contas AWS distintas.

### Justificativa

- reduz blast radius;
- evita impacto de testes em produção;
- separa permissões e responsabilidades;
- melhora auditoria e governança.

### Trade-offs

**Vantagens:**

- maior segurança;
- melhor controle de acesso;
- isolamento de falhas;
- facilidade de auditoria.

**Desvantagens:**

- aumento de custo;
- maior esforço de gestão;
- duplicação de infraestrutura;
- necessidade de sincronização entre ambientes.

---

## 3. GitOps com FluxCD

### Decisão

A Dito já usa, por isso escolhi o FluxCD e não o ArgoCD.
Utilizar FluxCD para implementar GitOps, tornando o Git a fonte de verdade do estado do cluster.

### Justificativa

- elimina necessidade de acesso direto à produção;
- promove mudanças via Pull Request;
- garante rastreabilidade;
- permite rollback simples via Git;
- desacopla pipeline de deploy.

### Trade-offs

**Vantagens:**

- alto nível de controle e auditoria;
- segurança (sem acesso direto ao cluster);
- consistência entre ambientes;
- facilidade de rollback.

**Desvantagens:**

- maior complexidade inicial;
- necessidade de mudança cultural no time;
- troubleshooting pode ser mais indireto;
- dependência do fluxo Git.

---

## 4. Pipeline sem acesso direto à produção

### Decisão

A pipeline CI/CD não realiza deploy direto em produção.

Ela apenas:

- builda a imagem;
- executa testes e scans;
- assina o artefato;
- atualiza o repositório GitOps.

### Justificativa

- reduz risco de credenciais comprometidas;
- evita deploys diretos sem revisão;
- força uso de Pull Request;
- garante rastreabilidade.

### Trade-offs

**Vantagens:**

- segurança elevada;
- controle de mudanças;
- compliance;
- menor risco operacional.

**Desvantagens:**

- aumento no tempo de deploy;
- necessidade de aprovação manual;
- fluxo mais rígido;
- menor agilidade em cenários emergenciais.

---

## 5. Supply Chain Security (Cosign + Kyverno)

### Decisão

Implementar validação da cadeia de software com:

- scan de imagem (Trivy/Grype);
- geração de SBOM;
- assinatura de imagem com Cosign;
- validação no cluster com Kyverno.

### Justificativa

- garante integridade da imagem;
- evita execução de código não confiável;
- atende boas práticas de segurança moderna;
- protege contra supply chain attacks.

### Trade-offs

**Vantagens:**

- segurança forte;
- rastreabilidade de artefatos;
- compliance com boas práticas;
- redução de risco de imagens maliciosas.

**Desvantagens:**

- aumento de complexidade;
- necessidade de gestão de chaves;
- impacto no tempo da pipeline;
- necessidade de manutenção de políticas.

---

## 6. Secrets centralizados (Secrets Manager + External Secrets)

### Decisão

Centralizar secrets no AWS Secrets Manager e sincronizar com Kubernetes via External Secrets Operator.

### Justificativa

- evita secrets no Git;
- permite rotação automatizada;
- integra com IAM e KMS;
- reduz exposição de credenciais.

### Trade-offs

**Vantagens:**

- maior segurança;
- centralização;
- rotação automatizada;
- melhor auditoria.

**Desvantagens:**

- dependência de serviço externo;
- maior complexidade;
- custo adicional;
- necessidade de integração com Kubernetes.

---

## 7. IAM com menor privilégio e IRSA

### Decisão

Aplicar princípio de menor privilégio com separação clara de papéis:

- dev;
- CI/CD;
- SRE/Operação;
- FluxCD;
- workloads (via IRSA).

### Justificativa

- reduz risco de acesso indevido;
- limita impacto de falhas;
- melhora auditoria;
- evita credenciais estáticas.

### Trade-offs

**Vantagens:**

- segurança elevada;
- controle granular;
- isolamento de permissões;
- menor blast radius.

**Desvantagens:**

- maior complexidade de configuração;
- necessidade de manutenção das roles;
- troubleshooting pode ser mais difícil;
- exige maturidade em IAM.

---

## 8. Infraestrutura como código com Terraform

### Decisão

Utilizar Terraform com módulos reutilizáveis e separação por ambiente.

### Justificativa

- padronização;
- versionamento;
- reprodutibilidade;
- controle de mudanças;
- automação de provisionamento.

### Trade-offs

**Vantagens:**

- consistência de ambientes;
- rastreabilidade;
- automação;
- reutilização de módulos.

**Desvantagens:**

- curva de aprendizado;
- necessidade de gestão de state;
- risco em mudanças mal planejadas;
- dependência de pipeline para execução.

---

## 9. Observabilidade desde o início

### Decisão

Implementar observabilidade com logs, métricas e alertas desde o início.

### Justificativa

- facilita operação;
- reduz MTTR;
- melhora visibilidade do sistema;
- permite abordagem SRE baseada em SLO/SLI.

### Trade-offs

**Vantagens:**

- melhor diagnóstico;
- operação mais eficiente;
- prevenção de incidentes;
- visibilidade completa.

**Desvantagens:**

- custo adicional;
- necessidade de tuning de alertas;
- risco de alert fatigue;
- necessidade de manutenção contínua.

---

# Decisão geral da arquitetura

A arquitetura prioriza:

- segurança;
- rastreabilidade;
- controle de mudanças;
- isolamento de ambientes;
- padronização;
- escalabilidade.

Essa abordagem é mais robusta do que soluções simplificadas, porém mais complexa.

---

# Trade-off global

## Abordagem adotada

Uma arquitetura mais madura, com:

- GitOps;
- validação de supply chain;
- IAM restritivo;
- ambientes isolados;
- observabilidade completa.

## Alternativa possível

Uma arquitetura mais simples poderia usar:

- ECS Fargate;
- deploy direto via pipeline;
- secrets em variáveis de ambiente;
- menos camadas de segurança.

## Comparação

| Critério | Arquitetura proposta | Arquitetura simplificada |
|--------|---------------------|--------------------------|
| Segurança | Alta | Média |
| Complexidade | Alta | Baixa |
| Governança | Alta | Baixa |
| Escalabilidade | Alta | Média |
| Tempo de implantação | Maior | Menor |
| Risco operacional | Baixo | Médio/Alto |

---

# Conclusão

A arquitetura proposta foi desenhada pensando em um cenário de produção real, onde segurança, controle e confiabilidade são essenciais.

Apesar da maior complexidade inicial, ela permite escalar o produto com segurança, mantendo governança e rastreabilidade desde o início.

---
