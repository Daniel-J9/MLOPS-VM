# MLOPS-VM
Virtual Machine for study

# Uruchomienie modelu językowego (LLM) llamafile jako usługi systemd

Dokumentacja opisuje proces wdrożenia i konfiguracji gotowego modelu uczenia maszynowego (`TinyLlama-1.1B-Chat-v1.0.Q4_K_M.llamafile`) jako usługi systemowej `systemd` na maszynie wirtualnej z systemem Ubuntu Server 24.04 LTS.

---

## 1. Architektura środowiska i wymagania wstępne

Maszyna wirtualna zarządzana przez Vagrant i VirtualBox posiada następującą konfigurację sieciową oraz sprzętową:
* **System operacyjny:** Ubuntu Linux 24.04 LTS (`bento/ubuntu-24.04`)
* **Zasoby sprzętowe:** 4096 MB RAM (ok. 2.9 GiB dostępne w systemie), 2 vCPU
* **Interfejsy sieciowe:**
  * `lo` (`127.0.0.1/8`) – pętla zwrotna
  * `eth0` / `enp0s3` (`10.0.2.15/24`) – interfejs NAT (dostęp do Internetu)
  * `eth1` / `enp0s8` (`192.168.56.10/24`) – interfejs **Host-Only** (komunikacja z gospodarzem)

---

## 2. Instrukcja wykonania zadania krok po kroku (odtworzenie na nowej maszynie)

### Krok 1: Uruchomienie maszyny i połączenie przez SSH
W terminalu gospodarza (Windows PowerShell) w katalogu z plikiem `Vagrantfile` uruchamiamy maszynę i logujemy się na nią przez SSH:
```powershell
vagrant up
ssh -p 22 vagrant@192.168.56.10
```

### Krok 2: Utworzenie dedykowanego użytkownika systemowego
Wewnątrz maszyny wirtualnej tworzymy bezuprzywilejowanego użytkownika systemowego `llamafile` bez katalogu domowego i bez możliwości interaktywnego logowania:
```bash
sudo useradd --system --no-create-home -s /usr/sbin/nologin llamafile
```

### Krok 3: Pobranie pliku `llamafile` i nadanie uprawnień
Tworzymy katalog `/opt/llamafile`, pobieramy plik wykonywalny modelu przez `curl` (ok. 869 MB), nadajemy mu prawa wykonywania (`chmod +x`) oraz przypisujemy właściciela `llamafile:llamafile`:
```bash
sudo mkdir -p /opt/llamafile
sudo curl -L [https://huggingface.co/Mozilla/TinyLlama-1.1B-Chat-v1.0-llamafile/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q4_K_M.llamafile](https://huggingface.co/Mozilla/TinyLlama-1.1B-Chat-v1.0-llamafile/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q4_K_M.llamafile) -o /opt/llamafile/model.llamafile
sudo chmod +x /opt/llamafile/model.llamafile
sudo chown -R llamafile:llamafile /opt/llamafile
```

Wynik pobierania:
```text
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   136  100   136    0     0    630      0 --:--:-- --:--:-- --:--:--   632
100  1154  100  1154    0     0   3330      0 --:--:-- --:--:-- --:--:--  3330
100  868M  100  868M    0     0  67.1M      0  0:00:12  0:00:12 --:--:-- 93.5M
```

### Krok 4: Utworzenie pliku jednostki `systemd`
Tworzymy plik `/etc/systemd/system/llamafile.service`:
```bash
sudo tee /etc/systemd/system/llamafile.service << 'EOF'
[Unit]
Description=Serwer modelu LLM llamafile
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=llamafile
Group=llamafile
WorkingDirectory=/opt/llamafile
Environment="HOME=/opt/llamafile"
MemoryAccounting=yes
ExecStart=/bin/sh /opt/llamafile/model.llamafile --server --nobrowser --host 192.168.56.10 --port 8080
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

*Uwagi konfiguracyjne:*
* Wywołanie przez `/bin/sh` pozwala na poprawne uruchomienie wieloplatformowego formatu APE (*Actually Portable Executable*) przez `systemd`.
* Zmienna `Environment="HOME=/opt/llamafile"` wskazuje katalog roboczy, do którego `llamafile` wypakowuje wewnętrzny mikro-loader `.ape-1.10` (ponieważ użytkownik systemowy ma flagę `--no-create-home`).
* Parametr `--host 192.168.56.10 --port 8080` ogranicza nasłuchiwanie serwera HTTP wyłącznie do interfejsu Host-Only.
* Parametr `Restart=always` gwarantuje ponowne uruchomienie usługi zarówno po błędzie, jak i po zamknięciu procesu sygnałem `SIGTERM` (domyślne polecenie `kill`).

### Krok 5: Przeładowanie `systemd` i włączenie usługi
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now llamafile.service
```

---

## 3. Weryfikacja działania i wyniki testów

### 3.1. Status usługi po uruchomieniu (`systemctl status`)
```text
vagrant@vagrant:~$ sudo systemctl status llamafile.service
● llamafile.service - Serwer modelu LLM llamafile
     Loaded: loaded (/etc/systemd/system/llamafile.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-10-25 18:28:06 UTC; 4s ago
   Main PID: 3193 (.ape-1.10)
      Tasks: 11 (limit: 3470)
     Memory: 59.5M (peak: 59.9M)
        CPU: 275ms
     CGroup: /system.slice/llamafile.service
             └─3193 /opt/llamafile/.ape-1.10 /opt/llamafile/model.llamafile --server --nobrowser --host 192.168.56.10 --port 8080

Oct 25 18:28:06 vagrant sh[3193]: llama_new_context_with_model:        CPU compute buffer size =   148.01 MiB
Oct 25 18:28:06 vagrant sh[3193]: llama_new_context_with_model: graph nodes  = 710
Oct 25 18:28:06 vagrant sh[3193]: llama_new_context_with_model: graph splits = 1
Oct 25 18:28:06 vagrant sh[3193]: {"function":"initialize","level":"INFO","line":502,"msg":"initializing slots","n_slots":1}
Oct 25 18:28:06 vagrant sh[3193]: {"function":"initialize","level":"INFO","line":514,"msg":"new slot","n_ctx_slot":2048}
Oct 25 18:28:06 vagrant sh[3193]: {"function":"server_cli","level":"INFO","line":3131,"msg":"model loaded","tid":"12265"}
Oct 25 18:28:06 vagrant sh[3193]: llama server listening at [http://192.168.56.10:8080](http://192.168.56.10:8080)
Oct 25 18:28:06 vagrant sh[3193]: In the sandboxing block!
```

### 3.2. Weryfikacja nasłuchiwania portów (`ss -tlnp`)
Serwer nasłuchuje wyłącznie na adresie Host-Only `192.168.56.10:8080`:
```text
vagrant@vagrant:~$ sudo ss -tlnp
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      5      192.168.56.10:8080       0.0.0.0:*     users:((".ape-1.10",pid=3193,fd=4))
LISTEN 0      4096   127.0.0.53%lo:53         0.0.0.0:*     users:(("systemd-resolve",pid=600,fd=15))
LISTEN 0      4096         0.0.0.0:22         0.0.0.0:*     users:(("sshd",pid=2610,fd=3),("systemd",pid=1,fd=184))
LISTEN 0      4096      127.0.0.54:53         0.0.0.0:*     users:(("systemd-resolve",pid=600,fd=17))
LISTEN 0      4096            [::]:22            [::]:*     users:(("sshd",pid=2610,fd=4),("systemd",pid=1,fd=185))
```

Weryfikacja braku dostępu na pozostałych interfejsach (np. `127.0.0.1`):
```text
vagrant@vagrant:~$ curl [http://127.0.0.1:8080/v1/models](http://127.0.0.1:8080/v1/models)
curl: (7) Failed to connect to 127.0.0.1 port 8080 after 0 ms: Couldn't connect to server
```

### 3.3. Przykładowe zapytanie `curl` i odpowiedź modelu
Zapytanie wysłane na adres Host-Only `192.168.56.10:8080`:
```bash
curl [http://192.168.56.10:8080/v1/chat/completions](http://192.168.56.10:8080/v1/chat/completions) \
  -H "Content-Type: application/json" \
  -d '{
    "model": "LLaMA_CPP",
    "messages": [
      {"role": "user", "content": "What is Linux? Answer in one short sentence."}
    ]
  }'
```

Odpowiedź modelu (JSON):
```json
{"choices":[{"finish_reason":"stop","index":0,"message":{"content":"Linux is a free and open-source operating system (OS) that is designed to be user-friendly, secure, and customizable. It is used in a wide range of applications, including servers, desktops, and mobile devices.</s>","role":"assistant"}}],"created":1792953161,"id":"chatcmpl-ntgxt7clSPNeluq2hzfWSjmC8XyOYvvt","model":"LLaMA_CPP","object":"chat.completion","usage":{"completion_tokens":51,"prompt_tokens":31,"total_tokens":82}}
```

### 3.4. Zużycie pamięci przez usługę (`top` oraz `systemctl status`)
Po przetworzeniu zapytania model zmapował wagi do pamięci RAM:
* Odczyt z `top -b -n 1 -u llamafile`:
  * **Pamięć wirtualna (`VIRT`):** `1859120` KiB (~1.81 GiB)
  * **Pamięć fizyczna rezydentna (`RES`):** `686080` KiB (~670 MiB, co stanowi **22.5%** całkowitej pamięci RAM maszyny)
  * **Pamięć współdzielona / zmapowane wagi `mmap` (`SHR`):** `624256` KiB (~610 MiB)
* Odczyt prywatnej pamięci procesu z `systemctl status`: `62.3M (peak: 62.6M)`

```text
vagrant@vagrant:~$ top -b -n 1 -u llamafile
top - 18:35:08 up  9:09,  1 user,  load average: 0.00, 0.01, 0.00
Tasks: 142 total,   1 running, 141 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   2972.7 total,   1154.9 free,    392.4 used,   1598.8 buff/cache
MiB Swap:   2972.0 total,   2972.0 free,      0.0 used.   2580.3 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   3193 llamafi+  20   0 1859120 686080 624256 S   0.0  22.5   0:04.98 .ape-1.10
```

### 3.5. Weryfikacja automatycznego ponownego uruchomienia po `kill`
Zabicie procesu o identyfikatorze `PID 3193` poleceniem `kill` powoduje automatyczne podniesienie usługi przez `systemd` z nowym identyfikatorem `Main PID: 3245`:
```text
vagrant@vagrant:~$ sudo kill 3193
vagrant@vagrant:~$ sleep 6
vagrant@vagrant:~$ sudo systemctl status llamafile.service --no-pager
● llamafile.service - Serwer modelu LLM llamafile
     Loaded: loaded (/etc/systemd/system/llamafile.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-10-25 18:36:27 UTC; 764ms ago
   Main PID: 3245 (.ape-1.10)
      Tasks: 11 (limit: 3470)
     Memory: 59.4M (peak: 59.7M)
        CPU: 205ms
     CGroup: /system.slice/llamafile.service
             └─3245 /opt/llamafile/.ape-1.10 /opt/llamafile/model.llamafile --server --nobrowser --host 192.168.56.10 --port 8080
```
