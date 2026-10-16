# vbbutler-sdk

La libreria Java che parla con [vbButler](../README.md) al posto tuo.

Dichiari **un array di topic** e un handler; da quel momento non tocchi più HTTP, JSON, polling,
conferme né i tetti del protocollo.

---

## Includerla

```bash
mvn -f sdk/pom.xml clean install     # nel repository locale
```

```xml
<dependency>
  <groupId>com.vonbuzzi.butler</groupId>
  <artifactId>vbbutler-sdk</artifactId>
  <version>1.0.0-SNAPSHOT</version>
</dependency>
```

Java 21. Nel tuo classpath arrivano **quattro artefatti e nient'altro**: `jackson-databind` con
`jackson-core` e `jackson-annotations`, più `slf4j-api`. Un `maven-enforcer-plugin` fa fallire la
build il giorno in cui questa riga smettesse di essere vera.

---

## Usarla

```java
ButlerClient client = ButlerClient.builder()
        .endpoint("https://butler.internal:8443")
        .apiKey(System.getenv("BUTLER_API_KEY"))
        .clientId("vbauth-node-1")
        .topics("config.reload", "cache.flush")
        .onMessage(m -> apply(m.payload()))
        .build();

client.start();                                          // internal thread

PublishReceipt r = client.publish("config.reload", "CFG-RELOAD-01");

client.topics("config.reload", "cache.flush", "audit.rotate");   // at runtime

client.close();                                          // AutoCloseable
```

`client.isRunning()` dice se il poller **sta davvero girando**, non solo se è stato avviato: puoi
costruirci sopra un healthcheck.

Opzionali: `onError`, `onGiveUp`, `pollInterval`, `connectTimeout`, `requestTimeout`,
`maxBackoff`, `maxDeliveryAttempts`, `publishRetries`, `quarantineCapacity`, `transport`.

---

## Le tre cose che restano tue

**1. L'idempotenza, e non è delegabile.** La consegna è *at-least-once*: se una conferma si perde
in rete il messaggio torna al giro successivo. Nessuna libreria può rendere innocuo eseguire due
volte un comando al posto di chi lo esegue.

**2. Decidere se un messaggio è ancora attuale.** I messaggi vivono 30 giorni, e un client che
rientra dopo settimane riceve tutto l'arretrato in un colpo solo. `Message.receivedAt()` è
l'orologio **del server** — è un `LocalDateTime` senza fuso perché è ciò che il server manda, e
inventargli una zona sarebbe plausibile e sbagliato.

**3. `discard()` è una perdita di dati.** È l'unico percorso in cui un messaggio viene confermato
senza essere stato elaborato, e lo chiedi tu: la SDK non lo fa mai di propria iniziativa.

---

## Cosa fa per te, e che altrimenti rifaresti

| | |
|---|---|
| Il parametro `topic` **ripetuto**, non una lista con virgole | una lista con virgole sarebbe un topic solo, con un nome che non esiste, e riceveresti zero messaggi per sempre |
| Normalizzazione a minuscolo | `CONFIG.RELOAD` e `config.reload` sono lo stesso topic per il server |
| I tetti a **50 topic** e **1.000 id** | spezzati in automatico; un backlog di 4.000 messaggi si conferma da solo |
| Ordine cronologico **globale** | con molti topic arrivano più risposte, ciascuna ordinata solo al proprio interno |
| Backoff con jitter | un server giù per un'ora costa una dozzina di tentativi, non duecento |
| Il `401` che **non** è sempre "chiave sbagliata" | un guasto interno del server esce così anche con una chiave valida: la SDK riprova, e alza il livello di log dal decimo consecutivo |
| Retry della publish **solo** se l'invio non è partito | il server non deduplica: riprovare un timeout creerebbe un secondo messaggio che tutti eseguirebbero |
| La chiave validata **alla costruzione** | un a capo finale letto da un file di secret farebbe citare la chiave per intero al JDK, e quel messaggio finirebbe nei tuoi log |
| Un tetto su **quanto il server può farti allocare** | 16 MiB **per risposta**, `payload` ≤ 64.000, ≤ 1.000 messaggi **per risposta**: senza, una risposta ostile è un `OutOfMemoryError` nel tuo processo |
| Un tetto su **quanto può costarti leggerla** | 100.000 elementi JSON per risposta, contati *prima* di costruire l'albero: fra i byte di una risposta e l'heap che costa c'è un fattore trentacinque, quindi un tetto in byte da solo non bastava |
| Un tetto su **quanto un giro può trattenere** | 16.000.000 di caratteri: un giro fa una richiesta ogni 50 topic, e con 2.000 topic le quaranta risposte — ciascuna dentro il proprio tetto — esaurivano l'heap. Oltre il tetto il giro consegna ciò che ha e riprende dal blocco successivo: nulla va perso, per l'at-least-once. Ogni messaggio conta **il suo testo più 150 caratteri fissi**: senza, un server che svuota tutti i campi azzerava il contatore e il tetto non interveniva mai |
| Il poller che **sopravvive anche a un `Error`** | un `NoSuchMethodError` da un aggiornamento di dipendenza uccideva il thread in silenzio, e `isRunning()` continuava a dire `true` |

---

## Quando qualcosa va storto

`onError` riceve un `ButlerError`, e `consecutiveFailures` vuol dire **due cose diverse** a seconda
di `operation`:

- su `FETCH`, `ACK` e `PUBLISH` conta i **giri** falliti di fila, e un giro riuscito lo azzera:
  è così che puoi allertare al terzo `401` invece di aspettare il decimo;
- su `HANDLER` conta i fallimenti **di quel messaggio**, cioè quanto manca alla quarantena.

> Fino al 2026-09-10 il secondo valeva sempre `1`, quindi vedevi 1, 1, 1, 1, 1 e poi `onGiveUp`
> senza preavviso — proprio nel caso in cui il preavviso serviva.

Un `close()` mentre il tuo handler sta bloccando **non** è un fallimento: non produce `onError`,
non consuma il budget di quarantena, e il messaggio torna al riavvio. Se il tuo handler incarta
l'`InterruptedException` in un'eccezione tua, ripristina il flag del thread e la SDK lo capisce
lo stesso.

---

## Il messaggio velenoso

Un handler che fallisce **sempre** su un certo messaggio non lo confermerebbe mai, e il server lo
riproporrebbe per tutto il TTL: a 15 secondi di intervallo sono circa **172.800 esecuzioni fallite
in 30 giorni**.

Dopo cinque fallimenti consecutivi la SDK smette di consegnarlo e chiama `onGiveUp` **una volta
sola**. Non scarta nulla: sul server il messaggio resta, e un riavvio lo riprova da capo.

Non devi configurare né ripulire niente — funziona da sé. Se però vuoi costruirci sopra:

```java
for (QuarantinedMessage q : client.quarantined()) {
    if (canFixItNow(q.message()))  client.release(q.message().id());   // retry
    else if (isStale(q.message())) client.discard(q.message().id());   // discard
}
```

---

## Publish: quando riprova, e quanto ti fa aspettare

`publish()` riprova **solo** quando è certo che il server non abbia ricevuto nulla — porta chiusa,
host sconosciuto. Su un timeout no, mai: il server potrebbe aver scritto il messaggio, e un
duplicato lo eseguirebbero *tutti* gli altri client.

Fra un tentativo e l'altro **attende**, circa un secondo e poi due (con jitter). Con i tre
tentativi di default significa meno di quattro secondi in più nel caso peggiore — a fronte dei
dieci secondi di `connectTimeout` che un solo tentativo verso un server irraggiungibile ti costa
comunque. Senza quell'attesa i tre tentativi cadevano tutti dentro la stessa finestra di riavvio
del server, e il retry non serviva a niente.

Se **interrompi** il thread da cui hai chiamato `publish()`, l'attesa si ferma, l'eccezione ti
arriva subito e il flag di interruzione resta alzato: la SDK non se lo tiene.

> ### Una riga nel tuo script di avvio può annullare tutto questo
>
> Se la tua JVM parte con `-Djdk.httpclient.enableAllMethodRetry=true`, `HttpClient` **ritenta da
> sé anche le `POST`** quando una connessione fallisce, dentro `send()`, in un punto che la SDK non
> vede e non può classificare: ogni retry è un messaggio duplicato che *tutti* gli altri client
> eseguono. È esattamente ciò che la regola qui sopra esiste per impedire.
>
> Il default della JVM è **disattivata**, quindi non devi fare nulla — e se qualcuno l'attiva, la
> SDK te lo dice con un `WARN` all'avvio che nomina la property. È un vincolo d'ambiente come il
> TLS: non è una libreria a poterlo garantire.

---

## Logging

La SDK dichiara **`slf4j-api` e mai un binding**: il logging lo decide chi include il jar. In un
client Spring Boot i log finiscono in Logback senza fare nulla:

```properties
logging.level.com.vonbuzzi.butler.sdk=DEBUG
```

In un contesto **senza** alcun binding vedrai `SLF4J(W): No SLF4J providers were found` e la SDK
tacerà. È corretto e voluto: una libreria che si porta dietro un binding decide il logging di chi
la include.

La **chiave API non compare a nessun livello**, né in alcun `toString()`, né in un messaggio
d'eccezione o in una sua causa. **Questa riga è verificata da un test**, non solo affermata: la
suite registra un provider SLF4J minimale e asserisce sulle righe realmente emesse, stack trace
compresi. Fino al 2026-09-09 era falsa — una chiave con un a capo finale finiva in chiaro in una
riga `ERROR` a ogni giro di polling.

Un'avvertenza se sostituisci il trasporto (`ButlerTransport`, punto di estensione S-12): la SDK
ripulisce e tronca il testo delle eccezioni che riceve, ma non può sapere che una parte di quel
testo è un segreto. Se il tuo trasporto scrive la chiave nel messaggio di un'eccezione, quel
messaggio finisce nei log. È il rischio accettato **R-7**.

Nessun messaggio d'eccezione grezzo raggiunge una riga di log: se ne scrive il tipo per intero e il
messaggio ripulito dai caratteri di controllo e troncato, perché quel testo può venire dal server.

A `DEBUG` trovi la riga di fine giro: quanti messaggi sono arrivati, quanti sono stati confermati
e i **tre contatori** dell'ACK. `unknown` sale a `INFO` da solo, perché è il solo dei tre che
segnali qualcosa di anomalo — che tu sia rimasto giù troppo a lungo, o che il TTL del server sia
troppo corto per il tuo ritmo di polling.

> Il payload è dichiarato a `TRACE` in `SPEC.md` §9, ma **in codice non esiste ancora alcun
> `log.trace`**: oggi il payload semplicemente non si logga. Rilievo F-14 dell'audit, aperto.

---

## TLS, e come si scrive l'endpoint

Un endpoint `http://` verso un host remoto **non viene rifiutato**: la SDK logga un WARNING e
parte. La chiave viaggia in un header, quindi senza TLS chiunque sia sul percorso di rete la legge
e diventa qualunque client. È un requisito di deployment (`../SPEC.md` §12.3), non qualcosa che
una libreria possa garantire. Il loopback è esente: lì non c'è rete da intercettare.

**Un endpoint che non può funzionare viene invece rifiutato subito da `build()`**, ed è cosa
diversa dal punto qui sopra:

| Scritto così | Cosa succede |
|---|---|
| `https://butler.interno:8443` | va bene |
| `https://butler.interno:8443/` | va bene: la barra finale viene tolta |
| `https://butler.interno:8443/butler` | va bene: un context path si conserva |
| `http://butler.interno:8443` | va bene, **con un WARNING**: la chiave viaggia in chiaro |
| `butler.interno:8443` | **rifiutato**: manca lo schema |
| `https://h:8443/?x=1` · `https://h#x` | **rifiutati**: query e frammento finirebbero dentro il percorso |
| `https://utente:password@h` | **rifiutato**: le credenziali finirebbero nel log |

> Fino al 2026-09-10 lo schema dimenticato — il refuso più facile che esista — non produceva
> **nessun** avviso, e il client falliva ogni chiamata per sempre. Se hai imparato che un endpoint
> in chiaro produce un WARNING, il silenzio deve poter voler dire «tutto a posto»: adesso lo vuole
> dire davvero.

**Le credenziali nell'URL sono rifiutate perché la riga di WARNING nomina l'endpoint**: con
`https://utente:password@host` quella riga sarebbe la tua password, scritta nei log a ogni avvio.

---

## Sviluppo

```bash
mvn -f sdk/pom.xml clean verify        # 239 test
mvn -f sdk/pom.xml dependency:tree     # devono essere quattro artefatti

# verifica di contratto contro un vbButler vero, prima di ogni handoff
export BUTLER_E2E_API_KEY=la-tua-chiave        # da un file non versionato
mvn -f sdk/pom.xml verify -Pe2e \
    -Dbutler.e2e.endpoint=http://localhost:8080
```

> **La chiave sta in una variabile d'ambiente e non sulla riga di comando**, ed è una differenza
> che conta: gli argomenti di un processo li legge *qualunque* utente locale (`ps aux`,
> `/proc/<pid>/cmdline`), finiscono nella `~/.bash_history` e su un runner di CI di solito nel log
> della build, che è spesso più accessibile del secret store. Chi possiede la chiave può
> impersonare qualunque `clientId` (`../SPEC.md` §11.3). Senza la variabile le cinque verifiche
> vengono **saltate**, non fatte fallire.

I test unitari girano contro un trasporto finto o contro l'`HttpServer` del JDK: per costruzione
dicono quello che ci si aspetta. **`-Pe2e` è l'unico test che si accorge se il contratto del
server e questa libreria divergono.**

| File | Contenuto |
|---|---|
| `SPEC.md` | Design completo, Decision Log S-1…S-14, rischi accettati |
| `AGENTS.md` | Vincoli di perimetro: le dipendenze ammesse, i divieti, il perché |
| `plans/` | Piano esecutivo |
