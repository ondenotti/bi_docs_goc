# PIS/COFINS alterado depois do custeio, documentação técnica

Para quem dá manutenção na fato de compras, no script SQL ou no modelo do Power BI.
Última revisão: 09/11/2026.

- **Fato:** `fcompra_scriptSQL.txt`, seções do cabeçalho:
  - "AJUSTES DE 09/11/2026: TROCA DE CST DE PIS/COFINS 73 -> 53 DEPOIS DO CUSTEIO"
  - "AJUSTES DE 09/11/2026: PIS/COFINS ALTERADO DEPOIS DO CUSTEIO"
- **Colunas:** `troca_de_cst_73_53` e `pis_cofins_alterado_apos_custeio`, mais três textos em `obs_composicao`
- **`custo_pela_composicao`** soma as duas colunas
- **Banco:** PostgreSQL, ERP Solution, acesso somente leitura

---

## 1. O problema

O Solution grava o custo da linha (`movext.e51vtotal`) e o lançamento contábil
quando a compra é fechada. Nesse momento, ele usa o PIS/COFINS da **pré-nota**
(`movped`):

- **CST 53** (com crédito): o PIS/COFINS sai do custo, e a contabilidade lança D 59 PIS a recuperar / D 55 COFINS a recuperar.
- **CST 73** (alíquota zero, sem crédito): o PIS/COFINS fica no custo, sem lançamento a recuperar.

Entre 02 e 22/01/2024, o PIS/COFINS de muitas linhas foi alterado depois do
fechamento, na `movext`:
- o CST foi trocado de 73 para 53, e a linha ganhou `e51vlrpis` e `e51vlrcofins`;
- o CST foi trocado de 53 para 73, e o PIS/COFINS foi zerado;
- o CST continuou 53, mas a base e o valor foram recalculados.

O custo e a contabilidade não foram refeitos. A alteração também não mudou
`e51alteracaodatahora`, então não passou pela tela normal da nota.

A composição do BI usa o PIS/COFINS de hoje. A diferença para o custo do
Solution é:

```text
diferença = (e51vlrpis + e51vlrcofins)  −  (e56vlrpis + e56vlrcofins)
             PIS/COFINS de hoje             PIS/COFINS da pré-nota

custo_composicao_sem_frete_div        (interno, composição da nota)
  + troca_de_cst_73_53                 (a diferença, quando é CST 73 → 53)
  + pis_cofins_alterado_apos_custeio   (a diferença, nos outros casos)
  + Frete_ctc_divergencia_custeio
  = custo_pela_composicao
  = custo_medio
```

---

## 2. A prova: a pré-nota e a contabilidade

A pré-nota (`movped`) é o registro que gera a entrada. Ela guarda o CST e o
valor de PIS/COFINS com que o item foi custeado. A ligação com a linha da nota é
`movped.e56lanctoid = movext.e51_e56sequencia` (`e56lanctoid` é a chave única da
`movped`).

**Três fatos sustentam a regra:**
- **O custo fecha.** Em todas as linhas marcadas, custo do Solution = composição + diferença, com resíduo zero.
- **A contabilidade bate com a pré-nota.** O PIS/COFINS lançado na nota (D 59 + D 55 em `crp_tab335`) é igual à soma da pré-nota em 189 das 190 notas de `pis_cofins_alterado_apos_custeio`.
- **A janela é estreita.** Fora de 02 a 22/01/2024 só aparecem 3 linhas, de mai/2024. Pré-nota diferente da linha é comum em qualquer época, mas nas outras épocas o custo segue a linha, não a pré-nota.

**A troca 73 → 53 por período**, nas compras "Valor Líquido" de dez/2023 a fev/2024:

| Pré-nota → linha | Entre 02 e 22/01/2024 | Fora dessa janela |
|---|---|---|
| 73 → 73 | 96 | 1.553 |
| **73 → 53** | **223** | **0** |
| 53 → 53 | 2.328 | 7.252 |

### Exemplo 1: 376704, troca 73 → 53

- **Compra:** filial 2 (Palmas/TO), NF 2737279 (notaint **376704**), operação 1, CFOP 2403, 16/01/2024.
- **Item:** `51667965` (NCM 8501.10.19), 2 unidades.

| Campo | Valor | Fonte |
|---|---|---|
| CST PIS/COFINS na pré-nota | 73 / 73, PIS/COFINS 0,00 | `movped` (`e56lanctoid` 1603500) |
| CST PIS/COFINS na linha hoje | 53 / 53 | `movext.e51cst_pis` |
| `e51vlrpis` / `e51vlrcofins` hoje | 70,68 / 325,58 (base 4.283,92) | `movext` |
| PIS/COFINS na contabilidade | nenhum lançamento | `crp_tab335` |
| **`e51vtotal`** | **5.527,03**, sem abater PIS/COFINS | `movext` |
| `custo_composicao_sem_frete_div` | 5.130,77 | composição |
| **`troca_de_cst_73_53`** | **396,26** | coluna |

O lançamento contábil da nota foi D 82 Peças para revenda 5.527,03 / C 350000
Compras a distribuir 4.283,92 / C 492 ICMS ST GR a recolher 1.243,11.

### Exemplo 2: 376419, CST 53 → 73 e valor recalculado

- **Compra:** filial 5 (Balsas/MA), NF 2736984 (notaint **376419**), 38 itens, custeada em 15/01/2024 às 10:09.
- **Na nota inteira:** custo do Solution 19.698,01 contra a composição de 19.696,59 sem as colunas, uma diferença de só 1,42. Item a item, as diferenças se compensam.

| | Pré-nota (`movped`) | Linha hoje (`movext`) |
|---|---|---|
| PIS + COFINS da nota | **1.449,16** | 1.450,58 |
| Contabilidade (`crp_tab335`) | D 59 258,49 + D 55 1.190,67 = **1.449,16** | |

**Item `47506948`** (injetor, NCM 8409.99.61):

| Campo | Pré-nota | Hoje |
|---|---|---|
| CST PIS/COFINS | 53 | 73 |
| Base PIS/COFINS | 2.495,22 | 0,00 |
| PIS + COFINS | 41,17 + 189,64 = **230,81** | 0,00 |

- **`e51vtotal`** = `e51totnota2` 2.964,90 − ICMS 114,86 − 230,81 = **2.619,23**.
- **`custo_composicao_sem_frete_div`** = 2.850,04, e a coluna `pis_cofins_alterado_apos_custeio` = **−230,81**.
- **O CST de hoje é o certo:** todas as compras do item de 2025 em diante entram com CST 73.

**Os outros 37 itens:** a pré-nota calculou o PIS/COFINS sobre uma base menor
(`e56vlrmer` × 0,869, sem IPI), e a linha de hoje usa `e51vlrmer` + IPI. Por
exemplo, no item `84130920` a pré-nota tinha 241,18 e hoje são 277,62. A coluna
soma +232,23 nesses itens. Na nota: −230,81 + 232,23 = +1,42.

### Exemplo 3: 376870 e 374665, linha sem pré-nota

Algumas compras são lançadas **sem pré-nota** (`e51_e56sequencia` sem par na
`movped`). Nelas a prova linha a linha não existe, mas a contabilidade da nota
mostra o total com que o custo foi feito. A partir de 2023 cada compra tem
lançamento contábil ligado pelo notaint (`crp_tab335.t335doctofiscalid`, origem
CTM). Antes disso, não.

- **376870**, filial 3 (Lagoa da Confusão/TO), 16/01/2024, 1 linha, item `90490990`:
  - **Contabilidade:** só o PIS/COFINS do CT-e, D 59 4,09 e D 55 18,86. Nada da mercadoria.
  - **Linha hoje:** CST 53, PIS 75,54 + COFINS 347,93 = 423,47.
  - **Custo:** `e51vtotal` 7.801,68 = composição 7.378,21 + 423,47.
- **374665**, filial 3, 06/01/2024, 1 linha, item `84219438`: nenhum PIS/COFINS na contabilidade. Linha hoje com 763,89, que estão dentro do `e51vtotal` 8.918,98.

Quando a contabilidade não tem PIS/COFINS da mercadoria, **toda** linha da nota
fechou sem crédito. Então a diferença por linha é o PIS + COFINS de hoje,
exato, mesmo numa nota de várias linhas.

Não é um comportamento geral do lançamento sem pré-nota. De 2023 em diante há
28.999 linhas sem pré-nota com PIS/COFINS, e 28.789 fecham normalmente. Só 3
ficaram com o PIS/COFINS no custo, e a terceira (373987, 11,42) não tem a prova
da contabilidade.

---

## 3. A regra no script

### CTE `prenota_piscofins`

```sql
SELECT e56lanctoid,
       btrim(e56cst_pis)                                  AS cst_pis_prenota,
       COALESCE(e56vlrpis, 0) + COALESCE(e56vlrcofins, 0) AS piscofins_prenota
  FROM movped
```

Ela lê a `movped` inteira (cerca de 870 mil linhas) pela chave primária. O tempo
do script não mudou: 37 s contra 43 s na versão anterior, no mesmo teste.

### Em `base`

```sql
LEFT JOIN prenota_piscofins ppc ON ppc.e56lanctoid = mv.e51_e56sequencia
...
ppc.cst_pis_prenota,     -- interno
ppc.piscofins_prenota,   -- interno, NULL quando a linha não tem pré-nota
```

O join é único por linha, porque `e56lanctoid` é a chave primária. O alias é
`ppc` porque `pn` já é usado pelo PIS/COFINS do XML.

### CTE `piscofins_custeio`, depois de `fcompra`

```sql
CASE WHEN f.piscofins_prenota IS NOT NULL
      AND f.eh_devolucao = 'Não'
      AND f.atualiza_estoque = 'Sim'
      AND abs(COALESCE(f.PIS, 0) + COALESCE(f.COFINS, 0) - f.piscofins_prenota) > 0.05
      AND abs(f.custo_medio - (f.custo_composicao_sem_frete_div
                               + COALESCE(f.PIS, 0) + COALESCE(f.COFINS, 0)
                               - f.piscofins_prenota)) <= 0.05
     THEN COALESCE(f.PIS, 0) + COALESCE(f.COFINS, 0) - f.piscofins_prenota
END AS dif_piscofins_custeio   -- interno
```

- **A diferença tem que passar de 5 centavos.** Abaixo disso é arredondamento entre a pré-nota e a linha, e marcar essas linhas só criaria ruído.
- **O custo tem que provar a causa:** o custo do Solution é a composição mais a diferença, com folga de 5 centavos. Uma linha alterada que **tenha** sido recalculada não recebe valor.

**Linha sem pré-nota** (desde 09/11/2026), um segundo ramo do mesmo `CASE`. A
contabilidade vem da CTE `contabil_nota` (a mesma do caso 3 do frete), ligada
em `base` como `piscofins_contabil`:

```sql
WHEN f.piscofins_prenota IS NULL
 AND f.piscofins_contabil IS NOT NULL                       -- nota com contabilidade (2023+)
 AND COALESCE(f.PIS, 0) + COALESCE(f.COFINS, 0) > 0.05
 AND abs(f.piscofins_contabil
         - SUM(Frete_pis + Frete_cofins da linha) OVER w) <= 0.05   -- D 59 + D 55 = só o do CT-e
 AND abs(f.custo_medio - (f.custo_composicao_sem_frete_div
                          + COALESCE(f.PIS, 0) + COALESCE(f.COFINS, 0))) <= 0.05
THEN COALESCE(f.PIS, 0) + COALESCE(f.COFINS, 0)
```

`w` é a nota (`PARTITION BY fk_cod_empresa_filial, cod_notaint`). O valor cai em
`pis_cofins_alterado_apos_custeio`. As condições da `troca_cst` usam
`COALESCE(cst_pis_prenota, '')`, para a linha sem pré-nota não virar nulo na
comparação.

### CTE `troca_cst`: uma coluna por linha

```sql
CASE WHEN p.cst_pis_prenota = '73' AND btrim(p.cst_pis_entrada) = '53'
     THEN p.dif_piscofins_custeio END       AS troca_de_cst_73_53,
CASE WHEN NOT (p.cst_pis_prenota = '73' AND btrim(p.cst_pis_entrada) = '53')
     THEN p.dif_piscofins_custeio END       AS pis_cofins_alterado_apos_custeio
```

As duas condições são excludentes, então uma linha nunca tem as duas colunas e
a soma não repete valor.

### No teste do frete

A diferença da nota (`nota_diferenca`) desconta `dif_piscofins_custeio`. Assim
uma nota com as duas causas não confunde o teste do frete. Na base de hoje,
nenhuma nota tem as duas.

### Em `fcompra_final`

- `custo_pela_composicao = custo_composicao_sem_frete_div + COALESCE(troca_de_cst_73_53, 0) + COALESCE(pis_cofins_alterado_apos_custeio, 0) + divergência do frete`.
- `obs_composicao` ganha estes textos, depois dos de nota complementar e antes dos do frete:
  - *CST de PIS/COFINS trocado de 73 para 53 depois do custeio: PIS/COFINS ficou no custo*
  - *CST de PIS/COFINS trocado de 53 para 73 depois do custeio: custo abateu crédito que não existe mais* (pré-nota 53, linha hoje 73)
  - *PIS/COFINS preenchido depois do custeio, sem pré-nota: PIS/COFINS ficou no custo* (linha sem pré-nota)
  - *PIS/COFINS alterado depois do custeio: custo ficou com o PIS/COFINS da pré-nota* (o resto)

---

## 4. Validação (09/11/2026, base inteira)

**Script novo × versão anterior**
- 367.631 linhas nas duas versões, com os mesmos `pk_id` e nenhuma duplicada.
- `custo_medio` e `Frete_ctc_divergencia_custeio` saíram idênticos.
- `custo_pela_composicao` mudou em 1.401 linhas: as 1.377 da coluna nova e as 24 que entraram em `troca_de_cst_73_53`. Em todas, o valor novo é o antigo mais as colunas.
- Nenhuma linha tem as duas colunas, e nenhuma nota tem PIS/COFINS alterado e frete ao mesmo tempo.
- Em jan/2024 eram cerca de 1.400 linhas com diferença. Sobram 5, de outras causas.
- Na base inteira, as linhas com `custo_diferenca_composicao` acima de 5 centavos caíram de 3.896 para 2.495.

**O que as colunas explicam**

| Coluna / caso | Linhas | Notas | Valor | `custo_diferenca_composicao` |
|---|---|---|---|---|
| `troca_de_cst_73_53` | 225 | 121 | +25.896,69 | 0,00 |
| `pis_cofins_alterado_apos_custeio`, CST 53 → 73 | 22 | 20 | −2.092,66 | 0,00 |
| `pis_cofins_alterado_apos_custeio`, valor recalculado | 1.355 | 188 | +7.198,40 | 0,00 |
| **`pis_cofins_alterado_apos_custeio`, total** | **1.377** | **190** | **+5.105,74** (absoluto 12.508,12) | **0,00** |

- **A troca 73 → 53 ganhou 24 linhas** (13 notas, 876,20). Na primeira versão, o valor era o PIS/COFINS de hoje, sem descontar a pré-nota. Essas 24 tinham valor na pré-nota e por isso não fechavam. Por mês: jan/2024 223 linhas, 25.406,41; mai/2024 2 linhas, 490,28.
- **`pis_cofins_alterado_apos_custeio` por mês:** jan/2024 1.376 linhas, 189 notas, +5.461,63; mai/2024 1 linha, −355,89.

**Linha sem pré-nota (validação de 09/11/2026, 367.649 linhas):** só as 2
linhas, 376870 e 374665, somando +1.187,36, as duas com diferença zero. Nenhuma
linha que já tinha valor nas colunas mudou.

**Por filial** (sem as 2 linhas sem pré-nota, ambas da filial 3)

| Filial | `troca_de_cst_73_53` linhas | Valor | `pis_cofins_alterado_apos_custeio` linhas | Líquido | Absoluto |
|---|---|---|---|---|---|
| 1 Gurupi | 25 | 1.588,79 | 184 | 244,46 | 775,42 |
| 2 Palmas | 27 | 3.646,83 | 210 | 661,43 | 1.082,43 |
| 3 Lagoa da Confusão | 19 | 1.276,71 | 200 | 1.090,61 | 3.000,01 |
| 5 Balsas | 121 | 16.622,09 | 484 | 1.441,50 | 5.625,30 |
| 6 Guaraí | 14 | 222,22 | 139 | 475,12 | 585,32 |
| 7 Araguaína | 14 | 1.416,27 | 38 | 679,73 | 710,31 |
| 8 Marabá | 4 | 1.112,21 | 98 | 332,16 | 541,12 |
| 9 Redenção | 1 | 11,57 | 24 | 180,73 | 188,21 |

---

## 5. O que fica em aberto

**Por que a pré-nota usou base menor.** Em notas como a 376419, a base de
PIS/COFINS da pré-nota é cerca de 87% do valor da mercadoria, sem IPI. A linha
de hoje usa a mercadoria mais o IPI. A regra não depende do motivo, mas o
fiscal precisa dizer qual base é a certa.

**Ponto fiscal.** As alterações parecem uma revisão fiscal aplicada para trás.
A contabilidade ficou com o PIS/COFINS da pré-nota. O fiscal precisa confirmar:
- qual crédito foi para a EFD-Contribuições, o da pré-nota ou o de hoje;
- se o crédito é devido nas linhas trocadas para 53 e indevido nas trocadas para 73.

Se a EFD usou o de hoje, o custo do estoque e a contabilidade dessas notas
ficaram diferentes da apuração.

**2016–2018.** Há linhas com o custo sem abater PIS/COFINS, sem pré-nota, e
nessa época a contabilidade não tem lançamento ligado à nota. Não dá para
provar a causa, e elas continuam em `custo_diferenca_composicao`.

**As linhas de jan/2024 que sobraram.** Das 5, duas (376870 e 374665) foram
explicadas pela regra sem pré-nota. Seguem abertas a 373987 (11,42, sem prova
da contabilidade) e a 379935 (−17.420,00, com frete na contabilidade, ainda não
investigada). A quinta é uma devolução com 0,51 de arredondamento.

---

## 6. Manutenção

**`fcompra_scriptSQL_PowerQuery.txt`.** Ele embute o mesmo SQL, então precisa
ser atualizado a partir do script.

**No Power BI.** As duas colunas são parcelas da composição. Não some nenhuma
delas de novo ao `custo_pela_composicao`, porque já estão dentro. Em
`pis_cofins_alterado_apos_custeio`, positivos e negativos se compensam no
total. Para medir o tamanho do erro, use o valor absoluto ou olhe item a item.

### Consulta para investigar uma nota

```sql
SELECT m.e62notaint, m.e18codpro,
       p.e56cst_pis AS cst_prenota, m.e51cst_pis AS cst_hoje,
       p.e56basepis AS base_prenota, m.e51basepis AS base_hoje,
       p.e56vlrpis + p.e56vlrcofins AS piscofins_prenota,
       m.e51vlrpis + m.e51vlrcofins AS piscofins_hoje,
       (m.e51vlrpis + m.e51vlrcofins) - (p.e56vlrpis + p.e56vlrcofins) AS diferenca,
       m.e51vtotal, m.e51cadastrodatahora, m.e51alteracaodatahora,
       (SELECT sum(t.t335valor) FROM crp_tab335 t
         WHERE t.t335doctofiscalid = m.e62notaint AND t.t335ctadeb IN (59, 55)) AS piscofins_contabil_nota
  FROM movext m
  LEFT JOIN movped p ON p.e56lanctoid = m.e51_e56sequencia
 WHERE m.e62notaint = 376419
 ORDER BY diferenca;
```
