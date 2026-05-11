---
title: "Julemarket: Fra Manuelt Papirarbejde til AI-Drevet Administration"
date: 2026-05-11
draft: false
description: "En fullstack Java/React prototype der automatiserer rekruttering, kontrakter og fakturering for et julemarkeds-event — og et kig på vejen mod et fuldt automatiseret AI-flow."
tags: ["java", "react", "ai", "automation", "ollama", "javalin", "postgresql"]
categories: ["projects"]
series: []
---

## Baggrund

Hvert år i starten af december åbner dørene til [Engestofte Gods](https://www.engestofte.com/da/for-stadeholdere) i Maribo for et af årets hyggeligste arrangementer — et julemarked i de historiske stalde, lader og den gamle gårdsplads. I 2026 løber det af stablen den **5.-6. december, kl. 10-16**.

Bag kulisserne koordinerer én person — eventkoordinatoren — hele rekrutteringsprocessen: hun finder potentielle stadeholdere, sender dem en ansøgningsformular manuelt, vurderer ansøgningerne ud fra produktkvalitet, originalitet og tidligere deltagelse, sender individuelle kontrakter, modtager signaturer og udsteder fakturaer. Alt dette foregår via e-mail, regneark og papir — for op mod 80-90 stadeholdere.

Det er præcis den slags manuelt, teksttungt arbejde, som AI er skabt til at hjælpe med.

Spørgsmålet vi stillede os selv: **Giver det mening at automatisere dette med AI?**

Svaret på det spørgsmål er selve udgangspunktet for dette projekt. Vi byggede en **fullstack admin-applikation** som en prototype for at teste, om en AI-assisteret arbejdsgang faktisk er brugbar i praksis — eller om det bare er teknologi for teknologiens skyld.

---

## Teknologivalg

### Backend: Java 17 + Javalin + Hibernate + PostgreSQL

Vi valgte vores kendte stack fra undervisning og egne projekter.

**Javalin** er et letvægts Java-framework, der giver os en simpel REST API uden al den boilerplate, som Spring Boot medfører. Til en intern admin-applikation med et begrænset antal endpoints er det et oplagt valg — hurtig opsætning, nem debugging.

**Hibernate + PostgreSQL** håndterer persistens. Data modellen er simpel: virksomheder, kontrakter, fakturaer og en log over sendte e-mails. Hibernate giver os ORM uden at skrive rå SQL for alle CRUD-operationer, og PostgreSQL er robust nok til produktion hvis vi skulle skalere.

**Ollama med llama3.1:8b** er vores lokale LLM-integration. Vi kørte modellen lokalt frem for at bruge en ekstern API (f.eks. OpenAI) af to grunde:
- **Ingen API-udgifter** under udvikling og test
- **Data forbliver lokalt** — virksomhedsoplysninger og kontaktdata sendes ikke til tredjeparter

### Frontend: React 19 + Vite + Bootstrap

React er vores foretrukne frontend-framework. Vite giver hurtig HMR og et moderne build-setup uden konfigurationsmareridt.

Bootstrap bruges til styling — ikke fordi det er det smukkeste, men fordi det er hurtigt at arbejde med for interne admin-interfaces, hvor UX er vigtigere end visuel originalitet.

---

## Implementeringsstrategi: Det 6-Trins Flow

Applikationen modellerer hele livscyklussen for en julemarkeds-deltager i seks trin, repræsenteret som statusser på en virksomhed i databasen.

```
OPRETTET → KONTAKTET → KONTRAKT_SENDT → KONTRAKT_UNDERSKREVET → FAKTURA_SENDT → GODKENDT
```

### Trin 1 — Opsøgende Kontakt
Admin opretter stadeholdere i systemet med relevante detaljer: standtype (mad, gaver, håndværk, dekoration), standens størrelse, varighed og antal medarbejdere. Engestofte vurderer ansøgere på bl.a. produktkvalitet, originalitet og om de adskiller sig fra eksisterende stadeholdere — disse parametre kan gives til AI'en som kontekst.

AI genererer derefter en **personlig opsøgende e-mail** på dansk, tilpasset virksomhedens profil og Engestofte Gods' koncept og atmosfære. Admin kan redigere teksten i et inline editor-felt før afsendelse.

### Trin 2 — Svarregistrering
Når stadeholderen svarer, registrerer admin svaret manuelt. Siger de ja, fortsætter flowet. Siger de nej, fjernes de fra listen. AI kan foreslå nye kandidater for at opretholde målet om 80-90 stadeholdere og sikre en god variation af standtyper.

### Trin 3 — Kontrakt
AI genererer en **individuel kontrakt** baseret på stadeholderens specifikke parametre — placering i stald, lade eller udendørs, standstørrelse, varighed og antal medarbejdere. Kontrakten inkluderer de krav Engestofte stiller: neutrale teltfarver, korrekt fastgørelse til underlag og brandregler for dekorationer. Admin gennemgår og sender den. Når kontrakten er underskrevet, markerer admin det i systemet.

### Trin 4 & 5 — Faktura
AI beregner den endelige pris ud fra en simpel formel:

```
Pris = (stole × 500 kr/dag) + (200 kr/dag fast) + (medarbejdere × 100 kr/dag)
```

Fakturaen genereres, præsenteres for admin til godkendelse, og sendes derefter. Betaling registreres manuelt.

### Trin 6 — Godkendelse
Virksomheden markeres som **bekræftet deltager** og optræder på deltagerlisten.

---

## Hvad Prototypen Beviste

Dette var en **demo** — ikke et færdigt produkt. E-mails sendes ikke rigtigt (de logges til konsollen). Der er ingen autentificering. Modellen er lille og kræver manuel opsætning af Ollama lokalt.

Men det er ikke pointen. Prototypen skulle besvare ét spørgsmål: **Er strukturen fornuftig? Sparer AI reelt tid?**

Svaret er ja. De mest tidskrævende manuelle opgaver — at skrive en personlig e-mail til 80 virksomheder, at generere 80 individuelle kontrakter, at beregne og sende fakturaer — er netop dem AI håndterer bedst. Outputtet er redigerbart, så admin bevarer kontrollen uden at skulle starte fra bunden.

---

## Vejen Fra Prototype til Automatiseret Flow

Den nuværende prototype kræver stadig mange manuelle klik og registreringer. Her er, hvordan vi vil forbedre det:

### 1. Reel E-mail Integration
Erstat mock-afsendelsen med en faktisk SMTP-integration (f.eks. via SendGrid eller JavaMail). Systemet skal sende e-mails automatisk, logge leveringsstatus og håndtere bounce.

### 2. Automatisk Statusopfølgning
I stedet for at admin manuelt registrerer svar, kan vi integrere med e-mail-indbakken via IMAP og bruge AI til at klassificere svar: "ja", "nej", "spørgsmål", "ingen svar". Status opdateres automatisk.

### 3. Automatiske Påmindelser
Systemet bør selv sende rykkere — f.eks. hvis en kontrakt ikke er underskrevet efter 5 dage, eller en faktura ikke er betalt inden forfaldsdatoen. Et simpelt cron-job tjekker dagligt og trigger AI-genererede opfølgnings-e-mails.

### 4. Upgrade til Stærkere Model
llama3.1:8b er god til simple tekster, men til mere nuancerede kontrakter og personaliserede mails ville en større model (eller et Claude API-kald) give markant bedre output. Arkitekturen er allerede sat op til at bytte model ud.

### 5. Digital Kontraktsignering
Integration med en e-signatur-tjeneste (f.eks. Penneo eller DocuSign) eliminerer det manuelle led med at sende PDF'er og afvente scannede signaturer.

### 6. Dashboard med Overblik
Et realtids-dashboard der viser, hvor mange virksomheder der er i hvert trin, hvad den forventede omsætning er baseret på underskrevne kontrakter, og hvilke der mangler handling — automatisk flagget.

---

## Konklusion

Julemarket-applikationen demonstrerer en klar tese: **AI egner sig godt til at automatisere repetitive, teksttunge administrative opgaver**, og en relativt lille mængde kode kan erstatte mange timers manuelt arbejde.

Prototypen er ikke klar til produktion, men den beviser, at arkitekturen holder. Den 6-trins arbejdsgang er logisk og dækker hele livscyklussen. AI-outputtet er brugbart og redigerbart. Strukturen er skalerbar.

Næste skridt er at erstatte mock-komponenterne med rigtige integrationer og tilføje den automatiske opfølgningslogik, der gør det til et reelt tidsbesparende værktøj — ikke bare en smart prototype.
