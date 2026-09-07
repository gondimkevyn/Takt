# PROMPT COMPLETO DE ESPECIFICAÇÃO: APLICAÇÃO TAKT GO (PLANEJAMENTO DE CAPACIDADE INDUSTRIAL)

Desenvolver um aplicativo nativo Android em Kotlin chamado TAKT GO, focado em Planejamento de Capacidade Industrial e Takt Time, projetado para substituir planilhas manuais em Excel por uma aplicação centralizada, reativa e relacional.

==================================================
1. DIRETRIZES FUNDAMENTAIS & UX
==================================================

- A aplicação deve iniciar 100% LIMPA (sem nenhum cliente, produto, linha ou planejamento pré-cadastrado no código).
- Na primeira inicialização (ou quando o banco de dados estiver vazio), exibir uma Tela de Boas-Vindas orientando o usuário a cadastrar os dados mestres ou criar o primeiro planejamento.
- Todos os dados (clientes, produtos, linhas, tarefas de postos, turnos e itens de planejamento) devem ser inteiramente cadastráveis, editáveis (CRUD) e removíveis pelo usuário através da interface.
- O planejamento mensal NUNCA trava: o usuário pode clicar em [Editar] em qualquer mês já criado para alterar Forecast, Turnos, Setups ou Backlog e ver a capacidade recalculada instantaneamente.
- Validação de Código/ID Único: Impedir a gravação de Clientes, Produtos ou Linhas Produtivas com códigos/SKUs repetidos, exibindo aviso visual de erro.

==================================================
2. ARQUITETURA DE CÓDIGO
==================================================

- Linguagem: Kotlin nativo.
- Interface UI: Jetpack Compose com Material 3 (`ModalNavigationDrawer` para o menu lateral deslizante).
- Banco de Dados: Room Database (SQLite local) versão relacional normalizada.
- Arquitetura: MVVM (Model-View-ViewModel) + Repository Pattern + Kotlin Flow.
- Módulos de Cálculo Isolados: As fórmulas industriais e relatórios devem ficar isolados em objetos de serviço puros (`CapacityCalculationService`, `MonthlyReportService`, `SimulationService`).

==================================================
3. MODELAGEM DO BANCO DE DADOS (ROOM ENTITIES)
==================================================

1. `clients` (ClientEntity):
   - id: Long (PK auto)
   - name: String
   - code: String (Validação de código único)
   - description: String
   - status: String ("Ativo")

2. `products` (ProductEntity):
   - id: Long (PK auto)
   - clientId: Long
   - clientName: String
   - name: String
   - code: String (Validação de SKU único)
   - demandQuantity: Int (Quantidade desejada pelo cliente)
   - botTopType: String ("BOT/TOP")

3. `production_lines` (ProductionLineEntity):
   - id: Long (PK auto)
   - code: String (Ex: "ML01" - Validação de código único)
   - name: String
   - area: String
   - maxShifts: Int (3)

4. `station_tasks` (StationTaskEntity):
   - id: Long (PK auto)
   - stationName: String (Nome do Posto de Trabalho, ex: "Posto 01")
   - lineCode: String
   - taskName: String (Nome da Tarefa/Operação)
   - t1, t2, t3, t4, t5: Double (Amostras de tempo de cronoanálise em segundos)
   - executionTimeSeconds: Double (Média calculada)

5. `work_shifts` (ShiftConfigEntity):
   - id: Long (PK auto)
   - name: String (Ex: "1º Turno")
   - startTime: String ("06:00")
   - endTime: String ("15:48")
   - lunchDinnerMin: Int (60)
   - smallBreakMin: Int (15)
   - gymnasticsMin: Int (10)
   - netAvailableHours: Double (Horas úteis líquidas descontando paradas)

6. `monthly_plans` (MonthlyPlanEntity):
   - id: Long (PK auto)
   - name: String (Ex: "Setembro / 2026")
   - month: Int (1 a 12)
   - year: Int (2026)
   - workingDays: Int (21)
   - status: String ("Rascunho", "Aprovado", "Fechado")

7. `monthly_plan_items` (MonthlyPlanItemEntity):
   - id: Long (PK auto)
   - planId: Long
   - lineCode: String
   - clientName: String
   - modelName: String
   - uph: Double
   - forecastMonth: Int
   - prevBacklog: Int
   - prevAntecipation: Int
   - activeShiftsCount: Int (1, 2 ou 3)
   - setupQty: Int
   - hoursPerSetup: Double
   - combinedHeadcount: Int

==================================================
4. REGRAS DE CÁLCULO E FÓRMULAS
==================================================

1. Volume Mensal:
   Volume Mensal = Forecast + Backlog Anterior - Antecipação Anterior + Backlog Próximo - Antecipação Próxima

2. Cálculo Automático do UPH (Unidades por Hora):
   - O UPH NÃO é digitado manualmente. Ele é derivado das Tarefas do Posto Gargalo da linha:
     Tempo do Posto = Soma do Tempo Médio das Tarefas daquele Posto
     Tempo do Posto Gargalo (T_max) = Maior Tempo entre os Postos da Linha
     UPH = 3600 segundos / T_max

3. Horas Necessárias de Produção:
   Horas Produção = Volume Mensal / UPH

4. Horas de Setup:
   Horas Setup = Quantidade de Setups × Horas por Setup

5. Horas Necessárias Totais:
   Total Need = Horas Produção + Horas Setup + Horas NPI + Horas Ramp-up + Horas Manutenção

6. Horas Disponíveis Totais:
   Total Avail = (Dias Produtivos × Turnos Ativos × Horas Úteis do Turno) + Horas Extras - Descontos

7. Utilização da Linha (%):
   Utilização = (Total Need / Total Avail) × 100

8. Classificação de Status da Linha:
   - 0% a 79.9%: Normal
   - 80% a 94.9%: Atenção
   - 95% a 100%: Crítico
   - Acima de 100%: Capacidade Insuficiente (Sobrecarga)

==================================================
5. MÓDULOS E ESTRUTURA DE TELAS DE UI
==================================================

Menu Lateral Deslizante (ModalNavigationDrawer):
1. Dashboard:
   - Stepper do Fluxo Guiado de Planejamento (6 passos).
   - Seletor do Mês de Planejamento.
   - Botões: [Novo Mês], [✏️ Editar Itens], [Relatório].
   - Cards com indicadores: Demanda Total, Forecast, Backlog, Horas Necessárias, Horas Disponíveis, Utilização Média, Headcount Total.
   - Barra de Progresso de Utilização por Linha de Produção.
   - Bloco de Alertas Dinâmicos (gargalos, sobrecarga > 100%, escassez de horas, produtos sem UPH).

2. Planejamento Mensal:
   - Tabela principal de capacidade responsiva (scroll horizontal) com colunas: Linha, Tipo, Cliente, Modelo, UPH, Cap 1/2/3 Turnos, Forecast, Backlog, Antecipação, Volume Mensal, Horas Need, Setups, Headcount, Status.
   - Botões [Editar] e [Excluir] em cada linha para modificação livre.
   - Diálogo modal para adicionar/editar itens ao planejamento.

3. Cadastros (CRUD Completo com edição em todas as abas):
   - Aba Clientes: Nome, Código Único, Descrição.
   - Aba Produtos: Seleção de Cliente, Nome do Modelo, Código/SKU Único, Demanda do Cliente em Peças. Exibição dos produtos agrupados/filtrados por cliente.
   - Aba Linhas: Código Único (ex: ML01), Nome, Área/Setor.
   - Aba Postos & Tarefas: Seleção de Posto Existente ou criação de Novo Posto. Adição de Múltiplas Tarefas/Operações por Posto. Amostras de cronoanálise (T1, T2, T3, T4 em segundos). Cálculo automático da Média da Tarefa e Tempo Total do Posto em segundos.

4. Turnos & Horários:
   - Cadastro de Turnos com horário de início/fim e minutos de almoço, pausas e ginástica laboral. Cálculo de horas úteis líquidas disponíveis. Botão [Editar] em cada turno.

5. Operação & Parâmetros:
   - Visualização de parâmetros de Setups, NPI, Ramp-up, Manutenção e Headcount.

6. Engenharia (Time Report):
   - Cadastro de medição de tempos para estudos de tempos e métodos (T1 a T5, Média e Tempo Padrão).

7. Simulações:
   - Módulo para comparar cenários (Oficial vs Simulado) para análise de sensibilidade.

8. Relatório Mensal:
   - Emissão do relatório consolidado do mês selecionado contendo:
     - Resumo do Período (dias produtivos, demanda, horas disponíveis e necessárias).
     - Plano Operacional do Mês (texto explicativo legível de como a fábrica funcionará).
     - Recomendações do Planejamento (alertas operacionais).
