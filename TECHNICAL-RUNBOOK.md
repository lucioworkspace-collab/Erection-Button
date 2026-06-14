# TECHNICAL RUNBOOK — ambiente, ferramentas, validação e workflow

> Levantamento **técnico/factual** do que foi processado nesta janela e de COMO foi feito.
> Objetivo: remover fricção **técnica** (ambiente, tooling, validação, git) numa nova sessão.
> Acompanha o `PROJECT-HANDOFF.md` (que cobre o "o quê/por quê" do produto). Este cobre o "como".
>
> Escopo: procedimentos de engenharia. Não substitui o julgamento sobre conteúdo.

---

## 1. AMBIENTE (e seus limites reais)

- Execução **remota gerenciada**, container efêmero. O repositório é clonado limpo no início;
  **só o que está commitado e pushado sobrevive.**
- **Rede com allowlist restrita.** Vários hosts externos respondem `Host not in allowlist`:
  - `scripts.converteai.net` / `cdn.converteai.net` (bundle do player VTurb) → **não dá para baixar**.
  - `api.cloudflare.com` → bloqueado salvo se a sessão tiver o host liberado na política de rede.
  - O **próprio domínio ao vivo** (`*.steeltrapicvs.com`) → bloqueado. **Não dá para bater na
    landing publicada** a partir da sessão (nem curl, nem WebFetch).
  - **Consequência prática:** validação é feita **localmente** (parser/seu node), não pela página
    no ar. Quem testa a página ao vivo é o operador, no navegador real.
- **GitHub:** a sessão é **escopada a UM repositório**. Tentar ler/escrever outro repo →
  `Access denied ... Allowed repositories: ...`. Para trabalhar noutro repo, a sessão precisa ser
  criada/escopada nele (não dá para ampliar de dentro).

---

## 2. FERRAMENTAS (instalação e pegadinhas)

- **Python `html.parser`** (stdlib, sem instalar) → validação de balanceamento de tags.
- **`node --check`** → checagem de sintaxe dos `<script>` inline extraídos.
- **Pillow (PIL):** `pip install Pillow` (avisa "running as root" — inofensivo). Para imagens.
- **openpyxl:** `pip install openpyxl` (ler `.xlsx`, ex.: métricas da VSL).
- **poppler-utils** (`pdftotext`, `pdftoppm`) → ler PDFs (ex.: relatórios PageSpeed). Instalar via
  `apt-get install -y poppler-utils` se necessário. (pdfminer/pypdf quebram aqui — usar poppler.)
- **PEGADINHA CRÍTICA:** scripts node/python que importam módulos (sharp, jsdom) **não resolvem
  `node_modules` se rodados a partir de `/tmp`**. **Rode-os a partir do diretório do projeto.**
- **Remover `node_modules` antes de commit** se algum for instalado.

---

## 3. VALIDAÇÃO (sempre antes do push)

### 3.1 Balanceamento de HTML (sem libs externas)
```python
from html.parser import HTMLParser
src = open('index.html', encoding='utf-8').read()
VOID = {'area','base','br','col','embed','hr','img','input','link','meta','param','source','track','wbr'}
class P(HTMLParser):
    def __init__(s): super().__init__(convert_charrefs=True); s.stack=[]; s.err=[]
    def handle_starttag(s,t,a):
        if t in VOID: return
        s.stack.append((t, s.getpos()))
    def handle_endtag(s,t):
        if t in VOID: return
        for i in range(len(s.stack)-1,-1,-1):
            if s.stack[i][0]==t:
                if i!=len(s.stack)-1: s.err.append(('unclosed', t))
                del s.stack[i:]; return
        s.err.append(('stray', t))
p=P(); p.feed(src)
print("HTML:", "OK" if not p.stack and not p.err else (p.stack[-3:], p.err[:4]))
```

### 3.2 Sintaxe dos scripts inline
```python
import re
src = open('index.html', encoding='utf-8').read()
for i, s in enumerate(re.findall(r'<script(?![^>]*src=)[^>]*>(.*?)</script>', src, re.S)):
    open(f'/tmp/s{i}.js','w').write(s)
# depois: for f in /tmp/s*.js; do node --check "$f"; done
```

### 3.3 Integridade dos links de checkout (afiliado — não podem quebrar)
```python
import re
links = re.findall(r'href="(https://buygoods\.com[^"]+)"', src)
# conferir: nº esperado de links, e que cada um termina com &subid5= (placeholder do VTurb)
```

---

## 4. OTIMIZAÇÃO DE IMAGEM (procedimento + pegadinha do alpha)

```python
from PIL import Image
# Right-size + WebP. PRESERVAR ALPHA em logos/PNGs transparentes:
im = Image.open('images/logo.webp').convert('RGBA')   # NUNCA 'RGB' em algo com transparência:
                                                       # convert('RGB') achata o alpha sobre PRETO
im.save('images/logo.webp', 'WEBP', quality=70, method=6)
```
- Conferir o resultado visualmente (ferramenta Read renderiza a imagem).
- Dimensionar para ~2× o tamanho exibido em CSS (cobre retina) — não maior.
- **Cache-bust** quando trocar uma imagem já publicada (cache de 1 ano imutável): mudar a URL para
  `arquivo.webp?v=2` no HTML, senão CDN/navegador servem a versão antiga.
- Responsivo: gerar variante menor + `srcset`/`sizes` quando o PageSpeed apontar "responsive images".

---

## 5. GIT / WORKFLOW

```bash
# desenvolver na branch da sessão; produção do Cloudflare Pages segue a main
git add -A
git commit -q -m "mensagem clara"
git push -u origin <branch>            # com retry/backoff 2/4/8/16s se a rede falhar
git branch -f main <branch>
git push origin main
```
- **NÃO criar Pull Request** salvo pedido explícito.
- Commits descritivos; uma mudança lógica por commit quando possível.
- Mensagem de commit termina com o link de sessão exigido pelo harness (o ambiente injeta).

---

## 6. ABORDAGEM DE EDIÇÃO DO `index.html`

- **Arquivo único** (~2056 linhas). **CSS está inlinado** num `<style>` (foi `css/app.min.css`).
- Editar com substituições exatas (ler o arquivo antes). Casar indentação/idioma do entorno.
- Há **2 blocos `<style>`** (o utilitário Bootstrap/Tailwind minificado + o custom) e vários
  `<script>` inline. Ao mexer em CSS custom, é o 2º bloco / as regras `.cbs-*`, `.vsl-*`, etc.

---

## 7. PEGADINHAS JÁ VIVIDAS (não repetir)

| Sintoma | Causa | Correção |
|---|---|---|
| Seção some/branco em webview (mas ok no Chrome) | `content-visibility:auto` | removido por completo |
| Logo/barra com **fundo preto** | `Image.convert('RGB')` achatou alpha | usar **RGBA** sempre |
| Notificação não aparece | classe `.toast` colidiu com Bootstrap | renomeado p/ `.lvt*` |
| Bloco `.esconder` aparece antes do pitch | `display:flex` sobrescreveu | flex em wrapper interno |
| `MODULE_NOT_FOUND` em script node | rodado de `/tmp` | rodar do dir do projeto |
| `pip`/lib ausente | container limpo | `pip install Pillow openpyxl` |
| Tentar `curl` no domínio/CDN | allowlist de rede | validar local; operador testa no ar |

---

## 8. LIMITES DURAS QUE PERMANECEM (independem de runbook)

Estas **não** são fricção técnica a "destravar" — são regras do projeto:
- **Não alterar tracking** (GTM/pixel/UTMify/sst/subid2/bridge subid5).
- **Afiliado:** não inventar preços/oferta/links — só o que o novo produto exige, com dados do operador.
- **DTC só no pitch** (`.esconder` + `displayHiddenElements`).
- **Congruência** dos 3 pools de identidade (chat ≠ comentários ≠ reviews).
- **Validar sempre** antes do push.

---

_Este runbook cobre o COMO técnico. Para o O QUÊ/PORQUÊ do produto e o estado do projeto,
ver `PROJECT-HANDOFF.md`._
