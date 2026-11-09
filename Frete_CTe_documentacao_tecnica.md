# Frete do CT-e no custo, documentação técnica

Para quem dá manutenção na fato de compras, no script SQL ou no modelo do Power BI.
Última revisão: 06/11/2026.

- **Fato:** `fcompra_scriptSQL.txt`, seção "AJUSTES DE 06/11/2026" do cabeçalho
- **Coluna nova:** `Frete_ctc_divergencia_custeio`, mais dois textos novos em `obs_composicao`
- **Banco:** PostgreSQL, ERP Solution, acesso somente leitura

---

## 1. O problema

O Solution grava o custo da linha (`movext.e51vtotal`) quando a compra é lançada,
com o frete do CT-e vinculado **naquele momento**. Se o vínculo do CT-e muda
depois, os campos de frete do item são atualizados (`e51frete_ctc`,
`e51vlicms_frete`, `e51valor_pis_ctc`, `e51valor_cofins_ctc`), mas o
`e51vtotal` às vezes não é recalculado.

O `custo_pela_composicao` do BI usa os campos de frete de hoje, então ele fica
**certo**. O `custo_medio` (que é o `e51vtotal`) fica **errado**, mas é o que
está no estoque e o que o sistema usa para formar preço. A coluna nova mede a
distância entre os dois:

```
custo_pela_composicao + Frete_ctc_divergencia_custeio = custo_medio
```

---

## 2. Os dois casos

| Caso | O que aconteceu | Sinal da coluna | Notas na base |
|---|---|---|---|
| **1. Rateado depois** | A nota foi custeada com o frete **inteiro** do CT-e, e depois outra nota entrou no mesmo CT-e | **+** frete a mais | 1 |
| **2. Vinculado depois** | A nota foi custeada **sem** CT-e, e o CT-e foi ligado depois | **−** frete a menos | 130 |

Os dois valores são **líquidos dos créditos do frete** (ICMS, PIS e COFINS),
porque é assim que o frete entra no custo.

---

## 3. Exemplo do caso 1: notas 607985 e 610161

**Os dados**
- **Filial:** 11, Teresina/PI, operação 1 (Compra para revenda, custo "Valor Líquido").
- **CT-e:** nº 9864 (`e13ctcid` 45903), emitido em 09/06/2026, R$ 9.000,00.

| Evento | Data/hora | Fonte |
|---|---|---|
| Itens da NF 964726 (notaint **607985**) gravados | 30/06/2026 16:40:02 | `movext.e51cadastrodatahora` |
| CT-e 9864 gravado | 30/06/2026 16:42:20 | `estctc.e13triggerdatahorainsert` |
| Itens da NF 964725 (notaint **610161**) gravados | 08/07/2026 17:00:43 | `movext.e51cadastrodatahora` |

**O rateio de hoje** (`notassctc`):

| notaint | `e16vlrrat` | `e16totalprodutos` |
|---|---|---|
| 607985 | 5.694,38 | 159.568,17 |
| 610161 | 3.305,62 | 93.019,75 |
| **CT-e** | **9.000,00** | |

O custo da 607985 foi calculado às 16:42, com os 9.000,00 inteiros nela. Em
08/07 a 610161 entrou no CT-e, o rateio foi refeito e o `e51frete_ctc` das
linhas da 607985 caiu. O `e51vtotal` ficou como estava.

**A 610161 está certa**: quando ela foi lançada o CT-e já tinha as duas notas,
e a composição dela fecha no centavo. A parte dela, R$ 3.305,62, está no custo
duas vezes: inteira na 607985 e de novo na 610161.

### Linha do produto 0452C331065ED

| Campo | Valor |
|---|---|
| `e51totnota2` | 12.199,15 |
| `e51vlicms` | 813,28 |
| `e51vlrpis` / `e51vlrcofins` | 178,28 / 821,18 |
| `e51frete_ctc` | 411,87 |
| `e51vlicms_frete` / `e51valor_pis_ctc` / `e51valor_cofins_ctc` | 28,83 / 6,79 / 31,30 |
| **`custo_pela_composicao`** | **10.731,36** |
| **`e51vtotal`** (`custo_medio`) | **10.931,61** |

```
fator           = frete inteiro dos CT-es / rateio de hoje
                = 9.000,00 / 5.694,38                       = 1,58051
frete liquido   = 411,87 - 28,83 - 6,79 - 31,30             = 344,95
divergencia     = 344,95 x (1,58051 - 1)                    = 200,25
```

10.731,36 + 200,25 = 10.931,61, igual ao Solution.

**Na nota inteira:** são 24 linhas, com coluna de 2.768,63 contra diferença de
2.768,17. Sobram centavos por linha (no máximo 0,24, em F0CV0D003F5ED), do
arredondamento que o Solution fez no custeio original. Testamos quatro formas
de arredondar (fator sobre o líquido, sobre cada componente, alíquotas
nominais, rateio direto por valor) e nenhuma reproduz o centavo em todas as
linhas, então a regra ficou com a forma mais simples.

### Efeito contábil observado (filial 11)

- **Fretes a distribuir (350005):** a conta recebeu o CT-e uma vez (9.000,00) e foi baixada duas vezes (9.000,00 na 607985 e 3.305,75 na 610161).
- **Créditos do frete:** os créditos da parte duplicada também foram contados duas vezes na contabilidade: ICMS 231,54, PIS 54,68 e COFINS 251,36.

---

## 4. Exemplo do caso 2: nota 269525

**Os dados**
- **Filial:** 5, Balsas/MA, NF 13009, operação 1.
- **Item:** um item só, a máquina `10H650/803` (subgrupo 1503).
- **CT-e:** nº 2636 (`e13ctcid` 14456), só desta nota.

| Evento | Data/hora | Fonte |
|---|---|---|
| CT-e emitido pela transportadora | 18/11/2022 | `estctc.e13datae` |
| Item da NF 13009 gravado (e nunca mais alterado) | 23/11/2022 10:34:07 | `e51cadastrodatahora` = `e51alteracaodatahora` |
| CT-e gravado no Solution | 08/12/2022 16:58:37 | `estctc.e13triggerdatahorainsert` |

| Campo | Valor |
|---|---|
| `e51totnota2` | 237.690,00 |
| `e51vlicms` | 9.745,05 |
| `e51frete_ctc` | 16.800,00 |
| `e51vlicms_frete` / `e51valor_pis_ctc` / `e51valor_cofins_ctc` | 0,00 / 277,20 / 1.276,80 |
| **`custo_pela_composicao`** | **243.190,95** |
| **`e51vtotal`** (`custo_medio`) | **227.944,95** = 237.690,00 − 9.745,05, só a mercadoria |

```
divergencia = - (16.800,00 - 0,00 - 277,20 - 1.276,80) = -15.246,00
```

243.190,95 − 15.246,00 = 227.944,95, igual ao Solution, no centavo.

O CT-e foi gravado 15 dias depois do item, e o item nunca foi alterado depois
disso. O frete foi rateado na nota, mas não entrou no custo.

---

## 5. A regra no script

### Por que é um teste de hipótese, e não uma data

A `notassctc` **não guarda quando** cada nota foi vinculada ao CT-e. E a data
de gravação do CT-e sozinha não serve de critério. Nas notas com CT-e e custo
"Valor Líquido", **4.733** tiveram o CT-e gravado depois dos itens, e **4.518**
delas fecham. Ou seja, normalmente o Solution recalcula o custo; às vezes não.

Por isso a regra testa as duas hipóteses por **nota**, que é o grão do custeio,
e só preenche a coluna quando uma delas explica a diferença da nota inteira.

### Peças novas no `fcompra_scriptSQL.txt`

**CTE `frete_ctc_nota`:** o frete de hoje e o frete inteiro de cada nota. É
única por nota, então não multiplica linha.

```sql
SELECT n.e01codigo, n.e35filial, n.e62notaint,
       SUM(n.e16vlrrat)  AS rateio_atual,
       SUM(c.frete_ctc)  AS frete_inteiro
  FROM notassctc n
  JOIN (SELECT e01codigo, e35filial, e13ctcid, SUM(e16vlrrat) AS frete_ctc
          FROM notassctc GROUP BY 1, 2, 3) c USING (e01codigo, e35filial, e13ctcid)
 GROUP BY 1, 2, 3
```

Quando o CT-e é só desta nota, `frete_inteiro = rateio_atual` e o fator é 1.

**Em `base`** (uso interno, não sai no select final):

- `fator_frete_ctc_inteiro = frete_inteiro / rateio_atual`
- `tipo_valor_no_calc_custo = codope.e22custo`

**Em `fcompra`:**

- `testa_frete_custeio = 1` só para linha que **deveria fechar**: `e22custo = '3'` ("Valor Líquido"), não é devolução, `atualiza_estoque = 'Sim'`, sem nota complementar e com CT-e.
- `frete_ctc_liquido = frete − ICMS − PIS − COFINS do frete`.

**CTE `frete_custeio_linha`:** as duas hipóteses na linha e os totais da nota,
por janela (`PARTITION BY fk_cod_empresa_filial, cod_notaint`), somando só as
linhas testadas.

| Coluna | Conta |
|---|---|
| `hip_rateado_depois` | `round(frete_ctc_liquido x (fator - 1), 2)` |
| `hip_vinculado_depois` | `- frete_ctc_liquido` |
| `nota_diferenca` | soma de `custo_medio - custo_pela_composicao` |
| `nota_rateado_depois` | soma de `hip_rateado_depois` |
| `nota_frete_liquido` | soma de `frete_ctc_liquido` |
| `nota_folga` | `0,05 x linhas testadas + 0,05` |

**CTE `frete_custeio`:** decide o caso, nesta ordem:

```sql
CASE WHEN testa_frete_custeio = 0                                   THEN NULL
     WHEN abs(nota_diferenca) <= nota_folga                         THEN NULL  -- a nota fecha
     WHEN nota_frete_liquido > nota_folga
      AND abs(nota_diferenca - nota_rateado_depois) <= nota_folga   THEN 1
     WHEN nota_frete_liquido > nota_folga
      AND abs(nota_diferenca + nota_frete_liquido) <= nota_folga    THEN 2
END AS caso_frete_custeio
```

**CTE `fcompra_final`:**

- `Frete_ctc_divergencia_custeio` recebe a hipótese do caso.
- `obs_composicao` ganha dois textos, depois dos motivos que já existiam:
  - *CT-e rateado com outra nota depois do custeio: frete a mais no custo*
  - *CT-e vinculado depois do custeio: frete fora do custo*

Toda linha de uma nota que caiu num caso recebe valor, inclusive zero, quando
a linha não tem frete.

---

## 6. Validação (06/11/2026, base inteira)

**Script novo × versão anterior**
- 367.550 linhas nas duas versões, com os mesmos `pk_id` e nenhuma duplicada.
- `custo_medio`, `custo_pela_composicao` e `custo_diferenca_composicao` saíram idênticos em todas as linhas.
- `obs_composicao` mudou em 355 linhas, todas que antes estavam vazias.

**O que a coluna explica**

| Caso | Ano | Notas | Linhas | Coluna | Sobra |
|---|---|---|---|---|---|
| 1. Rateado depois | 2026 | 1 | 24 | 2.768,63 | −0,46 |
| 2. Vinculado depois | 2019 | 1 | 6 | −340,36 | 0,00 |
| | 2022 | 39 | 162 | −35.385,25 | 0,00 |
| | 2023 | 64 | 114 | −157.080,80 | 0,00 |
| | 2024 | 26 | 49 | −19.914,50 | 0,00 |
| **Total caso 2** | | **130** | **331** | **−212.720,91** | **0,00** |

**Prova das datas:** nas 130 notas do caso 2, o CT-e foi gravado
(`e13triggerdatahorainsert`) **depois** dos itens (`e51cadastrodatahora`), em
todas.

**Por filial:**

| Filial | Notas | Coluna |
|---|---|---|
| 1 Gurupi | 14 | −28.444,95 |
| 2 Palmas | 19 | −66.945,34 |
| 3 Lagoa da Confusão | 44 | −5.186,01 |
| 4 Alvorada | 1 | −119,68 |
| 5 Balsas | 24 | −52.190,17 |
| 6 Guaraí | 1 | −1.482,16 |
| 7 Araguaína | 7 | −20.071,17 |
| 8 Marabá | 9 | −30.503,10 |
| 9 Redenção | 11 | −7.778,33 |
| 11 Teresina | 1 | +2.768,63 |

Desde 2025 não há nenhuma nota no caso 2. Das notas com CT-e gravado depois dos
itens, a única de 2025 em diante que não fecha é a própria 607985 (caso 1).

---

## 7. No Power BI

```DAX
Custo composição ajustado =
    SUM ( fcompra[custo_pela_composicao] )
        + SUM ( fcompra[Frete_ctc_divergencia_custeio] )
```

Essa medida deve bater com `SUM ( fcompra[custo_medio] )`. A sobra de centavos
do caso 1 continua em `custo_diferenca_composicao`.

---

## 8. Limites e manutenção

**Só dois padrões.** A regra reconhece "custeado com o CT-e inteiro" e
"custeado sem CT-e". Um meio-termo também ficaria de fora, por exemplo um CT-e
com três notas em que a nota foi custeada quando só havia duas. Nesse caso a
linha segue sem explicação em `custo_diferenca_composicao`. Na base de hoje há
8 notas com CT-e dividido que não fecham e não se encaixam em nenhum dos dois
casos: 7 da filial 1 de 2017-2018, de 2,91 a 250,86, e a 199470 da filial 9,
de 02/2022, com −2.060,25. Ficaram sem análise.

**Folga de 5 centavos por linha.** Ela existe para aceitar o arredondamento do
Solution. Abrir mais a folga pode atribuir ao frete diferenças de outra origem.

**Devolução fica de fora.** A composição dela vem da compra de origem (ajuste
de 03/11/2026), então ela não é testada aqui.

**Ordem do `obs_composicao`.** Os motivos antigos vêm primeiro. Uma linha com
nota complementar nunca é testada para frete.

**`fcompra_scriptSQL_PowerQuery.txt`.** Ele embute o mesmo SQL. Precisa ser
atualizado a partir do script, ou copiado do BI depois da atualização.

### Consulta para investigar uma nota

```sql
SELECT n.e62notaint, n.e16vlrrat, n.e16totalprodutos,
       e.e13numero, e.e13vlrfrete, e.e13datae,
       e.e13triggerdatahorainsert AS ctc_gravado,
       (SELECT min(m.e51cadastrodatahora) FROM movext m
         WHERE m.e62notaint = n.e62notaint)  AS itens_gravados
  FROM notassctc n
  JOIN estctc e USING (e01codigo, e35filial, e13ctcid)
 WHERE n.e13ctcid IN (SELECT e13ctcid FROM notassctc WHERE e62notaint = 607985)
 ORDER BY ctc_gravado, itens_gravados;
```
