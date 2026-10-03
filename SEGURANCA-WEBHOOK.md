# Seguranca: bloqueio do webhook WAHA

Este fork tem uma protecao customizada que NAO existe no repositorio
original (melgarafael/DeskcommCRM): o endpoint `/api/v1/webhooks/waha`
fica bloqueado publicamente via labels do Traefik no servico `worker`,
dentro de `docker-compose.prod.yml`.

## Por que isso existe

Sem esse bloqueio, o endpoint de webhook do WAHA fica acessivel
publicamente em `https://crm1.holowealth.com.br/api/v1/webhooks/waha`,
sem autenticacao. O bloqueio faz o Traefik responder 403 para qualquer
requisicao a esse caminho especifico, sem afetar o resto do site.

## Onde estao os labels

No servico `worker`, chave `labels:`, em `docker-compose.prod.yml`.
Os elementos criticos que precisam sempre existir juntos:

- `traefik.http.routers.deskcomm-waha-block.rule=Host(\`crm1.holowealth.com.br\`) && Path(\`/api/v1/webhooks/waha\`)`
- `traefik.http.routers.deskcomm-waha-block.middlewares=deskcomm-deny`
- `traefik.http.routers.deskcomm-waha-block.service=noop@internal`
- `traefik.http.middlewares.deskcomm-deny.ipallowlist.sourcerange=192.0.2.1/32`
- `traefik.http.routers.deskcomm-waha-block.priority=1000`

## Protecao automatica

O workflow `.github/workflows/verify-security-labels.yml` roda a cada
push ou pull request que toque `docker-compose.prod.yml` e falha (X
vermelho no commit/PR) se algum desses labels sumir. Isso cobre o caso
mais comum de perda: sincronizar este fork com o upstream
(melgarafael/DeskcommCRM) e o merge sobrescrever o arquivo.

## Como sincronizar este fork com seguranca

NUNCA use o botao "Sync fork" do GitHub direto na branch main sem revisar
antes — ele pode trazer mudancas no `docker-compose.prod.yml` que
sobrescrevem os labels acima.

Procedimento seguro recomendado:

1. No GitHub, va em "Sync fork" e escolha comparar antes de aplicar, ou
   crie uma branch separada a partir do upstream
   (`git fetch upstream && git checkout -b sync-upstream upstream/main`).
2. Veja especificamente o diff de `docker-compose.prod.yml` entre essa
   branch e a sua `main` atual.
3. Se o upstream mudou esse arquivo, traga as mudancas manualmente para a
   `main`, mas preserve o bloco `labels:` do servico `worker` listado
   acima.
4. Depois de qualquer sync, SEMPRE confira se o check "Verificar labels
   de seguranca do webhook WAHA" passou (verde) no commit, na aba
   "Actions" do repositorio, antes de fazer deploy no Coolify.
5. Se o check falhar, reaplique os labels manualmente (copie da secao
   "Onde estao os labels" acima) antes de redeploy.

## Teste manual

```
curl -s -o /dev/null -w "%{http_code}\n" https://crm1.holowealth.com.br/api/v1/webhooks/waha
```

Deve retornar `403`. Se retornar outra coisa (200, 404, etc.), o
bloqueio nao esta ativo e o endpoint pode estar exposto.
