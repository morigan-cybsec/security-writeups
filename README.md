# Security case studies — FastAPI CRM

Estudos de caso de vulnerabilidades que encontrei, explorei e corrigi num CRM em
produção (FastAPI + PostgreSQL, arquivo único, deploy contínuo sem CI, um
desenvolvedor). Cada caso segue o mesmo formato: **o padrão → por que é
explorável → o conserto → a lição → como foi verificado.**

O valor aqui não é o conserto — quase todos têm menos de dez linhas. É **achar**:
ler o sistema, separar o furo real do falso alarme e **provar** o comportamento
antes e depois, não só confiar que a linha mudou.

## Escopo e responsabilidade

- **Código ilustrativo, não fonte.** Os trechos reproduzem o *padrão* da falha e
  do conserto em FastAPI/Python genérico. Não são o código-fonte do sistema real:
  nomes de rota, tabela e função foram trocados por equivalentes neutros.
- **Nada sensível.** Nenhum segredo, credencial, dado de cliente, nome de empresa
  ou host. Nenhuma informação que identifique o sistema ou permita alcançá-lo.
- **Só o que já foi corrigido.** Publico um caso depois que a correção está em
  produção e verificada. Falhas ainda em aberto ficam retidas até o conserto —
  divulgar detalhe de falhas ativas num sistema em uso é irresponsável.

## Casos

| # | Falha | Classe | Gravidade |
|---|---|---|---|
| [01](cases/01-segredo-com-fallback-embutido.md) | Segredo de sessão com valor-padrão embutido no código | CWE-798 / CWE-1188 | Crítica |
| [02](cases/02-pin-sem-limite-de-tentativas.md) | Rota de ação sensível conferindo PIN sem limite de tentativas | CWE-307 | Alta |
| [03](cases/03-webhook-fail-open.md) | Webhook aceitando evento sem assinatura (fail-open) | CWE-347 | Alta |
| [04](cases/04-null-derruba-dedup.md) | `NULL` anulando uma trava de unicidade (replay/inchaço) | idempotência | Média |
| [05](cases/05-endpoint-caro-sem-teto.md) | Endpoint caro sem teto de concorrência (DoS) | CWE-400 | Média |

## Método

Para cada caso: medição só-leitura antes de tocar em qualquer coisa; conserto
mínimo e ancorado; e um **pentest de comportamento** que prova o buraco fechado
(ex.: a requisição que antes passava agora recebe 403). Verificadores
automatizados reexecutam a cada mudança para pegar regressão.

---

_Autoria: **Morigan Ventrici** — morigan.cs@gmail.com. Escrito a partir de trabalho real, sanitizado para publicação._
