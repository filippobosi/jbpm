# Check Di Portabilita' Verso JBoss EAP 8 E WildFly Equivalente

## Obiettivo

Questo documento riassume il check svolto sul repository `jbpm` per valutare il porting verso:

- `JBoss EAP 8`
- una versione `WildFly` community equivalente dal punto di vista Jakarta EE

Il focus e' sulla portabilita' applicativa e sulle aree che oggi bloccano o rallentano un deploy credibile sul target.

## Riferimenti Di Piattaforma

### JBoss EAP 8

JBoss EAP 8 e' una piattaforma `Jakarta EE 10` certificata. Red Hat documenta esplicitamente la compatibilita' con Jakarta EE 10 in EAP 8.0 e 8.1.

Riferimenti:

- [JBoss EAP 8.0 introduction](https://docs.redhat.com/en/documentation/red_hat_jboss_enterprise_application_platform/8.0/html/introduction_to_red_hat_jboss_enterprise_application_platform/assembly_intro-eap_assembly-intro-eap)
- [JBoss EAP supported standards](https://access.redhat.com/articles/113373)
- [JBoss EAP 8 migration guide](https://docs.redhat.com/en/documentation/red_hat_jboss_enterprise_application_platform/8.0/html-single/migration_guide/migration_guide)

### WildFly equivalente

Sul lato community, il target equivalente e' un `WildFly` con supporto `Jakarta EE 10` e `Java 17`.

Riferimenti:

- [WildFly 30](https://www.wildfly.org/news/2023/10/18/WildFly-30-is-released/)
- [WildFly 31](https://www.wildfly.org/news/2024/01/25/WildFly-31-is-released/)
- [WildFly 39](https://www.wildfly.org/news/2026/01/16/WildFly-39-is-released/)

## Valutazione Sintetica

- **Porting a EAP 8**: fattibile
- **Porting a WildFly EE 10**: fattibile
- **Prontezza attuale del repository**: bassa
- **Rischio prevalente**: migrazione Jakarta e test/container integration, piu' che puro passaggio a JDK 17

## Conclusione Operativa

Il repository oggi non e' ancora in uno stato adatto a un porting diretto e pulito verso EAP 8. La strada piu' realistica e':

1. migrare il runtime a `jakarta.*`
2. stabilizzare il build su `JDK 17`
3. validare prima su `WildFly EE 10`
4. chiudere infine la compatibilita' di prodotto su `JBoss EAP 8`

## Stato Attuale Del Repository

Il repo presenta ancora:

- numerosi riferimenti runtime a `javax.*`
- configurazioni XML e descriptor legacy
- test e profili fortemente legati a WildFly/EAP 7 e ad ambienti containerizzati storici
- una combinazione pericolosa di codice `jakarta.*` e dipendenze ancora risolte a versioni che espongono `javax.*`

## Principali Aree Di Rischio

### 1. Namespace Jakarta non ancora completato

I moduli con maggiore densita' di riferimenti enterprise `javax`/`jakarta` sono:

- `jbpm-human-task`
- `jbpm-services`
- `jbpm-audit`
- `jbpm-container-test`
- `jbpm-runtime-manager`
- `jbpm-case-mgmt`
- `jbpm-persistence`
- `jbpm-query-jpa`
- `jbpm-workitems`
- `jbpm-xes`

Questo indica che il cuore applicativo e il layer enterprise non sono ancora consolidati sul nuovo namespace.

### 2. Descriptor e configurazioni legacy

Sono ancora presenti riferimenti espliciti legacy in file XML e descriptor:

- [jbpm-test/src/test/resources/META-INF/kmodule.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-test/src/test/resources/META-INF/kmodule.xml:3)
- [jbpm-runtime-manager/src/test/resources/META-INF/kmodule.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-runtime-manager/src/test/resources/META-INF/kmodule.xml:4)
- [jbpm-services/jbpm-kie-services/src/test/resources/META-INF/kmodule.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-services/jbpm-kie-services/src/test/resources/META-INF/kmodule.xml:4)
- [jbpm-container-test/jbpm-in-container-test/jbpm-container-test-suite/src/test/resources/persistence.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-container-test/jbpm-in-container-test/jbpm-container-test-suite/src/test/resources/persistence.xml:21)
- [web-non-ee.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-container-test/jbpm-in-container-test/jbpm-container-test-suite/src/test/resources/org/jbpm/test/container/archive/registerrestservice/WEB-INF/web-non-ee.xml:40)

Questi sono blocker concreti per EAP 8 / WildFly EE 10.

### 3. Forte dipendenza da JNDI, transaction e pattern application server

Il codice usa ancora molti pattern tipici di application server tradizionali:

- `InitialContext`
- `UserTransaction`
- lookup JNDI
- integrazione JMS
- componenti EJB/CDI

Esempi:

- [JPAWorkingMemoryDbLogger.java](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-audit/src/main/java/org/jbpm/process/audit/JPAWorkingMemoryDbLogger.java:443)
- [AuditLoggerFactory.java](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-audit/src/main/java/org/jbpm/process/audit/AuditLoggerFactory.java:92)
- [ContainerManagedTransactionManager.java](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-persistence/jbpm-persistence-jpa/src/main/java/org/jbpm/persistence/jta/ContainerManagedTransactionManager.java:89)

Questa area e' portabile su EAP 8, ma richiede che tutti i namespace e le dipendenze siano coerenti.

### 4. Container test fortemente legati a stack storici

La parte `jbpm-container-test` e' il principale punto di rischio.

Evidenze:

- il README parla ancora esplicitamente di WildFly e `EAP 7`: [jbpm-in-container-test README](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-container-test/jbpm-in-container-test/README.md:4)
- il README dei remote EJB test cita WildFly ed `EAP 7`: [jbpm-remote-ejb-test README](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-container-test/jbpm-remote-ejb-test/README.md:4)
- i profili Arquillian usano configurazioni datate come `wildfly23x`: [jbpm-container-test-suite/pom.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/jbpm-container-test/jbpm-in-container-test/jbpm-container-test-suite/pom.xml:546)

Questo rende non realistico considerare la suite container come "pronta" per EAP 8.

### 5. Problema di convergenza delle versioni API

Nel repository ci sono moduli che dichiarano dipendenze `jakarta.*`, ma Maven risolve versioni che espongono ancora namespace vecchi.

Caso concreto emerso su `jbpm-query-jpa`:

- `jakarta.persistence-api:2.2.3`
- `jakarta.xml.bind-api:2.3.3`

Queste versioni non sono coerenti con codice che importa `jakarta.persistence.*` e `jakarta.xml.bind.*`.

Quindi il problema non e' solo "cambiare gli import": bisogna anche riallineare il dependency management.

## Compatibilita' Per Area

### Core engine e persistenza

Stato:

- portabili
- non ancora pronti

Note:

- richiedono convergenza Jakarta su JPA, transaction, JAXB e audit

### Servizi applicativi e human task

Stato:

- portabili
- area ad alto effort

Note:

- combinano JPA, JMS, CDI, EJB, serialization e servizi remoti

### Work items e integrazioni

Stato:

- mediamente portabili
- richiedono bonifica namespace e dipendenze

Note:

- attenzione particolare a REST, servlet, webservice, mail e JAXB

### Test e supporto

Stato:

- non prioritari per il primo deploy
- importanti per hardening successivo

### Container test e remote EJB

Stato:

- area piu' critica
- non pronta per EAP 8 senza lavoro dedicato

## Raccomandazione Di Percorso

### Step 1

Portare il runtime applicativo a:

- `jakarta.*`
- `JDK 17`

senza coinvolgere subito i container test.

### Step 2

Validare il core su una piattaforma community equivalente:

- `WildFly EE 10`

Questo riduce il costo iniziale e consente un ciclo di feedback piu' rapido.

### Step 3

Chiudere la parte piu' enterprise/server-specific:

- descriptor
- packaging
- profili container
- test Arquillian
- remote EJB

### Step 4

Effettuare la validazione finale sul target di prodotto:

- `JBoss EAP 8`

## Valutazione Finale

### Risposta breve

Sì, il porting a `JBoss EAP 8` o a un `WildFly` equivalente e' realistico.

### Risposta operativa

Non e' un lift-and-shift. Oggi il repository richiede:

- migrazione Jakarta sostanziale
- riallineamento delle versioni effettive delle API
- correzione dei descriptor
- revisione dedicata della parte container/integration

### Raccomandazione

Usare `WildFly EE 10 + JDK 17` come ambiente tecnico intermedio di validazione e considerare `JBoss EAP 8` come target finale di hardening e certificazione interna.

## BOM E Versioning Check

Le versioni non sono definite solo nei singoli moduli. La catena di controllo e' questa:

### Parent esterno

Il parent principale del repository e':

- [pom.xml](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/pom.xml:6)

che eredita da:

- `org.kie:kie-parent:7.75.0-SNAPSHOT`

Questo parent e' esterno al repository ed e' il primo punto da controllare per:

- versioni plugin
- proprieta' come `version.org.kie`
- possibili versioni gestite per librerie Jakarta

### BOM importati nel root pom

Nel `dependencyManagement` del root vengono importati questi BOM:

- `org.drools:drools-bom`
- `org.jbpm:jbpm-bom`
- `org.kie.soup:kie-soup-bom`
- `org.kie:kie-dmn-bom`

Riferimento:

- [pom.xml dependencyManagement](/Users/lacquaviva/Lavoro/progetti/CRIF/jbpm/pom.xml:170)

### Implicazione pratica

Se un modulo dichiara:

- `jakarta.persistence-api`
- `jakarta.xml.bind-api`

ma non ne specifica la versione, la versione puo' arrivare:

1. dal parent `kie-parent`
2. da uno dei BOM importati
3. da altro dependency management transitivo

### Conclusione sul versioning

Per rendere coerente il porting non basta modificare i moduli singoli. Va verificata la gestione centrale delle versioni nel parent/BOM stack, perche' oggi il repository puo' risolvere coordinate `jakarta.*` a versioni ancora incompatibili con i package `jakarta.*` del codice.
