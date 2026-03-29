# Fork: Environment Discovery para Skills Context-Aware

**Branch:** `laravel-boost-support`
**Data:** 2026-03-29
**Autor:** Daniel Lopes
**Base:** `main` do upstream `EveryInc/compound-engineering-plugin`

---

## Problema

As skills `ce:brainstorm`, `ce:plan` e `ce:work` do compound-engineering funcionam de forma generica. Quando executadas dentro de um projeto Laravel com Livewire e Laravel Boost MCP, elas:

1. **Ignoram o CLAUDE.md** — tratam como "compatibility fallback", mas projetos Laravel usam CLAUDE.md como arquivo principal de instrucoes
2. **Nao descobrem MCP servers** — o Laravel Boost MCP (que oferece introspecao de rotas, models, migrations) e completamente invisivel para o workflow
3. **Nao descobrem skills locais** — skills instaladas no `.claude/skills/` do projeto nao sao referenciadas
4. **Nao propagam contexto para subagentes** — quando ce:work despacha subagentes de implementacao, eles nao sabem quais ferramentas do projeto estao disponiveis

## O que foi alterado

### 6 arquivos modificados (158 linhas adicionadas, 17 removidas)

| Arquivo | Mudanca |
|---------|---------|
| `skills/ce-brainstorm/SKILL.md` | Phase 0.5: Environment Discovery + propagacao no handoff |
| `skills/ce-plan/SKILL.md` | Phase 0.7: Environment Discovery + contexto nos research agents + secao "Project Environment" no template do plano |
| `skills/ce-work/SKILL.md` | Phase 0.1: Environment Discovery + propagacao para subagentes |
| `agents/research/repo-research-analyst.md` | Phase 0.0 e 0.4: aceita environment context + detecta MCPs + inclui na saida |
| `agents/research/best-practices-researcher.md` | Phase 0: verifica MCPs do projeto antes de pesquisa externa + hierarquia de fontes atualizada |
| `agents/research/framework-docs-researcher.md` | Aceita environment context + usa MCPs do projeto primeiro para introspecao |

### Detalhes por mudanca

#### 1. CLAUDE.md elevado a cidadao de primeira classe

**Antes:** `"CLAUDE.md only if retained as compatibility context"`
**Depois:** `"CLAUDE.md and AGENTS.md -- both are primary sources when present"`

Alterado em: ce-brainstorm, ce-plan, ce-work, repo-research-analyst

#### 2. Environment Discovery (nova fase)

Adicionada uma fase de descoberta de ambiente em cada skill de workflow:

- **ce-brainstorm** → Phase 0.5
- **ce-plan** → Phase 0.7
- **ce-work** → Phase 0.1

Cada fase descobre:
- Arquivos de instrucao do projeto (`CLAUDE.md`, `AGENTS.md`)
- MCP servers disponiveis na sessao
- Skills locais do projeto (`.claude/skills/`)

Produz um bloco `<environment-context>` que e propagado entre workflows.

#### 3. Propagacao de contexto

- ce-brainstorm passa `<environment-context>` no handoff para ce:plan e ce:work
- ce-plan passa o contexto para research agents e inclui secao "Project Environment" no plano gerado
- ce-work passa o contexto para subagentes de implementacao com instrucao para preferir MCPs do projeto

#### 4. Research agents conscientes de MCPs

- **repo-research-analyst**: detecta MCP servers configurados e inclui na saida de Technology & Infrastructure
- **best-practices-researcher**: verifica MCPs do projeto ANTES de ir buscar docs online. Hierarquia: MCP data > skills > official docs > community
- **framework-docs-researcher**: usa MCPs do projeto primeiro para introspecao (routes, models, schema), depois Context7 para guidance conceitual

## Fluxo resultante

```
/ce:brainstorm "nova feature X"
  -> Phase 0.5: Descobre Laravel Boost MCP, CLAUDE.md, skills locais
  -> Brainstorm informado pelo contexto do projeto
  -> Handoff com <environment-context>

/ce:plan
  -> Phase 0.7: Adota environment context do brainstorm (ou redescobre)
  -> Research agents usam Laravel Boost MCP para routes/models/schema
  -> Plano gerado inclui secao "Project Environment" com MCPs listados

/ce:work
  -> Phase 0.1: Le "Project Environment" do plano (ou redescobre)
  -> Subagentes recebem contexto + instrucao para usar Laravel Boost MCP
  -> Implementacao usa ferramentas reais do projeto
```

## Compatibilidade com upstream

As mudancas sao **aditivas** — nenhum comportamento existente foi removido:

- Skills sem MCPs ou CLAUDE.md continuam funcionando normalmente
- A descoberta de ambiente e um passo adicional que simplesmente nao encontra nada em projetos sem essas ferramentas
- O template do plano tem a secao "Project Environment" como opcional (comentario HTML diz para omitir quando nao ha tooling especial)
- Research agents continuam usando Context7 e web search como fallback

**Risco de conflito de merge:** MEDIO. As mudancas tocam em areas de alto trafego das skills (Phase 0/1), entao atualizacoes upstream nessas fases provavelmente gerarao conflitos. Os conflitos serao textuais e resolviveis manualmente.

---

## Guia de manutencao: sincronizar com upstream

### Setup inicial (uma vez)

```bash
# Dentro do seu fork clonado
git remote add upstream https://github.com/EveryInc/compound-engineering-plugin.git
git fetch upstream
```

### Atualizar quando sair nova versao do upstream

```bash
# 1. Buscar atualizacoes do upstream
git fetch upstream

# 2. Ir para a branch main do seu fork
git checkout main
git merge upstream/main
git push origin main

# 3. Rebasear a branch de modificacoes sobre o main atualizado
git checkout laravel-boost-support
git rebase main
```

**Se houver conflitos durante o rebase:**

```bash
# Ver quais arquivos tem conflito
git status

# Para cada arquivo com conflito, abrir e resolver manualmente.
# Os conflitos estarao nos mesmos 6 arquivos listados acima.
# Manter AMBAS as mudancas: as do upstream E as do environment discovery.

# Depois de resolver cada arquivo:
git add <arquivo-resolvido>
git rebase --continue

# Quando terminar:
git push origin laravel-boost-support --force-with-lease
```

### Reinstalar o plugin apos atualizar

```bash
# No Claude Code
/plugin uninstall compound-engineering
/plugin install compound-engineering
```

### Checklist de resolucao de conflitos

Ao resolver conflitos, verifique que estas mudancas do fork estao preservadas:

- [ ] **ce-brainstorm**: Phase 0.5 (Environment Discovery) existe entre Phase 0.3 e Phase 1
- [ ] **ce-brainstorm**: Handoff (Phase 4.2) menciona `<environment-context>`
- [ ] **ce-plan**: Phase 0.7 (Environment Discovery) existe antes da Phase 1
- [ ] **ce-plan**: Phase 1.1 dispatch inclui "with environment context"
- [ ] **ce-plan**: Phase 1.3 dispatch inclui "with environment context"
- [ ] **ce-plan**: Template do plano tem secao "Project Environment"
- [ ] **ce-work**: Phase 0.1 (Environment Discovery) existe no inicio do Phase 0
- [ ] **ce-work**: Subagent dispatch inclui `<environment-context>` block
- [ ] **repo-research-analyst**: Phase 0.0 e 0.4 existem
- [ ] **repo-research-analyst**: Output format inclui "Available MCP servers" e "Project-local skills"
- [ ] **best-practices-researcher**: Phase 0 (Check Environment Context) existe antes do Phase 1
- [ ] **best-practices-researcher**: Hierarquia de qualidade inclui "project MCP server data" no topo
- [ ] **framework-docs-researcher**: Secao "Environment Context" existe antes de "Core Responsibilities"
- [ ] **framework-docs-researcher**: Documentation Collection menciona MCPs do projeto primeiro
- [ ] Em TODOS os 6 arquivos: CLAUDE.md e tratado como fonte primaria (nao "compatibility fallback")

### Verificar apos merge

```bash
# Rodar validacao do plugin
bun run release:validate

# Verificar que os testes nao quebraram
bun test
```

## Arquivos de referencia

Para entender o que cada arquivo faz no contexto do plugin:

- `plugins/compound-engineering/AGENTS.md` — Regras de desenvolvimento do plugin
- `AGENTS.md` (raiz) — Instrucoes gerais do repositorio
- `docs/solutions/skill-design/` — Decisoes de design das skills
