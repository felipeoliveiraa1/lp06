# Playbook: LP de funil (VSL) de alta performance — Astro + Vercel

> Handoff para outro chat construir uma landing page de funil (VSL + upsells/downsells)
> com **nota verde no PageSpeed** e **connect rate alto**, evitando os erros que já cometemos.
> Stack validada num projeto real (4 domínios no ar). Leia inteiro antes de codar.

---

## 0. Contexto / objetivo

Reconstruir um funil que estava em **WordPress/Elementor** (lento, connect rate ~40%) como
**site estático Astro** na Vercel. Resultado: Lighthouse mobile **99** local, ~**88–95 real**,
CLS 0, e o vídeo (VSL) só pesa quando o usuário interage.

**Regras de ouro:**
1. Tudo estático (`output: 'static'`) → Vercel serve do CDN (TTFB baixíssimo).
2. O que pesa de verdade é a **VSL (player vturb) e o tracking**. Todo o resto é barato.
3. A nota é só um *proxy*. O objetivo real é **carregar rápido + disparar o pixel** (connect rate).

---

## 1. Stack & estrutura

- **Astro** `output: 'static'`, build `npm run build` → `dist/`. Rotas: `src/pages/foo.astro` → `/foo/`
  (directory format = barra no final). Componentes `.astro`, `is:inline` para scripts crus
  (Astro processa/bundla `<script>` sem `is:inline` — quase sempre você quer `is:inline`).
- **Layouts:** um para a HOME (`Base.astro`) e um para as páginas de funil (`Funnel.astro`).
  Isso permite tracking/preloads diferentes entre home (tráfego de ads) e upsell (pós-compra).
- **Componentes reutilizáveis:** `Vsl.astro` (facade do player), `PaytButton.astro` (botão one-click).
- **CSS:** um `global.css` com tokens (`--green`, `--red`, fontes) + classes. Importado pelos layouts.
- **Imagens:** tudo `.webp` (converter com `cwebp -q 80`). Self-host das fontes em `.woff2`.

---

## 2. Performance — o que de fato move a nota

### 2.1 VSL = FACADE tap-to-play (NÃO o player nativo no load) ⭐ o aprendizado nº1
O player do vturb/converteai, quando embedado direto, **auto-bufferiza ~12 segmentos de 720p
no carregamento** (testado: baixa `segment_0..11` sozinho). Isso destrói LCP/SI/connect rate.

**Solução:** facade.
- Um `<div class="vsl-slot">` com **aspect-ratio fixo** (ex: `9/16`) → reserva o espaço → **CLS 0**.
- Dentro: um `<img>` poster (o 1º frame real do vídeo) + um overlay "CLIQUE PARA OUVIR".
- Um script que, **no toque**, injeta o `<vturb-smartplayer>` + o `player.js`. Só aí o vídeo carrega.
- Resultado: load limpo (nota 99) e o vídeo pesado só entra quando o usuário decide ver.
- Bônus: bate com o UX de VSL ("clique para ouvir/assistir").

> Se o dono EXIGIR autoplay nativo: dá, mas a nota cai (o vídeo baixa no load). Avise o trade-off
> e meça. Nesse caso o gate (seção 4) conta a partir do **load**, não do toque.

### 2.2 Poster real via ffmpeg
O thumbnail oficial do vturb é minúsculo/borrado. Extraia o **1º frame** do vídeo:
```bash
ffmpeg -y -i "https://cdn.converteai.net/<ACCOUNT>/<VIDEO_ID>/main.m3u8" \
  -ss 00:00:00.5 -frames:v 1 /tmp/frame.png
cwebp -q 82 /tmp/frame.png -o public/assets/poster-xxx.webp
```
Esse poster é o **LCP** → `preload as=image fetchpriority=high`. Pinta na hora.

### 2.3 CLS = 0 (sempre)
- Reserve TODA imagem: `width`/`height` ou `aspect-ratio` + `object-fit:cover; object-position:center`.
- O box da VSL com `aspect-ratio` evita o "pulo" quando o player injeta.
- Fontes preloadadas (as usadas acima da dobra) reduzem o reflow do swap.

### 2.4 Animações = só `transform`/`opacity`
Botões pulsantes, anéis de "radar" etc.: anime **só transform e opacity** (compositadas, rodam na
GPU, fora da main thread). **Nunca** anime `width`, `height`, `box-shadow`, `top/left`, `background`
(forçam paint/layout → arranham SI e podem dar CLS). `will-change:transform`+`translateZ(0)` promovem
a camada. Sempre inclua `@media (prefers-reduced-motion:reduce){…}`.

### 2.5 Preloads na medida
- Preload da imagem LCP (poster/hero) com `fetchpriority=high`. ✅
- **NÃO** dê preload de alta prioridade ao `gtm.js`/scripts de tracking — eles disputam o slot de
  rede do LCP e atrasam a imagem. `preconnect`/`dns-prefetch` bastam.
- `preconnect` para os domínios da VSL: `scripts.converteai.net`, `cdn.converteai.net`,
  `images.converteai.net`; `dns-prefetch` para `license.vturb.com`.

---

## 3. Tracking (GTM / Pixel) — o trade-off que mais gera discussão

Dois modos, **escolha consciente**:

| Modo | Como | Nota | Connect rate |
|---|---|---|---|
| **Diferido** | `dataLayer` na hora; `gtm.js` carrega no 1º gesto (scroll/touch/play) + fallback ~6s | ~99 | bom (numa VSL o usuário interage em 1-2s) |
| **Eager** | snippet padrão da GTM no `<head>`, dispara no load | ~88–93 | melhor (PageView na hora) |

- Numa página de VSL, **diferido quase não perde connect rate** (o usuário toca play/rola rápido; o
  fallback pega quem não interage). É o que mantém os 99.
- O pessoal de tracking costuma pedir **eager** ("dispara assim que a tela carrega") porque diferir
  pode perder o PageView de quem dá bounce em <1s. É legítimo — **deixe o dono decidir** e seja
  transparente: não dá pra ter "dispara no load via GTM" **e** 99 ao mesmo tempo.
- **Meio-termo possível:** beacon leve de PageView (img/`facebook.com/tr`) no load + GTM diferida —
  mas precisa de dedupe e alinhamento com o tracking. Só com o aval do responsável.

**Pixel só na GTM:** se o cliente "usa só GTM", o pixel do Meta precisa ser uma **tag dentro do
container** (PageView, gatilho All Pages, **publicada**). Se o container só tem GA4, o pixel **não
dispara** — não é a página, é a config da GTM (trabalho do tracking).

**Tracking bloqueado:** GTM/pixel é client-side → ad-blockers e navegadores iOS (Brave etc.) bloqueiam.
No Network: **X vermelho + "Provisional headers" + 0 kB = bloqueado no aparelho** (não é erro de
servidor — esse mostraria response headers/código). Sempre uma fatia se perde. A solução real é
**server-side (Conversions API / GTM server-side)** — projeto à parte.

**Validar tags:** a extensão "Tag Assistant" legada é furada (mostra "nenhuma tag" mesmo disparando).
Use o **Preview/Visualizar de dentro da GTM** (tagassistant.google.com via o botão Preview do container).

---

## 4. O "gate" da VSL (revelar a oferta no pitch)

Esconder o CTA + o conteúdo abaixo até X min do vídeo (o momento do pitch).

- CSS: `html.gated .gate{display:none !important}`. Envolva o que esconde em `<div class="gate">`.
- **Script "early-hide"** (no topo do body, ANTES dos elementos): adiciona `gated` ao `<html>` antes
  da pintura (sem flash). Pula se **não houver JS** (fallback) ou se a URL tiver `?preview` (QA).
  ```html
  <script is:inline>try{if(!localStorage.getItem('KEY')&&!/[?&]preview/.test(location.search))
  document.documentElement.classList.add('gated')}catch(e){document.documentElement.classList.add('gated')}</script>
  ```
- **Script "reveal"** (fim do body): timer que remove `gated` + grava `localStorage[KEY]='1'`
  (persiste — reload não esconde de novo).
- ⚠️ **Robustez a 2º plano:** um `setTimeout(reveal, 14*60*1000)` é **estrangulado quando a aba vai
  pra segundo plano** (celular trava a tela / troca de app) → o reveal nunca dispara. Ancore em
  **`Date.now()`** + `setInterval(check, 1000)` + `visibilitychange`/`focus`:
  ```js
  var started=0, iv=null;
  function reveal(){document.documentElement.classList.remove('gated');try{localStorage.setItem(KEY,'1')}catch(e){}if(iv){clearInterval(iv);iv=null}}
  function check(){if(started && Date.now()-started>=DELAY) reveal()}
  try{if(/[?&]preview/.test(location.search)){document.documentElement.classList.remove('gated');return}}catch(e){}
  try{if(localStorage.getItem(KEY)){reveal();return}}catch(e){}
  window.__startVslTimer=function(){if(started)return;started=Date.now();setTimeout(reveal,DELAY);
    iv=setInterval(check,1000);document.addEventListener('visibilitychange',check);window.addEventListener('focus',check)}
  ```
- **Quando começa:** no FACADE → chame `__startVslTimer()` no toque do play. No autoplay nativo →
  chame no load. (Guarda `if(window.__startVslTimer)` no facade pra não quebrar páginas sem gate.)
- **`?preview`** revela tudo na hora pra QA, **sem persistir** (não grava o flag) — assim o testador
  valida o funil em segundos sem cronometrar, e o gate real continua valendo depois.
- O tempo de cada página é o do **pitch daquele vídeo** (ex: home 13:55, upsell 4:14, downsell 1:30).

---

## 5. vturb / converteai — armadilhas

- **3 ids diferentes:** `account` (ex: `d9544c5d-…`), `playerId` (o wrapper) e `videoId` (o vídeo).
  - player.js: `scripts.converteai.net/<account>/players/<playerId>/v4/player.js`
  - vídeo (m3u8): `cdn.converteai.net/<account>/<videoId>/main.m3u8`
- ⭐ **BUG que já nos pegou:** o id do ELEMENTO tem prefixo `vid-`, mas a URL do `player.js` usa o id
  **PURO** (sem `vid-`). Se você montar a URL com o prefixo → `players/vid-XXX/…` → 404 →
  **"Vídeo não encontrado, contate o suporte da VTurb"**. Sempre faça `playerId.replace(/^vid-/,'')`
  pra montar a URL.
- Mapear player→vídeo: carregue o player isolado numa página de teste e capture a requisição
  `…/main.m3u8` no Network.
- **"Vídeo não encontrado"** também pode ser: vídeo em migração (transiente), domínio não autorizado,
  ou o bug do `vid-` acima. Teste o player isolado pra isolar a causa.
- Vários domínios do mesmo cliente costumam **compartilhar conta + vídeos**, mudando só o **player
  (wrapper)**. Nesse caso os **posters são idênticos** entre domínios (mesmo vídeo).

---

## 6. Payt — botão one-click (caminho do dinheiro, não erre)

Markup exato do botão de aceite (upsell/downsell):
```html
<a href="#" payt_action="oneclick_buy" data-object="HASH-DO-PRODUTO">TEXTO</a>
<select payt_element="installment" data-object="HASH-DO-PRODUTO" style="display:none"></select>
<!-- UMA vez por página, antes do </body>: -->
<script src="https://checkout.payt.com.br/multiple-oneclickbuyscript/XXXXX.js"></script>
```
- O script varre a página por `payt_action="oneclick_buy"` e cobra o produto do `data-object`. O
  `<select installment>` oculto é onde ele injeta as parcelas. **Cada página tem seu `data-object`.**
- Precisa do parâmetro **`?_o=<pedido>`** na URL (o Payt injeta ao redirecionar do checkout). Aberta
  direto dá `Uncaught parametro _o inexistente na url` no console — **normal**, funciona na esteira real.
- O botão pode ficar dentro de um `.gate` (display:none): o script ainda acha por `querySelectorAll`
  e amarra; ao revelar, funciona.
- **Checkout da home** (link normal): `?split=12` no fim já puxa o parcelamento em 12x.

---

## 7. Medição & expectativa (não se engane com 1 teste)

- **Lighthouse local é otimista** (máquina rápida, throttling simulado) → quase sempre ~99. O
  **PageSpeed Insights real** (servidores do Google) dá mais baixo (~88–95).
- **PageSpeed roda 1 vez e varia ±15 pontos** (LCP é o mais instável). Rode **3–4x e pegue a mediana**.
  Se FCP e LCP sobem JUNTOS de um teste pro outro, foi um run de rede/servidor lento — não o código.
- A verdade é o **field data (CrUX)** que o PageSpeed mostra (usuários reais, 28 dias) + **GA4**.
- **Não** instale `@vercel/speed-insights` num funil de ads: é mais script pra carregar e você já tem
  o field data de graça (CrUX/GA4).

---

## 8. Erros que já cometemos (NÃO repita)

1. **Player nativo no load** achando que seria rápido → vídeo baixa sozinho → nota despenca. Use facade.
2. **Prefixo `vid-` na URL do player.js** → "vídeo não encontrado" em todas as VSLs. Strip do `vid-`.
3. **GTM eager + preload de alta prioridade** → LCP disputa rede com o gate → nota cai. Sem preload da GTM.
4. **Trocar a imagem errada:** confirme com o cliente EXATAMENTE qual elemento (ele apontou um card de
   "Para Quem É" mas queria trocar a imagem do "Passo 4"). Renderize e mostre antes de assumir.
5. **Confundir o tempo de cada gate** (coloquei o tempo da home num upsell). Sempre confira qual página
   recebe qual tempo.
6. **setTimeout longo sem Date.now** → no celular em 2º plano o reveal nunca dispara.
7. **Esquecer que a nota lab ≠ real** → prometer 99 e o cliente ver 88. Sempre alinhe a expectativa.

---

## 9. Funil em vários domínios (clonagem)

Mesmo offer em N domínios (escalar ads / sobreviver a ban de domínio):
- Crie um repo por domínio (`gh repo create <nome> --private`), cópia do repo base.
- **Por domínio, troque:** os 4 players da VSL; os `data-object` do Payt; o checkout; o GTM/pixel; e
  o domínio (na Vercel). Os **links de recusa são relativos** (`/downsell-payt/`) → funcionam em
  qualquer domínio sem mexer.
- Se conta+vídeos forem compartilhados, só os **player wrappers** mudam (e os posters ficam iguais).
- Técnica limpa: branch a partir do repo base, troca só os players, `git push branch:main` pro repo do
  domínio, volta pro main. (Evita poluir o repo base com os ids do outro domínio.)

---

## 10. Deploy (Vercel)

1. `gh repo create owner/nome --private` → `git push <url> main`.
2. Vercel → Import Git Repository → detecta Astro (build `npm run build`, output `dist`) → Deploy.
3. Settings → Domains → adiciona o domínio → cria o CNAME/A (`76.76.21.21`) ou troca os nameservers.
4. `vercel.json` simples; nada de SSR (é estático).

---

## 11. Estrutura de arquivos (referência)

```
src/
  layouts/
    Base.astro      # layout da HOME (head: tracking, preloads, fontes, favicon)
    Funnel.astro    # layout dos upsell/downsell (mais enxuto; só o poster da página)
  components/
    Vsl.astro       # facade do player: poster + overlay "CLIQUE PARA OUVIR" + injeta player no toque
                    #   props: playerId (com vid-), poster, account. Monta player.js SEM o vid-.
    PaytButton.astro# <a payt_action=oneclick_buy data-object> + <select installment>
  pages/
    index.astro             # HOME (usa Base) — VSL inline + gate 13:55 + CTAs (checkout ?split=12)
    upsell-01-payt.astro    # usa Funnel + Vsl + PaytButton + gate
    downsell-payt.astro
    downsell-2-payt.astro   # sem VSL
    priority-pass-2.astro
    priority-pass-downsell.astro  # sem VSL
  styles/global.css # tokens + .vsl-slot/.vsl__cta + .gate + .accept-btn/.decline + responsivo
public/assets/      # webp (posters, imagens), .woff2 (fontes), favicon.svg
astro.config.mjs    # output:'static'
```

---

## 12. Checklist pra nova LP

- [ ] Astro estático, Base/Funnel layouts, global.css com tokens.
- [ ] VSL facade (box aspect-ratio + poster ffmpeg + overlay + injeta no toque). Strip do `vid-` na URL.
- [ ] Todas as imagens webp + dimensões/aspect-ratio (CLS 0).
- [ ] Fontes self-host woff2 + preload das de cima da dobra.
- [ ] Tracking diferido por padrão (alinhar eager vs nota com o dono). Sem preload da GTM.
- [ ] Gate da VSL com Date.now + visibilitychange + ?preview + localStorage.
- [ ] Payt one-click verbatim (data-object por página) + `?split=12` no checkout da home.
- [ ] Animações só transform/opacity + reduced-motion.
- [ ] Medir no PageSpeed 3x (mediana), olhar field data; não instalar speed-insights.
- [ ] Deploy Vercel + domínio (CNAME/nameservers).
```
```
