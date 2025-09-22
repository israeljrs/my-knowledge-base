# PostgreSQL COLLATE: guia prático de localidade, ordenação e comparação

Este guia e uma referência única e prática sobre collations no PostgreSQL: o que são, como escolher, como configurar, resolver erros comuns e as melhores práticas de uso em produção.

## 1) Conceitos rápidos

- **Encoding (ex.: UTF-8)**: modo de codificar caracteres. No PostgreSQL, é definido no banco (e herdado das bases-template). UTF-8 suporta praticamente todos os alfabetos.
- **Collation (COLLATE)**: regras de comparação e ordenação de strings. Define como `ORDER BY`, `GROUP BY` e comparações lidam com maiúsculas/minúsculas, acentos, cedilha etc.
- **Locale do SO x PostgreSQL**: em configurações baseadas em glibc, o PostgreSQL usa collations fornecidos pelo sistema operacional (o nome do collation precisa existir no SO). Em configurações com ICU, é possível usar collations ICU (mais consistentes entre plataformas).

Observações importantes:
- Não existe um collation chamado “utf8” por si só. Use um collation de localidade com UTF-8, como `pt_BR.UTF-8` (ou `pt_BR.utf8`, conforme o SO), `en_US.UTF-8`, `C.UTF-8`/`C.utf8`, ou ICU como `und-x-icu`.
- Collations baseados em regras linguísticas (pt_BR, en_US) custam mais CPU do que `C`/`POSIX` (ordem binária), mas oferecem ordenação correta para o idioma.

## 2) Onde o COLLATE é aplicado

- **Banco de dados** (padrões para objetos novos): `LC_COLLATE` e `LC_CTYPE` no momento da criação do banco.
- **Coluna**: `COLLATE` por coluna define como os valores daquele campo são ordenados/comparados.
- **Expressão/consulta**: `... ORDER BY coluna COLLATE "pt_BR.UTF-8"` aplica a regra só naquele `ORDER BY`.

## 3) Criando banco com UTF-8 e collation

Use `template0` para garantir independência das configurações da template padrão:

```sql
CREATE DATABASE meu_banco
  WITH ENCODING 'UTF8'
       LC_COLLATE 'pt_BR.UTF-8'
       LC_CTYPE   'pt_BR.UTF-8'
       TEMPLATE template0;
```

Variações do nome do collation podem ocorrer; ajuste para `pt_BR.utf8` ou `pt_BR.UTF-8` conforme o seu sistema (veja seção 6).

## 4) Definindo COLLATE em colunas e consultas

Coluna com collation específico:

```sql
CREATE TABLE usuarios (
  nome varchar(100) COLLATE "pt_BR.UTF-8"
);
```

Aplicando collation em uma consulta:

```sql
SELECT nome
FROM usuarios
ORDER BY nome COLLATE "pt_BR.UTF-8";
```

Exemplo de diferença prática (dados: 'Andre', 'Álvaro', 'Ana'):
- `pt_BR.UTF-8`: respeita regras do português (acentos, cedilha etc.).
- `C.UTF-8`/`C.utf8`: ordenação binária (rápida, mas sem regras linguísticas).

## 5) Verificando collations e encoding

No PostgreSQL:

```sql
SELECT collname, collcollate, collctype
FROM pg_collation
ORDER BY collname;

SHOW server_encoding;   -- deve ser UTF8
SHOW lc_collate;        -- collation padrão do banco
SHOW lc_ctype;          -- classificação de caracteres padrão
```

No sistema operacional (Linux):

```bash
locale -a | grep -i pt_BR
```

Procure por `pt_BR.UTF-8` ou `pt_BR.utf8`. Use o nome exato que aparecer.

## 6) Erro comum: “collation ... does not exist”

Mensagem típica:

```
ERROR: collation "pt_BR.UTF8" for encoding "UTF8" does not exist
```

Como resolver:
1. **Confirme o nome correto**: veja `locale -a` e use exatamente `pt_BR.UTF-8` ou `pt_BR.utf8` (varia por distro).
2. **Instale a localidade no SO** (Ubuntu/Debian):
   ```bash
   sudo apt-get install locales
   sudo locale-gen pt_BR.UTF-8
   sudo dpkg-reconfigure locales
   sudo systemctl restart postgresql
   ```
   Em RHEL/CentOS:
   ```bash
   sudo yum install glibc-langpack-pt
   sudo localedef -i pt_BR -f UTF-8 pt_BR.UTF-8
   sudo systemctl restart postgresql
   ```
3. **Verifique encoding do banco**:
   ```sql
   SHOW server_encoding; -- deve ser UTF8
   ```
   Se não for UTF8, crie um novo banco conforme a seção 3 e migre os dados.

## 7) Alterando collation depois

### 7.1) Em uma coluna

```sql
ALTER TABLE nomes
ALTER COLUMN nome TYPE varchar(50) COLLATE "pt_BR.UTF-8";
```

Recrie ou reindexe índices que dependem dessa coluna:

```sql
REINDEX TABLE nomes;
```

### 7.2) Em várias colunas de texto da mesma tabela

Use um bloco `DO` para automatizar (ajuste o nome da tabela):

```sql
DO $$
DECLARE r RECORD;
BEGIN
  FOR r IN (
    SELECT attname
    FROM pg_attribute
    WHERE attrelid = 'nomes'::regclass
      AND atttypid IN ('varchar'::regtype, 'text'::regtype)
      AND NOT attisdropped
  ) LOOP
    EXECUTE format(
      'ALTER TABLE nomes ALTER COLUMN %I TYPE %s COLLATE "pt_BR.UTF-8"',
      r.attname, pg_typeof(r.attname)::text
    );
  END LOOP;
END $$;
```

### 7.3) No banco inteiro

Não é possível alterar `LC_COLLATE`/`LC_CTYPE` de um banco existente. Crie um novo banco e migre:

```sql
CREATE DATABASE novo_banco
  WITH ENCODING 'UTF8'
       LC_COLLATE 'pt_BR.UTF-8'
       LC_CTYPE   'pt_BR.UTF-8'
       TEMPLATE template0;
```

Depois, exporte e restaure:

```bash
pg_dump -U usuario banco_atual > dump.sql
psql -U usuario -d novo_banco -f dump.sql
```

Recrie/reindexe índices quando necessário.

## 8) Índices, performance e consistência

- **Índices dependem do collation**: mude o collation → reindexe para garantir consistência.
- **Functional indexes com COLLATE**: aceleram `ORDER BY ... COLLATE ...` frequentes.

```sql
CREATE INDEX idx_usuarios_nome_ptbr
  ON usuarios ((nome COLLATE "pt_BR.UTF-8"));
```

- **Performance**: `C.UTF-8`/`C.utf8` é mais rápido (ordem binária), mas não segue regras linguísticas.
- **Consistência**: evite misturar collations sem necessidade. Defina padrões claros por banco/coluna.

## 9) ICU vs glibc (quando disponível)

- **ICU**: collations consistentes entre sistemas e configuráveis. Ex.: `und-x-icu` (genérico Unicode).
- **glibc (locale do SO)**: depende do que o SO fornece (`locale -a`).

Se sua instalação tiver ICU habilitado, prefira ICU para previsibilidade entre ambientes.

## 10) Testes rápidos de comportamento

```sql
-- Dados de exemplo
INSERT INTO nomes (nome) VALUES ('Andre'), ('Álvaro'), ('Ana');

-- Ordenação com português do Brasil
SELECT nome FROM nomes ORDER BY nome COLLATE "pt_BR.UTF-8";

-- Ordenação binária
SELECT nome FROM nomes ORDER BY nome COLLATE "C.UTF-8"; -- ou "C.utf8"
```

Compare os resultados para validar se a ordenação atende ao seu caso de uso.

## 11) Boas práticas

- **Escolha consciente**: use `pt_BR.UTF-8` para UX correta em português; use `C.UTF-8`/`C.utf8` para operações técnicas/bulk onde a ordem binária baste.
- **Padronize nomes**: confirme com `locale -a` o nome exato do collation no SO.
- **Planeje migrações**: alterar collation pode impactar índices e consultas; teste antes e faça backup.
- **Evite surpresas**: aplique `COLLATE` explicitamente em consultas críticas se houver risco de ambiente variar.

---

Referências úteis no próprio ambiente:
- `SELECT * FROM pg_collation;`
- `SHOW lc_collate; SHOW server_encoding;`
- `locale -a` (no SO) para listar localidades disponíveis.

