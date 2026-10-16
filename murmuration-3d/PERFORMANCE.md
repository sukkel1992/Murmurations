# Prestaties en browser-GPU

## De 8.000–10.000 boids uitproberen

De teller kan nu tot 10.000. De simulatie en de buurzoekactie draaien op dit moment op de CPU; de browser-GPU wordt niet gebruikt om de simulatie te berekenen. De aangepaste buurzoekactie en projectie verminderen CPU-werk, maar garanderen geen bepaalde FPS. Dat hangt onder meer af van de processor, browser, schermresolutie en gekozen 2D-, 3D- of 4D-ruimte.

1. Open de preview op `http://localhost:4173/` en vernieuw de pagina.
2. Kies volledig scherm om extra browser- en paneelwerk te beperken.
3. Verhoog het aantal in stappen van ongeveer 500 en wacht na iedere stap 10 seconden. Noteer het aantal en de getoonde FPS.
4. Herhaal de meting voor 2D, 3D en 4D. 4D doorzoekt meer buurcellen en kan eerder vertragen.
5. Houd voor een soepele animatie ongeveer 30 FPS of hoger aan. Zakt de teller langdurig onder 24 FPS, ga dan terug naar de laatste stabiele waarde. Test ook met de browser-tab op de voorgrond.
6. Deel bij een probleem de browser/versie, het apparaat, de ruimtevorm, het aantal boids en de gemeten FPS. Daarmee is te bepalen of verdere CPU-optimalisatie of de GPU-stap nodig is.

## Controleren of de browser WebGPU kan aanbieden

Dit controleert alleen beschikbaarheid; het zet deze app **niet** over op de GPU.

1. Gebruik een recente Chrome- of Edge-versie. `localhost` is een secure context; een preview via een beveiligde HTTPS-forwarding-URL kan ook.
2. Open DevTools → Console op de preview en voer uit:

   ```js
   Boolean(navigator.gpu)
   await navigator.gpu?.requestAdapter()
   ```

   De eerste regel hoort `true` te geven en de tweede een adapterobject. `false`, `undefined` of `null` betekent dat WebGPU voor die browser/context niet beschikbaar is.
3. Controleer `chrome://gpu` (Chrome) of `edge://gpu` (Edge) en zoek naar de WebGPU-status. Controleer ook dat hardwareversnelling in de browserinstellingen aan staat en herstart de browser als je die instelling wijzigt.
4. Als WebGPU geblokkeerd of niet beschikbaar is, test eerst een bijgewerkte browser en de lokale preview-URL. Zet niet willekeurig experimentele browserflags aan; apparaatbeleid of GPU-/driverondersteuning kan de blokkade veroorzaken.
5. Een beschikbare adapter versnelt de huidige simulatie niet: deze code gebruikt nog steeds JavaScript op de CPU. De GPU-stap vereist een aparte WebGPU-implementatie in de app.

## Volgende ontwikkelstap: echte GPU-simulatie

Als de metingen aantonen dat de CPU-versie te traag is, is de volgende stap een WebGPU-backend met:

1. GPU-buffers voor posities, snelheden en veldkleuren.
2. Een compute-shader voor buurzoekactie en boid-integratie; de huidige grid-structuur moet ook naar GPU-data worden vertaald.
3. Rendering vanuit GPU-buffers, bij voorkeur als instanced punten/sprites, om data niet iedere frame terug naar de CPU te kopiëren.
4. Behoud van 2D/3D/4D, ruimtebegrenzing, camera, velden en aanraking, plus de bestaande CPU-route als fallback wanneer WebGPU ontbreekt.
5. Vergelijkende tests bij 1.000, 8.000 en 10.000 boids op dezelfde browser en hetzelfde apparaat. Controleer FPS, correcte aantallen en dat beide backends dezelfde bediening behouden.

De GPU moet beschikbaar zijn in de browser die de preview toont. Een GPU in een remote ontwikkelomgeving is hiervoor niet nodig; het is de browser op het apparaat van de gebruiker die de WebGPU-adapter aanbiedt.
