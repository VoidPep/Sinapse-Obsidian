# Sinapse para Obsidian — design (v1)

Data: 2026-11-24 · Status: aguardando aprovação

## Intenção

O [Sinapse](https://github.com/VoidPep/Sinapse-RAG) é um executável (`sinapse`) que indexa notas markdown em SQLite FTS5 e serve busca BM25 ao Claude Code via MCP. Ele é configurado por um `sinapse.json` e operado por linha de comando. Para o time, editar JSON à mão e abrir terminal para reindexar é a barreira de adoção.

Este plugin é a "tela de setup": dentro do Obsidian, a pessoa escolhe quais pastas do vault entram no índice, reindexa com um clique, vê a economia de tokens e copia o comando para conectar o Claude Code — sem saber que existe um JSON.

## Objetivos (v1)

1. Aba de configurações que lê e grava o `sinapse.json` do vault (pastas, exclusões, pasta de dados), preservando campos que o plugin não conhece.
2. Localizar o executável no vault (padrão por SO) ou por caminho configurado, e validar a versão (`sinapse --version --json`, mínimo `0.1.0`).
3. Reindexar por botão e por comando da paleta (`sinapse index --config … --json`), mostrando resultado ou erro legível.
4. Abrir o relatório de economia (`sinapse report --config … --json` → abre o HTML gerado).
5. Mostrar o comando `claude mcp add …` pronto para copiar, com os caminhos do vault da pessoa.
6. Desktop only; Windows e Linux (Fedora). macOS não é alvo (não há binário), mas não deve quebrar o plugin — só avisar.
7. Release no formato exigido pelo Obsidian (tag `X.Y.Z` sem `v`, assets `main.js`, `manifest.json`, `styles.css`), instalável via BRAT.

## Não-objetivos (v1)

- Baixar ou atualizar o executável (as diretrizes do Obsidian não permitem baixar binários sem consentimento explícito; a pessoa coloca o exe no vault).
- Busca dentro do Obsidian (a busca é para o Claude; o Obsidian já tem a sua).
- Reindexar automaticamente a cada alteração de nota.
- Editar `include` (fica `**/*.md`; editável só no JSON).
- Publicação na loja oficial de plugins (passo posterior; v1 é via BRAT).
- Mobile.

## Anti-objetivos

- Corromper ou apagar o `sinapse.json` da pessoa (campos desconhecidos somem, JSON inválido sobrescrito sem aviso).
- Travar a interface do Obsidian enquanto indexa.
- Rodar duas indexações ao mesmo tempo a partir do plugin.
- Qualquer referência a empresa, usuário ou caminho local no código ou nos docs.

## Contrato com o executável (definido no Sinapse v0.1.0)

| Chamada | Saída (`--json`, uma linha em stdout) |
|---|---|
| `sinapse --version --json` | `{"ok":true,"version":"0.1.0"}` |
| `sinapse index --config <arq> --json` | `{"ok":true,"files":N,"chunks":N,"ms":N}` |
| `sinapse report --config <arq> --json` | `{"ok":true,"path":"<html>"}` |
| qualquer erro | `{"ok":false,"error":"..."}` |

Exit codes: `0` sucesso, `1` erro de execução, `2` configuração inválida/ausente.

`sinapse.json` (caminhos relativos à pasta do arquivo):

```json
{ "name": "...", "base": ".", "folders": [], "include": ["**/*.md"],
  "exclude": ["**/.obsidian/**", "**/.trash/**"], "dataDir": ".sinapse" }
```

`folders` vazio = vault inteiro.

## Abordagens consideradas

| Abordagem | Trade-off | Decisão |
|---|---|---|
| Plugin edita `sinapse.json` e chama o exe | Uma única fonte de verdade, o exe continua usável sem o plugin; depende do contrato acima | **Escolhida** |
| Plugin guarda a config no `data.json` dele e gera o `sinapse.json` | Duas fontes de verdade que divergem quando alguém edita o JSON à mão | Rejeitada |
| Reimplementar a indexação em TypeScript | Sem exe; mas duplica chunking/BM25 e quebra a paridade com o servidor MCP | Rejeitada |
| Plugin baixa o exe das releases | Instalação em um passo; viola diretrizes do Obsidian e é bloqueado por antivírus corporativo | Rejeitada |

## Comportamento

**Onde ficam as coisas** *(decisões do autor do design)*:
- `sinapse.json` na raiz do vault (configurável no plugin; padrão `sinapse.json`). O plugin sempre passa `--config <caminho absoluto>`.
- Executável: se a configuração do plugin estiver vazia, procura na raiz do vault `sinapse-win-x64.exe` ou `sinapse.exe` (Windows) / `sinapse-linux-x64` ou `sinapse` (Linux). Caminho configurável (relativo ao vault ou absoluto).
- Configurações do próprio plugin (caminho do exe, caminho do `sinapse.json`, toggles) ficam no `data.json` padrão do Obsidian.

**Aba de configurações**, nesta ordem:
1. **Executável** — campo de caminho + botão "Testar": roda `--version`; mostra "sinapse 0.1.0 ✓", ou erro (não encontrado, sem permissão de execução no Linux → sugerir `chmod +x`, versão abaixo da mínima).
2. **Pastas indexadas** — lista com botão remover; "Adicionar pasta" abre um seletor com busca das pastas do vault. Lista vazia mostra "Vault inteiro".
3. **Exclusões** — textarea, um glob por linha.
4. **Pasta de dados** — texto, padrão `.sinapse`.
5. **Reindexar** — botão; desabilitado enquanto roda; mostra o último resultado ("134 arquivos, 2.891 chunks em 850 ms" ou o erro).
6. **Reindexar ao abrir o Obsidian** — toggle, desligado por padrão; roda em segundo plano depois que o layout carrega.
7. **Relatório de economia** — botão "Abrir".
8. **Conectar ao Claude Code** — comando pronto, com botão copiar:
   `claude mcp add sinapse_knowledge_base -s user -- "<exe absoluto>" mcp --config "<sinapse.json absoluto>"`

Toda alteração nos campos 2–4 grava o `sinapse.json` (com debounce curto), mesclando com o conteúdo existente.

**Comandos da paleta:** "Sinapse: Reindexar", "Sinapse: Abrir relatório de economia".

**Regras de gravação do `sinapse.json`:**
- Arquivo ausente → criado com os padrões acima e `name` = nome do vault.
- Arquivo presente e válido → só `folders`, `exclude` e `dataDir` são alterados; demais campos e campos desconhecidos preservados.
- Arquivo com JSON inválido → **não** sobrescreve; aba mostra o erro e um botão "Recriar com padrões" (que renomeia o atual para `sinapse.json.bak` antes).
- Pastas gravadas relativas ao vault, com `/`.

**Execução do exe:** `child_process.execFile` (sem shell), `cwd` = raiz do vault, timeout de 10 min, uma execução por vez (fila de 1: clique durante indexação mostra "já em andamento"). Resultado = última linha de stdout que seja JSON; se não houver, usar stderr como erro. Exit `2` → mensagem "Configuração inválida: …".

**Notificações:** `Notice` no início ("Indexando…") e no fim (resultado ou erro) quando a ação vem da paleta ou do toggle de abertura; na aba, o resultado aparece no próprio campo de status.

## Identidade do plugin

`manifest.json`: id `sinapse`, nome `Sinapse`, `isDesktopOnly: true`, `minAppVersion` `1.5.0`, autor `VoidPep`. O id não contém "obsidian" (regra da loja).

## Estrutura

```
sinapse-obsidian/
  manifest.json  versions.json  styles.css
  package.json  tsconfig.json  esbuild.config.mjs  version-bump.mjs
  src/
    main.ts              plugin: comandos, ciclo de vida, wiring
    settings.ts          aba de configurações (UI)
    folderSuggest.ts     seletor de pastas do vault
    sinapseConfig.ts     ler/mesclar/gravar sinapse.json (puro, fs injetado)
    sinapseCli.ts        montar args, executar, interpretar saída (execFile injetado)
    executable.ts        localizar exe por SO
    version.ts           comparar versões
  tests/                 vitest para os módulos puros
  .github/workflows/     ci.yml (build + testes + lint), release.yml (tag X.Y.Z)
  docs/superpowers/      specs/ plans/
```

Módulos puros (`sinapseConfig`, `sinapseCli`, `executable`, `version`) não importam `obsidian`; recebem dependências por parâmetro e são testados com vitest. `main.ts`/`settings.ts`/`folderSuggest.ts` são a casca de UI, validada manualmente.

## Testes

- Unidade (vitest): mesclagem do `sinapse.json` (preserva desconhecidos, JSON inválido não sobrescreve, caminhos com `/`), interpretação da saída do CLI (JSON ok, JSON erro, sem JSON, exit 2), localização do exe por plataforma, comparação de versão.
- Manual em um vault de teste, com o exe da release v0.1.0 (Windows e Fedora): roteiro no plano.
- CI: `npm ci`, lint, `tsc --noEmit`, vitest, build.

## Release

- Branches `feat/*` → `develop` → `main`. Tag `X.Y.Z` (sem `v`) na `main` → workflow publica `main.js`, `manifest.json`, `styles.css`.
- `versions.json` mapeia versão → `minAppVersion`; `npm version` atualiza `manifest.json` e `versions.json` (script `version-bump.mjs`, padrão do sample plugin).
- README: instalação via BRAT, onde colocar o exe, `chmod +x` no Linux, o que cada campo faz.

## Deixado para o builder

- Versões exatas de `obsidian` (latest hoje: 1.14.4), `esbuild`, `typescript`, `vitest`, e se usa `eslint-plugin-obsidianmd`.
- Estilo da aba (usar componentes `Setting` nativos; CSS mínimo).
- Tempo de debounce da gravação.
- Como abrir o HTML do relatório (`window.open('file://…')` ou `electron.shell.openPath`).

## Dependência do Sinapse

`sinapse mcp --config <arq>` já é aceito no Sinapse v0.1.0 (`src/Sinapse/Program.cs`), então o comando da seção 8 funciona sem mudanças no exe.
