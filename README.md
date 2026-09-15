# Campo calculado de Frete no Portal de Importação de XML (TGFIXN)

Registro técnico da investigação e da solução aplicada no Sankhya para exibir, no Portal de
Importação de XML, o valor de frete cadastrado no cabeçalho da nota de venda — permitindo ao
operador conferir o valor cobrado no CT-e contra o valor previsto.

Ambiente: Sankhya sobre Oracle.

---

## 1. Objetivo

No Portal de Importação de XML (tabela `TGFIXN`) chegam os CT-e emitidos pelas transportadoras,
com o valor efetivamente cobrado pelo frete. O valor previsto/contratado está no campo
customizado `AD_VALORFRETE` do cabeçalho da nota (`TGFCAB`).

A conferência era manual, abrindo dois documentos em telas diferentes. O objetivo foi trazer o
valor da `TGFCAB` para dentro da grade da `TGFIXN`.

---

## 2. Onde se cria o campo

**Dicionário de Dados**, não Construtor de Telas.

Esse é o ponto que mais custou tempo. Para telas nativas do Sankhya, campos personalizados e
calculados são cadastrados no **Dicionário de Dados**, localizando a instância pelo nome da
tabela. O Construtor de Telas serve para layout e comportamento, não para criar o campo.

Caminho: Dicionário de Dados → instância `ImportacaoXMLNotas` (tabela `TGFIXN`) → aba Campos →
Novo.

Configuração do campo:

| Propriedade | Valor |
|---|---|
| Nome do campo | `AD_FRETECAB` |
| Descrição | Frete Cabeçalho |
| Tipo de dados | Número Decimal |
| Apresentação | Padrão |
| Permite pesquisa | Não |
| Visível no grid de pesquisa | Não |
| Campo calculado | **Sim** |

`Permite pesquisa` deve ficar desligado: com ele ativo o Sankhya tenta usar a subconsulta em
cláusula de filtro, o que causa erro ou lentidão severa na tela de busca.

---

## 3. A diretiva `#type.sql#`

O campo Expressão **não aceita SQL por padrão**. Ele passa por um parser proprietário do Sankhya,
com gramática própria e restrita, que rejeita subconsultas e funções Oracle.

Sintomas de quem não sabe disso (todos foram encontrados nesta investigação):

```
Parse error at line 3, column 9. Encountered: NVL      Código: CORE_E01906
Parse error at line 3, column 9. Encountered: CAB      Código: CORE_E01906
Parse error at line 3, column 9. Encountered: VLRFRETE Código: CORE_E01906
```

O erro muda de token a cada tentativa de reescrita, o que dá a falsa impressão de erro de
sintaxe. Não é: o parser está rejeitando a construção inteira.

A solução é a diretiva `#type.sql#` **sozinha na primeira linha**, sem nada antes. A partir da
segunda linha o conteúdo é tratado como SQL bruto e entregue ao Oracle:

```
#type.sql#
(SELECT CAB.AD_VALORFRETE FROM TGFCAB CAB WHERE CAB.NUNOTA = TGFIXN.NUNOTA)
```

Essa diretiva foi descoberta inspecionando um campo calculado nativo já existente na mesma
instância (`DIASEMISSAOCALC`). **Quando algo não funciona numa tela do Sankhya, procure um campo
nativo que faça algo parecido e copie o padrão dele.**

Se o alias `TGFIXN.NUNOTA` não for aceito, use apenas `NUNOTA` — depende de como o Sankhya monta
o alias da tabela principal na versão.

---

## 4. Estrutura do dado

### TGFIXN (portal de importação)

Não possui campo de frete. De valores, apenas `VLRNOTA` (valor total do documento importado).

Distribuição por tipo de documento:

| TIPO | Qtd | Significado |
|---|---|---|
| C | 52.703 | CT-e (conhecimento de transporte) |
| N | 50.736 | NF-e |
| (vazio) | 3.936 | — |

Colunas relevantes: `NUNOTA`, `DOCSREF` (CLOB), `XML` (CLOB), `VLRNOTA`, `TIPO`, `DHIMPORT`,
`XNOMEEMIT`, `CHAVEACESSO`.

### TGFCAB (cabeçalho de nota)

Dois campos distintos de frete, facilmente confundidos:

| Campo | Onde aparece na tela | O que é |
|---|---|---|
| `VLRFRETE` | Rodapé → Transporte | Frete declarado na NF-e |
| `AD_VALORFRETE` | Cabeçalho → "Valor do Frete" | Campo customizado — frete previsto/contratado |

Na nota 200599 os dois divergem: `AD_VALORFRETE` = 76,26 e `VLRFRETE` = 57,65. São números
diferentes com finalidades diferentes.

`AD_VALORFRETE` é campo **físico**, sem regra de cálculo (Campo calculado = Não, Expressão
vazia). É gravado pelo fluxo de venda, já no pedido, antes da nota existir.

Preenchimento: 40.219 de 168.506 registros (24%).

---

## 5. O vínculo entre CT-e e nota

O CT-e não tem `NUNOTA` preenchido. A ligação está na coluna `DOCSREF`, que **é XML, não texto
puro**:

```xml
<docsRef><chaveAcesso>35260928414558000132550030000436131265541475</chaveAcesso></docsRef>
```

Um `SUBSTR(DOCSREF, 44, 1)` retorna `<docsRef><chaveAcesso>352210284145580001` — tags incluídas.
Por isso o join nunca casava.

A extração correta ignora as tags pegando a primeira sequência de 44 dígitos:

```sql
REGEXP_SUBSTR(DBMS_LOB.SUBSTR(DOCSREF, 300, 1), '[0-9]{44}')
```

### Anatomia da chave de 44 dígitos

| Posição | Conteúdo |
|---|---|
| 1-2 | UF |
| 3-6 | AAMM da emissão |
| 7-20 | CNPJ do emitente |
| 21-22 | Modelo (55 = NF-e, 57 = CT-e) |
| 23-25 | Série |
| 26-34 | Número da nota |

Decodificar a chave manualmente foi decisivo: revelou que os CT-e referenciam notas **série 3**,
enquanto as vendas internas são **série 1**. Também revelou CT-e que referenciam outro CT-e
(modelo 57), que não têm nota correspondente.

### Resultado do join (últimos 30 dias)

```sql
SELECT COUNT(*) AS TOTAL_CTE,
       COUNT(CAB.NUNOTA) AS CASOU_NOTA,
       COUNT(CAB.AD_VALORFRETE) AS COM_FRETE
  FROM TGFIXN IXN
  LEFT JOIN TGFCAB CAB
    ON CAB.CHAVENFE = REGEXP_SUBSTR(DBMS_LOB.SUBSTR(IXN.DOCSREF, 300, 1), '[0-9]{44}')
 WHERE IXN.TIPO = 'C'
   AND IXN.DHIMPORT >= TRUNC(SYSDATE) - 30;
```

| TOTAL_CTE | CASOU_NOTA | COM_FRETE |
|---|---|---|
| 1.739 | 1.576 (91%) | 266 |

O join funciona. 266 CT-e têm contraparte com valor de frete cadastrado.

---

## 6. Solução final

### Campo 1 — `AD_FRETECAB` (Frete Cabeçalho)

```
#type.sql#
(SELECT CAB.AD_VALORFRETE FROM TGFCAB CAB WHERE CAB.CHAVENFE = REGEXP_SUBSTR(DBMS_LOB.SUBSTR(TGFIXN.DOCSREF, 300, 1), '[0-9]{44}'))
```

**Sem `NVL` de propósito.** O `NVL(campo, 0)` transforma "não cadastrado" e "cadastrado como
zero" no mesmo `0,00`, apagando a distinção que importa numa conferência. Em branco significa
pendente de cadastro; `0,00` significa frete zero cadastrado.

### Campo 2 — `AD_DIFFRETE` (Diferença Frete)

```
#type.sql#
(SELECT CAB.AD_VALORFRETE - TGFIXN.VLRNOTA FROM TGFCAB CAB WHERE CAB.CHAVENFE = REGEXP_SUBSTR(DBMS_LOB.SUBSTR(TGFIXN.DOCSREF, 300, 1), '[0-9]{44}'))
```

Positivo = transportadora cobrou menos que o previsto. Negativo = cobrou mais.

### Após salvar

1. Fechar a tela
2. Reiniciar a unidade de dados / limpar cache
3. Se a coluna não aparecer na grade, botão direito no cabeçalho → configurar colunas

---

## 7. Desempenho

`DBMS_LOB.SUBSTR` combinado com `REGEXP_SUBSTR` **não usa índice** e roda uma vez por linha
exibida. Com dois campos, são duas varreminações de CLOB por registro.

Testar com o portal filtrado num período curto antes de liberar. Abrir a grade sem filtro sobre
50 mil CT-e trava a tela.

Se ficar lento, materializar numa view `AD_VW...` ou num campo físico alimentado por rotina.

---

## 8. Metodologia — o que fez diferença

Esta seção é o principal aprendizado. A investigação levou dezenas de consultas e produziu três
conclusões erradas pelo caminho, todas pelo mesmo motivo.

### Buscar por `IS NOT NULL` em vez de amostrar às cegas

A pergunta "esse campo é preenchido?" não se responde olhando linhas aleatórias. Responde-se
perguntando onde o campo **tem** valor:

```sql
-- Errado: mostra 30 linhas quaisquer, provavelmente as mais antigas
SELECT NUNOTA, AD_VALORFRETE FROM TGFCAB WHERE ROWNUM <= 30;

-- Certo: mostra onde o dado vive
SELECT NUNOTA, NUMNOTA, SERIENOTA, CODTIPOPER, DTNEG, AD_VALORFRETE, CODPARCTRANSP
  FROM TGFCAB
 WHERE AD_VALORFRETE IS NOT NULL
 ORDER BY DTNEG DESC
 FETCH FIRST 30 ROWS ONLY;
```

A segunda consulta entregou de uma vez: quais TOPs alimentam o campo, em que datas, com quais
transportadoras, e que ele é gravado já no pedido (série nula + `VLRFRETE` zero + valor
preenchido). Foi o ponto de virada da investigação.

### `ROWNUM` corta antes de ordenar

```sql
-- Errado: o Oracle corta primeiro e ordena depois
SELECT ... FROM T ORDER BY DATA DESC WHERE ROWNUM <= 30;

-- Certo
SELECT ... FROM T ORDER BY DATA DESC FETCH FIRST 30 ROWS ONLY;
```

Esse detalhe produziu **três conclusões erradas** nesta investigação:

1. "O `AD_VALORFRETE` nunca é preenchido no portal" — baseado em 30 linhas que eram todas
   registros de teste antigos (`NUNOTA` entre 1.549 e 3.912, contra 200.000+ atuais)
2. "A série 3 não existe" — quando existem 43.281 notas
3. "A nota referenciada pelo CT-e não está na TGFCAB" — estava, em outra faixa de data

Em Oracle 11g ou anterior, usar a subquery com `ROWNUM` externo.

### Contagem agregada antes de investigar caso a caso

```sql
SELECT COUNT(*) AS TOTAL, COUNT(CAMPO) AS PREENCHIDOS FROM TABELA;
```

`COUNT(coluna)` ignora nulos. Uma linha responde "vale a pena investigar isso?" antes de gastar
tempo com amostras.

Aplicado por TOP, revela a que fluxo o campo pertence:

```sql
SELECT CAB.CODTIPOPER,
       COUNT(*) AS QTD_NO_PORTAL,
       COUNT(CAB.AD_VALORFRETE) AS COM_VALOR
  FROM TGFIXN IXN
  JOIN TGFCAB CAB ON CAB.NUNOTA = IXN.NUNOTA
 GROUP BY CAB.CODTIPOPER
 ORDER BY QTD_NO_PORTAL DESC;
```

### Descobrir nomes de coluna em vez de adivinhar

Três erros `ORA-00904` foram gastos chutando nomes (`VLRDESC` na TGFCAB, `EVENTO` na TSIEVE).

```sql
SELECT COLUMN_NAME, DATA_TYPE
  FROM ALL_TAB_COLUMNS
 WHERE TABLE_NAME = 'TGFCAB'
   AND COLUMN_NAME LIKE '%FRETE%'
 ORDER BY COLUMN_ID;
```

E para achar o campo por trás de um rótulo da tela, exportar uma linha inteira (`SELECT *`) e
procurar a coluna que contém o valor visível. Foi assim que se achou o `AD_VALORFRETE`: o
rótulo era "Valor do Frete" e o valor na tela, 290,84.

### `DUMP` para CLOB com conteúdo inesperado

```sql
SELECT DUMP(DBMS_LOB.SUBSTR(DOCSREF, 50, 1)) AS BYTES FROM TGFIXN WHERE NUARQUIVO = 189;
```

Retornou `60,100,111,99,115,82,101,102,62...` = `<docsRef>`. Revelou que o campo era XML e não
texto puro — o que explicava todos os joins vazios.

### Tipos de coluna nem sempre são o óbvio

`SERIENOTA` é VARCHAR, não número. A base contém séries `-`, `U`, `*` além dos numéricos. Filtrar
com `SERIENOTA = 3` retorna vazio; o correto é `SERIENOTA = '3'`.

---

## 9. Consultas de referência

```sql
-- Estrutura de uma tabela
SELECT COLUMN_NAME, DATA_TYPE FROM ALL_TAB_COLUMNS
 WHERE TABLE_NAME = 'TGFIXN' ORDER BY COLUMN_ID;

-- Descrição de TOPs (chave composta com DHALTER)
SELECT CODTIPOPER, DESCROPER FROM TGFTOP T
 WHERE CODTIPOPER IN (1728, 1722, 3100, 3200)
   AND DHALTER = (SELECT MAX(T2.DHALTER) FROM TGFTOP T2
                   WHERE T2.CODTIPOPER = T.CODTIPOPER);

-- Distribuição de séries por empresa
SELECT SERIENOTA, CODEMP, COUNT(*) QTD, MIN(NUMNOTA) MENOR, MAX(NUMNOTA) MAIOR
  FROM TGFCAB WHERE SERIENOTA IS NOT NULL
 GROUP BY SERIENOTA, CODEMP ORDER BY QTD DESC;

-- Extrair chave de acesso do DOCSREF
SELECT NUARQUIVO, REGEXP_SUBSTR(DBMS_LOB.SUBSTR(DOCSREF, 300, 1), '[0-9]{44}') AS CHAVE
  FROM TGFIXN WHERE TIPO = 'C' AND DOCSREF IS NOT NULL
 FETCH FIRST 10 ROWS ONLY;

-- Extrair frete direto do XML do documento (alternativa não usada)
SELECT NUNOTA,
       REGEXP_SUBSTR(DBMS_LOB.SUBSTR(XML, 4000, DBMS_LOB.INSTR(XML, '<vFrete>')),
                     '<vFrete>([0-9.]+)</vFrete>', 1, 1, NULL, 1) AS FRETE_XML
  FROM TGFIXN WHERE XML IS NOT NULL AND DBMS_LOB.INSTR(XML, '<vFrete>') > 0
 FETCH FIRST 20 ROWS ONLY;
```

---

## 10. Limitação conhecida

`AD_VALORFRETE` é preenchido no fluxo de venda interno (TOPs 1728, 1722, 3100, 3200, 1761, 1755,
1107). Notas faturadas fora do ERP — série 3, TOPs 1234 e 1011 — não têm o valor.

Para esses casos a coluna virá em branco, o que é informação legítima (pendente de cadastro),
mas significa que a conferência não cobre 100% dos CT-e. Dos 1.739 CT-e dos últimos 30 dias, 266
têm contraparte com valor.

Se for necessário cobrir o restante, o valor de referência teria que vir de fora do ERP — tarifas
do marketplace ou Intelipost — o que é escopo de integração, não de campo calculado.
