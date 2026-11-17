---
title: "ALTINNAPP0503: instansegenskap som instansmutatoren ikke lagrer"
description: "Kode endrer en egenskap på instansen eller et dataelement som IInstanceDataMutator ikke lagrer"
weight: 53
---

Denne diagnostikken melder fra når koden i appen endrer en egenskap på instansen eller et
dataelement som `IInstanceDataMutator` ikke lagrer. Meldingen ser slik ut:

> 'Instance.DueBefore' is not saved by the instance data mutator: a save throws on a change to it,
> or ignores one that Storage makes itself. Only these are saved: Instance.DataValues,
> Instance.PresentationTexts, DataElement.Tags, DataElement.Metadata,
> DataElement.UserDefinedMetadata, DataElement.Refs, DataElement.Filename.

Kategori `CodeSmells`, alvorlighetsgrad **advarsel**.

Prosess-hooks, systemoppgaver, `IInstantiationProcessor` og `IDataWriteProcessor` endrer
instansen ved å endre `IInstanceDataMutator.Instance` direkte. Appen lagrer bare disse
egenskapene:

- `DataValues` og `PresentationTexts` på instansen
- `Tags`, `Metadata`, `UserDefinedMetadata`, `Refs` og `Filename` på dataelementene

Endrer koden noe annet, stopper lagringen med en feil. Egenskaper som Altinn Storage skriver
selv, som `LastChanged` og `IsRead`, stopper ikke lagringen, men appen lagrer heller ikke
endringene i dem. Vi har valgt dette fordi kode som ser ut til å virke, men som ikke gjør noe,
er vanskelig å oppdage. Hele regelsettet står i
[Endre dataverdier og metadata fra koden]({{< relref "/altinn-studio/v9/develop-a-service/data/instance-metadata" >}}).

## Dette melder regelen fra om

Regelen sjekker all kode i appen, ikke bare kode som får en `IInstanceDataMutator`, fordi
appen sender instansen videre til mange grensesnitt. Den melder fra om

- tilordninger med `=`, `+=`, `??=`, `++`, `--` og dekonstruksjon til egenskaper på typene
  instansen består av: `Instance`, `DataElement`, `InstanceStatus`, `Substatus`,
  `InstanceOwner`, `ProcessState`, `ProcessElementInfo`, `CompleteConfirmation` og de andre
  typene du når fra `Instance`, unntatt de lagrede egenskapene over
- kall som endrer en liste eller ordbok på disse typene som appen ikke lagrer, for eksempel
  `Add`, `Remove`, `Clear`, `Insert` og tilordning med indeks, som `instance.Data.Add(...)` og
  `instance.CompleteConfirmations.Add(...)`

Regelen melder ikke fra om `KeyValueEntry`, fordi `Key` og `Value` er en del av metadataene
appen lagrer. Den melder heller ikke fra om objektinitialiserere, som
`new Instance { DueBefore = ... }`, fordi de bygger et nytt objekt.

## Slik retter du det

Feil:

```csharp
mutator.Instance.DueBefore = DateTime.UtcNow.AddDays(14);
mutator.Instance.Data.Add(vedlegg);
```

Riktig:

```csharp
(mutator.Instance.DataValues ??= [])["caseNumber"] = "2026-1";
mutator.GetDataElement(element).Tags = ["signed"];
mutator.AddBinaryDataElement("vedlegg", "application/pdf", "vedlegg.pdf", bytes);
```

Legg til og fjern dataelementer med `AddFormDataElement`, `AddBinaryDataElement` og
`RemoveDataElement`, ikke direkte i `Instance.Data`. En dataverdi eller presentasjonstekst som
`dataFields` eller `presentationFields` regner ut, endrer du gjennom feltet i datamodellen.

Lager appen sin egen `Instance`, for eksempel for å sende den til et annet system, kan du sette
egenskapene i en objektinitialiserer:

```csharp
var instance = new Instance
{
    DueBefore = DateTime.UtcNow.AddDays(14),
};
```

Må du sette egenskapene etter at objektet er laget, kan du skru av advarselen bare for den
koden: sett `#pragma warning disable ALTINNAPP0503` foran og
`#pragma warning restore ALTINNAPP0503` rett etter.

```csharp
#pragma warning disable ALTINNAPP0503
instance.DueBefore = DateTime.UtcNow.AddDays(14);
#pragma warning restore ALTINNAPP0503
```
