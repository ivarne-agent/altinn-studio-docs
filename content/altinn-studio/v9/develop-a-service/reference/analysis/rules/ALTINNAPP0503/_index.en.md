---
draft: true
title: "ALTINNAPP0503: instance property the instance data mutator does not save"
description: "Code changes a property of the instance or a data element that IInstanceDataMutator does not save"
weight: 53
---

This diagnostic is reported when code in the app changes a property of the instance or of a data
element that `IInstanceDataMutator` does not save. The message reads:

> 'Instance.DueBefore' is not saved by the instance data mutator: a save throws on a change to it,
> or ignores one that Storage makes itself. Only these are saved: Instance.DataValues,
> Instance.PresentationTexts, DataElement.Tags, DataElement.Metadata,
> DataElement.UserDefinedMetadata, DataElement.Refs, DataElement.Filename.

Category `CodeSmells`, severity **warning**.

Process hooks, service tasks, `IInstantiationProcessor` and `IDataWriteProcessor` change the
instance by editing `IInstanceDataMutator.Instance` directly. The app saves only these
properties:

- `DataValues` and `PresentationTexts` on the instance
- `Tags`, `Metadata`, `UserDefinedMetadata`, `Refs` and `Filename` on data elements

A change to anything else makes the save fail with an error. Properties that Altinn Storage
writes itself, such as `LastChanged` and `IsRead`, do not make the save fail, but the app does
not save changes to them either. App code should never look as if it works while it silently
does nothing. The full rules are described in
[Endre dataverdier og metadata fra koden]({{< relref "/altinn-studio/v9/develop-a-service/data/instance-metadata" >}})
(Norwegian).

## What the rule reports

The rule checks all code in the app, not only code that gets an `IInstanceDataMutator`, because
the app passes the instance on to many interfaces. It reports

- assignments with `=`, `+=`, `??=`, `++`, `--` and deconstruction to properties of the types
  the instance is made of: `Instance`, `DataElement`, `InstanceStatus`, `Substatus`,
  `InstanceOwner`, `ProcessState`, `ProcessElementInfo`, `CompleteConfirmation` and the other
  types reachable from `Instance`, except the saved properties above
- calls that change a list or dictionary property of these types that the app does not save,
  such as `Add`, `Remove`, `Clear`, `Insert` and indexer assignment, for example
  `instance.Data.Add(...)` and `instance.CompleteConfirmations.Add(...)`

The rule does not report `KeyValueEntry`, because its `Key` and `Value` are part of the metadata
the app saves. Nor does it report object initialisers, such as `new Instance { DueBefore = ... }`,
because they build a new object.

## How to fix it

Incorrect:

```csharp
mutator.Instance.DueBefore = DateTime.UtcNow.AddDays(14);
mutator.Instance.Data.Add(attachment);
```

Correct:

```csharp
(mutator.Instance.DataValues ??= [])["caseNumber"] = "2026-1";
mutator.GetDataElement(element).Tags = ["signed"];
mutator.AddBinaryDataElement("attachment", "application/pdf", "attachment.pdf", bytes);
```

Add and remove data elements with `AddFormDataElement`, `AddBinaryDataElement` and
`RemoveDataElement`, not directly in `Instance.Data`. To change a data value or presentation
text that `dataFields` or `presentationFields` derives, change the field in the data model.

If the app builds its own `Instance`, for example to send it to another system, set the
properties in an object initialiser:

```csharp
var instance = new Instance
{
    DueBefore = DateTime.UtcNow.AddDays(14),
};
```

If you have to set the properties after the object is built, suppress the warning for that code
only: put `#pragma warning disable ALTINNAPP0503` before it and
`#pragma warning restore ALTINNAPP0503` right after it.

```csharp
#pragma warning disable ALTINNAPP0503
instance.DueBefore = DateTime.UtcNow.AddDays(14);
#pragma warning restore ALTINNAPP0503
```
