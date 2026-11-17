---
title: Endre dataverdier og metadata fra koden
linktitle: Dataverdier og metadata
description: Slik endrer du dataverdier, presentasjonstekster og metadata for dataelementer fra egen kode i appen
toc: true
---

En instans er én utfylling av tjenesten. I tillegg til skjemadataene og vedleggene har både instansen og hvert dataelement opplysninger om seg selv. Noen av disse opplysningene kan appen endre fra egen kode:

| Opplysning | Hvor | Eksempel på bruk |
| --- | --- | --- |
| `DataValues` | instansen | verdier som andre systemer kan lese eller søke på, for eksempel et saksnummer |
| `PresentationTexts` | instansen | tekster som skiller instansene fra hverandre i innboksen i Altinn |
| `Tags` | dataelementet | merkelapper, for eksempel at et vedlegg er signert |
| `Metadata` | dataelementet | nøkler og verdier som appen setter |
| `UserDefinedMetadata` | dataelementet | nøkler og verdier knyttet til et dataelement |
| `Refs` | dataelementet | henvisninger til andre dataelementer i instansen |
| `Filename` | dataelementet | filnavnet brukeren og mottakerne ser |

Dette er de eneste opplysningene appen lagrer når koden endrer instansen. Endrer koden noe annet, stopper lagringen med en feil. Se [Dette kan du ikke endre](#dette-kan-du-ikke-endre).

## Hvor du kan endre instansen

Du endrer instansen gjennom `IInstanceDataMutator`, som koden får fra appen i

- [prosess-hooks]({{< relref "/altinn-studio/v9/develop-a-service/reference/configuration/process/pre-post-hooks" >}}), gjennom `context.InstanceDataMutator`
- [systemoppgaver]({{< relref "/altinn-studio/v9/develop-a-service/process/service-tasks/custom" >}}), gjennom `context.InstanceDataMutator`
- [`IInstantiationProcessor`]({{< relref "/altinn-studio/v9/develop-a-service/data/prefill/how-to/custom" >}}), i overlasten av `DataCreation` som tar imot en `IInstanceDataMutator`
- [`IDataWriteProcessor`]({{< relref "/altinn-studio/v9/develop-a-service/data/dataprocessing" >}}), som får `instanceDataMutator` som første parameter

Når appen lagrer endringene, avhenger av hvor koden kjører:

- I en prosess-hook eller systemoppgave lagrer appen endringene med prosessovergangen, når koden er ferdig uten feil.
- I `IInstantiationProcessor` lagrer appen endringene når den oppretter instansen.
- I `IDataWriteProcessor` lagrer appen endringene sammen med det brukeren lagrer.

Ikke bruk `IInstanceClient.UpdateDataValues`, `IInstanceClient.UpdatePresentationTexts` eller `IDataClient` til dette i disse klassene. Mens prosessen er i en overgang, avviser Altinn Storage slike direkte endringer.

## Endre instansen

Du endrer `mutator.Instance` direkte. Dataelementene henter du med `mutator.GetDataElement(...)`, og endrer dem på samme måte. Eksempelet under er en prosess-hook som kjører når oppgaven `Task_1` avsluttes:

{{< code-title >}}
App/logic/SaksnummerHandler.cs
{{< /code-title >}}

```csharp
using Altinn.App.Core.Features;
using Altinn.App.Core.Features.Process;
using Altinn.Platform.Storage.Interface.Models;

namespace Altinn.App.Logic;

public class SaksnummerHandler : IOnTaskEndingHandler
{
    public bool ShouldRunForTask(string taskId) => taskId == "Task_1";

    public Task<HandlerResult> Execute(OnTaskEndingContext context)
    {
        IInstanceDataMutator mutator = context.InstanceDataMutator;

        // Sett en dataverdi
        (mutator.Instance.DataValues ??= [])["caseNumber"] = "2026-1";

        // Sett en presentasjonstekst
        (mutator.Instance.PresentationTexts ??= [])["applicant"] = "Ola Nordmann";

        // Merk vedleggene som signert
        foreach (DataElement element in mutator.GetDataElementsForType("vedlegg"))
        {
            mutator.GetDataElement(element).Tags = ["signed"];
        }

        return Task.FromResult(HandlerResult.Success());
    }
}
```

Dette bør du vite om hvordan appen lagrer endringene:

- Appen sender bare nøklene og egenskapene du har endret. Har en annen del av appen eller et annet system satt andre nøkler i `DataValues` eller `PresentationTexts`, beholder instansen dem.
- Erstatter du hele `Instance.DataValues` med en ny ordbok, fjerner appen alle nøklene som ikke er med i den nye. Det samme gjelder `PresentationTexts`. Vil du beholde de andre nøklene, endrer du ordboken som finnes.
- Setter du en nøkkel til `null` eller en tom tekst, fjerner appen nøkkelen.
- Legger du til et dataelement med `AddFormDataElement` eller `AddBinaryDataElement`, kan du endre egenskapene i tabellen over på elementet du får tilbake. Appen lagrer dem sammen med elementet.

## Dette kan du ikke endre

Endrer koden noe annet på instansen eller dataelementene, stopper lagringen med en feil. I en prosess-hook eller systemoppgave regner appen feilen som permanent og prøver ikke på nytt. Lagringen stopper for eksempel når koden endrer

- `DueBefore`, `VisibleAfter`, `Status` eller `Locked` på instansen
- en egenskap inne i et objekt på instansen, for eksempel `Instance.Status.Substatus.Label`
- `ContentType` eller `Locked` på et dataelement
- listen `Instance.Data` direkte, ved å legge til eller fjerne elementer i den, i stedet for å bruke `AddFormDataElement`, `AddBinaryDataElement` og `RemoveDataElement`
- en dataverdi eller presentasjonstekst som `dataFields` eller `presentationFields` i `applicationmetadata.json` regner ut fra skjemadataene, enten koden setter eller fjerner den
- `Filename` til `null` på et dataelement som har et filnavn
- `Refs` slik at listen peker på et dataelement som koden har lagt til, men som appen ikke har lagret ennå

En verdi som `dataFields` eller `presentationFields` regner ut, overskriver appen når den lagrer skjemadataene. Vil du styre verdien fra koden, fjerner du feltet fra `dataFields` eller `presentationFields`, eller du endrer feltet i datamodellen som verdien kommer fra.

Noen egenskaper skriver Altinn Storage eller prosessmotoren selv. Endrer koden dem, stopper ikke lagringen, men appen lagrer heller ikke endringen:

- `LastChanged` og `LastChangedBy` på instansen og på dataelementene
- `IsRead` og `FileScanResult` på dataelementene
- `Process` på instansen

Vi har valgt at lagringen stopper i stedet for å hoppe over endringen, fordi kode som ser ut til å virke, men som ikke gjør noe, er vanskelig å oppdage. Vi kan komme til å støtte flere egenskaper senere, for eksempel `DueBefore`.

Byggesjekken [ALTINNAPP0503]({{< relref "/altinn-studio/v9/develop-a-service/reference/analysis/rules/ALTINNAPP0503" >}}) varsler om kode som endrer en egenskap appen ikke lagrer, allerede når du bygger appen.
