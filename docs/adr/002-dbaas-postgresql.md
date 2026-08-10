# ADR 002 — Usar DBaaS PostgreSQL da Magalu Cloud

**Status:** Aceito
**Data:** 2026-08-04

## Contexto
A aplicação precisa de um banco relacional durável, que sobreviva a
reinicializações de containers e atenda múltiplas réplicas da API ao mesmo
tempo. O cluster K3s roda em uma única VM (ver [ADR 001](001-kubernetes-deploy.md)),
e é justamente por o estado morar fora dela que o `Deployment` pode ter 2
réplicas stateless e ser recriado à vontade pelo pipeline.

## Alternativas consideradas
- **DBaaS PostgreSQL gerenciado (externo)** — backup, patch e HA pelo provedor;
  custo maior; menos controle fino sobre configurações.
- **PostgreSQL em pod com PVC** — custo baixo, tudo em um lugar; volume, backup
  e recuperação por nossa conta; o dado morre junto com o cluster.

## Decisão
Usar o serviço DBaaS PostgreSQL da Magalu Cloud, fora do cluster, acessado via
`Secret db-secret`. Critério decisivo: disponibilidade e custo de operação —
estado é caro de operar manualmente.

## Consequências
**Positivas:**
- Backup automático e patches de segurança pelo provedor.
- O banco sobrevive a qualquer redeploy do cluster.
- Conexões simultâneas de múltiplos pods sem conflito.

**Negativas:**
- Custo por hora de uso, mesmo com pouco tráfego.
- Menor controle sobre configurações avançadas do PostgreSQL.
- Dependência de conectividade de rede entre o cluster e o DBaaS.
