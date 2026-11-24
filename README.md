# databricks-cgn

Base simulada para o case de priorização de leads da Captação, com as decisões técnicas por trás de cada etapa: como os dados foram modelados para refletir as restrições do cenário e como o processamento foi pensado para escalar no Databricks.

## Estrutura

```
notebooks/
  01_gerar_dados.ipynb        gera as tabelas em cogna.captacao
  02_composicao_dados.ipynb   mostra como a base ficou composta
```

Os notebooks foram exportados do Databricks com os resultados da última execução, então dá para ver as saídas direto no GitHub.

## Tabelas

| Tabela | Colunas |
|---|---|
| `tb_leads` | lead_id, dt_criacao, curso, modalidade, canal_origem, fl_matricula, dt_matricula |
| `tb_abordagem_crm` | lead_id, dt_tentativa_contato, canal_contato, status_contato |
| `tb_navegacao` | session_id, lead_id, ts_evento, pagina, tempo_permanencia_seg |

Números aproximados: 280 mil leads em 180 dias, 124 mil tentativas de contato e 6,5 milhões de eventos de navegação. No case real a navegação tem bilhões de linhas. Aqui o volume foi reduzido para rodar no serverless sem custo alto, mantendo as proporções.

## Como a lógica foi montada

- **Leads**: o volume diário varia por dia da semana (fim de semana menor) e por mês (pico em julho e novembro). Curso, modalidade e canal são sorteados com pesos diferentes.
- **canal_origem (~25% nulo)**: a chance de vir vazio depende do canal. Orgânico e e-mail chegam sem utm com mais frequência, então o nulo não é aleatório.
- **Abordagem (FIFO, 20%/dia)**: dentro de cada dia os leads são ordenados pela hora de entrada e só os primeiros 20% entram no CRM, com até 3 tentativas por WhatsApp ou telefone.
- **Matrícula (~3% nos abordados)**: cada lead tem uma propensão que depende de canal, curso, modalidade e horário. A matrícula é sorteada a partir dela, e ser atendido aumenta a chance. Matrícula que cairia depois do fim do período fica como 0. `dt_matricula` só é preenchida quando `fl_matricula = 1`.
- **Navegação (~40% com lead_id)**: cada lead tem sessões antes e depois do cadastro. Antes do cadastro não há lead_id; na sessão do cadastro só os eventos a partir do formulário são identificados, e o `session_id` liga as duas partes. Visitantes que nunca viraram lead completam o volume anônimo.

Os parâmetros principais (período, volume, capacidade de 20%) ficam no começo do `01_gerar_dados`.

## Decisões técnicas

- **Spark nativo em vez de UDF**: toda a geração usa funções do `pyspark.sql.functions` (`rand`, `randn`, `sequence` + `explode`). Nada passa por UDF Python nem pelo driver, então o mesmo código escala do volume atual para bilhões de linhas só aumentando partições.
- **Janelas sempre particionadas**: o `lead_id` é numerado por dia (`PARTITION BY dt`) e não com um `row_number()` global, que jogaria tudo numa partição só. A mesma numeração já define a ordem FIFO, então a etapa do CRM não precisa de outra janela.
- **Staging em Delta em vez de cache**: no serverless não existe `cache()`/`persist()`. As etapas intermediárias são gravadas em tabelas `stg_` e lidas de volta, o que também garante que CRM, matrícula e navegação partem exatamente da mesma base de leads. No fim elas são removidas.
- **Volume reduzido, proporção mantida**: a navegação tem 6,5 milhões de eventos e não bilhões, para rodar rápido e barato. O que importa para o case é manter as proporções: 40% identificado, 25% de canal nulo, 20% de cobertura.
- **Nulos e anonimato com causa**: o canal vazio e os eventos sem `lead_id` seguem uma regra (canal sem utm, navegação antes do cadastro), como acontece em produção. Assim a base permite discutir tratamento de nulo e identity stitching via `session_id` de forma realista.
- **Censura no fim do período**: matrícula que aconteceria depois da data de corte não entra. Leads recentes parecem converter menos, o que obriga a pensar na janela de maturação antes de treinar ou avaliar qualquer coisa.
- **Análise com uma leitura por tabela grande**: no `02`, a navegação é agregada por sessão uma única vez em PySpark e essa base serve para todas as análises seguintes. O CRM vira uma linha por lead com `PARTITION BY lead_id` e é cruzado com os leads uma vez só. O SQL fica para as consultas leves e legíveis.
- **Reprodutível**: todas as funções aleatórias usam seed fixa. Nas execuções feitas, os volumes bateram exatamente entre uma rodada e outra.

## Como rodar

1. No Databricks, crie o catálogo `cogna` (Catalog > Create catalog, usando o Default Storage). O notebook cria o schema `captacao` sozinho.
2. Ligue este repositório ao workspace em Workspace > Create > Git folder, com a URL `https://github.com/brocchirodrigo/databricks-cgn`.
3. Abra `notebooks/01_gerar_dados` e rode tudo em Serverless. Leva menos de 1 minuto.
4. Abra `notebooks/02_composicao_dados` e rode tudo para ver volumes, distribuições, cobertura do CRM, navegação e checagens de integridade.

Rodar o 01 de novo sobrescreve as tabelas.
