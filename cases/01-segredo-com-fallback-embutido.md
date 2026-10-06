# 01 — Segredo de sessão com valor-padrão embutido no código

**Classe:** CWE-798 (credencial embutida) + CWE-1188 (configuração-padrão insegura)
**Gravidade:** crítica — sequestro de sessão de administrador sem senha.

## O padrão

A chave que assina os tokens de sessão era lida com um valor-padrão de "exemplo"
embutido, usado quando a variável de ambiente faltasse:

```python
SECRET = os.getenv("APP_SECRET", "change-me-in-env").encode()

def assinar(uid, horas=12):
    corpo = f"{uid}.{int(time.time()) + horas*3600}"
    mac = hmac.new(SECRET, corpo.encode(), hashlib.sha256).hexdigest()
    return f"{corpo}.{mac}"
```

Parece inofensivo: "se a config faltar, o app ainda sobe". Para um segredo de
autenticação, é o contrário do que se quer.

## Por que é explorável

O token é `uid.expira.HMAC(SECRET, "uid.expira")`. Se o `.env` não carregar
(systemd sem `EnvironmentFile`, diretório errado, container novo), o app sobe
assinando sessão com uma string que está no repositório público. Qualquer um
calcula o MAC de `1.<futuro>` com o segredo conhecido, monta o cookie e entra
como o usuário 1 — admin numa instalação nova. Sem senha, sem segundo fator.
Isso derrota todas as outras travas do sistema de uma vez.

## O conserto

Falhar na subida em vez de cair no padrão:

```python
SECRET = (os.getenv("APP_SECRET") or "").encode()
if len(SECRET) < 32:
    raise RuntimeError("APP_SECRET ausente ou curto no .env — o app nao sobe sem ele.")
```

Gerar o segredo: `python -c "import secrets; print(secrets.token_urlsafe(48))"`.

## A lição

Config que compromete a segurança quando falta **não deve ter fallback — deve ter
fail-closed.** Um `raise` na inicialização é barato e visível; o buraco silencioso
não é. "O app nunca quebra por falta de config" é uma anti-feature para segredos.

## Como foi verificado

Pentest de comportamento: forjei um cookie com o segredo-de-exemplo e bati numa
rota autenticada. **Antes:** 200, sessão válida. **Depois do conserto:** 401 —
o segredo de exemplo não assina mais nada, prova de que o `.env` real está no ar.
