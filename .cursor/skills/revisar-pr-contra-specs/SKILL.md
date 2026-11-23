---
name: "revisar-pr-contra-specs"
description: "Revisa um PR do Menu Chef contra as specs. Use antes de aprovar ou ao pedir revisão de segurança multi-tenant, centavos, fuso, idempotência, Redis, cupom ou copy comercial."
---

# Revisão de PR contra as specs

Use numa revisão de pull request. A spec ganha do diff. Se o PR muda uma regra que está em mais de um arquivo de `specs/`, os outros arquivos precisam ter mudado juntos.

## Checklist

- **Tenant.** Toda query nova de tabela multi-tenant tem `branch_id` ou `company_id`. Allowlist nova tem motivo e é de fato pública ou da plataforma. Não há `ENABLE ROW LEVEL SECURITY`.
- **IDOR.** Dá para apontar o teste com duas empresas, ou o PR não mexe em leitura/escrita de recurso de filial.
- **Vínculo.** Migration de `memberships` preserva `UNIQUE NULLS NOT DISTINCT (user_id, company_id, branch_id)` e o `CHECK` do `company_owner`.
- **Dinheiro.** Centavos inteiros de ponta a ponta. Sem `float`. Sem coluna ou campo de taxa da plataforma. Copy de landing, FAQ, `llms.txt` e extrato diz "sem comissão, apenas mensalidade".
- **Fuso.** Horário de loja, "aberto agora", dashboard "hoje" e comprovante usam `branches.timezone`. Instante continua `timestamptz`. Sem `time.Local`. O caso Manaus × São Paulo está coberto se o PR mexe nisso.
- **Idempotência.** `POST` de pedido mantém `Idempotency-Key` na mesma transação do insert. Replay, em voo e payload divergente continuam 201, 409 e 422.
- **Cupom (23/11/2026).** Limite por cliente só com `customer_user_id`. Convidado recebe 422 e a sacola fica. Sem hash de telefone. Lock em `(coupon_id, customer_user_id)`. Estorno continua no cancelamento, na recusa e no Pix expirado.
- **Redis.** Um serviço, prefixos `mc:sess:`, `mc:cache:`, `mc:rl:`, canais `mc:ps:`, `volatile-ttl`, AOF, cache com TTL. Sem segundo Redis.
- **Sessão.** Cookie `HttpOnly`. Sem token em `localStorage`. Troca de filial rotaciona o session id.
- **UI.** 375 / 768 / 1440, contraste Brasa, alvo de toque, erro inline. Landing de produto não puxa a skill cinematográfica.

## Saída da revisão

Liste o que falhou com o caminho da spec e o trecho do diff. Não aprove com "parece ok" um item do checklist sem ter olhado o arquivo.
