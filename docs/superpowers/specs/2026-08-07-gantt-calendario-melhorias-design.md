# Melhorias Gantt + Calendário — Design

**Data:** 2026-08-07
**Status:** Aprovado

## Contexto

Dashboard de Folgas (`index.html`, arquivo único). Duas views precisam de ajustes:

1. **Gantt (Timeline)** — `renderGanttTimeline()`, linha ~956. Hoje desenha uma coluna por
   dia do mês (1 a `daysInMonth`), largura igual, mesmo em dias sem nenhuma ausência.
   Isso desperdiça espaço horizontal e espreme o texto dentro das barras.
2. **Calendário Mensal** — `renderCalendar()`, linha ~1234. Quando um dia tem mais de 3
   ausências, mostra só `+N mais` sem indicar quem são essas pessoas.

## Mudança 1 — Ocultar dias sem ausência no Gantt

- Calcular `activeDays`: lista ordenada dos dias do mês (1..`daysInMonth`) em que pelo
  menos um registro de `monthRecords` cobre aquele dia (considerando o recorte de início/fim
  já aplicado ao mês navegado).
- Cabeçalho (`html += ... for d 1..daysInMonth`) e células de fundo por colaborador passam
  a iterar sobre `activeDays` em vez do range fixo — número de colunas cai, cada uma fica
  mais larga.
- Barras de período: posição (`left`/`width` em %) passa a ser calculada pelo **índice do
  dia dentro de `activeDays`**, não mais pela posição real no mês
  (`leftPct = index/activeDays.length * 100`).
- Como todo dia coberto por uma barra é por definição um dia ativo (o próprio registro o
  torna ativo), uma barra sempre ocupa um intervalo de índices consecutivos em
  `activeDays` — nunca fica cortada, mesmo pulando dias ocultos no meio.
- **Sem indicador visual de "salto"** entre colunas não-consecutivas (decisão do usuário) —
  o número do dia em cada coluna já basta como referência.
- Fins de semana continuam destacados normalmente nas colunas que sobrarem (a lógica de
  `isWeekend` por `dayOfWeek` não muda, só passa a rodar sobre `activeDays`).
- Caso `employees.length === 0` (já tratado) permanece igual. `activeDays` nunca fica vazio
  quando há `employees`, porque todo `monthRecord` que gera um funcionário também gera pelo
  menos um dia ativo.

## Mudança 2 — Tooltip nativo no "+N mais" do Calendário

- Quando `dayLeaves.length > 3`, o `<div>+N mais</div>` (linha ~1269) ganha um atributo
  `title` (mesmo padrão nativo já usado nas barras do Gantt e nos chips individuais do
  calendário) listando as pessoas que não aparecem nos 3 primeiros chips
  (`dayLeaves.slice(3)`), uma por linha: `Nome — Tipo`.
- Sem tooltip customizado — mantém consistência com o resto do dashboard, que já usa
  `title` nativo em todo lugar.

## Fora de escopo

- Não mexe em `renderAgenda`/`renderTimedCard` (views Dia/3 Dias).
- Não altera o cálculo de `monthRecords`/`employees` já existente.
- Sem mudança de dependências ou build — continua HTML/JS vanilla, arquivo único.

## Teste manual

1. Abrir view Timeline (Gantt) num mês com poucas ausências espalhadas — confirmar que só
   os dias com ausência aparecem como coluna, e que o texto dentro das barras não trunca
   mais (ou trunca menos) que antes.
2. Conferir que uma barra de período multi-dia (ex: férias de 5 dias) continua contígua e
   sem cortes, mesmo com dias ocultos antes/depois dela.
3. Abrir view Calendário num dia com mais de 3 ausências — passar o mouse sobre "+N mais" e
   confirmar que aparece a lista dos nomes/tipos restantes.
