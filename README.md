# Chiaro Downloader + WebShot

Extensão do Chrome com duas ferramentas, uma em cada aba do popup:

- **Baixar** — vídeos do **YouTube, TikTok, Instagram e X**, inteiros ou em
  trechos — e, com sorte, de quase qualquer outro site.
- **Capturar** — o antigo **Chiaro WebShot**: captura o site inteiro em
  altíssima resolução e monta o pacote pronto para o After Effects. Ver
  [Capturar páginas](#capturar-páginas-o-antigo-chiaro-webshot).

No download a extensão é uma casca fina: o trabalho de verdade é feito pelo
`yt-dlp` rodando localmente, invocado via Native Messaging. Isso significa que
quando um desses sites muda alguma coisa, quem conserta é o projeto yt-dlp —
você só clica em "Atualizar". A captura não depende do helper: roda inteira no
Chrome.

Uso pessoal, instalação sem loja.

---

## Instalação

**Duplo clique em `instalar.bat`** e aceite o pedido de administrador.

Depois, no Chrome:

1. abra `chrome://extensions`
2. ligue o **Modo do desenvolvedor** (canto superior direito)
3. **Carregar sem compactação** → escolha a pasta `extension\`

O ID que aparecer tem que ser igual ao que o instalador imprimiu no final.

### Por que precisa de administrador

O Chrome procura o helper no registro do Windows em dois lugares: `HKCU`
(nível usuário) e `HKLM` (nível máquina). A política
`NativeMessagingUserLevelHosts`, quando desligada, faz ele **ignorar o `HKCU`
inteiro**. O sintoma nesse caso é cruel: chave de registro presente, JSON
válido, caminhos corretos — e mesmo assim `Specified native messaging host
not found`. Escrever em `HKLM` resolve, e `HKLM` exige elevação. O instalador
escreve nos dois.

### O que o instalador faz

Instala o Python (via `winget`, escopo de usuário), cria um venv isolado em
`helper\.venv`, instala `yt-dlp[default,curl-cffi]`, baixa `ffmpeg.exe` para
`helper\bin`, gera a chave RSA da extensão e escreve os registros do Native
Messaging. Nada vai parar no PATH e nada é instalado no Python global.

Os extras do yt-dlp não são opcionais: sem `curl_cffi` o yt-dlp cai no
`urllib` da biblioteca padrão, e o TikTok reprova esse fingerprint de TLS.
Medido no mesmo vídeo e mesma versão: **0/3 sem os extras, 3/3 com eles.**

É **idempotente**: rodar de novo conserta em vez de duplicar. Se você mover a
pasta do projeto, rode de novo — os caminhos absolutos são reescritos e o ID
da extensão continua o mesmo.

Para só diagnosticar, sem instalar nada (não precisa de administrador):

```
instalar.bat /check
```

### Vindo do Chiaro WebShot

Até a 2.4 o WebShot era uma extensão e uma instalação separadas. O instalador
da 3.0 encontra a instalação antiga e oferece removê-la (vem marcado, mas à
vista): apaga a pasta dela, o atalho do Menu Iniciar e a entrada em
*Aplicativos instalados*. Ele procura pela chave de registro **e** pela pasta
padrão (`%USERPROFILE%\Chiaro WebShot`), porque já apareceu máquina com a pasta
e o atalho no lugar e a chave sumida. E só apaga a pasta se ela tiver o
`Desinstalar Chiaro WebShot.exe` que o instalador do WebShot deixa lá: um
registro apontando para o lugar errado não vira uma pasta errada apagada.

Falta um passo que só você pode dar: em `chrome://extensions`, clicar em
**Remover** no cartão antigo do Chiaro WebShot — o Chrome não deixa uma
extensão descompactada ser removida por fora. Os pacotes que você já baixou
continuam em `Downloads\Chiaro WebShot`; as três capturas recentes guardadas
dentro da extensão antiga vão embora com ela.

Se preferir manter os dois por um tempo, desmarque a opção. Só saiba que o
atalho `Alt+Shift+S` existe nos dois e o Chrome o dá a quem chegou primeiro —
o antigo.

---

## Instalar em outra máquina

Não copie a pasta do projeto: o `.venv` aponta para este Python, os `.exe` do
ffmpeg pesam ~180 MB e o manifesto do Native Messaging tem o caminho absoluto
desta máquina cravado dentro.

```powershell
.\Empacotar.ps1
```

Gera `Chiaro Downloader - portatil.zip` na Área de Trabalho, só com o código
(sem o `.venv` nem os binários). Na outra máquina: descompacte, duplo clique em
`instalar.bat`, carregue a pasta `extension\` no Chrome.

O **ID da extensão é o mesmo em todas as máquinas**, porque a chave pública
viaja dentro do `extension\manifest.json`. A chave privada
(`extension-key.xml`) **não** vai no pacote — ela não é necessária para
instalar e não há motivo para espalhá-la. Se quiser incluí-la mesmo assim, use
`.\Empacotar.ps1 -IncluirChave` e trate o zip como material sensível.

Na máquina de destino o instalador baixa Python, yt-dlp e ffmpeg (~100 MB),
então precisa de internet e leva alguns minutos. Evite descompactar em pasta
sincronizada pelo OneDrive.

---

## Como usar

| Gatilho | O que faz |
|---|---|
| `Alt+Shift+D` | Baixa o vídeo da aba atual em MP4 1080p |
| `Alt+Shift+A` | Baixa só o áudio da aba atual |
| `Alt+Shift+S` | Captura a página da aba atual com as últimas opções |
| Botão direito | No vídeo, na página ou **em um link** |
| Clique no ícone | Popup com duas abas: **Baixar** (os cinco modos, recorte, fila e histórico) e **Capturar** |

O popup abre na última aba usada — exceto com uma captura em andamento, que
sempre abre em Capturar, porque é o progresso dela que você veio ver.

### Os cinco modos

| Modo | O que faz | Custo |
|---|---|---|
| **MP4 1080p** | Melhor H.264 até 1080p | — |
| **MP4 720p** | Melhor H.264 até 720p | arquivo menor |
| **MP4 MAX** | Resolução máxima em qualquer codec, **convertido** para H.264 | reencoda |
| **MKV MAX** | Resolução máxima, codec nativo (`.mkv`) | pode não abrir no editor |
| **AUDIO** | Trilha de áudio em `.m4a` | — |

**MP4 MAX** é o único que reencoda, e existe para o caso em que a melhor
resolução só está em AV1 ou VP9: baixa esse arquivo, converte para H.264 8 bits
e **apaga o original**. Se o melhor formato já for H.264, ele percebe e pula a
conversão em vez de degradar a imagem à toa.

Por ser o único que pode prender a máquina, ele **pede confirmação** antes de
começar. O aviso é inline, e não um `confirm()` do navegador: o diálogo nativo
tira o foco da janela, e o popup do Chrome fecha sozinho quando perde o foco —
o clique em "OK" se perderia.

A conversão usa a **GPU** quando possível — NVENC, QuickSync ou AMF, nessa
ordem, caindo para CPU (`libx264`) se nenhuma funcionar. A escolha não confia
na lista de encoders do ffmpeg: cada um é testado com uma codificação real de
um quadro, porque NVENC aparece listado mesmo em máquina sem driver capaz. O
histórico registra qual encoder foi usado.

O 8 bits (`yuv420p`) é o ponto do modo: AV1 e VP9 em alta resolução costumam
vir em 10 bits, e H.264 10 bits quase não tem suporte em player e editor.

Modos diferentes do mesmo vídeo **não colidem**: o arquivo ganha um sufixo
(`720p`, `MAX`). Sem isso o yt-dlp encontraria o arquivo pronto e pularia o
download — clicar em 1080p depois de já ter baixado em 720p não faria nada.

Os atalhos podem ser trocados em `chrome://extensions/shortcuts`.

### Streaming (HLS e DASH)

Player de streaming busca o manifesto `.m3u8` (ou `.mpd`) por `fetch` e entrega
ao `<video>` uma URL `blob:` — que existe só dentro daquela aba e não serve
para nada fora dela. Varrer o DOM não encontra nada.

A saída é a própria página: ela guarda o log das requisições que fez, em
`performance.getEntriesByType("resource")`, e o manifesto está ali. **Não exige
permissão nova** — só o `activeTab` que já era usado.

Entre vários manifestos, vale o **pedido primeiro**: a API devolve em ordem
cronológica e o *master* sempre vem antes das variantes. Pegar uma variante
traria uma única qualidade; o master traz todas, e o yt-dlp escolhe a maior.

### Sempre o maior arquivo da página

Numa página qualquer convivem coisas bem diferentes: um preview curto num
`<video>`, um embed de trailer e o manifesto do conteúdo inteiro. Aceitar o
primeiro que funciona costuma pegar o menor.

Em site desconhecido, a extensão manda **todos** os candidatos ao helper, que
sonda cada um sem baixar nada e fica com o de maior tamanho estimado. Quando o
servidor não informa o tamanho — o normal em HLS — a conta é `bitrate ×
duração`, que basta para comparar entre si.

A sondagem é feita **sem o seletor de formato**: com ele aplicado, um candidato
cujo seletor não casa levanta exceção e some da comparação — e era justamente o
master HLS, o maior deles, que caía fora.

O popup mostra `comparando o que há na página` durante esse passo. Teto de 5
candidatos, porque cada sondagem custa uma ida à rede.

### Vimeo

O yt-dlp passou a exigir login para qualquer página `vimeo.com`, inclusive de
vídeo público ("The web client only works when logged-in"). O mesmo vídeo pelo
endereço do player (`player.vimeo.com/video/<id>`) sai sem login. Por isso o
helper tenta, nesta ordem:

1. **O player**, sem cookie. Resolve vídeo público e não listado (o hash do
   link `vimeo.com/<id>/<hash>` vai junto).
2. **A página original com o seu login do Chrome**, só se o vídeo pedir conta
   (privado, de equipe). É o mesmo esquema de cookie sob demanda do YouTube.
3. **O vídeo que a aba está tocando.** Os links de review novos
   (`vimeo.com/reviews/…/videos/…`) o yt-dlp nem reconhece. Mas se o vídeo
   abre no Chrome, o manifesto dele está na página, e sai de lá. O arquivo
   leva o título da aba, e não "playlist". Só vale quando a aba é a do próprio
   vídeo, por isso **dê play antes de baixar**.

### Outros sites

Os cinco principais (YouTube, TikTok, Instagram, X e, desde a 3.1.1, Vimeo)
têm tratamento dedicado — cookies, detecção de feed,
política de formato. Mas em **qualquer página `http` ou `https`** os botões
continuam ativos: o yt-dlp reconhece mais de mil sites e ainda tem um extrator
genérico que procura vídeo embutido na própria página. O popup avisa que está
em terreno desconhecido, e a tentativa pode simplesmente não dar certo.

Nesses sites a extensão **não envia cookies**. Conteúdo que exija login ali
provavelmente falha, e é assim de propósito: mandar cookies de qualquer site
para o yt-dlp seria uma troca ruim por um caso de uso ocasional. O popup avisa
disso na exclamação ao lado do nome do site.

**Quem garante isso mudou na 3.0.** Até a 2.4 a barreira era do Chrome: a
extensão só tinha permissão sobre os quatro domínios, e não *conseguiria* ler
um cookie de outro site. Com o WebShot dentro, ela pede `<all_urls>` e
`debugger` — e o próprio Chrome descreve o `debugger` como acesso a "todos os
seus dados em todos os sites". A partir daí a barreira é **do código**:
`cookiesFor()` em `background.js` só lê os `cookieDomains` dos cinco sites de
`sites.js` e devolve vazio para qualquer outro. Mudar isso é mudar a política
de privacidade da ferramenta, não um detalhe — revise junto.

Deixar `<all_urls>` opcional não devolveria a barreira antiga: o `debugger` não
aceita ser opcional (o Chrome recusa em `optional_permissions`), então o acesso
amplo existe desde a instalação de qualquer jeito.

Os arquivos caem em `Vídeos\Baixados\outros\`.

### O truque do feed

No **feed do TikTok** e na **timeline do X** existem vinte vídeos na mesma
página — "o vídeo desta página" não quer dizer nada, e o atalho vai errar.

Nesses casos, clique com o **botão direito no link** do post (no X, o link da
data/hora do tweet; no TikTok, o link do vídeo) e use o menu de contexto. A
extensão usa o `linkUrl` daquele elemento, então ela sabe exatamente qual vídeo
você quis.

### Por que o H.264 tem teto

Os modos **MP4 1080p** e **MP4 720p** entregam **H.264 + AAC**: abre em qualquer
editor, player, celular e no PowerPoint. No YouTube isso limita a **1080p na
maioria dos vídeos**, porque acima disso ele só serve VP9 e AV1 em WebM — não
existe H.264 em 4K lá.

**Mas o teto pode ser mais baixo.** Em alguns vídeos o YouTube nem chega a
codificar 1080p em H.264, e o maior disponível é 720p. Exemplo real:

```
av01   [144, 240, 360, 480, 720, 1080]
avc1   [144, 240, 360, 480, 720]   <- teto do H.264 neste video
vp9    [144, 240, 360, 480, 720, 1080]
```

Quando isso acontece, o popup **avisa durante o download**: *"720p é o maior
H.264 deste vídeo. Há 1080p em VP9/AV1 — use MP4 MAX (converte) ou MKV
MAX."* O histórico também passa a registrar a resolução e o encoder de
cada arquivo. A informação sai da mesma sondagem que já era feita antes de
baixar, então não custa nenhuma requisição a mais.

É exatamente para esse caso que existe o **MP4 MAX**: ele baixa o 1080p em AV1
e converte para H.264, entregando a resolução alta num arquivo que abre em
tudo. O preço é o tempo de conversão.

Para 4K, os caminhos são **MKV MAX** (rápido, `.mkv` em AV1/VP9, pode não
abrir no editor) ou **MP4 MAX** (converte para H.264, abre em tudo, demora).

### Cortar um trecho

Marque **Cortar um trecho** no popup e preencha dois campos:

| Início | Segundo campo | Resultado |
|---|---|---|
| `2:00` | `1:00` | 1 minuto a partir de 2:00 |
| `2:00` | `=3:00` | de 2:00 **até** 3:00 |
| *(vazio)* | `30` | os primeiros 30 segundos |

O segundo campo é duração por padrão; o prefixo `=` faz dele uma marcação de
fim. Os dois aceitam `90`, `1:30`, `0:01:30` ou `1m30s` indiferentemente — o
popup mostra o intervalo interpretado antes de você baixar.

O botão **usar o tempo atual do vídeo** preenche o início com o ponto onde
você parou o player. Ele lê só o elemento `<video>` padrão do HTML, sob demanda
e apenas quando você clica — não é um seletor de CSS por site, que é o tipo de
coisa que quebra a cada redesenho.

**Ele baixa só o trecho, não o vídeo inteiro.** O yt-dlp pede ao servidor
apenas aquele intervalo. Medido num vídeo de 5 minutos em 4K: um corte de
1 minuto puxou 33 MB em vez do arquivo completo.

### Corte exato depende do modo

Fazer o corte cair no ponto pedido exige **reencodar** — e o reencode adota os
codecs padrão do container de saída. Isso muda tudo conforme o modo:

| Modo | Corte | Por quê |
|---|---|---|
| **MP4 1080p**, **MP4 720p** | **exato** | o reencode sai em `h264 + aac`, que é o destino de qualquer forma |
| **MP4 MAX**, **MKV MAX** | no keyframe | exato transformaria `av1 + opus` em `h264 + vorbis`, jogando fora justamente o que se foi buscar |

Nos modos MAX o corte pode cair alguns segundos antes ou depois do pedido — é o
preço de preservar o vídeo. Nos modos MP4 ele é exato e não custa nada.

Isso não é teoria: medido no mesmo vídeo, `MKV MAX` sem recorte dava
`av1 + opus` e com recorte exato dava `h264 + vorbis`. O recorte estava
destruindo em silêncio o que o modo existe para preservar.

O arquivo sai com o trecho no nome, então não colide com o vídeo inteiro nem
com outro corte do mesmo vídeo:

```
Jacob + Katie Schwarz - COSTA RICA IN 4K 60fps HDR [LXb3EKWsInQ] (2m00s-3m00s).mp4
```

---

## Capturar páginas (o antigo Chiaro WebShot)

A aba **Capturar** do popup é o Chiaro WebShot inteiro: captura a página em
altíssima resolução (4000 px ou mais de largura) e monta o resultado pronto
para o **Adobe After Effects**.

1. Abra o site e clique no ícone. Se houver banner de cookies ou popup, feche-o
   antes.
2. Na aba **Capturar**, clique em **Capturar página**. Os valores padrão servem
   para a maioria dos sites.
3. **Não troque de aba** durante a captura. Uma aba de resultado abre no final.

`Alt+Shift+S` captura com as últimas opções, sem abrir o popup.

Durante a captura o Chrome mostra uma faixa dizendo que a extensão está
depurando o navegador. É o `chrome.debugger`, que é como a captura controla a
página (viewport de alta resolução, captura além da área visível); a faixa
some quando ela termina.

### After Effects

Nos dois pacotes, extraia o `.zip` e, no AE, use *Arquivo › Scripts › Executar
arquivo de script…* no `.jsx`. O script cria a composição no tamanho da tela
(16:9 por padrão), com todas as camadas filhas de um null "Página" que já vem
com uma rolagem de exemplo.

- **Pack Leve** (recomendado): poucas camadas, cada uma com a página inteira.
  Textos e fundo vêm em **vetor** (PDF, e também EPS), com *Rasterização
  contínua* ligada — dá para ampliar sem perder nitidez. Os contornos das
  letras são vetorizados da própria renderização do Chrome, então funciona com
  qualquer fonte. Imagens vêm em PNG transparente, e há reservas em imagem
  (desligadas) para o caso de o AE não importar um vetor.
- **Pack Completo**: uma camada por bloco de texto e por imagem, com texto
  editável opcional (instale antes as fontes de `fontes_usadas.txt`).

A resolução (100%, 75%, **50%**, 33%) define o tamanho da composição; acima de
30.000 px de altura (o limite do AE) cada camada vem dividida em partes.

### Captura e outras saídas

- **Página inteira sem rolar** (padrão): o Chrome renderiza a página toda de
  uma vez. Nada que acompanha a rolagem se repete, e o `vh` não muda.
  **Rolando por trechos** fica para sites que não renderizam fora da tela.
- **Animações de rolagem**: a página é rolada antes, para carregar lazy-load e
  disparar animações; o estado revelado de cada elemento é reaplicado.

| Saída | Para quê |
|---|---|
| PNG / JPEG / WebP | A imagem inteira num arquivo só |
| SVG vetorial completo | Vetores, texto e imagens originais — Illustrator, Figma, navegador |
| SVG Web | HTML/CSS fiel dentro do SVG, com animações CSS opcionais — só navegador |

As capturas ficam guardadas na extensão (as 3 mais recentes) e os pacotes
baixados caem em `Downloads\Chiaro WebShot\`.

### Como as duas metades convivem

As duas rodam no mesmo service worker (`background.js` importa `captura.js`)
e não se enxergam. Duas regras mantêm isso:

- **Mensagens da captura têm prefixo `ws:`** (`ws:start`, `ws:cancel`,
  `ws:status`, `ws:progress`…). Sem ele, o `cancel` de um download também
  cancelava a captura — e o da captura mandava ao helper um cancelamento sem
  `id`.
- **Ninguém escreve direto no badge do ícone.** Os dois passam por `badge.js`,
  que decide: captura em andamento > download em andamento > erro da captura.
  Antes, cada tique de progresso do download apagava a porcentagem da captura.

---

## Onde os arquivos caem

```
%USERPROFILE%\Videos\Baixados\
├── youtube\
├── tiktok\
├── instagram\
├── x\
├── vimeo\
└── outros\
```

Nome: `autor - título [id].mp4`, com autor em até 25 caracteres e título em 40.

O link **pasta de downloads**, no rodapé do popup, abre essa pasta principal.
Cada item do histórico tem o seu **abrir na pasta**, que abre o Explorer com o
arquivo selecionado. Se o arquivo mudou de extensão depois da conversão, ele
acha o novo; se foi apagado ou movido, abre só a pasta e diz isso no próprio
botão.

**Cancelar** para o download na hora em qualquer fase. Isso inclui o corte de
trecho e a junção de áudio e vídeo, em que o yt-dlp entrega o trabalho ao
ffmpeg e não dá notícia até o fim. O helper encerra o ffmpeg daquele download e
apaga os arquivos parciais.

### Higiene do nome

Os nomes saem em **ASCII puro**, para não quebrar editores, players e scripts
que engasgam com caractere fora do comum. O sanitizador **traduz** em vez de
simplesmente apagar:

| Entra | Sai |
|---|---|
| `ESCRITÓRIO` | `ESCRITORIO` |
| `𝐅𝐮𝐫𝐫𝐲 𝐋𝐨𝐛𝐨` (negrito matemático) | `Furry Lobo` |
| `ʟᴠʟ30+` (versalete) | `LVL30+` |
| `Mibal ｜ Furry` (barra de largura total) | `Mibal Furry` |
| `Star Mountain – O fim` (travessão) | `Star Mountain - O fim` |
| `nota 10 👍 bom` | `nota 10 bom` |
| `ᎩᎧᏕᏂᎥᏒᎧ˚☽˚｡⋆𓃦` (cherokee, hieróglifo) | *(vazio — cai para só o título)* |

O truque é olhar o **nome Unicode** do caractere: `MATHEMATICAL BOLD SMALL F`
vira `f`, `LATIN LETTER SMALL CAPITAL L` vira `l`. Isso cobre negrito
matemático, versalete e largura total sem tabela manual. Emoji e escritas sem
equivalente saem fora.

Também trata: caracteres proibidos do Windows (`< > : " / \ | ? *`), nomes
reservados (`CON`, `PRN`, `COM1`…), espaços colapsados, e corte **em palavra
inteira** em vez de no meio.

O truncamento não é estética: TikTok e Instagram não têm título de verdade — o
que o yt-dlp chama de título lá é a legenda inteira, com emoji e trinta
hashtags. Sem cortar, o caminho estoura o limite de 260 caracteres do Windows e
o download falha **depois** de já ter baixado o arquivo. O `[id]` no fim evita
colisão e te deixa achar o original depois.

Nomes típicos ficam entre 55 e 80 caracteres, contra os 90+ de antes:

```
Porta dos Fundos - ESCRITORIO CHATGPT [9n6sxWtYz6I].mp4
parquesbrasil - Star Mountain - O fim de uma era [7674619332311338260].mp4
```

Para mudar a pasta, edite `downloadRoot` em `helper\config.json`.

---

## Conteúdo com login

Desde o Chrome 127, o Windows usa **App-Bound Encryption** nos cookies: a chave
de decriptação está presa ao processo do Chrome. Na prática, o
`yt-dlp --cookies-from-browser chrome` **não funciona mais** nesta plataforma.

É por isso que isso aqui é uma extensão e não um script. Rodando **dentro** do
Chrome, a API `chrome.cookies` entrega o cookie já decriptado.

Como está implementado:

- os cookies são lidos **só do site daquele vídeo**, nunca o jar inteiro;
- **no momento do download**, não em background;
- vão pelo pipe do stdio direto para o `cookiejar` em memória do yt-dlp —
  **nada é gravado em disco**, em nenhum momento.

Sem `cookies.txt`, que é justamente o arquivo que um infostealer procura.

### Quando o cookie é enviado

Não é sempre, e a diferença é obrigatória, não preferência:

| Site | Política | Motivo |
|---|---|---|
| Instagram, X | **sempre** | exigem login para praticamente todo conteúdo |
| YouTube, TikTok | **só se precisar** | funcionam sem cookie |

O YouTube **quebra** quando recebe um jar de sessão logada sem os tokens de
sessão que ele passou a exigir: responde *"The page needs to be reloaded"* e
devolve zero formatos. Mesmo vídeo sem cookie: 48 formatos. Por isso ele tenta
limpo primeiro e só repete com cookie se o site pedir login ou idade — o popup
mostra "tentando de novo com login" nesse intervalo. Vídeo com restrição de
idade continua alcançável, apenas na segunda tentativa.

Vale saber o que isso significa: um cookie de sessão do Instagram equivale a
acesso à sua conta. Ele nunca sai da sua máquina, mas a extensão tem permissão
para lê-lo.

---

## Quando quebrar

### "O site mudou e o yt-dlp precisa ser atualizado"

A falha mais provável de todas, e a de conserto mais fácil: abra o popup e
clique em **Atualizar agora**. O popup também avisa sozinho quando sai versão
nova (checa uma vez por semana). Depois de atualizar, recarregue a extensão em
`chrome://extensions`.

A atualização nunca acontece sozinha, de propósito — ferramenta mudando de
comportamento sem você saber é pior que ferramenta desatualizada.

### "O helper não respondeu"

```powershell
.\setup.ps1 -Check
```

Isso diagnostica cada peça — venv, yt-dlp, ffmpeg, chave, ID, registro — e
termina fazendo um **ping real pelo mesmo protocolo que o Chrome usa**. Se o
ping passa, a ponte está de pé e o problema é outro.

O log fica em `helper\host.log`.

### O Instagram falhando do nada

O Instagram é, de longe, o mais hostil dos sites principais. Mesmo com cookie válido ele
limita acesso de forma agressiva e falha de modo intermitente, sem padrão claro.
Isso não é bug daqui e não tem conserto deste lado — espere alguns minutos e
tente de novo. YouTube, TikTok e X são bem mais estáveis.

---

## Atualizações do Chiaro Downloader

A partir da 3.1.0 a extensão se atualiza pelos releases do GitHub
([audiovisualpecege/ChiaroDownloader](https://github.com/audiovisualpecege/ChiaroDownloader/releases)).
Ao abrir o popup, o helper pergunta ao GitHub se há versão nova, no máximo uma
vez a cada 6 horas. Havendo, aparece uma faixa abaixo das abas com **o que
mudou** e **Atualizar**. Nada é instalado sem esse clique.

O clique baixa o pacote, confere a assinatura, troca os arquivos de
`C:\VideoDownloader` e recarrega a extensão. Dá para atualizar só sem download
nem captura rodando, porque recarregar a extensão mataria os dois. O
`config.json`, o histórico e o ambiente Python não são tocados.

**A assinatura.** Cada pacote é assinado com uma chave Ed25519
(`chave-atualizacao.pem`, na raiz do projeto, **fora do Git**). O helper só
tem a parte pública. Um pacote adulterado, ou trocado por alguém com acesso à
conta do GitHub, é recusado antes de qualquer arquivo mudar. **Guarde um
backup dessa chave fora deste computador.** Sem ela não dá para assinar a
próxima versão, e as instalações existentes só voltam a se atualizar pelo
instalador de uma versão com chave nova.

**Quando o updater manda baixar o instalador.** Quando a versão muda o
`helper/pyproject.toml` (dependência Python nova), quando muda a `key` da
extensão, ou quando o release foi gerado com `--requer-instalador` (ffmpeg,
Deno ou Python novos, que só o `setup.ps1` sabe montar). A faixa então mostra
**Baixar instalador**.

**Se der errado.** Se a troca falha no meio, por exemplo porque o Chrome
estava lendo um arquivo, tudo volta ao que era. A versão anterior inteira fica
em `C:\VideoDownloader\_atualizacao\anterior`. Na pasta do projeto o updater
só avisa e não troca nada, para não sobrescrever o código-fonte.

### Lançar uma versão

1. Suba o `"version"` em `extension/manifest.json` (ex.: `3.1.0` → `3.1.1`).
2. Gere os arquivos com o Python do helper (é ele que tem a biblioteca de
   assinatura):

   ```powershell
   .\helper\.venv\Scripts\python.exe installer\build_installer.py
   ```

   Some `--requer-instalador` ao fim se a versão mexe no ffmpeg, no Deno ou no
   Python.
3. Em [Releases → Draft a new release](https://github.com/audiovisualpecege/ChiaroDownloader/releases/new),
   crie a tag `v3.1.1` e anexe os três arquivos de `dist\`:
   - `Chiaro-Downloader-Setup-3.1.1.exe` (instalações novas)
   - `ChiaroDownloader-3.1.1.zip` (o pacote do updater)
   - `ChiaroDownloader-3.1.1.zip.sig` (a assinatura)

   Escreva na descrição o que mudou: é o que o link "o que mudou" abre.
4. Publique sem marcar "pre-release": o updater só enxerga o release marcado
   como *latest*.

O nome dos anexos tem que ser exatamente esse: o updater procura
`ChiaroDownloader-<versão>.zip` e `.zip.sig` com a versão da tag.

---

## Detalhes de implementação que valem saber

**Por que o download não morre no meio.** O service worker do MV3 dorme depois
de ~30s ociosos, e o Chrome mata o processo do helper quando a porta fecha.
Enquanto há download, o yt-dlp manda progresso ~1x por segundo, e cada mensagem
reseta o timer de ociosidade. Quando acaba, o worker dorme, a porta fecha e o
helper encerra sozinho. Se ainda assim algo morrer no meio, o yt-dlp deixou um
`.part` e o próximo disparo **retoma** em vez de recomeçar.

**Por que dois downloads por vez.** Mais que isso e YouTube e Instagram passam a
devolver HTTP 429. Você baixaria mais devagar tentando ir mais rápido.

**Por que não existe content script.** Um botão dentro do player seria mais
bonito, e custaria quatro conjuntos de seletores CSS que quebram sem aviso, um
site por vez. Menu de contexto e atalho não têm esse problema.

**Por que o `key` no manifest.json.** Extensão descompactada tem o ID derivado
do caminho da pasta. Sem a chave fixa, mover a pasta mudaria o ID, e o
`allowed_origins` do Native Messaging deixaria de bater. É também o que mantém
o ID igual ao instalar em outra máquina.

**Por que o host é um `.exe` e não um `.bat`.** O `.bat` roda sob `cmd.exe`, o
que cria um processo neto: matar o pai deixa o `python.exe` vivo segurando os
pipes, e qualquer leitura seguinte trava para sempre. O `.exe` vem do
`console_scripts` do `pyproject.toml`, instalado em modo editável — editar
`host.py` continua tendo efeito imediato, sem reinstalar.

**Por que os JSONs são escritos sem BOM.** `Set-Content -Encoding utf8` no
Windows PowerShell 5.1 grava UTF-8 **com** BOM, e o parser de Native Messaging
do Chrome rejeita um manifesto que começa com `EF BB BF` — falhando de forma
muda, sem nunca lançar o processo. O `json.load` do Python também quebra. O
`-Check` reprova se qualquer JSON tiver BOM.

---

## Marca

A interface segue o **Design System da Chiaro Filmes** (`Chiaro Filmes Design
System/`), tema **noite**, por uma folha só: `extension/styles/chiaro.css`
(tokens + componentes `.btn`, `.chip`, `.campo`, `.check`, `.aviso`, `.barra`,
`.marca`, `.card`), que veio do WebShot e é usada pelo popup e pela página de
resultado da captura. **Esta é a cópia que vale**: o projeto `WebShot/` ficou
como referência e não recebe mais mudanças. O `popup.css` só tem o que é do
popup (a cabeça com as abas, a fila, o histórico, o recorte, o aviso do site,
a moldura da aba Capturar), sempre sobre os tokens de lá, e o instalador
(`installer/Setup.cs`, classe `Theme`) repete os mesmos valores em
`Color.FromArgb`.

As abas são o `Tabs` do sistema, variante sublinhada (a ativa vai de Altivo
Light a Black), com uma mudança: o texto ativo é quase-branco, e não Lavanda.
Lavanda sobre Noite dá 4,16:1, abaixo dos 4,5:1 do AA para texto de 15px; o
Lavanda ficou no sublinhado.

| | |
|---|---|
| Fundo | Noite `#070010` com o brilho violeta descendo do topo (`--gradient-noite`) |
| Acento | Lavanda `#8C4EDB`, sólido no botão principal (`.btn.primario`), igual ao "Capturar página" do WebShot |
| Aviso | `--warning` `#E8A93B`, o token semântico do sistema |
| Tipos | **Altivo** (caixa alta, Light + Black na mesma linha) nos títulos e botões; **Archivo** no texto |
| Cantos | Sempre arredondados: pílula nos botões, 12px nos campos e nas linhas de lista |

As fontes viajam dentro da extensão (`extension/fonts/`, ~1 MB) porque o popup
não pode buscar fonte na rede. Altivo é licenciada; Archivo é OFL e a licença
vai junto no mesmo diretório.

Três regras de conteúdo do sistema valem para qualquer texto novo: **português
com acento correto**, **título em caixa alta** e **sem emoji** (nem ponto de
exclamação em texto institucional).

---

## Estrutura

```
extension/          o que você carrega no Chrome
├── manifest.json   MV3, key RSA fixa
├── background.js   service worker: menu, atalhos, porta nativa, cookies
├── captura.js      a captura (antigo WebShot), importada pelo background.js
├── badge.js        o badge do ícone, dividido entre download e captura
├── sites.js        sites, política de cookie, feed vs vídeo
├── popup.*         as duas abas; a de Baixar: fila, recorte, histórico
├── captura-popup.js  a aba Capturar
├── timecode.js     leitura do recorte (1:30, 90, 1m30s, =3:00)
├── lib/            captura: IndexedDB, opções, PNG, PDF/EPS, zip, After Effects
├── content/        captura: scripts injetados na página capturada
├── result/         captura: a página de resultado, com os downloads
├── styles/         chiaro.css, o design system
├── fonts/          Altivo + Archivo, as fontes da marca
├── logo.svg        wordmark Chiaro (o "O" em Violeta #672FA2)
└── icons/
helper/             o que faz o trabalho
├── host.py         protocolo stdio, cookiejar em memória, fila, progresso
├── atualizador.py  atualização do app pelos releases do GitHub
├── selftest.py     ping de ponta a ponta pelo protocolo do Chrome
├── pyproject.toml  gera o dlv-host.exe (console_script)
├── .venv/          criado pelo instalador
└── bin/            ffmpeg.exe, ffprobe.exe
instalar.bat        duplo clique: eleva, instala, mostra o resultado
setup.ps1           o instalador de fato (-Check para diagnosticar)
Empacotar.ps1       gera o zip portátil para outra máquina
installer/          build_installer.py: instalador .exe + pacote de atualização assinado
chave-atualizacao.pem  chave privada das atualizações (fora do Git; faça backup)
```

---

## Nota

Baixar vídeos do YouTube viola os Termos de Serviço da plataforma. Esta
extensão é para uso pessoal e não é publicável na Chrome Web Store — a política
de Produtos Proibidos veda explicitamente extensões que facilitam download de
conteúdo protegido por direitos autorais ou por login, e o enforcement inclui
banimento permanente da conta de desenvolvedor. Por isso: instalação
descompactada, sem loja.
