# 03 — Webhook aceitando evento sem assinatura (fail-open)

**Classe:** CWE-347 (verificação imprópria de assinatura criptográfica), fail-open
**Gravidade:** alta — evento de pagamento forjado aceito sem conta.

## O padrão

Um webhook de provedor de pagamento (sem login, autenticado por HMAC) conferia a
assinatura, mas só recusava quando ela estava **errada**:

```python
def confere(corpo_assinado, assinatura, chave):
    if not (corpo_assinado and assinatura and chave):
        return None                     # "nao da para afirmar nem negar"
    calc = hmac.new(chave.encode(), corpo_assinado.encode(), hashlib.sha256).hexdigest()
    return hmac.compare_digest(assinatura, calc)

ok = confere(corpo_assinado, assinatura, chave)
if ok is False:                         # <-- so barra a assinatura ERRADA
    raise HTTPException(403, "assinatura nao confere")
# ... segue gravando o evento e disparando trabalho
```

## Por que é explorável

A verificação tem três estados: confere (`True`), não confere (`False`),
indefinido (`None` — faltou a assinatura ou a chave). O ponto de recusa só olhava
o `False`. Um corpo **sem o campo de assinatura** cai no `None` e **escorre pelo
meio**: o handler segue, grava o evento numa tabela que nunca é apagada e dispara
uma leitura (paga) na API do provedor. É fail-open clássico — a ausência de prova
virou permissão. (Era também o único dos vários webhooks do sistema que não
recusava sem segredo; os outros devolviam 503 — o contraste denunciou o descuido.)

## O conserto

Inverter a condição: só passa quem **confere**.

```python
if ok is not True:                      # fail-CLOSED: ausente ou errada -> 403
    raise HTTPException(403, "assinatura nao confere (ou ausente)")
```

## A lição

Em verificação de segurança, "indefinido" tem que cair do lado do **não**. Todo
teste de três estados precisa responder explicitamente o que acontece com o
terceiro — é nele que o fail-open se esconde. `if not valido` e `if invalido` não
são a mesma coisa quando existe um meio-termo.

## Como foi verificado

Pentest: POST no webhook com um corpo **sem** o campo de assinatura.
**Antes:** 200, evento gravado. **Depois:** 403, nada gravado — e o caminho
legítimo (assinatura correta) seguiu em 200, pra não trocar um fail-open por um
fail-closed que derruba a integração real.
