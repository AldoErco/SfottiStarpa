# Handoff Tecnico: Diagnosi SfottiStarpa su Windows 11 (Utente "Ufo")

> **Scopo di questo documento:** Fornire a un'istanza esterna di ChatGPT (che assiste direttamente l'utente non tecnico **"Ufo"** su Windows 11) tutte le informazioni tecniche verificate sul codice sorgente, sull'ambiente runtime e sull'installer di **SfottiStarpa**, senza che tale istanza abbia accesso al repository.
>
> **Regola di sicurezza per l'assistente di Ufo:** Durante la diagnosi **NON** proporre mai di disabilitare Microsoft Defender Antivirus, SmartScreen, Smart App Control (Controllo intelligente delle app), UAC o altre protezioni di sistema.

---

## 0. Sintesi del Problema Segnalato

- **Utente:** "Ufo" (utente non tecnico).
- **Sistema Operativo:** Windows 11.
- **Sintomo:** L'installazione di **SfottiStarpa** sembra completarsi regolarmente, ma quando Ufo prova ad avviare l'applicazione tramite il collegamento (Menu Start o Desktop), **Windows mostra un avviso secondo cui il software non è sicuro o non è attendibile** (testo esatto del messaggio non ancora acquisito).

---

## 1. Architettura Applicativa

### 1.1 Scopo e funzionamento generale (`FATTO VERIFICATO`)
- **SfottiStarpa** (identificato internamente negli script di supporto anche come **RoopBox**) è una distribuzione pacchettizzata e preconfigurata per Windows di **`roop-unleashed` versione `4.3.1`** (rif. `app/roop-unleashed/roop/metadata.py`).
- È un'applicazione Python per **face-swapping** e **face-enhancement** (deepfake su immagini, video e webcam live) dotata di interfaccia web locale basata su **Gradio**.
- All'avvio, lo script Python lancia un server web locale (Uvicorn / FastAPI / Gradio) su `http://127.0.0.1:7860` (o sulla prima porta libera disponibile, poiché `server_port: 0` in `app/roop-unleashed/config.yaml`) e apre automaticamente il browser predefinito di Windows (`launch_browser: True` di default in `app/roop-unleashed/settings.py`).
- **Nota sui tempi di avvio:** L'avvio iniziale mantiene aperta una finestra console nera (`cmd.exe` / `python.exe`) per circa **30–120 secondi** mentre vengono importati PyTorch, ONNX Runtime, InsightFace e Gradio prima che si apra il browser web (comportamento esplicitamente descritto anche nel messaggio finale dell'installer).

### 1.2 File Python principale e moduli (`FATTO VERIFICATO`)
Tutti i file applicativi si trovano in `{app}\app\roop-unleashed\` (dove `{app}` è di default `C:\SfottiStarpa`):

| File relativo a `{app}` | Ruolo verificato nel codice |
| :--- | :--- |
| `app\roop-unleashed\run.py` | **Entrypoint principale.** Importa `from roop import core` ed esegue `core.run()`. |
| `app\roop-unleashed\roop\core.py` | Gestisce gli argomenti CLI (`--server_share`, `--cuda_device_id`), esegue `pre_check()` (verifica Python `>= 3.9`, controlla la presenza dei modelli in `../models` e di `ffmpeg` nel `PATH`), carica `Settings('config.yaml')`, imposta i limiti di memoria tramite `ctypes.windll.kernel32.SetProcessWorkingSetSize` (solo se `memory_limit > 0`) e invoca `ui.main.run()`. |
| `app\roop-unleashed\settings.py` | Classe `Settings` che legge/salva `config.yaml` (tema, porta, provider `cuda`/`cpu`, codec video, thread, `use_os_temp_folder`, `launch_browser`). |
| `app\roop-unleashed\config.yaml` | Configurazione distribuita: `provider: cuda`, `force_cpu: false`, `max_threads: 2`, `memory_limit: 0`, `server_name: ''`, `server_port: 0`, `server_share: false`, `use_os_temp_folder: false`. |
| `app\roop-unleashed\ui\main.py` | Funzione `prepare_environment()` (crea le cartelle `output/` e `temp/` nella working directory `app\roop-unleashed` e redirige `TEMP`, `TMP` e `GRADIO_TEMP_DIR` su `app\roop-unleashed\temp`, disabilita la telemetria con `GRADIO_ANALYTICS_ENABLED=0`) e funzione `run()` (verifica CUDA con fallback automatico su `cpu` se `torch.cuda.is_available() == False`, costruisce l'interfaccia Gradio e chiama `ui.queue().launch(inbrowser=launch_browser, ...)`). |
| `app\roop-unleashed\roop\face_util.py` | Inizializza `insightface.app.FaceAnalysis(name="buffalo_l", root=...)` con i modelli in `app\roop-unleashed\models\buffalo_l\`. |
| `app\roop-unleashed\roop\ProcessMgr.py` | Gestisce la pipeline di elaborazione frame per immagini e video e carica i plugin in `roop\processors\`. |
| `app\roop-unleashed\roop\util_ffmpeg.py` e `ffmpeg_writer.py` | Invocano l'eseguibile `ffmpeg` per estrazione frame, codifica video, creazione GIF e ripristino traccia audio. |
| `app\roop-unleashed\roop\utilities.py` | Funzioni di utilità, download condizionale modelli (`conditional_download`), gestione cartelle temporanee e apertura file/cartelle su Windows tramite `os.startfile()`. |
| `app\roop-unleashed\installer\installer.py` e `windows_run.bat` | Script di installazione upstream di `roop-unleashed` (basati su Miniconda). **NON vengono usati** dall'avvio di SfottiStarpa, che usa invece `run_roop.bat` nella root `{app}`. |

### 1.3 Versione Python: richiesta vs distribuita (`FATTO VERIFICATO`)
- **Versione minima richiesta dal codice:** **Python >= 3.9** (verificato in `app/roop-unleashed/roop/core.py`, righe 131–133: `if sys.version_info < (3, 9): update_status('Python version is not supported - please upgrade to 3.9 or higher.')`). Lo script upstream `app/roop-unleashed/installer/windows_run.bat` (riga 54) prevede `python=3.10`.
- **Versione effettivamente distribuita nell'installer:** **Python 3.10.19 (64-bit)** pacchettizzato da `conda-forge` (`python-3.10.19-hc20f281_2_cpython`, verificato in `env/conda-meta/python-3.10.19-hc20f281_2_cpython.json` e `env/conda-meta/history`).

### 1.4 Librerie e dipendenze (`FATTO VERIFICATO`)
- **Dipendenze dichiarate** in `app/roop-unleashed/requirements.txt`:
  - `numpy==1.24.2`, `gradio`, `fastapi<0.113.0`, `opencv-python==4.9.0.80`, `onnx==1.16.0`, `insightface==0.7.3`, `albucore==0.0.16`, `psutil==5.9.6`, `tqdm==4.66.4`, `ftfy`, `regex`, `pyvirtualcam`, `pydantic==2.10.6`.
- **Principali pacchetti effettivamente preinstallati** in `{app}\env\Lib\site-packages\` (verificati dalle directory `.dist-info`):
  - **Deep Learning / GPU:** `torch-2.5.1+cu121`, `torchvision-0.20.1+cu121`, `torchaudio-2.5.1+cu121`, `onnxruntime_gpu-1.17.1`, `onnx-1.16.0`, `insightface-0.7.3`.
  - **Web UI / Server:** `gradio-5.1.0`, `gradio_client-1.4.0`, `fastapi-0.112.4`, `starlette-0.38.6`, `uvicorn-0.38.0`, `pydantic-2.10.6`, `pydantic_core-2.27.2`, `websockets-12.0`, `httpx-0.28.1`.
  - **Computer Vision / Scientific:** `opencv_python-4.9.0.80`, `opencv_python_headless-4.11.0.86`, `numpy-1.24.2`, `scipy-1.15.3`, `scikit_image-0.25.2`, `scikit_learn-1.7.2`, `pillow-10.4.0`, `albumentations-1.3.1`, `albucore-0.0.16`, `imageio-2.37.2`, `pyvirtualcam-0.14.0`.
  - **Utility:** `huggingface_hub-0.24.7`, `hf_xet-1.2.0`, `psutil-5.9.6`, `ffmpy-1.0.0`, `pyyaml-6.0.3`, `tqdm-4.66.4`, `requests-2.32.5`, `cython-3.2.1`.
- Nella cartella `{app}\wheels\torch\` sono inoltre presenti i file `.whl` offline di PyTorch 2.5.1+cu121 e relative dipendenze (~2.4 GB), non utilizzati a runtime ma inclusi nel pacchetto installato.

### 1.5 Componenti esterni, eseguibili e DLL (`FATTO VERIFICATO`)
1. **Interprete Python e runtime C/C++ in `{app}\env\`:**
   - `env\python.exe` (93.184 byte) e `env\pythonw.exe` (91.648 byte).
   - `env\python310.dll`, `env\python3.dll`, `env\zlib.dll`.
   - Runtime Microsoft Visual C++ e Universal CRT distribuiti direttamente in `env\`: `vcruntime140.dll`, `vcruntime140_1.dll`, `vcruntime140_threads.dll`, `msvcp140.dll`, `msvcp140_1.dll`, `msvcp140_2.dll`, `concrt140.dll`, `vcomp140.dll`, `vccorlib140.dll`, `vcamp140.dll`, `ucrtbase.dll` e 40 DLL `api-ms-win-*.dll` (`conda-forge::ucrt-10.0.26100.0` e `vc14_runtime-14.44.35208`).
   - Numerose estensioni native `.pyd` e `.dll` in `env\DLLs\`, `env\Library\bin\` e `env\Lib\site-packages\` (inclusi i binari CUDA/cuDNN di `torch\lib\` e `onnxruntime\capi\`).
2. **FFmpeg statico in `{app}\ffmpeg\bin\`:**
   - `ffmpeg\bin\ffmpeg.exe`, `ffplay.exe`, `ffprobe.exe` (build statica Windows 64-bit `2025-11-17-git-e94439e49b-full_build-www.gyan.dev`, verificata in `ffmpeg/README.txt`).
3. **Uninstaller Inno Setup nella root `{app}\`:**
   - `unins000.exe` (3.832.243 byte) e `unins000.dat` (11.811.181 byte).
4. **Residuo installer upstream (non usato):**
   - `app\roop-unleashed\installer\installer_files\miniconda_installer.exe`.

### 1.6 Processi secondari, download e accessi alla rete (`FATTO VERIFICATO`)
- **Processi secondari generati:**
  1. `cmd.exe` (interprete batch che esegue `run_roop.bat`).
  2. `{app}\env\python.exe` (processo principale dell'applicazione).
  3. Browser predefinito di sistema, lanciato da Gradio quando il server locale è pronto su `http://127.0.0.1:7860`.
  4. `{app}\ffmpeg\bin\ffmpeg.exe`, invocato tramite `subprocess.check_output` (`roop/util_ffmpeg.py`) e `subprocess.Popen` (`roop/ffmpeg_writer.py`) durante l'elaborazione di video/GIF.
  5. `os.startfile(...)` (`roop/utilities.py:273`) se l'utente clicca sui pulsanti della UI per aprire la cartella di output.
- **Accessi alla rete e download:**
  1. **Server HTTP locale:** Uvicorn/Gradio apre una porta TCP su `127.0.0.1` (di default `7860`). Poiché in `config.yaml` `server_name: ''` e `server_share: false`, il server ascolta **solo in locale (loopback)** e non crea tunnel pubblici Gradio.
  2. **Controllo modelli all'avvio (`pre_check()` in `roop/core.py:135-153`):** Il codice chiama `util.conditional_download()` con URL verso `https://huggingface.co/countfloyd/deepfake/resolve/main/...` e `https://github.com/csxmli2016/DMDNet/releases/download/v1/DMDNet.pth`. Tuttavia, `conditional_download()` (`roop/utilities.py:197`) scarica un file **solo se non esiste già** nel percorso locale. **Nel pacchetto SfottiStarpa tutti i modelli richiesti da `pre_check()` e la cartella `buffalo_l` sono già presenti in `app\roop-unleashed\models\`** (~4,3 GB totali: `inswapper_128.onnx`, `GFPGANv1.4.onnx`, `DMDNet.pth`, `GPEN-BFR-512.onnx`, `restoreformer_plus_plus.onnx`, `xseg.onnx`, `CLIP/rd64-uni-refined.pth`, `CodeFormer/CodeFormerv0.1.onnx`, `Frame/deoldify_artistic.onnx`, `Frame/deoldify_stable.onnx`, `Frame/isnet-general-use.onnx`, `Frame/real_esrgan_x4.onnx`, `Frame/real_esrgan_x2.onnx`, `Frame/lsdir_x4.onnx`, più `buffalo_l/*.onnx`). Pertanto **all'avvio non viene effettuato alcun download da Internet** se l'installazione è integra.
  3. **Possibile download opzionale a runtime:** Se l'utente attiva il masking testuale CLIP (`clip/clipseg.py:250` → `clip/clip.py:116`), la libreria `clip` può scaricare il modello `ViT-B-16.pt` nella cartella utente `C:\Users\<utente>\.cache\clip`.

---

## 2. Installer

### 2.1 Tecnologia e artefatti di setup (`FATTO VERIFICATO`)
- **Tecnologia:** **Inno Setup versione `6.6.1` (64-bit)** (verificato dall'header `Inno Setup Setup Data (6.6.1)` nell'eseguibile di setup e dall'intestazione `Inno Setup Uninstall Log (b) 64-bit` in `unins000.dat`).
- **Pacchetto di setup distribuito:** Setup multi-volume composto da:
  - `PrestoVoglioSfottereStarpa.exe` (~7,19 MB, launcher Inno Setup)
  - `PrestoVoglioSfottereStarpa-1.bin` .. `PrestoVoglioSfottereStarpa-6.bin` (6 volumi dati da ~1,57 GB ciascuno, totale ~9,4 GB).
- **Metadati Inno Setup estratti dallo stream header di `PrestoVoglioSfottereStarpa.exe`:**
  - `AppName`: `SfottiStarpa`
  - `AppPublisher`: `Bank of Atlas`
  - `AppVersion`: `1.0-a.prova.di.ufo`
  - `ArchitecturesAllowed`: `x64compatible`
- **Nome e posizione dello script `.iss` sorgente:** `NON VERIFICATO` (il file sorgente `.iss` non è incluso nella directory installata; tutte le direttive Inno Setup riportate di seguito sono state estratte e verificate direttamente dallo stream header compilato di `PrestoVoglioSfottereStarpa.exe` e dal registro di disinstallazione `unins000.dat`).

### 2.2 Directory di destinazione dell'installazione (`FATTO VERIFICATO`)
- **Directory predefinita (`DefaultDirName`):** `C:\SfottiStarpa`
- **Vincolo forzato dallo script Pascal (`[Code]`) dell'installer:**
  - L'installer mostra nella pagina di selezione cartella l'avviso:
    > *"ATTENZIONE UTENTE UFO: per sfottere Starpa devi installare in un percorso senza spazi (es. "C:\SfottiStarpa" -> OK mentre invece "C:\Program Files\SfottiStarpa" -> KO!!)"*
  - La funzione `NextButtonClick` nello script Pascal (`IFPS`) controlla la presenza di spazi nel percorso scelto (`Pos(' ', ...)`). Se il percorso contiene uno spazio, mostra un popup di errore (`"Devi scegliere un percorso senza spazi, ad es. C:\SfottiStarpa."`) e **impedisce di proseguire**.
  - Di conseguenza, l'installazione di Ufo si trova in `C:\SfottiStarpa` (o in un altro percorso privo di spazi scelto da Ufo, ad es. su un altro disco come `D:\SfottiStarpa`).

### 2.3 File e directory copiati e operazioni durante il setup (`FATTO VERIFICATO`)
1. **Fase pre-copia / durante la copia (`[Code]`):**
   - L'installer estrae nella directory temporanea `{tmp}\video\` un player VLC portatile (`{tmp}\video\vlc\vlc.exe` e relativi plugin) e il video `{tmp}\video\lord_of_the_rosiko.mp4`, avviandolo a schermo intero (`--fullscreen --no-video-title-show --play-and-exit`) mentre copia i ~15+ GB di file.
2. **Albero installato su Windows 11 nativo in `{app}` (es. `C:\SfottiStarpa`):**
   - `{app}\run_roop.bat` (script di avvio standard Windows)
   - `{app}\run_roop_debug.bat` (script di avvio diagnostico Windows con `pause` finale)
   - `{app}\check_env.ps1` (script PowerShell di verifica post-installazione)
   - `{app}\SfottiStarpa.ico` (icona dei collegamenti Windows)
   - `{app}\unins000.exe` e `{app}\unins000.dat` (disinstallatore Inno Setup 64-bit)
   - `{app}\app\roop-unleashed\...` (codice Python + `config.yaml` + `models\...`)
   - `{app}\env\...` (ambiente Windows 64-bit Python 3.10.19 pre-costruito)
   - `{app}\ffmpeg\bin\ffmpeg.exe` (più `ffplay.exe`, `ffprobe.exe`, build Windows 64-bit)
   - `{app}\wheels\torch\...` (wheel Windows `.whl` `cp310-win_amd64` di backup)
   - *(Nota: sulla macchina Linux Nobara da cui è stata eseguita questa analisi la cartella è collegata a un ambiente Wine/Bottles e contiene anche aggiunte locali per Linux come `env_linux\`, `wheels\torch_linux\`, `run_roop.sh` e `SfottiStarpa.desktop`; tali file Linux/Wine **NON sono presenti** nell'installer Windows `PrestoVoglioSfottereStarpa.exe` né sul PC Windows 11 nativo di Ufo).*
3. **Esecuzione automatica di verifica in `[Run]` a fine copia:**
   - Prima di mostrare la pagina finale del setup, Inno Setup esegue:
     ```text
     powershell.exe -ExecutionPolicy Bypass -File "{app}\check_env.ps1"
     ```
     con messaggio di stato: *"Porta pazienza qualche minuto mentre verifico che sia tutto a posto per poter sfottere per bene Starpa..."*
   - Lo script `check_env.ps1` verifica l'esistenza di `env`, `env\python.exe`, `app\roop-unleashed`, `ffmpeg\bin\ffmpeg.exe`, poi scrive un file temporaneo `%TEMP%\roopbox_env_test.py` e lo esegue con `{app}\env\python.exe` per verificare `import torch` e `import huggingface_hub`.
4. **Avvio opzionale a fine setup (`[Run]` postinstall):**
   - Nella schermata finale del wizard è presente la spunta `"Avvia SfottiStarpa"` che esegue `{app}\run_roop.bat`.

### 2.4 Modalità di distribuzione di Python (`FATTO VERIFICATO`)
- Python **non** viene installato tramite l'installer ufficiale di Python.org né viene scaricato durante il setup.
- L'ambiente è stato pre-creato sulla macchina di build con Micromamba (`micromamba create -p C:\RoopBox\env python=3.10 -y` in data `2025-11-21`, verificato in `env/conda-meta/history`) e viene **copiato integralmente come cartella autocontenuta** in `{app}\env\`.

### 2.5 Modifiche a registro, PATH, variabili d'ambiente e privilegi (`FATTO VERIFICATO`)
- **Privilegi richiesti dall'installer:** Nel manifest PE esterno di `PrestoVoglioSfottereStarpa.exe` compare `<requestedExecutionLevel level="asInvoker" uiAccess="false"/>`, ma la configurazione interna di Inno Setup ha `PrivilegesRequired=admin` (`prAdmin`, con messaggio *"Per installare questo programma sono richiesti privilegi di amministratore"*), per cui il setup richiede l'elevazione amministrativa (UAC) e installa i collegamenti nelle cartelle di sistema per tutti gli utenti (`CommonPrograms` e `CommonDesktop`).
- **Registro di sistema:** Viene creata solo la chiave di disinstallazione standard di Inno Setup:
  `HKLM\Software\Microsoft\Windows\CurrentVersion\Uninstall\SfottiStarpa_is1` (verificato in `unins000.dat`).
- **PATH e variabili d'ambiente globali:** Nessuna modifica permanente al `PATH` di Windows o alle variabili d'ambiente di sistema/utente.
- **Associazioni file / Permessi NTFS:** Nessuna associazione di estensioni file e nessuna regola ACL NTFS personalizzata oltre ai permessi ereditati dalla directory padre al momento della creazione da parte del processo installer elevato.

### 2.6 Firme digitali Authenticode (`FATTO VERIFICATO`)
L'analisi dell'header PE (`IMAGE_DIRECTORY_ENTRY_SECURITY`) conferma che **nessuno degli eseguibili è firmato digitalmente** (`cert_rva=0, cert_size=0`):
- `PrestoVoglioSfottereStarpa.exe` (installer): **NON FIRMATO**
- `unins000.exe` (disinstallatore): **NON FIRMATO**
- `env\python.exe` e `env\pythonw.exe`: **NON FIRMATI** (binari `conda-forge` privi di firma Authenticode)
- `ffmpeg\bin\ffmpeg.exe`: **NON FIRMATO** (build statica `gyan.dev` priva di firma Authenticode)
- `run_roop.bat`, `run_roop_debug.bat`, `check_env.ps1`: **NON FIRMATI**

---

## 3. Meccanismo di Avvio

### 3.1 Catena completa di avvio (`FATTO VERIFICATO`)

```text
[Collegamento Windows (.lnk)]
   ├── Menu Start: C:\ProgramData\Microsoft\Windows\Start Menu\Programs\SfottiStarpa\Sfotti subito Starpa.lnk
   └── Desktop:    C:\Users\Public\Desktop\Sfotti Starpa.lnk
        │
        ▼
[Target del collegamento]
   C:\SfottiStarpa\run_roop.bat   (Working Directory: C:\SfottiStarpa)
        │  (Windows avvia: C:\Windows\System32\cmd.exe /c "C:\SfottiStarpa\run_roop.bat")
        ▼
[Operazioni di run_roop.bat]
   1. set ROOT=%~dp0                           (es. C:\SfottiStarpa\)
   2. set PATH=%ROOT%ffmpeg\bin;%PATH%         (aggiunge ffmpeg\bin in testa al PATH del processo)
   3. set PYTHON_EXE=%ROOT%env\python.exe
   4. set APP_DIR=%ROOT%app\roop-unleashed
   5. Verifica esistenza di %PYTHON_EXE%, %ROOT%ffmpeg\bin\ffmpeg.exe e %APP_DIR%\run.py
   6. cd /d "%APP_DIR%"                        (cambia Working Directory in C:\SfottiStarpa\app\roop-unleashed)
   7. "%PYTHON_EXE%" run.py
        │
        ▼
[Interprete Python: C:\SfottiStarpa\env\python.exe]
   Esegue: C:\SfottiStarpa\app\roop-unleashed\run.py
        │
        ▼
[Inizializzazione Python]
   1. roop.core.run() -> pre_check() (verifica Python >= 3.9, modelli in ..\models, ffmpeg nel PATH)
   2. Settings('config.yaml') -> legge C:\SfottiStarpa\app\roop-unleashed\config.yaml
   3. ui.main.prepare_environment() -> crea .\output e .\temp e imposta TEMP/TMP/GRADIO_TEMP_DIR
   4. Verifica GPU CUDA (torch.cuda.is_available()) -> se assente o fallita, fallback su CPU
   5. Avvia server Gradio/Uvicorn su http://127.0.0.1:7860
        │
        ▼
[Processi secondari a runtime]
   ├── Browser predefinito di Windows aperto su http://127.0.0.1:7860
   └── C:\SfottiStarpa\ffmpeg\bin\ffmpeg.exe (invocato durante l'elaborazione dei video)
```

### 3.2 Proprietà esatte dei collegamenti Windows (`FATTO VERIFICATO`)
Dallo stream header di `PrestoVoglioSfottereStarpa.exe` e dal log `unins000.dat`, l'installer crea due collegamenti (assumendo `{app}` = `C:\SfottiStarpa`):

1. **Collegamento nel Menu Start:**
   - **Nome visualizzato:** `Sfotti subito Starpa`
   - **Percorso file:** `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\SfottiStarpa\Sfotti subito Starpa.lnk`
   - **Destinazione (Target):** `C:\SfottiStarpa\run_roop.bat`
   - **Argomenti:** *(nessuno)*
   - **Da (Working Directory):** `C:\SfottiStarpa`
   - **Icona:** `C:\SfottiStarpa\SfottiStarpa.ico`

2. **Collegamento sul Desktop** (creato se la casella *"Crea un'icona sul desktop"* è stata lasciata spuntata):
   - **Nome visualizzato:** `Sfotti Starpa`
   - **Percorso file:** `C:\Users\Public\Desktop\Sfotti Starpa.lnk`
   - **Destinazione (Target):** `C:\SfottiStarpa\run_roop.bat`
   - **Argomenti:** *(nessuno)*
   - **Da (Working Directory):** `C:\SfottiStarpa`
   - **Icona:** `C:\SfottiStarpa\SfottiStarpa.ico`

### 3.3 Contenuto integrale di `run_roop.bat` (`FATTO VERIFICATO`)
File: `{app}\run_roop.bat`
```bat
@echo off
setlocal

REM Cartella root del box (quella dove sta questo .bat)
set ROOT=%~dp0

REM Aggiungo ffmpeg del box al PATH
set PATH=%ROOT%ffmpeg\bin;%PATH%

REM Interprete Python dell'ambiente (Micromamba usato solo in build, qui non serve)
set PYTHON_EXE=%ROOT%env\python.exe

REM Cartella dell'app
set APP_DIR=%ROOT%app\roop-unleashed

if not exist "%PYTHON_EXE%" (
    echo [ERRORE] Non trovo "%PYTHON_EXE%"
    echo L'installazione sembra incompleta - manca la cartella env
    pause
    exit /b 1
)

if not exist "%ROOT%ffmpeg\bin\ffmpeg.exe" (
    echo [ERRORE] Non trovo ffmpeg in "%ROOT%ffmpeg\bin\ffmpeg.exe"
    echo L'installazione sembra incompleta - manca la cartella ffmpeg
    pause
    exit /b 1
)

if not exist "%APP_DIR%\run.py" (
    echo [ERRORE] Non trovo run.py in "%APP_DIR%"
    echo L'installazione sembra incompleta - manca l'app
    pause
    exit /b 1
)

cd /d "%APP_DIR%"

"%PYTHON_EXE%" run.py

endlocal
```

### 3.4 Gestione degli errori in fase di avvio (`FATTO VERIFICATO`)
- **Uso di `python.exe` (non `pythonw.exe`):** Il batch usa `env\python.exe`, quindi apre sempre una finestra console.
- **Criticità in `run_roop.bat`:** I controlli `if not exist` iniziali hanno il comando `pause`, ma **dopo la riga `"%PYTHON_EXE%" run.py` NON c'è alcun `pause`**. Se `python.exe` viene bloccato da Windows o termina con un'eccezione (es. DLL mancante/bloccata, errore CUDA/ONNX, `PermissionError`), la finestra nera si chiude istantaneamente.
- **Presenza di `run_roop_debug.bat`:** Nella stessa cartella `{app}\` è già installato `run_roop_debug.bat`, che stampa a video `ROOT`, `PYTHON_EXE`, `APP_DIR` e `PATH`, lancia `"%PYTHON_EXE%" run.py` e **termina con `pause`**, mantenendo visibile qualunque messaggio di errore.

---

## 4. Compatibilità e Sicurezza Windows 11

Di seguito sono elencati gli elementi tecnici verificati nel repository che possono interagire con i meccanismi di sicurezza di Windows 11:

### 4.1 Smart App Control / Controllo intelligente delle app (`FATTO VERIFICATO` + `IPOTESI`)
- **Fatto verificato:** Il collegamento avvia uno script batch (`run_roop.bat`) che lancia `env\python.exe` (non firmato) il quale carica decine di DLL/PYD non firmate (`python310.dll`, `torch`, `onnxruntime`, `insightface`, `cv2`).
- **Impatto su Windows 11:** Su installazioni recenti di Windows 11 in cui **Controllo intelligente delle app (Smart App Control)** è attivo, Windows blocca l'esecuzione di script `.bat` o di eseguibili/DLL non firmati privi di reputazione positiva nel cloud Microsoft. A differenza di SmartScreen tradizionale, la finestra di blocco di **Controllo intelligente delle app** **non offre il pulsante "Esegui comunque"** sul singolo file.

### 4.2 Microsoft Defender SmartScreen (`FATTO VERIFICATO` + `IPOTESI`)
- **Fatto verificato:** Tutti i binari (`PrestoVoglioSfottereStarpa.exe`, `env\python.exe`, `ffmpeg\bin\ffmpeg.exe`) e gli script (`run_roop.bat`) sono privi di firma digitale Authenticode (`cert_size=0`).
- **Impatto su Windows 11:** Se l'installer o i file hanno mantenuto l'attributo NTFS Alternate Data Stream `Zone.Identifier` (Mark-of-the-Web, `ZoneId=3`, applicato da Windows quando i file vengono scaricati da browser, Google Drive, Mega, Telegram, ecc.), o quando Explorer avvia un `.bat` non riconosciuto tramite collegamento `.lnk`, **Microsoft Defender SmartScreen** può mostrare la schermata blu/azzurra *"PC protetto da Windows - Microsoft Defender SmartScreen ha impedito l'avvio di un'app non riconosciuta"*.

### 4.3 Microsoft Defender Antivirus / Euristica / Accesso alle cartelle controllato (`FATTO VERIFICATO` + `IPOTESI`)
- **Fatti verificati nel codice:**
  1. Il collegamento nel Menu Start (`C:\ProgramData\Microsoft\Windows\Start Menu\Programs\SfottiStarpa\Sfotti subito Starpa.lnk`) punta a un file `.bat` (`run_roop.bat`) mascherato con un'icona `.ico` (`SfottiStarpa.ico`).
  2. Durante il setup, l'installer ha eseguito `powershell.exe -ExecutionPolicy Bypass -File "C:\SfottiStarpa\check_env.ps1"`, generando ed eseguendo codice Python in `%TEMP%\roopbox_env_test.py`.
  3. `roop/core.py` importa `ctypes.windll.kernel32.SetProcessWorkingSetSize`.
- **Impatto su Windows 11:** Microsoft Defender Antivirus (o un antivirus di terze parti come Norton, McAfee, Avast, Kaspersky, Bitdefender spesso preinstallati sui PC consumer Windows 11) può mettere in quarantena o bloccare `run_roop.bat` o `env\python.exe` o `insightface`/`onnxruntime` tramite scansione comportamentale/AMSI.

### 4.4 UAC (Controllo account utente) e Permessi NTFS (`FATTO VERIFICATO` + `IPOTESI`)
- **Fatti verificati:**
  1. L'installer richiede privilegi di amministratore (`PrivilegesRequired=admin`).
  2. L'applicazione scrive file dentro la propria directory di installazione `{app}\app\roop-unleashed\`:
     - crea e usa `{app}\app\roop-unleashed\output\` per i file generati (`ui/main.py:22-23`);
     - crea e usa `{app}\app\roop-unleashed\temp\` come directory temporanea (`TEMP`, `TMP`, `GRADIO_TEMP_DIR`) perché in `config.yaml` è impostato `use_os_temp_folder: false` (`ui/main.py:24-27`);
     - riscrive `{app}\app\roop-unleashed\config.yaml` quando si salvano le impostazioni (`settings.py:65`).
- **Impatto su Windows 11:** Se Ufo ha installato il programma in `C:\SfottiStarpa`, di norma gli utenti standard hanno permessi di scrittura; ma se Ufo ha installato con un account Amministratore separato dal suo account utente standard, oppure in una radice/cartella con permessi NTFS ristretti, l'avvio non elevato fallirà su `os.makedirs` o sulla scrittura in `temp/`.

### 4.5 Windows Defender Firewall (`FATTO VERIFICATO` + `IPOTESI`)
- **Fatto verificato:** All'avvio, `env\python.exe` avvia il server web Uvicorn/Gradio sulla porta locale `7860` (`http://127.0.0.1:7860`).
- **Impatto su Windows 11:** Su alcune configurazioni di rete o se `server_name` / `server_share` sono attivi, Windows Defender Firewall può mostrare un avviso di sicurezza chiedendo di consentire a `python.exe` l'accesso alla rete.

### 4.6 Dipendenze native (GPU / CUDA / MSVC) (`FATTO VERIFICATO`)
- **Runtime MSVC:** Tutte le DLL MSVC (`vcruntime140*.dll`, `msvcp140*.dll`, `vcomp140.dll`, `concrt140.dll`) e UCRT (`ucrtbase.dll`, `api-ms-win-*.dll`) sono già incluse direttamente nella cartella `{app}\env\`.
- **CUDA / GPU:** In `config.yaml` il provider predefinito è `provider: cuda` e sono installati `torch-2.5.1+cu121` e `onnxruntime_gpu-1.17.1`. Se il PC di Ufo non ha una scheda NVIDIA o ha driver NVIDIA troppo vecchi per CUDA 12.1, `ui/main.py:36-37` esegue il fallback automatico su `cpu` (`if roop.globals.CFG.provider == "cuda" and util.has_cuda_device() == False: roop.globals.CFG.provider = "cpu"`).

---

## 5. Diagnostica Disponibile

### 5.1 Log ed eccezioni (`FATTO VERIFICATO`)
- **Nessun file di log su disco su Windows:** Né `run_roop.bat` né il codice Python di `roop-unleashed` salvano file `.log` su disco su Windows. Tutti i messaggi (`print`, stato di avanzamento, errori FFmpeg, traceback Python) vengono stampati solo nella finestra console `cmd.exe`.
- **Script diagnostici già presenti nella cartella installata (`C:\SfottiStarpa\`):**
  1. `C:\SfottiStarpa\run_roop_debug.bat`: avvia l'app stampando i percorsi di ambiente e mantiene la finestra aperta con `pause` finale.
  2. `C:\SfottiStarpa\check_env.ps1`: verifica l'esistenza di cartelle/eseguibili (`env\python.exe`, `ffmpeg\bin\ffmpeg.exe`, `app\roop-unleashed`) e testa l'avvio di `env\python.exe` importando `torch` e `huggingface_hub`.

### 5.2 Comandi diagnostici sicuri e non distruttivi per Ufo (`FATTO VERIFICATO`)
Tutti i comandi seguenti sono **di sola lettura o di test locale**, non modificano il sistema e **non disabilitano alcuna protezione di Windows**. Assumono l'installazione nel percorso predefinito `C:\SfottiStarpa` (se Ufo ha installato altrove, sostituire `C:\SfottiStarpa` con il suo percorso).

#### Test A — Verifica presenze file e attributo Mark-of-the-Web (da Prompt dei comandi `cmd.exe`)
Far aprire a Ufo il **Prompt dei comandi** (cercando `cmd` nel menu Start) e fargli incollare:
```cmd
dir "C:\SfottiStarpa" "C:\SfottiStarpa\env\python.exe" "C:\SfottiStarpa\ffmpeg\bin\ffmpeg.exe" "C:\SfottiStarpa\app\roop-unleashed\run.py"
```
Per verificare se i file hanno l'attributo `Zone.Identifier` (Mark-of-the-Web) che fa scattare SmartScreen:
```cmd
dir /R "C:\SfottiStarpa\run_roop.bat" "C:\SfottiStarpa\env\python.exe"
```
*(Se nell'output compaiono righe che terminano con `:Zone.Identifier:$DATA`, il file è marcato come proveniente da Internet).*

#### Test B — Avvio diagnostico con finestra che resta aperta (`run_roop_debug.bat`)
Nel **Prompt dei comandi** (`cmd.exe`) già aperto, far incollare:
```cmd
"C:\SfottiStarpa\run_roop_debug.bat"
```
*(In questo modo, anche se Windows blocca `run_roop.bat` o `python.exe`, il messaggio esatto di errore viene stampato direttamente nella finestra di `cmd.exe` senza chiudersi).*

#### Test C — Test isolato dell'interprete Python distribuito
Per capire se Windows blocca il file `.bat`/`.lnk` oppure l'eseguibile `python.exe` o le sue librerie (`torch`, `onnxruntime`), far incollare nel **Prompt dei comandi** (`cmd.exe`):
```cmd
"C:\SfottiStarpa\env\python.exe" -c "import sys; print('Python OK:', sys.version)"
```
Se questo funziona, testare l'import delle librerie native:
```cmd
"C:\SfottiStarpa\env\python.exe" -c "import torch, onnxruntime, insightface, gradio; print('Librerie OK - CUDA:', torch.cuda.is_available())"
```

#### Test D — Esecuzione dello script di verifica ambiente incluso (`check_env.ps1`)
Dal **Prompt dei comandi** (`cmd.exe`):
```cmd
powershell -NoProfile -ExecutionPolicy Bypass -File "C:\SfottiStarpa\check_env.ps1"
```

---

## 6. Ipotesi Ordinate per Probabilità

> **Nota metodologica:** Le evidenze nel repository sono **fatti verificati**; le cause del blocco sul PC di Ufo sono **ipotesi** da confermare in base al messaggio esatto mostrato da Windows 11.

### Ipotesi 1 (Molto Probabile): Blocco di Microsoft Defender SmartScreen ("PC protetto da Windows")
- **Evidenze nel repository (`FATTO VERIFICATO`):**
  - I collegamenti nel Menu Start (`Sfotti subito Starpa.lnk`) e sul Desktop (`Sfotti Starpa.lnk`) puntano a `C:\SfottiStarpa\run_roop.bat`.
  - Né `PrestoVoglioSfottereStarpa.exe`, né `run_roop.bat`, né `env\python.exe`, né `ffmpeg\bin\ffmpeg.exe` hanno firma digitale Authenticode (`cert_size=0`).
- **Informazioni mancanti:**
  - Screenshot o testo esatto della finestra mostrata a Ufo (es. finestra blu/azzurra con titolo *"PC protetto da Windows"* e testo *"Microsoft Defender SmartScreen ha impedito l'avvio di un'app non riconosciuta. L'esecuzione di questa app potrebbe mettere a rischio il PC"*).
- **Test semplice per confermarla o escluderla:**
  - Chiedere a Ufo se nella finestra di avviso c'è la scritta **"PC protetto da Windows"** e un link/pulsante **"Ulteriori informazioni"** (*More info*).
  - Eseguire `dir /R "C:\SfottiStarpa\run_roop.bat" "C:\SfottiStarpa\env\python.exe"` per vedere se è presente `:Zone.Identifier:$DATA`.
- **Soluzione (senza disabilitare SmartScreen):**
  - Se è la schermata SmartScreen standard: cliccare su **"Ulteriori informazioni"** nella finestra dell'avviso e poi sul pulsante **"Esegui comunque"**; oppure cliccare col tasto destro su `C:\SfottiStarpa\run_roop.bat` → **Proprietà** → nella scheda **Generale** in basso spuntare **"Annulla blocco"** (*Unblock*) → **Applica** (oppure rimuovere il solo stream `Zone.Identifier` sui file di `C:\SfottiStarpa` con `powershell -Command "Get-ChildItem -Path 'C:\SfottiStarpa' -Recurse | Unblock-File"` dopo aver confermato con Ufo l'origine fidata del pacchetto).

### Ipotesi 2 (Probabile su Windows 11 22H2/23H2/24H2): Blocco di "Controllo intelligente delle app" (Smart App Control)
- **Evidenze nel repository (`FATTO VERIFICATO`):**
  - `env\python.exe` (build `conda-forge`), tutte le DLL/PYD in `env\Lib\site-packages\` e `run_roop.bat` sono completamente privi di firma digitale Authenticode.
- **Informazioni mancanti:**
  - Sapere se l'avviso mostrato a Ufo dice **"Controllo intelligente delle app ha bloccato una app che potrebbe non essere sicura"** (o *"Impossibile eseguire questa app nel tuo PC - Controllo intelligente delle app ha bloccato..."*), avviso che **non** presenta il pulsante "Esegui comunque".
- **Test semplice per confermarla o escluderla:**
  - Guardare il titolo/testo della finestra di blocco o far eseguire da `cmd.exe`:
    `"C:\SfottiStarpa\env\python.exe" -V`
    Se Smart App Control blocca `python.exe` o `run_roop.bat`, `cmd.exe` stamperà:
    `Il programma o la funzionalità "...\python.exe" non può essere avviato o eseguito a causa di una incompatibilità con le versioni a 64 bit di Windows` oppure `Controllo intelligente delle app ha bloccato...` / `Accesso negato` / codice errore di blocco policy (`0x800711C7` / `4551`).
  - Far controllare a Ufo in **Sicurezza di Windows** → **Controllo delle app e del browser** → **Impostazioni di Controllo intelligente delle app** se lo stato è **Attivato**, **Valutazione** o **Disattivato**.
- **Soluzione (senza chiedere di disabilitare la protezione):**
  - **Importante:** Smart App Control non supporta esclusioni per singoli file o cartelle e, una volta disattivato su Windows 11, non può essere riattivato senza reinstallare Windows. Pertanto **non** chiedere mai a Ufo di disattivarlo.
  - Se il blocco è causato da Smart App Control, la soluzione corretta lato distribuzione è firmare digitalmente gli script/eseguibili o utilizzare una distribuzione dell'interprete Python dotata di firma Authenticode Microsoft/Python Software Foundation valida per i binari `.exe`/`.dll`, oppure verificare quale specifico file (`.bat` vs `python.exe` vs `.dll`) viene bloccato nel Visualizzatore eventi (`Microsoft-Windows-AppLocker/MSI and Script` o `CodeIntegrity/Operational`).

### Ipotesi 3 (Possibile): Rilevamento o quarantena da parte di Microsoft Defender Antivirus o Antivirus di terze parti
- **Evidenze nel repository (`FATTO VERIFICATO`):**
  - Il collegamento `.lnk` in `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\SfottiStarpa\` lancia un file `.bat` (`run_roop.bat`) con icona personalizzata (`SfottiStarpa.ico`).
  - L'installer ha eseguito durante il setup `powershell.exe -ExecutionPolicy Bypass -File "{app}\check_env.ps1"` e `{tmp}\video\vlc\vlc.exe`.
- **Informazioni mancanti:**
  - Sapere se sul PC di Ufo è presente un antivirus di terze parti (Norton, McAfee, Avast, AVG, Kaspersky, Bitdefender) o se la notifica proviene da **Sicurezza di Windows** (*"Minaccia rilevata"* / *"Azione necessaria"*).
- **Test semplice per confermarla o escluderla:**
  - Chiedere a Ufo di aprire **Sicurezza di Windows** → **Protezione da virus e minacce** → **Cronologia della protezione** e verificare se compare un elemento recente relativo a `C:\SfottiStarpa` (o controllare l'eventuale antivirus di terze parti).
  - Verificare con il **Test A** se `C:\SfottiStarpa\run_roop.bat` e `C:\SfottiStarpa\env\python.exe` sono ancora presenti su disco o se sono stati spostati in quarantena.
- **Soluzione:**
  - Se un file legittimo di `C:\SfottiStarpa` è stato messo in quarantena come falso positivo, ripristinarlo dalla **Cronologia della protezione** (senza disattivare la protezione in tempo reale).

### Ipotesi 4 (Meno Probabile): Avviso "Controllo dell'account utente" (UAC) o avviso di sicurezza file di Windows Explorer (`Open File - Security Warning`)
- **Evidenze nel repository (`FATTO VERIFICATO`):**
  - `run_roop.bat` non richiede privilegi amministrativi, ma se Ufo clicca per errore sull'eseguibile dell'installer (`PrestoVoglioSfottereStarpa.exe`) o su `unins000.exe` invece che sul collegamento `Sfotti subito Starpa`, oppure se `run_roop.bat` ha lo stream `Zone.Identifier`, Explorer mostra la finestra *"Avviso di sicurezza - Eseguire questo file? Autore: Autore sconosciuto"*.
- **Informazioni mancanti:**
  - Quale icona esatta Ufo sta cliccando e cosa dice la finestra.
- **Test semplice per confermarla o escluderla:**
  - Chiedere a Ufo se la finestra ha i pulsanti **"Esegui"** / **"Annulla"** con la voce *"Autore: Autore sconosciuto"* e il percorso `C:\SfottiStarpa\run_roop.bat`.
- **Soluzione:**
  - Cliccare **"Esegui"** (e, se presente, rimuovere il blocco dalle Proprietà di `run_roop.bat`).

### Ipotesi 5 (Bassa Probabilità per l'avviso di sicurezza, ma possibile subito dopo l'avvio): Popup di Windows Defender Firewall o errore di permessi NTFS su `temp/` e `output/`
- **Evidenze nel repository (`FATTO VERIFICATO`):**
  - `ui/main.py` scrive in `C:\SfottiStarpa\app\roop-unleashed\temp` e `output` e avvia un server HTTP su `127.0.0.1:7860`.
- **Informazioni mancanti:**
  - Sapere se l'avviso compare **prima** che si apra la finestra nera del prompt dei comandi, oppure **dopo** che la finestra nera è rimasta aperta per 1-2 minuti.
- **Test semplice per confermarla o escluderla:**
  - Eseguire `"C:\SfottiStarpa\run_roop_debug.bat"` da `cmd.exe` (Test B) e osservare a quale riga si ferma.
- **Soluzione:**
  - Se è il popup di Windows Firewall per `python.exe`, consentire l'accesso su reti private o verificare che `server_name: ''` e `server_share: false` in `C:\SfottiStarpa\app\roop-unleashed\config.yaml`.
  - Se è un `PermissionError` su `C:\SfottiStarpa\app\roop-unleashed\temp` o `output`, verificare che l'utente Windows di Ufo abbia i permessi di modifica sulla cartella `C:\SfottiStarpa`.

---

## 7. Informazioni da Chiedere a Ufo (Guida Interattiva Passo-Passo)

> **Istruzioni per ChatGPT che assiste Ufo:** Ufo non è un utente tecnico. Fai **una sola domanda o richiesta alla volta**, con tono amichevole e istruzioni semplicissime. Parti sempre dal Passo 1.

### Passo 1 — Identificare il messaggio esatto (Screenshot o testo)
Chiedi a Ufo:
1. **"Quale icona stai cliccando per avviare il programma?"**
   - L'icona **Sfotti Starpa** sul Desktop?
   - La voce **Sfotti subito Starpa** nel Menu Start?
2. **"Puoi mandarmi una foto (anche col telefono) o uno screenshot della finestra di avviso che ti compare, oppure copiarmi le prime due righe scritte in quella finestra?"**
3. **"Prima che compaia quell'avviso, si apre per un attimo una finestra nera oppure l'avviso compare subito appena fai doppio clic?"**

---

### Passo 2 — Rami diagnostici in base alla risposta di Ufo al Passo 1

#### Ramo A: La finestra è azzurra/blu e dice *"PC protetto da Windows"* / *"Microsoft Defender SmartScreen ha impedito l'avvio di un'app non riconosciuta"*
1. Spiega a Ufo che l'avviso compare perché `SfottiStarpa` è un programma amatoriale non firmato con un certificato commerciale Microsoft.
2. Chiedigli di cliccare sulla scritta sottolineata **"Ulteriori informazioni"** (*More info*) dentro quella stessa finestra azzurra.
3. Appena compare il pulsante **"Esegui comunque"** (*Run anyway*) in basso, farglielo cliccare.
4. Avvisalo che dopo aver cliccato **"Esegui comunque"** si aprirà una **finestra nera** e dovrà **aspettare circa 1–2 minuti senza chiuderla** finché non si aprirà da sola la pagina di SfottiStarpa nel browser.
5. *(Se l'avviso dovesse ricomparire a ogni avvio)*: fargli aprire Esplora File in `C:\SfottiStarpa`, cliccare col tasto destro su `run_roop` (`run_roop.bat`) → **Proprietà** → spuntare in basso **"Annulla blocco"** → **OK**.

#### Ramo B: La finestra dice *"Avviso di sicurezza - Impossibile verificare l'autore. Eseguire questo file?"* (con Nome: `...run_roop.bat` e Autore: `Autore sconosciuto`)
1. Chiedi a Ufo di togliere la spunta da *"Avvisa sempre prima di aprire questo file"* (se presente) e cliccare su **"Esegui"**.
2. Ricordagli di attendere 1–2 minuti con la finestra nera aperta finché non si apre il browser.

#### Ramo C: La finestra dice *"Controllo intelligente delle app ha bloccato una app che potrebbe non essere sicura"* (Smart App Control) e NON c'è alcun tasto "Esegui comunque"
1. **NON** dire a Ufo di disattivare Controllo intelligente delle app.
2. Per capire quale file specifico viene bloccato (se il `.bat` o `python.exe`), chiedi a Ufo di fare questo test guidato:
   - Premi il tasto **Start** di Windows, scrivi **`cmd`** e premi **Invio** (si aprirà una finestra nera).
   - Copia e incolla questa riga nella finestra nera e premi **Invio**:
     ```cmd
     "C:\SfottiStarpa\env\python.exe" -c "import sys; print('PYTHON OK:', sys.version)"
     ```
   - Chiedi a Ufo: **"Cosa ti scrive sotto? Scrive `PYTHON OK: 3.10.19` oppure compare di nuovo l'avviso di blocco?"**
3. Se `python.exe` stampa `PYTHON OK: 3.10.19`, significa che Smart App Control blocca solo l'avvio tramite il file `.bat`/collegamento, ma permette l'esecuzione di `python.exe`. In tal caso, fai provare a Ufo nella stessa finestra nera:
   ```cmd
   cd /d C:\SfottiStarpa\app\roop-unleashed && set PATH=C:\SfottiStarpa\ffmpeg\bin;%PATH% && C:\SfottiStarpa\env\python.exe run.py
   ```
4. Se invece anche `"C:\SfottiStarpa\env\python.exe"` viene bloccato da Controllo intelligente delle app, chiedi a Ufo uno screenshot del messaggio nel Prompt dei comandi per riferirlo allo sviluppatore (poiché su un PC con Smart App Control attivo gli eseguibili non firmati come il `python.exe` di `conda-forge` vengono bloccati dal kernel Windows).

#### Ramo D: Compare una notifica di *Sicurezza di Windows* ("Minaccia rilevata") o di un altro Antivirus (McAfee, Norton, Avast, ecc.), oppure compare un errore *"Impossibile trovare run_roop.bat / python.exe"*
1. Chiedi a Ufo di aprire il **Prompt dei comandi** (tasto Start → scrivi `cmd` → Invio) e incollare:
   ```cmd
   dir "C:\SfottiStarpa" "C:\SfottiStarpa\env\python.exe" "C:\SfottiStarpa\ffmpeg\bin\ffmpeg.exe"
   ```
2. Se `C:\SfottiStarpa` non viene trovato, chiedi a Ufo in quale disco o cartella ha scelto di installare SfottiStarpa durante il setup (ricordandogli che il setup obbligava a scegliere un percorso senza spazi, es. `D:\SfottiStarpa`).
3. Se `C:\SfottiStarpa` esiste ma manca `run_roop.bat` o `env\python.exe`, l'antivirus lo ha messo in quarantena: guida Ufo ad aprire **Sicurezza di Windows** → **Protezione da virus e minacce** → **Cronologia della protezione** (o la quarantena del suo antivirus) per verificare quale file è stato bloccato e mandarne uno screenshot.

#### Ramo E: Si apre la finestra nera ma si richiude subito da sola, o il browser non si apre
1. Chiedi a Ufo di aprire il **Prompt dei comandi** (Start → `cmd` → Invio), incollare questo comando (che usa lo script di debug già installato) e premere **Invio**:
   ```cmd
   "C:\SfottiStarpa\run_roop_debug.bat"
   ```
2. La finestra resterà aperta alla fine con la scritta `Premi un tasto per chiudere`. Chiedi a Ufo di mandare una foto o uno screenshot di tutto quello che è comparso scritto nella finestra nera.
