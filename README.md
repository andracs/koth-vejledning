# King of the Hill (KoTH)
**Praktisk vejledning**

## Hvad er CTF og KoTH?

**Capture the Flag (CTF)** er en konkurrence, hvor man løser sikkerhedsopgaver og finder skjulte *flags* som bevis. Opgaverne handler typisk om netværk, web, kryptografi, systemer og forensics.

**King of the Hill (KoTH)** er en særlig CTF-form: Alle angriber den samme maskine. Målet er at opnå administratorrettigheder (*root/SYSTEM*), blive konge og derefter forsvare sin position mod de andre deltagere.

Link: https://tryhackme.com/games/koth

På TryHackMe varer en runde 60 minutter. Man får point for at være konge (10 point pr. hele minut) og for at finde flags.

**Hvorfor KoTH?** Fordi du skal kombinere offensive og defensive færdigheder under tidspres. Du lærer at undersøge ukendte systemer, identificere sårbarheder, udnytte dem og efterfølgende lukke dem. Det minder om virkelige sikkerhedshændelser, blot komprimeret til en konkurrence.

**MITRE ATT&CK:** KoTH giver praktiske eksempler på Discovery (T1046), Initial Access (fx T1190), Execution (T1059), Privilege Escalation (T1548) og Defense Evasion. Brug ATT&CK til at beskrive, hvad du gør, og hvilke spor handlingerne efterlader.

**Hvordan bliver man bedre?** Øv enumeration, Linux/Windows, shells, `sudo`, SUID, GTFOBins og grundlæggende scripting. Gentag maskiner, før korte noter, og forstå dine kommandoer. Hurtighed kommer af genkendelse, ikke af tilfældige forsøg.

## 1. Sådan griber du konkurrencen an

**Før start:** Hav Kali/AttackBox klar, kontrollér VPN, og åbn terminal, browser og noter. Brug kun den IP-adresse, som konkurrencen tildeler.

| Tid | Aktivitet |
|---|---|
| 0-5 min | Find åbne porte og tjenester |
| 5-15 min | Identificér og afprøv en indgang |
| 15-25 min | Få shell og eskalér rettigheder |
| 25-40 min | Bliv konge, luk sårbarheder |
| 40-60 min | Forsvar positionen, find flags og alternative indgange |

Det er en prioritering, ikke en tidsgrænse. Bliv konge tidligere, hvis du kan.

## 2. Kommandoer du skal kende

**Enumeration: Find angrebsfladen**

| Kommando | Formål |
|---|---|
| `nmap -p- --open <IP>` | Find alle åbne TCP-porte |
| `nmap -sV -sC -p <PORTE> <IP>` | Identificér tjenester, versioner og standardinformation |
| `curl -i http://<IP>` | Undersøg HTTP-svar og headers |
| `gobuster dir -u http://<IP> -w <WORDLIST>` | Find skjulte mapper og filer |
| `smbclient -L //<IP> -N` | Undersøg tilgængelige SMB-shares |
| `showmount -e <IP>` | Undersøg eksporterede NFS-shares |

**Efter adgang: Forstå dine rettigheder**

| Kommando | Formål |
|---|---|
| `whoami; id; hostname` | Hvem er du, og hvor er du? |
| `sudo -l` | Hvilke kommandoer må du køre med forhøjede rettigheder? |
| `find / -perm -4000 -type f 2>/dev/null` | Find SUID-programmer |
| `getcap -r / 2>/dev/null` | Find Linux capabilities |
| `ss -tulpen` | Se lyttende tjenester |
| `ps aux` | Undersøg processer |

Et åbent portnummer er ikke i sig selv en sårbarhed. Et SUID-program er heller ikke automatisk en vej til root. Undersøg konfiguration, version og rettigheder.

**Bliv konge:** Når du har root på en Linux-maskine, skriv dit TryHackMe-brugernavn i `/root/king.txt` og verificér resultatet. På Windows varierer placeringen. Kontrollér altid, om filen har særlige attributter, inden du ændrer den.

**Forsvar:** Find den adgangsvej, du selv brugte. Ret sårbar konfiguration, fjern unødvendige privilegier og begræns adgangen uden at ødelægge legitime tjenester. Kontrollér, om andre angrebsveje stadig er åbne.

## 3. Den aktuelle maskinpool

TryHackMe har 14 maskiner: 12 Linux og 2 Windows. Tabellen viser kendte indgange og relevante undersøgelsesmetoder. Detaljer kan ændre sig, og nogle maskiner varierer mellem runder.

| Maskine | Hvad skal du kigge efter? |
|---|---|
| **Carnage** | PHP-upload, skjulte endpoints, `sudo -l` og SUID-programmer |
| **Fireworks** | Fuld portscanning; varierende tjenester og privilegier |
| **Food** | Usædvanlige porte, Telnet, API-kommandoudførelse, SUID |
| **Fortune** | Base64/ZIP, credentials, NFS og sudo-konfiguration |
| **H1: Easy** | CMS, webbaseret kommandoudførelse, cron og PATH |
| **H1: Hard** | XML/XXE, API, SSH, Docker og privilege escalation |
| **H1: Medium** | Windows, SMB, credentials og administratorrettigheder |
| **Hackers** | FTP/SSH, web-login, credentials og sudo |
| **Hogwarts** | Varierende porte, upload, SQLi og SUID |
| **Offline** | Windows, WebDAV, SMB og Windows-privilegier |
| **Panda** | WordPress, skjulte webmapper, sudo og SUID |
| **Production** | FTP/SMB, SSH-nøgler og usikre sudo-rettigheder |
| **Shrek** | Web-API, FTP, SSH, sudo og Docker |
| **Tyler** | SMB, LibreNMS, kommandoinjektion og SUID |

**Tommelfingerregel:** Undersøg alle porte. Identificér tjenesterne. Vælg den mest lovende indgang. Få adgang. Undersøg rettigheder. Bliv konge. Luk indgangen. Find flags.

Læs om privilege escalation på [GTFOBins](https://gtfobins.github.io/) (Linux) og [LOLBAS](https://lolbas-project.github.io/) (Windows). Brug [MITRE ATT&CK](https://attack.mitre.org/) til at forstå teknikkerne.

## 4. Hacking KoTH med AI (Mistral Vibe)

**Mistral Vibe kan fungere som din tekniske sparringspartner.** Den kan læse terminaloutput, foreslå undersøgelser, forklare exploits og hjælpe med fejlfinding. Du styrer processen og verificerer resultaterne.

**Under konkurrencen:**

1. **Enumeration:** Giv Vibe dit Nmap-output. Bed den prioritere tre mulige angrebsflader.
2. **Foothold:** Beskriv en mistænkelig tjeneste eller webfunktion. Bed om forslag til sikker verifikation.
3. **Privilege escalation:** Indsæt resultater fra `sudo -l`, SUID eller Windows-privilegier. Bed om forklaring af relevante GTFOBins-/LOLBAS-teknikker.
4. **Defense:** Forklar den sårbarhed, du anvendte. Bed om en konkret rettelse, der bevarer tjenestens funktion.
5. **Efter spillet:** Få Vibe til at analysere din fremgangsmåde, fejl og oversete muligheder.

**Eksempelprompt til Vibe:**

"Jeg deltager i en autoriseret TryHackMe KoTH-konkurrence. Her er mit Nmap-output: [indsæt]. Prioritér tre undersøgelser, forklar hvorfor de er relevante, og giv de næste manuelle kommandoer. Skeln mellem observationer og hypoteser. Angiv relevante MITRE ATT&CK-teknikker."

Arbejd trinvis. Bed om én undersøgelse ad gangen, kør kommandoen, og giv AI'en resultatet. Brug ikke autopwn-agenter eller automatiske angrebs- og hardening-scripts i offentlige spil. De er ikke tilladt.

AI kan foreslå kommandoer, som ikke virker, eller misforstå output. **Forstå, før du eksekverer.**

## 5. Spilleregler og læring

Hold maskinen brugbar. Du må ikke sabotere tjenester, angribe konkurrenterne, slette eller ændre flags, gennemføre DoS eller påvirke king-servicen på port 9999. Følg altid de aktuelle konkurrenceregler.

Efter runden skal du kunne besvare tre spørgsmål:

- Hvordan fik du adgang, og hvorfor virkede metoden?
- Hvilke MITRE ATT&CK-teknikker anvendte eller observerede du?
- Hvordan kunne angrebet være opdaget eller forhindret?

**Målet er ikke kun at vinde. Målet er at forstå et angreb så godt, at du også kan forhindre det.**

*Kilder: [TryHackMe KoTH](https://tryhackme.com/games/koth) · [KoTH-regler](https://help.tryhackme.com/en/articles/6498315-king-of-the-hill-overview) · [Maskinnoter](https://github.com/andracs/koth-vejledning) · [MITRE ATT&CK](https://attack.mitre.org/)*
