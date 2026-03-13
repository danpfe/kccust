# Analys Keycloak

## Observationer

I huvudsak ser vi att Keycloak tidvis kommer till ett tillstånd där lasten till synes blir så stor att vi hamnar i ett scenario där en nods
fall drar med sig övriga noder.

Från tidigare dagar har vi med framgång hittat att "datakällan" (`KeycloakDS`) inte lyckades allokera tillräckligt med anslutningar mot
Oracle, ännu tidigare ovserverades att det stora overhead i AppDynamics (där varje "hit" mot en metod måste skickas till appdyn-servern)
utlöste en "tipping-point".

Vi har också observerat att vissa klienter beter sig osedvanliga och som i viss mån skyddas genom "max sessions", men vi har idag på kvällen
också uppmärksammad klienter som bombarderar Keycloak utan att ens lyckas etablera en session (över 50k anrop per timme).

Vad vi dock ser numera att Infinispan-fel ökar i samband med att klustret hamnar i ett osunt tillstånd.

## Undersökningar

### Minne

Vi kan inte se vad som händer när Keycloak går ner, detta beror på att Native Memory Tracking eller Garbage Collection Logging inte är
aktiv. Förslaget är att aktivera det framgent. Under aktiv kontroll såg minnesutnyttjandet sunt ut.

### Processorkraft

Vi ser tidvis hög last på processorerna, men har redan gett så mycket mer CPU att vi inte kan se att vi når ett tak.

### Körtidslast

Vi ser att diverse kunder skapar onormalt mycket mer last än rimlig (D-konton som har 1000+ sessioner utan att logga ut normalt, klienter
som bombarderar servern med fel input som bara leder till HTTP 401).

### Cachning

Resursbrist tyder på felen vi får. Nätverket kan anses vara uteslutet då det är CNI-nätet som berörs och om det fanns påtagliga problem där
misstänker vi att många andra skulle ha märkbara problem.

## Förslag

### CPU

Noderna (dvs podar) ska som "request CPU" ha det som anses vara normal last. Om samtliga podar har en genomsnittlig last om 1.5 cores så
ska requests också vara på 1.5 cores. Detta för att garantera att Podar inte blir felschedulerade. Vidare kan för låga requests leda till
att Kuberntes CPU schedulern i förtid stoppar Podar då man hamnar på en nod som eventuellt har andra CPU-intensiva applikationer (men som
i sin tur är ärliga med vad de anser är normalt). "Requested CPU" = "What we need", "CPU Limits" = "If shit hits the fan, lets hope there
is something left on the node". Dvs requested CPU = observerad normal CPU-åtgång.

### Minne

Vi har inga belägg alls att Garbage Collection är ett problem, men vi ser timeouts och vi kan iaf misstänka att detta är fallet. RHSSO 7.6
kör ParallelNewGC och ParallelOldGC. Dessa är "throughput collectors", dvs skräpsamling fördröjs så länge det är möjligt för man lägger mest
värde på att processera så mycket som möjligt så länge minne finns. Vid stora minnesmängder kan det alltså betyda att det finns en märkbar
paus då GC till slut slår in.

Detta undantaget kör man i ett läge där skillnaden mellan garanterad minne (requested) och potentiellt tillgängligt minne (limits) inte
är likadant. Det är få programmeringsspråk som klarar att minne som låtsas vara tillgängligt inte finns, det är normalt sett skräddasydda
"Native Languages" (såsom C) med speciell kod som kan hantera det.

Vidare har vi observerat att huvudsaken av tiden ges åt applikationen medans bara 1% av tiden får ges åt GC (GC_TIME_RATIO=99). Om vårt
antagande är att GC-pauser kan utlösa problemet så bör vi sänka -XX:GCTimeRatio (GC_TIME_RATIO) så att vi får lite mer luft åt GC. Beräkningen
är (1/(1+GC_TIME_RATIO))*100) för att komma till önskad procentsats. Vi kan dessutom ge mer Heap (detta kan dock i sin tur leda till problem om vi har en Native Memory-läcka) genom att sätta -XX:MaxRAMPercentage=50 (eller som minst något högre än 20, som är default i en containermiljö).

### Distribuerad Cache

Keycloak är i betydande omfattning beroende av att den interna distribuerade cachen fungerar och externa försök har visat att det kan finnas
problem. Problemen kan givetvis bero på ovan punkter, men en ytterligare observation i sammanhanget är viktig: Infinispan kräver "quorum".
Dvs om ett jämnt antal noder kommer fram till samma slutsats om "the state of the world" måste det finnas en nod som kan rösta åt något håll.
3 alt 5 noder ska därför vara tillgängliga (och bör inte ändras i panik, detta förvärrar ombalanseringen). Att skala upp vid driftstörningar
från 3 till 4 kan potentiellt leda till problem. Vidare kan ombalansering, omstarter, nodnambyten leda till svårigheter, varför det i en
Kubernetes-miljö är rådsamt att köra Infinispan (och således
Keycloak) som "StatefulSet". Detta då podarna inte får slumpmässiga namn utan dom behåller stabila namn (JDBC_PING sparar hostnamnet!).
Förvisso förklarar detta inte hela situationen, men bör tas i beaktan.

## Ytterligare förbättringsförslag

### Kunder

Vi bör analysera loggar kring kunder som beter sig alldeles för dåliga (tiotusentals felaktiga anrop per timme bör ageras på; likaså
tusentals nya sessioner med samma användare). Även fast en enskild kund inte leder till en tipping-point så kan många samtidiga kunder med
ett orimligt beteende orsaka att lasten ökar till orimliga nivåer: Eventuellt skulle man kunna throttla klienter (via WAF) som ständigt orsakar HTTP 401-anrop i framtiden?

### Analysmöjligeheter

För att bättre kunna förstå vad som händer bör vi se till att vi kan logga minnesåtgången. Aktivera NMT och Stdout-GC genom följande flaggor:

```bash
-verbose:gc -XX:+PrintGCDetails \
-XX:+PrintNMTStatistics -XX:NativeMemoryTracking=detail \
XX:+UnlockDiagnosticVMOptions
```

