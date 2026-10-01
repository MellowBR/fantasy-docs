# MYPFF Project — Context & Reference

> **Objetivo**: este arquivo serve como contexto para sessões futuras do Claude (ou qualquer dev) que precise trabalhar com o banco de dados MYPFF e os projetos de Fantasy Manager. Ele descreve tudo o que foi feito, as decisões tomadas, os padrões adotados e os caminhos dos arquivos.

---

## 1. Localização dos Arquivos

A pasta principal do projeto foi movida de `Downloads` para:

```
C:\Users\Erico Mello\Documents\Sleeper\
```

### Estrutura

```
Sleeper/
├── players_nfl.json                ← Sleeper API (11,564 jogadores, atualizado Mar/2026)
├── rosters.json                    ← Rosters da liga Sleeper
├── elenco_sleeper_liga_atual.csv
├── sleeper_exporter.py
├── sleeper_rosters_atual.py
├── sleeper_roster_id_map.py
├── sleeper_extract_draft_trades.py
├── rodar_sleeper.bat
│
└── PFF_Data/                       ← Subpasta com TODOS os dados PFF + banco
    ├── MYPFF_Complete.db           ← Banco SQLite principal (76 colunas, 53,662 rows)
    ├── MYPFF_Technical_Reference.docx  ← Documento técnico detalhado
    ├── CONTEXT_MYPFF_PROJECT.md    ← Este arquivo
    │
    ├── NFL_Rushing_2015_2016.csv   ← 46 CSVs do PFF
    ├── NFL_Rushing_2016_2017.csv
    ├── ... (11 NFL Rushing + 11 NFL Receiving + 11 NFL Passing + 12 NCAA Rushing + 12 NCAA Receiving + 12 NCAA Passing = 69 CSVs)
    ├── NCAA_Receiving_2025_2026.csv
    │
    ├── nflverse_players.csv        ← Mapeamento de IDs (24,375 jogadores, atualizado diariamente)
    ├── draft_picks_nflverse_v2.csv ← Draft completo 1980–2025 (12,670 picks, 36 colunas)
    ├── draft_picks_nflverse.csv    ← Versão antiga do draft (até 2024, 10 colunas) — backup
    ├── pff_pfr_map_v1.csv          ← Mapeamento legado PFF↔PFR (6,470 entries, 7 anos)
    ├── combine_nflverse.csv        ← NFL Combine data 2000–2026 (~5,500 rows, 18 colunas)
    ├── PFF_BigBoard_Draft2026.csv  ← PFF Big Board 2026 (450 prospects, 15 colunas, extraído 28/03/2026)
    ├── MDDB_ConsensusBigBoard_Draft2026.csv ← NFL Mock Draft Database Consensus (677 prospects, 7 colunas, extraído 28/03/2026)
    ├── PFN_MockDraft_Draft2026_2026-03-28.csv ← PFN 7-Round Mock Draft (99 picks, 10 colunas, Rounds 1-3)
    └── player_stats_nflverse.csv   ← Fantasy points semanais 1999–2024 (134,470 rows, 53 colunas, nflverse)
```

---

## 2. Convenção de Nomes dos CSVs

Todos os CSVs seguem o padrão:

```
{Liga}_{TipoStat}_{AnoInício}_{AnoFim}.csv
```

Exemplos:
- `NFL_Rushing_2025_2026.csv` → NFL, rushing stats, temporada 2025-2026
- `NCAA_Receiving_2014_2015.csv` → NCAA (FBS), receiving stats, temporada 2014-2015

### ATENÇÃO — Convenção de Anos do PFF

O PFF usa o **ano de início** como identificador da temporada. Quando o PFF mostra "2025", significa a temporada que **começou** em 2025 e **terminou** em 2026.

| PFF Year | Temporada Real | Nome no Banco (source) |
|----------|---------------|----------------------|
| 2025     | 2025-2026     | NFL_Rushing_2025_2026 |
| 2020     | 2020-2021     | NCAA_Receiving_2020_2021 |
| 2015     | 2015-2016     | NFL_Receiving_2015_2016 |

### URLs do PFF Premium Stats

```
NFL Rushing:    premium.pff.com/nfl/positions/{year}/REGPO/rushing?position=HB,FB,non-RB
NFL Receiving:  premium.pff.com/nfl/positions/{year}/REGPO/receiving?position=WR,TE,HB,FB
NCAA Rushing:   premium.pff.com/ncaa/positions/{year}/REGPO/rushing?division=fbs
NCAA Receiving: premium.pff.com/ncaa/positions/{year}/REGPO/receiving?division=fbs
```

Filtro utilizado: **REGPO** (Regular Season + Playoffs).

---

## 3. O Banco de Dados — MYPFF_Complete.db

### Resumo

| Propriedade | Valor |
|-------------|-------|
| Arquivo | `MYPFF_Complete.db` |
| Formato | SQLite 3 |
| Tabela | `mypff` (tabela única) |
| Rows | 60,743 |
| Colunas | 96 (90 PFF + 6 integração) — **98 desde 22/09/2026** (+`epa`, `positive_epa_percent`, anexadas ao fim; vazias na `mypff` desde a limpeza de 25/09, preenchidas na `mypff_weekly`; ver §21) |
| Jogadores únicos | ~17,000+ (NFL + NCAA) |
| Fontes (CSVs) | 69 (46 rushing/receiving + 23 passing) |
| Duplicatas | 0 |

### Colunas de Integração (adicionadas por nós)

Essas 6 colunas **não vêm do PFF**. Foram adicionadas usando dados do nflverse e Sleeper API:

| Coluna | O que é | Fonte | Cobertura NFL |
|--------|---------|-------|--------------|
| `gsis_id` | ID oficial da NFL (Game Statistics) | nflverse_players.csv | 100% |
| `sleeper_id` | ID do jogador no app Sleeper | Sleeper API + name match | 97.5% |
| `draft_year` | Ano do draft | nflverse_players.csv | 62.4% |
| `draft_round` | Round do draft (1-7) | nflverse_players.csv | 62.4% |
| `draft_pick` | Pick overall | nflverse_players.csv | 62.4% |
| `draft_team` | Time que draftou | nflverse_players.csv | 62.4% |

**Por que 62.4%?** Os 37.6% restantes são **UDFAs** (Undrafted Free Agents) — jogadores que nunca foram draftados. Seus campos de draft são `NULL`, o que é correto.

**Por que 97.5% sleeper_id?** Os 51 jogadores NFL sem sleeper_id têm nomes formatados diferente entre PFF e Sleeper (ex: "D.J. Moore" vs "DJ Moore"). Podem ser resolvidos com limpeza manual.

### Índices

| Índice | Coluna(s) | Uso |
|--------|-----------|-----|
| idx_source | source | Filtrar por temporada/tipo |
| idx_player_id | player_id | Buscar por ID PFF |
| idx_source_player | (source, player_id) | Identificador único por row |
| idx_gsis_id | gsis_id | Join com NFL/Sleeper |
| idx_sleeper_id | sleeper_id | Join com Sleeper API |
| idx_draft_year | draft_year | Filtrar por classe de draft |

### Tipo das Colunas

**TODAS as colunas são TEXT**. Para operações matemáticas, é necessário fazer cast:

```sql
-- Correto:
SELECT AVG(CAST(grades_offense AS REAL)) FROM mypff;

-- Errado (compara como texto):
SELECT AVG(grades_offense) FROM mypff;
```

### Observação sobre Colunas Rushing vs Receiving

Colunas específicas de rushing (ex: `attempts`, `yards`, `yco_attempt`) ficam NULL em rows de receiving, e vice-versa. O campo `source` indica qual tipo de dado é:
- `*_Rushing_*` → colunas de rushing preenchidas
- `*_Receiving_*` → colunas de receiving preenchidas

---

## 4. Cadeia de Mapeamento de IDs — PFF ↔ Sleeper

O Sleeper **não tem** campo `pff_id`. A ligação entre PFF e Sleeper é feita assim:

```
PFF player_id → nflverse pff_id → nflverse gsis_id → Sleeper gsis_id → Sleeper player_id
```

### Passo a passo

1. **PFF player_id → nflverse**: O arquivo `nflverse_players.csv` tem coluna `pff_id` que corresponde ao nosso `player_id`. Match de 100% para jogadores NFL.
2. **nflverse → gsis_id**: Mesma row no nflverse tem `gsis_id`. 100% de cobertura.
3. **gsis_id → Sleeper**: O JSON `players_nfl.json` tem `gsis_id` como campo. Match direto.
4. **Fallback por nome**: Se gsis_id não bater, normalizar nome (lowercase, sem pontuação, sem espaços) e comparar com `search_full_name` do Sleeper.

### Arquivos envolvidos

| Arquivo | O que contém | Onde pegar atualizado |
|---------|-------------|----------------------|
| `nflverse_players.csv` | 24,375 jogadores com pff_id, gsis_id, pfr_id, espn_id, draft info | github.com/nflverse/nflverse-data/releases/tag/players |
| `draft_picks_nflverse_v2.csv` | 12,670 picks (1980-2025), 36 colunas com stats | github.com/nflverse/nflverse-data/releases/tag/draft_picks |
| `players_nfl.json` | 11,564 jogadores Sleeper com IDs e metadata | api.sleeper.app/v1/players/nfl |
| `pff_pfr_map_v1.csv` | 6,470 mapeamentos PFF↔PFR (legado, 7 anos) | github.com/nflverse/nfldata |

### Exemplo de jogador mapeado

| Campo | Patrick Mahomes |
|-------|----------------|
| PFF player_id | 11765 |
| gsis_id | 00-0033873 |
| sleeper_id | 4046 |
| pfr_id | MahoPa00 |
| espn_id | 3139477 |
| draft | 2017 R1 P10 KC |

---

## 5. Draft Capital — Como foi Populado

### Estratégia em duas camadas

1. **Primária**: `nflverse_players.csv` → campos `draft_year`, `draft_round`, `draft_pick`, `draft_team` (matched por `pff_id`)
2. **Fallback**: `draft_picks_nflverse_v2.csv` → matched via `pfr_id` obtido do nflverse

### Draft 2025

O `draft_picks_nflverse_v2.csv` inclui todos os **257 picks do draft 2025**. Exemplos verificados:

| Jogador | PFF ID | Sleeper ID | Draft | Time |
|---------|--------|-----------|-------|------|
| Cam Ward | 133244 | 12522 | 2025 R1 P1 | TEN |
| Travis Hunter | 160766 | 12530 | 2025 R1 P2 | JAX |
| Abdul Carter | - | - | 2025 R1 P3 | NYG |

### UDFAs

Jogadores sem draft data (~37.6% do NFL) são UDFAs. Campos de draft ficam NULL. Isso é **correto e esperado**.

---

## 6. Sleeper API — Detalhes

### Endpoint

```
GET https://api.sleeper.app/v1/players/nfl
```

Retorna JSON com ~11,500 jogadores, keyed pelo `player_id` do Sleeper.

### Campos disponíveis no player object

O Sleeper **NÃO TEM** `pff_id`. IDs externos disponíveis:
- `gsis_id` ← **este é o bridge para PFF**
- `espn_id`
- `sportradar_id`
- `fantasy_data_id`
- `rotowire_id`
- `rotoworld_id`
- `yahoo_id`
- `stats_id`

### Atualização

O JSON deve ser atualizado periodicamente (rookies e UDFAs são adicionados ao longo da temporada). Última atualização: **Março 2026**.

---

## 7. Queries SQL Úteis

### Buscar jogador por nome
```sql
SELECT * FROM mypff WHERE player = 'Saquon Barkley' ORDER BY source;
```

### Buscar por Sleeper ID
```sql
SELECT * FROM mypff WHERE sleeper_id = '4866' ORDER BY source;
```

### Primeira rodada do draft 2025
```sql
SELECT DISTINCT player, player_id, sleeper_id, draft_pick, draft_team
FROM mypff WHERE draft_year = '2025' AND draft_round = '1'
ORDER BY CAST(draft_pick AS INTEGER);
```

### Evolução de um jogador ao longo das temporadas
```sql
SELECT source, yards, attempts, touchdowns, grades_run, elusive_rating
FROM mypff WHERE player_id = '11765' AND source LIKE 'NFL_Rushing_%'
ORDER BY source;
```

### Career stats para WRs com Sleeper ID
```sql
SELECT player, sleeper_id,
  SUM(CAST(receptions AS REAL)) as career_rec,
  SUM(CAST(rec_yards AS REAL)) as career_yards,
  SUM(CAST(touchdowns AS REAL)) as career_tds
FROM mypff WHERE position = 'WR'
  AND source LIKE 'NFL_Receiving_%'
  AND sleeper_id IS NOT NULL
GROUP BY player_id ORDER BY career_yards DESC;
```

### PFF grade médio por round de draft
```sql
SELECT draft_round,
  AVG(CAST(grades_offense AS REAL)) as avg_grade,
  COUNT(DISTINCT player_id) as num_players
FROM mypff WHERE source LIKE 'NFL_Rushing_2025_2026'
  AND draft_year IS NOT NULL
GROUP BY draft_round ORDER BY CAST(draft_round AS INTEGER);
```

---

## 8. Python — Join PFF + Sleeper

```python
import sqlite3, json

# Carregar dados do Sleeper
with open('players_nfl.json') as f:
    sleeper = json.load(f)

# Consultar banco PFF
conn = sqlite3.connect('PFF_Data/MYPFF_Complete.db')
c = conn.cursor()
c.execute("""
    SELECT player, sleeper_id, grades_offense, yards, touchdowns
    FROM mypff
    WHERE source = 'NFL_Rushing_2025_2026' AND sleeper_id IS NOT NULL
    ORDER BY CAST(grades_offense AS REAL) DESC
""")

for name, sid, grade, yds, tds in c.fetchall():
    team = sleeper.get(sid, {}).get('team', 'FA')
    print(f'{name} ({team}): Grade={grade}, Yds={yds}, TDs={tds}')
```

---

## 9. Problemas Resolvidos e Decisões

### Banco contaminado (MYPFF_Complete (1).db)
O banco antigo na pasta Downloads tinha ~14,764 duplicatas de receiving (mix de regular season e playoffs) e contaminação cross-season. **Solução**: reconstruir do zero com CSVs limpos.

### Colunas de draft com dados de fonte desconhecida
O banco antigo tinha 4 colunas de draft (`draft_year`, `draft_round`, `overall_pick`, `nfl_team_drafted_by`) com 4,273 rows que não vinham dos CSVs do PFF. **Solução**: descartar e popular com dados verificáveis do nflverse.

### Erro de I/O em SQLite no drive montado
SQLite dava `disk I/O error` ao escrever direto no drive Windows montado. **Solução**: trabalhar com cópia local no VM, depois copiar com `dd` de volta para o drive montado.

### Convenção de anos off-by-one
Inicialmente nomeamos os arquivos com erro (PFF 2025 → 2024_2025). **Correção**: PFF 2025 = temporada 2025_2026. Todos os 46 arquivos foram renomeados.

---

## 10. Manutenção — O Que Atualizar e Quando

| O quê | Quando | Como |
|-------|--------|------|
| CSVs do PFF | Anualmente (fim da temporada + playoffs) | Baixar de premium.pff.com, renomear, importar no banco |
| `players_nfl.json` | Periodicamente durante a temporada | `GET api.sleeper.app/v1/players/nfl` |
| `nflverse_players.csv` | Anualmente após o draft | github.com/nflverse/nflverse-data/releases/tag/players |
| `draft_picks_nflverse_v2.csv` | Anualmente após o draft | github.com/nflverse/nflverse-data/releases/tag/draft_picks |
| `combine_nflverse.csv` | Anualmente antes do draft (pós-combine, ~Mar) | Download direto: github.com/nflverse/nflverse-data/releases/download/combine/combine.csv |
| `PFF_BigBoard_Draft{year}.csv` | Anualmente antes do draft (~Mar-Abr) | Scraping via JS no Chrome em pff.com/draft/big-board?season={year} |
| `MDDB_ConsensusBigBoard_Draft{year}.csv` | Anualmente antes do draft (~Mar-Abr) | Scraping via JS no Chrome em nflmockdraftdatabase.com/big-boards/{year}/consensus-big-board-{year} |
| `PFN_MockDraft_Draft{year}_{date}.csv` | Semanalmente antes do draft (~Mar-Abr) | Scraping via JS no Chrome em profootballnetwork.com (buscar "7-Round {year} NFL Mock Draft") |
| `player_stats_nflverse.csv` | Semanalmente durante a temporada | Download direto: github.com/nflverse/nflverse-data/releases/download/player_stats/player_stats.csv |
| Banco MYPFF_Complete.db | Após qualquer atualização acima | Reimportar CSVs ou UPDATE incremental |

---

## 11. Lista Completa de Colunas (96)

### PFF (90 colunas — rushing/receiving + passing)
`accuracy_percent`, `aimed_passes`, `attempts`, `avg_depth_of_target`, `avg_time_to_throw`, `avoided_tackles`, `bats`, `big_time_throws`, `breakaway_attempts`, `breakaway_percent`, `breakaway_yards`, `btt_rate`, `caught_percent`, `completion_percent`, `completions`, `contested_catch_rate`, `contested_receptions`, `contested_targets`, `declined_penalties`, `def_gen_pressures`, `designed_yards`, `drop_rate`, `dropbacks`, `drops`, `elu_recv_mtf`, `elu_rush_mtf`, `elu_yco`, `elusive_rating`, `explosive`, `first_downs`, `franchise_id`, `fumbles`, `gap_attempts`, `grades_hands_drop`, `grades_hands_fumble`, `grades_offense`, `grades_offense_penalty`, `grades_pass`, `grades_pass_block`, `grades_pass_route`, `grades_run`, `grades_run_block`, `hit_as_threw`, `inline_rate`, `inline_snaps`, `interceptions`, `longest`, `pass_block_rate`, `pass_blocks`, `pass_plays`, `passing_snaps`, `penalties`, `player`, `player_game_count`, `player_id`, `position`, `pressure_to_sack_rate`, `qb_rating`, `rec_yards`, `receptions`, `route_rate`, `routes`, `run_plays`, `sack_percent`, `sacks`, `scramble_yards`, `scrambles`, `slot_rate`, `slot_snaps`, `source`, `spikes`, `targeted_qb_rating`, `targets`, `team_name`, `thrown_aways`, `total_touches`, `touchdowns`, `turnover_worthy_plays`, `twp_rate`, `wide_rate`, `wide_snaps`, `yards`, `yards_after_catch`, `yards_after_catch_per_reception`, `yards_after_contact`, `yards_per_reception`, `yco_attempt`, `ypa`, `yprr`, `zone_attempts`

### Integração (6 colunas)
`gsis_id`, `sleeper_id`, `draft_year`, `draft_round`, `draft_pick`, `draft_team`

---

## 12. NFL Combine Data — combine_nflverse.csv

### Resumo

| Propriedade | Valor |
|-------------|-------|
| Arquivo | `combine_nflverse.csv` |
| Fonte | nflverse (github.com/nflverse/nflverse-data/releases/tag/combine) |
| Período | 2000–2026 |
| Rows totais | ~5,500 |
| Prospects 2026 | 319 |
| Com dados de drills (2026) | 229 de 319 |
| Adicionado em | Março 2026 |

### Colunas (18)

| Coluna | Descrição |
|--------|-----------|
| `season` | Ano do combine (= ano do draft) |
| `draft_year` | Ano do draft (pode ficar vazio se não foi draftado) |
| `draft_team` | Time que draftou |
| `draft_round` | Round do draft |
| `draft_ovr` | Pick overall |
| `pfr_id` | Pro Football Reference ID |
| `cfb_id` | College Football ID |
| `player_name` | Nome do jogador |
| `pos` | Posição |
| `school` | Universidade |
| `ht` | Altura (formato "6-2") |
| `wt` | Peso (lbs) |
| `forty` | 40-yard dash (segundos) |
| `bench` | Bench press (repetições de 225 lbs) |
| `vertical` | Salto vertical (polegadas) |
| `broad_jump` | Broad jump (polegadas) |
| `cone` | 3-cone drill (segundos) |
| `shuttle` | 20-yard shuttle (segundos) |

### Uso principal

Este arquivo serve como a **lista de elegíveis para o Draft 2026** e fornece medidas físicas e resultados de drills. Pode ser cruzado com os CSVs do PFF via nome do jogador + escola.

Para filtrar apenas os elegíveis de 2026:
```sql
-- Se importado para SQLite:
SELECT * FROM combine WHERE season = '2026' ORDER BY pos, player_name;
```

### URL de download (atualização automática pelo nflverse)
```
https://github.com/nflverse/nflverse-data/releases/download/combine/combine.csv
```

---

## 13. PFF Big Board 2026 — Referência

O PFF publica um Big Board com ranking de prospects para o draft. Em Março 2026:

| Propriedade | Valor |
|-------------|-------|
| URL | pff.com/draft/big-board?season=2026 |
| Total de prospects | 450 (18 páginas × 25) |
| Atualização | 23/03/2026 |
| Dados por prospect | PFF Rank, Position Rank, Season Grade, PFF WAA, snaps, altura, peso, idade, classe |
| Acesso | Gratuito (sem necessidade de PFF+) |

### Top 10 Consensus (Big Board PFF + NFL Mock Draft Database, Mar 2026)

| PFF Rank | Jogador | Posição | Universidade |
|----------|---------|---------|-------------|
| 1 | Fernando Mendoza | QB | Indiana |
| 2 | Arvell Reese | LB | Ohio State |
| 3 | Jeremiyah Love | RB | Notre Dame |
| 4 | Sonny Styles | LB | Ohio State |
| 5 | Rueben Bain Jr. | EDGE | Miami (FL) |
| 6 | David Bailey | EDGE | Texas Tech |
| 7 | Francis Mauigoa | OT | Miami (FL) |
| 8 | Caleb Downs | S | Ohio State |

**Status**: Big Board **extraído** como CSV ✓ (28/03/2026). Arquivo: `PFF_BigBoard_Draft2026.csv`.

---

## 14. PFF Big Board 2026 — CSV Extraído

### Resumo

| Propriedade | Valor |
|-------------|-------|
| Arquivo | `PFF_BigBoard_Draft2026.csv` |
| Fonte | pff.com/draft/big-board?season=2026 (scraping via JavaScript no Chrome) |
| Total de prospects | 450 |
| Páginas extraídas | 18 (25 por página) |
| Colunas | 15 |
| Duplicatas | 0 (450 ranks únicos) |
| Atualização do Big Board | 23/03/2026 |
| Extraído em | 28/03/2026 |

### Colunas (15)

| Coluna | Descrição | Exemplo |
|--------|-----------|---------|
| `pff_rank` | Ranking geral do PFF | 1 |
| `player_name` | Nome do jogador | Fernando Mendoza |
| `position` | Posição | QB |
| `class_year` | Classe/ano escolar | RS Jr. |
| `school` | Universidade | Indiana |
| `age` | Idade | 22.4 |
| `height` | Altura | 6' 5" |
| `weight` | Peso (lbs) | 225 |
| `speed` | Velocidade (40yd, quando disponível) | — |
| `latest_season` | Temporada mais recente | 2025 |
| `snaps` | Snaps na temporada mais recente | 976 |
| `season_grade` | PFF Season Grade | 91.6 |
| `grade_rank` | Ranking da grade na posição | 8th / 316 QB |
| `pff_waa` | PFF Wins Above Average | 1.46 |
| `waa_rank` | Ranking do WAA na posição | #1 QB |

### Distribuição por posição

| Posição | Qty | Posição | Qty | Posição | Qty |
|---------|-----|---------|-----|---------|-----|
| WR | 64 | ED | 47 | CB | 44 |
| TE | 39 | T | 37 | HB | 35 |
| DI | 35 | LB | 33 | S | 31 |
| G | 30 | QB | 28 | C | 18 |
| K | 4 | P | 4 | FB | 1 |

### Método de extração

Scraping automatizado via JavaScript no Chrome (Claude in Chrome):
1. Navegou até `pff.com/draft/big-board?season=2026`
2. Função JS extraiu dados dos cards expandidos (`.g-card.g-card--border-gray`)
3. Grade extraído de `.kyber-grade-badge`, WAA de `.g-data`, rank de `.g-pill`
4. Paginação via click programático no botão `caret-right` com `xlink:href`
5. Loop de 18 páginas com 2.5s de espera entre cada
6. CSV gerado via `Blob` download no browser → salvo em PFF_Data

### Cruzamento com outros dados

O Big Board pode ser cruzado com:
- **combine_nflverse.csv**: via `player_name` + `school` para obter medidas de drills (40yd, bench, vertical, etc.)
- **CSVs de stats do PFF**: via `player_name` + `position` para obter stats detalhados de passing/rushing/receiving
- **Banco MYPFF_Complete.db**: jogadores NCAA podem ser encontrados pelo nome nas tabelas NCAA

**Nota**: O Big Board NÃO contém `player_id` do PFF. Para integração futura no banco, será necessário match por nome + escola.

---

## 15. Passing Stats — Concluído

### Resumo

| Propriedade | Valor |
|-------------|-------|
| CSVs NFL | 11 (2015–2025) |
| CSVs NCAA | 12 (2014–2025) |
| Total | **23 CSVs** |
| Colunas por CSV | 41 (player, player_id, position, team_name, grades_pass, grades_offense, attempts, completions, yards, touchdowns, interceptions, etc.) |
| Filtro | REGPO (Regular Season + Playoffs) |
| Baixado em | 27/03/2026 |

### URLs utilizadas
```
NFL:  premium.pff.com/nfl/positions/{year}/REGPO/passing?position=QB
NCAA: premium.pff.com/ncaa/positions/{year}/REGPO/passing?division=fbs
```

### Convenção de nomes (segue o padrão existente)
```
NFL_Passing_2015_2016.csv ... NFL_Passing_2025_2026.csv
NCAA_Passing_2014_2015.csv ... NCAA_Passing_2025_2026.csv
```

### Colunas do CSV de Passing (41)
`player`, `player_id`, `position`, `team_name`, `player_game_count`, `accuracy_percent`, `aimed_passes`, `attempts`, `avg_depth_of_target`, `avg_time_to_throw`, `bats`, `big_time_throws`, `btt_rate`, `completion_percent`, `completions`, `declined_penalties`, `def_gen_pressures`, `drop_rate`, `dropbacks`, `drops`, `first_downs`, `franchise_id`, `grades_hands_fumble`, `grades_offense`, `grades_pass`, `grades_run`, `hit_as_threw`, `interceptions`, `passing_snaps`, `penalties`, `pressure_to_sack_rate`, `qb_rating`, `sack_percent`, `sacks`, `scrambles`, `spikes`, `thrown_aways`, `touchdowns`, `turnover_worthy_plays`, `twp_rate`, `yards`, `ypa`

**Status**: Download **concluído** ✓ e **integrado no banco** ✓ (27/03/2026).

### Integração
- 20 novas colunas adicionadas ao banco (total agora: 96)
- 7,081 rows inseridas (1,136 NFL + 5,945 NCAA)
- NFL gsis_id: 100% cobertura
- NFL sleeper_id: 69% cobertura (QBs backups/aposentados ausentes do Sleeper)
- NCAA: sem integração de IDs (esperado, jogadores college)

---

## 16. Weekly Fantasy Points — player_stats_nflverse.csv

### Resumo

| Propriedade | Valor |
|-------------|-------|
| Arquivo | `player_stats_nflverse.csv` |
| Fonte | nflverse (github.com/nflverse/nflverse-data/releases/tag/player_stats) |
| Período | 1999–2024 (26 temporadas) |
| Rows totais | 134,470 |
| Colunas | 53 |
| Granularidade | **Semanal** (por jogador, por semana) |
| Inclui playoffs | Sim (season_type = REG e POST) |
| Fantasy points | Standard + PPR pré-calculados |
| Jogadores únicos por ano | ~590–660 |
| Tamanho do arquivo | 31.9 MB |
| Baixado em | 28/03/2026 |

### Colunas (53)

**Identificação**: `player_id` (= gsis_id), `player_name`, `player_display_name`, `position`, `position_group`, `headshot_url`, `recent_team`, `season`, `week`, `season_type`, `opponent_team`

**Passing**: `completions`, `attempts`, `passing_yards`, `passing_tds`, `interceptions`, `sacks`, `sack_yards`, `sack_fumbles`, `sack_fumbles_lost`, `passing_air_yards`, `passing_yards_after_catch`, `passing_first_downs`, `passing_epa`, `passing_2pt_conversions`, `pacr`, `dakota`

**Rushing**: `carries`, `rushing_yards`, `rushing_tds`, `rushing_fumbles`, `rushing_fumbles_lost`, `rushing_first_downs`, `rushing_epa`, `rushing_2pt_conversions`

**Receiving**: `receptions`, `targets`, `receiving_yards`, `receiving_tds`, `receiving_fumbles`, `receiving_fumbles_lost`, `receiving_air_yards`, `receiving_yards_after_catch`, `receiving_first_downs`, `receiving_epa`, `receiving_2pt_conversions`, `racr`, `target_share`, `air_yards_share`, `wopr`

**Special Teams + Fantasy**: `special_teams_tds`, `fantasy_points`, `fantasy_points_ppr`

### Integração com MYPFF

O campo `player_id` do nflverse é o **gsis_id** (formato `00-0033873`), que é exatamente a coluna `gsis_id` do banco MYPFF.

| Temporada | nflverse | MYPFF | Matched | Match% |
|-----------|----------|-------|---------|--------|
| 2015 | 594 | 575 | 574 | 96.6% |
| 2016 | 593 | 586 | 586 | 98.8% |
| 2017 | 587 | 581 | 580 | 98.8% |
| 2018 | 614 | 602 | 602 | 98.0% |
| 2019 | 617 | 604 | 604 | 97.9% |
| 2020 | 636 | 630 | 629 | 98.9% |
| 2021 | 660 | 650 | 650 | 98.5% |
| 2022 | 621 | 614 | 613 | 98.7% |
| 2023 | 591 | 583 | 582 | 98.5% |
| 2024 | 612 | 601 | 601 | 98.2% |

### Exemplo de join com MYPFF

```sql
-- Supondo que player_stats foi importado como tabela 'weekly_fantasy'
-- Top 10 jogadores com melhor grade PFF E mais fantasy points PPR (2024)
SELECT w.player_display_name, w.position, w.recent_team,
       SUM(w.fantasy_points_ppr) as total_ppr,
       AVG(CAST(m.grades_offense AS REAL)) as avg_pff_grade
FROM weekly_fantasy w
JOIN mypff m ON w.player_id = m.gsis_id
WHERE w.season = '2024' AND w.season_type = 'REG'
  AND m.source LIKE 'NFL_%2024_2025'
  AND m.source LIKE '%Rushing%' OR m.source LIKE '%Receiving%'
GROUP BY w.player_id
ORDER BY total_ppr DESC
LIMIT 10;
```

### Métricas avançadas incluídas

| Métrica | Descrição |
|---------|-----------|
| `passing_epa` | Expected Points Added em passes |
| `rushing_epa` | Expected Points Added em corridas |
| `receiving_epa` | Expected Points Added em recepções |
| `dakota` | DAKOTA (Adjusted EPA + CPOE composite) |
| `pacr` | Passing Air Conversion Ratio |
| `racr` | Receiving Air Conversion Ratio |
| `wopr` | Weighted Opportunity Rating |
| `target_share` | % de targets do time |
| `air_yards_share` | % de air yards do time |

### Status e próximos passos

- **Download**: concluído ✓ (28/03/2026)
- **NÃO integrado ao banco MYPFF** (é um dataset independente por enquanto)
- Pode ser importado como tabela separada `weekly_fantasy` no SQLite se desejado
- Também disponível por ano individual: `player_stats_{year}.csv`
- Atualizado automaticamente pelo nflverse ao longo da temporada

### URL de download
```
https://github.com/nflverse/nflverse-data/releases/download/player_stats/player_stats.csv
```

---

## 17. NFL Mock Draft Database — Consensus Big Board 2026

### Avaliação da Fonte

O NFL Mock Draft Database (nflmockdraftdatabase.com) agrega dados de mock drafts de múltiplas fontes. O produto principal é o **Consensus Big Board**, que combina 124 big boards + 967 first-round mocks + 1,118 team-specific mocks em um ranking agregado.

### Resumo do CSV Extraído

| Propriedade | Valor |
|-------------|-------|
| Arquivo | `MDDB_ConsensusBigBoard_Draft2026.csv` |
| Fonte | nflmockdraftdatabase.com/big-boards/2026/consensus-big-board-2026 |
| Total de prospects | 677 |
| Colunas | 7 |
| Com consensus mock pick (R1) | 32 |
| Times representados nos mocks | 27 |
| Escolas únicas | 139 |
| Extraído em | 28/03/2026 |

### Colunas (7)

| Coluna | Descrição | Exemplo |
|--------|-----------|---------|
| `consensus_rank` | Ranking agregado de 124 big boards | 1 |
| `player_name` | Nome do jogador | Fernando Mendoza |
| `position` | Posição | QB |
| `school` | Universidade | Indiana |
| `consensus_mock_pick` | Pick médio no mock (Round 1 apenas) | 1 |
| `consensus_mock_team` | Time mais frequente no mock | LV |
| `mddb_slug` | URL slug do jogador no MDDB | /players/2026/fernando-mendoza |

### Distribuição por posição

| Posição | Qty | Posição | Qty | Posição | Qty |
|---------|-----|---------|-----|---------|-----|
| WR | 91 | EDGE | 75 | IOL | 75 |
| CB | 74 | DL | 61 | OT | 58 |
| LB | 55 | RB | 48 | S | 46 |
| TE | 41 | QB | 35 | P | 6 |
| K | 6 | LS | 6 | | |

### Comparação com PFF Big Board

| Métrica | MDDB | PFF |
|---------|------|-----|
| Total de prospects | 677 | 450 |
| Jogadores em ambos | 400 | 400 |
| Exclusivos | 277 | 50 |
| Posições usadas | IOL, OT, EDGE, DL | G, C, T, ED, DI |
| Posições fantasy comuns | QB, RB, WR, TE, S, LB, CB, K, P | QB, HB, WR, TE, S, LB, CB, K, P, FB |

**Mapeamento de posições MDDB → PFF**: IOL = G + C, OT = T, EDGE = ED, DL = DI, RB = HB. As demais são iguais.

**Concordância de ranking**: Os top 10 são quase idênticos entre MDDB e PFF (diff máxima de ±3 posições). A divergência cresce no top 50, com diferenças de até ±37 posições (ex: Anthony Hill Jr. — MDDB #49, PFF #86).

### Limitações Importantes

1. **Apenas Round 1 nos mocks**: O MDDB só exibe a pick de Round 1 para cada prospect, mesmo quando a fonte original tem 7 rounds. Dos 677 prospects, apenas 32 têm consensus mock pick.
2. **Sem player_id**: Não há identificador numérico do PFF. O `mddb_slug` serve como ID interno do MDDB.
3. **Sem stats de performance**: Apenas ranking e posição — sem grades, snaps, ou métricas.
4. **Download CSV pago**: O botão "Download CSV" do site requer assinatura Mock+ ($5.99/mês). Extração feita via scraping JS.

### Método de extração

Scraping automatizado via JavaScript no Chrome (Claude in Chrome):
1. Navegou até a página do Consensus Big Board 2026
2. DOM responsivo com 4 variantes de layout — parsing focou na variante `d-sm-none` (mobile) que contém todos os dados
3. Dois padrões de LI identificados: 423 com classe `d-none d-xl-block` (rank em `children[0]`) e 254 sem (rank em `children[1]`)
4. Schools extraídas separadamente em 7 chunks via JS tool (contorno de limite de 4KB no output)
5. CSV final reconstruído via Python: merge de positions corrigidas + schools dos chunks

### Valor para o Pipeline

O MDDB Consensus Big Board é útil como **fonte de ranking agregado** (consenso de 124+ analistas) mas **NÃO serve como fonte de mock drafts multi-round**. Para dados de multi-round mocks, será necessário avaliar outras fontes.

### Cruzamento com outros dados

- **PFF Big Board**: via `player_name` + `school` (400 de 450 prospects PFF encontrados no MDDB)
- **combine_nflverse.csv**: via `player_name` + `school`
- **Banco MYPFF_Complete.db**: jogadores NCAA via nome

---

## 18. Pro Football Network — Mock Draft Multi-Round

### Avaliação da Fonte

O Pro Football Network (PFN) publica mock drafts de 7 rounds regularmente (autor principal: Ian Cummings). Apesar do título "7-Round", o conteúdo do artigo tipicamente cobre ~3 rounds (~100 picks), o que já é significativamente mais do que o Round 1 disponível no MDDB.

A estrutura HTML é consistente e parseável: H2 = `pick#) Team`, H3 = `Player Name, School | Position`.

### Avaliação de Outras Fontes Multi-Round

| Fonte | Rounds Disponíveis | Estrutura | Observação |
|-------|-------------------|-----------|------------|
| PFN (profootballnetwork.com) | ~3 rounds (100 picks) | Artigo, HTML limpo | Mock geral completo, semanal |
| The Draft Network (thedraftnetwork.com) | 1-2 rounds (geral) + 7 rounds (por time) | Artigo, HTML limpo | Mocks team-specific fragmentados |
| NFL Mock Draft Database | Round 1 apenas | Lista estruturada | Só exibe R1 mesmo quando fonte tem 7 |

### Resumo do CSV Extraído

| Propriedade | Valor |
|-------------|-------|
| Arquivo | `PFN_MockDraft_Draft2026_2026-03-28.csv` |
| Fonte | profootballnetwork.com |
| Autor | Ian Cummings |
| Data do mock | 28/03/2026 |
| Total de picks | 99 |
| Colunas | 10 |
| Round 1 | 32 picks |
| Round 2 | 32 picks |
| Round 3 | 35 picks |
| Jogadores únicos | 99 (zero duplicatas) |

### Colunas (10)

| Coluna | Descrição | Exemplo |
|--------|-----------|---------|
| `pick` | Número overall da pick | 1 |
| `round` | Rodada (1-3) | 1 |
| `team` | Time que seleciona | Las Vegas Raiders |
| `player_name` | Nome do jogador | Fernando Mendoza |
| `position` | Posição | QB |
| `school` | Universidade | Indiana |
| `mock_source` | Fonte do mock | Pro Football Network |
| `mock_author` | Autor | Ian Cummings |
| `mock_date` | Data de publicação | 2026-03-28 |
| `mock_url` | URL do artigo original | (link completo) |

### Convenção de Nomes

```
PFN_MockDraft_Draft{year}_{date}.csv
```

Exemplo: `PFN_MockDraft_Draft2026_2026-03-28.csv` = Mock para o Draft 2026, publicado em 28/03/2026.

Cada arquivo representa um snapshot semanal. Arquivos de semanas diferentes coexistem na pasta, permitindo rastrear a evolução das projeções ao longo do tempo.

### Mapeamento de Posições PFN → PFF

| PFN | PFF | Nota |
|-----|-----|------|
| OLB | LB | PFN diferencia OLB/ILB, PFF usa LB genérico |
| OG | G | Guard |
| OT | T | Tackle |
| OL | G/T/C | PFN usa OL genérico, PFF especifica G/T/C |
| DT | DI | Defensive Tackle |
| DL | DI | Defensive Line genérico |
| DB | CB/S | PFN usa DB genérico para versatile players |
| IOL | G/C | Interior Offensive Line |
| EDGE | ED | Edge Rusher |

### Método de extração

Scraping via JavaScript no Chrome (Claude in Chrome):
1. Buscar no PFN por "7-Round {year} NFL Mock Draft" para achar o artigo mais recente
2. Função JS extrai H2 (pick# + team) e H3 (player, school | position)
3. CSV gerado com metadados de fonte (autor, data, URL)
4. Salvo em PFF_Data com nome incluindo a data do mock

### Cruzamento com outros dados

- **PFF Big Board**: via `player_name` + `school` para obter grades e WAA
- **MDDB Consensus Big Board**: via `player_name` para comparar ranking consensus vs projeção de pick
- **combine_nflverse.csv**: via `player_name` + `school` para medidas físicas
- **Banco MYPFF_Complete.db**: jogadores NCAA via nome para stats de produção

---

## 19. Fontes Externas — Resumo de URLs

| Recurso | URL |
|---------|-----|
| PFF Premium Stats | premium.pff.com/nfl/positions/{year}/REGPO/{stat_type} |
| PFF Big Board 2026 | pff.com/draft/big-board?season=2026 |
| NFL Mock Draft Database (Consensus) | nflmockdraftdatabase.com/big-boards/2026/consensus-big-board-2026 |
| nflverse Combine CSV | github.com/nflverse/nflverse-data/releases/download/combine/combine.csv |
| nflverse Players CSV | github.com/nflverse/nflverse-data/releases/tag/players |
| nflverse Draft Picks | github.com/nflverse/nflverse-data/releases/tag/draft_picks |
| nflverse Weekly Fantasy Stats | github.com/nflverse/nflverse-data/releases/download/player_stats/player_stats.csv |
| Pro Football Network Mock Drafts | profootballnetwork.com/?s=7-round+{year}+nfl+mock+draft |
| Sleeper API (players) | api.sleeper.app/v1/players/nfl |

---

## 20. Deploy do MYPFF atualizado para o Render — Runbook

### Por que é um passo manual separado

O pipeline `deploy_scores.sh` do Optimizer atualiza apenas o `optimizer.db` em produção (via seed commit + `/admin/reseed`). O `MYPFF_Complete.db` **não** é coberto: o `init_data.py` do Optimizer (`fantasy_optimizer/init_data.py`, linhas 36-38) só copia bancos para `/data/` no Render se o destino ainda não existir — em redeploys, preserva o `/data/` existente. Logo, atualizar o MYPFF em produção exige substituição manual via Render Shell.

Aplicável **após cada ciclo anual MYPFF-UN** (reabastecimento pós-draft — ver §5 estratégia de população e §10 tabela de manutenção).

### Pré-requisitos

- `pff_data/MYPFF_Complete.db` local já atualizado (output da fase F2 do MYPFF-UN).
- Acesso ao dashboard do Render: abas **Shell** (terminal web) e **Disk** (snapshots de rollback).

### Etapa 1 — Atualizar o seed do MYPFF no repo

⭐ **O seed leva SÓ a tabela `mypff` (decisão do owner, 01/10/2026, publicação B37/B39).** O MYPFF local também
tem `mypff_weekly` (parcial da temporada em curso), `mypff_identity` e `mypff_identity_meta`. O Optimizer de
produção não lê nenhuma delas, e o parcial fica fora de produção (P19). ⛔ **Não usar `cp` do arquivo inteiro.**

```bash
cd /c/Users/Erico\ Mello/Fantasy/fantasy_optimizer
python - <<'EOF'
import os, sqlite3
out = "MYPFF_Complete.db.new"
if os.path.exists(out): os.remove(out)
src = sqlite3.connect("file:../pff_data/MYPFF_Complete.db?mode=ro", uri=True)
src.execute("VACUUM INTO ?", (out,)); src.close()
d = sqlite3.connect(out)
for (t,) in d.execute("SELECT name FROM sqlite_master WHERE type='table' AND name <> 'mypff'").fetchall():
    d.execute(f"DROP TABLE {t}")
d.commit(); d.execute("VACUUM"); print(d.execute("PRAGMA integrity_check").fetchone()[0], d.execute("SELECT COUNT(*) FROM mypff").fetchone()[0]); d.close()
EOF
mv MYPFF_Complete.db.new MYPFF_Complete.db
sha256sum MYPFF_Complete.db      # anotar: é o sha que a etapa 3 confere no Shell
git add MYPFF_Complete.db
git commit -m "MYPFF seed refresh — class YYYY (MYPFF-UN)"
git push origin main
```

Conferir antes do commit que a `mypff` do seed é idêntica à local, linha a linha (na publicação de 01/10: 60.743
linhas, 98 colunas). Commit binário (~22-24 MB): o push demora um pouco. Normal.

### Etapa 2 — Aguardar o redeploy automático do Render

O push dispara redeploy automático. Acompanhar na aba **Events**. Tipicamente 2-4 min até ficar **Live**. Só prosseguir depois disso (o seed novo precisa estar no container).

### Etapa 3 — Substituir o arquivo via Render Shell

Abrir a aba **Shell** no dashboard. Executar em ordem (um bloco por vez):

```bash
# 3a — pré-check da env var MYPFF_DB (confirma o alvo correto)
echo "MYPFF_DB=$MYPFF_DB"
ls -la "$MYPFF_DB"
# Esperado: /data/MYPFF_Complete.db, ~23.8 MB

# 3b — confirmar os dois MYPFFs no container
ls -la /opt/render/project/src/MYPFF_Complete.db   # seed novo (recém-deployado)
ls -la /data/MYPFF_Complete.db                     # produtivo atual (a substituir)
# Se o caminho do seed variar, localizar com: find / -name "MYPFF_Complete.db" 2>/dev/null

# 3c — cópia atômica fase 1: copia o seed novo para arquivo temporário
cp /opt/render/project/src/MYPFF_Complete.db /data/MYPFF_Complete.db.new

# 3d — backup não-sobrescritivo do atual (nome com data do ciclo) + rename atômico do novo
# ⛔ Se imprimir PARAR, NÃO rodar o mv seguinte: ele substituiria o atual sem backup (defeito da
#    versão anterior deste bloco, que só avisava e seguia — corrigido em 01/10/2026).
[ ! -f /data/MYPFF_Complete_pre_<ciclo>_<data>.db ] && mv /data/MYPFF_Complete.db /data/MYPFF_Complete_pre_<ciclo>_<data>.db || echo "backup ja existe - PARAR"
mv /data/MYPFF_Complete.db.new /data/MYPFF_Complete.db
# 3e' — conferir: sha256sum /data/MYPFF_Complete.db = sha da etapa 1; contagem e integrity_check via python3

# 3e — confirmar
ls -la /data/MYPFF_Complete.db /data/MYPFF_Complete_pre_<ciclo>_<data>.db
```

**Por que `cp .new` + `mv` em vez de `mv` direto:** o `mv` no mesmo filesystem é um rename POSIX atômico. Copiar primeiro para `.new` e depois renomear evita uma janela em que `/data/MYPFF_Complete.db` não existe — janela que quebraria qualquer request `/player/<name>` em andamento (a página consome o MYPFF via `optimizer/loader.py:get_pff_stats`). Conexões SQLite já abertas continuam servidas pelo descriptor antigo até fecharem; conexões novas pegam o arquivo novo de imediato.

### Etapa 4 — Validar end-to-end

Abrir no navegador 2 rookies ofensivos da classe — um de tier baixo (provável free-agent path) e um de tier alto (provável roster path), para cobrir as duas branches de renderização de `player_detail.py`:

```
https://<render-url>/player/<rookie-tier-LOW>     # ex: Caleb Douglas (2026)
https://<render-url>/player/<rookie-tier-ELITE>   # ex: Carnell Tate (2026)
```

Esperar ver **"Draft YYYY"** no card de trajetória. Se aparecer nos dois, o ciclo fechou.

### Rollback

- **Backup imediato:** `mv /data/MYPFF_Complete_pre_<ciclo>_<data>.db /data/MYPFF_Complete.db` (reverte ao estado anterior em segundos).
- **Snapshot do disco:** aba **Disk** → escolher snapshot diário anterior → **Restore** (reverte o disco inteiro em ~minutos).

### Cadência

Anual, logo após a fase F2 de cada ciclo `MYPFF-UN`. Substituir `YYYY`/`UN` pelos valores do ciclo corrente (ex: `MYPFF-U2`, `2027`).

> Item de automação candidato (não implementado): rota `/admin/upload-mypff` (POST multipart) ou Git LFS, para evitar o commit binário de ~24 MB a cada ciclo. Enquanto não existir, este runbook é o procedimento oficial.

---

## 21. Dados in-season 2026 — carga de 22/09 e fluxo semanal da `mypff_weekly` (registrado em 25/09/2026, PRED-P19-F1)

> **Atualização 25/09/2026 (PRED-P19-F2a):** a `mypff` **voltou ao estado pré-carga** — as 3.922
> linhas `*_2026_2027` foram removidas (60.743 linhas, fonte mais nova `2025_2026`; as colunas `epa` e
> `positive_epa_percent` ficaram). **O dado de 2026 vive só na `mypff_weekly`** (7.805 linhas, intacta),
> que ainda não tem consumidor. O congelamento operacional foi retirado. Backup pré-limpeza em
> `C:\Users\Erico Mello\fantasy_backups\MYPFF_Complete_pre_P19F2a_2026-09-25.db`, retido até existir
> um `MYPFF_Complete.pre_weekly_*.db` validado. A tabela de estado abaixo registra 22–25/09.

### O que existe no banco

**Carga de 22/09/2026 (15:17–15:36, owner, intencional, via Cowork).** Objetivo: ter os dados de 2026
disponíveis. Foi feita com três scripts hoje em `pff_data/scripts/` (versões de 22/09):

| Script | O que faz |
|---|---|
| `grab.sh` | captura o download do PFF no navegador do Cowork (o PFF salva todo export como `<tipo>_summary.csv`, **mesmo nome para NFL/NCAA e para toda semana**) |
| `load_season.py` | por source `*_2026_2027`: `DELETE` + `INSERT` na tabela **`mypff`**, `ALTER TABLE ADD COLUMN` para colunas novas, enriquecimento `gsis_id`/`sleeper_id`/draft via `nflverse_players.csv` + `players_nfl.json` |
| `load_weekly.py` | cria a tabela **`mypff_weekly`** (colunas da `mypff` + `season` + `week`), `DELETE` por `(source, week)` + `INSERT`, IDs copiados da `mypff` |

Estado medido em 25/09 (read-only):

| | Valor |
|---|---|
| `mypff` | 64.665 rows (60.743 + **3.922** `*_2026_2027`), **98 colunas** |
| NFL `*_2026_2027` | Rushing 161, Receiving 303, Passing 45 — **1 a 2 jogos** |
| NCAA `*_2026_2027` | Rushing 1.226, Receiving 1.828, Passing 359 — **1 a 4 jogos** |
| `mypff_weekly` | 7.805 rows, 100 colunas — NFL W1–W2, NCAA W0–W3 |
| Reprodutibilidade | os 6 CSVs de temporada em `pff_data/` e os 18 semanais em `pff_data/weekly/` batem linha a linha com o banco |

⛔ **(Até 25/09) As linhas `*_2026_2027` da `mypff` eram lidas como TEMPORADA COMPLETA** por ~100 pontos de consumo
no Predictor e no Optimizer (inventário no P19 do `optimizer_improvements.md`). Nenhum consumidor
conhece a `mypff_weekly`. Até a F2 do P19 vale o **congelamento operacional** registrado nos CLAUDE.md
dos dois projetos: nada de recálculo, publicação ou deploy de scores contra o MYPFF vivo, nem cópia do
MYPFF para o Render.

**Idempotência (leitura de código, 25/09):**
- `load_season.py` é idempotente por source (`DELETE` + `INSERT`; `ADD COLUMN` só para coluna ausente;
  enriquecimento só em linha sem `gsis_id`). Limite: CSV ausente ou vazio é **pulado**, e a source fica
  com o retrato anterior — uma execução parcial deixa sources de datas diferentes, sem aviso.
- `load_weekly.py` é idempotente por `(source, week)`. Limite: os IDs vêm da `mypff`, então jogador que
  aparecer pela primeira vez depois do retrato de 22/09 entra na `mypff_weekly` **sem** `gsis_id`/`sleeper_id`.
- Os dois operam sobre `/tmp/mypff_local.db` com caminhos `/sessions/*/mnt/...` do Cowork: **não rodam
  localmente sem edição**, e a troca do arquivo no drive não está nos scripts.

### Tarefa semanal do Cowork — desenho vigente desde 25/09/2026

Roda às **quartas, 07:00–~07:45 (Brasília)**, e troca o arquivo `MYPFF_Complete.db`. Uma mudança de
hash do MYPFF com mtime nessa janela é **escrita esperada da tarefa**, não violação de invariante.

| Proteção | Risco que a motivou |
|---|---|
| **Não grava mais na tabela `mypff`** — as linhas `*_2026_2027` ficam congeladas no retrato de 22/09 | cada carga semanal na `mypff` avançaria o `player_game_count`; ao passar de 4 jogos, a NFL parcial atravessa a única guarda existente (histórico do TBO, `player_game_count >= 4`) e passa a contaminar trajetória, pico e flags dos veteranos |
| **Grava só a `mypff_weekly`** | é a única tabela sem consumidor: dado novo entra sem mudar nenhum score até o P19 decidir o armazenamento |
| **Pasta por execução** | o PFF salva todo export com o mesmo nome (`<tipo>_summary.csv`); sem isolamento, uma semana ou liga sobrescreve a outra (há `passing_summary.csv` e `rushing_summary.csv` de 0 byte na raiz de `pff_data/` como resíduo de 22/09) |
| **Backup antes da troca** | o arquivo é substituído inteiro; sem backup não há rollback (mesma lógica do `MYPFF_Complete_pre_U1.db`) |
| **Troca por renomeação, fallback `dd`** | escrever SQLite direto no drive montado dá `disk I/O error` (§9); o rename evita a janela com arquivo meio escrito; o `dd` cobre o caso em que o rename falha entre montagens |
| **Log em `weekly/_log_<data>.txt`** | a carga de 22/09 não deixou registro em nenhum documento nem commit; a F1 do P19 teve de reconstruí-la por mtime |

A partir de 25/09, **qualquer mudança de forma** no MYPFF (tabela nova, source nova, schema, carga de
natureza nova) passa antes por um item `MYPFF-*` — regra do `DEV_METHODOLOGY.md`. A execução semanal
acima é rotina do fluxo registrado aqui e não precisa de item.

### ⭐ Estado vigente: MODO MANUAL até a migração (01/10/2026 — item C24)

O Cowork anunciou a **descontinuação das tarefas agendadas neste computador** (sem tarefas novas a partir de
06/10/2026, sem manutenção nas existentes, que "ainda funcionam"). Em 30/09 a tarefa não rodou: estava como
execução única, e o wrapper abortou por limite (exit 2, correto). Em 01/10 a carga foi disparada à mão e rodou
limpa; o STEP 0 restaurou o cron (quartas 07:00), **mas não se conta com ele**.

**Toda quarta, nesta ordem, até o C24 fechar:**
1. Disparar **à mão** a tarefa semanal do Cowork e esperar o `weekly/_log_<data>.txt` com o banco atualizado.
2. Disparar o rebuild da identidade:
   `Start-ScheduledTask -TaskPath '\Fantasy\' -TaskName 'MYPFF identity rebuild (W1-F3)'`.
   O wrapper encontra o log do dia e roda de imediato. Conferir `DESFECHO: REBUILD RODOU` em
   `identity/_windows_rebuild_<data>.log`.

Se o cron restaurado rodar sozinho às 07:00 e o wrapper das 10:00 também, não há nada a fazer. Se não rodar, os
dois passos acima são o desfecho. O roteiro de validação e as regras do wrapper descritos abaixo continuam válidos.

**Divergência semanal × SEASON não é, por si, falha de export.** Na carga de 01/10, o Deebo Samuel (NFL Receiving)
somou 79 jardas nas semanas contra 159 no SEASON. Na W03 ele não recebeu passe: foi uma lateral depois da recepção
de outro jogador, com as jardas contadas como corrida. É **classificação de jogada** diferente entre os dois cortes.
A nota do `_log_2026-10-01.txt` (linha 58) que chama isso de "falha do export PFF" está errada; o log não foi
reescrito. Para o consumo in-season (P19): tolerância a divergências pequenas, com registro, e um corte canônico
(provavelmente SEASON).

### Mapa de identidade `mypff_identity` (MYPFF-W1-F2, 25/09/2026)

A identidade para consumo semanal **não** vem mais das colunas de ID da `mypff`/`mypff_weekly` (cópia
por linha, que congelou erros de método). Vem da tabela **`mypff_identity`**, reconstruída do zero a
cada execução por `predictor/scripts/mypff_identity.py` (infraestrutura do banco compartilhado
hospedada no Predictor; revisitar o local antes de duplicar).

| | |
|---|---|
| Chave | `pff_id` (o `player_id` do PFF) |
| Universo | `mypff_weekly` + quem tem linha NFL na `mypff` + quem tem `draft_year` na `mypff` (4.819 em 25/09; 4.936 em 01/10, `content_sha256` `61fca09b4c78fd4a`, `conflict` = 1: Brandon Williams) |
| Escopo | `nfl` · `drafted_no_nfl` (só `exact_ffids`) · `ncaa_only` (nunca recebe ID por nome) |
| Métodos | `exact_ffids` · `ambiguous_ffids` · `name_normalized` (só `nfl`, só skill, candidato único) · `unmatched` · `ncaa_undrafted` |
| Marcas | `conflict` (legado da `mypff` difere) · `shared_sleeper` (mesmo `sleeper_id` para mais de um `pff_id`) |
| Fonte primária | `ff_playerids` (DynastyProcess / ecossistema nflverse), `pff_id → sleeper_id` na mesma linha |
| Fallback | `players/nfl` do Sleeper, nome normalizado (caixa, acento, pontuação, sufixo Jr./Sr./II/III/IV/V) |
| Downloads | `pff_data/identity/ff_playerids_<data>.csv` e `sleeper_players_<data>.json` |
| Auditoria | `mypff_identity_meta`: uma linha por reconstrução (carimbo, `content_sha256`, fontes, contagens) |

Invocação (a partir de `predictor/`): `python -m scripts.mypff_identity --rebuild` (ou `--dry-run`,
`--check`; `--db` para outro banco; `--ff-file`/`--sleeper-file` para fixar fontes).

**`--check` depois de um rebuild:** ele recomputa com os arquivos de fonte registrados na última linha
da `mypff_identity_meta`. Se o rebuild foi feito em cópia (`--db /tmp/...`) ou no sandbox do Cowork, o
meta aponta para caminhos que não existem no Windows, e o `--check` falha ao abrir. Nesse caso, passar
`--ff-file`/`--sleeper-file` com os arquivos de `pff_data/identity/`. Com o rebuild semanal no Windows,
o meta guarda caminhos do Windows e o problema não aparece. Decisão MYPFF-W1-F3: só documentar, sem
ajuste no script.

### Rebuild semanal da identidade — no Windows, não no Cowork (MYPFF-W1-F3, 25/09/2026)

**Por quê:** a rede do sandbox da tarefa do Cowork bloqueia `raw.githubusercontent.com` e
`api.sleeper.app` (403 no proxy, teste de 25/09). O navegador interno libera o GitHub, mas recusa
`api.sleeper.app` por restrição de segurança, sem contorno. A lista de domínios só é configurável em
planos Team/Enterprise. No Windows o script funciona. **A tarefa do Cowork não reconstrói a identidade.**
O passo 5b que ela tinha é removido pelo owner antes de 30/09.

**Como:** o Agendador de Tarefas do Windows roda, **quartas às 10:00**, o wrapper
`predictor/scripts/mypff_identity_weekly.py`. Ele espera a tarefa do Cowork do dia terminar e só então
chama `python -m scripts.mypff_identity --rebuild` contra o MYPFF real.

| Regra | Detalhe |
|---|---|
| Pronto para rodar | existe `weekly/_log_<data>*.txt`, **e** nenhuma `weekly/run_<data>_<HHMM>` começou depois do último log do dia (execução em andamento), **e** não existe `MYPFF_Complete.db.new_tmp` (troca em curso) |
| Espera | reverifica a cada 15 min, até 3 h (`--interval`, `--max-wait`) |
| Segunda linha de defesa | se houver colisão mesmo assim, a guarda do Cowork (banco vivo × retrato do início) cancela a troca; o banco não corrompe |
| Log próprio | `pff_data/identity/_windows_rebuild_<data>.log` (acrescenta; uma linha por verificação, saída completa do rebuild, linha `DESFECHO`) |
| Código de saída | 0 rodou (de imediato ou após esperar) · 1 erro ou timeout do rebuild (15 min), mapa anterior intacto · 2 abortou por limite · 3 erro de configuração |

**Tarefa agendada — criação** (PowerShell do owner, uma vez):
```powershell
$py   = 'C:\Users\Erico Mello\AppData\Local\Programs\Python\Python313\python.exe'
$repo = 'C:\Users\Erico Mello\Fantasy\predictor'
$action    = New-ScheduledTaskAction -Execute $py -Argument '-m scripts.mypff_identity_weekly' -WorkingDirectory $repo
$trigger   = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Wednesday -At 10:00
$settings  = New-ScheduledTaskSettingsSet -StartWhenAvailable -MultipleInstances IgnoreNew `
               -ExecutionTimeLimit (New-TimeSpan -Hours 4) -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
$principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" -LogonType Interactive -RunLevel Limited
Register-ScheduledTask -TaskPath '\Fantasy\' -TaskName 'MYPFF identity rebuild (W1-F3)' `
  -Action $action -Trigger $trigger -Settings $settings -Principal $principal `
  -Description 'MYPFF-W1-F3: rebuild da mypff_identity depois da tarefa do Cowork das 07:00'
```
- `python.exe` é o do Python 3.13 com `python-dotenv` (confirmado pelo owner). O `WindowsApps\python.exe`
  é o stub da Store: não usar.
- O console fica aberto durante a espera. Para rodar sem janela, trocar por `pythonw.exe` da mesma
  pasta: o wrapper funciona sem stdout, e o registro fica no log próprio e no código de saída.

**Inspecionar, testar e remover:**
```powershell
Get-ScheduledTask     -TaskPath '\Fantasy\' -TaskName 'MYPFF identity rebuild (W1-F3)'
Get-ScheduledTaskInfo -TaskPath '\Fantasy\' -TaskName 'MYPFF identity rebuild (W1-F3)'   # LastRunTime, LastTaskResult, NextRunTime
schtasks /Query /TN "\Fantasy\MYPFF identity rebuild (W1-F3)" /V /FO LIST
Start-ScheduledTask   -TaskPath '\Fantasy\' -TaskName 'MYPFF identity rebuild (W1-F3)'   # execucao manual
Unregister-ScheduledTask -TaskPath '\Fantasy\' -TaskName 'MYPFF identity rebuild (W1-F3)' -Confirm:$false
```

**Só com o owner logado** (`-LogonType Interactive`, trade-off aceito para não guardar senha):
- **PC desligado ou sem sessão às 10:00:** o `StartWhenAvailable` roda a tarefa quando ele ligar.
- **O wrapper ainda exige o log do Cowork do dia.** Se a tarefa do Cowork também não rodou (PC desligado
  às 07:00), o wrapper aborta por limite (exit 2). O desfecho correto é rodar as duas à mão: primeiro a
  do Cowork, depois `Start-ScheduledTask` (ou o wrapper com `--date`).

**Roteiro de validação no Windows** (owner, fora de quarta):
1. A partir de `predictor/`, rodar `python -m scripts.mypff_identity --rebuild` e anotar o
   `content_sha256` da linha `GRAVADO`.
2. Criar um log fictício do dia:
   `New-Item "C:\Users\Erico Mello\Fantasy\pff_data\weekly\_log_$(Get-Date -Format yyyy-MM-dd)_TESTE.txt"`.
3. `Start-ScheduledTask` e aguardar.
   - `Get-ScheduledTaskInfo` deve mostrar `LastTaskResult = 0`.
   - `identity\_windows_rebuild_<hoje>.log` deve ter `DESFECHO: REBUILD RODOU de imediato` com o
     **mesmo** `content_sha256` do passo 1 (as fontes baixadas no mesmo dia são as mesmas, salvo
     publicação entre os dois downloads).
4. Rodar `python -m scripts.mypff_identity --check`: exit 0.
5. Apagar o `_log_<hoje>_TESTE.txt`.

Backup pré-criação da tabela: `C:\Users\Erico Mello\fantasy_backups\MYPFF_Complete_pre_W1F2_2026-09-25.db`.

### Correção dos `sleeper_id` da `mypff` (OPT-B37-F2, 25/09/2026)

Os `sleeper_id` da tabela `mypff` foram corrigidos pela `mypff_identity` (script de uso único
`predictor/scripts/mypff_b37_fix.py`, autorizado pela regra `MYPFF-*` via B37): 130 linhas de 15 legados
errados reescritas e 168 linhas de homônimos anuladas em 47 IDs compartilhados com árbitro. Restam 68 IDs
compartilhados sem árbitro, sem escrita. A `mypff_weekly` **não** foi tocada (67 linhas carregam IDs copiados).
Backup pré-correção: `C:\Users\Erico Mello\fantasy_backups\MYPFF_Complete_pre_B37F2_2026-09-25.db`.

~~⚠️ **O Render ainda não tem a correção:** a cópia de produção do MYPFF só muda pelo runbook §20, e o seed de maio
do repositório do Optimizer também carrega os homônimos. Até essa cópia rodar, as telas do Render que calculam
por requisição leem os homônimos (pendência nomeada no item B37).~~ **✅ Resolvido em 01/10/2026:** o seed do
Optimizer foi atualizado (só a `mypff`, sha256 `041e5dd6…a46f9ac0`, commit `bf7e3d1`), o owner trocou o
`/data/MYPFF_Complete.db` pelo §20 e o PROC1 foi conferido. **Backup de produção:**
`/data/MYPFF_Complete_pre_B37B39_2026-10-01.db` (o seed de maio, com os homônimos). **Retenção:** até a próxima troca
do MYPFF em produção validada por PROC1 (MYPFF-U2); depois pode ser apagado.

