# Documentação dos painéis de BI

Material de apoio para quem usa os painéis. Cada página explica o que um número
significa, de onde ele vem e como lê-lo sem tropeçar.

> Use o menu à esquerda para navegar, ou a busca no topo da barra lateral.

---

## Documentos disponíveis

| Documento | Sobre o que é | Última revisão |
|---|---|---|
| [Formação do custo das compras](Formacao_custo_para_usuarios_do_BI.md) | Como o BI monta o Custo Médio Calculado a partir das notas, o que cada linha da matriz significa e por que ele é comparado com o custo do Solution | 10/11/2026 |
| [Formação do custo das compras (Técnico)](Formacao_custo_documentacao_tecnica.md) | A fórmula do `custo_pela_composicao`, a coluna e o campo do Solution de cada parcela, as medidas DAX, por que devolução, estorno e outras entradas entram pelo `custo_medio`, e a validação ano a ano | 10/11/2026 |
| [Compra NET](NET_para_usuarios_do_BI.md) | O que o valor NET é, o que não é, e como o número é montado | 16/10/2026 |
| [Compra NET (Técnico)](NET_documentacao_tecnica.md) | Para quem dá manutenção na fato, no script SQL ou no modelo do Power BI | 16/10/2026 |
| [Frete do CT-e no custo](Frete_CTe_para_usuarios_do_BI.md) | Por que o custo do Solution às vezes não fecha com a composição quando o CT-e muda depois do lançamento | 09/11/2026 |
| [Frete do CT-e no custo (Técnico)](Frete_CTe_documentacao_tecnica.md) | A regra da coluna `Frete_ctc_divergencia_custeio`, os exemplos e a validação | 09/11/2026 |
| [ICMS complementar no custo](ICMS_complementar_para_usuarios_do_BI.md) | Por que o ICMS da nota complementar às vezes baixa o custo da peça e às vezes não | 06/11/2026 |
| [ICMS complementar no custo (Técnico)](ICMS_complementar_documentacao_tecnica.md) | A regra do reprocessamento do custo, os quatro casos do `obs_composicao`, exemplos e validação | 06/11/2026 |
| [PIS/COFINS alterado depois do custeio](Troca_CST_PIS_COFINS_para_usuarios_do_BI.md) | Por que compras de jan/2024 ficaram com o custo diferente depois que o PIS/COFINS das linhas foi alterado (troca de CST 73 → 53, 53 → 73 e valor recalculado) | 09/11/2026 |
| [PIS/COFINS alterado depois do custeio (Técnico)](Troca_CST_PIS_COFINS_documentacao_tecnica.md) | A regra das colunas `troca_de_cst_73_53` e `pis_cofins_alterado_apos_custeio`, a prova pela pré-nota e pela contabilidade, exemplos e validação | 09/11/2026 |

---

## Dúvidas

Se algum número não bater com o que você esperava, procure primeiro a página do
indicador — a resposta costuma estar na seção "o que ele não é". Se ainda assim
ficar dúvida, fale com o time de BI.
