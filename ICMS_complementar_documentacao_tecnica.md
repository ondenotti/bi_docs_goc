# ICMS da nota complementar no custo, documentação técnica

Para quem dá manutenção na fato de compras, no script SQL ou no modelo do Power BI.
Última revisão: 06/11/2026.

- **Fato:** `fcompra_scriptSQL.txt`, seção "AJUSTES DE 06/11/2026: ICMS DA NOTA COMPLEMENTAR E REPROCESSAMENTO DO CUSTO" do cabeçalho
- **Colunas afetadas:** `custo_pela_composicao` e `obs_composicao` (quatro textos). Já existiam `cod_notaint_complementar` e `ICMS_nota_complementar`
- **Banco:** PostgreSQL, ERP Solution, acesso somente leitura

---

## 1. O problema

A nota complementar de entrada aponta para a compra original em
`notass.e62notacomplementar`. Ela não tem quantidade nem valor de mercadoria,
só imposto. O Solution grava o ICMS dela em `movext.e51vlicms` da própria
complementar, e também copia o valor para a linha **original**, em
`e51icmsvalornfcomplementar`.

O crédito é contabilizado sempre igual: **D 47 ICMS a recuperar / C 2192 ICMS
de outras entradas e saídas**. O que varia é o **custo** da linha original
(`e51vtotal`):

| Situação | `e51vtotal` da linha original |
|---|---|
| Custo reprocessado **depois** da complementar | já sem o ICMS da complementar |
| Sem reprocessamento depois da complementar | igual ao do lançamento da compra |

O reprocessamento é a rotina de custo médio do Solution. Ela deixa um carimbo
na linha, em `e51campo_livre_c15`:

```
ESTOQUE.ESTCUSTOMEDIOREPROCESSAR  03/07/2025 07:43:20
```

Linhas mais antigas têm o mesmo carimbo sem o prefixo `ESTOQUE.`. O carimbo
guarda só o **último** reprocessamento da linha.

---

## 2. A regra no script

### CTE `nota_complementar`

Soma por (nota original, produto) as complementares de **entrada**, e agora
guarda também quando a última delas entrou:

```sql
SELECT nc.e01codigo, nc.e35filial,
       nc.e62notacomplementar        AS notaint_original,
       mc.e18codpro,
       MIN(nc.e62notaint)            AS notaint_complementar,
       SUM(mc.e51vlicms)             AS icms_complementar,
       MAX(mc.e51cadastrodatahora)   AS dt_cadastro_complementar
  FROM notass nc
  JOIN codope oc ON oc.e22codope = nc.e22codope
  JOIN movext mc ON mc.e62notaint = nc.e62notaint
 WHERE COALESCE(nc.e62notacomplementar, 0) <> 0
   AND oc.e22tipoes = 'E'
   AND mc.e51situa = ''
 GROUP BY 1, 2, 3, 4
```

### Em `base` (uso interno, não sai no select final)

```sql
ncp.dt_cadastro_complementar,
CASE WHEN mv.e51campo_livre_c15 LIKE '%ESTCUSTOMEDIOREPROCESSAR%' THEN
    to_timestamp(substring(mv.e51campo_livre_c15
                           from '[0-9]{2}/[0-9]{2}/[0-9]{4} [0-9]{2}:[0-9]{2}:[0-9]{2}'),
                 'DD/MM/YYYY HH24:MI:SS')::timestamp
END as dt_reprocessamento_custo,
```

### `custo_pela_composicao`, em `fcompra`

A composição de sempre, com um termo a mais no fim. Ele subtrai o ICMS da
complementar **só** quando o reprocessamento veio depois dela:

```sql
- CASE WHEN base.cod_notaint_complementar IS NOT NULL
        AND base.dt_reprocessamento_custo > base.dt_cadastro_complementar
       THEN COALESCE(base.ICMS_nota_complementar, 0) ELSE 0 END
```

### `obs_composicao`, em `fcompra_final`

A mesma condição, na mesma ordem dos outros motivos. O texto antigo "Custo
ajustado por nota complementar" deixou de existir.

Quando a complementar tem ICMS zero, o campo `e51icmsvalornfcomplementar` da
linha original separa os dois casos. Em `base` ele sai como
`icms_complementar_na_original`, para uso interno:

| ICMS na complementar | `e51icmsvalornfcomplementar` na original | Caso |
|---|---|---|
| zero | maior que zero | complementar de ICMS escriturada sem ICMS |
| zero | zero | complemento de valor |

```sql
WHEN fc.cod_notaint_complementar IS NOT NULL
 AND COALESCE(fc.ICMS_nota_complementar, 0) = 0
 AND COALESCE(fc.icms_complementar_na_original, 0) > 0
     THEN 'Nota complementar de ICMS lançada sem ICMS: crédito não tomado, não altera o custo'
WHEN fc.cod_notaint_complementar IS NOT NULL
 AND COALESCE(fc.ICMS_nota_complementar, 0) = 0
     THEN 'Nota complementar de valor, sem ICMS: não altera o custo'
WHEN fc.cod_notaint_complementar IS NOT NULL
 AND fc.dt_reprocessamento_custo > fc.dt_cadastro_complementar
     THEN 'Nota complementar: ICMS abatido do custo (custo reprocessado depois dela)'
WHEN fc.cod_notaint_complementar IS NOT NULL
     THEN 'Nota complementar: ICMS não abatido do custo (custo não reprocessado depois dela)'
```

As linhas com nota complementar **não entram** no teste do frete do CT-e
(`testa_frete_custeio = 0`), então as duas regras não se misturam.

---

## 3. Exemplo do ICMS abatido: 505086 × 512769

- **Compra:** filial 11 (Teresina/PI), NF 17990 (notaint **505086**), operação 1, item `8094848`, 6 unidades.
- **Complementar:** NF 18042 (notaint **512769**), operação 240, `finNFe` = 2 no XML.

| Evento | Data/hora | Fonte |
|---|---|---|
| Item da compra gravado | 02/06/2025 11:32:39 | `e51cadastrodatahora` |
| Complementar gravada | 02/07/2025 22:20:33 | `e51cadastrodatahora` da 512769 |
| Custo reprocessado | 03/07/2025 07:43:20 | `e51campo_livre_c15` = `e51alteracaodatahora` |

| Campo da linha original | Valor |
|---|---|
| `e51totnota2` | 8.843,34 |
| `e51vlicms` / `e51vlrpis` / `e51vlrcofins` | 0,00 / 0,00 / 0,00 (CST 60, PIS/COFINS 73) |
| `e51icmsvalornfcomplementar` | 2.401,79 (base 10.674,64 × 22,5%) |
| **`e51vtotal`** | **6.441,55** = 8.843,34 − 2.401,79 |
| `custo_pela_composicao` | 6.441,55 (antes do ajuste: 8.843,34) |

As outras 7 compras da Mutum de Teresina com complementar lançada na noite de
02/07 foram reprocessadas na mesma manhã, entre 07:41 e 07:44. Todas abateram.

---

## 4. Exemplo do ICMS não abatido: 502776 × 512835

- **Compra:** filial 5 (Balsas/MA), NF 63914 (notaint **502776**), operação 1, item `1100BR`, 1 unidade.
- **Complementar:** NF 64062 (notaint **512835**), operação 240, `finNFe` = 2 no XML.

| Evento | Data/hora | Fonte |
|---|---|---|
| Item da compra gravado (e nunca mais alterado) | 23/05/2025 20:32:30 | `e51cadastrodatahora` = `e51alteracaodatahora` |
| Complementar gravada | 03/07/2025 08:46:37 | `e51cadastrodatahora` da 512835 |
| Custo reprocessado | nunca | `e51campo_livre_c15` vazio |

| Campo da linha original | Valor |
|---|---|
| `e51totnota2` | 11.510,68 |
| `e51vlicms` | 460,43 (4%) |
| `e51vlrpis` / `e51vlrcofins` | 182,33 / 839,82 |
| `e51icmsvalornfcomplementar` | 2.187,03 (base 11.510,68 × 19%, complemento até 23%) |
| **`e51vtotal`** | **10.028,10**, sem abater a complementar |
| `custo_pela_composicao` | 10.028,10 |

O crédito da 512835 foi contabilizado: D 47 / C 2192, R$ 4.279,32 nos dois
itens dela. É o mesmo lançamento da 512769.

---

## 5. Exemplo da complementar de ICMS lançada sem ICMS: 504632 × 520586

- **Compra:** filial 12 (São Mateus do Maranhão/MA), NF 26365 (notaint **504632**).
- **Complementar:** NF 26450 (notaint **520586**), operação 240.

| | Valor |
|---|---|
| ICMS no XML da complementar (`ICMSTot/vICMS`, `finNFe` = 2) | 1.083,06 |
| `e51icmsvalornfcomplementar` nas linhas da compra original | 328,55 + 754,51 = 1.083,06 |
| `e51vlicms` nas linhas da complementar | **0,00** |
| Livro fiscal (`lif_tab422.t422vlricms`) | **0,00** |
| Contabilidade (`crp_tab335`) | nenhum lançamento |

O Solution sabe o valor, porque o copiou para a compra original, mas a
complementar foi escriturada sem ICMS. O custo não muda, e o crédito não foi
tomado.

### Os dois casos de complementar com ICMS zero

| `obs_composicao` | Origem | Notas | ICMS no XML | ICMS no Solution |
|---|---|---|---|---|
| ICMS lançada sem ICMS | 240 com ICMS zero (filiais 11 e 12, jun–jul/2025) | 8 | 2.320,00 | 0,00 |
| ICMS lançada sem ICMS | 240 com ICMS parcial (520595, filial 12) | 1 | 99,54 | 79,55 |
| Complemento de valor | 117 "Complementar de valores" (filiais 5, 7 e 8, 2023, lubrificante) | 3 | não se aplica | 0,00 |

- **ICMS das complementares que não foi creditado:** 2.320,00 + 19,99 = **R$ 2.339,99**. O fiscal precisa conferir.
- **Na 520595**, a linha do item `5165505` caiu em "lançada sem ICMS". A do item `4570799`, que tem os 79,55, caiu em "não abatido".
- **As três da operação 117** complementam preço, não imposto. O valor complementado (ex. NF 1105658, R$ 238,72, D 350000 Compras a distribuir / C 726) também **não** entra em `e51vtotal`: a linha da complementar tem `e51totnota2` zero. Esse efeito não é tratado pela regra atual.

---

## 6. Validação (06/11/2026, base inteira)

**Script novo × versão anterior**
- Mesmo número de linhas, com os mesmos `pk_id`.
- `custo_medio` e `Frete_ctc_divergencia_custeio` saíram idênticos.
- `custo_pela_composicao` mudou só nas 8 linhas "abatido". Nenhuma devolução está ligada a essas compras, então `composicao_compra_origem` não muda.
- A divisão do caso de ICMS zero em dois textos mexeu só no `obs_composicao` dessas 18 linhas.

**As 120 linhas com nota complementar**

| `obs_composicao` | Linhas | Notas de compra | ICMS da complementar | `custo_diferenca_composicao` |
|---|---|---|---|---|
| ICMS abatido do custo | 8 | 8 | 4.090,82 | 0,00 |
| ICMS não abatido do custo | 94 | 66 | 48.145,76 | 0,00 |
| Complementar de ICMS lançada sem ICMS | 15 | 9 | 0,00 | 0,00 |
| Complemento de valor | 3 | 3 | 0,00 | 0,00 |

**A regra contra o dado.** Contado por par (complementar, produto), com ICMS:
- Com reprocessamento **depois** da complementar: **8**, e todos abateram.
- Sem reprocessamento depois: **97**, e nenhum abateu. Em 31 deles houve reprocessamento, mas **antes** da complementar.

**Onde está o não abatido:**

| Ano | Filial | Operação | Notas | ICMS |
|---|---|---|---|---|
| 2021 | 3 Lagoa da Confusão | 117 | 15 | 13.457,86 |
| 2025 | 5 Balsas | 240 | 2 | 7.247,20 |
| 2025 | 11 Teresina | 240 | 19 | 4.815,11 |
| 2025 | 12 São Mateus do Maranhão | 240 | 30 | 22.625,59 |

---

## 7. Pontos para o fiscal

**Complementares sem ICMS escriturado:** são R$ 2.339,99 em 9 notas da Mutum
de jun–jul/2025, listadas na seção 5.

**Tratamento diferente para o mesmo tipo de item:** na 505086 o item é CST 60
(com ST) e a complementar foi creditada a 22,5%. Na 504632 a complementar foi
escriturada sem ICMS. O fiscal precisa dizer qual tratamento é o correto.

**Complementos maiores que o item:** em alguns casos o ICMS da complementar é
maior que o próprio custo da peça.
- **Item `378247` da 502776:** custo de 1,53 e complementar de 2.092,29, sobre uma base de 9.096,92.
- **Item `84043491` da 505064:** depois do abatimento, o custo ficou **negativo**, em −102,56.

---

## 8. Limites e manutenção

**O carimbo guarda só o último reprocessamento.** Se uma linha for reprocessada
depois por outro motivo, ela passa a "abatido". Isso está certo se o
reprocessamento sempre recalcula com as complementares que já existem, que é o
que a base mostra: 8 de 8.

**Mais de uma complementar no mesmo produto.** A CTE soma o ICMS e usa a data
da **última**. Se o reprocessamento caiu entre duas complementares, a linha sai
"não abatido" e a diferença aparece em `custo_diferenca_composicao`. Na base de
hoje isso não acontece.

**Produto repetido na nota original.** O ICMS da complementar se repete em
cada linha do produto, e a composição abateria duas vezes. Nenhuma linha
"abatido" de hoje está nessa situação.

**`fcompra_scriptSQL_PowerQuery.txt`.** Ele embute o mesmo SQL, então precisa
ser atualizado a partir do script.

### Consulta para investigar uma compra

```sql
SELECT m.e62notaint, m.e18codpro, m.e51vtotal, m.e51icmsvalornfcomplementar,
       m.e51cadastrodatahora, m.e51alteracaodatahora, m.e51campo_livre_c15,
       c.e62notaint AS complementar, c.e22codope,
       (SELECT sum(x.e51vlicms) FROM movext x
         WHERE x.e62notaint = c.e62notaint AND x.e18codpro = m.e18codpro) AS icms_compl,
       (SELECT min(x.e51cadastrodatahora) FROM movext x
         WHERE x.e62notaint = c.e62notaint)                             AS compl_gravada
  FROM movext m
  JOIN notass c ON c.e62notacomplementar = m.e62notaint
 WHERE m.e62notaint = 502776
   AND EXISTS (SELECT 1 FROM movext x
                WHERE x.e62notaint = c.e62notaint AND x.e18codpro = m.e18codpro)
 ORDER BY m.e18codpro;
```
