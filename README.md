# Automação AutoDespa ↔ KaizenCRM

Sistema de automação para processos de primeiro emplacamento de veículos 0km, integrando o sistema AutoDespa ao KaizenCRM.

## Estrutura do Projeto

```
Autodespa-Kaizencrm/
├── lancamento_debitos/    # Lança débitos (honorários, taxa, IPVA) no KaizenCRM
├── anexa_documentos/      # Baixa e anexa documentos (CRLV, REC, IPVA) no KaizenCRM
├── atualiza_tarefas/      # Atualiza tarefas do fluxo (emplacamento, IPVA, rodízio)
├── agendador/             # Orquestrador central (rodar_tudo.ps1)
└── relatorios/            # Planilhas Excel geradas
```

## Automações

### 1. Lançamento de Débitos
- Acessa AutoDespa → filtra processos "Entregue"
- Para cada processo: extrai placa, CPF, valores do REC.pdf
- Busca cliente no KaizenCRM → lança honorários, Taxas e Valores de IPVA
- Checkpoint por débito (`checkpoint_debitos.json`) — zero duplicatas

### 2. Anexo de Documentos
- Baixa PDFs do AutoDespa (CRLV, REC, comprovante IPVA)
- Anexa no KaizenCRM com classificação automática
- Detecta CRLV por conteúdo (keywords + extração de texto)

### 3. Atualização de Tarefas
- Extrai datas do AutoDespa (inclusão, entrega, NFe, CRLV)
- Preenche "Primeiro Emplacamento" (status → Entrada)
- Atualiza "Isenção IPVA" e "Liberação Rodízio" → Entrada (se Em aberto/Parado)
- Exporta Excel com processos de primeiro emplacamento

## Execução

### Execução Manual
```powershell
# Lançamento de Débitos
cd lancamento_debitos
python main.py

# Anexa Documentos
cd anexa_documentos
python main.py

# Atualiza Tarefas
cd atualiza_tarefas
python main.py
```

### Execução Automatizada (Task Scheduler)
```powershell
# Orquestrador central — executa as 3 automações em sequência
cd agendador
.\rodar_tudo.ps1
```

**Agendamento**: Segunda a sexta, 08:00h, repetição a cada 1h por 8h

## Configuração

### Variáveis de Ambiente (`.env`)
```env
# AutoDespa
AUTODESPA_URL=https://autodespa-prod.deepmindia.com.br
AUTODESPA_EMAIL=...
AUTODESPA_SENHA=...

# KaizenCRM
KAIZEN_URL=https://fidelisencoes.kaizencrm.app.br
KAIZEN_EMAIL=...
KAIZEN_SENHA=...

# Limite de processos por execução
LIMITE_PROCESSOS=20
```

### Checkpoints
- `checkpoint_debitos.json` — débitos já lançados por processo
- `checkpoint_anexa.json` — processos já anexados
- `checkpoint_atualiza.json` — processos já atualizados

## Funcionalidades Principais

### Checkpoint por Débito
Cada débito é registrado individualmente no checkpoint, prevenindo duplicatas mesmo em caso de falha intermediária.

### Formatos de REC.pdf Suportados
- Padrão: `HONORARIOS DESP` + `PRIMEIRO EMPLACAMENTO- TAXA`
- Alternativo: `TAXAS, HONORARIO` (valor combinado)

### Tratamento de Erros
- Toast de sucesso detectado via polling (10s)
- Modal fechado = débito registrado (fallback)
- Busca KaizenCRM sempre navega para home antes de pesquisar
- Reabre modal antes de cada débito subsequente

## Logs

Logs centralizados em `C:\automacao\logs\rodar_tudo.log`

## Requisitos

- Python 3.10+
- Playwright
- pdfplumber
- openpyxl (para Excel)

## Instalação

```bash
pip install playwright pdfplumber openpyxl
playwright install chromium
```
