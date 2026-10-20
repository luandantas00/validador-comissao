# Prompt mestre: Validador de Comissão

Use este prompt sempre que for pedir a uma IA (Claude, por exemplo) para manter, corrigir ou ampliar o programa. Cole o texto abaixo e, em seguida, anexe o `index.html` atual do repositório e os arquivos novos (manual do canal, extratos de exemplo).

---

Você vai manter o **Validador de Comissão**, um programa web de arquivo único (`index.html`) usado por colaboradores da Claro dos canais indiretos para validar a remuneração variável (RV). O programa roda 100% no navegador: lê os arquivos com SheetJS (`xlsx` 0.18.5 via cdnjs), não envia dados a servidor nenhum e é publicado no GitHub Pages.

## Objetivo

Encontrar as vendas que **foram pagas ao parceiro e não foram pagas ao colaborador** e mostrar o impacto disso na meta e na RV.

## Arquivos de entrada (detectados pelos cabeçalhos)

1. **Extrato do parceiro** (`VALIDO_COMISSAO`, `TIPO_OPERACAO`, `DATA_ATIVACAO`, `NUN_NTC`, `CONTRATO_EBT_NET`, `PLANO`, `ARQUIVO`, `PENDENCIA`, `COD_MOBILE`, `MES_REFERENCIA`, `VALOR_COMISSAO`, `DSC_NOME_PLANO`). Vale só o que está com **VALIDO_COMISSAO = T**.
2. **Analítico do colaborador** (`NR_MES_REFERENCIA`, `DT_EVENTO`, `NM_TIPO_VENDA`, `NR_NTC`, `NR_CONTRATO`, `FG_CONSIDERA_PG`, `DS_MOTIVO_NAO_PG`, `VL_PESO`, `NM_COLABORADOR`, `CD_REVENDA`). Pode vir de vários meses de referência.
3. **Planilha de meta** (`INDICADOR`, `META`, `PESO`, `REALIZADO`).

## Regras de negócio que não podem mudar sem pedido explícito

- **A data da venda manda.** O mês pesquisado é o da `DATA_ATIVACAO` (parceiro) e da `DT_EVENTO` (analítico). O `MES_REFERENCIA` só diz quando foi pago. Uma venda de maio paga em junho conta para a meta de maio.
- **Venda regularizada muda de data no extrato.** Quando a pendência é resolvida, o parceiro recebe numa referência posterior e a linha vem com data nova. Por isso as linhas da mesma venda (mesmo NTC ou contrato + mesmo indicador) são agrupadas quando estão até 45 dias da primeira, e vale a data mais antiga.
- **Chave de cruzamento:** linha móvel pelo NTC; TV e Banda Larga Fixa pelo contrato, usando os **9 últimos dígitos** de `CONTRATO_EBT_NET` contra `NR_CONTRATO`.
- **Classificação do extrato do parceiro (canal AA):**
  - Eventos de venda: `ATIVACAO POS-PAGO`, `MIGRACAO PRE-POS`, `CONTROLE FACIL`, `CONTROLE FACIL MIG`, `TV`, `BANDA LARGA FIXA`, `CONTESTACAO ATIVACOES`, `CONTESTACAO TV`.
  - Controle: tipo com "CONTROLE", plano com "CONTROLE" ou "FLEX", ou arquivo "CLARO FLEX". Banda Larga Móvel: plano com "INTERNET". O restante móvel é Conta.
  - Fora da conta: serviços, bônus, comissões adicionais, estornos (não reduzem a RV do colaborador), upgrade/downgrade, migração Controle→Conta, transferência de titularidade, `PAGAR_2`/`PAGAR_3` (2ª recarga em diante de TV pré).
  - **Gratuidade/degustação (regra 2.1.3 xi):** se qualquer linha da venda tiver "GRATUITO", "GRATIS", "DEGUSTACAO" ou "CORTESIA" no arquivo, na pendência ou no plano, a venda inteira sai, mesmo que depois seja paga ao parceiro como dependente comum. Ela aparece na lista de excluídos.
  - **Claro TV pré** (arquivo `PAGAR_*` ou contrato de 11 dígitos começando com 21): na meta conta 0,5 (1 gross a cada 2 vendas).
  - **Claro Multi:** o extrato do parceiro não marca Combo como o analítico. Mostrar só como indicativo (contratos com linha "- CM" paga ao parceiro sem Combo pago no analítico).
- **Ação de cada divergência:**
  - **Contestar:** o analítico da referência em que o parceiro recebeu foi enviado e a venda não está paga ao colaborador. Prazo de 15 dias corridos.
  - **Conferir:** o analítico dessa referência não foi enviado. Pedir o arquivo.
- **Cálculo da RV (Etapa I):** realizado ÷ meta × peso, com cada indicador limitado a 150%. Abaixo de 80% no ponderado não há RV; o teto da Etapa I é 150%. A Meta Estrela (faixas 80/100/120%) e o E-learning (+10%) só entram se houver elegibilidade. Teto total de 220%.

## Canais e perfis cadastrados

- **AA (ativo)**, norma PL-DRH-042, Anexo I-F rev. 09:
  - Grupo II Perfil 1 (área cabeada): Conta 30, Controle 15, TV 15, BLF 25, Claro Multi 15, BLM 0. Estrela: TV 3,5/7,5/15; BLF PF 5/10/20; Multi 5/10/20; BLF PME 1,5/3/5.
  - Grupo II Perfil 2 (área não cabeada): BLM 5, Conta 30, Controle 25, TV 25, Claro Multi 15. Estrela: TV 3,5/7,5/15; BLM 5,5/11/22,5; Multi 5,5/11/22,5.
  - Grupo II Perfil 3 (Brand Shop): Conta 100%, sem Estrela e sem E-learning.
  - Grupo I (Gerente Vendas AA): mesmos pesos e Estrela do Perfil 1.
  - Canal híbrido AA + Varejo: 60% AA e 40% Varejo na elegibilidade (ainda não implementado).
- **LP, Varejo, Recarga e Claro Empresas:** pré-cadastrados como "em breve", aguardando os manuais.

## Onde mexer no código

- O objeto `CHANNELS` no motor (`VC`) guarda canais, perfis, pesos e faixas da Estrela. Para ativar um canal novo, preencha os perfis e troque `ativo: false` por `true`.
- `classify()` define o indicador de cada linha do extrato. `GROSS_OPS`, `EXCLUDE_ARQ` e `GRATIS` controlam o que entra e o que sai.
- `validate()` faz o cruzamento e `simulate()` calcula a RV.

## Como responder

- Antes de mudar uma regra, mostre qual trecho do manual sustenta a mudança.
- Teste com os arquivos anexados e informe os números antes e depois (vendas pagas ao parceiro, pagas ao colaborador, divergências por indicador e ponderado).
- Devolva o `index.html` completo, pronto para substituir o do repositório.
- Mantenha tudo em um arquivo, sem servidor e sem enviar dados para fora do navegador.

## Pedido de hoje

[descreva aqui o que você quer: ativar o canal Varejo, corrigir uma regra, incluir um campo etc.]
