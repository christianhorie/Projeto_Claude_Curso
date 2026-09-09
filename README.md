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
server.js                                 servidor local opcional (Node puro)
.env.example                              modelo das variáveis de ambiente
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

### Com chat de IA e previsão do tempo

Dois recursos dependem de APIs externas e, portanto, de chaves: o chat com o
Google Gemini e a previsão do tempo (OpenWeatherMap). Chave em JavaScript de
front-end fica exposta a qualquer visitante — por isso elas vivem no `.env`,
lidas por `server.js`, e **nunca chegam ao browser**.

```bash
cp .env.example .env      # preencha GEMINI_API_KEY e OPENWEATHER_API_KEY
node server.js            # http://localhost:4173
```

O `server.js` é Node puro, sem nenhuma dependência: lê o `.env`, serve
`Dashboards/` e expõe três rotas de proxy — `/api/config` (só flags de
disponibilidade), `/api/weather` e `/api/chat`.

Abrir o `index.html` direto pelo `file://` continua funcionando: apenas o
chat e o widget de clima se desligam, com aviso na própria interface.

> **Nunca versione o `.env` nem cole chaves em chat, e-mail ou ticket.** Uma
> chave que vazou deve ser revogada e regerada, não reaproveitada.

### Chat com IA — o que o modelo enxerga

O assistente recebe **apenas a seleção ativa**: a mesma lista que `filtered()`
entrega aos gráficos, mais as estatísticas descritivas e as correlações
daquele recorte. Trocar continente, país fixado, métrica ou faixa muda a
resposta. Acima de 250 países no recorte, o contexto leva as duas pontas do
ranking em vez da tabela inteira. A fita no topo do painel mostra exatamente
o que está sendo enviado.

`GEMINI_MODELS` define uma **cadeia de fallback**: se o modelo preferido
falhar por cota (429), indisponibilidade (5xx) ou por ter sido aposentado
(404), o servidor tenta o próximo da lista e informa na resposta qual modelo
respondeu. Credencial inválida interrompe a cadeia — insistir não resolve.

## Arquitetura

O arquivo `Dashboards/index.html` está organizado em `<style>` → markup →
`window.NITRO_GEO` (payload geográfico TopoJSON) → script principal, dividido
em 19 seções numeradas por comentário (`1 · ESTATÍSTICA`, `2 · IMPORTAÇÃO DO
CSV`, …, `16 · BOOT / EVENTOS`, `17 · SERVIÇOS EXTERNOS`, `18 · CLIMA`,
`19 · CHAT COM IA`). Detalhes completos do fluxo de dados,
convenções de cores/paleta e acessibilidade estão em [`CLAUDE.md`](CLAUDE.md).

## Pendência conhecida

O FAB "Falar com o suporte" abre um modal cujo `submit` apenas exibe
confirmação local — a integração com banco de dados está marcada como `TODO`
no fim do script.
