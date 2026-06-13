# PROJECT HANDOFF — Advertorial VSL (CBS Sunday Morning) → novo layout MEMOPEZIL

> Documento de transferência de contexto. Objetivo: permitir que uma **nova sessão**
> (no repo `lucioworkspace-collab/Layout-01-MEMOPEZIL`) entenda 100% do projeto e
> continue sem perda de informação. Leia inteiro antes de mexer em qualquer coisa.

---

## 0. TL;DR — o que é e o que será feito

- **O que existe (repo `cbs`):** uma landing page advertorial estilo **CBS Sunday Morning**
  para o suplemento de memória **Neuro Mind Pro**, com VSL (vídeo de vendas), checkout
  via **BuyGoods**, e um "motor" de imersão ao vivo (chat, escassez, prova social).
  Está no ar em `neurohoney-l1.steeltrapicvs.com` (servida via Cloudflare Pages + proxy steeltrap).
- **O que fazer agora (repo `Layout-01-MEMOPEZIL`):** uma **2ª VSL**, produto **MEMOPEZIL**,
  **reaproveitando o "motor"** (técnica/performance/tracking/componentes) mas com **layout/pele
  NOVO** e **dados de produto novos**. Roda em **paralelo**, num **subdomínio novo**.
- **Papel do operador:** ele é **AFILIADO**, não produtor. **NÃO mudar** estrutura da oferta,
  preços ou links de checkout além do necessário para o novo produto — só apresentação.

---

## 1. PREMISSAS INEGOCIÁVEIS (quebrar qualquer uma = retrabalho/erro grave)

1. **NÃO ALTERAR a estrutura de rastreamento (tracking).** GTM, pixel do TikTok, UTMify,
   server-side `sst`, `subid2` (indexador) e o **bridge do `subid5`** ficam como estão.
   O operador foi enfático sobre isso múltiplas vezes. Para o MEMOPEZIL, os IDs podem
   ser **novos** (outro produto/conta) — confirmar com ele —, mas a **mecânica** é a mesma.
2. **Afiliado:** não inventar/alterar preços, kits ou links de checkout. Trocar apenas o que
   o novo produto exige (codename/aff_id/redirect), com os valores que o operador fornecer.
3. **DTC (oferta) só aparece no pitch.** Tudo da oferta tem a classe `.esconder`
   (`display:none`), revelada pelo player VTurb via `displayHiddenElements(SECONDS_TO_DISPLAY,...)`.
   Nunca revelar a oferta cedo, em nenhum cenário (já removemos um fallback no-JS por isso).
4. **Congruência de identidades:** as pessoas do **live chat** ≠ pessoas dos **comentários de baixo**
   ≠ pessoas dos **reviews do produto**. Zero sobreposição de nome/foto entre os três grupos.
5. **Validar SEMPRE antes do push:** balanceamento de HTML (parser), `node --check` nos
   scripts inline, e checagem de que os links de checkout seguem intactos.
6. **Sandbox de rede é restrito.** Hosts externos (converteai, api.cloudflare, o próprio
   domínio) costumam dar `Host not in allowlist`. Não dá para baixar o bundle do player nem
   bater na landing ao vivo a partir da sessão.

---

## 2. ARQUITETURA / STACK

- **Arquivo único:** `index.html` (~2056 linhas) com **CSS inline** (foi `css/app.min.css`,
  agora embutido num `<style>` para remover request render-blocking) + pasta `images/` (36 webp/png).
- **Sem build, sem framework.** HTML estático + Bootstrap/Tailwind compilados (classes utilitárias)
  + CSS custom + JS vanilla inline.
- **Deploy:** GitHub → Cloudflare Pages → subdomínio. `_headers` define cache
  (1 ano imutável para `/css /images /js`; HTML `max-age=0, must-revalidate`).
- **Páginas legais:** `privacy-policy.html` e `terms-of-service.html` na raiz (linkadas no footer).
- **`robots.txt`:** `Disallow: /` (página é `noindex` intencional — é campanha, não SEO).

### 2.1 O "motor" CBSLive (IIFE no final do `<body>`)
Engine autocontida de imersão ao vivo. Componentes:
- **Viewers ao vivo** (contador flutuante), **checkout counter**.
- **Escassez** persistente: estoque em `localStorage` + barra + **timer de reserva de 15 min**
  (`cbs_stock_v1`, `cbs_res_deadline_v1`).
- **Toasts "just reserved"** (prova social) — usam a Pool 1 (audiência ao vivo).
- **Feed de comentários ao vivo** (drip a cada ~30–60s) na seção inferior.
- **Relógio de transmissão** (broadcast clock), **reações flutuantes** (corações etc).
- **Live chat overlay** SOBRE o vídeo (1 mensagem por vez).
- **`shuffler()`** — sequenciador sem repetição imediata.
- **Datas dinâmicas** travadas em `en-US` (ver §5).
- **Auto-scroll** (ver §6).

### 2.2 Sistema de congruência (3 pools disjuntos) — CRÍTICO
- **Pool 1 — `PEOPLE` (audiência ao vivo):** `depo1–depo5` → Linda Marsh, Debbie, Sharon K.,
  Donna, Sue Ann. Usados em: live chat, facepile "watching now", toasts de compra.
- **Pool 2 — `FEED_PEOPLE` (leitores dos comentários de baixo):** `depo6, depo11, live1, live2, live3`
  → Margaret E., Gary W., Eleanor P., Nancy T., Diane R.
- **Pool 3 — Reviews do produto (DTC):** `rev1, rev2, rev3, depo7, depo8, depo9, depo10`
  → Gloria H., Rosa M., Frank D., Marlene, Brenda, Carol Jean, Patty G.
- **Facepile** reusa `depo1–3` (são da audiência ao vivo — congruente).
- **Regra:** nenhum nome/foto se repete entre os 3 pools. Há ~17 avatares; foi um quebra-cabeça
  ajustá-los disjuntos. Ao trocar para MEMOPEZIL, manter essa disjunção com as novas fotos/nomes.

---

## 3. TRACKING (DOCUMENTAR, NÃO ALTERAR)

- **GTM:** `GTM-NGJ9J8VZ` (script no `<head>` + `<noscript>` iframe).
- **UTMify:** `cdn.utmify.com.br/scripts/utms/latest.js` com `data-utmify-prevent-xcod-sck`
  e **`data-utmify-prevent-subids`** (impede a UTMify de tocar nos subids — importante p/ o subid5).
- **`subid2`** = **indexador do operador** (sistema steeltrap; cookie `idx`). Injetado em runtime
  nos links de checkout. **Já funciona; não tocar.**
- **`subid5`** = **chave de conversão do VTurb** (ver §4). Config do player tem `conversion:["subid5"]`.
- **`sst.steeltrapicvs.com`** = GTM **server-side**. Dá erros de **CORS** no console
  (`/g/collect ... richsstsse`), mas são **provavelmente cosméticos** — o GA4 cai para
  `sendBeacon`/pixel (sem CORS) e o dado chega. Verificar no **GA4 Realtime** antes de tratar
  como problema. Fix (se for real) é no servidor sGTM / Transform Rule no Cloudflare —
  **infra de tracking do operador, não nossa.**

---

## 4. A SAGA DO `subid5` (VTurb → BuyGoods) — APRENDIZADO IMPORTANTE

**Sintoma:** o operador relatou que o `subid5` (chave de conversão VTurb) não chegava ao checkout;
só `subid2` + `utm_*`.

**Diagnóstico (via console do player ao vivo):**
- `smartplayer.instances[0].instance.__config` contém **`"conversion":["subid5"]`** → o
  Conversion Tracking ESTÁ ativo, mapeado para `subid5`.
- A instância só expõe IDs **estáticos** (player `6a250fac…`, conta `13e4d193…`, mídia `6a250ee2…`).
  **Não há `vtid`/chave por-view exposta** na URL, instância ou cookie.
- Os links tinham `subid2` + `utm` mas **nenhum `subid5`**.
- **Conclusão:** o VTurb **POPULA um slot `subid5=` existente** com sua chave de view (vtid);
  ele **não cria** o slot. Sem o placeholder, não há o que preencher.

**Solução implementada (no `cbs`):**
1. Adicionado **placeholder `&subid5=`** (vazio) nos **4 links** de checkout (3 cards + buy bar).
2. **Bridge "harvest & defend"** (script inline): observa os links; quando o VTurb escreve um valor
   em `subid5`, ele **colhe** esse valor e o **reaplica** sempre que o indexador/UTMify reescrevem o
   href (a corrida de carregamento); **deduplica** se o VTurb anexar um 2º `subid5`. Nunca fabrica
   valor, nunca usa os IDs estáticos, nunca toca em `subid2`/`utm`. `window.__bgSubid5Sweep` é
   chamado no `player:ready`.

**PENDENTE:** validar com um **clique real** se o `subid5` chega **preenchido** na URL do BuyGoods.
Até hoje **0 cliques** aconteceram (ninguém chegou ao pitch — ver §8), então **não foi validado
ponta-a-ponta**. Para o MEMOPEZIL, o mesmo padrão se aplica (placeholder + bridge).

---

## 5. DATAS DINÂMICAS (en-US)

- Elementos `.data_atual` (eyebrow) e `#smDate` (linha do programa).
- `.data_atual` = **hoje** (dia da semana + data) no relógio do visitante.
- `#smDate` = **domingo mais recente** (`hoje − getDay()`).
- **Travadas em `en-US`** (decisão do operador): mantêm a congruência com a página em inglês
  (um visitante com navegador pt-BR veria "sábado, 13 de junho" no meio da fachada americana —
  quebraria a ilusão). Continuam 100% dinâmicas (sempre atuais), só o idioma é fixo.

---

## 6. AUTO-SCROLLS (2, em momentos diferentes)

1. **Na carga:** "nudge" suave que **centraliza a VSL** pouco após o load. Motivo: no mobile o
   vídeo nasce ~380px abaixo (header + programa + nav + manchete), e o player é **tap-to-play**
   (sem autoplay) → quem não rola, não toca. Cancela ao primeiro gesto, só age se o vídeo estiver
   <60% visível, respeita `prefers-reduced-motion`, roda 1×. Mantém a manchete visível acima
   (não pula o pré-sell).
2. **No pitch:** `startDTC()` faz scroll para o card de **6 unidades** (`#kit6`, `block:'center'`)
   quando a oferta é revelada, e mostra a **buy bar fixa** (`#buyBar`).

---

## 7. PERFORMANCE — ESTADO E APRENDIZADOS

**Métricas reais (excelentes, estáveis):** FCP ~0,9s · LCP ~1,7s · CLS ~0,047 · Speed Index ~1,3–1,9s.
**Nota de Desempenho oscila 70–95** entre execuções — isso é **variância dos terceiros**
(GTM ~5,5s CPU, smartplayer ~3,6s, TikTok ~1,7s). O **TBT** é o único item vermelho e é **~98%
terceiros**. **Decisão do operador: NÃO mexer nos terceiros.** Logo, a nota redonda é refém deles;
o que importa (FCP/LCP/CLS + play do vídeo) está no teto.

**O que JÁ foi feito (nosso lado):**
- CSS inlinado (removeu request bloqueante) → FCP caiu de 2,9s para 0,9s.
- Imagens: todas WebP, right-sized, `loading=lazy`, `width/height`, `fetchpriority=high` no logo
  e no selo Advertorial (LCP). Variante responsiva `refs-logos-760.webp` + `srcset`.
- Fontes não-bloqueantes (`media=print` swap); pesos enxutos.
- CSS morto do Bootstrap removido (~4KB).
- `content-visibility` **REMOVIDO** (causava seções em branco em webviews — ver §9).

**Cloudflare (zona `steeltrapicvs.com`) — JÁ configurado via API/painel:**
- ✅ HTTP/3, 0-RTT, Early Hints, Brotli, TLS 1.3 (zrt) — ON.
- ✅ `browser_cache_ttl = 0` ("respeitar headers" — deixa nosso `_headers` valer; antes era 4h).
- ✅ Smart Tiered Cache — ON.
- ✅ RUM (Web Analytics / `beacon.min.js`) — **desativado** (era 208KB/348ms redundantes).
- ❌ **Rocket Loader — NUNCA LIGAR** (quebra player VTurb, pixels, revelação do pitch; piora a nota).
- ❌ **Cloudflare Fonts — manter OFF** (testado: piorou — fontes entraram no caminho crítico,
  Speed Index foi a 3,5s).
- `cache_level` = aggressive/Standard (NÃO usar "Ignore Query String" — quebraria o cache-bust `?v=2`).

---

## 8. DADOS DA VSL (análise do dia 1) — leitura honesta

- **Amostra minúscula:** ~25 visitantes humanos reais (os "227 views" são inflados ~8,7×/único
  por bots/probes/testes; desktop = bot, `::TESTE123::` = teste, Brasil = operador testando).
- **Play rate real (EUA): 80%** (saudável). No navegador do TikTok: 100%. → **play rate NÃO é o
  gargalo** (o auto-scroll da §6 ajuda na margem).
- **Engajamento 0,18% (~8s de média assistida)** num vídeo de 77 min → as pessoas dão play e saem
  em segundos. **Esse é o vazamento real.**
- **0% retenção no pitch / 0 cliques / 0 conversões:** esperado com 20 plays (ninguém chega aos
  58 min). 0 clique é **consequência**, não bug.
- **Causas prováveis (fora da landing):** (1) tráfego frio TikTok 50+; (2) hook do vídeo (ativo do
  produtor). **Maior alavanca sugerida:** ligar **autoplay mudo** no VTurb (`smartAutoPlay` está
  `false`) — alinha à expectativa do público de TikTok. É config de playback, não tracking.
- **⚠️ Achado a investigar:** o ratio views/único de ~8,7× + o console mostrando VTurb/UTMify
  **inicializando 7+ vezes** sugere **re-inicialização múltipla por visita** → infla métricas e
  **pode disparar o pixel várias vezes** (desperdício de verba). Verificar: abrir a página 1×,
  ver se "VTurb :: v4.17.0" aparece 1× (ok) ou várias (bug, provavelmente lado produtor/cloaker).

---

## 9. BUGS ENCONTRADOS E CORREÇÕES (aprendizados p/ não repetir)

- **`.toast` do Bootstrap colidia** com nossa classe de notificação (`:not(.show){display:none}`)
  → renomeado tudo para `.lvt*`.
- **`content-visibility:auto`** (classe `.cv-auto`) **deixava seções em branco** em alguns
  Android/webviews (TikTok incluso) — o Lighthouse (Chrome headless) renderizava ok, então
  auditoria nenhuma pegava. **Removido por completo.** (Lição: webview ≠ Chrome headless.)
- **`refs-logos.webp` ficou com fundo preto:** um passe de otimização usou `Image.convert('RGB')`
  que **achata o alpha sobre preto**. **Sempre preservar RGBA** em logos/PNGs com transparência.
  Corrigido restaurando do histórico do git + recomprimindo RGBA. Cache-bust `?v=2` aplicado.
- **`.cbs-deals{display:flex}`** aparecia antes do pitch (sobrescrevia `.esconder`) → flex movido
  para wrapper interno `.cbs-deals__row`.
- **Scripts node em `/tmp` não resolvem módulos** (sharp/jsdom) → rodar a partir do dir do projeto.
- **PIL/openpyxl ausentes** → `pip install Pillow openpyxl` quando precisar processar imagem/xlsx.
- **CORS do `sst`** → provável cosmético (ver §3); verificar no GA4 antes de mexer.

---

## 10. PALETA DE CORES (CBS atual — trocar no MEMOPEZIL)

Variáveis CSS em `:root`:
- `--brand: #cd8900` (dourado CBS — autoridade; header, nav, selo Recommended)
- `--brand-light #e09c12`, `--brand-dark #a66e00`, `--brand-darker #855700`
- `--cta: #15803d` (verde — ação/compra), `--cta-light #1a9e4b`, `--cta-dark #126a32`
- `--trust: #1d4ed8` (azul — confiança), `--urgent: #e0103a` (vermelho — urgência)
- `--ink #15181e`, `--muted #525b69`, `--line #e6e8ec`, fundo `#fff`
- Princípios validados (livro "A Psicologia das Cores"): verde=saúde/ação, azul=confiança,
  vermelho=urgência, preto=autoridade, branco=pureza, amarelo=atenção (preto-sobre-amarelo=
  máxima visibilidade), laranja evitado.
- **Para o MEMOPEZIL:** nova paleta conforme a nova pele/marca, mantendo os mesmos PAPÉIS
  (autoridade/ação/confiança/urgência) e bom contraste (AA) sobre fundo branco.

---

## 11. O QUE TROCAR para o MEMOPEZIL (checklist de variáveis)

| # | Item | Onde no `index.html` (valores atuais p/ referência) |
|---|------|------|
| 1 | **Player VTurb** (id, embed, conta) | `<head>` 3 preloads; bloco do player; script. Atual: id `vid-6a250fac5306f4786ccf8321`, conta `13e4d193-3f66-46ab-8eb0-6e6da4bd133b`, mídia `6a250ee23f9c6e277b158bc3` |
| 2 | **`SECONDS_TO_DISPLAY`** (minuto do pitch do novo vídeo) | linha 1029 — atual `3485` |
| 3 | **Links de checkout** (codename/aff_id/account_id/redirect) | 3 cards (cta-1/2/3) + buy bar. Atual template BuyGoods: `aff_id=2574&account_id=12595&product_codename=neu2/neu3/neu6&redirect=<base64>&subid5=` (manter o `&subid5=` no fim!) |
| 4 | **Preços/economia/kits** | Try Two $79 (save $200, neu2), Most Popular $69 (save $440, neu3), Best Value $49 (save $780, neu6) + buy bar |
| 5 | **Manchete + dek** (ângulo) | `<h1 class="headline-serif">` + citação "Over 15,000 Americans…" |
| 6 | **Imagens dos potes** + supplement facts | `images/2-/3-/6-potes-aff-neuro.webp`, `images/label-facts.webp` |
| 7 | **Reviews/depoimentos** | seção reviews (Pool 3) + nomes nos comentários e chat (Pools 1/2) |
| 8 | **Tracking** | confirmar com operador: mesmos IDs ou novos (GTM/pixel/UTMify) |
| 9 | **Favicon/logo/seals/CBS Deals** | manter CBS? Ou trocar conforme a nova marca |

---

## 12. PENDÊNCIAS (abertas, não urgentes)

1. **Validar `subid5` no clique** (BuyGoods) — nunca houve clique real ainda.
2. **E-mail de suporte real** nas páginas legais — hoje placeholder `support@example.com`.
3. **Revogar o token Cloudflare** que foi exposto no chat (escopo zona; provavelmente já revogado).
4. **Autoplay mudo no VTurb** — avaliar ligar (maior alavanca de engajamento).
5. **Investigar re-init múltiplo** (views 8,7×/único; possível pixel duplicado).
6. **CORS do `sst`** — confirmar no GA4 se é cosmético ou perda real.

---

## 13. FLUXO DE TRABALHO / VALIDAÇÃO (como operar)

```bash
# 1) Balanceamento de HTML (python html.parser) — sem libs externas
# 2) node --check em cada <script> inline extraído
# 3) Conferir que os 4 links de checkout seguem íntegros (com &subid5=)
# 4) Imagens: PIL (pip install Pillow) — PRESERVAR ALPHA (RGBA) em logos/transparências
#    Rodar scripts node/python A PARTIR do dir do projeto (módulos não resolvem em /tmp)
# 5) Commit + push
```
- **Branch:** desenvolver na branch designada da sessão; depois `git branch -f main <branch>` e
  push das duas (a produção do Pages segue a `main`).
- **Push com retry** (rede instável): `git push -u origin <branch>` com backoff 2/4/8/16s.
- **NÃO criar PR** salvo pedido explícito.

---

## 14. DEPLOY DO MEMOPEZIL (paralelo)

1. Repo `Layout-01-MEMOPEZIL` já criado (duplicata do `cbs` — já vem com o motor).
2. Cloudflare → Workers & Pages → criar projeto Pages conectado a esse repo → Framework: None →
   deploy.
3. Custom domains → mapear o **subdomínio novo** (cria DNS sozinho, zona é do operador).
4. `_headers` (veio na cópia) cuida do cache. Confirmar `browser_cache_ttl` respeitando headers,
   Rocket Loader OFF, Cloudflare Fonts OFF (mesmas regras da §7).

---

## 15. DECISÕES JÁ TOMADAS PELO OPERADOR (não re-perguntar)

- Live chat aparece **desde o início** (não esperar clique).
- 1 comentário por vez no live chat overlay.
- Datas em **en-US** (não no idioma do visitante).
- DTC **só no pitch** (sem fallback no-JS que revele cedo).
- **Não mexer em terceiros/tracking** para ganhar PageSpeed.
- Cloudflare Fonts **OFF**, Rocket Loader **OFF**, RUM **OFF**.
- Auto-scroll suave da VSL na carga: **sim** (implementado).
- MEMOPEZIL: **produto diferente**, **paralelo** (novo subdomínio), **reaproveitar o motor mas
  mudar o layout**.

---

## 16. PERGUNTAS EM ABERTO PARA O OPERADOR (antes de construir o MEMOPEZIL)

1. **Direção do layout** (a "pele" nova): outro telejornal/morning show? portal de saúde/revista
   científica? documentário/exposé? ou mesma ossatura com restyle?
2. **Assets do produto:** player VTurb + pitch time; links de checkout (aff_id/account_id/codename/
   redirect); preços/kits; imagens dos potes; supplement facts; reviews (fotos+textos).
3. **Tracking:** mesmos IDs (GTM/pixel/UTMify) ou novos?
4. **Subdomínio** definitivo.
5. **Marca/seals:** manter identidade CBS ou trocar tudo conforme a nova pele?

---

_Fim do handoff. Mantém o motor, troca a pele e os dados, valida sempre, e nunca toca no tracking._
