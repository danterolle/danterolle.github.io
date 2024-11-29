---
title: A New Beginning?
categories: [initial commit]
tags: [software, ruby, linux, bash]
---

# Benvenuti!

Era da un po' di tempo che avevo in mente di iniziare a scrivere qualche articolo tecnico che potesse essere utile alle community. Prima per mancanza di tempo poi per altre esigenze ma alla fine eccoci qui. Senza troppi giri di parole, pubblicherò alcuni articoli in italiano e/o in inglese, in base al contenuto.

Come visto nella pagina About, mi occupo principalmente di sviluppo per software open source usando tecnologie come Go, Python, React, etc... e prediligo l'utilizzo di IDE come GoLand/WebStorm/PyCharm, anche se, per progetti semplici, ho già VSCode configurato e per fare modifiche "al volo" utilizzo vi/vim/nano. Che si utilizzi MacOS o qualche OS GNU/Linux, ho una certa esperienza con entrambi e, da un punto di vista tecnico (ad eccezione dell'uso di certi software), non mi cambia nulla.

Quasi ogni anno tengo degli speech strettamente tecnici in alcuni eventi come i Linux Day o le GDG DevFest e I/O Extended, le cui presentazioni un giorno verranno caricate sul mio [profilo GitHub](https://github.com/danterolle/public-speaking).

Quindi benvenuti! E buona lettura!

# Come nasce questo blog? Un po' di troubleshooting

Non avendo grosse pretese, mi interessava soltanto che questa idea funzionasse, ovvero un tema per blog molto semplice ma con una grafica decente e con delle caratteristiche utili già disponibili out-of-the-box, senza plugins o altra roba da configurare.

Avendo già esperienza con la generazione di pagine web statiche; con SSG come Hugo, Pelican, Gatsby, NextJS e altri, senza nemmeno troppo impegno ho cercato, per diverso tempo, un tema che mi convincesse, fino a quando ne ho trovato uno su Reddit, costruito sopra Jekyll e attivamente mantenuto, oltre che con una serie di [caratteristiche interessanti](https://github.com/cotes2020/jekyll-theme-chirpy/#features).

Mai usato Jekyll, mai usato Ruby. 

Eppure la [live demo](https://chirpy.cotes.page/) di questo tema si presenta molto bene: c'è la procedura per clonare subito il progetto e vederlo immediatamente online su `username.github.io`, già tutto configurato, insomma pronto per scriverci dentro.

Se state usando un dispositivo con MacOS per procedere alla scrittura di articoli in locale e avere una preview del blog, però, potreste incontrare **alcuni problemi**:

## 1. Ruby su MacOS è già preinstallato.

Anche se la versione non è delle più recenti. E questo potrebbe essere un problema se il progetto su cui state lavorando necessita di dipendenze che richiedono di avere una versione più recente di Ruby (ad esempio, questo tema). Il consiglio, come meglio descritto anche nella documentazione di Jekyll, è di usare `brew` come package manager per gestire questi pacchetti. 

Il problema è che, semplicemente, sul terminale vi ritroverete:

```bash
$ which ruby
/usr/bin/ruby
```

Per quanto sia corretto, non è la versione che avete appena installato (il path non contiene `$HOMEBREW_PREFIX`). Sarebbe interessante capire per cosa Ruby viene usato in MacOS ma, nel dubbio, non conviene fare un upgrade diretto. Quindi meglio tenere le due versioni.

Per fare in modo che, permanentemente, venga utilizzata la versione installata tramite Homebrew, viene inoltre consigliato l'uso di `chruby`, da installare sempre tramite `brew`. L'importante però è seguire ciò che viene riportato nella documentazione di Homebrew riguardo al [pacchetto](https://formulae.brew.sh/formula/chruby), ovvero:

```bash
Add the following to the ~/.bash_profile or ~/.zshrc file:
    source $HOMEBREW_PREFIX/opt/chruby/share/chruby/chruby.sh

To enable auto-switching of Rubies specified by .ruby-version files,
add the following to ~/.bash_profile or ~/.zshrc:
    source $HOMEBREW_PREFIX/opt/chruby/share/chruby/auto.sh
```

Quindi, banalmente, per chi come me usa `zsh`:

```bash
echo "source $HOMEBREW_PREFIX/opt/chruby/share/chruby/chruby.sh" >> ~/.zshrc
echo "source $HOMEBREW_PREFIX/opt/chruby/share/chruby/auto.sh" >> ~/.zshrc
```

Si usa l'operatore `>>` quando si desidera aggiungere X contenuto alla fine del file. In questo modo dovreste tranquillamente vedere la nuova versione di Ruby:

```bash
$ which ruby
/opt/homebrew/bin/ruby
```

> `$HOMEBREW_PREFIX` corrisponde al path `/opt/homebrew`.
{: .prompt-info }

E procedere all'installazione di bundle, Jekyll, etc...

## 2. Non è detto che si stia utilizzando un sistema GNU/Linux

Inizialmente era bello vedere il proprio blog già online, pronto per essere popolato di contenuti. Dopo alcuni commit, pagina bianca. Perché? 

il deploy era ok (o almeno stando a quanto detto da GitHub Pages) e localmente vedevo perfettamente il blog. Pare che il problema riguardasse `pages-deploy.yml` all'interno di `.github/workflows`, insomma il file che "comunica" a GH Pages di buildare il progetto e renderlo accessibile online. Per qualche motivo viene usata un'immagine di Ubuntu e curiosamente, secondo la documentazione di questo tema, se non si usa un sistema GNU/Linux va fatta qualche semplice modifica al file Gemfile.lock e cambiata qualche impostazione su GH Pages. Il tutto è condensato in un [paragrafo](https://chirpy.cotes.page/posts/getting-started/#deploy-by-using-github-actions). 

Seguite le istruzioni, tutto per questo blog ha funzionato e sta funzionando a meraviglia con una facilità disarmante.

## Conclusioni

Jekyll è veloce, il live reload funziona benissimo e dubito che avrò bisogno di apportare modifiche a questo tema. Quasi viene voglia di capire come funziona l'ecosistema di Ruby e metterlo a paragone con altri linguaggi/piattaforme.