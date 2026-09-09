# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este projeto

Dashboard de consumo de bebidas alcoólicas para a Nitro / "Deixa Comigo Bebidas".
Não é uma aplicação com build: **todo o produto é um único arquivo estático**,
`Dashboards/index.html` (~2.050 linhas, ~230 KB), com CSS e JS inline e **zero
dependências de runtime** — nada de npm, bundler, CDN ou fetch em tempo de execução.

```
Dados/drinks.csv                          fonte de teste (193 países, read-only)
Dados/nitro_brand_book_by_pomelli.pdf     brand book (está aqui, NÃO em Referencias/)
Dados/nitro_consumo_alcool_*.csv          exportações geradas pelo próprio dash
Dashboards/index.html                     o entregável — único arquivo de produto
server.js                                 proxy local opcional (Node puro, zero deps)
.env / .env.example                       chaves de API — .env NÃO é versionado
Referencias/                              vazia
```

## Executar e verificar

Não há build, lint nem suíte de testes. Para rodar, abra o arquivo:

```bash
start Dashboards/index.html
```

Chat com IA e previsão do tempo exigem o proxy local, porque as chaves ficam no
`.env` e **nunca** podem ir para o front-end:

```bash
cp .env.example .env   # preencher GEMINI_API_KEY e OPENWEATHER_API_KEY
node server.js         # http://localhost:4173
```

Sem servidor (`file://`) o painel continua completo; só esses dois recursos se
desligam, com aviso na interface. Nunca introduza um `config.js` com chaves,
nem embuta chave no HTML: se precisar de um novo serviço externo, some uma rota
de proxy ao `server.js`.

O dashboard **exige uma importação de CSV** — sem arquivo ele mostra apenas a
tela inicial (dropzone). Não existe nenhum dado embutido além da geografia; isso
é um requisito do cliente, não um detalhe de implementação: **nunca hard-code
valores do dataset**.

Para inspecionar estado renderizado durante o desenvolvimento, use o browser
in-app e `javascript_tool` (`S`, `filtered()`, `stats(...)` são acessíveis pelo
escopo do script inline). Para automatizar a importação, copie o `index.html`
para um harness temporário que chame `load()` com um `File` sintético — `load`
não é exposto em `window`, então o harness precisa injetar a chamada dentro do
próprio `<script>`. Apague o harness ao terminar.

Notas do ambiente: Python não está instalado; para PDFs use o `pdftotext` do
mingw64 do Git (`pdfimages`/`pdftoppm` não existem, por isso a logo é um lockup
tipográfico em SVG e não o asset oficial).

## Arquitetura de `Dashboards/index.html`

A ordem do arquivo é: `<style>` (tokens → app bar → layout → empty state →
filtros → KPIs → chrome de gráficos → footer → FAB/modal) → markup →
`<script>window.NITRO_GEO={…}</script>` → script principal, dividido em 16
seções numeradas por comentário (`1 · ESTATÍSTICA`, `2 · IMPORTAÇÃO DO CSV`, …,
`16 · BOOT / EVENTOS`, `17 · SERVIÇOS EXTERNOS`, `18 · CLIMA`,
`19 · CHAT COM IA`). Use esses cabeçalhos para navegar: `grep -n "^   [0-9]* ·"`.

Fluxo de dados, ponta a ponta:

1. **Importação** (§2) — `load(file)` → `FileReader` → `ingest(text)`:
   `sniffDelim` (`, ; tab |`) → `parseCSV` (campos com aspas) → `mapHeader`
   (aliases PT/EN em `COLS`, match exato e depois por substring) → `toNum`
   (decimal pt-BR e en) → dedupe por `norm(name)`.
   `ingest` **nunca lança**: devolve `{rows, problems, file}` ou
   `{problems}`, e `problems` vira aviso no console / painel de erro.
2. **Join geográfico** — `GEO.index[norm(nome)]` → id numérico ISO-3166 →
   `GEO.meta[id]` = `{n:nome, a:ISO3, r:continente, s:sub-região}`. Nomes que
   não casam entram nas estatísticas e na tabela com `cont:'Não classificado'`
   e ficam fora do mapa (`mapped:false`) — degradação intencional, não erro.
3. **Estado** — o objeto único `S` (métrica ativa, continentes, países,
   faixa, classificação do mapa, par do scatter, ordenação, zoom). Toda
   interação muta `S` e chama `renderAll()`; não há estado em DOM.
4. **Seleção** — `filtered()` aplica continente + país + faixa; `recomputeDomain()`
   refaz o domínio quando a métrica muda.
5. **Render** (§7–§14) — cada gráfico é uma função pura `render*(rows, …)` que
   monta string de SVG/HTML e substitui `innerHTML`. Sem lib de charting.
6. **Serviços externos** (§17–§19) — `fetch` só contra `/api/*` na mesma
   origem, servido por `server.js`. `aiContexto()` serializa a saída de
   `filtered()` (com estatísticas e correlações do recorte) e é a única coisa
   que o Gemini enxerga: o chat respeita os filtros por construção, não por
   instrução no prompt. `renderAll()` chama `AI.sincronizar()` para manter a
   fita de contexto fiel ao estado — ao criar um novo filtro, garanta que ele
   apareça em `aiContexto()`.

`window.NITRO_GEO` (§3) é um payload TopoJSON Natural Earth 1:110M com
`transform`/`arcs` delta-encoded, `geometries`, `meta` e `pts` (centroides dos
29 micro-Estados sem polígono nessa escala, para que os 193 países do CSV
apareçam no mapa). Projeção **Equal Earth** (`EE`), enquadrada por `FIT`,
Antártica excluída via `SKIP_GEO`. Rings que cruzam o antimeridiano (Rússia,
Fiji) são quebrados em sub-caminhos dentro do builder `GEOPATH` — remover esse
tratamento reintroduz faixas horizontais atravessando o mapa.

Estatística (§1) é implementada à mão e deve continuar assim: desvio padrão
amostral (n−1), mediana/quartis por interpolação linear, `pearson`, e p-valor
bicaudal via beta incompleta (`betacf`/`gammaln`/`betai`).

## Convenções que precisam ser mantidas

- **Idioma**: toda a UI, os comentários e os rótulos em **português (pt-BR)**;
  números via `nf()`/`nf0()` com `toLocaleString('pt-BR')`.
- **Cores**: as cores puras do brand book (Admiral `#003663`, Citron `#B9DA00`,
  Chartreuse `#94C356`, Gunmetal `#424242`) são identidade; as **séries de dados
  são uma paleta derivada e validada** — bebidas `#8A9B00`/`#0075BF`/`#A62C5B`/`#003663`,
  continentes em `KCOLOR`, rampa sequencial `RAMP`, divergente `DIV`. Ao mexer em
  qualquer uma delas, rode o validador da skill `dataviz`
  (`node scripts/validate_palette.js "<hex,…>" --mode light`) antes de aceitar;
  o citron cru tem contraste 1,56:1 e não serve como cor de série.
- Estrutura de componentes (raio `.75rem`, cards brancos, sombras, badges, fundo
  `#F4F4F6`) segue a skill `nitro-padrao-sistemas`; tipografia Poppins.
- Um único tema claro, deliberado. Não adicionar dark mode por inversão — exigiria
  novos passos de paleta validados contra a superfície escura.
- Acessibilidade já implementada e a manter: legenda sempre presente, tabela como
  alternativa textual, `aria-pressed`/`aria-sort`/`aria-expanded`,
  `prefers-reduced-motion`, foco visível em citron, estilos de impressão.
- Animações via `rAF` nunca devem ser a única fonte de um valor exibido —
  `countUp()` escreve o número final primeiro e só depois anima (rAF é
  suspenso em aba/painel oculto).
- Atalhos de teclado: `O` importar, `1`–`4` tipo de bebida, `R` redefinir,
  `G` conversar com os dados. Ao
  adicionar filtro ou controle, incluí-lo também em `#btnReset` e ressincronizar
  os `aria-pressed` no DOM.

## Pendência conhecida

O FAB "Falar com o suporte" abre um modal cujo `submit` apenas exibe confirmação
local — a integração com banco de dados está marcada como `TODO` no fim do script.
