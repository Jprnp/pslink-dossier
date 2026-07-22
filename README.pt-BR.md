# Adaptador PS Link da Sony mata o áudio do Windows: causa raiz encontrada (violação do protocolo USB), com correção

*Read this in [English](README.md).*

> [!NOTE]
> **Transparência sobre IA:** eu não sou especialista em protocolo USB, stack de áudio nem análise de firmware. Este trabalho está muito além da minha profundidade técnica pessoal. Tanto a investigação quanto este dossiê foram feitos com ajuda de LLMs (Claude Opus 4.8 e Claude Fable 5, da Anthropic). Elas conduziram a metodologia, o ferramental e a análise, enquanto eu operava o hardware e tomava as decisões. Não confie em nenhum de nós de graça: toda afirmação aqui é sustentada por capturas e experimentos reproduzíveis, então verifique.

> [!IMPORTANT]
> ### A parte humana desta história
> Minha contribuição de verdade aqui foi curiosidade e teimosia. O suporte da PlayStation disse que era "um problema do Windows" e encerrou o caso. Mais tarde descobri que as LLMs também desistiam se eu deixasse: mais de uma vez elas concluíram que se a Sony não atualizar o firmware não tem o que fazer, e sugeriram que eu aceitasse um watchdog de recuperação automática como resposta final. Precisei insistir bastante pra investigação continuar depois disso, e a causa raiz deste documento foi encontrada porque ela continuou.
>
> A evidência sempre esteve no barramento, acessível a qualquer um com as ferramentas certas. Só precisava de alguém disposto a continuar perguntando.

**TL;DR:** Se o seu headset PULSE Elite / PULSE Explore (via adaptador USB PS Link) mata TODO o áudio do Windows aleatoriamente — principalmente ao mudar o volume — não é seu driver, não é a Realtek, não é o Windows. O firmware do adaptador viola a spec USB: transmite **rajadas de 256 bytes num endpoint interrupt que ele mesmo declarou como máximo de 64 bytes**. O Windows detecta isso como **babble** (`USBD_STATUS_BABBLE_DETECTED`), reseta o pipe, e quando há áudio tocando, essa recuperação de erro derruba o stream junto — escalando até o travamento completo do `audiodg.exe`. **Workaround rápido abaixo (sem software nenhum) — o áudio fica estável e os botões de volume continuam funcionando.** Um app grátis e open-source que corrige *por completo* — sem freeze **e** com todas as funções preservadas (auto-sync, sidetone, EQ, mute do mic, volume, bateria) — também já está disponível: veja [**A correção completa**](#a-correção-completa-app-open-source).

- Dispositivo: adaptador USB PlayStation Link, `VID_054C PID_0ECC`, firmware/bcdDevice **1.43** (`REV_0143`, o mais recente até a data). **Todos os achados deste documento foram capturados especificamente no firmware 1.43** — outras versões podem ou não se comportar igual. Pra conferir o seu: Gerenciador de Dispositivos → entrada "PlayStation Link" do adaptador → Propriedades → Detalhes → IDs de Hardware → o sufixo `REV_xxxx` é o firmware (ex.: `USB\VID_054C&PID_0ECC&REV_0143`). Se o seu for outra versão, relatos em qualquer direção são muito bem-vindos.
- Host: Windows 11 (build 26200), somente drivers inbox da Microsoft (`usbaudio.sys` + `HidUsb`) — o app da Sony pra PC NÃO participa da falha (verificado: o freeze reproduz com ele morto)
- Reproduzido em **dois laptops Windows diferentes** (hardwares distintos), o que descarta um único PC ou controlador USB defeituoso como causa. As capturas detalhadas de barramento deste documento foram feitas em um deles.
- Evidência: capturas de barramento USB (USBPcap), traces ETW de áudio, dump do `audiodg.exe` travado e experimentos A/B controlados. Capturas cruas disponíveis a quem pedir (serial do device redigido).

> **Nota sobre o suporte PlayStation:** eu contatei o suporte da PlayStation sobre esses travamentos muito antes de qualquer análise daqui existir. O caso foi descartado como *"um problema do Windows"* e o suporte foi negado — nenhum caminho de escalonamento foi oferecido. Tudo abaixo existe porque essa porta se fechou. Como as capturas mostram, não é um problema do Windows: o Windows é o lado que detecta e recupera corretamente a violação de protocolo do dispositivo.

---

## Sintoma

Com áudio tocando pelo adaptador PS Link, o áudio do Windows morre no sistema inteiro: todo app fica mudo, o slider de volume mexe mas nada toca, e só matar o `audiodg.exe` (ou reiniciar os serviços de áudio / alternar qualquer efeito que reconstrua o grafo de áudio) traz o som de volta. Gatilhos: mudar o volume no headset, mutar o mic, mexer em qualquer slider do app da Sony — mas também acontece "espontaneamente" só reproduzindo áudio. No meu caso: mediana de ~7 minutos entre travamentos em uso ativo (48 travamentos numa sessão de ~7 h, todos logados).

Buscas mostram relatos espalhados de "áudio do Pulse Elite caindo a cada poucos segundos no PC" que batem com a forma mais leve dessa falha. A maioria culpa o Windows ou os drivers e segue a vida. O mecanismo abaixo explica a forma leve e a grave.

## Causa raiz (capturada no barramento)

O adaptador é um dispositivo USB composto Full-Speed: uma função de áudio (interfaces 0–2, USB Audio Class 1.0 padrão, driver inbox `usbaudio.sys` da Microsoft) e uma função HID (interface 3, driver inbox `HidUsb`). A interface HID declara um endpoint interrupt IN 0x81 com `wMaxPacketSize = 64`, e seu maior input report (ID 0xB0) com 64 bytes.

**Toda vez que o estado do headset muda** — botão de volume, mute do mic, slider de sidetone, preset de EQ (cada um atualiza o report de estado 0xB0 de 64 bytes) — **o firmware empurra uma rajada de 256 bytes (4 cópias idênticas concatenadas do report 0xB0) por esse endpoint de 64 bytes.**

Isso é violação dura do protocolo USB, e o host marca exatamente assim:

```
completion do interrupt IN, EP 0x81, tamanho=256, USBD_STATUS = 0xC0000012 (BABBLE_DETECTED)
→ URB pendente cancelada (0xC0010000)
→ URB_FUNCTION_ABORT_PIPE
→ URB_FUNCTION_SYNC_RESET_PIPE_AND_CLEAR_STALL
```

Capturei essa sequência idêntica **9 vezes numa única sessão** — uma por aperto de botão / movimento de slider, deterministicamente.

Uma nota pra quem vai dizer "babble costuma ser falha elétrica/de cabo/do controlador do host, não do device": aqui não é. É **determinístico** (um por evento de mudança de estado, nunca aleatório), o tamanho reportado é **exatamente 4× o tamanho de report que o próprio device declara** (256 = 4 × 64), e não um comprimento truncado ou lixo, e o payload são **quatro cópias limpas e alinhadas do report 0xB0**, não dados corrompidos. Babble elétrico não produz quatro repetições byte-a-byte de um report HID específico sob demanda. Isto é o firmware emitindo deliberadamente uma transferência de tamanho excessivo.

**Por que mata o áudio:** sem áudio tocando, a recuperação (abort/reset) é inofensiva (as 9 ocorrências acima não tiveram efeito audível — o stream de áudio estava fechado na hora). Mas quando o stream isócrono **está** ativo no mesmo dispositivo Full-Speed, a recuperação do babble desorganiza o serviço USB do device e o stream de áudio para de completar:

- Em modo WASAPI **exclusivo** (motor de áudio do Windows fora do caminho): o player morre com *"Unrecoverable playback error: Waiting for hardware timed out"* — capturei um babble no **segundo exato** do aperto de botão que matou o stream, seguido ~2 s depois pelo teardown do stream (`SET_INTERFACE alt 0`). O canal de eventos de glitch do Windows registrou **zero** eventos — provando que a falha acontece abaixo do motor de áudio.
- Em modo **compartilhado** normal: o stream estagnado aparece como `KS "BASE Output Unexpected Buffer Completed"`, depois uma tempestade de ~130 glitches/s (`Microsoft-Windows-Audio/GlitchDetection`), e as threads do `audiodg.exe` entram em deadlock — o travamento do sistema inteiro.

Isso também explica os travamentos "espontâneos" (o device atualiza o report de estado sozinho de vez em quando — ex.: nível de bateria — emitindo a mesma rajada malformada) e por que os botões parecem "gatilhos certeiros" (cada aperto emite uma).

**Confirmação por eliminação (A/B):** com a interface HID desabilitada no host (ver workaround) — o pipe interrupt nunca é pollado e o device fisicamente não consegue transmitir a rajada — o freeze ficou **irreprodutível** sob a mesma martelada de botões que antes matava o áudio em 1–3 minutos, de forma confiável. Testado com playback exclusivo contínuo.

Violação de spec secundária, por completude: o endpoint isócrono de SAÍDA de áudio é declarado **assíncrono** mas não fornece **nenhum endpoint de feedback** (nem explícito, nem implícito; `bSynchAddress = 0`), que a USB 2.0 §5.12.4.2 exige para sinks assíncronos. Não é o gatilho aqui, mas indica o nível de conformidade USB do firmware.

**Atribuição de culpa:** o dispositivo viola a spec; o Windows detecta a violação e recupera o pipe conforme o manual. A única coisa discutivelmente do lado da Microsoft é a severidade — a tempestade de glitches escalar pra deadlock do `audiodg` em vez de um restart gracioso do stream. O firmware presumivelmente foi projetado contra o host stack da própria Sony (PS5 / PlayStation Portal), onde ninguém cobra a declaração HID — no Windows, o driver de classe inbox cobra.

## Limites desta análise (o que eu NÃO provei)

Por honestidade, e pra ninguém precisar apontar isso por mim:

- **O elo causal interno é inferido, não capturado.** Provei que o babble e a morte do áudio são fortemente correlacionados no tempo, e que remover a interface HID remove o freeze. **Não** capturei o mecanismo exato *dentro* dos drivers pelo qual a recuperação de erro do pipe interrupt desorganiza o pipe isócrono de áudio — esse passo é uma inferência fundamentada no timing, na sequência de erro e no resultado do A/B. Também é possível (embora eu considere menos provável) que o babble e o travamento sejam dois sintomas de uma falha mais profunda do device, em vez de um causar o outro. De todo jeito, a conclusão prática se mantém: sem poll do HID, sem freeze.
- **O USBPcap não capturou pacotes isócronos neste sistema.** Então não pude observar diretamente a degradação do timing do stream de áudio no fio; a evidência do lado do áudio vem do ETW (a assinatura de glitch do KS) e da falha no nível do player, correlacionadas por timestamp com o babble. Um analisador USB de hardware fecharia o mecanismo interno de forma definitiva.
- **A culpa é compartilhada, não 100% Sony.** O device viola a spec e é o gatilho raiz, mas dá pra argumentar que os drivers de classe do Windows poderiam isolar um erro de endpoint HID de uma função de áudio não relacionada no mesmo device composto de forma mais graciosa. Atribuo o *gatilho* ao firmware da Sony e a *severidade* (deadlock completo do audiodg vs. um breve restart de stream) em parte a como o Windows lida com ele.
- **Escopo da amostra.** Uma unidade de headset, firmware 1.43, dois laptops, um investigador. As violações de spec são inerentes aos descriptors do device (independentes das minhas máquinas), mas não testei múltiplas unidades de headset nem versões de firmware. Reprodução independente — especialmente em outro firmware — é exatamente o que reforçaria ou delimitaria isto.

Nenhum destes enfraquece o achado principal nem o workaround. Eles marcam a fronteira entre o que é fato capturado e o que é raciocinado a partir dele.

## Por que os botões continuam funcionando sem nada disso

Achado bônus: os botões de volume são processados **inteiramente dentro do headset** (o DSP dele aplica o nível localmente; o report de estado só informa o host). O app da Sony pra PC apenas espelha esse estado no slider de volume do Windows. Ou seja: o workaround abaixo NÃO sacrifica os botões de volume.

## Workaround (sem software, reversível, testado)

Desabilite a interface HID do adaptador, pra que o Windows nunca faça poll do endpoint malformado. O adaptador tem exatamente uma interface HID (interface 3, "MI_03" no hardware ID), então só existe um alvo correto. Feche o app "PlayStation Link" da Sony antes, se estiver rodando (ele segura o dispositivo aberto e faz o disable falhar).

**Opção A — Gerenciador de Dispositivos (a mais fácil de acertar):**

1. Abra o Gerenciador de Dispositivos (`devmgmt.msc`) → menu **Exibir → Dispositivos por contêiner**.
2. Expanda o contêiner **"PlayStation Link Adapter"**. Você verá as entradas de áudio e um único **"Dispositivo de Entrada USB"**.
3. Clique com o direito nesse "Dispositivo de Entrada USB" → **Desabilitar dispositivo**.

(Se preferir a visualização padrão: ele fica em Dispositivos de Interface Humana, no meio de possivelmente vários "Dispositivo de Entrada USB" idênticos. Escolha aquele cujas Propriedades → Detalhes → **IDs de Hardware** mostram `USB\VID_054C&PID_0ECC&MI_03`. O "MI_03" nunca aparece no nome de exibição do dispositivo, só nessa propriedade.)

**Opção B — PowerShell como Administrador (acha pelo ID, sem ambiguidade):**

```powershell
Stop-Process -Name 'PlayStation Link' -Force -ErrorAction SilentlyContinue
Get-PnpDevice | Where-Object InstanceId -like 'USB\VID_054C&PID_0ECC&MI_03*' |
    Disable-PnpDevice -Confirm:$false
```

Pra desfazer qualquer uma das opções depois: clique direito → Habilitar dispositivo, ou o mesmo PowerShell com `Enable-PnpDevice`.

Resultados e custos:

- O áudio continua funcionando (outra interface, intocada). Os botões de volume continuam funcionando (processados no headset). O mecanismo do freeze é fisicamente eliminado.
- Enquanto desabilitado, o app da Sony não enxerga o device (sem ajustes de sidetone/EQ, sem leitura de bateria, sem update de firmware). Reabilite temporariamente pra mudar essas coisas, e desabilite de novo.
- Sobrevive a reboots. Outro adaptador (serial diferente) precisa do disable aplicado mais uma vez.
- **Ressalva descoberta depois:** desabilitar a interface HID inteira também para o poll de controle a ~5 Hz que avisa o headset que há um PC presente. Com ela desabilitada, o headset deixa de auto-sincronizar ao ligar e acaba se desligando sozinho por timeout de presença. Ou seja, o disable puro troca o freeze por uma regressão de conexão. **O app abaixo evita isso** — mantém o poll de presença sem nunca abrir o endpoint que faz babble.

## A correção completa (app open-source)

O workaround de disable mata o freeze mas, como notado, também mata a noção de presença do headset.
A sacada que resolve os dois: o freeze vem do endpoint de **interrupt** (0x81), enquanto a presença vem
de um poll **separado**, a ~5 Hz, no endpoint de **controle** (`GET_REPORT(Feature 0xB0)` no EP0). O
driver HID inbox do Windows acopla os dois — servir a interface abre o pipe de interrupt, e é isso que
deixa o babble acontecer.

Então desacople: rebinde a interface HID do adaptador (MI_03) pra **WinUSB** e controle-a por um pequeno
app em user-space. O WinUSB não abre o pipe de interrupt sozinho, então o endpoint que faz babble **nunca
é pollado** — o freeze é eliminado por construção. O app faz o mesmo poll de controle a ~5 Hz, então **a
presença é preservada** e o headset auto-sincroniza normalmente. Todo controle foi reimplementado pelo
endpoint de controle — volume, mute do mic, sidetone, EQ — mais bateria e estado de conexão ao vivo,
vários decodificados especificamente pra isso. Mesmo mecanismo, dito simples: **mantenha o poll de
controle, nunca abra o pipe de interrupt.**

**→ Pulse Elite Companion — https://github.com/Jprnp/pslink-libusb** (open source, MIT; instalador de um
clique; app de bandeja em inglês / português / espanhol). Validou o mecanismo inteiro em hardware real:
freeze eliminado, presença mantida, todos os controles funcionando, sem software da fabricante.

Uma nota honesta: o Windows x64 recusa instalar um pacote de driver sem assinatura, então o instalador
gera um **certificado auto-assinado e o adiciona à loja de confiança da máquina** pra instalar nosso INF
WinUSB — a mesma concessão que o popular *Zadig* faz internamente. O repo documenta isso e como desfazer
por completo; um driver com assinatura oficial (attestation) é o próximo passo planejado.

## Reprodução (pra quem quiser verificar)

0. Confira a versão do seu firmware primeiro (ver a nota do `REV_xxxx` no topo). Tudo abaixo foi verificado no **1.43**; uma versão diferente pode não reproduzir, e saber quais versões reproduzem ou não já é dado valioso por si só.
1. Toque áudio contínuo pelo adaptador PS Link (qualquer modo; WASAPI exclusivo torna a morte legível como um erro limpo do player).
2. Aperte volume +/− no headset repetidamente (a cada ~10 s). Tempo-até-travar típico aqui: menos de 3 minutos.
3. Pra prova no nível do fio: capture com USBPcap no root hub do adaptador durante o passo (2); filtre o endereço do device; observe completions `0xC0000012` (256 bytes) no EP 0x81 a cada aperto, e correlacione a morte do áudio com uma delas. (Nota: o USBPcap não capturou pacotes isócronos no meu sistema — não é necessário; a sequência babble + reset do pipe e a correlação temporal aparecem mesmo assim.)
4. Depois desabilite o filho HID MI_03 e repita o passo (2) — o freeze não deve reproduzir.

## O que a Sony deveria corrigir

Enquadrar corretamente as transferências do interrupt IN: entregar o report 0xB0 como uma única transferência de ≤64 bytes por poll (ou declarar `wMaxPacketSize`/tamanho de report maiores de forma consistente). Descrição de uma linha; só firmware; sem mudança de hardware. Corrigir o endpoint de feedback ausente também seria bem-vindo.

Vale notar o quão dispensável é a transferência problemática: o mesmo estado 0xB0 (botões, volume, mute do mic) **também** é servido pelo control endpoint via `GET_REPORT(Feature 0xB0)` — é assim que o próprio app de PC da Sony lê o estado, com poll a ~5 Hz. A rajada malformada de 256 bytes no interrupt é uma notificação redundante da qual nada essencial depende. Dava pra dimensioná-la corretamente, ou removê-la por completo, sem perder função nenhuma.

## Sinais de que o port pra PC foi baixa prioridade

Nenhum destes é o bug — o bug é o babble lá em cima. Mas, juntos, sugerem que o lado Windows recebeu pouca atenção, o que dá contexto pra como algo tão reproduzível foi parar em produção:

- **O firmware viola a spec USB em dois lugares independentes.** O endpoint interrupt de 64 bytes que transmite 256 (o freeze), e o endpoint async de saída de áudio que vem sem endpoint de feedback nenhum (`bSynchAddress = 0`, sem feedback explícito nem implícito), que a USB 2.0 §5.12.4.2 exige pra sinks assíncronos. Os dois são visíveis nos próprios descriptors do dispositivo.
- **O pacote de driver assinado (WHQL) ainda carrega um comentário de teste de desenvolvedor.** No Extension INF da Sony (`PlayStationLink.inf`), logo acima da linha que define o friendly name do dispositivo:
  ```
  [Dev_AddReg]
  ; Just to test that it installed
  HKR,,FriendlyName,, "PlayStation Link"
  ```
  Um "só pra testar" esquecido é inofensivo, mas não é o tipo de coisa que se espera sobreviver até um release de produção assinado, distribuído a todo usuário de PC via Windows Update.

Pra ser justo, o mesmo pacote acerta em algumas coisas (`DriverIsolation`, `PnpLockDown`). O ponto não é que a Sony seja incompetente; é que o port pra PC parece ter sido validado de leve — consistente com um dispositivo projetado pro PS5, onde a Sony controla o host e nada disso apareceria.

---

## A trajetória da investigação (como chegamos aqui)

Isso não foi achado numa tarde. A trilha, em ordem — incluindo as curvas erradas, porque elas fazem parte da evidência:

1. **Maio/2026 — primeiro mergulho.** Depois de meses de travamentos: tracing ETW de áudio + dump de memória do `audiodg.exe` travado. Achada a assinatura da falha (`BASE Output Unexpected Buffer Completed` → tempestade de glitches → threads do motor em deadlock). Primeiro suspeito — os efeitos de áudio da Realtek (APOs) — **inocentado**: removê-los não mudou nada (desabilitar efeitos só "consertava" porque reconstrói o grafo de áudio, que é o que qualquer recuperação faz). Suporte da PlayStation contatado; caso descartado como "problema do Windows"; suporte negado.
2. **Maio–julho/2026 — a era do band-aid.** Construí um watchdog que detecta a tempestade de glitches e reinicia o motor de áudio automaticamente (~1 s de recuperação). Foram quatro gerações (polling do canal de eventos; imunidade a saltos do relógio do sistema; hotkey global pra freezes que o canal nem loga; fallback de restart de serviço pra quando o próprio `audiosrv` trava). Tornou a vida suportável — mediana de ~7 minutos entre travamentos em uso ativo — mas é recuperação, não cura.
3. **21/jul — reconhecimento completo do adaptador.** Dump fresco dos descriptors USB (todas as interfaces/endpoints, as duas configurations), o report descriptor HID completo de 951 bytes, e a decodificação ao vivo do protocolo HID vendor: report de estado `0xB0` (bitfield dos botões, nível de volume do headset 0–13, estado do mute do mic), report de config `0xD0` (nível de sidetone, presets de EQ), confirmados byte a byte no barramento. Descoberto que o app da Sony nunca recebe *input reports* dos botões — ele faz polling de `GET_REPORT(0xB0)` ~4×/s. Primeiro avistamento das rajadas malformadas de 256 bytes no interrupt — anotadas como curiosidade, ainda não entendidas como a arma do crime.
4. **Resultado negativo chave:** matar o app da Sony inteiro **não** parou os travamentos. Isso inocentou o *software* da Sony e toda ideia de mitigação tipo "higienizar o tráfego" — o problema tinha que morar no device ou abaixo dele.
5. **Hipótese de clock drift.** O dump dos descriptors mostrou que o endpoint async de saída de áudio **não tem endpoint de feedback nenhum** (violação de spec) — cenário estrutural de drift de clock, e por um tempo a teoria líder. Explicava os travamentos passivos e a "janela de graça" pós-recuperação. Estava errada sobre o mecanismo principal, mas motivou o experimento decisivo.
6. **Noite de 21/jul, virando a madrugada do 22 — a noite decisiva.** Playback WASAPI exclusivo (motor de áudio fora do caminho) morreu num aperto de volume em menos de um minuto — com o canal de glitches do Windows **completamente silencioso**. Captura de barramento da morte seguinte: `BABBLE_DETECTED` no EP 0x81 no segundo exato do aperto fatal. Re-análise da captura do dia anterior: a mesma sequência babble + abort/reset em **todo** evento de botão/slider — inofensiva na época, porque não havia áudio tocando. Confirmação final: com a interface HID desabilitada (pipe interrupt nunca pollado → babble fisicamente impossível), a martelada de botões não conseguiu mais matar o áudio — e os botões de volume continuaram funcionando, revelando que sempre foram processados dentro do headset.
7. **Janela de graça explicada:** depois de cada crash, o pipe babbleado fica morto até o driver HID reabri-lo; enquanto está morto, sem IN tokens → sem babble possível → botões "seguros" por alguns segundos. Todo comportamento observado agora tem um único mecanismo.
8. **22/jul — do diagnóstico à cura.** Transformei o achado num produto funcional. Confirmei a peça que faltava — que a presença depende do poll de controle, não do pipe de interrupt — cortando o disable puro e vendo o headset cair. Então rebindei o MI_03 pra WinUSB, rodei o poll de controle em user-space e nunca abri o pipe de interrupt: freeze eliminado, presença mantida. Decodifiquei os comandos host→device restantes pelo endpoint de controle capturando o app da Sony e sondando com verificação por read-back — volume (`0xD0` máscara `0x02`), mute do mic (`0xD0` máscara `0x01`), bateria (report `0x82`, byte numa escala 0–15), estado de conexão (report `0xB0` byte 39 bit0). Prototipei e validei cada passo em Python no hardware real, e depois montei um app de bandeja em C#/.NET em volta. Open-source como *Pulse Elite Companion* (link acima).

Ferramentas: USBPcap + Wireshark, UsbTreeView, ETW do Windows (`Microsoft-Windows-Audio/GlitchDetection`), WinDbg (análise de dump), Python (parsers próprios de pcap/USBPcap, enumeração HID e teste de acesso via ctypes, monitoramento Core Audio via pycaw).

## Apêndice — evidências-chave capturadas

Sequência de babble numa mudança de estado (repetida identicamente em todo evento de botão/slider; timestamps relativos ao início da captura):

```
48.736  EP 0x81 interrupt IN  cpl  status=C0000012 (BABBLE_DETECTED)  len=256
        payload: b0 50 01 4e b4 be 1a bf ... (4x o report 0xB0 de 64 bytes)
48.737  EP 0x81               cpl  status=C0010000 (CANCELED)         len=0
48.737  URB_FUNCTION_ABORT_PIPE (EP 0x81)
48.934  URB_FUNCTION_SYNC_RESET_PIPE_AND_CLEAR_STALL (EP 0x81)
```

A morte, capturada ao vivo (playback WASAPI exclusivo; segundo aperto de volume da rodada):

```
189.897  EP 0x81 interrupt IN  cpl  status=C0000012  len=256   ← o babble (= o aperto do botão)
189.897  EP 0x81               cpl  status=C0010000  len=0
189.897  ABORT_PIPE / 190.087 RESET_PIPE_AND_CLEAR_STALL
191.942  SET_INTERFACE(if=2, alt=0)                            ← stream de áudio desmontado (~timeout do player)
194.902  EP 0x81 interrupt re-submetido … nunca mais completa
Erro do player: "Unrecoverable playback error: Waiting for hardware timed out"
Canal de glitches do Windows: zero eventos (motor de áudio não envolvido)
```

Re-captura independente (uma sessão posterior, sem áudio tocando) — o babble é determinístico entre sessões. Aqui ele disparou uma vez, no momento em que o headset associou ao adaptador (mudança de estado do dispositivo), e foi inofensivo porque não havia stream ativo:

```
72.640  EP 0x81 interrupt IN  cpl  status=C0000012 (BABBLE_DETECTED)  len=256
72.641  EP 0x81               cpl  status=C0010000 (CANCELED)         len=0
72.641  URB_FUNCTION_ABORT_PIPE (EP 0x81)
72.830  URB_FUNCTION_ABORT_PIPE / reset (EP 0x81)
        ...depois o EP 0x81 ficou mudo pelos ~130 s restantes, com o headset conectado.
        Durante toda a captura, o host fez poll de GET_REPORT(Feature 0xB0) no control
        endpoint a ~5 Hz constantes — é esse poll de controle, não o endpoint de
        interrupt, que a operação normal de fato usa.
```

Declarações dos endpoints (dos descriptors do próprio dispositivo, configuration ativa):

```
EP 0x81  interrupt IN   wMaxPacketSize=64   ← o endpoint babbleado
EP 0x0C  iso OUT  async wMaxPacketSize=196  bSynchAddress=0, nenhum EP de feedback
EP 0x8C  iso IN   sync  wMaxPacketSize=96
Report HID 0xB0 (input): declarado 64 bytes  ← entregue com 256
```

Assinatura da falha em modo compartilhado (ETW, do diagnóstico de maio do mesmo freeze):

```
KS Endpoint Glitch: BASE Output Unexpected Buffer Completed
→ ~130 eventos de glitch/s (Microsoft-Windows-Audio/GlitchDetection, IDs 38/40/41)
→ threads do audiodg.exe estacionadas (WaitForMultipleObjectsEx / EventPairLow) — áudio do sistema morto
```

*As capturas pcap completas, os traces ETW, o dump do audiodg e o dump do report descriptor HID existem e podem ser compartilhados (serial do dispositivo redigido). Feliz em responder perguntas ou ajudar alguém a reproduzir.*
