# Dashboard de Consumo de Bebidas Alcoólicas — Nitro

Dashboard interativo de consumo de bebidas alcoólicas para a Nitro / "Deixa
Comigo Bebidas". Não é uma aplicação com build: **todo o produto é um único
arquivo estático**, [`Dashboards/index.html`](Dashboards/index.html), com CSS
e JS inline e **zero dependências de runtime** (sem npm, bundler, CDN ou fetch
em tempo de execução).

## Estrutura

```
Dados/drinks.csv                          fonte de teste (193 países, read-only)
Dados/nitro_brand_book_by_pomelli.pdf     brand book da Nitro
Dashboards/index.html                     o entregável — único arquivo de produto
Referencias/                              vazia
```

> Exportações geradas pelo próprio dashboard (`Dados/nitro_consumo_alcool_*.csv`)
> não são versionadas — veja `.gitignore`.

## Como executar

Não há build, lint nem suíte de testes. Basta abrir o arquivo no navegador:

```bash
start Dashboards/index.html
```

O dashboard **exige a importação de um CSV** para exibir qualquer dado — sem
arquivo importado ele mostra apenas a tela inicial (dropzone). Não existe
nenhum dado embutido além da geografia; isso é um requisito do cliente, não um
detalhe de implementação: **nunca hard-code valores do dataset**.

Para testar rapidamente, importe `Dados/drinks.csv` pela própria interface.

## Arquitetura

O arquivo `Dashboards/index.html` está organizado em `<style>` → markup →
`window.NITRO_GEO` (payload geográfico TopoJSON) → script principal, dividido
em 16 seções numeradas por comentário (`1 · ESTATÍSTICA`, `2 · IMPORTAÇÃO DO
CSV`, …, `16 · BOOT / EVENTOS`). Detalhes completos do fluxo de dados,
convenções de cores/paleta e acessibilidade estão em [`CLAUDE.md`](CLAUDE.md).

## Pendência conhecida

O FAB "Falar com o suporte" abre um modal cujo `submit` apenas exibe
confirmação local — a integração com banco de dados está marcada como `TODO`
no fim do script.
