# KoTH: Maskinreference

**TryHackMe King of the Hill · teknisk opslagsværk · november 2026**

> **Status:** Denne reference er udarbejdet på baggrund af de 14 maskinbeskrivelser i [andracs/koth-vejledning](https://github.com/andracs/koth-vejledning). Det er **ikke** en ny live-verifikation af hver maskine. Adgangsveje, porte, konti og fejlkonfigurationer kan ændre sig, især efter reset. Brug kun fremgangsmåderne på din tildelte TryHackMe KoTH-maskine og inden for konkurrencens regler.
>
> **Formål:** Forstå teknikken og kom hurtigere fra observation til begrundet næste skridt. Ikke en samling kommandoer, der skal køres blindt.

## 1. Hurtigt overblik

Den dokumenterede pool omfatter **12 Linux- og 2 Windows-maskiner**. **Fireworks, Fortune og Hogwarts** beskrives som variable mellem resets. Brug derfor altid en ny scanning, også når du har løst maskinen før.

| Maskine | OS | Muligt første spor | Vigtig eskalationsidé | Særligt at huske |
|---|---|---|---|---|
| Carnage | Linux | Webupload, skjult PHP | Sudo/SUID-programmer | `king.txt` kan være append-only |
| Fireworks | Linux | Variable web- og netværkstjenester | Misbrugt sudo-konfiguration | Scan hele portområdet |
| Food | Linux | Usædvanlig Telnet-/API-tjeneste | SUID-editor, gammel screen | Ikke kun standardporte |
| Fortune | Linux | Kodet ZIP, credentials, NFS | Sudo og SUID | Varierer efter reset |
| H1: Easy | Linux | CMS og PHP-eksekvering | PATH/cron | Flere værter/tjenester |
| H1: Hard | Linux | XML/API, SSH | Sudo, Docker-socket | Kæd flere svagheder sammen |
| H1: Medium | Windows | SMB og legitimationsoplysninger | Windows-administratoradgang | Windows-kommandoer |
| Hackers | Linux | FTP/SSH og web-login | Sudo/capabilities | Flere brugere/indgange |
| Hogwarts | Linux | Upload, FTP, SQLi | SUID, sudo | Variable porte |
| Offline | Windows | WebDAV, SMB | Windows-token/ACL | Særligt king-filnavn |
| Panda | Linux | WordPress og skjult websti | Sudo/SUID | `chattr` kan mangle |
| Production | Linux | FTP/SMB og SSH-nøgle | Sudo-programmer | Nøgler i delte mapper |
| Shrek | Linux | Web-API, FTP og SSH | Sudo/SUID/Docker | Flere parallelle indgange |
| Tyler | Linux | SMB, LibreNMS og webinput | SUID, credential-fund | Skeln version fra sårbarhed |

## 2. Standardworkflow: Før du vælger angrebsvej

Brug **kun** konkurrence-IP'en. Gem gerne output, så du undgår at gentage scanninger.

```bash
export TARGET=<KOTH-IP>
mkdir -p scans
nmap -p- --open -oN scans/all-tcp.txt "$TARGET"
nmap -sV -sC -p <FUNDNE_PORTE> -oN scans/services.txt "$TARGET"
```

**Hvorfor:** Første scanning viser *hvilke* TCP-porte der er åbne. Den anden viser *hvad* der sandsynligvis lytter på dem. En åben port er et spor, ikke et bevis på en sårbarhed.

Ved webtjenester:

```bash
curl -i http://<IP>:<PORT>/
curl -i http://<IP>:<PORT>/robots.txt
gobuster dir -u http://<IP>:<PORT>/ -w /usr/share/wordlists/dirb/common.txt
```

Ved fil- og delingstjenester:

```bash
smbclient -L //<IP> -N
showmount -e <IP>
ftp <IP>
```

**Vælg prioritet:** (1) synlige credentials eller offentlige filer, (2) tydelig fejlkonfiguration, (3) sårbar version med relevant dokumentation, (4) mere tidskrævende hypoteser. Kontroller at eventuelle loginforsøg og automatiserede tests følger KoTH-reglerne.

Efter shell på **Linux**:

```bash
whoami; id; hostname; uname -a
sudo -l
find / -perm -4000 -type f 2>/dev/null
getcap -r / 2>/dev/null
ss -tulpen
ps aux
```

Efter adgang på **Windows**:

```cmd
whoami /all
hostname
ipconfig /all
net user
net localgroup administrators
```

**Hvorfor:** Du skal kende identiteten, de tildelte privilegier, lokale tjenester og mulige veje videre, før du vælger teknik. Brug [GTFOBins](https://gtfobins.github.io/) til at forstå Linux-programmer med særlige rettigheder og [LOLBAS](https://lolbas-project.github.io/) til Windows.

## 3. Maskinerne, én for én

### Carnage · Linux

**Hvad maskinnoterne peger på:** En webupload på port 82, en skjult `dice.php`-funktion på port 80 og privilegerede programmer såsom `find`, `vim.tiny` og `netkit-ftp`.

**Undersøg i denne rækkefølge:**

1. Kontrollér webtjenester på de åbne porte. Notér formularer, filtyper, HTTP-status og uploadrespons.
2. Test i labbet, om uploadkontrollen validerer **indhold** eller kun filnavn/`Content-Type`. En fil, der ser ud som et billede i HTTP-headeren, er ikke nødvendigvis et sikkert billede.
3. Undersøg, om `dice.php` behandler input på en overraskende måde; originalnoterne nævner baglæns behandling.
4. Ved shell: brug `sudo -l` og SUID-listen, og slå de konkrete programmer op i GTFOBins.

```bash
curl -i http://<IP>:80/
curl -i http://<IP>:82/
sudo -l
find / -perm -4000 -type f 2>/dev/null
lsattr /root/king.txt 2>/dev/null
```

**Særligt:** Noterne angiver, at king-filen kan være **append-only**. Kontroller filattributter, inden du forsøger normal overskrivning. Ændr ikke attributter eller konkurrencemekanismer uden for spillets regler.

**MITRE-kobling:** Exploit Public-Facing Application (T1190), Command and Scripting Interpreter (T1059) og Abuse Elevation Control Mechanism (T1548), hvis det faktisk er de anvendte teknikker.

### Fireworks · Linux

**Hvad maskinnoterne peger på:** Flere tilfældige varianter, hvor åben port og tilgængelig tjeneste kan skifte. En ældre eller usikker `sudo`-adgang til `nmap` nævnes som privilegievej.

**Undersøg i denne rækkefølge:**

1. Start med **fuld TCP-portscanning**, ikke kun top-1000.
2. Klassificér web, SSH, filservices og eventuelle usædvanlige porte.
3. Når du har almindelig brugeradgang, kontrollér `sudo -l` og versionen af eventuelle privilegerede værktøjer.
4. Vurder, om den dokumenterede teknik faktisk gælder den aktuelle version. Ældre `nmap`-funktioner er ikke tilgængelige i alle udgaver.

```bash
nmap -p- --open <IP>
nmap -sV -sC -p <FUNDNE_PORTE> <IP>
sudo -l
nmap --version
```

**Særligt:** Tving ikke en løsning fra en tidligere reset ned over en anden variant.

**MITRE:** Network Service Discovery (T1046) og eventuelt T1548.

### Food · Linux

**Hvad maskinnoterne peger på:** Telnet på en usædvanlig høj port, et kommando-API på en anden høj port, og efterfølgende SUID-/`screen`-relaterede fejlkonfigurationer. Noterne omtaler også et simpelt Caesar-kodet credential-spor.

**Undersøg i denne rækkefølge:**

1. Find højportene med fuld scanning, og identificér protokol før login.
2. Kig efter tekst, banner eller hint, der antyder kodning. Dokumentér, hvordan du afkoder det.
3. Undersøg API'ets metoder og svar, og om input håndteres som data eller kommando.
4. Ved shell: kontrollér SUID-binarier, tilgængelige editors og `screen`-version.

```bash
nmap -p- --open <IP>
nmap -sV -p <FUNDNE_PORTE> <IP>
curl -i http://<IP>:<WEBPORT>/
find / -perm -4000 -type f 2>/dev/null
screen --version
```

**Særligt:** En versionstekst alene beviser ikke udnyttelighed. Kontroller rettigheder og forudsætninger.

**MITRE:** T1190/T1059 (hvis API eksekverer kommandoer) og T1548.

### Fortune · Linux

**Hvad maskinnoterne peger på:** En tjeneste på port 3333, der leverer kodet indhold/ZIP, mulige SSH-oplysninger og en NFS-eksport. Maskinen beskrives som variabel.

**Undersøg i denne rækkefølge:**

1. Identificér tjenesten på den aktuelle port og gem responsen som evidens.
2. Undersøg filtype og kodning, før du forsøger at åbne eller dekode filen.
3. Kontrollér, hvilke NFS-eksporter der tilbydes, og om adgangskontrol er passende.
4. Efter adgang: `sudo -l`, SUID og filrettigheder.

```bash
curl -i http://<IP>:3333/
file <NEDHENTET_FIL>
showmount -e <IP>
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

**Særligt:** NFS uden passende root-squash kan være alvorlig, men undersøg serverens faktiske eksportpolitik i stedet for at antage den.

**MITRE:** T1046, Valid Accounts (T1078) og T1548, hvor det er relevant.

### H1: Easy · Linux

**Hvad maskinnoterne peger på:** Flere webtjenester: et CMS, en PHP-funktion og muligheder for eskalation via usikker `PATH` eller cron-script.

**Undersøg i denne rækkefølge:**

1. Kortlæg de enkelte webporte separat; de kan repræsentere forskellige services.
2. Undersøg CMS-version, administratorgrænseflade og tilladte redigeringsfunktioner.
3. Hvis du får shell, undersøg om privilegerede scripts kalder programmer med relative navne.
4. Se efter cronjobs og rettigheder på scripts, som de afvikler.

```bash
curl -i http://<IP>:8000/
curl -i http://<IP>:8002/
sudo -l
cat /etc/crontab
ls -la /etc/cron.d /etc/cron.hourly 2>/dev/null
```

**Særligt:** PATH-hijacking kræver en konkret privilegeret eksekveringskontekst. Det er ikke nok, at en mappe er skrivbar.

**MITRE:** T1190, Scheduled Task/Job (T1053) og T1548, afhængigt af den konkrete kæde.

### H1: Hard · Linux

**Hvad maskinnoterne peger på:** XML-endpoint, mulige kildekode-/cookie-læk, API på særskilt port, SSH på en alternativ port og senere Docker-socket.

**Undersøg i denne rækkefølge:**

1. Identificér API-format og XML-parserens forventede input.
2. Kig efter indikationer på XXE, men skeln mellem hypotese og reproducerbar observation.
3. Undersøg autentifikationsflow, cookies og SSH-port, hvis de er relevante for den tildelte instans.
4. Efter shell: kontrollér gruppemedlemskab og adgang til container-runtime/socket.

```bash
curl -i http://<IP>/api/user
curl -i http://<IP>:8888/
id
groups
ls -l /var/run/docker.sock 2>/dev/null
```

**Særligt:** Medlemskab af Docker-gruppen eller skrivbar Docker-socket kan i praksis give meget høje privilegier på værten. Verificér grænserne i labbet.

**MITRE:** T1190, Exploitation for Privilege Escalation (T1068) eller T1548, hvis observeret.

### H1: Medium · Windows

**Hvad maskinnoterne peger på:** SMB og svage legitimationsoplysninger, som kan føre til administrativ fjernadgang. Maskinen er Windows-baseret.

**Undersøg i denne rækkefølge:**

1. Identificér SMB-port og netværksnavne.
2. Undersøg tilgængelige shares og autentifikationskrav.
3. Verificér brugernavn, grupper og effektivt privilege level ved legitim adgang.
4. Forstå hvor konkurrencefilen ligger, og hvordan Windows-filrettigheder virker.

```bash
nmap -sV -p 139,445 <IP>
smbclient -L //<IP> -N
```

```cmd
whoami /all
net localgroup administrators
icacls C:\king.txt
```

**Særligt:** `attrib +r` er en filattribut, ikke en robust adgangskontrol. Skeln mellem attributter og NTFS-ACL'er.

**MITRE:** Valid Accounts (T1078), Remote Services (T1021) og eventuelt T1548.

### Hackers · Linux

**Hvad maskinnoterne peger på:** FTP/SSH, web-login, flere brugerkonti og mulige privilegerede funktioner i `openssl`, editor eller Python.

**Undersøg i denne rækkefølge:**

1. Sammenhold kontonavne fra FTP, web og SSH. Er der genbrugte identiteter?
2. Undersøg web-loginets fejlrespons uden at antage, at det er sikkert at automatisere passwordforsøg.
3. Ved brugeradgang: `sudo -l`, capabilities og præcis sti/version på programmer.
4. Vurder om privilegiet tillader filadgang eller proceseksekvering, og begrund hvorfor.

```bash
nmap -sV -p 21,22,<WEBPORT> <IP>
sudo -l
getcap -r / 2>/dev/null
command -v python3
```

**Særligt:** Credential-angreb skal holde sig inden for konkurrencens regler og ressourcer; ukontrolleret brute force er ikke en standardmetode.

**MITRE:** Brute Force (T1110), hvis anvendt, Valid Accounts (T1078) og T1548.

### Hogwarts · Linux

**Hvad maskinnoterne peger på:** Tilfældige porte; en SSH-lignende indgang kan præsentere webindhold. Der nævnes upload, FTP-arkiv, SQL-injektion og flere privilegiefejl.

**Undersøg i denne rækkefølge:**

1. Stol på den *observerede protokol*, ikke portnummerets navn.
2. Kig efter uploadpolitik og uventede Content-Type-/filtypekontroller.
3. Undersøg downloads og metadata fra FTP, herunder arkivernes indhold.
4. Hvis en databasefejl mistænkes, dokumentér den relevante inputparameter og responsforskel.
5. Ved shell: gennemgå SUID, sudo og lokale specialprogrammer.

```bash
nmap -p- --open <IP>
nmap -sV -sC -p <FUNDNE_PORTE> <IP>
find / -perm -4000 -type f 2>/dev/null
sudo -l
```

**Særligt:** En tidligere exploit eller adresse er ikke nødvendigvis gyldig efter reset.

**MITRE:** T1190, T1059 og T1548, afhængigt af faktiske fund.

### Offline · Windows

**Hvad maskinnoterne peger på:** WebDAV og SMB, domæne-/servicekonti og privilegier knyttet til backup eller fil-ACL'er. Kongefilen kan ligge under `C:\Users\Administrator\king-server\king.txt`.

**Undersøg i denne rækkefølge:**

1. Tjek HTTP og WebDAV-metoder, herunder om indhold kan listes eller downloades.
2. Kortlæg SMB-shares og domænekontekst.
3. Ved adgang: læs tokenprivilegier og ACL'er, før du ændrer noget.
4. Bekræft den aktuelle sti til king-filen; antag ikke, at den er `C:\king.txt`.

```bash
curl -i -X OPTIONS http://<IP>/
nmap -sV -p 80,139,445 <IP>
smbclient -L //<IP> -N
```

```cmd
whoami /all
net use
icacls C:\Users\Administrator\king-server\king.txt
```

**Særligt:** Backup-privilegier er forskellige fra almindelige administratorrettigheder. Brug Windows' rettighedsmodel til at forklare forskellen.

**MITRE:** T1021 (Remote Services), T1078 og relevante teknikker for misbrug af privilegier.

### Panda · Linux

**Hvad maskinnoterne peger på:** Skjult websti, WordPress og mulighed for eskalation gennem sudo- eller SUID-programmer.

**Undersøg i denne rækkefølge:**

1. Scan webtjenester og undersøg WordPress-version, plugins og synlige endpoints.
2. Find skjulte webstier; læs indholdet og vurder, hvilke funktioner der er tilgængelige.
3. Ved shell: kontrollér konkrete sudo-regler og SUID-programmer.
4. Undersøg filsystemets attributter i stedet for at antage, at `chattr` findes.

```bash
curl -i http://<IP>/
curl -i http://<IP>/wp-login.php
gobuster dir -u http://<IP>/ -w /usr/share/wordlists/dirb/common.txt
sudo -l
```

**Særligt:** Webadministration kan give ændringsadgang uden at give systemadministration. Beskriv rettighedsspringene separat.

**MITRE:** T1190/T1078 og T1548, hvis observeret.

### Production · Linux

**Hvad maskinnoterne peger på:** Delte mapper over FTP/SMB med en SSH-nøgle, særskilt service på port 9001 og privilegeret brug af Git/andre værktøjer.

**Undersøg i denne rækkefølge:**

1. Enumerér FTP og SMB, og se efter filer, der ikke burde kunne læses anonymt.
2. Undersøg filernes type, ejer og tilladelser. Private nøgler i shares er alvorlige fund.
3. Identificér tjenesten på højporten; send ikke tilfældige kommandoer uden at forstå protokollen.
4. Efter shell: gennemgå bruger-overgange og sudo-regler, især interaktive programmer.

```bash
nmap -sV -p 21,22,139,445,9001 <IP>
smbclient -L //<IP> -N
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

**Særligt:** Et program, som åbner editor eller pager som root, kan udvide en ellers snæver sudo-tilladelse. Dokumentér præcis hvilken konfiguration der skaber risikoen.

**MITRE:** Unsecured Credentials (T1552), Valid Accounts (T1078), T1548.

### Shrek · Linux

**Hvad maskinnoterne peger på:** Web-API, FTP, `robots.txt`, SSH og flere mulige eskalationsveje, herunder sudo, SUID og Docker-gruppe.

**Undersøg i denne rækkefølge:**

1. Se efter API-parametre og `robots.txt`.
2. Undersøg filer, der eksponeres via FTP og web.
3. Hvis adgang etableres: undersøg `sudo -l`, gruppemedlemskaber og SUID-binarier.
4. Prioritér den vej, du bedst kan begrunde, frem for at afprøve alt på én gang.

```bash
curl -i http://<IP>/robots.txt
curl -i http://<IP>/api/search.php
id; groups
sudo -l
```

**Særligt:** En offentliggjort nøgle eller en gruppe med adgang til container-runtime er en fejlkonfiguration, der kan være vigtigere end et versionsbaseret exploit.

**MITRE:** T1552, T1078, T1548.

### Tyler · Linux

**Hvad maskinnoterne peger på:** SMB-fil med hints, LibreNMS på port 8080, mulig webinput-fejl, en SUID-editor og backup af passworddata.

**Undersøg i denne rækkefølge:**

1. Læs tilgængelige SMB-filer; adskil hints fra bevis på fungerende credentials.
2. Identificér LibreNMS-version og kontroller en relevant sårbarhedsbeskrivelse mod den faktiske installation.
3. Undersøg, om webinput bliver behandlet som kommando eller data, ved kontrollerede tests i labbet.
4. Efter shell: tjek sudo, SUID og adgangskontrol på eventuelle backupfiler.

```bash
smbclient -L //<IP> -N
curl -i http://<IP>:8080/
sudo -l
find / -perm -4000 -type f 2>/dev/null
ls -l /var/backups 2>/dev/null
```

**Særligt:** Passwordbackups er følsomme, selv når de er ældre. Problemet er både potentiel adgang og mangelfuld beskyttelse af data.

**MITRE:** T1190, T1552 og T1548, hvis forudsætningerne bekræftes.

## 4. Sådan arbejder du med kongen – uden at ødelægge spillet

Når du har den krævede rettighed, skal du følge konkurrencens aktuelle instruktion om **king-filen** og kontrollere scoreboardet. Kendte placeringer fra noterne:

- Linux: `/root/king.txt` (kan have særlige attributter på enkelte maskiner).
- H1: Medium: `C:\king.txt`.
- Offline: `C:\Users\Administrator\king-server\king.txt`.

**Forsvar prioriteret:** Luk den konkrete indgang, du brugte; gennemgå legitimationsoplysninger og unødvendige rettigheder; beskyt tjenester uden at ødelægge dem; kontrollér flere mulige indgange. Forsøg ikke at sabotere king-service, slette flags eller gøre systemet utilgængeligt. Permanente bagdøre, skjulte root-konti og destruktive ændringer er ikke nødvendige for at lære at forsvare en KoTH-maskine.

## 5. Hacking KoTH med AI: Mistral Vibe

Brug Vibe til **analyse**, ikke til uovervåget autopwn. Del et begrænset output, bed om én prioriteret handling, verificér resultatet og fortsæt.

**Prompt ved enumeration:**

> Jeg deltager i en autoriseret TryHackMe KoTH-runde. Her er mit Nmap-output: [indsæt]. Giv tre sandsynlige undersøgelsesspor, argumenter ud fra de observerede data, og angiv de næste manuelle kommandoer. Markér antagelser tydeligt.

**Prompt efter shell:**

> Jeg har en shell i en TryHackMe KoTH-lab. Her er `id`, `sudo -l` og SUID-listen: [indsæt]. Forklar hvilke rettighedsfejl der er relevante, hvad der mangler at blive verificeret, og hvilken MITRE ATT&CK-teknik det muligvis svarer til.

**Prompt efter runden:**

> Her er min rækkefølge af handlinger og resultater: [indsæt]. Lav en tidslinje, klassificér teknikkerne i ATT&CK, identificér beslutningsfejl og foreslå tre defensive kontroller.

**Tjekliste til AI-forslag:** Findes kommandoen i din version? Har du den rigtige IP og port? Er hypotesen understøttet af output? Følger handlingen reglerne? Hvad kan gå galt?

## 6. Faglig opsamling efter hver maskine

Lav fem korte noter, højst én linje pr. punkt:

1. **Indgang:** Hvilken tjeneste eller fejlkonfiguration gav første adgang?
2. **Privilegier:** Hvad muliggjorde et spring i rettigheder?
3. **Bevis:** Hvilke konkrete outputs underbygger konklusionen?
4. **MITRE:** Hvilke teknikker blev faktisk observeret, med teknik-ID?
5. **Forsvar:** Hvilken kontrol ville have brudt kæden tidligst?

## Kilder og forbehold

- [TryHackMe KoTH](https://tryhackme.com/games/koth)
- [Oprindelig maskinreference](https://github.com/andracs/koth-vejledning)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [GTFOBins](https://gtfobins.github.io/)
- [LOLBAS](https://lolbas-project.github.io/)

*Bemærk:* Maskinspecifikke noter er historiske spor fra den eksisterende README, ikke bekræftede live-resultater. Brug aldrig denne reference til systemer uden udtrykkelig tilladelse.
