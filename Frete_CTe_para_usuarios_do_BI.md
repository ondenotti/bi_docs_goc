# Frete do CT-e no custo, guia para quem usa o BI

Por que, em algumas compras, o custo do Solution não fecha com a composição do
BI, e como ler a coluna que explica isso.
Última revisão: 09/11/2026.

---

## 1. Em uma frase

**O Solution calcula o custo da peça no momento em que a compra é lançada, com o
frete do CT-e que está ligado à nota naquela hora. Se o CT-e muda depois, o
frete da nota é atualizado, mas o custo não.**

O custo que ficou errado é o que está no estoque. É ele que o sistema usa para
formar preço e para o CMV. Por isso o BI mostra os dois números e explica a
diferença, em vez de esconder um deles.

---

## 2. Dois números, os dois com razão

| | O que é | Para que serve |
|---|---|---|
| **Custo pela composição** | O custo montado pelo BI a partir da nota, com o frete do CT-e **como está hoje** | Mostra qual seria o custo correto |
| **Custo do Solution** (`custo_medio`) | O custo gravado no lançamento, que entrou no estoque | É o custo que o sistema realmente usa |

Na grande maioria das compras os dois são iguais. Quando o CT-e mudou depois do
lançamento, eles se separam, e a diferença é exatamente o frete que ficou a
mais ou a menos no custo do Solution.

---

## 3. O exemplo: nota 964726, filial Teresina

Um CT-e de **R$ 9.000,00** cobre duas notas de compra do mesmo fornecedor.

| Quando | O que aconteceu |
|---|---|
| 30/06/2026, 16:40 | A NF **964726** é lançada |
| 30/06/2026, 16:42 | O CT-e é lançado e o frete **inteiro**, de R$ 9.000,00, é rateado nesta nota. O custo das peças é calculado aqui |
| 08/07/2026, 17:00 | A NF **964725**, da mesma compra, é lançada e ligada ao **mesmo** CT-e |
| 08/07/2026 | O rateio é refeito: R$ 5.694,38 para a 964726 e R$ 3.305,62 para a 964725 |
| — | O frete gravado nas peças da 964726 é atualizado. **O custo delas, não** |

Resultado: as peças da 964726 ficaram no estoque com o frete dos R$ 9.000,00
inteiros, e a parte da 964725 entrou no custo duas vezes, uma em cada nota.

### Uma peça da nota, em números

Produto **0452C331065ED**, 8 unidades.

| | Composição do BI | Custo do Solution |
|---|---|---|
| Mercadoria, já sem os créditos de ICMS, PIS e COFINS | 10.386,41 | 10.386,41 |
| (+) Frete do CT-e, já sem os créditos do frete | 344,95 | 545,20 |
| **= Custo** | **10.731,36** | **10.931,61** |

- **A mercadoria é igual** nos dois. A diferença está toda no frete.
- **O BI usa o frete de hoje:** R$ 411,87 de frete, R$ 344,95 depois dos créditos.
- **O Solution usou o frete do CT-e inteiro:** 411,87 × 9.000,00 ÷ 5.694,38 =
  R$ 650,96 de frete, R$ 545,20 depois dos créditos.
- **A diferença é de R$ 200,25** nesta peça. Somando as 24 peças da nota, são
  R$ 2.768,17 a mais no estoque.

---

## 4. O outro caso: o CT-e chegou depois

O erro também acontece ao contrário. Se a compra é lançada **sem** CT-e e o
CT-e é ligado à nota dias depois, às vezes o custo não é recalculado e o frete
**nunca entra** no custo.

Isso aconteceu em **130 notas**, de 2019 e de 2022 a 2024, somando
**R$ 212.720,91** de frete que ficou fora do estoque. Não aparece desde 2025.

Um exemplo: a NF **13009** da filial Balsas, uma máquina lançada em 23/11/2022.
O CT-e dela, de R$ 16.800,00, só foi lançado em 08/12/2022. O custo da máquina
no Solution ficou só com a mercadoria, sem nenhum frete: R$ 15.246,00 a menos
(o frete já sem os créditos de PIS e COFINS).

---

## 4b. O terceiro caso: o CT-e foi tirado da nota depois

Também acontece de a compra ser custeada **com** o CT-e e o CT-e ser
desvinculado da nota depois. O frete continua no custo do Solution, mas a nota
fica sem frete nenhum, e o BI não tem como vê-lo pela nota.

Quem mostra o frete é a **contabilidade do lançamento**, que registrou o frete
e os créditos dele na hora do custeio.

Um exemplo: a nota interna **414546** da filial Balsas, de 25/05/2024, com 53
itens.
- **No lançamento:** um CT-e de R$ 3.816,83 foi lançado e o custo foi calculado já com ele, minutos depois dos itens.
- **Em 17/06/2024:** o CT-e foi alterado e hoje não está em nenhuma nota.
- **O resultado:** as peças ficaram no estoque com R$ 3.203,87 de frete (já sem os créditos), cerca de 3,5% a mais em cada item.

Isso aconteceu em **3 notas**, de abril e maio de 2024, somando **R$ 11.389,93**
de frete a mais no estoque.

---

## 5. Como ler no BI

A coluna **`Frete_ctc_divergencia_custeio`** mostra, linha a linha, quanto de
frete o custo do Solution tem a mais ou a menos que o correto.

O **custo pela composição já soma essa coluna**, para o custo calculado pelo BI
coincidir com o custo do Solution. A coluna continua separada para o erro do
sistema aparecer como uma parcela da composição:

```text
composição da nota (mercadoria + frete de hoje, sem os créditos)
  (+) Frete CTC Divergência - Erro Solution   (Frete_ctc_divergencia_custeio)
  = custo pela composição
  = custo do Solution
```

Se o custo calculado e o custo do Solution estiverem diferentes numa linha, é
porque apareceu uma situação que o BI ainda não conhece. Avise o time de BI.

| Valor da coluna | Significa |
|---|---|
| **Positivo** | O custo do Solution tem frete **a mais**. O CT-e foi dividido com outra nota, ou tirado da nota, depois do lançamento |
| **Negativo** | O custo do Solution tem frete **a menos**. O CT-e foi ligado depois do lançamento e o frete ficou fora |
| **Vazio** | A nota não tem esse problema |

A coluna `obs_composicao` diz o motivo em texto:

- *CT-e rateado com outra nota depois do custeio: frete a mais no custo*
- *CT-e vinculado depois do custeio: frete fora do custo*
- *CT-e desvinculado depois do custeio: frete a mais no custo*

> **Resumo:** quando a observação fala de CT-e, o custo do estoque está errado
> e a coluna `Frete_ctc_divergencia_custeio` mostra de quanto. O custo pela
> composição já inclui esse erro, para fechar com o Solution.

---

## 6. O que fazer quando aparecer

**Para análise de preço e margem**, lembre que o sistema usa o custo do
Solution. Se a coluna for grande para uma peça, a margem calculada pelo sistema
para ela está distorcida na mesma medida.

**Para corrigir o custo no Solution**, o caminho é o suporte do Solution ou um
ajuste de custo feito pela controladoria. O BI só aponta. Ele não corrige nada
no sistema.

**Pequenas sobras de centavos** entre o custo calculado e o do Solution vêm do
arredondamento que o Solution fez no lançamento. Elas não são outro erro.
Exemplos: no caso do CT-e rateado depois, como na nota 964726, no máximo
R$ 0,24 por linha; no CT-e desvinculado, R$ 0,55 numa linha da 414546.

---

## Dúvidas

A explicação de como a regra é calculada está na
[documentação técnica](Frete_CTe_documentacao_tecnica.md). Se algum caso não se
encaixar no que está aqui, fale com o time de BI.
