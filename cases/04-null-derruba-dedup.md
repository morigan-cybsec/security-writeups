# 04 — `NULL` anulando uma trava de unicidade (replay / inchaço)

**Classe:** falha de idempotência / replay (relacionada a CWE-20)
**Gravidade:** média — reenvio sem fim de eventos, crescimento ilimitado de tabela.

## O padrão

Um webhook guardava cada evento com uma trava de duplicata no banco:

```python
cur.execute(
    """INSERT INTO eventos (message_id, payload)
       VALUES (%s, %s::jsonb)
       ON CONFLICT (message_id) DO NOTHING
       RETURNING id""",
    (corpo.get("id"), json.dumps(corpo)))   # <-- corpo.get("id") pode ser None
```

Parecia blindado: `ON CONFLICT ... DO NOTHING` diz "se já existe, ignora".

## Por que é explorável

O `message_id` vinha de um campo do corpo que **podia chegar nulo** (evento sem o
id, ou corpo que nem era JSON). E em PostgreSQL `NULL` nunca conflita com `NULL` —
a cláusula de unicidade deixa passar quantos nulos você mandar. Resultado: todo
evento sem id entrava como linha nova, a cada reentrega, sem fim. A trava existia
no papel; o dado de que ela dependia podia ser nulo.

## O conserto

Gerar um id determinístico quando o do corpo vier vazio — o mesmo hash do corpo,
que já era usado em outro webhook do sistema:

```python
mid = corpo.get("id") or ("evt:" + hashlib.sha256(bruto or b"").hexdigest()[:40])
cur.execute("""INSERT INTO eventos (message_id, payload) VALUES (%s, %s::jsonb)
               ON CONFLICT (message_id) DO NOTHING RETURNING id""",
            (mid, json.dumps(corpo)))
```

## A lição

`ON CONFLICT`/`UNIQUE` sobre uma coluna que aceita `NULL` é uma trava que
destranca sozinha — porque, em SQL, `NULL` é "desconhecido", e dois desconhecidos
não são iguais por definição. Ao confiar numa constraint de unicidade, a pergunta
é: **essa coluna pode ser nula?** Se pode, a trava tem um buraco do tamanho de
todos os nulos.

## Como foi verificado

Pentest: enviei o mesmo corpo sem id duas vezes. **Antes:** duas linhas novas.
**Depois:** a segunda volta `repetido` (deduplicada). O id-fallback nunca é nulo,
então o `DO NOTHING` volta a ter contra o que conflitar.
