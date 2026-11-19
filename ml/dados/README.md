# Dados utilizados

## Fonte
- **Nome:** Dados do SP156 – Solicitações
- **Responsável:** Prefeitura de São Paulo – Secretaria Municipal de Inovação e Tecnologia (SMIT)
- **Link:** https://dados.prefeitura.sp.gov.br/dataset/dados-do-sp156
- **Arquivo usado:** "Dados do SP156 – 2º TRI 2026" (XLSX), aba "Solicitações"
- **Dicionário dos dados:** "Dicionário dados abertos – SP156 2020 em diante", na mesma página
- **Licença:** Creative Commons CCZero (uso livre)
- **Tipo:** dados **reais e públicos**, com informações pessoais removidas pela própria Prefeitura

## Período
| Uso | Período | Reclamações |
|---|---|---|
| Arquivo completo | 01/04/2026 a 30/06/2026 | 463.686 (todos os temas) |
| Após filtro e limpeza | 01/04/2026 a 30/06/2026 | 87.238 (Trânsito e Transporte) |
| Treino | abril e maio de 2026 | 57.957 |
| Teste | junho de 2026 | 29.281 |

Segundo o portal, os relatórios foram extraídos em **19/10/2026**, então o status de cada reclamação é o daquela data.

## Como baixar
O arquivo não está no repositório porque passa do limite de upload do GitHub (cerca de 40 MB).

1. Baixe o arquivo pelo link direto:
   https://dados.prefeitura.sp.gov.br/dataset/0aecfa2b-aa3a-40d4-8183-0d4351b7fd0a/resource/3ae65d07-d3e2-417a-becc-659a51cdce57/download/arquivo-final-2-trim-26.xlsx
2. Coloque o arquivo `arquivo-final-2-trim-26.xlsx` **na mesma pasta do notebook** (`ml/`) e rode o notebook.

O notebook já está salvo **executado**, então todos os resultados e gráficos podem ser vistos no GitHub sem baixar a base.

## Colunas usadas
| Coluna original | Uso no modelo |
|---|---|
| Data de Abertura | dia do mês e dia da semana |
| Status | **alvo**: Finalizada = 1 · Aberto / Em Andamento = 0 · Cancelada = removida |
| Tema | filtro: só "Trânsito e Transporte" |
| Assunto, Serviço | variáveis |
| Prefeitura Operacional (subprefeitura), Distrito | variáveis (vazio → "NAO INFORMADO") |
| Meio de Comunicação | variável (canal) |

**Colunas não usadas, de propósito:** Data de Finalização e Órgão. Elas só existem depois que a Prefeitura responde, e usá-las vazaria a resposta para o modelo.
