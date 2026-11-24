# databricks-cgn

Case de priorização de leads da Captação: uma base simulada com as restrições do cenário e uma V1 de score que ordena a fila do comercial no lugar do FIFO, com as decisões técnicas de cada etapa.

## Estrutura

```
notebooks/
  01_gerar_dados.ipynb        gera as tabelas em cogna.captacao
  02_composicao_dados.ipynb   mostra como a base ficou composta
  03_score_v1.ipynb           variáveis por lead, score V1 e comparação com o FIFO
```

Os notebooks foram exportados do Databricks com os resultados da última execução, então dá para ver as saídas direto no GitHub.

## Tabelas

| Tabela | Colunas |
|---|---|
| `tb_leads` | lead_id, dt_criacao, curso, modalidade, canal_origem, fl_matricula, dt_matricula |
| `tb_abordagem_crm` | lead_id, dt_tentativa_contato, canal_contato, status_contato |
| `tb_navegacao` | session_id, lead_id, ts_evento, pagina, tempo_permanencia_seg |

Além delas, o `02` mantém a `tb_nav_sessao`, com a classificação de cada sessão de navegação, e o `03` cria `tb_features_lead` (variáveis por lead), `tb_pesos_score` (pontos de cada faixa) e `tb_score_lead` (score de cada lead).

Números aproximados: 280 mil leads em 180 dias, 120 mil tentativas de contato e 6,5 milhões de eventos de navegação.

## Como a lógica foi montada

- **Leads**: entram todos os dias, com menos volume no fim de semana e pico em julho e novembro.
- **canal_origem (~25% nulo)**: orgânico e e-mail chegam sem utm com mais frequência, então o nulo tem causa.
- **Abordagem**: o comercial trabalha em dias úteis, das 9h às 18h. Lead do fim de semana, de feriado ou de depois das 18h entra na fila do próximo dia útil. Em cada dia útil são abordados os primeiros 20% da fila, com até 3 tentativas.
- **Matrícula (~3% nos abordados)**: depende do perfil do lead (canal, curso, modalidade, horário) e de ele ter sido atendido. Matrícula depois da data de corte não entra, e `dt_matricula` só existe quando `fl_matricula = 1`.
- **Navegação (~40% com lead_id)**: antes do cadastro a navegação é anônima; a sessão do cadastro liga as duas partes pelo `session_id`. Visitantes que nunca viraram lead completam o volume anônimo.

Os parâmetros (período, volume, capacidade, feriados) ficam no começo do `01_gerar_dados`.

## Decisões técnicas

- **Feito para escalar**: a geração roda toda em Spark, sem processamento no driver. O mesmo código serve para bilhões de linhas.
- **Volume menor, proporções iguais**: a navegação tem milhões de eventos e não bilhões, para rodar rápido e barato, mantendo 40% identificado, 25% de canal nulo e 20% de cobertura.
- **Dados com problemas reais**: nulos, navegação anônima e leads recentes que ainda não converteram aparecem como em produção, para que o tratamento deles faça parte da discussão.
- **Abordagem fiel à operação**: só dias úteis e horário comercial, com a segunda-feira acumulando o fim de semana.

## Navegação com bilhões de linhas

- A `tb_navegacao` é clusterizada por data e lead, então consultas por período leem só os arquivos daquele período.
- A classificação das sessões fica gravada em `tb_nav_sessao`. Cada rodada do `02` processa só os eventos novos e classifica só as sessões que ainda não existem; o que já foi classificado não é reprocessado.
- As análises leem esse resumo e uma janela de tempo da tabela bruta, nunca o histórico inteiro.
- Distribuições usam amostra e contagens distintas são aproximadas.

## Score V1

- **Variáveis** só com o que existe no momento de priorizar: curso, modalidade, canal, período e tipo de dia de entrada, e navegação até o lead entrar na fila (sessões, tempo no site, visitas a preço, bolsas e inscrição). Nada de CRM, data de matrícula ou páginas de pós-matrícula.
- **Score por pontos**: cada faixa ganha pontos pela conversão histórica, aprendida nos leads que o FIFO já abordou. Cabe no prazo, é explicável para o comercial e os pesos ficam auditáveis.
- **Avanço sobre o FIFO**: como o FIFO escolhe por hora de chegada, os leads abordados por ele servem de amostra sem viés de intenção. Num período fora do aprendizado, os 20% de maior score converteram cerca de 2 vezes a média do FIFO e concentraram mais de 40% das matrículas. O ganho real se confirma em produção mantendo parte da capacidade no FIFO.

## Como rodar

1. No Databricks, crie o catálogo `cogna` (Catalog > Create catalog, usando o Default Storage). O notebook cria o schema `captacao` sozinho.
2. Ligue este repositório ao workspace em Workspace > Create > Git folder, com a URL `https://github.com/brocchirodrigo/databricks-cgn`.
3. Abra `notebooks/01_gerar_dados` e rode tudo em Serverless. Leva cerca de 1 minuto.
4. Abra `notebooks/02_composicao_dados` e rode tudo para ver volumes, distribuições, cobertura do CRM, navegação e checagens de integridade.
5. Abra `notebooks/03_score_v1` e rode tudo para gerar as variáveis, o score e a comparação com o FIFO.

Rodar o 01 de novo recria a base e zera a `tb_nav_sessao`. Rodar o 02 de novo só classifica sessões novas.
