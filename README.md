# databricks-cgn

Base simulada para o case de priorização de leads da Captação.

O cenário: o comercial (Call Center/WhatsApp) consegue abordar só 20% dos leads por dia, hoje por ordem de chegada (FIFO), com conversão em torno de 3%. A ideia é ter uma V1 que priorize esses leads em 5 dias úteis. Este repositório não traz a solução. Ele monta os dados para discutir e construir a solução em cima deles.

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

## Como rodar

1. No Databricks, crie o catálogo `cogna` (Catalog > Create catalog, usando o Default Storage). O notebook cria o schema `captacao` sozinho.
2. Ligue este repositório ao workspace em Workspace > Create > Git folder, com a URL `https://github.com/brocchirodrigo/databricks-cgn`.
3. Abra `notebooks/01_gerar_dados` e rode tudo em Serverless. Leva menos de 1 minuto.
4. Abra `notebooks/02_composicao_dados` e rode tudo para ver volumes, distribuições, cobertura do CRM, navegação e checagens de integridade.

Rodar o 01 de novo sobrescreve as tabelas.
