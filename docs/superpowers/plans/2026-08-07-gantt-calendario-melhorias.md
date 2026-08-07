# Melhorias Gantt + Calendário Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** No Gantt, ocultar colunas de dias sem nenhuma ausência pra dar mais espaço ao texto das barras; no Calendário, mostrar quem são as pessoas escondidas atrás do "+N mais" ao passar o mouse.

**Architecture:** Projeto é um único arquivo estático `index.html` (HTML/CSS Tailwind/JS vanilla, sem build, sem framework de testes). As duas mudanças são edições cirúrgicas dentro de `renderGanttTimeline()` e `renderCalendar()`, sem novos arquivos nem dependências. Verificação é manual via navegador, servindo o arquivo com `python3 -m http.server`.

**Tech Stack:** HTML5, Tailwind (CDN), JS vanilla, Chart.js (CDN), Google Sheets CSV como fonte de dados.

## Global Constraints

- Zero build complexity: continua um único `index.html`, sem novas dependências. (spec: "Fora de escopo")
- Sem indicador visual de "salto" entre colunas não-consecutivas no Gantt. (spec: Mudança 1)
- Tooltip do "+N mais" usa o atributo `title` nativo, mesmo padrão já usado nas barras do Gantt e chips do calendário — sem popover customizado. (spec: Mudança 2)
- Não mexer em `renderAgenda`/`renderTimedCard` (views Dia/3 Dias) nem no cálculo de `monthRecords`/`employees` já existente. (spec: "Fora de escopo")

---

### Task 1: Ocultar dias sem ausência no Gantt

**Files:**
- Modify: `index.html:956-1061` (função `renderGanttTimeline`)

**Interfaces:**
- Consumes: `filteredRecords`, `currentDate`, `dateToComparable()`, `typeStyle()`, `timeRangeLabel()`, `escapeHtml()` — todos já existentes, sem mudança de assinatura.
- Produces: nenhuma interface nova exposta a outras funções — `renderGanttTimeline` continua sem parâmetros e sem retorno, só muda o HTML que gera.

- [ ] **Step 1: Ler o estado atual da função para confirmar linhas exatas**

Abra `index.html` e confirme que as linhas 956-1061 ainda correspondem a `renderGanttTimeline()` sem mudanças desde a leitura original (o arquivo pode ter sido tocado por outra sessão). Se os números de linha mudaram, localize a função pelo nome e ajuste os steps abaixo pelo conteúdo, não pelo número de linha.

- [ ] **Step 2: Adicionar o cálculo de `activeDays` logo após `monthRecords`**

Localize este trecho (logo antes de `const employees = ...`):

```javascript
            const monthRecords = filteredRecords.filter(r => {
                const startStr = dateToComparable(r.dataInicio);
                const endStr = dateToComparable(r.dataFim);
                return startStr && endStr && endStr >= monthStartStr && startStr <= monthEndStr;
            });

            const employees = [...new Set(monthRecords.map(r => r.nome))].sort();
```

Substitua por:

```javascript
            const monthRecords = filteredRecords.filter(r => {
                const startStr = dateToComparable(r.dataInicio);
                const endStr = dateToComparable(r.dataFim);
                return startStr && endStr && endStr >= monthStartStr && startStr <= monthEndStr;
            });

            // Só entram como coluna os dias em que pelo menos um registro tem ausência —
            // libera espaço horizontal pro texto das barras.
            const activeDaysSet = new Set();
            monthRecords.forEach(r => {
                const startStr = dateToComparable(r.dataInicio);
                const endStr = dateToComparable(r.dataFim);
                const clippedStart = startStr < monthStartStr ? 1 : parseInt(startStr.split('-')[2], 10);
                const clippedEnd = endStr > monthEndStr ? daysInMonth : parseInt(endStr.split('-')[2], 10);
                for (let d = clippedStart; d <= clippedEnd; d++) activeDaysSet.add(d);
            });
            const activeDays = [...activeDaysSet].sort((a, b) => a - b);
            const dayColumnIndex = new Map(activeDays.map((d, i) => [d, i]));

            const employees = [...new Set(monthRecords.map(r => r.nome))].sort();
```

- [ ] **Step 3: Trocar o loop do cabeçalho de `1..daysInMonth` para `activeDays`**

Localize:

```javascript
            for (let d = 1; d <= daysInMonth; d++) {
                const dateObj = new Date(year, month, d);
                const dayOfWeek = dateObj.getDay();
                const isWeekend = dayOfWeek === 0 || dayOfWeek === 6;

                html += `<div class="flex-1 text-center py-3 border-r border-slate-200 ${isWeekend ? 'bg-slate-200/50 text-slate-600' : 'bg-slate-50'}">
                            <div class="font-bold">${d}</div>
                            <div class="text-[9px] font-semibold uppercase">${['D','S','T','Q','Q','S','S'][dayOfWeek]}</div>
                         </div>`;
            }
```

Substitua por:

```javascript
            activeDays.forEach(d => {
                const dateObj = new Date(year, month, d);
                const dayOfWeek = dateObj.getDay();
                const isWeekend = dayOfWeek === 0 || dayOfWeek === 6;

                html += `<div class="flex-1 text-center py-3 border-r border-slate-200 ${isWeekend ? 'bg-slate-200/50 text-slate-600' : 'bg-slate-50'}">
                            <div class="font-bold">${d}</div>
                            <div class="text-[9px] font-semibold uppercase">${['D','S','T','Q','Q','S','S'][dayOfWeek]}</div>
                         </div>`;
            });
```

- [ ] **Step 4: Trocar o loop das células de fundo de `1..daysInMonth` para `activeDays`**

Localize (dentro do `employees.forEach`):

```javascript
                for (let d = 1; d <= daysInMonth; d++) {
                    const isWeekend = new Date(year, month, d).getDay() % 6 === 0;
                    html += `<div class="flex-1 border-r border-slate-200 h-full ${isWeekend ? 'bg-slate-50/60' : ''}"></div>`;
                }
```

Substitua por:

```javascript
                activeDays.forEach(d => {
                    const isWeekend = new Date(year, month, d).getDay() % 6 === 0;
                    html += `<div class="flex-1 border-r border-slate-200 h-full ${isWeekend ? 'bg-slate-50/60' : ''}"></div>`;
                });
```

- [ ] **Step 5: Reposicionar as barras usando o índice em `activeDays` em vez da posição real no mês**

Localize:

```javascript
                    const clippedStart = startStr < monthStartStr ? 1 : parseInt(startStr.split('-')[2], 10);
                    const clippedEnd = endStr > monthEndStr ? daysInMonth : parseInt(endStr.split('-')[2], 10);
                    const spanDays = clippedEnd - clippedStart + 1;

                    const leftPct = ((clippedStart - 1) / daysInMonth) * 100;
                    const widthPct = (spanDays / daysInMonth) * 100;
```

Substitua por:

```javascript
                    const clippedStart = startStr < monthStartStr ? 1 : parseInt(startStr.split('-')[2], 10);
                    const clippedEnd = endStr > monthEndStr ? daysInMonth : parseInt(endStr.split('-')[2], 10);
                    const spanDays = clippedEnd - clippedStart + 1;

                    const colStart = dayColumnIndex.get(clippedStart);
                    const colEnd = dayColumnIndex.get(clippedEnd);
                    const leftPct = (colStart / activeDays.length) * 100;
                    const widthPct = ((colEnd - colStart + 1) / activeDays.length) * 100;
```

`spanDays` continua usado logo abaixo (na escolha do label `spanDays >= 3 ? r.tipo : ...`) — não remova essa linha.

- [ ] **Step 6: Verificar sintaxe do arquivo**

Rode:

```bash
node --check <(sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d')
```

Expected: nenhum output (sem erro de sintaxe). Se o comando falhar por causa do `<script src=...>` das CDNs misturado no meio, ignore erros de "Unexpected token" fora da região das linhas 956-1061 — o objetivo aqui é só confirmar que a função editada não quebrou a sintaxe. Alternativa mais simples: abra o DevTools do navegador no Step 7 e confira que não há erro de parsing no console.

- [ ] **Step 7: Verificar visualmente no navegador**

```bash
cd ~/projetos/dashboard-folgas && python3 -m http.server 8080
```

Abra `http://localhost:8080` no navegador, vá pra view **Timeline (Gantt)**, escolha um mês com ausências espalhadas (não todo santo dia) e confirme:
- Só aparecem colunas de dias com pelo menos uma ausência.
- O texto dentro das barras (tipo ou horário) não trunca mais que antes — idealmente menos.
- Uma barra de período multi-dia (ex: férias de vários dias seguidos) continua contígua, sem cortes ou buracos.
- O DevTools Console não mostra nenhum erro novo.

- [ ] **Step 8: Commit**

```bash
cd ~/projetos/dashboard-folgas
/usr/bin/git add index.html
/usr/bin/git commit -m "$(cat <<'EOF'
feat: ocultar dias sem ausência no Gantt pra dar mais espaço ao texto

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Tooltip com nomes no "+N mais" do Calendário

**Files:**
- Modify: `index.html:1268-1270` (dentro de `renderCalendar`)

**Interfaces:**
- Consumes: `dayLeaves` (array de records do dia, já filtrado/ordenado por `renderCalendar`), `escapeHtml()` — existentes, sem mudança de assinatura.
- Produces: nenhuma interface nova.

- [ ] **Step 1: Confirmar as linhas atuais**

Confirme que o trecho abaixo ainda está em `renderCalendar()` (pode ter deslocado de linha por causa do Task 1, já que está numa função diferente do arquivo — a edição do Task 1 não deveria mover essas linhas, mas confirme pelo conteúdo):

```javascript
                if (dayLeaves.length > 3) {
                    html += `<div class="text-[9px] text-slate-400 font-semibold text-center">+${dayLeaves.length - 3} mais</div>`;
                }
```

- [ ] **Step 2: Adicionar o `title` com a lista de nomes**

Substitua por:

```javascript
                if (dayLeaves.length > 3) {
                    const extraNames = dayLeaves.slice(3)
                        .map(l => `${l.nome} — ${l.tipo}`)
                        .join('\n');
                    html += `<div class="text-[9px] text-slate-400 font-semibold text-center" title="${escapeHtml(extraNames)}">+${dayLeaves.length - 3} mais</div>`;
                }
```

- [ ] **Step 3: Verificar visualmente no navegador**

Com o servidor do Task 1 ainda rodando (ou suba de novo: `python3 -m http.server 8080` em `~/projetos/dashboard-folgas`), abra a view **Calendário**, encontre um dia com mais de 3 ausências (ou ajuste temporariamente o filtro/mês pra achar um) e confirme:
- O card mostra "+N mais" normalmente.
- Passar o mouse por cima exibe o tooltip nativo do navegador listando "Nome — Tipo" de cada pessoa a mais, uma por linha.
- Nenhum erro no DevTools Console.

- [ ] **Step 4: Commit**

```bash
cd ~/projetos/dashboard-folgas
/usr/bin/git add index.html
/usr/bin/git commit -m "$(cat <<'EOF'
feat: tooltip com nomes no +N mais do calendário de folgas

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```
