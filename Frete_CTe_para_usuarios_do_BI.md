# Frete do CT-e no custo, guia para quem usa o BI

Por que, em algumas compras, o custo do Solution não fecha com a composição do
BI, e como ler a coluna que explica isso.
Última revisão: 06/11/2026.

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

## 5. Como ler no BI

A coluna **`Frete_ctc_divergencia_custeio`** mostra, linha a linha, quanto de
frete o custo do Solution tem a mais ou a menos que o correto.

```
custo pela composição
  (+) Frete_ctc_divergencia_custeio
  = custo do Solution
```

| Valor da coluna | Significa |
|---|---|
| **Positivo** | O custo do Solution tem frete **a mais**. O CT-e foi dividido com outra nota depois do lançamento |
| **Negativo** | O custo do Solution tem frete **a menos**. O CT-e foi ligado depois do lançamento e o frete ficou fora |
| **Vazio** | A nota não tem esse problema |

A coluna `obs_composicao` diz o motivo em texto:

- *CT-e rateado com outra nota depois do custeio: frete a mais no custo*
- *CT-e vinculado depois do custeio: frete fora do custo*

> **Resumo:** quando as duas colunas não batem e a observação fala de CT-e, a
> composição do BI está certa e o custo do estoque está errado. A coluna nova
> mostra de quanto.

---

## 6. O que fazer quando aparecer

**Para análise de preço e margem**, lembre que o sistema usa o custo do
Solution. Se a coluna for grande para uma peça, a margem calculada pelo sistema
para ela está distorcida na mesma medida.

**Para corrigir o custo no Solution**, o caminho é o suporte do Solution ou um
ajuste de custo feito pela controladoria. O BI só aponta. Ele não corrige nada
no sistema.

**Pequenas sobras de centavos** no caso do CT-e rateado depois, como na nota
964726, vêm do arredondamento que o Solution fez no lançamento. Elas não são
outro erro.

---

## Dúvidas

A explicação de como a regra é calculada está na
[documentação técnica](Frete_CTe_documentacao_tecnica.md). Se algum caso não se
encaixar no que está aqui, fale com o time de BI.
