# Fantasy Projects — Contexto Consolidado para Claude.ai

> Documento gerado em: 28/03/2026
> Propósito: carregar no Claude.ai (Project Knowledge) para dar contexto completo
> sobre o ecossistema de projetos Fantasy do Erico.

---

## Visao Geral do Ecossistema

Tres projetos independentes que se comunicam via bancos de dados compartilhados:

```
C:\Users\Erico Mello\Fantasy\
├── fantasy_manager/     ← Gestao operacional da liga (Port 5000)
├── fantasy_optimizer/   ← Analise e decisoes (Port 5001)
├── predictor/           ← Previsao de breakout (CLI)
└── pff_data/            ← Banco PFF compartilhado
```

### Perguntas que cada projeto responde

| Projeto | Pergunta | Interface |
|---------|----------|-----------|
| Fantasy Manager | "Qual o estado atual da minha liga?" | Flask (port 5000) |
| Fantasy Optimizer | "O que eu deveria fazer agora com este elenco?" | Flask (port 5001) |
| Predictor | "Quem vai estourar?" | CLI + publicacao no Optimizer |

---

## Fluxo de Dados

```
fantasy_manager
    ↓ dynasty.db (roster, contratos, salary, picks)
    ├──→ fantasy_optimizer/optimizer/loader.py (read-only)
    └──→ predictor/predictor/loader.py (read-only)

MYPFF_Complete.db (60.743 rows, 96 colunas — PFF stats)
    ├──→ fantasy_optimizer (grades, performance)
    └──→ predictor (NCAA + NFL historico, passing stats)

players_nfl.json (Sleeper API — 11.564 jogadores)
    ├──→ predictor (birth_date, depth_chart, combine matching)
    └──→ optimizer (filtro de jogadores ativos)

predictor → optimizer.db (tabela breakout_scores, 1.263 rows)
    └──→ optimizer_app/ (Dashboard, Cap, Trades, /predictor)
```

**Regra absoluta:** MYPFF_Complete.db e dynasty.db sao READ-ONLY para todos os projetos exceto o fantasy_manager (que escreve no dynasty.db).

---

## Fantasy Manager

- Flask + SQLAlchemy, port 5000
- Dono do dynasty.db: roster, contratos, salary cap ($200), picks, trades, leiloes
- 15+ tabelas (Team, Player, Pick, Trade, SalaryHistory, etc.)
- Sync com Sleeper API (rosters, drafts)
- Nao importa dos outros projetos

---

## Fantasy Optimizer

- Flask dashboard (port 5001) + modulos analiticos CLI
- 7 rotas: / (dashboard), /cap, /trades, /profiles, /flags, /predictor, /player/<name>
- Dono do optimizer.db: asset_scores, player_mapping, trade_proposals, player_flags, market_context, breakout_scores
- Modulos: value_model.py (scoring 0-100), trade_evaluator.py (7 niveis de veredicto), cap_optimizer.py (saude contratual 3 dimensoes)
- Performance score position-aware: RB usa rushing metrics, WR/TE usa receiving metrics
- Trade evaluation com 3 perfis (CONTENDER/RELOAD/REBUILD) e multiplicadores por fase sazonal

### Integracao com Predictor
- `predictor_bridge.py` le breakout_scores do optimizer.db
- 4 pontos de integracao: Dashboard (alertas), Cap (coluna FUT), Trades (badge), /predictor (rankings)
- Degradacao elegante: `—` quando sem scores
- Contrato formal: schema_version 2, flags validadas via contract_test.py

---

## Predictor

- CLI Python, sem interface propria
- 3 modelos de breakout: RBO (rookies), SBO (segundo ano), TBO (veteranos)
- 1.263 jogadores publicados no optimizer.db
- Backtest validado: 77% precision overall (RBO 67%, SBO 67%, TBO 97%)

### Modelos

**RBO (Rookie Breakout)** — 0 NFL seasons, 89 jogadores
- Elegibilidade: combine_nflverse.csv (prospects draft 2026)
- WR/TE: YPRR 35% + Target Share 30% + YPT 20% + Dominator 15% x Draft Multiplier
- HB: Rushing Profile 40% + Receiving Profile 30% + Target Share 15% + Dominator 15%

**SBO (Sophomore Breakout)** — 1 NFL season, 229 jogadores
- Growth Factor dual: volume (60%) + eficiencia (40%), zero normalizado em 60
- WR/TE: YPRR 25% + Target Rate 25% + YPT 20% + Route Participation 15% + Growth Factor 15%

**TBO (Trajectory Breakout)** — 2+ NFL seasons, 945 jogadores
- Peak Grade 30% + Trajectory 25% + Opportunity 20% + Draft Capital 15% + Age Window 10%
- Variancia Pontual: x1.15 quando trajectory declining + peak > 80
- RETORNO_ESPERADO: janela estendida +3 anos para peak > 85
- Depth chart via Sleeper API (depth_chart_order), fallback route_rate

### Resultados dos casos de validacao

| Jogador | Modelo | Score | Tier | Flags |
|---------|--------|-------|------|-------|
| Saquon Barkley | TBO | 77.1 | HIGH | RETORNO_ESPERADO, ELITE_PEAK |
| Jake Ferguson | TBO | 76.4 | HIGH | NO_PICO (TE de 27 no pico, nao veterano) |
| Jerome Ford | TBO | 50.9 | MEDIUM | POS_PICO, DECLINING |
| Rome Odunze | TBO | 77.0 | HIGH | JANELA_ABERTA, ASCENDING |
| Luther Burden III | SBO | 90.6 | ELITE | EFFICIENCY_MAINTAINED |

### Contrato com o Optimizer

Tabela `breakout_scores`: player_name, sleeper_id, composite_score, dominant_model, tier, rbo/sbo/tbo_score, season, details_json (schema_version 2 + flags), computed_at.
Validacao: `python -m predictor.contract_test`

---

## Bancos de Dados

### MYPFF_Complete.db
- 60.743 rows, 96 colunas (90 PFF + 6 integracao)
- Fontes: 69 CSVs (46 rushing/receiving + 23 passing)
- NFL: 2015-2026, NCAA: 2014-2026
- Colunas de integracao: gsis_id, sleeper_id, draft_year/round/pick, draft_team
- **CRITICO:** todas as colunas sao TEXT — usar CAST(col AS REAL)
- Rushing e receiving sao linhas separadas por jogador/temporada

### dynasty.db
- Fonte de verdade para roster, contratos, salary ($200 cap), picks
- Gerenciado pelo fantasy_manager
- Tabela principal: players (name, position, salary, contract_year, sleeper_player_id, fantasy_team)

### optimizer.db
- Estado analitico do Optimizer
- Tabelas principais: asset_scores, player_mapping (278 mapeamentos), trade_proposals, player_flags, breakout_scores
- breakout_scores pertence ao Predictor (nao dropar/alterar direto)

### predictor.db
- Estado interno do Predictor
- Tabelas: player_metadata (10.456 birth_dates), breakout_history, model_inputs, combine_player_mapping, backtest_results

---

## Backlog Ativo (optimizer_improvements.md)

Itens pendentes relevantes:

| ID | Item | Prioridade |
|---|---|---|
| A16 | Breakdown por componente na tela /predictor | Media |
| A17 | Destaque do roster na tela /predictor | Media |
| A3 | Redesign bloco de estados no Dashboard | Media |
| A14 | Tela dedicada de jogador | Media |
| B10 | Complementaridade de backfield no evaluator | Media |
| B6 | Integracao market_context no trade_ranker | Media |

---

## Versionamento (Git)

Todos os projetos estao versionados com git local:

| Projeto | Tag | Hash | Data |
|---------|-----|------|------|
| Fantasy Manager | `manager-v1.0` | `f2271ba` | 28/03/2026 |
| Predictor | `predictor-v1.0` | `52f6bb6` | 28/03/2026 |
| Fantasy Optimizer | `optimizer-post-predictor` | `a7de8d9` | 28/03/2026 |

Todos excluem `.env`, `*.db`, `__pycache__/`, `.claude/`, `data/` via `.gitignore`.

---

## Decisoes de Design Importantes

1. **MYPFF read-only absoluto** — nenhum projeto escreve no banco PFF
2. **Comunicacao via banco** — projetos nao importam codigo entre si, so leem tabelas
3. **Degradacao elegante** — cada projeto funciona mesmo sem os outros
4. **Position-aware scoring** — RB usa rushing metrics, WR/TE usa receiving metrics (nao misturar)
5. **HB peak grade = max(grades_run, grades_offense)** — captura impacto total de dual-threats
6. **Ferguson/Ford correcao** — devplan previa VETERANO_SEM_JANELA mas modelo corretamente identifica NO_PICO/POS_PICO
7. **Jogadores ativos filtrados** — /predictor filtra por players_nfl.json campo `team` (1.263 → 412 ativos)
8. **Contrato formal** — schema_version 2, contract_test.py valida integracao
