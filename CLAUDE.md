# CLAUDE.md — Contexto do Projeto

## Visão Geral
Sistema de 3 automações Python+Playwright para processos de primeiro emplacamento de veículos 0km, integrando AutoDespa ao KaizenCRM.

## Automações

### 1. Lançamento Débitos (`lancamento_debitos/`)
- Acessa AutoDespa → filtra "Entregue" → extrai placa, CPF, valores do REC.pdf
- Busca cliente no KaizenCRM → lança honorários (R$70, service_id=63), taxa (R$470, service_id=63), IPVA (service_id=67 parcial / 68 integral)
- **Checkpoint por débito** em `checkpoint_debitos.json` — cada débito salvo individualmente

### 2. Anexa Documentos (`anexa_documentos/`)
- Baixa CRLV, REC, comprovante IPVA do AutoDespa
- Anexa no KaizenCRM com classificação automática
- Detecta CRLV por conteúdo (keywords + extração de texto)

### 3. Atualiza Tarefas (`atualiza_tarefas/`)
- Extrai datas do AutoDespa (inclusão, entrega, NFe, CRLV)
- "Primeiro Emplacamento" → sempre vai para **Entrada (3)** (não Concluído)
- "Isenção IPVA" / "Liberação Rodízio" → Entrada se Em aberto(1) ou Parado(2)
- Exporta Excel com processos de primeiro emplacamento

## Orquestrador (`agendador/rodar_tudo.ps1`)
- Mata apenas processos Playwright (zombies), preserva Chrome pessoal
- Executa as 3 automações em sequência
- Task Scheduler: seg-sex 08:00, repete 1h por 8h

## Decisões Importantes

### Checkpoint por Débito (não por processo)
Cada débito (honorários, taxa, IPVA) é registrado separadamente no checkpoint. Isso previne duplicatas mesmo se a automação falhar no meio.

### Busca KaizenCRM sempre navega para home
`buscar_cliente_por_cpf()` faz `page.goto(KAIZEN_URL)` antes de pesquisar — evita falhas em cascata após timeouts anteriores.

### Primeiro Emplacamento → Entrada (3)
Usuário pediu para sempre voltar status para "Entrada" ao atualizar, não "Concluído".

### Modal reabre antes de cada débito
`_abrir_modal_debito()` fecha modal se já aberto, espera 1s, depois clica em "Registrar débito" com scroll + fallback de seletores.

### Toast detectado via polling
`_salvar_debito()` faz polling por 10s (1s por iteração) → toast encontrado = True; modal fechou (fallback) = True com warning; timeout = False.

### Filtro de processos já concluídos no checkpoint
`main.py` filtra processos que já têm algum débito no checkpoint antes de processar. Isso evita reprocessar sempre os mesmos N processos e avança para os mais antigos não processados.

### Verificação de cliente antes de lançar débitos
`verificar_cliente_na_pagina()` extrai `document.body.innerText` e busca o nome do cliente esperado. Se não encontrar, aborta os lançamentos — evitando débito na conta errada (bug onde Taxa de URX2I82 foi parar em Valdir Ramos Pires).

## Problemas Resolvidos

| Problema | Solução |
|---|---|
| Débitos duplicados | Checkpoint por débito individual |
| Toast auto-dismiss (~3s) causava falso negativo | Polling 10s em vez de wait fixo |
| Modal aberto com dados stale | Fecha e reabre antes de cada débito |
| "campo de busca não encontrado" | Sempre navega para home antes de buscar |
| IPVA regex não capturava "PARC" / "À VISTA" | Regex `_PAT_IPVA` atualizado |
| Combobox "Posto Fiscal" conflitava | Usa `page.locator(".modal.show select:visible").first` |
| REC "TAXAS, HONORARIO" não reconhecido | Regex `_LINHA_ITEM` adicionou vírgula + novo padrão `_PAT_TAXAS_HON` |
| Upload IPVA não aparecia na lista | Wait 3s + fallback por nome parcial |
| Botão "Registrar débito" não encontrado | Scroll + wait + fallback seletores |
| Sempre processava os mesmos 10 processos, nunca avançava | Filtro no checkpoint antes de processar (pula já concluídos) |
| Débito caía no cliente errado | `verificar_cliente_na_pagina()` antes de cada lançamento |

## Estado Atual (10/07/2026)

### Últimas execuções (10/07/2026)
- **Lançamento Débitos**: 3 execuções, total de 17 processos novos lançados, 0 erros
  - Inclui URX2I82 (Eliana), UOP0E16 (Lucineide), UPM7A18, UOR4I75, UPU0D56, URA4E23, UQO5I90, UQF6J38 + IPVA, entre outros
- **Anexa Documentos**: 50/50 no checkpoint, sem novos
- **Atualiza Tarefas**: 50/50 no checkpoint, sem novos

### Checkpoints
- **Débitos (lancamento_debitos)**: 50+ processos, todos os 50 "Entregue" com débitos registrados
- **Anexos**: 60+ processos
- **Tarefas**: 50+ processos

### Pendências
- Nenhuma pendência no momento — filga de processos "Entregue" totalmente processada

## Arquivos Chave

| Arquivo | Função |
|---|---|
| `lancamento_debitos/src/kaizencrm.py` | Login, busca cliente, lança débitos |
| `lancamento_debitos/src/pdf_parser.py` | Extrai valores do REC.pdf |
| `lancamento_debitos/src/main.py` | Orquestração do lançamento |
| `lancamento_debitos/src/utils.py` | Checkpoint, helpers |
| `anexa_documentos/src/kaizencrm.py` | Login, busca, anexa documentos |
| `atualiza_tarefas/src/kaizencrm.py` | Login, busca, atualiza tarefas |
| `agendador/rodar_tudo.ps1` | Orquestrador central |
| `.env` | Credenciais e configurações |
| `checkpoint_debitos.json` | Estado dos débitos lançados |
