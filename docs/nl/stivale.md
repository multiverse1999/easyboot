Stivale-ondersteuning in Easyboot
=================================

Gaat niet gebeuren, nooit. Dit bootprotocol heeft een aantal zeer ernstige slechte ontwerpkeuzes en het is een enorm
beveiligingsrisico.

Ten eerste hebben stivale kernels een ELF header, maar op de een of andere manier zou je moeten weten dat de header niet geldig is;
niets, absoluut niets, impliceert in de header dat het geen geldige SysV ABI kernel is. In Multiboot moeten er wat magische bytes
aan het begin van het bestand staan, zodat je het protocol kunt detecteren; er is niets dergelijks in stivale / stivale2. (Het
anker helpt je niet, omdat dat *overal* in het bestand kan voorkomen, dus moet je eigenlijk *het hele bestand doorzoeken* om er
zeker van te zijn dat het niet stivale compatibel is.)

Ten tweede gebruikt het secties; die volgens de ELF-specificatie (zie [pagina 46](https://www.sco.com/developers/devspecs/gabi41.pdf))
optioneel zijn en waar geen enkele loader zich druk om zou moeten maken. Loaders gebruiken de Execution View en niet de Linking
View. Het implementeren van sectieparsing alleen vanwege dit ene protocol is een krankzinnige overhead in elke loader, waar de
systeembronnen meestal al schaars zijn.

Ten derde bevinden sectieheaders zich aan het einde van het bestand. Dit betekent dat u, om stivale te detecteren, het begin van
het bestand moet laden, ELF-headers moet parsen, vervolgens het einde van het bestand moet laden en sectieheaders moet parsen,
en dan ergens in het midden van het bestand moet laden om de daadwerkelijke taglijst te krijgen. Dit is de slechtste mogelijke
oplossing die er is. En nogmaals, er is absoluut niets dat aangeeft dat een loader dit zou moeten doen, dus u moet het voor alle
kernels doen om erachter te komen dat de kernel stivale niet gebruikt. Dit vertraagt ​​ook de detectie van *alle andere*
bootprotocollen, wat onacceptabel is.

De taglijst wordt actief gepolled door de applicatieprocessors en de kernel kan op elk moment bootloadercode aanroepen, wat
betekent at u het geheugen van de bootloader niet kunt terugvorderen, anders is een crash gegarandeerd. Dit gaat in tegen de
filosofie van **Easyboot**.

Het ergste is dat het protocol verwacht dat bootloaders code injecteren in elke stivale-compatibele kernel, die vervolgens wordt
uitgevoerd op het hoogst mogelijke privilegeniveau. Ja, wat kan er misgaan, toch?

Omdat ik weiger slechte kwaliteit code uit mijn handen te geven, zal er geen stivale support zijn in **Easyboot**. En als je mijn
advies opvolgt, zou geen enkele hobby OS-ontwikkelaar het ooit moeten gebruiken voor hun eigen bestwil.
