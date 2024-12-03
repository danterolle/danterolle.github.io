---
title: Appunti e note varie su Python e PySide6
categories: [initial commit]
tags: [software, python, gui, linux]
image:
    path: /assets/img/general/python-pyside.jpg
    alt: Possiamo usare Python per creare GUI?
---

## Introduzione

Da un anno circa, per questioni lavorative e non, ho deciso di immergermi nel mondo dello sviluppo di applicazioni desktop con interfaccia grafica. Ho sviluppato UI per anni con JS e React, quindi, inizialmente, mi ero convinto che cercare un tool con cui poter soddisfare questa volontà fosse più o meno semplice. In realtà, non è andata proprio così. Sebbene trovo che sia vero che con un po' di esperienza in JS si possano costruire ottime GUI grazie ad Electron, la sua pesantezza e la complessità di Nodejs (che forse non ho mai apprezzato e approfondito come merita) sono stati i due motivi i quali mi hanno fatto desistere dallo sperimentarci e creare qualcosa di interessante. 

Da quel momento, ho cercato una soluzione che quanto più si avvicinasse alle mie capacità tecniche e che mi permettesse di creare GUI senza perdere tempo dietro a configurazioni e simili, e passando da Go con il recente Wails/Fyne a Dart con Flutter, alla fine dei giochi, mi sono trovato molto bene con Python, in special modo con le librerie Qt. Ci sarebbero non poche ragioni per il quale, in questa situazione, potrei preferire Python a JS, e viceversa, o metterci in mezzo anche Go, ma no, il confronto non è lo scopo di questo post. E, del resto, tutto ciò sono preferenze personali: si tratterebbe solo di un'analisi prettamente personale. 

L'obiettivo di questo post, invece, è quello di riportare alcuni appunti e nozioni riguardanti l'uso di Python e il binding ufficiale di Qt, cioè PySide.

## Segnali e Slot

Quando si sviluppa un'applicazione interattiva, è fondamentale che i componenti della GUI possano comunicare tra loro in modo efficace. In PySide, questo compito viene svolto grazie al potente sistema di segnali e slot.

I segnali sono eventi emessi da un oggetto, mentre gli slot sono funzioni o metodi che rispondono a questi eventi. Banalmente, a livello di sintassi:

```python
signal.connect(slot) # per collegare un segnale e uno slot
signal.disconnect(slot) # per interrompere il collegamento tra un segnale e uno slot
```

Per chiarire ulteriormente il concetto, possiamo immaginare un interruttore della luce come un segnale: ogni volta che viene premuto, invia un comando, e la lampadina (il nostro slot) risponde cambiando il suo stato, accendendosi o spegnendosi.

```python
from PySide6.QtWidgets import QApplication, QPushButton

def on_button_clicked():
    print("Il pulsante è stato clickato!")

app = QApplication([])
button = QPushButton("Click!")
button.clicked.connect(on_button_clicked)  # Tramite connect viene "collegato" il segnale "clicked" allo slot
button.show()
app.exec()
```

Con Signal() è possibile anche creare dei segnali personalizzati, specificando persino un tipo di dato (bool, intero, stringa, etc...)

```python
from PySide6.QtCore import QObject, Signal

class MyObject(QObject):
    my_signal = Signal() # vuoto, int o str o altro, o anche più di un parametro "int, str"

    def emit_signal(self):
        print("Emetto un segnale")
        self.my_signal.emit()

obj = MyObject()
obj.my_signal.connect(lambda: print("Segnale ricevuto"))
obj.emit_signal()
```

## Architettura basata sugli eventi e il ciclo degli eventi

Le applicazioni sviluppate con PySide6 sono "event-driven", ovvero, basate sulla gestione di eventi. Questa architettura è il cuore del funzionamento di un'applicazione con GUI: ogni azione dell'utente, come un click su un pulsante o una modifica su un campo di testo, genera un evento che viene messo in coda e successivamente elaborato dal framework.

L'elemento chiave è il ciclo degli eventi (event loop), avviato dal metodo `.exec()` su un'istanza di `QApplication`:

```python
from PySide6.QtWidgets import QApplication, QLabel

app = QApplication([])
label = QLabel("Buona lettura!")
label.show()
app.exec()
```

Questo ciclo di eventi consente all'applicazione di ricevere ed elaborare continuamente gli eventi in sospeso. 

È importante notare che l’event loop viene eseguito nello stesso thread in cui è stato avviato (detto GUI thread), pertanto, qualsiasi operazione bloccante eseguita in questo thread può fermare temporaneamente l'elaborazione degli eventi, rendendo l'interfaccia utente non responsiva.

## Problemi di esecuzione sincrona

Le operazioni avviate dall'event loop sono, per impostazione predefinita, sincrone. Questo significa che ogni operazione deve terminare prima che il ciclo possa passare al prossimo evento. Se un’operazione richiede molto tempo (ad esempio, l'elaborazione di grandi quantità di dati o il download di file di grandi dimensioni), l’interfaccia si bloccherà fino al termine dell’operazione, causando potenziali problemi di usabilità. Questo comportamento è facilmente osservabile quando l’applicazione appare "congelata" o visualizza segnali di non responsività, come il cursore arcobaleno su MacOS oppure la classica finestra tendente al grigio su Windows che a volte fanno intendere che il programma sia crashato.

## Multithreading?

Per mantenere l’interfaccia reattiva durante l’esecuzione di operazioni lunghe, è essenziale spostare queste operazioni fuori dal thread principale. Qt, tramite PySide6, offre diverse interfacce per implementare il multithreading, che consente di eseguire compiti di lunga durata in thread separati. Questo approccio permette al thread principale di continuare a elaborare gli eventi dell’interfaccia grafica, evitando il problema evidenziato precedentemente.

I thread condividono lo stesso spazio di memoria, quindi possono essere avviati rapidamente e utilizzano poche risorse hardware. Questa condivisione della memoria rende facile lo scambio di dati tra i thread, ma può portare a situazioni di race conditions o segmentation faults, se più threads leggono o scrivono sulla stessa memoria simultaneamente.

Inoltre, in Python, i thread condividono lo stesso Global Interpreter Lock (GIL), che limita l’esecuzione del codice Python a un solo thread per volta. Tuttavia, questo limite non è molto problematico in PySide6, poiché il tempo di elaborazione può avvenire al di fuori di Python (tramite librerie esterne scritte in C++):

Sostanzialmente vengono messi a disposizione la classe `QThread` e il sistema di QtConcurrent (e QThreadPool con QRunnable) per lavorare con i threads. Questi approcci permettono di spostare operazioni pesanti al di fuori del controllo diretto del GIL. In particolare se il lavoro "CPU-bound" è implementato in C++ o utilizza librerie Python che rilasciano il GIL (come NumPy), permettendo effettivamente di sfruttare più risorse della CPU.

### Quando utilizzare QThread? Quando QThreadPool/QRunnable? 

TLDR;

Ha senso usare QThread:

- Se la task richiede frequenti comunicazioni con altre parti dell'applicazione tramite segnali e slot.
- Per task complessi o lunghi che devono essere separati dal thread principale.

Ha senso usare QRunnable e QThreadPool:

- Per task brevi, ripetuti e indipendenti.
- Quando bisogna eseguire molti task paralleli senza gestire manualmente i threads.

#### QThread

QThread è una soluzione ideale quando un task richiede un ciclo di eventi attivo. Questo è spesso il caso quando c'è una task che deve comunicare frequentemente con la GUI o altre parti dell'applicazione utilizzando segnali e slot. Si può anche definire un comportamento specifico sovrascrivendo il metodo `run()`, oppure spostare un worker personalizzato su un thread usando `moveToThread()`. Usare QThread permette quindi di spostare il lavoro in un thread separato, dove è possibile mantenere un ciclo di eventi indipendente. Difatti potrebbe essere adatto a task complessi o di lunga durata che necessitano di aggiornamenti continui, come l'elaborazione di dati con un feedback in tempo reale.

```python
from PySide6.QtCore import QThread
import time

class WorkerThread(QThread):
    def run(self):
        for i in range(5):
            time.sleep(1)
            print(f"Step {i+1}/5 completato")

worker = WorkerThread()
worker.start()
```

Altro esempio:

```python
from PySide6.QtCore import QObject, QThread, Signal
import time

"""
La classe Worker eredita da QObject per accedere a funzionalità
come segnali e il supporto per i thread.
"""
class Worker(QObject):
    finished = Signal()
    progress = Signal(int)

    def do_work(self):
        for i in range(5):
            time.sleep(1)
            self.progress.emit(i + 1)
        self.finished.emit()

worker_thread = QThread()
worker = Worker()

"""
Questo metodo sposta l'oggetto worker nel contesto del thread. 
Una volta spostato, i metodi dell'oggetto verranno eseguiti nel thread separato, 
evitando di bloccare il thread principale (dove risiede la GUI).
"""
worker.moveToThread(worker_thread)
worker_thread.started.connect(worker.do_work)
worker.finished.connect(worker_thread.quit)
worker.progress.connect(lambda p: print(f"Step {p}/5"))

worker_thread.start()
```

#### QRunnable/QThreadPool 

QRunnable e QThreadPool, invece, sono pensati per task che non richiedono un ciclo di eventi. QRunnable rappresenta un'unità di lavoro (worker), mentre QThreadPool gestisce un pool di thread riutilizzabili per eseguire queste unità in parallelo. Questo approccio è utile per attività semplici, come calcoli brevi o operazioni ripetitive, in cui la comunicazione frequente con altre parti dell'applicazione non è necessaria.

```python
from PySide6.QtCore import QRunnable, QThreadPool
import time

class SimpleTask(QRunnable):
    def __init__(self, task_id):
        super().__init__()
        self.task_id = task_id

    def run(self):
        time.sleep(1)
        print(f"Task {self.task_id} completato")


"""
globalInstance() serve per ottenere una singola istanza globale della classe QThreadPool, 
evitando di creare manualmente più istanze separate. Questa istanza globale è condivisa 
in tutto il programma e gestisce i thread per eseguire i task in parallelo
"""
pool = QThreadPool.globalInstance()
for i in range(5):
    pool.start(SimpleTask(i))
```

I thread creati da QThreadPool sono riutilizzabili, evitando l'overhead di creare e distruggere nuovi thread ogni volta. Motivo per cui può essere utile per applicazioni con un certo numero di task brevi.

Di QThreadPool può essere interessante notare che dispone di alcuni metodi e parametri per il controllo dei thread, ad esempio:

- `activeThreadCount()` per monitorare quanti thread sono attualmente in esecuzione;
- `setMaxThreadCount()` per configurare il numero massimo di thread nel pool;
- `.HighPriority` per dare precedenza ad uno specifico task;
- QRunnable, di default, è distrutto automaticamente dopo l'esecuzione, tuttavia, si può invocare `setAutoDelete(False)` per riutilizzare la stessa istanza;
- `waitForDone()` per bloccare il thread principale finché tutti i task nel pool non sono completati;

Non ho ancora avuto modo di sperimentare QtConcurrent, quindi non verrà trattato in questo articolo.

## Conclusioni

La documentazione di Qt e PySide è incredibilmente vasta e gli argomenti esposti qui sono solo un piccola (ma importante, a mio parere) parte di ciò che offre questa libreria. Chissà, magari in futuro aggiungerò ulteriori contenuti al fine di poter rendere queste parole degli appunti interessanti nello sviluppo di GUI con PySide6.