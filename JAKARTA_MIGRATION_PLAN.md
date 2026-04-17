# Piano Di Migrazione A Jakarta

## Obiettivo

Questo documento riassume una prima analisi del repository `jbpm` rispetto alla migrazione da namespace `javax.*` a `jakarta.*`, con particolare attenzione a:

- fattibilita' della migrazione
- attivita' consigliate
- criticita' tecniche e organizzative
- perimetro di automazione ottenibile con OpenRewrite

L'obiettivo non e' proporre un "big bang", ma un percorso di migrazione progressivo e verificabile.

## Executive Summary

Lo stato attuale del repository e' ibrido:

- diversi moduli dichiarano gia' dipendenze Maven `jakarta.*`
- una parte molto ampia del codice Java runtime e dei test usa ancora package `javax.*`
- alcuni `pom.xml` contengono metadata OSGi e configurazioni esplicite ancora legate a `javax.*`
- sono presenti descriptor XML, file di test e profili container che referenziano ancora namespace e property legacy

Conclusione pratica:

- la migrazione a Jakarta e' fattibile
- OpenRewrite e' appropriato come strumento principale per la prima fase
- OpenRewrite non basta da solo
- la migrazione completa del repo va considerata di complessita' medio-alta

## Stima Iniziale

### Valutazione sintetica

- Portabilita' attuale verso Jakarta: `4.5/10`
- Copertura attesa di OpenRewrite sul lavoro meccanico: `50-70%`
- Livello di rischio complessivo: `medio-alto`
- Strategia consigliata: migrazione per cluster di moduli, non all-in-one

### Dati emersi dall'analisi

- File Java runtime con riferimenti `javax.*`: circa `431`
- File Java di test con riferimenti `javax.*`: circa `683`
- File XML/properties/txt con riferimenti `javax.*`: circa `38`

Namespace legacy piu' presenti:

- `javax.persistence`: `995` occorrenze
- `javax.xml.bind`: `776`
- `javax.ejb`: `159`
- `javax.enterprise`: `122`
- `javax.jms`: `109`
- `javax.naming`: `102`
- `javax.transaction`: `77`

Questo conferma che il problema non e' limitato ai test o alle dipendenze Maven: il debito di migrazione e' distribuito su runtime, packaging e ambienti di test.

## Cosa Puo' Fare OpenRewrite

OpenRewrite e' molto utile per:

- sostituire import `javax.*` con `jakarta.*`
- aggiornare dipendenze Maven standard
- allineare annotazioni e API standard dove esistono recipe consolidate
- produrre una prima differenza massiva, utile per misurare la dimensione reale del refactoring

OpenRewrite e' poco o per nulla sufficiente da solo per:

- metadata OSGi in `maven-bundle-plugin` con `Import-Package` espliciti
- descriptor XML e configurazioni con valori `javax.*`
- profili di test containerizzati legati a WildFly/JBoss o spec legacy
- conflitti di classpath dovuti a dipendenze miste `javax` e `jakarta`
- casi in cui librerie terze non sono migrate in modo coerente

## Principali Criticita'

### 1. Repo in stato ibrido

Diversi moduli dichiarano dipendenze Jakarta ma il codice corrispondente e' ancora `javax.*`. Questo crea un rischio elevato di:

- build verdi solo in alcuni moduli
- mismatch tra API compile-time e classpath runtime
- errori di packaging o bootstrap non visibili nella sola compilazione

### 2. JPA e transaction sono trasversali

`javax.persistence` e `javax.transaction` sono diffusi in molti moduli core. Sono il primo cluster da trattare, ma anche quello con maggiore rischio di regressioni funzionali.

Moduli particolarmente esposti:

- `jbpm-persistence`
- `jbpm-human-task`
- `jbpm-runtime-manager`
- `jbpm-services`
- `jbpm-audit`
- `jbpm-query-jpa`

### 3. JAXB ancora molto presente

`javax.xml.bind` e' molto diffuso e impatta:

- serializzazione/deserializzazione
- test
- modelli XML
- integrazioni con payload e deployment metadata

OpenRewrite puo' aiutare sul rename, ma qui servira' attenzione anche alle dipendenze e ai provider effettivi.

### 4. OSGi e bundle metadata

Alcuni moduli hanno `Import-Package` e simili esplicitamente ancorati a `javax.*`. Questo e' un punto critico perche' la build puo' anche compilare, ma il bundle risultante restare inconsistente.

Esempi rilevanti:

- [jbpm-persistence/jbpm-persistence-jpa/pom.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-persistence/jbpm-persistence-jpa/pom.xml:196)
- [jbpm-runtime-manager/pom.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-runtime-manager/pom.xml:280)

### 5. Descriptor XML e configurazioni legacy

Sono presenti file di configurazione con riferimenti espliciti a namespace o property `javax.*`.

Esempi:

- [jbpm-test/src/test/resources/META-INF/kmodule.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-test/src/test/resources/META-INF/kmodule.xml:3)
- [jbpm-runtime-manager/src/test/resources/META-INF/kmodule.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-runtime-manager/src/test/resources/META-INF/kmodule.xml:4)
- [jbpm-services/jbpm-kie-services/src/test/resources/META-INF/kmodule.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-services/jbpm-kie-services/src/test/resources/META-INF/kmodule.xml:4)
- [jbpm-container-test/jbpm-in-container-test/jbpm-container-test-suite/src/test/resources/persistence.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-container-test/jbpm-in-container-test/jbpm-container-test-suite/src/test/resources/persistence.xml:21)

Questi elementi difficilmente vengono sistemati in modo affidabile con una sola recipe generalista.

### 6. Test containerizzati e ambienti applicativi

Il sottoalbero `jbpm-container-test` contiene profili e dipendenze con riferimenti a specifiche legacy `javax`, comprese dipendenze JBoss/WildFly e configurazioni specifiche di container.

Questo e' il punto a rischio maggiore in termini di effort, instabilita' e tempo speso in fix ambientali.

### 7. EJB/CDI/JMS

I moduli che usano `EJB`, `CDI` e `JMS` non richiedono solo rename import. Potrebbero emergere problemi su:

- bootstrap container
- wiring CDI
- test Arquillian / container-managed
- listener e produttori JMS
- packaging e dipendenze provided

### 8. Dipendenze doppie o residue

In vari `pom.xml` sono presenti esclusioni o dipendenze legacy `javax.*` ancora esplicite. Questo aumenta il rischio di classpath sporco e di transitive dependency difficili da isolare.

## Moduli Da Considerare Piu' Critici

Ordinati in prima approssimazione per attenzione richiesta:

- `jbpm-human-task`
- `jbpm-services`
- `jbpm-audit`
- `jbpm-runtime-manager`
- `jbpm-persistence`
- `jbpm-container-test`
- `jbpm-workitems`
- `jbpm-query-jpa`
- `jbpm-case-mgmt`
- `jbpm-xes`

## Strategia Consigliata

### Principio guida

Non migrare tutto insieme. Conviene dividere il lavoro in quattro stream:

- dipendenze Maven e convergenza del classpath
- codice runtime
- descriptor, metadata OSGi e configurazioni
- test e integrazioni containerizzate

### Sequenza consigliata

1. stabilizzare il perimetro e scegliere il target Jakarta
2. applicare OpenRewrite su un cluster piccolo ma centrale
3. correggere manualmente packaging e configurazioni
4. validare build e test del cluster
5. estendere ai cluster successivi
6. affrontare per ultimi container test, distribution e profili applicativi

## Piano Di Attivita'

## Fase 0 - Assessment E Preparazione

Obiettivo: definire scope e baseline tecnica.

Attivita':

- fissare il target della migrazione:
  - Jakarta EE 9 namespace only
  - oppure target piu' alto con upgrade librerie connesso
- identificare la versione minima di Java da supportare
- chiarire quali moduli devono essere:
  - compilabili
  - testati
  - rilasciabili
  - rimandati a una seconda ondata
- creare una baseline di build per cluster di moduli
- censire le dipendenze terze non allineate a Jakarta

Deliverable:

- matrice moduli vs stato migrazione
- lista dipendenze bloccanti
- definizione del perimetro di prima wave

## Fase 1 - Convergenza Dipendenze E Recipe OpenRewrite

Obiettivo: preparare una trasformazione ripetibile.

Attivita':

- introdurre o configurare plugin OpenRewrite nel parent o in un profilo dedicato
- selezionare recipe per:
  - `javax.persistence` -> `jakarta.persistence`
  - `javax.transaction` -> `jakarta.transaction`
  - `javax.xml.bind` -> `jakarta.xml.bind`
  - `javax.jms` -> `jakarta.jms`
  - `javax.ejb` -> `jakarta.ejb`
  - `javax.enterprise` / `javax.inject` / `javax.annotation`
- aggiornare dipendenze Maven standard coerenti con le recipe
- produrre un report delle trasformazioni candidate senza applicazione diretta su tutto il repo

Deliverable:

- recipe list condivisa
- branch tecnico di prova
- report dei moduli toccati

## Fase 2 - Migrazione Del Core Persistente

Obiettivo: migrare il primo cluster con maggior valore e forte riuso.

Cluster consigliato:

- `jbpm-persistence`
- `jbpm-query-jpa`
- `jbpm-audit`
- `jbpm-runtime-manager`

Attivita':

- eseguire OpenRewrite sui moduli del cluster
- correggere manualmente import, dipendenze residue e casi ambigui
- aggiornare metadata OSGi nei `pom.xml`
- verificare `persistence.xml`, property name e riferimenti di bootstrap
- eseguire compilazione e test selettivi del cluster

Criterio di uscita:

- il cluster compila
- i test essenziali del cluster passano
- nessun riferimento `javax.persistence` o `javax.transaction` rimane nel runtime del cluster

## Fase 3 - Migrazione Human Task E Services

Obiettivo: migrare i moduli con la maggiore densita' di riferimenti legacy.

Cluster consigliato:

- `jbpm-human-task`
- `jbpm-services`
- `jbpm-case-mgmt`

Attivita':

- applicare OpenRewrite per JPA, JAXB, JMS, EJB, CDI
- rivedere listener JMS, servizi EJB, producer e componenti CDI
- correggere test che fanno affidamento su bootstrap container o provider legacy
- bonificare dipendenze `provided` ancora miste

Criterio di uscita:

- compilazione pulita dei moduli
- rimozione dei riferimenti runtime `javax.*` principali
- test unitari e di integrazione locale stabili

## Fase 4 - Workitems, XES, Event Emitters E Moduli Accessori

Obiettivo: completare i moduli periferici ma ancora esposti.

Cluster consigliato:

- `jbpm-workitems`
- `jbpm-xes`
- `jbpm-event-emitters`
- `jbpm-document`

Attivita':

- migrare REST, mail, servlet, JAXB, activation
- validare serializzazione XML e compatibilita' payload
- gestire eventuali fix manuali su dependency tree o test

## Fase 5 - Descriptor, Configurazioni E Metadata

Obiettivo: eliminare i riferimenti legacy fuori dal codice Java.

Attivita':

- aggiornare `kmodule.xml`
- aggiornare `persistence.xml`
- aggiornare file `web.xml`, `weblogic.xml`, `tomcat-context.xml` e simili
- aggiornare property e chiavi `javax.persistence.*` dove richiesto
- rivedere metadata di installer e script collegati a moduli `javax.*`

Nota:

questa fase e' fondamentale. Una compilazione verde non basta se descriptor e ambienti restano legacy.

## Fase 6 - Container Test E Compatibilita' Ambientale

Obiettivo: stabilizzare i test in-container e i profili applicativi.

Attivita':

- migrare `jbpm-container-test`
- aggiornare profili ShrinkWrap/WildFly/Arquillian
- sostituire dipendenze JBoss spec legacy ove necessario
- verificare il comportamento nei container target

Nota:

questa e' la fase da pianificare per ultima, perche' e' la piu' costosa e la meno predicibile.

## Fase 7 - Hardening E Chiusura

Obiettivo: impedire regressioni future.

Attivita':

- aggiungere controlli CI per bloccare nuovi import `javax.*` nei moduli migrati
- documentare eventuali eccezioni consentite
- aggiungere report o check automatici di dipendenze legacy residue
- consolidare linee guida per i contributor

## Ordine Di Priorita' Consigliato

### Wave 1

- `jbpm-persistence`
- `jbpm-query-jpa`
- `jbpm-audit`
- `jbpm-runtime-manager`

### Wave 2

- `jbpm-human-task`
- `jbpm-services`
- `jbpm-case-mgmt`

### Wave 3

- `jbpm-workitems`
- `jbpm-xes`
- `jbpm-document`
- `jbpm-event-emitters`

### Wave 4

- `jbpm-container-test`
- `jbpm-distribution`
- `jbpm-installer`

## Rischi Di Progetto

- sottostimare il peso dei test containerizzati
- rompere il packaging OSGi senza accorgersene subito
- lasciare dipendenze `javax` transitive in moduli gia' migrati
- ottenere build locali verdi ma deployment instabile
- introdurre incompatibilita' tra moduli migrati e moduli ancora legacy

## Mitigazioni Consigliate

- lavorare per cluster coesi di moduli
- usare OpenRewrite in branch dedicati e con passaggi incrementali
- aggiungere controlli automatici su import e dipendenze residue
- separare la migrazione del codice dalla stabilizzazione dei container test
- trattare OSGi e descriptor come attivita' esplicite, non come dettaglio secondario

## Definizione Di Successo

La migrazione puo' essere considerata riuscita quando, per il perimetro concordato:

- il runtime non usa piu' package `javax.*` rilevanti
- le dipendenze Maven sono coerenti con Jakarta
- i descriptor e i metadata di packaging sono allineati
- i test critici passano
- la CI impedisce regressioni verso `javax.*`

## Raccomandazione Finale

La strada migliore e':

1. usare OpenRewrite come acceleratore principale
2. migrare prima i cluster core legati a persistenza e runtime
3. trattare OSGi, XML e container test come workstream dedicati
4. rimandare `jbpm-container-test`, distribution e installer alla parte finale del progetto

In sintesi: la migrazione e' realistica, ma non e' un semplice rename di package. Il successo dipendera' soprattutto dalla disciplina nel dividere il lavoro, validare per wave e non mescolare insieme refactoring meccanico e stabilizzazione ambientale.
