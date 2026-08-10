# ADR 001 — Rodar a aplicação em K3s numa VM da Magalu Cloud

**Status:** Aceito
**Data:** 2026-08-04

## Contexto
A API de pedidos é um container stateless que precisa rodar na Magalu Cloud com
mais de uma réplica atrás de um balanceador, receber deploys automatizados pelo
GitHub Actions e expor `/metrics` para o Prometheus. O projeto é acadêmico, com
teto de custo apertado e prazo curto — o ambiente precisava subir em minutos e
ser recriável quantas vezes fosse necessário durante o desenvolvimento.

Duas restrições pesaram na escolha: o orçamento não cobre o plano de controle
gerenciado do MKS somado às VMs dos nós, e o pipeline precisa de um endpoint de
API Kubernetes alcançável de fora (`kubectl apply` a partir do runner do
GitHub).

## Alternativas consideradas
- **K3s numa VM única** — distribuição Kubernetes leve, instalada em uma VM
  BV2-2-40; provisiona em menos de 2 minutos, custa apenas a VM, e aceita os
  mesmos manifests de um cluster completo (`Deployment`, `Service`
  `LoadBalancer` via Klipper, `ServiceMonitor` do kube-prometheus-stack).
  Contra: nó único, sem alta disponibilidade; a administração do cluster
  (upgrade, certificados, etcd/SQLite) é nossa.
- **MKS — Kubernetes gerenciado da Magalu Cloud** — plano de controle operado
  pelo provedor, múltiplos nós, alta disponibilidade real e integração nativa
  com o balanceador da nuvem. Contra: custo bem maior (plano de controle + nós),
  provisionamento mais lento e mais superfície de configuração para um projeto
  de duas réplicas.
- **VM com Docker Compose** — o mais simples de todos: `docker compose up` na
  VM, sem camada de orquestração. Contra: sem réplicas balanceadas de verdade,
  sem probes de liveness/readiness reagindo por conta própria, sem rollout com
  `rollout status`, e nenhum caminho de evolução para HPA. Também jogaria fora o
  objetivo de aprendizado de Kubernetes do projeto.

## Decisão
Usar **K3s em uma VM única** (BV2-2-40) na Magalu Cloud, com o `Deployment` de 2
réplicas e o `Service` do tipo `LoadBalancer` atendido pelo Klipper ServiceLB na
porta 80, e o deploy aplicado pelo GitHub Actions via `MGC_KUBECONFIG` contra o
kube-apiserver na 6443.

Critério decisivo: **custo por unidade de aprendizado**. O K3s entrega a API
Kubernetes de verdade — os mesmos manifests que rodariam no MKS — pelo preço de
uma VM. O Docker Compose seria mais barato ainda, mas não entrega orquestração;
o MKS entrega alta disponibilidade que o projeto não precisa demonstrar e o
orçamento não paga.

## Consequências
**Positivas:**
- Custo reduzido a uma única VM, sem cobrança de plano de controle.
- Manifests portáveis: migrar para o MKS depois é trocar o kubeconfig, não
  reescrever YAML.
- Rollout controlado (`kubectl rollout status`) e probes cuidando de reinício de
  pod sem intervenção manual.
- Ambiente recriável em menos de 2 minutos quando algo é destruído.

**Negativas:**
- **A VM é ponto único de falha.** As 2 réplicas protegem contra falha de
  processo (crash, OOM, rollout), não contra queda do nó — o alvo de 99,5% de
  disponibilidade depende da VM, não das réplicas.
- Manutenção do cluster (upgrade de K3s, certificados, backup do estado) fica
  por nossa conta.
- Exige expor a porta 6443 do kube-apiserver para o runner do GitHub Actions,
  ampliando a superfície de rede.
- Escalar além da capacidade da VM exige adicionar nós manualmente ou migrar
  para o MKS — não há autoescalonamento de nós.
