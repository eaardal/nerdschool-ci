# Oppstart

### Nødvendige ting

- Java
- Maven
- Git
- Tålmodighet
- Godt humør

### Hent Java-løsningen

1. Gå til [https://github.com/eaardal/nerdschool-ci](https://github.com/eaardal/nerdschool-ci) og bruk `Fork`-knappen oppe til høyre. Fork repositoriet til ditt repo (kopier repositoriet over til ditt repo).
2. Når det er forket, hent ned repositoriet lokalt:

`git clone <url til repository du nettopp forket>`

### Skaff gruppe-spesifikke innstillinger

For å jukse litt kjører vi alle løsningene på byggeservere, og ikke på egne servere som vi ville gjort i virkeligheten. Men siden vi kommer til å kjøre flere løsninger side om side må de ha sin egen, unike port. Du får et tildelt portnummer av instruktør.

Når alt kjører i TeamCity vil addressen til din løsning være: `http://nerdschool.cloudapp.net:PORT/`

### Bygg og start løsningen lokalt for å se at det fungerer

`mvn clean install jetty:run`

Om du går til `http://localhost:8080/` i nettleseren skal du nå se en "Hello World" side.

# TeamCity Login

#### Url: [http://nerdschool.cloudapp.net/](http://nerdschool.cloudapp.net/) 
#### Bruker: public
#### Passord: "red green refactor"

## Plan

Målet er at når man sjekker inn kode på en git branch i GitHub, så skal et bygg utføres i TeamCity. Bygget vi skal jobbe med her består av følgende steg:

1. Kompiler koden og kjør tester
	- TeamCity poller git repositoriet hvert 30 sekund
	- Når en endringer er funnet, kopierer TeamCity alle filene i git repositoriet over til byggeserveren til en mappe på disk
	- Når koden er kopiert, utføres byggetrinnene vi angir, i dette tilfellet vil det være å kjøre en Mavn kommando `mvn clean install` som vil kompilere og kjøre tester.
3. Pakk løsningen (for hvert miljø appen skal ut til)
	- Når vi har bekreftet at løsningen er gyldig (kompilerer + kjører tester), pakker vi hele løsningen sammen i en `.war` fil, som skal deployest til web serveren.
4. Kjør løsningen (for å minske kompleksiteten kjører vi appen på byggeserveren her)
	- I en virkelig løsning vil vi kopiere `.war`-filen til en annen server for å bli kjørt der, men for enkelhets skyld kjører vi løsningen på byggeserveren her.
	- For å kjøre løsningen bruker vi webserveren Jetty som vi gir `.war` filen til.  

## Forberedelser

### Velg et gruppenavn

Siden vi kommer til å ha flere like konfigurasjoner side ved side må vi ha et unikt navn en del steder. Gruppenavnet bør være kort og ikke ha mellomrom. Vi kommer til å referere til gruppenavnet en del steder videre med `<gruppenavn>`-taggen.

### Opprett prosjekt
Hver gruppe må sette opp sitt eget sub-prosjekt

**Merk: Vi refererer til dette prosjektet som gruppeprosjektet videre**

Logg på TeamCity.

1. Velg `Administration` oppe til høyre 
2. Trykk på `Nerdschool Training` prosjektet
3. `Create subproject`
4. Name: `<gruppenavn>`

### Opprett parametere
Parametere er systemvariabler som referer til verdier, navn eller mapper på byggeserveren som vi kan bruke under oppsettet. F.eks så referer variabelen `%teamcity.agent.work.dir%` til mappen `C:\TeamCity\buildAgent\work` på byggeserveren, som er rotmappen for der filer TeamCity henter fra git blir lagt. 

Vi må opprette noen slike variabler som er unike for *gruppeprosjektet*.

1. På Admin-siden til gruppeprosjektet du nettopp opprettet, velg `Parameters` i menyen til venstre og legg til følgende parametere:

<table>
	<thead>
		<tr>
			<td>Name</td>
			<td>Value</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>
			 	GRUPPENAVN.app
			</td>
			<td>app</td>
		</tr>
		<tr>
			<td>
			 	GRUPPENAVN.app.target
			</td>
			<td>app\target</td>
		</tr>
		<tr>
			<td>
			 	GRUPPENAVN.dropfolder
			</td>
			<td>%teamcity.agent.work.dir%\GRUPPENAVN\</td>
		</tr>
		<tr>
			<td>
			 	GRUPPENAVN.dropfolder.app.target
			</td>
			<td>%GRUPPENAVN.dropfolder%%GRUPPENAVN.app.target%</td>
		</tr>
	</tbody>
</table>

### Opprett VCS Root (Version Control System = Git)

1. `Create VCS root`
2. Type of VCS: Git
3. VCS root name: Valgfritt, anbefaler å bruker GitHub repository navn
4. Fetch URL: `.git` url fra GitHub (f.eks `https://github.com/eaardal/eaardal.github.io.git` - du finner den nede til høyre på GitHub repoet's fremside)
6. Test connection

## Build Configuration: Compile & Test

Ofte vil man skille mellom Compile og Test i en slik prosess, dvs ha disse i to separate trinn, der Test-trinnet bare kjører om Compile-trinnet er ok. I dette tilfellet bruker vi Maven for å compile og teste i samme steg.

Maven (og Java) er installert på byggeserveren, så alle kommandolinjefunksjoner vi kan gjøre med Maven og Java på vår utviklermaskin kan gjøres på byggeserveren, via TeamCity. (Det vil også si at om vi tar inn andre avhengigheter i et prosjekt senere, som f.eks Node/npm, så må disse også installeres på byggeserveren).  

Kommandoen for å bygge med Maven er `mvn clean install`, gitt at man står i mappen med `POM.xml` filen, som er Maven sitt register over avhengigheter og konfigurasjon.

1. På admin-fremsiden til *gruppeprosjektet*, velg `General Settings` og `Create build configuration`
2. Name: `Compile & Test` -> `Save`

### VCS Root

1. I menyen til venstre velg `VCS Root` -> `Attach VCS root`
2. Velg git-konfigurasjonen tilhørende din gruppe i nedrekkslisten
3. Trykk `Attach`-knappen for å legge den til

### Build Step 1: Compile & Test

1. I menyen til venstre velg `Build Steps` -> `Add build step`
2. Runner type: `Maven`
3. Step name: `Compile & Test`
4. Goals: `clean install`
5. `app\POM.xml`.
6. `Save`

### Build Step 2: Copy to dropfolder

1. `Add build step`
2. Runner type: `Command line`
3. Step name: `Copy to dropfolder`
4. Execute step: `If all previous steps finished successfully`
5. Working dir: `%teamcity.agent.work.dir%`
6. Custom script: `xcopy %teamcity.build.default.checkoutDir% %<gruppenavn>.dropfolder% /s /e /Y`. Her bruker vi en av *Parameterene* vi lagde i forberedelsene. Dette steget er nødvendig fordi de forskjellige byggekonfigurasjonene i TeamCity befinner seg i ulike mapper på byggeserveren, så for at `Package`-steget (som vi ikke har laget enda, men som blir neste steg i rekken) skal kunne pakke filene fra `Compile & Test`-steget (som vi lager nå), så må vi kopiere filene til en felles mappe der begge byggekonfigurasjonene kan nå filene. Derfor, når vi har kompilert og testet løsningen, så kopierer vi rubbel og bit til en "dropfolder" slik at neste byggesteg kan fortsette.
7. `Save`

### Trigger: VCS Trigger

1. I menyen til venstre velg `Triggers` -> `Add new trigger`
2. `VCS Trigger`
3. `Save`

Nå skal TeamCity automatisk se etter endringer i git repoet hvert 60. sekund og sette i gang `Compile & Test` byggekonfigurasjonen. Gå til `Projects`-siden og se at bygget starter og fullfører ok. Når det har fullført ok, prøv å  start det manuelt med `Run` knappen.

## Build Configuration: Package

1. På admin-fremsiden til *gruppeprosjektet*, velg `General Settings` og `Create build configuration`
2. Name: `Package` -> `Save`

### Build Step 1: Package .war

1. I menyen til venstre velg `Build Steps` -> `Add build step`
2. Runner Type: `Maven`
3. Step name: `Package war`
4. Goals: `compile war:war`
5. Path to POM file: `%<gruppenavn>.dropfolder%\app\POM.xml`. Nå refererer vi til POM.xml filen som ligger i dropfolderen som forrige byggesteg ("Copy to dropfolder") gjorde klar for oss.
6. `Save`

Dette vil kjøre Maven kommandoen `mvn compile war:war` som egentlig er å kompilere løsningen igjen, og pakke den sammen til et `.war`-arkiv.

### Build Step 2: Copy to deploy folder

1. I menyen til venstre velg `Build Steps` -> `Add build step`
2. Runner type: `Command Line`
3. Step name: `Copy to deploy folder`
4. Execute step: `If all previous steps finished successfully`
5. Custom script:

```
mkdir %<gruppenavn>.dropfolder%\deploy -p
xcopy %<gruppenavn>.dropfolder.app.target%\ciapp-1.0-SNAPSHOT.war %<gruppenavn>.dropfolder%\deploy /Y
```

Først oppretter vi en ny mappe som vi kan kopiere den ferdig bydge `.war`-filen som er løsningen vi skal deploye, som 1 fil. Vi gjør dette av samme grunn som vi opprettet dropfolderen tidligere, for å kunne dele filer på kryss på av byggekonfigurasjoner. Om mappen finnes, oppretter vi den på nytt (`-p`-flagget).

Så kopierer vi `.war`-filen som vi lagde i forrige byggesteg, over til deploy folderen.

### Trigger: Finish Build Trigger

1. I menyen til venstre velg `Triggers` -> `Add new trigger`
2. Velg `Finish build trigger` og i listen over byggekonfigurasjoner, velg `Compile & Test`-steget tilhørende gruppen din. 

Dette sørger for at `Package`-konfigurasjonen starter automatisk når `Compile & Test` fullfører.

## Build Configuration: Deploy

1. På admin-fremsiden til *gruppeprosjektet*, velg `General Settings` og `Create build configuration`
2. Name: `Deploy` -> `Save`

### Build Step 1: Run Jetty webserver

1. I menyen til venstre velg `Build Steps` -> `Add build step`
2. Runner type: `Command Line`
3. Step name: `Run Jetty webserver`
4. Execute step: `If all previous steps finished successfully`
5. Working directory: `%<gruppenavn>.dropfolder%`
6. Custom script:

```
start "" java -jar %<gruppenavn>.app.target%\dependency\jetty-runner.jar %<gruppenavn>.app.target%\ciapp-1.0-SNAPSHOT.war
```

(Ja, med doble quotes etter Start)

Dette kjører pluginen `jetty-runner` med `.war` filen vår, dvs starter løsningen i en web server.

### Trigger: Finish Build Trigger

1. I menyen til venstre velg `Triggers` -> `Add new trigger`
2. Velg `Finish build trigger` og i listen over byggekonfigurasjoner, velg `Package`-steget tilhørende gruppen din.

Dette sørger for at `Deploy`-konfigurasjonen starter automatisk når `Package` fullfører.

# Test at det fungerer

1. Gjør en endring lokalt i git repoet og push endringen til GitHub:
	- Gjør endring
	- `git commit -am "commit melding i doble quotes"`
	- `git push origin master`
2. Gå til Projects-siden (hovedsiden) i TeamCity
3. Vent tålmodig (inntil 60 sek) og se at `Compile & Test` starter automagisk
4. Se at `Package` steget starter automatisk når `Compile & Test` fullfører
5. Se at `Deploy` steget starter automatisk når `Package` fullfører
6. Gå til http://nerdschool.cloudapp.net:<din port> og se at nettsiden kjører


