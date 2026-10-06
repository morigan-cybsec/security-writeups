# 06 — O alarme que eu derrubei: "SQL dinâmico" que não era injeção

**Classe:** falso positivo de SQL injection (CWE-89) — *descartado com prova*
**Resultado:** não explorável. Nenhuma mudança de código.

> Achar bug é metade do trabalho de auditoria. A outra metade é **descartar** o
> que parece bug e não é — com a mesma prova que você exigiria pra confirmar. Um
> falso positivo reportado como real queima confiança e tempo da equipe. Este
> caso existe pra mostrar o lado que raramente aparece em portfólio.

## O que acendeu o alarme

Uma varredura apontou **46 consultas** montadas com f-string em vez de parâmetros
— o padrão que, à primeira vista, grita SQL injection:

```python
cur.execute(f"UPDATE clientes SET {sets} WHERE id = %(id)s", {**campos, "id": cid})
cur.execute(f"SELECT {COLUNAS} FROM leads cl WHERE {where}", par)
```

`f"...{sets}..."` e `f"...{where}..."` interpolam texto direto na query. Reportar
"46 pontos de SQLi" aqui seria fácil — e errado.

## A investigação (de onde vem cada pedaço interpolado)

SQL injection exige que **entrada do usuário** chegue à string da query. Então a
pergunta não é "há interpolação?", é "o que exatamente é interpolado, e de onde
vem?". Fui peça por peça:

- **`{sets}` e `{cols}`** (INSERT/UPDATE dinâmicos): montados **só** a partir de
  uma lista fechada de nomes de coluna definida no código
  (`CAMPOS_EDITAVEIS = {"nome", "cpf", ...}`). O corpo da requisição é filtrado
  contra essa lista **antes**; uma chave fora dela é descartada. O que entra na
  string são nomes de coluna do próprio programa, nunca texto do usuário.
- **Os valores** correspondentes vão todos por parâmetro nomeado (`%(campo)s`),
  não na f-string. É a separação que mata a injeção.
- **`{where}` / `{cond}`**: vêm de um construtor central de filtros. Todo valor
  do usuário entra por `%s`; as datas passam por `re.fullmatch(r"\d{4}-\d{2}-\d{2}")`
  antes; listas de ids passam por `int()`; e o "nome da coluna de data" é
  resolvido por um dicionário fixo de dois valores, não por texto livre.
- **A cláusula de dono** interpola só um *alias* de tabela (`"cl"`), que é
  constante no código.

Resultado: em nenhuma das 46 existe um caminho de string da requisição para
dentro do SQL. O que interpola é sempre identificador do próprio programa
(coluna, alias), vindo de lista fechada; o que é do usuário vai parametrizado.

## O veredito

**Não explorável.** As 46 ficam como estão. O que eu entreguei não foi um
patch — foi a recomendação de **rebaixar o item** de "46 SQLi a corrigir" para
"manter a disciplina de lista-fechada em revisão de código", e o raciocínio
acima pra quem quisesse reconferir.

## A lição

- **f-string em SQL é um cheiro, não um diagnóstico.** O diagnóstico é a resposta
  a "o que é interpolado e de onde vem". Identificador de lista fechada ≠ valor
  do usuário.
- **Descartar com rigor é uma entrega.** Um auditor que reporta 46 falsos
  positivos faz a equipe parar de ler os relatórios dele. Precisão — nos dois
  sentidos, confirmar e descartar — é o produto.
- O custo de verificar foi real (ler as 46 uma a uma), e foi o certo. "Parece
  perigoso" não é severidade.

## Como foi verificado

Leitura dirigida das 46 ocorrências, rastreando a origem de cada expressão
interpolada até a constante/lista que a define, e confirmando que todo valor de
requisição chega por parâmetro. Zero exigia correção.
