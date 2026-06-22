# Cloudflare Deploy — Diagnóstico & Correção

> Runbook do problema "publiquei mas o site não atualiza".
> Não é servido (`*.md` está no `.assetsignore`).
> Worker: **`erection-button`** · URL: `https://erection-button.lucioworkspace.workers.dev`
> Repo: `lucioworkspace-collab/Erection-Button` · branch dev: `claude/pensive-archimedes-sci3sj`

---

## 0. Re-verificação (2026-06-22, sessão `claude/intelligent-maxwell-q184f4`)

Re-testei a integração ponta a ponta. **Veredito: conecta, mas o build não promove a produção.**

**Mudou desde o §3 (boa notícia):**
- ✅ **O token agora TEM acesso a Workers.** `GET /accounts/9ce1202…/workers/scripts` → **200**;
  `wrangler whoami` lista a conta; `wrangler deploy --dry-run` leu os 113 assets sem erro.
  A afirmação do §3.1 ("token só Zone / `code 10000`") está **DESATUALIZADA**.
  (O `401` em `/user/tokens/verify` é esperado: este é um **token de conta**, não de usuário —
  esse endpoint valida tokens de usuário.)

**Continua igual:**
- ❌ **Egress bloqueado:** `GET …workers.dev/` → **403 `x-deny-reason: host_not_allowed`**.
  Não dá para conferir os bytes servidos a partir do sandbox.
- ❌ **MCP só-leitura:** sem ferramenta de deploy/promote.

**Estado da implantação (via API, confirmado):**

| | versão | id | criada (UTC) | alias | promovida? |
|---|---|---|---|---|---|
| **Produção ATIVA** | **#9** | `8044ea31` | 02:34 | — | ✅ última `deployment` real |
| build `main` + recente | #14 | `3dcd704f` | 03:04 | `main` | ❌ só `version_upload` |
| build `main` | #12 | `405ccfec` | 02:47 | `main` | ❌ só `version_upload` |

→ Confirma o §2: produção parou na **#9 (02:34)**; os builds `main` **#12** e **#14** subiram como
*versão* mas **nunca viraram implantação**. O "Deploy command" do build segue em `versions upload`.

**Correção permanente (§4B) = AÇÃO ESCOLHIDA.** Verifiquei que ela **só** existe no painel:
não há rota de API para o build config (`/accounts/…/workers/builds/configs` → `7000 No route`)
e **não há** workflow em `.github/` no repo. Logo, o toggle é manual no dashboard (passos no §4B).

### Atualização (2026-06-22 ~03:32 UTC) — testado + produção promovida na mão
- **Teste decisivo:** um commit vazio em `main` (`89ba278`) disparou build de produção → criou
  **versão #16** (`bfdb39da`, alias `main`) como **`version_upload`**, e as implantações
  **continuaram em 4**. Prova direta: o "Deploy command" do build **ainda é `versions upload`**
  (a correção do §4B **não** foi aplicada/efetivada).
- **Token TEM escrita (novo):** `wrangler versions deploy bfdb39da@100%` → **SUCCESS**.
  Implantações **4 → 5**; ativa agora = **#16 `bfdb39da` @100%** (era #9 `8044ea31`, 02:34).
  Ou seja, o §4C ("liberar token") já está atendido — dá para publicar/promover daqui.
- **Ainda pendente:** a correção **permanente** (§4B — trocar o deploy command no painel)
  continua necessária; sem ela, o próximo push em `main` volta a só subir versão sem promover.
- Egress ao site segue bloqueado → não dá para conferir os bytes servidos do sandbox.

---

## 1. Sintoma
Os assets novos (frascos **ForceVital** + faixa de pagamentos **DACH**) foram commitados,
o `main` recebeu o push e o **build rodou**, mas a página em produção continua mostrando o
conteúdo antigo (frascos **Lipo Jaro** / cartões US).

## 2. Causa-raiz (confirmada)
**O build sobe a *versão* mas não a *promove* a produção.** No Cloudflare Workers existem
duas etapas:

| Comando | O que faz |
|---|---|
| `wrangler versions upload` | cria uma **versão** (preview), **não** ativa em produção |
| `wrangler deploy` | cria a versão **e** a torna a **implantação ativa** (produção) |

O "Deploy command" do build deste worker está em **`versions upload`** (ou a branch de
produção não está mapeada para `main`). Resultado: cada push cria uma versão nova na aba
**Versões**, mas a **Implantação ativa** nunca troca.

**Evidência (painel "Versões"):**
- `f7791adb` — *Swap bottle + payment assets to new WEBP renders* (= commit `f7e6176`)
- `bef45d75` — *Point Worker config… (erection-button)* (build da branch dev)
- `ecdaba03` — *Point Worker config… (erection-button)* (build do `main`, **versão mais recente e completa**)

**Evidência (API `workers_list`, `modified_on` em UTC, 2026-06-22):**
`02:16:07` → `02:22:47` → `02:34:02`. O worker recebe atividade a cada build/promoção,
mas `modified_on` sobe tanto em *upload* quanto em *deploy* — então **não** prova, sozinho,
que produção foi promovida. (O worker errado, `bariatric-seed-wl`, segue com data de
**2026-06-21** → nenhum deploy foi parar nele. ✅)

## 3. Por que o Claude **não** consegue publicar do sandbox
Três bloqueios independentes (todos verificados nesta sessão):

1. **Token sem permissão de Workers.** ⚠️ **DESATUALIZADO — ver §0** (em 2026-06-22 o token
   já lê Workers e o `wrangler deploy --dry-run` funciona). Texto original abaixo:
   `CLOUDFLARE_API_TOKEN` do ambiente é válido
   (`/user/tokens/verify` = `200 active`), mas qualquer endpoint de Workers retorna
   **`code 10000 Authentication error`**. `wrangler whoami` falha em listar a conta. O
   token só tem escopo de **Zone** (Read/Settings) — como o `handoff.md §4` já avisava.
2. **MCP da Cloudflare é só-leitura.** Tem `workers_list` / `workers_get_worker`, mas
   **nenhuma** ferramenta de deploy/promote/versions. `workers_get_worker_code` retorna
   nulo porque é um worker **só-assets** (sem código de servidor).
3. **Egress bloqueado.** Requisições a `*.lucioworkspace.workers.dev` voltam
   `403 "Host not in allowlist"` (proxy do sandbox, sem `cf-ray`) → não dá nem para abrir
   o site e verificar os bytes.

**Conta Cloudflare:** `9ce1202873000e1a318b07390c182493` (mesma que hospeda
`erection-button` e `bariatric-seed-wl`).

---

## 4. Como corrigir

### A) Publicar AGORA (1 vez, no painel) — mais rápido
Workers e Pages → **erection-button** → **Implantações** → botão **"Nova implantação"** →
selecionar a versão **`ecdaba03`** → **100%** → confirmar. O site atualiza na hora.

> Se o build mais recente estiver **FAILED** com *"build token no longer valid"*:
> Configurações → **Compilações** → renovar o **API token** do build → **Retry**
> (procedimento do `handoff.md §2`).

### B) Auto-deploy a cada push (permanente) — recomendado
Workers e Pages → erection-button → **Configurações** → **Compilações (Builds)**:
- **Deploy command:** trocar `npx wrangler versions upload` → **`npx wrangler deploy`**
- **Branch de produção:** garantir que é **`main`**

Depois disso, todo `git push …:main` publica sozinho. Avise que eu faço um push trivial e
**confirmo via MCP** que a versão virou ativa.

### C) Delegar a publicação ao Claude — escolha 1 dos 2
- **Liberar egress:** adicionar `erection-button.lucioworkspace.workers.dev` ao
  *network egress allowlist* do ambiente → eu confirmo o conteúdo ao vivo (tamanhos de
  bytes / marcadores do HTML) imediatamente.
- **Liberar token:** adicionar **`Workers Scripts: Edit`** (+ **Workers Assets**) ao
  `CLOUDFLARE_API_TOKEN` da conta `9ce1202…` → eu publico direto daqui com
  `npx wrangler deploy` (o `wrangler.jsonc` já aponta `name: "erection-button"`).

---

## 5. ⚠️ Cache pode mascarar o deploy (verifique isto primeiro!)
`_headers` põe `/images/*` em **`max-age=31536000, immutable`** e os assets novos
**reusam os mesmos nomes** (`lipojaro-2/3/6.webp`, `card.webp`). Logo, **um navegador que
já visitou o site continua exibindo as imagens antigas** mesmo com o deploy correto —
"immutable" instrui o cache a **não revalidar**. O HTML não tem esse problema
(`max-age=0, must-revalidate`).

**Para conferir de verdade:** abra em **aba anônima** ou dê **hard-refresh
(Ctrl/Cmd + Shift + R)**.

**Conserto definitivo (opcional):** *cache-busting* — renomear os assets para
`forcevital-*.webp` (e a faixa para `payments.webp`) e atualizar as `src`, **ou** anexar
`?v=2`. Aí até visitantes recorrentes recebem o novo na hora. (Hoje, como o tráfego é
majoritariamente novo do TikTok, o impacto é pequeno — mas isto elimina o sintoma.)

---

## 6. Checklist de verificação
Marcar **na aba anônima** / após hard-refresh:

- [ ] Frascos: **pretos**, "ForceVital · 30 KAPSELN" (não branco/amarelo "LIPO JARO").
- [ ] Faixa de pagamento sob os botões: **Visa · Mastercard · PayPal · Klarna · Apple Pay · G Pay**
      (não MasterCard/Visa/Amex/Discover).
- [ ] (Se eu tiver egress) `GET /images/lipojaro-6.webp` → `content-length = 35846`
      (nova) e **não** `27718` (antiga); `card.webp` = `3248` (nova) e não `11128`.
- [ ] (Se eu tiver egress) HTML contém `width="505" height="48"` e **não** `width="425" height="74"`.
- [ ] Painel → Implantações → **ativa = `ecdaba03`**.

---

## 7. Estado do código (já pronto, aguardando só a promoção)
- `origin/main` = **`33f2e0a`** (inclui a troca de assets `f7e6176` + correção do nome).
- `wrangler.jsonc` → `name: "erection-button"` ✅ (antes era `bariatric-seed-wl`, herdado do template).
- Validador de scripts inline: **12 scripts, 0 erros**.
- **Nada pendente no repositório.** O bloqueio é 100% na etapa de **promoção a produção**,
  fora do alcance do sandbox.
