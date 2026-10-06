# 02 — Ação sensível conferindo o PIN sem limite de tentativas

**Classe:** CWE-307 (restrição imprópria de tentativas de autenticação)
**Gravidade:** alta — força bruta de um PIN de 4 dígitos.

## O padrão

O sistema tinha um guard de PIN correto, com castigo (3 erros → bloqueio) e
contador. Mas uma rota de ação sensível conferia o PIN por um atalho que pulava
os dois:

```python
# guard correto, usado na maioria das rotas
def guard_pin(cur, u, pin):
    if travado(u):                      # respeita o castigo
        raise HTTPException(429, "muitas tentativas")
    if not pin_ok(cur, pin):
        registrou_erro(u)               # conta a tentativa
        raise HTTPException(403, "PIN invalido")

# a rota furada: confere o PIN cru, sem castigo, sem contador
if not pin_ok(cur, body.pin):
    raise HTTPException(403, "PIN incorreto")
```

## Por que é explorável

A rota chamava só o `pin_ok` (o `bcrypt.verify`), sem consultar o castigo nem
incrementar o contador. Dava para tentar o PIN em laço: cada erro devolvia 403 e
não anotava nada, então o bloqueio nunca disparava. Pior — como o contador global
ficava intacto, o ataque era **invisível**: a vítima legítima nunca via o 429 que
sinalizaria abuso. PIN de 4 dígitos = 10.000 tentativas, minutos.

## O conserto

Uma linha: trocar a conferência crua pelo guard completo.

```python
guard_pin(cur, u, body.pin)   # castigo + contador, como as outras rotas
```

## A lição

Quando existe o "jeito seguro" e o "jeito cru" lado a lado, cedo ou tarde alguém
chama o cru. O conserto durável é não deixar o cru acessível — ou, no mínimo, ter
um teste que acuse toda rota que o chame. Foi um verificador varrendo as chamadas
que achou esta; o mesmo problema já tinha sido corrigido em três outras rotas, e
esta quarta tinha escapado.

## Como foi verificado

Pentest: errei o PIN cinco vezes seguidas nessa rota (com um id de recurso
inexistente, pra não tocar em nada real). **Antes:** 5× 403, nenhum 429.
**Depois:** 3× 403 e aí 429 — o castigo passou a morder, e o ataque passou a
contar no contador global.
