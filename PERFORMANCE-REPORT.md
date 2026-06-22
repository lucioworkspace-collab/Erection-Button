# RELATÓRIO DE PERFORMANCE — landing advertorial VSL

> Levantamento completo e detalhado de **cada otimização de desempenho** aplicada à página:
> o que foi feito, por quê, o impacto e a justificativa técnica. Inclui também o que foi
> **testado e revertido** e o que está **fora de escopo** (e por quê).
>
> Página: `index.html` (arquivo único, ~127 KB / **~32 KB gzip**). Métricas medidas no
> PageSpeed Insights (mobile, Moto G Power emulado, 4G lento, Lighthouse 13.x).

---

## 0. PADRÕES OBRIGATÓRIOS PARA ALTERAÇÕES FUTURAS

> Toda mudança neste repositório **deve** seguir estes padrões — eles mantêm o desempenho
> no teto **sem comprometer o rastreamento/medição** (regra absoluta do cliente).

**Imagens**
- Servir **WEBP** sempre que possível.
- **Dimensionar para o uso real** (máx. ~3× o tamanho exibido). Nunca subir mockup
  3000×2000 para exibir a ~285px. Cards de frasco → **≤ 900px de largura**.
  Ferramenta: `PIL.Image.resize(..., LANCZOS)` + `save(quality≈85, method=6)`.
- Sempre `loading="lazy" decoding="async"` + `width`/`height` na proporção correta (evita CLS).
- `/images/*` é cache `immutable`: ao **trocar o conteúdo** de um asset que visitantes já
  podem ter cacheado, **renomear** (cache-busting).

**HTML / CSS / JS inline**
- Manter CSS **crítico inline** e **fontes assíncronas** (`media="print" onload`).
- **Não** minificar a ponto de perder a editabilidade — ganho pós-gzip é desprezível e o
  fluxo exige edição cirúrgica legível. Sem frameworks/libs novas (página estática).
- Rodar o **validador de scripts §12 do handoff** (0 erros) após editar qualquer `<script>`.

**Cache / entrega (Cloudflare Workers assets)**
- `_headers`: `/css /images /js` = `max-age=31536000, immutable`; `*.html` = `max-age=0,
  must-revalidate`. Não afrouxar.
- `.assetsignore` deve excluir `.git`, `.wrangler`, `node_modules`, `*.md`, `wrangler.jsonc`.

**🚫 INTOCÁVEL — rastreamento/medição e player**
- **Nunca** quebrar/adiar de forma que comprometa **GTM**, **GA4**, **pixel TikTok**,
  **UTMify**, **sGTM**, a bridge **`__djSidSweep`** ou o **player VTurb**.
- O peso pesado dos relatórios (GTM ~2,8MB, vídeo VTurb ~5,5MB, TikTok ~0,9MB) é
  **third-party** e **fica como está** — não é regressão da página.

**Fora de escopo (não "consertar")**
- `noindex`/SEO baixo (proposital), `alt` do thumbnail (interno do VTurb), TBT de
  terceiros, CSP/HSTS (CSP **arrisca quebrar o tracking**).

**Registro — 2026-06-22 (1):** "Properly size images" do relatório atendido — `forcevital-2/3/6.webp`
redimensionados de **3000×2000 → 900×600** (WEBP q85): **313KB → 97KB (−216KB / −69%)**.
Nenhum outro asset first-party estava superdimensionado.

**Registro — 2026-06-22 (2):** priorização do first paint, sem tocar tracking:
- **Player hints:** removidos os `preload as="script"` de `player.js`/`smartplayer.js` (front-carregavam
  ~290KB disputando com a headline/LCP). Conexão aquecida via **`preconnect`** a `scripts.converteai.net`;
  o player carrega pelo loader in-body (tap-to-play, não precisa para o paint). `main.m3u8` segue em preload.
- **Deferral:** o motor **CBSLive** (chat/reações/toasts/contadores) virou `window.__cbsLiveInit` e roda
  **após o primeiro paint** (`requestIdleCallback` + fallback `setTimeout`), não mais no parse. Corpo do
  motor inalterado; `__djSidSweep` continua no `DOMContentLoaded`+clique.
- ⚠️ **`content-visibility` é proibido aqui** — já testado e revertido (some/branqueia seções em
  webviews Android/in-app, que é o público TikTok). Ver comentário no `index.html`.

---

## 1. RESUMO EXECUTIVO

- **Core Web Vitals reais (estáveis, excelentes):**
  **FCP ~0,9 s · LCP ~1,7 s · CLS ~0,047 · Speed Index ~1,3–1,9 s.**
- **Nota de Desempenho:** chegou a **95**. **Oscila entre ~70 e ~95** entre execuções — e essa
  oscilação é **variância de terceiros**, não regressão da página (provado com relatórios da mesma
  página em horários diferentes: TBT 200 ms vs 790 ms vs 3.950 ms, com o mesmo HTML).
- **Único item vermelho:** **TBT (Total Blocking Time)** — **~98 % é JavaScript de terceiros**
  (GTM, pixel TikTok, player VTurb), que o operador decidiu **não tocar**.
- **Conclusão:** o que está sob nosso controle está **no teto**. O número redondo é refém do peso
  dos pixels; o que o usuário sente (ver e interagir) está em nível de elite.

---

## 2. A JORNADA (scores ao longo do trabalho)

| Momento | Perf | FCP | LCP | TBT | CLS | Speed Index | Observação |
|---|---|---|---|---|---|---|---|
| Inicial | 78 → 87 | 2,9 s | 2,9 s | 200 ms | 0,048 | 2,9 s | antes das otimizações de render/imagem |
| Após CSS inline + imagens | **95** | **0,9 s** | **1,7 s** | 250 ms | 0,047 | 1,9 s | salto principal (FCP 3×) |
| Variância de terceiros | 81 | 1,2 s | 1,7 s | 790 ms | 0,047 | 1,3 s | mesma página; TBT subiu por agendamento dos pixels |
| Regressão Cloudflare Fonts | 69 | 1,2 s | 1,7 s | 3.950 ms | 0,048 | 3,5 s | recurso da CF colocou fontes no caminho crítico → revertido |

> Acessibilidade **96**, Práticas Recomendadas **96–100**, SEO **66** (baixo **de propósito** —
> a página é `noindex`; não é meta deste projeto).

---

## 3. OTIMIZAÇÕES IMPLEMENTADAS (detalhe por detalhe)

### A) Caminho de renderização

**A1. CSS crítico inlinado (eliminação de request render-blocking)**
- **O que:** o `css/app.min.css` (~24 KB) foi **embutido** num `<style>` no `<head>`
  (marcador `Critical CSS inlined`); o `<link>` externo foi removido.
- **Por quê:** CSS externo é **render-blocking** — o navegador não pinta nada até baixá-lo. Pelo
  proxy (steeltrap), o documento já levava ~900 ms; somar mais um round-trip de CSS no caminho
  crítico custava caro.
- **Impacto:** **FCP caiu de 2,9 s para 0,9 s** (≈3×). Foi a maior alavanca isolada.
- **Justificativa técnica:** remove 1 RTT do caminho crítico; o CSS já chega no 1º byte do HTML.
  Custo: +24 KB no HTML (mas o HTML gzipa para ~32 KB total — irrelevante).

**A2. Fontes não-bloqueantes (carregamento assíncrono)**
- **O que:** `<link ... rel="stylesheet" media="print" onload="this.media='all'">` + `<noscript>`
  fallback. Pesos reduzidos ao mínimo usado (Open Sans 400/600/700/800, Roboto 700, Jost 500/700,
  Source Serif 4 700).
- **Por quê:** Google Fonts via `<link>` normal bloqueia a renderização. O truque `media="print"`
  faz o navegador baixar a fonte **sem bloquear**, e troca para `all` quando carrega.
- **Impacto:** fontes saem do caminho crítico; texto pinta imediatamente (fallback do sistema) e
  faz swap. Cortar pesos não usados reduziu disputa de banda na janela crítica.
- **Justificativa técnica:** `display=swap` + carregamento assíncrono = sem FOIT (texto invisível).

### B) Imagens

**B1. Tudo WebP, dimensionado e comprimido**
- **O que:** 36 imagens — **31 WebP, 4 PNG, 1 ICO**. Cada `<img>` com `width`/`height` explícitos,
  `loading="lazy"` (40 ocorrências) e `decoding="async"` (25).
- **Por quê:** WebP é ~30 % menor que JPEG/PNG; `lazy` adia o que está abaixo da dobra; `width/height`
  evita reflow (CLS).
- **Impacto:** payload de imagem minúsculo (**~384 KB a pasta inteira**); economia de centenas de KB
  apontadas pelo PSI resolvidas.

**B2. Right-sizing + variante responsiva (`srcset`)**
- **O que:** avatares reduzidos de 160/200 px → 120 px (uso máx. em tela é 46 px CSS). `refs-logos`
  com variante **760 px** + `srcset`/`sizes` (mobile baixa a menor; desktop a de 1100 px).
- **Por quê:** servir imagem maior que o exibido é desperdício de banda e de decode.
- **Impacto:** resolveu os flags "properly size images" e "responsive images" do PSI.

**B3. LCP priorizado (`fetchpriority="high"`)**
- **O que:** `fetchpriority="high"` no **logo CBS do header** e no **selo Advertorial** (ambos
  acima da dobra).
- **Por quê:** sinaliza ao navegador para buscar esses recursos antes dos demais.
- **Impacto:** pintura mais rápida do conteúdo acima da dobra; ajuda LCP/percepção do 1º segundo.

**B4. Preservação de alpha (correção de bug)**
- **O que:** logos/transparências salvos como **RGBA** (nunca `convert('RGB')`).
- **Por quê:** `convert('RGB')` **achata o alpha sobre preto** → o `refs-logos` ficou com faixa
  preta num device real (Lighthouse headless não pegava). Cache-bust `?v=2` aplicado.
- **Justificativa técnica:** lição de processo — recomprimir imagem com transparência **sempre**
  preservando o canal alpha.

### C) Estabilidade visual (CLS)

**C1. Dimensões + aspect-ratio reservados**
- **O que:** `width`/`height` em todas as imagens; `.vsl-stage vturb-smartplayer { aspect-ratio:16/9 }`
  reserva a caixa do vídeo antes do player carregar; `.img_adv { aspect-ratio: 750/123 }` no selo.
- **Por quê:** elemento sem dimensão "empurra" o layout quando carrega → CLS.
- **Impacto:** **CLS ~0,047** (verde; limite "bom" é < 0,1). Selo Advertorial saiu do flag
  "unsized image".

### D) CPU / thread principal (o que dá para reduzir do nosso lado)

**D1. CSS morto removido**
- **O que:** 15 blocos não usados do Bootstrap (countdown, loading, placeholder, progress, badge,
  card, nav, table + keyframes órfãos) — **~4 KB** a menos de parse. Auditoria por token exato
  confirmou que nenhuma classe viva os usava.
- **Impacto:** menos CSS para analisar/estilizar; pequeno alívio em Style & Layout.

**D2. Animações compositadas (sem repaint por frame)**
- **O que:** o pulso da buy bar passou a animar **`opacity`** num pseudo-elemento (`buybar-pulse`),
  em vez de `box-shadow`.
- **Por quê:** animar `box-shadow` repinta a cada frame (caro); `opacity`/`transform` rodam na GPU,
  sem repaint.
- **Impacto:** limpou o flag "non-composited animations"; rolagem mais suave.

**D3. Números tabulares (estabilidade, não custo)**
- **O que:** `font-variant-numeric: tabular-nums` no timer e contadores ao vivo (3 grupos).
- **Por quê:** dígitos de largura fixa não "empurram" o layout a cada tick (micro-jitter).

### E) Cache

**E1. `_headers` (Cloudflare Pages)**
- **O que:** `/css /images /js` → `public, max-age=31536000, immutable` (1 ano). HTML →
  `max-age=0, must-revalidate`.
- **Por quê:** assets imutáveis em cache longo aceleram visitas repetidas; HTML revalida sempre
  para que updates entrem na hora.
- **E2. Cache-busting:** ao trocar uma imagem já publicada, muda-se a URL para `?v=N` (ex.:
  `refs-logos.webp?v=2`), forçando CDN/navegador a buscar a nova (senão o cache de 1 ano serve a velha).

### F) Hints de conexão (enxutos)

- **O que:** `preload` do player.js/smartplayer.js e do `.m3u8` (recursos críticos do vídeo);
  `preconnect` para fonts.googleapis/gstatic; `dns-prefetch` para cdn/scripts.converteai.
  **Reduzidos para 4** (removidos `images.converteai` e `license.vturb`).
- **Por quê:** o PSI alerta para **>4 preconnects** — conexões em excesso competem por banda.
- **Impacto:** atendeu ao aviso; mantém só as origens realmente críticas.

### G) Meta de robustez

- **`color-scheme: light`** — trava a paleta calibrada (impede inversão de dark-mode que desbotaria
  as cores no mobile).
- **`format-detection: telephone=no`** — impede o iOS de transformar preços/datas em links de
  telefone (azul sublinhado) que sujariam o layout.

---

## 4. CAMADA CLOUDFLARE (zona `steeltrapicvs.com`)

Aplicado via API/painel — impacta entrega, não o código:

| Config | Estado | Justificativa |
|---|---|---|
| HTTP/3 (QUIC) | **ON** | handshake mais rápido em redes móveis (tráfego TikTok) |
| 0-RTT | **ON** | retomada de conexão sem round-trip para quem volta |
| Early Hints | **ON** | edge envia 103 → navegador pré-carrega antes do 200 |
| Brotli | **ON** | compressão melhor que gzip |
| TLS 1.3 (`zrt`) | **ON** | handshake curto + 0-RTT integrado |
| `browser_cache_ttl` | **0 (respeitar headers)** | **deixa o nosso `_headers` valer** (antes a CF forçava 4 h, atropelando) |
| Smart Tiered Cache | **ON** | datacenters buscam cache entre si → TTFB consistente |
| Web Analytics (RUM) | **OFF** | removeu o `beacon.min.js` (208 KB / 348 ms de CPU) redundante |

---

## 5. EXPERIMENTOS TESTADOS E **REVERTIDOS** (com justificativa)

- **Cloudflare Fonts → REVERTIDO.** Prometia servir fontes pelo edge, mas **colocou as fontes no
  caminho crítico de renderização** (cadeia crítica de 14,7 s terminando num `.woff2`). Resultado
  medido: **FCP 0,9 → 1,2 s, Speed Index 1,3 → 3,5 s, nota → 69.** Nossa solução original
  (`media="print"`) já era superior. **Mantido OFF.**
- **`content-visibility: auto` → REMOVIDO.** Economizava ~100–250 ms de layout adiado, mas
  **deixava seções em branco** em alguns Android/webviews (TikTok incluso) — onde o tráfego
  aterrissa. Lighthouse (Chrome headless) renderizava ok, então auditoria nenhuma pegava.
  **Correção visual venceu o micro-ganho.**
- **Rocket Loader → NUNCA LIGAR.** Adia/reordena todo o JS; **quebra player VTurb, GTM, pixels e a
  revelação do pitch**, e **piora** a nota em páginas cheias de terceiros. Regra fixa: OFF.

---

## 6. FORA DE ESCOPO (e por que a nota oscila)

**Terceiros, por decisão do operador, intocáveis** — somam ~95 % do peso e do TBT:
- **GTM/gtag:** ~3,8 MB transferidos, ~5,5 s de CPU, ~800–1.200 ms só de script.
- **Pixel TikTok:** ~1,4 MB.
- **Player VTurb (smartplayer):** ~3,2 MB de JS, dos quais grande parte "não usada", ~660 ms–3,6 s
  de execução.
- **UTMify, Cloudflare beacon (este desativado).**

Por que o TBT (e a nota) **variam tanto** com o mesmo HTML: o TBT mede **bloqueio da thread por JS**.
O agendamento de quando GTM/TikTok/VTurb executam muda a cada carregamento, então o TBT salta entre
~200 ms e ~3.950 ms sem nada mudar na página. Como o TBT pesa ~30 % da nota, o número redondo
oscila junto. **FCP/LCP/CLS — o que depende de nós — permanecem estáveis e verdes.**

**Importante para a conversão:** numa VSL, o usuário **assiste a um vídeo** nos primeiros segundos —
ele **não interage** com a UI nesse momento. O TBT pune no laboratório um cenário que quase não
existe no funil real. A interação que paga a conta (clicar em "Add to cart") acontece **muito depois**
de todos os scripts assentarem, com a thread livre.

---

## 7. COMO MEDIR CORRETAMENTE (recomendação)

1. Rodar o PageSpeed **3×** e olhar a **mediana** — não um teste isolado (variância de terceiros).
2. Olhar **FCP / LCP / CLS**, não só a nota composta. Se continuarem **~0,9 s / ~1,7 s / ~0,047**,
   a página está perfeita, independentemente do que o TBT dos pixels fizer com o número.
3. Garantir que o teste pegou a **versão nova** (cache do proxy): conferir, p.ex., se `refs-logos`
   aparece com `width="1100"` (novo) e não `3291` (antigo). Purge no Cloudflare se necessário.

---

## 8. SÍNTESE — "elevar acima de 90"

O caminho técnico do **nosso lado** para >90 **já foi executado por completo**: CSS inline, fontes
assíncronas, imagens WebP/right-sized/responsivas, CLS reservado, CSS morto removido, animações
compositadas, cache afinado, hints enxutos, Cloudflare otimizado, regressões revertidas. Com isso a
página **atingiu 95** e entrega CWV de elite de forma **estável**.

A barreira para **garantir** >90 em **todo** teste é **exclusivamente** o JavaScript de terceiros
(GTM/TikTok/VTurb), que está **fora de escopo por decisão de negócio** (os sinais de pixel valem mais
que a nota de laboratório). As únicas alavancas restantes seriam: enxugar o container do GTM,
adiar pixels até a interação, ou reduzir o JS do player — todas no terreno dos terceiros/produtor.
Enquanto eles permanecerem como estão, **a faixa realista é ~80–95, com FCP/LCP/CLS sempre verdes.**
