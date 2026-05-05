---
title: "Lesen und Schreiben"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
# bookHref: ''
# bookIcon: ''
---

# Lesen und Schreiben

Dieser Guide behandelt die alltägliche Arbeit damit, Fakten ins Ledger zu schreiben und wieder auszulesen: die zwei Schreibwege des SDK, einfache Reads einzelner Entitäten und filterbasierte Reads, Konventionen für Metadaten beim Schreiben, die Semantik von Retraktionen und idempotenten Re-Runs sowie die typisierten Fehler, die bei falschen Aufrufen ausgelöst werden. Vorausgesetzt sind ein definiertes Schema und ein geöffneter Store. Zur Schemaseite siehe [Ein Schema definieren](/docs/guides/defining-a-schema), zur konzeptionellen Grundlage von Writes, Projektionen und Retraktionen siehe [Das Ledger](/docs/concepts/the-ledger).

Das SDK bietet zwei Wege zum Schreiben. Die *Batch API* gruppiert mehrere Schreibvorgänge zu einer transaktionalen Einheit und ist der Weg, den der meiste Anwendungscode nutzt. Die *Direct API* schreibt jeweils eine einzelne Assertion und gibt deren ID zurück. Sie ist passend, wenn ein bestimmter Schreibvorgang später einzeln adressiert werden muss. Beide Wege erzeugen identische Einträge im Ledger. Der Unterschied liegt in der Ergonomie und darin, wann der Kernel die Writes anwendet.

## Der Batch-Weg

Eine Batch-Transaktion wird mit `sdk.batch` geöffnet, mit Entity-Handles und Feldoperationen befüllt und atomar committet.

```python
tx = sdk.batch(meta={"trace_id": "seed-001", "source": "import_job"})

de = tx.entity(Country, iso_code="DE")
de.name.set("Germany")

alice = tx.entity(Person, person_id="p-001", locale="en")
alice.name.set("Alice")
alice.tag.add("vip")
alice.home_country.set(de)

result = tx.commit()
print("written assertions:", len(result.apply_result.assertion_ids))
```

Das an `sdk.batch` übergebene Argument `meta` wird auf jede Assertion der Transaktion übernommen, sofern es nicht pro Aufruf überschrieben wird. `tx.entity(Cls, **identity)` gibt ein Handle zurück, auf dem Feldoperationen gesammelt werden: `.field.set(...)` für Felder mit einfacher Kardinalität, `.field.add(...)` für Felder mit mehrfacher Kardinalität. Handles kennen die Adresse der Entität und die Prädikate des Schemas. Writes sind dadurch typisiert und werden lokal validiert, bevor sie in die Queue gehen. Typ- oder Kardinalitätsfehler erscheinen an der Aufrufstelle, nicht erst beim Commit.

Referenzen zwischen Entitäten werden über Handles übergeben, nicht über Strings. Wenn `alice.home_country.set(de)` aufgerufen wird und `de` das Handle einer `Country` ist, speichert das SDK die Adresse dieses Landes als Wert von `alice.home_country`. Die Referenz wird beim Commit in einen verwalteten String aufgelöst. Die API lehnt Werte ab, die nicht aus `tx.entity` oder als verwaltete Referenz aus `sdk.ref` stammen. Referenzstrings sollten nie von Hand gebaut werden.

Eine Transaktion kann vor dem Commit geprüft werden. `tx.preview()` gibt einen Plan mit allen Operationen zurück, die der Commit ausführen würde, ohne etwas zu schreiben. Das ist nützlich, wenn eine Transaktion dynamisch aufgebaut wird und vor dem Sichtbarwerden der Writes noch validiert werden soll.

```python
plan = tx.preview()
print("planned ops:", len(plan.ops))
```

`tx.commit()` wendet die gesammelten Operationen atomar an: Entweder erreichen alle Assertions des Batches das Ledger, oder keine. Fehler treten auf, bevor Writes sichtbar werden. Einen teilweise angewendeten Batch lässt der Kernel nicht zu.

### Partielle Identity

Wenn eine Entität eine zusammengesetzte Identity hat und beim Erzeugen eines Handles nur ein Teil davon bekannt ist, wird das Handle mit den bekannten Werten deklariert und später über `bind` vervollständigt.

```python
bob = tx.entity(Person, person_id="p-002")
bob = bob.bind(locale="en")
bob.name.set("Bob")
```

`bind` gibt ein vollständig identifiziertes Handle zurück. Feldmethoden auf einem partiell identifizierten Handle lösen einen Fehler aus, statt stillschweigend an eine falsche Adresse zu schreiben.

## Der direkte Weg

Die Direct API ist passend, wenn ein einzelner Schreibvorgang individuell adressierbar sein muss: wenn die Assertion-ID eines bestimmten Writes für eine spätere Retraktion behalten werden soll, wenn jeder Call eigene Metadaten trägt oder wenn ein externer Event-Stream Assertion für Assertion übersetzt wird.

```python
ref = sdk.ref(Person, person_id="p-001", locale="en")

asrt_alias = sdk.add(Person.alias, ref, "Alice Cooper",
                     meta={"source": "review", "trace_id": "rev-44"})
asrt_age   = sdk.set(Person.age, ref, 31,
                     meta={"source": "review", "trace_id": "rev-44",
                           "version": "v2", "valid_from": "2024-01-01T00:00:00+00:00"})
revoker    = sdk.retract(asrt_alias,
                         meta={"source": "review", "trace_id": "rev-45"})
```

`sdk.ref(Cls, **identity)` registriert eine Identity und gibt den verwalteten Referenzstring zurück, den die folgenden Calls verwenden. `sdk.set` und `sdk.add` geben die Assertion-ID des Writes zurück. Diese ID sollte behalten werden, wenn die Assertion später referenziert oder zurückgezogen werden muss. `sdk.retract` nimmt eine Assertion-ID, keinen Wert, und gibt die Assertion-ID der Retraktion zurück — oder `None`, wenn die Ziel-Assertion bereits zurückgezogen wurde.

Die Direct API ist die richtige Wahl, wenn ein einzelner Write ein eigenes Handle braucht: für Retraktion per ID, für unterschiedliche Metadaten pro Call oder für die Übersetzung eines externen Event-Streams, bei dem jedes Event genau eine Assertion wird. Für alles andere ist die Batch API kürzer, standardmäßig atomar und der empfohlene Weg.

## Einzelne Entitäten lesen

Der einfachste Read ist ein Snapshot einer einzelnen Entität, adressiert über ihre Identity-Koordinaten.

```python
alice = sdk.get(Person, person_id="p-001", locale="en")
print(alice.name)         # "Alice"
print(alice.tag)          # {"vip"}
print(alice.home_country) # the Country snapshot
```

`sdk.get(Cls, **identity)` gibt einen read-only Snapshot zurück: die Projektion des Ledgers in der aktiven View, bei Bedarf aus den relevanten Einträgen berechnet. Felder mit einfacher Kardinalität erscheinen als Werte, Felder mit mehrfacher Kardinalität als Sets, und `entity_ref`-Felder als Snapshots der referenzierten Entität. Navigation über Referenzfelder ist dadurch gewöhnlicher Attributzugriff, etwa `alice.home_country.iso_code`.

Wenn zusätzlich zum reduzierten Feldwert die Assertion-Historie gebraucht wird, bietet der Snapshot einen Assertions-Namespace pro Feld.

```python
print(alice.assertions.age.active)         # current assertions
print(alice.assertions.age.history)        # all assertions ever
print(alice.assertions.age.version("v2"))  # filtered by meta version
```

`active` gibt die Assertions zurück, die in den Snapshot reduziert wurden. `history` gibt jede Assertion zurück, die das Ledger für dieses Feld an dieser Entität hält, einschließlich zurückgezogener Einträge und mit Flag für den Retraktionsstatus. `version(...)` filtert nach dem Meta-Key `version`. Das ist die übliche Form, um parallele Versionen eines Werts im Ledger zu führen — etwa einen alten und einen neuen Steuersatz oder einen Entwurfs- und einen Veröffentlichungspreis.

Ein von `sdk.get` zurückgegebener Snapshot ist `None`, wenn an der angegebenen Identity in der aktiven View keine Entität existiert, also keine `<T>:exists`-Assertion im Scope liegt. Die Funktion wirft bei einer fehlenden Entität keinen Fehler, sondern gibt `None` zurück. Der Caller muss diese Abwesenheit behandeln.

## Per Filter lesen

Für Entitäten, die zu einem groben Identity-und-Feld-Filter passen, gibt `sdk.find` eine Liste entitätsförmiger Zeilen zurück.

```python
vips = sdk.find(Person, tag="vip", limit=50)
for row in vips:
    print(row.ref, row.name)
```

Jede Zeile trägt ein `ref`-Attribut mit dem verwalteten Referenzstring und stellt die Felder der Entität als Attribute bereit. Die Funktion ist die ergonomische Form, um Entitäten nach einfachen Kriterien aufzulisten, und passt, wenn der Filter als Feldgleichheiten ausgedrückt werden kann. Für reichere Queries — Joins über Entitäten hinweg, Negation, Komposition über Subregeln — ist eine `Rule`, ausgeführt mit `sdk.run`, der richtige Weg. Die Rule-DSL und ihre Mechanik werden unter [Regeln schreiben](/docs/guides/writing-rules) behandelt.

## Metadaten-Konventionen

Das Argument `meta` akzeptiert beliebige Werte mit String-Keys. Der Kernel verlangt nur, dass die Werte JSON-kompatibel sind. Einige Keys tauchen in factpy-Codebasen regelmäßig als Konvention auf, nicht als Pflicht. Sie konsistent zu verwenden macht spätere Audit-Pakete leichter lesbar.

`source` identifiziert das System oder den Prozess, der für die Assertion verantwortlich ist, etwa `"import_job"`, `"reviewer"` oder `"derivation:drv.vip_inference"`. `trace_id` verbindet mehrere Assertions, die unter derselben logischen Operation geschrieben wurden. Das ist beim Lesen von Audit-Paketen und beim Rekonstruieren einzelner Läufe zentral. `version` trägt parallele Versionen eines Werts, wenn mehrere Varianten desselben logischen Fakts gleichzeitig im Ledger existieren sollen; `assertions.<field>.version(...)` liest sie aus dem Snapshot zurück. `valid_from` und `valid_to` tragen semantische Zeit, falls diese sich vom physischen Assertion-Zeitstempel unterscheidet. `confidence` ist die übliche Form für Fakten aus probabilistischen Quellen oder Modelloutputs.

Keiner dieser Keys ist vom Kernel vorgeschrieben. Die Konventionen sind das, worauf sich die meisten factpy-Codebasen einpendeln. Wer ihnen folgt, sorgt dafür, dass ein Audit-Reader die Metadaten in einer vertrauten Struktur vorfindet.

## Retraktion ist ein neuer Eintrag

Eine Retraktion verändert oder entfernt die Assertion, auf die sie zielt, nicht. Sie ist ein neuer Ledger-Eintrag, der über dessen ID auf die Ziel-Assertion verweist und dafür sorgt, dass diese Assertion bei der Snapshot-Berechnung übersprungen wird. Die Ziel-Assertion bleibt im Ledger, ist über `assertions.<field>.history` abfragbar und gehört zu jedem Audit-Paket, das der Lauf erzeugt. In factpy gibt es keine Operation, die einen Fakt in-place verändert. Was wie Korrektur aussieht, ist immer entweder eine neue Assertion, die eine frühere in der Projektion ersetzt, oder eine Retraktion, die den ausdrücklichen Rückzug der früheren Assertion festhält.

Die beiden Muster drücken Unterschiedliches aus, und die Wahl ist für die Audit-Story relevant. Eine neue Assertion sagt: *Die Wahrheit hat sich geändert.* Eine Person hieß Alice und heißt jetzt Alicia; beide Assertions gehören zur Historie, und die Projektion zeigt den neuesten nicht zurückgezogenen Wert. Eine Retraktion sagt: *Diese Assertion hätte nicht geschrieben werden sollen.* Jemand hat geschrieben, dass eine Person den Tag `blocked` trägt, und hält nun fest, dass dieser Write ein Fehler war. Die ursprüngliche Assertion bleibt zusammen mit der Retraktion erhalten, und der Audit-Trail macht den Akt des Zurückziehens genauso sichtbar wie den Akt des Schreibens. Im Zweifel ist eine neue Assertion die passende Wahl. Retraktionen sind für Fälle reserviert, in denen der frühere Schreibvorgang selbst zurückgewiesen wird.

## Idempotenz und Re-Runs

Einen Write erneut auszuführen, der bereits ausgeführt wurde, ist kein No-op. Es entsteht ein neuer Ledger-Eintrag mit demselben Prädikat, Subjekt und Wert wie vorher, aber mit eigener Assertion-ID, eigenem Zeitstempel und möglicherweise anderen Metadaten. Der Snapshot sieht nach dem Re-Run gleich aus wie vorher; das Ledger ist um einen Eintrag gewachsen.

Bei einem Feld mit mehrfacher Kardinalität ist der Re-Run für die Projektion harmlos, weil der Snapshot doppelte Werte zu einem zusammenfasst. Bei einem Feld mit einfacher Kardinalität bleibt der Snapshot ebenfalls unverändert, weil die neue Assertion als neuester Wert an die Stelle der alten tritt und denselben Wert trägt. In beiden Fällen hält das Ledger nun fest, dass der Wert zu einem späteren Zeitpunkt erneut bestätigt wurde. Das kann selbst nützliche Information sein: „Dieser Wert wurde am 2024-04-01 vom Import-Job erneut bestätigt“ ist eine sinnvolle Audit-Beobachtung, die ein überschreibender Store nicht erhalten würde. Wenn echte Idempotenz gebraucht wird — also „nur schreiben, wenn noch nicht vorhanden“ —, muss die Anwendung den Write in einen Read-then-Write-Check einbetten. Die Alternative ist, zu akzeptieren, dass Re-Runs äquivalente Einträge ansammeln und die Audit-Story diese zusätzlichen Records trägt.

## Wichtige Fehler

Das SDK wirft eine typisierte Exception-Hierarchie mit stabilen `code`-Attributen, auf die nachgelagerter Code pattern-matchen kann. `SDKSchemaError` wird ausgelöst, wenn ein Read oder Write etwas referenziert, das das Schema nicht erlaubt: ein unbekanntes Feld, fehlende Identity-Werte oder ein Type-Domain-Mismatch bei einem Wert. `CardinalityError` wird ausgelöst, wenn `sdk.set` auf einem Feld mit mehrfacher Kardinalität oder `sdk.add` auf einem Feld mit einfacher Kardinalität aufgerufen wird, da die Kardinalität Teil der Felddeklaration ist und der Fehler anzeigt, welche der beiden Operationen hätte verwendet werden müssen. `EntityNotFoundError` wird von `sdk.edit(...)` ausgelöst, wenn an der angegebenen Identity keine Entität existiert. `sdk.get(...)` dagegen wirft bei fehlender Entität keinen Fehler, sondern gibt `None` zurück. `SDKStoreError` ist der Sammelfall für Store-Level-Probleme: ungültige Referenzen, Schema-Digest-Mismatches beim Öffnen oder strukturelle Probleme im Aufruf. Jede Exception trägt ein `code`-Attribut, etwa `"FIELD_CARDINALITY_MISMATCH"` oder `"UNRESOLVABLE_E_REF"`, das über Patch-Versionen stabil bleibt. Die vollständige Liste steht in der [Error-Codes-Referenz](/docs/reference/error-codes).

## Nächste Schritte

Der [Guide zum Schreiben von Regeln](/docs/guides/writing-rules) behandelt Queries mit der Rule-DSL, einschließlich Joins über Entitäten hinweg, Subregeln und Negationsmuster. Der [Persistenz-Guide](/docs/guides/persistence) behandelt Ledger-Dateien, den Schema-Digest und Prozessgrenzen. Die konzeptionelle Grundlage aller oben beschriebenen Reads und Writes steht unter [Das Ledger](/docs/concepts/the-ledger).