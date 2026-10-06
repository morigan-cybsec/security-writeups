# 05 — Endpoint caro sem teto de concorrência (DoS)

**Classe:** CWE-400 (consumo descontrolado de recurso)
**Gravidade:** média — um usuário comum pode saturar o servidor.

## O padrão

Uma rota gerava PDF sob demanda. Cada render custa muita RAM e CPU, no mesmo host
do banco de dados — e nada limitava quantos rodavam ao mesmo tempo:

```python
@app.post("/documento/pdf")
def documento_pdf(body, req):
    u = usuario_logado(req)
    # ... valida permissao e tamanho do HTML ...
    pdf = render_pdf(body.html)     # ~50 MB de RAM, sincrono, sem teto
    return entregar(pdf)
```

O tamanho do HTML já tinha limite. O que faltava era limite de **quantidade
simultânea**.

## Por que é explorável

Cada requisição é tratada na hora, de forma independente. Operações baratas
aguentam isso de sobra; uma cara, não. Um usuário logado abrindo vários PDFs em
sequência — ou um laço — consome a máquina e leva o banco junto. Não precisa de
má-fé: basta uso intenso.

## O conserto

Um semáforo limitando renders simultâneos; o excedente espera um pouco e, se não
entrar, recebe 503 em vez de empilhar carga:

```python
_PDF_SEM = threading.Semaphore(int(os.getenv("PDF_RENDER_SIMULTANEOS", "2")))
_ESPERA_S = float(os.getenv("PDF_RENDER_ESPERA_S", "20"))

if not _PDF_SEM.acquire(timeout=_ESPERA_S):
    raise HTTPException(503, "servidor ocupado gerando PDFs; tente em instantes")
try:
    pdf = render_pdf(body.html)
finally:
    _PDF_SEM.release()
```

## A lição

Toda operação cara exposta a usuário precisa de um teto de concorrência — não
porque alguém vá atacar, mas porque um dia alguém vai clicar rápido demais. A
pergunta não é "e se abusarem?", é "e se usarem muito?". Degradar com elegância
(503 honesto) é melhor que quebrar (host no chão).

## Como foi verificado

Revisão do caminho de render + o limite parametrizável por ambiente, pra ajustar
o teto sem novo deploy. O teste de carga (disparar N renders concorrentes e ver o
503 a partir do N+1) fica como exercício de stress à parte.
