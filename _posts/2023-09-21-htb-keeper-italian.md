---
title: HTB Machine - Keeper [ITALIANO]
categories: [writeups/walkthroughs]
tags: [pwning, hydra, linux, burpsuite, keepass, easy]
image:
    path: /assets/img/writeups/keeper/keeper.png
    alt: Pwning Keeper machine.
---

## Introduzione

Nonostante abbia già una certa conoscenza, da diversi mesi ho deciso di cominciare a studiare i moduli di [Hack The Box Academy](https://academy.hackthebox.com/), dai concetti più basilari a quelli più complessi e seguendo tutti i contenuti offerti da ciascun modulo. Ci sono oltre 30 moduli di teoria e pratica e ovviamente non li ho ancora completati tutti, anche perché ne esce almeno uno ogni mese e potrebbero richiedere un certo impegno e del tempo per essere completati.

Dopo aver studiato e completato un certo numero di moduli, ho deciso di dedicarmi un po' alle CTF offerte da HTB iniziando dagli Starting Points (mini CTF, a scopo introduttivo). 

"pwnare" queste semplici macchine mi ha fornito quella consapevolezza in più per passare a qualcosa di un po' più complesso, e quindi ho iniziato con una delle prime macchine liberamente accessibili dalla piattaforma di HTB, cioè **Keeper**.

Per chi non è del campo, cercherò di renderlo quanto più fruibile possibile ma vi servirà un po' di pazienza per leggerlo e provare a comprenderlo. Non dirò a cosa serve e come si utilizza ogni tool poiché servirebbe un intero corso.

Benvenuti in questo walkthrough!

## Keeper

Io non ho mai fatto CTF e, sebbene abbia da sempre avuto una certa curiosità, non mi ci sono mai dedicato. HTB ti fornisce solo un indirizzo IP (ad esempio, 10.11.8.217), e poi lascia a te, ovviamente, il compito di ottenere le flags per l'account utente e root. 

Non ci sono suggerimenti o altro, sei libero di procedere come preferisci, puoi aiutarti cercando su Google oppure, semplicemente, usando `man` per i tools (a patto che ti serva qualcosa nello specifico). 

Per collegarsi all'infrastruttura di HTB e cominciare a giocare, viene fornita una chiave `.ovpn`, e tramite essa puoi collegarti al server VPN dedicato:

```bash
sudo openvpn /path/to/key/lab.ovpn
```

Una volta collegati, possiamo accedere tranquillamente all'indirizzo IP sopracitato: 

![Address](/assets/img/writeups/keeper/address.png)

Notiamo già qualcosa, un link con la seguente frase:

*"To raise an IT support ticket, please visit tickets.keeper.htb/rt/"*

Cliccandoci, tuttavia, non riusciamo a visualizzare il presunto sito web specificato poco prima.

![Finding trouble](/assets/img/writeups/keeper/finding-trouble.png)

L'indirizzo è giusto, **perché non riusciamo a visualizzarlo?**

La risposta sta nel file [`/etc/hosts`](https://tldp.org/LDP/solrhe/Securing-Optimizing-Linux-RH-Edition-v1.3/chap9sec95.html). Questo file permette al nostro computer locale di connettersi ad un dominio con un indirizzo specifico e ha diversi casi d'uso ma solitamente viene utilizzato per ragioni di reindirizzamento o di blocco di siti web.

Noi però conosciamo sia l'indirizzo IP che il nome del dominio, quindi potremmo banalmente... 

![Hosts](/assets/img/writeups/keeper/hosts.png)

Difatti, se torniamo su Firefox, potremo finalmente visualizzare il sito web:

![Request Tracker](/assets/img/writeups/keeper/rt.png)

Ci viene presentato come una sorta di gestionale e, facendo una breve ricerca su Google, nella versione 4.4.4 questo [Request Tracker](https://bestpractical.com/request-tracker) si definisce, da [GitHub](https://github.com/bestpractical/rt), precisamente come un 

*"enterprise-grade issue tracking system. It allows organizations
to keep track of what needs to get done, who is working on which tasks,
what's already been done, and when tasks were (or weren't) completed."*

C'è poco da aggiungere. È probabile che una o entrambe le flags saranno dentro questa piattaforma o comunque conterrà degli indizi che ci porteranno a trovarle.

A prima vista mi vengono in mente alcune opzioni:

1. usare `nmap` e vedere quali servizi si nascondono dietro questo server. Magari uno di questi ha una qualche vulnerabilità oppure mi saprà dire qualcosa.

2. brute-forzare la pagina di login che abbiamo appena visto. Mi serviranno potenzialmente due tools, `burpsuite` e `hydra`.

### Output di nmap

Facciamo una prima scansione del server usando `nmap`:

```bash
nmap -sV 10.11.8.217 -p 1-10000
```

> L'indirizzo IP soprastante è soltanto un esempio.
{: .prompt-warning }

In questo modo dovremmo poter visualizzare la maggior parte se non tutti i servizi attivi presenti sulla macchina:

![Nmap output](/assets/img/writeups/keeper/nmap.png)

A quanto pare abbiamo ovviamente il web server (nginx 1.18.0) sulla porta 80 e la porta 22 aperta. Teoricamente potremmo accedere al server tramite SSH. Quindi, se dovessimo trovare una password all'interno del gestionale (al momento l'unica strada percorribile), essa potrebbe aiutarci nell'accedere al server.

### Login brute forcing

Fare brute-force è una pratica normalmente sconsigliata, difficilmente darà il risultato sperato, ma nulla vieta comunque di fare un tentativo.

E per fare questo tentativo, avremmo bisogno di BurpSuite (personalmente mi basta la Community Edition, quella free) per capire quali parametri utilizzare durante la richiesta POST e il tool Hydra per effettuare l'attacco. 

Su ParrotOS sto utilizzando la versione più recente del momento, la 2023.10.11, ma anche con versioni precedenti non ci saranno problemi. Avviamo quindi BurpSuite con le configurazioni di default e andiamo sul tab *Proxy*, qui premiamo su **"Intercept is off"** per avviarlo e poi apriamo il browser integrato cliccando su **"Open browser"**.

![BurpSuite](/assets/img/writeups/keeper/burpsuite.png)

Inseriamo nella barra degli indirizzi l'URL precedentemente fornito, ovvero: **http://tickets.keeper.htb/rt/**

![GET request](/assets/img/writeups/keeper/burpsuite-get.png)

E andiamo a premere su **"Forward"** per inoltrare la richiesta GET e avere renderizzata la pagina web. Quindi andiamo sul form di login, e proviamo ad inserire dei dati a caso, magari il classico *admin* come nome utente e *admin* come password. Al momento non ci interessa che siano giusti o sbagliati, ricordiamo che ci interessa capire quali parametri vengono utilizzati dal form, anche se potremmo vederli ispezionando la pagina web.

![POST request](/assets/img/writeups/keeper/burpsuite-post.png)

BurpSuite ci dice che viene effettuata una richiesta POST all'indirizzo `/rt/NoAuth/Login.html` e per il form vengono banalmente utilizzati parametri come `user` e `pass` oltre che `next` (che potrei interpretare come un cookie). 

Dunque, andiamo ad inoltrare la richiesta.

![Wrong login](/assets/img/writeups/keeper/wrong-login.png)

Naturalmente i dati di login sono errati.

Eppure anche questo ci fornisce un'informazione in più: **"Your username or password is incorrect"**. 

Questa frase sarà uno dei parametri che andremo a specificare su Hydra. Questo tool farà molteplici tentativi e continuerà se si ritroverà davanti questa frase.

Riassumendo, adesso sappiamo che parametri dare in pasto ad Hydra e utilizzando la più comune lista di utenti e password potremo provare ad effettuare l'attacco di brute-forcing.

Proviamo quindi questo comando:

```bash
hydra -L '/usr/share/seclists/Usernames/top-usernames-shortlist.txt' -P '/usr/share/wordlists/rockyou.txt.gz' tickets.keeper.htb http-form-post "/rt/noAuth/Login.html:user=^USER^&pass=^PASS^&next=e02d4fc263a2d8c8d9e28880494594d5:Your username or password is incorrect"
```

Dove

- `-L` indica un file con dentro una shortlist degli username più usati.
- `-P` contiene il path di un database con migliaia e migliaia di password, comuni e non.

E poi il resto riguarda la richiesta POST che Hydra andrà a fare finché non avrà trovato dei dati di login accettabili.

E lo ha fatto.

![Login data](/assets/img/writeups/keeper/login-data.png)

Palese che il focus di questa CTF non fosse il login per la piattaforma. Pare proprio che, secondo Hydra, il nome utente **root** e la password **password** permettano l'accesso al RT.

![Login success](/assets/img/writeups/keeper/login-success.png)

Abbiamo l'accesso root al gestionale. Semplice, mica male! 

### Dentro Request Tracker

Ci sono tantissime impostazioni e "girovagando" noto che sono presenti due utenti:

![User](/assets/img/writeups/keeper/user.png)

In particolare è molto probabile che l'account **lnorgaard@keeper.htb** servirà per accedere al server. Però non possiamo fare alcuna prova. Ancora ci serve una password.

Scavando e scavando ancora all'interno di questo sito, troviamo delle informazioni interessanti, ma tra tutte salta all'occhio:

![User password](/assets/img/writeups/keeper/user-password.png)

Ebbene sì, sembra proprio che abbiamo una password: **Welcome2023!**

### First try?

Non resta che provare ad accedere al server con questi dati:

- user: lnorgaard@keeper.htb
- password: Welcome2023!

Ci siamo, forse abbiamo trovato la prima flag.

```bash
ssh lnorgaard@keeper.htb
```

![User flag](/assets/img/writeups/keeper/user-flag.png)

E così è stato, abbiamo trovato la prima flag! Abbastanza facile, no?

### Non è ancora finita.

In qualche modo serve accedere all'account root della macchina. Non ci sono vulnerabilità degne di nota però all'interno della directory di lnorgaard sono presenti due ulteriori files:

![Keys](/assets/img/writeups/keeper/keys.png)

Svolgendo una breve ricerca **KeePassDumpFull.dmp**, come suggerisce lo stesso file, sembra essere un "full dump" di un database KeePass. Per chi non lo sapesse, [KeePass](https://keepass.info/) è un password manager open source utile per gestire una moltitudine di password.

**passcodes.kdbx** e nello specifico [`.kdbx`](https://keepass.info/help/kb/kdbx_4.html) è il file [crittografato](https://github.com/p-h-c/phc-winner-argon2#argon2) dentro cui sono presenti dei dati sensibili (usernames, passwords, note, etc...). Usa la funzione di password-hashing Argon2, non ci sono grandi speranze.

L'unica speranza (e l'unico indizio che abbiamo) per accedere a questo file è proprio **KeePassDumpFull.dmp**. Fortunatamente facendo un'altra ricerca su Google riesco a trovare una notizia molto molto interessante: pare che in KeePass <=2.54 sia presente una CVE in grado di farmi recuperare la master password da un memory dump (come nel nostro caso).

[CVE-2023-32784](https://nvd.nist.gov/vuln/detail/CVE-2023-32784)

*"In KeePass 2.x before 2.54, it is possible to recover the cleartext master password from a memory dump, even when a workspace is locked or no longer running. The memory dump can be a KeePass process dump, swap file (pagefile.sys), hibernation file (hiberfil.sys), or RAM dump of the entire system. The first character cannot be recovered. In 2.54, there is different API usage and/or random string insertion for mitigation."*

Questa è un'ottima notizia! Basterà trovare qualcuno che abbia creato una PoC e potremmo ottenere la nostra desiderata password.

Inizialmente avevo trovato questa PoC: **https://github.com/vdohney/keepass-password-dumper**

Il problema? Scritto in C#, funziona solo in ambiente .NET ed ho avuto qualche difficoltà ad installarmi l'ecosistema .NET su ParrotOS quindi ho preferito cercare altri porting, magari scritti in C oppure Python. 

Un altro po' di ricerche e trovo finalmente il porting in Python: **https://github.com/CMEPW/keepass-dump-masterkey**

Direi che è tempo di provarlo:

![PoC](/assets/img/writeups/keeper/poc.png)

Ha funzionato e come già anticipato nella descrizione della CVE, alcuni caratteri (come il primo) non possono essere recuperati, quindi non c'è molto da fare oltre a questo. Eppure abbiamo la maggior parte dei caratteri di questa, plausibile, frase: ●Mdgr●d med fl●de

Proviamo a cercare tutte le combinazioni su Google! 

Alla fine, sebbene ancora nel dubbio, solo di una combinazione ho avuto un risultato soddisfacente:

![Master password](/assets/img/writeups/keeper/master-password.png)

Pare che il nome del dolce "Rødgrød Med Fløde" sia il candidato ideale per il nostro DB di KeePass.

Non rimane altro che provare ad aprire **passcodes.kdbx** con questa potenziale password o una sua ulteriore combinazione. Per farlo ci si può installare il client ufficiale oppure KeePassXC tramite `apt`:

```bash
sudo apt install keepassxc
```

Provo "as it is" e non funziona, provo con le lettere iniziali minuscole e invece va!

![KeePassXC](/assets/img/writeups/keeper/keepassxc.png)

Abbiamo decrittografato il file `.kdbx`.

### Sorpresa!

Guardando dentro il database, nella tab Network trovo ciò che stavo cercando, la password per l'account root!

![DB content](/assets/img/writeups/keeper/db-content.png)

Corro subito a provarla:

```bash
ssh root@keeper.htb
```

Inserisco la password "F4><3K0nd!" e... non funziona. È errata. Qualcosa non va, mi è sfuggita qualcosa. E, in effetti, qualcosa mi è sfuggito davvero! Se esamino meglio la sezione "Notes", trovo effettivamente dettagli di quella che sembra una chiave SSH, generata in qualche modo da PuTTY. 

Leggendo in giro sul web trovo che [esiste un tool della suite di PuTTY, chiamato PuTTYgen che mi permette di creare delle chiavi SSH](https://github.com/Eugeny/tabby/issues/5145). 

Quindi, nelle note, ho una chiave SSH generata da PuTTYgen ma non nel formato che mi permetterebbe di accedere al server. 

Ho bisogno di una chiave `.pem` non di una chiave `.ppk`.

Indagando ulteriormente comprendo che tramite PuTTYgen posso esportare la chiave nel formato che mi interessa. 

Quindi, creo un file, `key.ppk` con dentro il contenuto mostrato sulla sezione Notes, poi apro una finestra di terminale e, per ottenere questo tool, immediatamente digito:

```bash
sudo apt install putty-tools
```

Installato, nella stessa directory eseguo il comando:

```bash
puttygen key.ppk -O private-openssh -o root.pem
```

![PuTTYgen](/assets/img/writeups/keeper/puttygen.png)

Non funziona, il formato della chiave è "too new". Allora cerco di capire qual è il problema facendo la solita ricerca su Google. Trovo questo [thread su superuser.com](https://superuser.com/questions/1647896/putty-key-format-too-new-when-using-ppk-file-for-putty-ssh-key-authentication) e vedo che viene usata una GUI per fare la conversione da un formato all'altro, ma io non ho una GUI a disposizione. Leggendo tutti i commenti però mi accorgo che una versione più aggiornata di PuTTYgen (la `0.75`) è in grado di fare la conversione senza alcun problema.

Io su ParrotOS ho installata la versione `0.74` di default. 

Sorpresa!

L'unica soluzione è dirigermi nel [sito web di PuTTYgen](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) e compilare manualmente la versione `0.75`.

Quindi, sostanzialmente:

```bash
wget https://the.earth.li/~sgtatham/putty/0.75/putty-0.75.tar.gz
tar -xf putty-0.75.tar.gz
cd putty-0.75/
./configure
make puttygen
```

E così ho ottenuto il mio eseguibile, `PuTTYgen v0.75`. A questo punto, ripetiamo lo stesso step seguito poco fa, ovvero:

```bash
puttygen key.ppk -O private-openssh -o root.pem
```

Ed ho ottenuto la chiave `root.pem`! Non resta che provarla e ottenere la flag.

![Root flag](/assets/img/writeups/keeper/root-flag.png)

Ci siamo, abbiamo finito. Abbiamo ottenuto l'accesso root e preso la flag. **Keeper è stata pwnata**.

## Conclusioni

Ci ho messo due ore per finire questa CTF. Per essere la prima (vera) volta non mi sembra che sia andata male. Ma non è stato facile. Spero che per voi sia stato un walkthrough interessante!