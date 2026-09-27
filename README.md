# Subaudio Player

**Doppiaggio AI per simulcast anime** — nella tua lingua, sul tuo hardware.

Subaudio Player è un player video standalone che doppiatura il dialogo di una
simulcast (o di qualsiasi video) in tempo quasi reale: incolli l'URL della
pagina della simulcast, premi **Vai**, e guardi l'episodio con il dialogo
parlato nella tua lingua mentre l'audio originale resta udibile sotto
(abbassato). Tutto gira in locale — ASR, traduzione e TTS — **niente cloud,
niente account, niente abbonamento**.

Release corrente: **v0.2.4** · piattaforme: **Windows x64**, **Linux x64**,
**macOS (Apple Silicon)** · [Changelog](https://github.com/iCreil/subaudio-releases/releases)

---

## Cosa è

Un unico programma (un eseguibile Go, nessuna dipendenza da installare) che:

1. **apre la pagina della simulcast** in un browser integrato (o un file
   video locale, o un link diretto),
2. **trova da solo il `<video>`** — funziona anche con login e challenge
   Cloudflare, che vengono ricordati tra un avvio e l'altro,
3. **riproduce il video** e **doppia il dialogo in tempo quasi reale**:
   mentre guardi, ogni battuta viene trascritta, tradotta e parlata nella
   lingua che hai scelto, piazzata sul video con un piccolo ritardo costante;
4. **lascia l'audio originale udibile**: musica di sottofondo ed effetti
   restano, attenuati sotto le righe doppiate (ducking) e ripristinati dopo.

Stessa semantica del simulcast che conosci: il doppiaggio segue l'immagine
con un lag costante, la pausa del video ferma il doppiaggio, il seek lo
ri-ancora da solo.

## Come funziona

```
 <video> nella pagina (o file locale / link diretto)
      │  video.captureStream()  (a livello elemento, PCM mono 16 kHz)
      ▼
 chunk da 4 secondi ──► whisper.cpp (stessa GPU del LLM, o CPU) + Silero VAD ← voce → testo
      │                     (se la VAD scarta un chunk → retry senza VAD:
      │                      il dialogo debole o sopra la BGM non va perso)
      ▼
 LLM sul device scelto (Ternary-Bonsai-2-27B / Qwen3.5)        ← testo → traduzione
      │                     una riga JSON: {traduzione, voce F/M/C, song}
      ▼
 TTS Kokoro-82M (ONNX) + fonemi espeak-ng                      ← testo → voce
      ▼
 Scheduler: ogni riga è piazzata a  t_video + from + lag
            audio originale ducked sotto la riga, ripristinato dopo
```

Scelte tecniche che contano:

- **Cattura a livello elemento** (`video.captureStream()`), non di tab o
  finestra: il routing audio del sistema non viene toccato, quindi l'audio
  originale (musica, effetti) resta nella tua uscita. Se lo stream è
  taintato da CORS, l'app passa automaticamente alla cattura loopback.
- **VAD** (Silero) salta i chunk musicali o rumorosi: le sigle OP/ED non
  vengono "doppiate". Se un chunk viene scartato ma contiene dialogo debole,
  il chunk viene ritentato **senza VAD** e la riga viene recuperata.
- **Traduzione riga per riga** con le ultime righe come contesto: i nomi
  propri restano romanizzati in modo coerente e le voci restano coerenti
  (maschile/femmina) per personaggio.
- **Voci TTS**: italiano, inglese, spagnolo, francese — voce maschile o
  femminile per lingua; il LLM sceglie la voce riga per riga in base al
  dialogo.

## Funzionalità

- **Video automatico**: URL della pagina simulcast (il player trova da solo
  il video e gestisce login/Cloudflare), link diretto (`.mp4`, `.m3u8`, …) o
  percorso locale.
- **Scelta hardware automatica**: al primo avvio l'app elenca CPU e GPU e ti
  chiede dove far girare la traduzione; poi **dimensiona da sola il modello
  LLM** sulla memoria disponibile (vedi [Primo avvio](#primo-avvio-scelta-hardware)).
  La scelta viene ricordata.
- **ASR sulla GPU** (v0.2.3): whisper small condivide la GPU del LLM ed è
  ~10x più veloce della CPU; se dopo il LLM la VRAM non basta, whisper torna
  da solo su CPU.
- **Meno buchi nel dialogo**: VAD permissiva + retry senza VAD sui chunk
  scartati; i chunk solo-musica non fanno più crashare il server.
- **Lingue**: origine `auto`/ja/en/zh/ko/es/fr/de/it, destinazione
  it/en/es/fr — selettore nel pannello, remembered tra un avvio e l'altro.
- **Ducking naturale**: l'originale scende sotto ogni riga doppiata con una
  release lunga (niente "salti" udibili) e torna dopo.
- **Lag costante**: pre-buffering di ~8 s e piazzamento delle righe a
  `t_video + from + lag`; pausa e seek ri-ancorano il doppiaggio da soli.
- **Pannello di stato**: righe doppiate, conteggio *late* (righe in
  ritardo), livello di cattura in dBFS, modalità di cattura.
- **Auto-update in-app**: al lancio verifica le release; su Windows scarica
  solo i file cambiati (delta, ~4 MB), su Linux/macOS lo zip completo;
  rileva i download "impallati" e si riavvia da solo.
- **LLM locale o remoto**: il modello locale viene scaricato al primo avvio
  (2,7–7,2 GB a seconda del device); in alternativa si può puntare a un
  server OpenAI-compatibile in rete (`-llm remote`, utile per sviluppo o
  per macchine senza GPU).
- **Modalità kiosk**: `-auto-start 15s` avvia la simulcast da sola dopo N
  secondi (TV di sala d'attesa, stand, …).

## Download

| Piattaforma | Zip | Note |
|---|---|---|
| Windows x64 | `subaudio-vX.Y.Z-windows-x64.zip` | ~2,0 GB. Win 10 21H2+ / Win 11 |
| Linux x64 | `subaudio-vX.Y.Z-linux-x64.zip` | ~1,1 GB. GTK3 + WebKit2GTK 4.0 (vedi [Requisiti](#requisiti)) |
| macOS (Apple Silicon) | `subaudio-vX.Y.Z-macos-arm64.zip` | ~0,8 GB. GPU via Metal |

Scompatta ed esegui — non c'è un installer. Al primo avvio l'app scarica il
modello LLM per il device scelto (2,7–7,2 GB, vedi
[Primo avvio](#primo-avvio-scelta-hardware)).

Gli zip contengono **solo l'applicazione** (binari, modelli whisper, runtime
CUDA/ONNX, librerie TTS e fonetici) — nessun codice sorgente.

## Primo avvio (scelta hardware)

All'avvio l'app mostra una piccola finestra con **CPU** e tutte le GPU
visibili, e chiede quale device debba eseguire la **traduzione** (il LLM).
La scelta viene ricordata per l'avvio successivo.

L'app dimensiona poi da sola il LLM sulla memoria del device scelto:

| Device scelto | Modello LLM | Contesto |
|---|---|---|
| GPU con ≥ 16 GiB VRAM libera | Ternary-Bonsai-2-27B (ternary 2-bit, distillato da Qwen3.8-27B) | 8192 |
| GPU con ≥ 10 GiB liberi | Ternary-Bonsai-2-27B | 4096 |
| GPU da 8 GiB (es. RTX 2070 Super) | Ternary-Bonsai-2-27B | 2048 |
| GPU ~5,5–7 GiB | Qwen3.5 9B Q4 | 2048 |
| GPU 3–5,5 GiB | Qwen3.5 4B Q4 | 2048 |
| CPU, ≥ 12 GiB RAM | Ternary-Bonsai-2-27B | 2048 |
| CPU, < 12 GiB RAM | Qwen3.5 4B Q4 | 2048 |

Il riconoscimento vocale (**whisper.cpp, modello small**) da v0.2.3
**condivide la GPU del LLM** (default): l'app parte prima il LLM (che sceglie
modello e contesto guardando la VRAM libera) e poi mette whisper sulla stessa
GPU. whisper small occupa ~0,8 GB di VRAM: su una GPU da 8 GiB ci sta
accanto al LLM senza problemi. Se dopo il LLM sulla GPU restano meno di
~900 MiB liberi, whisper torna da solo su CPU.

La finestra di scelta viene saltata se esiste un'opzione sola o se il device
è forzato da flag (`-llm-device`).

## Uso del player

1. Avvia l'app (doppio click su `subaudio.exe` su Windows, `subaudio` su
   Linux/macOS).
2. Nel pannello in basso a sinistra, incolla uno di questi:
   - l'**URL della pagina della simulcast** (la pagina dell'episodio sul
     sito: l'app la apre nel browser integrato e trova da sola il `<video>`;
     login e challenge Cloudflare funzionano e vengono ricordati),
   - un **link diretto al video** (`.mp4`, `.m3u8`, …), oppure
   - un **percorso locale** (es. `C:\video\episodio.mp4`).
3. Premi **Vai**. Il pulsante **Avvia** si abilita quando la pagina caricata
   espone un video.
4. L'app pre-buffera ~8 secondi di doppiaggio con un reader nascosto, poi
   parte il video visibile — niente silenzio all'inizio, e il doppiaggio
   mantiene un piccolo lag costante rispetto all'immagine.

Durante il doppiaggio l'audio originale viene abbassato (ducking) sotto ogni
riga doppiata e ripristinato dopo, così musica di sottofondo ed effetti
restano udibili. Il pannello mostra lo stato corrente (righe doppiate,
conteggio late, livello cattura) e il **selettore lingua** viene ricordato
tra un avvio e l'altro.

Pausa e seek del video ri-ancorano il doppiaggio automaticamente.

## Requisiti

- **Windows**: Win 10 21H2+ o Win 11 (il runtime WebView2 è già installato
  sui sistemi recenti).
- **Linux**: GTK 3 e WebKit2GTK 4.0, es. su Ubuntu/Debian:
  `sudo apt install libgtk-3-0 libwebkit2gtk-4.0-0 libgomp1`
  (testato su Ubuntu 22.04+; CUDA NVIDIA incluso per il LLM).
- **macOS**: Apple Silicon (Intel non supportato in questa release).
- **RAM**: 8 GiB minimo (modalità CPU), 16 GiB consigliati.
- **GPU**: facoltativa — una NVIDIA (o Apple Silicon) rende la traduzione
  molto più veloce; le librerie CUDA 12.4 sono incluse nello zip.
- **Disco**: lo zip più il modello LLM (2,7–7,2 GB a seconda del device).

## Auto-update

Il pannello del player verifica le release pubbliche per una nuova versione.
Quando ce n'è una, appare il pulsante **Scarica e installa**:

- **Windows**: l'updater scarica solo i **file cambiati** (delta, ~4 MB)
  rispetto alla tua versione, li sostituisce e riavvia l'app da solo;
- **Linux / macOS**: scarica lo zip completo della piattaforma e lo
  applica allo stesso modo.

Se il download non riceve byte da 2 minuti (il CDN a volte si impalla senza
dare errore) appare un messaggio chiaro con il bottone **Riprova**.

## Avanzato (command line)

Il binary accetta flag; i default sopra valgono per il player standalone. I
più utili:

```
-l ja|en|zh|ko|es|fr|de|it|auto   lingua di origine (default ja; auto = rilevata)
-t it|en|es|fr                    lingua del doppiaggio (default it)
-lag 0.5                          lag del doppiaggio in secondi
-prebuf 8                         lead di pre-buffering in secondi
-asr-model auto|small|tiny        modello whisper (auto = small)
-asr-gpu auto|cpu|N               GPU per whisper (auto = stessa del LLM; cpu; o indice)
-asr-vad-thresh 0.4               soglia VAD (0.4 = meno buchi)
-llm-device cpu|N                 forza il device del LLM (salta il picker)
-llm-ctx 2048                     forza la dimensione del contesto LLM
-llm remote -llm-url URL -llm-model M -llm-key K
                                  usa un LLM remoto OpenAI-compatibile
-no-pick                          salta la schermata di scelta hardware
-auto-start 15s                   avvia la simulcast da solo dopo N secondi (kiosk)
```

La scelta hardware è salvata in `device.json` accanto al binary; i
login/cookie dei siti aperti sono tenuti dal browser integrato, separati per
piattaforma (Windows: `%APPDATA%\subaudio.exe`, Linux:
`~/.local/share/subaudio`, macOS: `~/Library/Application Support/subaudio`).

## Troubleshooting

- **Niente doppiaggio, "SILENZIO" nel pannello** — la cattura non trova
  audio sull'elemento video (stream taintato da CORS). L'app passa
  automaticamente alla cattura loopback; verifica che il dispositivo di
  uscita sia quello che riproduce il video.
- **Doppiaggio più lento del tempo reale** — hai scelto la CPU o un modello
  piccolo. Riavvia e scegli la GPU. Su CPU 8-core la pipeline va ~3x più
  lenta del tempo reale — usabile, ma la GPU è consigliata.
- **Primo avvio lento** — sta scaricando il modello LLM (il progresso è
  mostrato nella finestra del picker).
- **Lingua rilevata sbagliata** — imposta la lingua di origine esplicitamente
  nel pannello (la rilevazione `auto` è instabile su chunk brevi con musica).
- **Una riga "traduzione fallita"** — il LLM ha risposto in un formato non
  parsabile per quella riga; le righe successive riprendono da sole.

## Asset di questa repo

| Asset | Contenuto |
|---|---|
| `subaudio-vX.Y.Z-<piattaforma>.zip` | App completa (binari, modelli, librerie) |
| `subaudio-vX.Y.Z-delta-from-vA.B.C.zip` | **Solo Windows**: i file cambiati da vA.B.C (per l'auto-update) |
| `manifest.json` | Versione, nomi asset, dimensioni e sha256 |

Ogni release GitHub è una versione completa; `latest` è sempre la versione
corrente. Il codice sorgente è in [iCreil/subaudio](https://github.com/iCreil/subaudio) (privata).
