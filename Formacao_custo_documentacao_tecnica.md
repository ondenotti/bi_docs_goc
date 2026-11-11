# Formação do custo das compras, documentação técnica

Para quem dá manutenção na fato de compras, no script SQL ou no modelo do Power BI.
Última revisão: 10/11/2026.

- **Fato:** `fcompra_scriptSQL.txt`. A regra está nas seções do cabeçalho "COMPOSICAO DO CUSTO (29/10/2026)" e "AJUSTES DE ..." que vêm depois dela (03/11, 06/11, 09/11 e 10/11/2026)
- **Colunas principais:** `custo_pela_composicao`, `custo_medio`, `custo_diferenca_composicao`, `obs_composicao`
- **Visual:** matriz de formação do custo, com as parcelas do custo em linhas
- **Banco:** PostgreSQL, ERP Solution, acesso somente leitura

---

## 1. O que a matriz prova

O Solution grava em `movext.e51vtotal` o valor que cada linha de nota levou ao
estoque. No fcompra esse valor é a coluna `custo_medio`. O Solution não grava
de que ele é feito.

O script refaz a conta a partir dos campos da própria linha e entrega o
resultado em `custo_pela_composicao`. A matriz abre essa conta em parcelas.
A prova é a igualdade:

```text
Σ parcelas da matriz  =  Σ custo_pela_composicao  =  Σ custo_medio
```

- **A primeira igualdade** é de construção: as parcelas são exatamente os
  termos da fórmula do `custo_pela_composicao`. Se ela não fechar, alguma
  medida está com filtro ou sinal errado.
- **A segunda** é o teste de verdade: a composição explica o custo do Solution.
  O que sobra está, linha a linha, em
  `custo_diferenca_composicao = custo_medio − custo_pela_composicao`.

> `custo_medio` e o "Custo Médio" da matriz são **valores totais em R$** da
> movimentação, não o custo unitário. O nome vem do campo do Solution, que
> alimenta o custo médio da peça.

---

## 2. A fórmula no script

O cálculo tem duas etapas. Primeiro, no CTE `fcompra`, a composição da própria
nota (coluna interna `custo_composicao_sem_frete_div`):

```sql
CASE WHEN base.atualiza_estoque = 'Não' THEN 0
     WHEN base.fk_tipo_lancamento IN (27, 34, 38) THEN base.custo_medio
ELSE
    COALESCE(base.saldo_de_compra, 0)
  + COALESCE(base.ICMSTGR, 0)
  - COALESCE(base.ICMS, 0)
  - COALESCE(base.ICMS_credito_simples, 0)
  + COALESCE(base.Frete_vinculado_a_nota_do_produto, 0)
  - COALESCE(base.Frete_icms_vinculado_a_nota_do_produto, 0)
  - COALESCE(base.Frete_pis_vinculado_a_nota_do_produto, 0)
  - COALESCE(base.Frete_cofins_vinculado_a_nota_do_produto, 0)
  - COALESCE(base.PIS, 0)
  - COALESCE(base.COFINS, 0)
  - CASE WHEN base.cod_notaint_complementar IS NOT NULL
          AND base.dt_reprocessamento_custo > base.dt_cadastro_complementar
         THEN COALESCE(base.ICMS_nota_complementar, 0) ELSE 0 END
END
```

Depois, no CTE `fcompra_final`, ela soma os erros do Solution já mapeados:

```sql
custo_composicao_sem_frete_div
+ COALESCE(troca_de_cst_73_53, 0)
+ COALESCE(pis_cofins_alterado_apos_custeio, 0)
+ Frete_ctc_divergencia_custeio            -- caso 1, 2 ou 3; senão 0
= custo_pela_composicao
```

A ordem do `CASE` importa:
1. **Linha que não mexe no estoque** (`e51atuest = 0`): custo zero. Uso e
   consumo (6, 108, 109), 111, energia elétrica (171), devoluções 33 e 126 etc.
2. **Tipos 27, 34 e 38**: o custo é o próprio `custo_medio` (seção 5).
3. **Todo o resto** (na prática, o tipo 26): a composição pela nota.

---

## 3. Cada parcela da matriz

Todas as parcelas do bloco da nota valem **só para o tipo 26** e para linhas
com `atualiza_estoque = 'Sim'`. Para o tipo 26 as colunas do fcompra vêm
positivas. A medida multiplica por −1 as que são crédito.

| Linha da matriz | Coluna do fcompra | Campo do Solution (`movext`) | Sinal na matriz |
|---|---|---|---|
| (+) Compra | `saldo_de_compra` | `e51totnota2` | + |
| (+) ICMS ST GR | `ICMSTGR` | `e51vlicmstgr` | + |
| (−) ICMS | `ICMS` | `e51vlicms` | − |
| (−) PIS | `PIS` | `e51vlrpis` | − |
| (−) COFINS | `COFINS` | `e51vlrcofins` | − |
| (+) Frete | `Frete_vinculado_a_nota_do_produto` | `e51frete_ctc` | + |
| (−) ICMS Frete | `Frete_icms_vinculado_a_nota_do_produto` | `e51vlicms_frete` | − |
| (−) PIS Frete | `Frete_pis_vinculado_a_nota_do_produto` | `e51valor_pis_ctc` | − |
| (−) COFINS Frete | `Frete_cofins_vinculado_a_nota_do_produto` | `e51valor_cofins_ctc` | − |
| (−) ICMS NF Complementar | `ICMS_nota_complementar`, **só** com `obs_composicao` = abatido | `e51vlicms` da nota complementar | − |
| (−) ICMS Simples Nacional | `ICMS_credito_simples` | `e51icmscredsimplesnacionalvalor` | − |
| (+) Frete CTC Divergência - Erro Solution | `Frete_ctc_divergencia_custeio` | calculada | como vem |
| (+) PIS/COFINS Troca CST | `troca_de_cst_73_53` | calculada | como vem |
| (+) PIS/COFINS Alterado Após Custeio | `pis_cofins_alterado_apos_custeio` | calculada | como vem |
| (−) Custo da Devolução | `custo_medio` do tipo 27 | `e51vtotal` | como vem (já negativo) |
| (+) Custo do Estorno da Devolução | `custo_medio` do tipo 34 | `e51vtotal` | como vem |
| (+) Custo Outras Entradas | `custo_medio` do tipo 38 | `e51vtotal` | como vem |
| **(=) Custo Médio Calculado** | `custo_pela_composicao` | | soma das parcelas |
| Custo Médio Solution | `custo_medio` | `e51vtotal` | como vem |
| ICMS NF Complementar (Não Reprocessado no Custo) | `ICMS_nota_complementar`, **só** com `obs_composicao` = não abatido | `e51vlicms` da nota complementar | − (fora da soma) |

### 3.1 (+) Compra: `saldo_de_compra`

`e51totnota2` é o total da linha na nota, já pronto no Solution:

```text
e51vlrprod − e51totdesc_mercadorias + e51vlripi + e51vlicmstnf + e51frete_nf + e51seguro
```

Conferido nas compras de peças (`e51_e22tipolc = '1'`, grupo 400–799) de 2025
em diante: **99,86%** das 88.711 linhas fecham com essa soma em até 2
centavos. Nas 124 que sobram, o `e51totnota2` é maior, com mediana de 90
centavos. Provavelmente são outras despesas acessórias da nota, mas isso não
foi conferido. Não importa para a composição, que usa o `e51totnota2` pronto.

- O IPI e o ICMS ST da nota ficam no custo porque não geram crédito.
- O frete **destacado na nota** (`Frete_destacado_na_propria_nota_do_produto`,
  `e51frete_nf`) já está aqui dentro. **Não** some essa coluna de novo na matriz.
- Na devolução (tipolc 7) o script inverte o sinal de `saldo_de_compra` e de
  todas as colunas de valor. Esse é mais um motivo para as parcelas da nota
  excluírem o tipo 27.

### 3.2 (+) ICMS ST GR: `ICMSTGR`

ICMS ST recolhido em guia, que não está no total da nota. Linhas com ST GR
quase nunca têm ICMS ST destacado na nota (341 de 136.541) e quase todas são
de compra interestadual (136.539 de 136.541). São muito comuns: no tipo 26
que mexe no estoque, cerca de 41% das linhas têm ST GR.

### 3.3 (−) ICMS: `ICMS`, e não `ICMS_da_nota`

`e51vlicms` é o ICMS **creditado**. `e51vlicms_danota` (`ICMS_da_nota`) é o
destacado. Peça com ST quase sempre tem ICMS destacado e crédito zero. No tipo
26 que mexe no estoque, de 2025 em diante:

| Linhas | Quantas | ICMS destacado | ICMS creditado |
|---|---|---|---|
| Com ICMS ST (na nota ou em guia) | 52.395 | 3.022.350,46 | 30.076,53 (em 917 linhas) |
| Sem ICMS ST | 38.159 | 18.438.593,00 | 17.934.200,02 |

Usar `ICMS_da_nota` quebraria a composição. As 917 exceções, como a nota
interna 493989 (ICMS ST 453,80 na nota e crédito de 45,48), também fecham,
porque a composição usa o que foi creditado, seja qual for o motivo.

### 3.4 (−) PIS e (−) COFINS

`e51vlrpis` e `e51vlrcofins` já são só o crédito: CST 50 a 56 vem com valor,
CST 70 a 75 vem com zero. Não precisa de regra, é só subtrair. As colunas
`cst_pis_entrada` e `pis_cofins_trib_cadastro` saem só para explicar.

### 3.5 (+) Frete e os créditos do frete

`e51frete_ctc` é o rateio do CT-e na linha, com os créditos em
`e51vlicms_frete`, `e51valor_pis_ctc` e `e51valor_cofins_ctc`. São os valores
**de hoje**. Se o CT-e mudou depois do custeio, a diferença aparece na
`Frete_ctc_divergencia_custeio` (seção 3.8).

### 3.6 (−) ICMS NF Complementar e a linha informativa

`ICMS_nota_complementar` é a soma de `e51vlicms` das notas complementares de
entrada que apontam para esta nota (`notass.e62notacomplementar`), no mesmo
produto (CTE `nota_complementar`).

O Solution só abate esse ICMS do custo quando a rotina de reprocessamento do
custo médio rodou na linha **depois** da complementar (carimbo
`ESTCUSTOMEDIOREPROCESSAR` em `e51campo_livre_c15`, comparado com
`MAX(e51cadastrodatahora)` da complementar). A coluna, porém, vem preenchida
nos dois casos. Por isso a matriz separa as duas situações pelo
`obs_composicao` (as medidas estão na seção 4.3):

| `obs_composicao` | Onde entra na matriz |
|---|---|
| *Nota complementar: ICMS abatido do custo (custo reprocessado depois dela)* | (−) ICMS NF Complementar |
| *Nota complementar: ICMS não abatido do custo (custo não reprocessado depois dela)* | ICMS NF Complementar (Não Reprocessado no Custo), **fora da soma** |
| *Nota complementar de ICMS lançada sem ICMS...* e *Nota complementar de valor, sem ICMS...* | Em nenhuma das duas: o ICMS é zero |

Separar pelo texto é seguro aqui. No `CASE` do `obs_composicao`, os textos de
complementar só perdem para "não entra no estoque" e para os tipos 27, 34 e
38, e essas linhas já estão fora das parcelas.

Detalhes: [ICMS complementar no custo (Técnico)](ICMS_complementar_documentacao_tecnica.md).

### 3.7 (−) ICMS Simples Nacional: `ICMS_credito_simples`

Crédito de ICMS que o fornecedor do Simples informa no XML (pCredSN /
vCredICMSSN, art. 23 da LC 123). Não vem em `e51vlicms`, mas o Solution abate
do custo. Exemplo: nota interna 589186 (filial 13, 07/04/2026), 1 unidade:
741,00 − 29,89 = **711,11**, igual ao `custo_medio`.

### 3.8 Os três erros do Solution já mapeados

São colunas calculadas que medem um erro de custeio. A matriz as mostra como
vêm, porque elas já têm o sinal do erro (+ custo do Solution mais alto, −
mais baixo):

| Coluna | Regra resumida | Documento |
|---|---|---|
| `Frete_ctc_divergencia_custeio` | CT-e rateado, vinculado ou desvinculado depois do custeio | [Frete do CT-e (Técnico)](Frete_CTe_documentacao_tecnica.md) |
| `troca_de_cst_73_53` | CST da pré-nota 73 e CST de hoje 53: o PIS/COFINS de hoje ficou dentro do custo | [PIS/COFINS (Técnico)](Troca_CST_PIS_COFINS_documentacao_tecnica.md) |
| `pis_cofins_alterado_apos_custeio` | PIS/COFINS de hoje diferente do usado no custeio (pré-nota ou contabilidade) | [PIS/COFINS (Técnico)](Troca_CST_PIS_COFINS_documentacao_tecnica.md) |

As três ficam **NULL** quando a linha não tem o erro. Uma linha nunca tem
`troca_de_cst_73_53` e `pis_cofins_alterado_apos_custeio` ao mesmo tempo.

---

## 4. As medidas no Power BI

### 4.1 O filtro comum

Todas as medidas da matriz partem de:
- `dtipolancamento[agrupamento] = "Compras"`;
- `fcompra[atualiza_estoque] = "Sim"`.

Linhas com `atualiza_estoque = "Não"` têm composição zero, mas trazem valor
nas colunas de imposto e de compra. No tipo 26 que não mexe no estoque, por
exemplo, são R$ 671 mil de COFINS em toda a base. Sem esse filtro, as parcelas
não fecham com o `custo_pela_composicao`.

### 4.2 Parcela da nota (exemplo: COFINS)

```dax
cofins_compra =
CALCULATE (
    SUM ( fcompra[cofins] ),
    KEEPFILTERS ( dtipolancamento[agrupamento] = "Compras" ),
    // 27 = devolução de compra, 34 = estorno da devolução, 38 = outras entradas:
    // entram pelo custo_medio, então impostos e fretes dessas notas não entram.
    KEEPFILTERS ( NOT dtipolancamento[pk_tipo_lancamento] IN { 27, 34, 38 } ),
    KEEPFILTERS ( fcompra[atualiza_estoque] = "Sim" )
) * -1
```

- **Valem o mesmo filtro e o mesmo × −1:** ICMS, PIS, ICMS Frete, PIS Frete,
  COFINS Frete e ICMS Simples Nacional.
- **Sem o × −1:** Compra, ICMS ST GR e Frete.
- **Também sem o × −1:** as três colunas de erro, que já têm o sinal.
- **ICMS NF Complementar:** recebe ainda o filtro do `obs_composicao`
  (seção 4.3).

`KEEPFILTERS` faz a condição se somar à segmentação do relatório, em vez de
substituí-la.

A parcela **(+) Compra** usa a medida `[saldo_de_compra_personalizado_]`, que
é `SUM ( fcompra[saldo_de_compra] )`, com o mesmo filtro comum e a mesma
exclusão dos tipos 27, 34 e 38. Por isso ela é **só compra**: a nota de
devolução, de estorno e de outras entradas não entra nela.

```dax
Compra =
CALCULATE (
    [saldo_de_compra_personalizado_],
    KEEPFILTERS ( dtipolancamento[agrupamento] = "Compras" ),
    KEEPFILTERS ( NOT dtipolancamento[pk_tipo_lancamento] IN { 27, 34, 38 } ),
    KEEPFILTERS ( fcompra[atualiza_estoque] = "Sim" )
)
```

### 4.3 As duas medidas do ICMS da nota complementar

As duas somam a mesma coluna, `ICMS_nota_complementar`, com o filtro comum e
a exclusão dos tipos 27, 34 e 38. O que as separa é o texto do
`obs_composicao`:

```dax
icms_compra_credito_nota_complementar =      // (-) ICMS NF Complementar
CALCULATE (
    SUM ( fcompra[icms_nota_complementar] ),
    KEEPFILTERS ( dtipolancamento[agrupamento] = "Compras" ),
    KEEPFILTERS ( NOT dtipolancamento[pk_tipo_lancamento] IN { 27, 34, 38 } ),
    KEEPFILTERS ( fcompra[atualiza_estoque] = "Sim" ),
    KEEPFILTERS ( fcompra[obs_composicao]
        = "Nota complementar: ICMS abatido do custo (custo reprocessado depois dela)" )
) * -1

icms_compra_credito_nota_complementar_nao_reprocessado =
                    // ICMS NF Complementar (Não Reprocessado no Custo), fora da soma
CALCULATE (
    SUM ( fcompra[icms_nota_complementar] ),
    KEEPFILTERS ( dtipolancamento[agrupamento] = "Compras" ),
    KEEPFILTERS ( NOT dtipolancamento[pk_tipo_lancamento] IN { 27, 34, 38 } ),
    KEEPFILTERS ( fcompra[atualiza_estoque] = "Sim" ),
    KEEPFILTERS ( fcompra[obs_composicao]
        = "Nota complementar: ICMS não abatido do custo (custo não reprocessado depois dela)" )
) * -1
```

- **As duas multiplicam por −1.** A segunda também aparece negativa: é o
  crédito que **deveria** ter saído do custo e não saiu. Ela fica abaixo do
  Custo Médio Calculado e não entra na soma.
- **Atenção ao texto.** O filtro compara o texto exato do `obs_composicao`. Se
  o texto mudar no script, as duas medidas passam a dar zero sem nenhum aviso.
  Qualquer mudança nesses dois textos precisa ser repetida aqui.

### 4.4 Parcelas pelo custo do Solution

```dax
custo_outras_entradas =
CALCULATE (
    SUM ( fcompra[custo_medio] ),
    KEEPFILTERS ( dtipolancamento[agrupamento] = "Compras" ),
    // 38 = outras entradas (54, 60, 211): o Solution lança pelo custo médio
    // da filial, não pelo valor da nota.
    KEEPFILTERS ( dtipolancamento[pk_tipo_lancamento] = 38 ),
    KEEPFILTERS ( fcompra[atualiza_estoque] = "Sim" )
)
```

A devolução usa o tipo 27 e o estorno, o tipo 34. O `custo_medio` da devolução
já vem negativo, então a medida **não** multiplica por −1: ela entra somando no
total.

### 4.5 Medida de conferência

Vale ter uma medida que deve dar sempre zero:

```text
[Custo Médio Calculado (soma das parcelas)]
  − CALCULATE ( SUM ( fcompra[custo_pela_composicao] ), <filtro comum> )
```

Se não der zero, alguma parcela está sem filtro, com sinal trocado ou somando
uma coluna a mais. Por exemplo, o frete destacado na nota já está em Compra e
não pode ser somado de novo.

---

## 5. Por que 27, 34 e 38 entram pelo `custo_medio`

Nessas três, a nota não explica o valor que o Solution movimentou.

| Tipo | Operações | O que o Solution faz | Evidência |
|---|---|---|---|
| **27**, devolução de compra (tipolc 7) | 33, 42, 115, 126, 166, 180, 217 | Baixa pelo custo da compra de origem (devolução parcial) ou pelo médio da filial (quando devolve todo o saldo). Os impostos da nota de devolução vão para a contabilidade (conta 478), não para o estoque | Nas 2.782 linhas que mexem no estoque, a referência que bate com o `e51vtotal` é a origem em 2.496 e o médio em 272 |
| **34**, estorno (tipolc 4) | 66, 87, 186, 269, 274 | 274 (anular NF de saída) entra pelo custo médio; as demais, pela composição da própria nota | `codope.e22custo`: 274 = '1' (custo médio); 66, 87, 186, 269 = '3' (valor líquido) |
| **38**, outras entradas (tipolc 4) | 54, 211 (brindes/doações), 60 (cestas de Natal) | Entra pelo custo médio da filial, não pelo valor da nota; com o médio zerado, entra sem custo | `codope.e22custo` = '1'. No fcompra, '1' só aparece no tipo 38 e na 274 |

Usar o `custo_medio` nos três dá o valor que de fato saiu ou entrou no estoque.
Nos estornos 66, 87, 186 e 269 a composição da própria nota também fecharia
(107 de 107 linhas, em até 5 centavos), então o resultado é o mesmo. Na 274,
nenhuma das 48 linhas fecha pela nota.

### Exemplos

**Devolução, 518195 → compra 509515** (filial 3, item 90349799): a compra entrou
por 98.719,04 (84.706,20 + ICMS ST GR 21.115,04 − PIS 1.266,88 − COFINS
5.835,32). A devolução (operação 166, 28/07/2025) teve ICMS de 3.302,39
destacado e PIS e COFINS estornados. Pela própria nota daria −74.301,61. Saiu
do estoque por **−98.719,04**, o custo da origem. A diferença de 24.417,43 foi
para a conta 478.

**Devolução que não bate com nenhuma referência, 87768** (filial 2,
16/07/2020, produto 73382665, 5 un): pela origem (compra 31927, 14 un por
107,56) seria −38,41. O Solution baixou **−37,90**, e o médio que ficou na
filial foi 8,20. Por isso a matriz usa o que saiu, não uma referência
calculada.

**Estorno, 616744** (operação 274, filial 11, 11/08/2026, item F0CV0D003F5ED):
a nota vale 11.320,17, com ICMS de 2.547,04. Pela nota daria 8.773,13. O
estoque estava zerado (`e51quantidade_existente_ajuste` = 0), e a peça entrou
pelo último médio da filial, **10.059,62** (`e51custo_medio_unitario` da
movimentação anterior).

**Outras entradas, 601345** (NF 63628, operação 211, filial 2, 30/05/2026, item
761759J1BR, 1 un): a nota vale 35.187,93, sem impostos. O estoque estava
zerado, e o último médio era 27.985,38 (compra 570570, de 06/02/2026). Entrou
por **27.985,38** e saiu no mesmo dia por venda, pelo mesmo valor.

### O `obs_composicao` desses tipos

O texto diz qual referência explica o valor que o Solution usou. A coluna
interna `base_custo_devolucao` testa as candidatas com folga de 5 centavos.

| Tipo | `obs_composicao` | Linhas |
|---|---|---|
| 27 | *Devolução: custo da compra de origem* | 2.496 |
| 27 | *Devolução: custo médio da filial* | 227 |
| 27 | *Devolução sem compra ligada: custo médio da filial* | 45 |
| 27 | *Devolução: custo igual à composição da própria nota* | 5 |
| 27 | *Devolução: custo baixado não bate com a compra de origem nem com o médio da filial* | 7 |
| 27 | *Devolução sem compra ligada: custo baixado não bate com o médio da filial nem com a própria nota* | 2 |
| 34 | *Estorno: custo lançado no estoque pelo Solution, sem os impostos da nota* | 155 |
| 38 | *Outras entradas: custo médio da filial lançado pelo Solution, não o valor da nota* | 96 |
| 38 | *Outras entradas: custo médio da filial estava zerado, entrou sem custo* | 937 |

A coluna `custo_devolucao`, com o valor da referência, está **comentada** no
script, em `base` e no select final. Ela repetiria o `custo_medio` e só serve
para provar o `obs_composicao`. Para investigar uma devolução, descomente.

---

## 6. Exemplos do tipo 26, linha a linha

**Nota 91976** (notaint 581754, filial 1, 13/03/2026, operação 1), item
92123058, 1 un, sem ICMS ST:

```text
  saldo_de_compra   8.390,50   (9.988,69 − desconto 1.598,19)
− ICMS                335,62   (= ICMS_da_nota: sem ST, tem crédito)
− PIS                 132,91
− COFINS              612,17
+ Frete CT-e          276,87
− ICMS do frete        33,22
− PIS do frete          4,57
− COFINS do frete      21,04
= custo_pela_composicao  7.527,84   =  custo_medio 7.527,84
```

**Nota 3286411** (notaint 464502, filial 1, 02/01/2025, operação 1), item
47459734, 6 un:

```text
  saldo_de_compra   5.322,23   (5.187,36 + IPI 134,87)
+ ICMSTGR           1.477,13
− ICMS                  0,00   (ICMS_da_nota 212,68, sem crédito)
− PIS                  82,08
− COFINS              378,08
= custo_pela_composicao  6.339,20   =  custo_medio 6.339,20
```

**Nota 168379** (notaint 493153, filial 1, 10/04/2025), item 42577079, 6 un,
com desconto:

```text
  saldo_de_compra     262,20   (245,22 − desconto 36,78 + ICMS ST 53,76)
− ICMS                  0,00   (ICMS_da_nota 25,01, sem crédito)
− PIS                   3,03
− COFINS               13,94
= custo_pela_composicao    245,23   =  custo_medio 245,23
```

---

## 7. Validação

Rodado em 10/11/2026 na base inteira (367.718 linhas). Filtro igual ao da
matriz: `atualiza_estoque = 'Sim'`, o que deixa só os tipos 26, 27, 34 e 38
(335.621 linhas). O tipo 40 (operação 192) só tem linhas que não mexem no
estoque. As parcelas foram calculadas como nas medidas da seção 4, fora do
Power BI, então o teste supõe que esses quatro tipos estão no agrupamento
"Compras" da `dtipolancamento`.

**Primeira igualdade:** em todos os anos, de 2016 a 2026, a soma das parcelas
é igual à soma do `custo_pela_composicao`, no centavo.

**Segunda igualdade, ano a ano:**

| Ano | Linhas | Não fecham (> R$ 0,05) | Custo Médio Calculado | Custo Médio Solution | Solution − Calculado |
|---|---|---|---|---|---|
| 2016 | 7.623 | 249 | 845.936,33 | 844.677,27 | −1.259,06 |
| 2017 | 10.838 | 446 | 26.401.720,37 | 26.180.772,07 | −220.948,30 |
| 2018 | 13.280 | 270 | 59.179.117,24 | 59.188.580,47 | 9.463,23 |
| 2019 | 11.968 | 0 | 60.518.211,11 | 60.518.211,11 | 0,00 |
| 2020 | 35.089 | 3 | 133.669.382,14 | 133.669.448,75 | 66,61 |
| 2021 | 39.835 | 42 | 308.170.166,15 | 308.169.403,28 | −762,87 |
| 2022 | 39.451 | 9 | 498.849.933,82 | 498.850.008,02 | 74,20 |
| 2023 | 44.261 | 17 | 440.796.529,15 | 440.798.404,56 | 1.875,41 |
| 2024 | 41.916 | 4 | 495.302.092,48 | 495.284.684,52 | −17.407,96 |
| 2025 | 56.683 | 0 | 380.749.831,20 | 380.749.831,20 | 0,00 |
| 2026 | 34.677 | 1 | 163.899.877,09 | 163.899.876,63 | −0,46 |

- **No total**, 99,69% das linhas fecham. De 2019 em diante, 99,97% (76 linhas
  de 303.880).
- **As parcelas completas de 2024 e 2025** estão no
  [guia para usuários](Formacao_custo_para_usuarios_do_BI.md), seção 9.

---

## 8. O que ainda não fecha

| O quê | Tamanho | Situação |
|---|---|---|
| **2016–2018**, começo do uso do Solution | 965 linhas, −212.744,13 líquido | Não investigado. As maiores são linhas únicas de operação 1 na filial 1 e 2, em 2017 (ex.: notaint 2342, −68.450,00) |
| **Notaint 379935** (filial 5, 2024, operação 22) | −17.420,00 | Em investigação: a contabilidade tem um frete de 20.800,00 que não aparece na nota, ainda sem explicação |
| **Notaint 373987** | 11,42 | Sem prova contábil ainda |
| **Devoluções que não batem com nenhuma referência** | 9 linhas | Não afetam a matriz (entram pelo `custo_medio`), só o `obs_composicao` |
| **2026** | 1 linha, −0,46 | Arredondamento do caso 1 do frete (nota 607985) |

Qualquer outra diferença expressiva entre Calculado e Solution é situação
nova, e deve ser investigada com `custo_diferenca_composicao` e
`obs_composicao` no nível do item.

---

## 9. Manutenção

- **Erro novo do Solution mapeado:** crie uma coluna própria, some no
  `custo_pela_composicao` em `fcompra_final`, dê um texto no `obs_composicao`
  e crie a parcela na matriz. Nunca embuta o erro em outra parcela.
- **Operação nova no tipo 27, 34 ou 38**, ou nova operação com
  `codope.e22custo = '1'`: confira se ela deve entrar pelo `custo_medio`. Hoje
  a regra é pelo `fk_tipo_lancamento`, não pelo `e22custo`.
- **Coluna nova de imposto ou despesa na nota:** confira se ela já está dentro
  de `e51totnota2` antes de somar. Somar o frete destacado na nota, por
  exemplo, contaria esse frete duas vezes.
- **Depois de qualquer mudança:** rode o script antigo e o novo no mesmo
  momento e compare `pk_id`, `custo_medio` e `custo_pela_composicao`. As linhas
  que mudaram devem ser só as do ajuste.
- **No modelo:** a medida de conferência da seção 4.5 deve dar zero em
  qualquer recorte.
