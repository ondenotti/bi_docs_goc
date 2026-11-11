# PIS/COFINS alterado depois do custeio, guia para quem usa o BI

Por que compras de janeiro de 2024 ficaram com o custo diferente da composição
do BI depois que o PIS/COFINS das linhas foi alterado, e como o BI mostra isso.
Última revisão: 09/11/2026.

---

## 1. Em uma frase

**O Solution calculou o custo da peça com o PIS/COFINS que estava na compra no
momento do lançamento. Em janeiro de 2024, o PIS/COFINS de muitas linhas foi
alterado depois, mas o custo da peça não foi recalculado.**

O custo que ficou no estoque é o do lançamento. É ele que o sistema usa para
formar preço e para o CMV. O BI monta a composição com o PIS/COFINS **de hoje**,
por isso os dois se separam, e o BI mostra a diferença em duas colunas.

---

## 2. Como o PIS/COFINS entra no custo

Toda linha de compra tem um código, o **CST**, que diz se a empresa toma ou não
crédito de PIS/COFINS:

| CST | Significa | O que acontece com o custo |
|---|---|---|
| **53** | Com crédito | O PIS/COFINS **sai** do custo, porque a empresa recupera esse imposto |
| **73** | Sem crédito (alíquota zero) | O PIS/COFINS fica **dentro** do custo da peça |

Quanto mais crédito, mais barato fica o custo. Se o crédito muda depois do
lançamento e o custo não é refeito, o custo fica errado na mesma medida.

Entre **02 e 22 de janeiro de 2024**, três tipos de alteração foram feitos
depois do lançamento:

| O que mudou | Efeito no custo do Solution | Coluna no BI |
|---|---|---|
| CST trocado de **73 para 53**: a linha passou a ter crédito | Ficou **mais caro**: o PIS/COFINS ficou dentro | `troca_de_cst_73_53` |
| CST trocado de **53 para 73**: a linha perdeu o crédito | Ficou **mais barato**: abateu um crédito que não existe mais | `pis_cofins_alterado_apos_custeio` |
| CST continuou 53, mas o **valor** do PIS/COFINS foi recalculado | Mais caro ou mais barato, conforme o valor subiu ou desceu | `pis_cofins_alterado_apos_custeio` |
| Compra lançada **sem pré-nota** e sem PIS/COFINS, que foi preenchido depois | Ficou **mais caro**: o PIS/COFINS ficou dentro | `pis_cofins_alterado_apos_custeio` |

No caso sem pré-nota, quem prova é a **contabilidade do lançamento**: ela não
tem PIS/COFINS da mercadoria. Exemplo: a nota interna **376870** da filial
Lagoa da Confusão, de 16/01/2024. O custo do Solution, 7.801,68, tem dentro o
PIS 75,54 e a COFINS 347,93 que a linha mostra hoje.

---

## 3. Exemplo 1: troca de 73 para 53, NF 2737279, Palmas

Compra lançada em 16/01/2024. Item **51667965**, 2 unidades.

| Momento | CST | PIS + COFINS |
|---|---|---|
| Lançamento da compra | 73, sem crédito | 0,00 |
| Hoje, na linha da nota | 53, com crédito | 396,26 (PIS 70,68 + COFINS 325,58) |

| | Valor |
|---|---|
| Custo pela composição, com o crédito de hoje | 5.130,77 |
| (+) `troca_de_cst_73_53` | 396,26 |
| **= Custo do Solution** | **5.527,03** |

A peça ficou no estoque 7,7% mais cara do que o custo líquido do crédito.

---

## 4. Exemplo 2: a nota que quase fecha, nota interna 376419, Balsas

Compra lançada em 15/01/2024, com 38 itens. Na nota inteira, a diferença é só
de **R$ 1,42**: custo calculado 19.696,59 contra 19.698,01 no Solution. Mas
item a item, as diferenças são grandes e se compensam.

**O injetor 47506948:**

| Momento | CST | PIS + COFINS |
|---|---|---|
| Lançamento da compra | 53, com crédito | 230,81 |
| Hoje, na linha da nota | 73, sem crédito | 0,00 |

| | Valor |
|---|---|
| Custo pela composição, sem crédito, como hoje | 2.850,04 |
| (+) `pis_cofins_alterado_apos_custeio` | −230,81 |
| **= Custo do Solution** | **2.619,23** |

O Solution abateu um crédito de 230,81 que hoje não existe. Esse item entra
sempre sem crédito nas compras seguintes, então o certo é o de hoje: **a peça
ficou no estoque R$ 230,81 mais barata do que deveria.**

**Os outros 37 itens:** no lançamento, o PIS/COFINS foi calculado sobre uma base
menor. Hoje a base é maior, então o crédito de hoje é maior do que o abatido. O
custo do Solution ficou **R$ 232,23 mais caro** no total. Exemplo: no item
84130920 o crédito abatido foi 241,18, e hoje é 277,62. Diferença: +36,44.

Somando: −230,81 + 232,23 = **+1,42**. Por isso a nota parecia quase certa.

---

## 5. Como ler no BI

As duas colunas mostram, linha a linha, quanto o custo do Solution difere por
causa do PIS/COFINS alterado depois do lançamento. Uma linha nunca tem as duas.

| Coluna | Valor | Significa |
|---|---|---|
| `troca_de_cst_73_53` | Sempre positivo | O PIS/COFINS ficou dentro do custo |
| `pis_cofins_alterado_apos_custeio` | Positivo | O custo do Solution ficou mais caro: abateu menos crédito que o de hoje |
| | Negativo | O custo do Solution ficou mais barato: abateu mais crédito que o de hoje |
| As duas | Vazio | A linha não tem esse problema |

O **custo pela composição já soma as duas colunas**, para o custo calculado pelo
BI coincidir com o custo do Solution. As colunas continuam separadas para o erro
aparecer como uma parcela da composição:

```text
composição da nota (com o PIS/COFINS de hoje)
  (+) troca_de_cst_73_53
  (+) pis_cofins_alterado_apos_custeio
  = custo pela composição
  = custo do Solution
```

A coluna `obs_composicao` diz o motivo em texto:

- *CST de PIS/COFINS trocado de 73 para 53 depois do custeio: PIS/COFINS ficou no custo*
- *CST de PIS/COFINS trocado de 53 para 73 depois do custeio: custo abateu crédito que não existe mais*
- *PIS/COFINS alterado depois do custeio: custo ficou com o PIS/COFINS da pré-nota*
- *PIS/COFINS preenchido depois do custeio, sem pré-nota: PIS/COFINS ficou no custo*

> **Atenção à soma:** em `pis_cofins_alterado_apos_custeio` os valores
> positivos e negativos se compensam. O total da coluna é pequeno, mas cada
> peça pode estar bem errada. Para medir o tamanho do problema, olhe item a
> item.

---

## 6. O tamanho

| Coluna | Linhas | Notas | Valor |
|---|---|---|---|
| `troca_de_cst_73_53` | 225 | 121 | +25.896,69 |
| `pis_cofins_alterado_apos_custeio`, CST 53 → 73 | 22 | 20 | −2.092,66 |
| `pis_cofins_alterado_apos_custeio`, valor recalculado | 1.355 | 188 | +7.198,40 líquido (10.415,46 somando sem sinal) |
| `pis_cofins_alterado_apos_custeio`, sem pré-nota | 2 | 2 | +1.187,36 |

Quase tudo é de **janeiro de 2024**. A exceção são 3 linhas de maio de 2024.
**Balsas** é a filial com mais casos. Também aparecem Gurupi, Palmas, Lagoa da
Confusão, Guaraí, Araguaína, Marabá e Redenção.

---

## 7. O que fazer quando aparecer

**Para análise de preço e margem:** a peça está no estoque com o custo do
Solution.
- **Valor positivo:** o custo do Solution está mais alto que o correto. Na
  venda, o PIS/COFINS é cobrado de novo sobre a receita, então o imposto pesou
  duas vezes no preço dessas unidades.
- **Valor negativo:** o custo do Solution está mais baixo que o correto, e a
  margem calculada pelo sistema para essas unidades parece maior do que é.
- **O efeito é limitado** às unidades que entraram na janela. As compras
  seguintes entraram com o PIS/COFINS certo e diluíram o custo médio.

**Para entender por que alteraram o PIS/COFINS:** as alterações parecem uma
revisão fiscal aplicada para trás, porque as mesmas peças entram depois já com
o CST e o valor de hoje. Quem confirma o motivo, e qual crédito foi para a
apuração, é o fiscal. A contabilidade das notas ficou com o PIS/COFINS do
lançamento.

**Para corrigir o custo no Solution:** o caminho é o suporte do Solution ou um
ajuste de custo pela controladoria. O BI só aponta.

---

## Dúvidas

A explicação de como a regra é calculada está na
[documentação técnica](Troca_CST_PIS_COFINS_documentacao_tecnica.md). Se algum
caso não se encaixar no que está aqui, fale com o time de BI.
